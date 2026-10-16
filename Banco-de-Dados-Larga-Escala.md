# Aula 03 — System Design: Banco de Dados em Larga Escala

> **Contexto: Checkout na Black Friday**
>
> A plataforma deve processar pedidos e pagamentos com integridade financeira, reservar estoque sob alta concorrência e manter desempenho mesmo durante picos de tráfego.

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

**Escolha:** ( ) Relacional Puro (PostgreSQL) **(X) Persistência Poliglota / Híbrida**

**Justificativa arquitetural:**

Adotaria persistência híbrida, usando **PostgreSQL como fonte de verdade de pedidos, pagamentos e estoque** e soluções especializadas para necessidades complementares, como cache, sessões e busca de produtos.

O PostgreSQL fornece transações **ACID**:

- **Atomicidade:** as alterações de uma transação são confirmadas juntas ou desfeitas.
- **Consistência:** as transações preservam restrições e invariantes definidas no banco.
- **Isolamento:** transações simultâneas ficam protegidas de interferências indevidas, de acordo com o nível de isolamento adotado.
- **Durabilidade:** após um commit confirmado, os dados permanecem persistidos conforme a configuração de durabilidade.

Para cobranças, não podemos aceitar que uma operação seja tratada como aprovada apenas porque houve uma tentativa de comunicação com o parceiro. O status precisa refletir evidências confiáveis e, em resultados desconhecidos, deve existir reconciliação e idempotência.

O **Teorema CAP** trata de sistemas distribuídos sujeitos a partições de rede: durante uma partição, não é possível garantir simultaneamente consistência forte e disponibilidade de todas as operações. No fluxo financeiro, **priorizaria a consistência**, podendo recusar ou adiar operações quando não for possível confirmar e registrar o resultado de forma segura. Isso não significa que o PostgreSQL seja automaticamente “CP” em toda configuração; a escolha depende de como o banco é distribuído e configurado.

Já catálogo e sessões suportam diferentes níveis de consistência, inclusive **consistência eventual**, frequentemente associada a sistemas **BASE**. Essas cargas podem ser atendidas por outras tecnologias sem comprometer a integridade do registro financeiro.

**Além do modelo documental — outras famílias NoSQL:**

1. **Chave-valor (Redis):** adequado para sessões, carrinhos temporários e cache de dados de catálogo, com acesso de baixa latência. Não seria a fonte definitiva do estoque nem das cobranças.
2. **Motor de busca (OpenSearch/Elasticsearch):** adequado para pesquisa textual, filtros e relevância de produtos. Índices de busca podem ser atualizados de maneira assíncrona; portanto, não devem determinar o saldo financeiro ou de estoque.
3. **Grafos (Neo4j):** adequado para analisar conexões entre contas, dispositivos, endereços e métodos de pagamento em investigações de fraude.

**Decisão:** PostgreSQL preserva os dados críticos; bancos especializados atendem outros padrões de acesso.

### 1.2 Estratégia da Chave Primária

**Escolha:** ( ) Sequence / Identity ( ) UUIDv4 **(X) UUIDv7 / TSID**

**Impacto no armazenamento:**

Escolheria **UUIDv7** para pedidos e pagamentos, pois pode ser gerado por diferentes instâncias da aplicação sem depender de uma sequência central e contém uma componente temporal que favorece uma **ordenação aproximada** dos identificadores.

Em índices **B-Tree**, UUIDv4 espalha inserções por páginas aleatórias, aumentando a possibilidade de **page splits**, fragmentação e I/O. UUIDv7 tende a concentrar novas inserções em regiões próximas do fim do índice, melhorando a localidade das escritas, embora não garanta ordem global perfeita entre servidores.

| Estratégia | Vantagem | Limitação |
|---|---|---|
| Sequence / Identity | Índices compactos e inserções ordenadas | Geração distribuída exige coordenação ou particionamento de intervalos |
| UUIDv4 | Identificadores gerados independentemente | Inserções aleatórias podem gerar mais page splits |
| UUIDv7 | Identificadores distribuídos e aproximadamente ordenados no tempo | Ocupa 16 bytes e tem ordenação temporal aproximada |

**Decisão:** UUIDv7 equilibra geração distribuída e eficiência de indexação melhor que UUIDv4.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

### 2.1 Mudança de Status do Pagamento (baixa contenção por registro)

**Abordagem:** **(X) Lock otimista (`@Version`)** ( ) Lock pessimista (`SELECT FOR UPDATE`)

**Justificativa:**

A alteração de status de um pagamento apresenta baixa disputa pelo mesmo registro. O **lock otimista** permite leituras sem bloquear antecipadamente a linha e verifica se sua versão ainda é válida ao salvar.

```java
@Entity
public class Payment {
    @Id
    private UUID id;

    @Version
    private Long version;

    private String status;
}
```

Se duas transações carregam a mesma versão, a primeira atualização confirmada a incrementa; a outra detecta um conflito e não sobrescreve silenciosamente o estado atual.

Além do versionamento, validaria as **transições permitidas entre status**, processaria notificações com **idempotência** e usaria reconciliação quando o resultado do parceiro fosse desconhecido. O lock otimista não substitui essas garantias.

**Decisão:** é mais eficiente para baixa contenção porque evita bloquear registros sem necessidade.

### 2.2 Controle de Estoque Concorrente (alta contenção no mesmo registro)

**Abordagem:** ( ) Lock otimista (`@Version`) **(X) Lock pessimista (`SELECT FOR UPDATE`)**

**Justificativa:**

Quando centenas de compradores disputam as últimas unidades de um SKU, muitos processos tentam atualizar a mesma linha. O lock pessimista permite **serializar as reservas daquele SKU** e evita uma sucessão de conflitos e tentativas com lock otimista.

Exemplo conceitual de uma transação de reserva:

```sql
BEGIN;

SELECT quantity
FROM inventory
WHERE sku = 'PROMO-001'
FOR UPDATE;

-- Se quantity > 0:
UPDATE inventory
SET quantity = quantity - 1
WHERE sku = 'PROMO-001'
  AND quantity > 0;

COMMIT;
```

Se não houver unidade, a aplicação encerra a operação sem reservar e informa indisponibilidade. O banco também deve impor a invariante:

```sql
ALTER TABLE inventory
ADD CONSTRAINT chk_inventory_nonnegative
CHECK (quantity >= 0);
```

Para um simples decremento, também avaliaria uma alternativa ainda mais curta: um **UPDATE condicional e atômico**, verificando se houve uma linha atualizada:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE sku = 'PROMO-001'
  AND quantity > 0
RETURNING quantity;
```

Essa operação mantém a segurança do saldo sem exigir uma leitura separada. É uma opção de otimização que precisa ser comparada com o fluxo de reserva real.

**Como evitar saturação:** manter transações curtas, ter índice/PK em `sku`, não realizar chamadas a gateways externos segurando locks, usar **Rate Limiter**, **Bulkhead** e limites de tempo de espera.

**Decisão:** usar exclusão transacional na reserva do SKU, com tempo de bloqueio mínimo e proteção de carga.

### 2.3 Riscos e Mitigações

**Lock otimista — riscos:**

- Sob disputa intensa, muitos `OptimisticLockException` podem causar falhas percebidas pelo cliente.
- Retries irrestritos pioram a carga e podem amplificar a concorrência.

**Mitigações:** retry **limitado**, com backoff e jitter somente quando seguro; recarregar estado antes de tentar novamente; validar transição de status; garantir idempotência de operações com efeitos externos.

**Lock pessimista — riscos:**

- Filas esperando locks aumentam latência e mantêm transações/conexões ocupadas.
- Podem ocorrer deadlocks, timeouts e esgotamento do pool HikariCP.

**Mitigações:**

- Usar transações curtas e ordem consistente para adquirir vários locks.
- Definir `lock_timeout`, `statement_timeout` e limites adequados no pool.
- Usar **Rate Limiter** para reduzir pressão no SKU mais disputado.
- Usar **Bulkhead** para isolar a capacidade do fluxo de estoque.
- Evitar chamadas externas dentro de transações com linhas bloqueadas.
- Monitorar tempo de espera por locks, transações abertas e conexões pendentes.

**Decisão:** utilizar lock conforme o nível de disputa e controlar explicitamente tempo de espera, retries e recursos consumidos.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

**Cenário:** p99 de **4,8 segundos**, saturação do **HikariCP** e lentidão na consulta:

```sql
SELECT *
FROM orders
WHERE customer_id = ?
  AND order_status = 'processing';
```

**Plano de diagnóstico — antes de alterar o banco:**

**1. Correlacionar métricas da aplicação.** Verificar p95/p99, volume de requisições, tempo de aquisição de conexões, conexões ativas/ociosas/pendentes do HikariCP e exceções de timeout. Confirmar se a aplicação espera por conexões e se o tempo gasto está no banco.

**2. Identificar SQL de maior custo.** Se `pg_stat_statements` estiver habilitado:

```sql
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Isso ajuda a distinguir consulta lenta ocasional de consulta frequente com alto custo total.

**3. Inspecionar o plano da consulta.** Em ambiente seguro, com parâmetros representativos:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 'customer-123'
  AND order_status = 'processing';
```

Procurar `Seq Scan` custoso, número de linhas lidas, diferença entre linhas estimadas/reais, uso dos índices, leituras de buffers, I/O e duração por etapa. `EXPLAIN ANALYZE` **executa** a consulta; seu uso em produção exige cuidado.

**4. Investigar sessões, esperas e bloqueios.**

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle';
```

Verificar transações longas, sessões bloqueadas e se a lentidão decorre de CPU/I/O, plano ruim ou disputa por locks.

**5. Confirmar a hipótese e medir a correção.** Comparar planos e tempos antes/depois sob carga representativa. Evitar simplesmente aumentar o pool HikariCP, pois isso pode intensificar a pressão no PostgreSQL sem corrigir a causa raiz.

**Decisão:** corrigir somente após demonstrar, com métricas, o gargalo principal.

### 3.2 Decisão de Engenharia: A Abordagem de Índices

**Posicionamento:** **Não aprovaria o PR como proposto**, criando um índice individual para toda coluna presente em filtros.

Embora índices melhorem certas leituras, também consomem armazenamento e impõem custos adicionais em `INSERT`, `UPDATE` e `DELETE`. Índices individuais nem sempre são a melhor escolha para um filtro combinado; o otimizador pode não aproveitá-los como se espera.

Para a consulta crítica, avaliaria um **índice composto**:

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_status
ON orders (customer_id, order_status);
```

Também consideraria um **índice parcial**, caso haja muitas consultas pelo status `processing` e essa condição represente apenas parte dos pedidos:

```sql
CREATE INDEX CONCURRENTLY idx_orders_processing_customer
ON orders (customer_id)
WHERE order_status = 'processing';
```

O índice parcial pode ser menor, mas depende da distribuição dos registros e da possibilidade de o planejador reconhecer o predicado na consulta.

**Critérios antes de aprovar:**

- Verificar plano e tempo antes/depois com `EXPLAIN (ANALYZE, BUFFERS)`.
- Medir seletividade e frequência dos filtros.
- Medir custo nas escritas, espaço ocupado e impacto em I/O.
- Verificar se `SELECT *` é necessário ou se poucas colunas bastam.
- Executar testes de carga representativos, incluindo inserções e alterações de status.
- Planejar criação e manutenção do índice com mínimo impacto operacional.

**Decisão:** aprovar o índice que demonstre ganho mensurável para a consulta real, sem degradar excessivamente as escritas do Checkout.

---

## Resumo das decisões arquiteturais

| Tema | Decisão | Motivo principal |
|---|---|---|
| Persistência | Híbrida; PostgreSQL como fonte de verdade | Integridade financeira e ferramentas adequadas a cada carga |
| CAP | Priorizar consistência nas operações financeiras em partições | Não confirmar operações sem estado confiável |
| Chave primária | UUIDv7 | Geração distribuída e melhor localidade de escrita que UUIDv4 |
| Status de pagamento | Lock otimista | Pouca disputa por registro |
| Estoque promocional | Lock pessimista / UPDATE atômico condicional | Proteger o saldo sob alta concorrência |
| Proteção operacional | Timeouts, Rate Limiter e Bulkhead | Evitar filas ilimitadas e saturação |
| Diagnóstico | HikariCP, `pg_stat_statements`, `EXPLAIN`, `pg_stat_activity` | Demonstrar causa raiz |
| Índices | Composto ou parcial, conforme evidências | Equilibrar performance de leitura e custo de escrita |

**Conclusão:** O Checkout deve manter uma fonte transacional confiável, tratar concorrência conforme a contenção de cada operação e otimizar a persistência a partir de medições reais. O objetivo não é apenas responder rápido, mas preservar pagamentos e estoque corretos durante os picos da Black Friday.
