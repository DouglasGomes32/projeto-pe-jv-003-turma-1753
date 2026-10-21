# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

### Campos do Log

Todo log do Checkout deve ser **estruturado**, preferencialmente em JSON, para permitir busca e correlação em grande volume.

Campos mínimos:

- `timestamp`: momento exato do evento;
- `level`: nível do log, como `INFO`, `WARN` ou `ERROR`;
- `service`: nome do serviço, por exemplo `checkout`;
- `environment`: ambiente, como `prod`;
- `version`: versão do serviço/imagem em execução;
- `trace_id`: identifica toda a jornada distribuída da requisição;
- `span_id`: identifica uma etapa específica dentro do trace;
- `request_id`: identifica aquela requisição HTTP no serviço;
- `operation`: nome da operação, por exemplo `finalizar_pedido`;
- `method`: método HTTP;
- `route`: rota normalizada, por exemplo `/checkout/{id}`;
- `status_code`: status HTTP retornado;
- `duration_ms`: duração da operação;
- `outcome`: resultado funcional, como `success`, `timeout`, `rejected`;
- `dependency`: quando aplicável, identifica a dependência chamada, como `pagamento` ou `estoque`;
- `error_type`: tipo de erro quando houver;
- `message`: mensagem curta e estruturada.

Exemplo:

```json
{
  "timestamp": "2026-10-19T22:10:15Z",
  "level": "WARN",
  "service": "checkout",
  "environment": "prod",
  "version": "checkout@sha256:abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "request_id": "req-8f2a",
  "operation": "finalizar_pedido",
  "method": "POST",
  "route": "/checkout",
  "status_code": 504,
  "duration_ms": 3012,
  "outcome": "timeout",
  "dependency": "pagamento",
  "error_type": "dependency_timeout",
  "message": "timeout aguardando resposta de pagamento"
}
```

Isso permite localizar rapidamente uma chamada específica e reconstruir o caminho entre os serviços.

### O Caminho do Contexto

O identificador de rastreamento deve viajar entre os serviços usando headers HTTP padronizados de tracing distribuído.

A preferência é pelo padrão **W3C Trace Context**, usando principalmente `traceparent` e `tracestate`. Cada serviço mantém o mesmo `trace_id`, mas cria seu próprio `span_id`.

```text
Cliente
  |
  v
Checkout (trace_id = ABC)
  |
  +--> Pagamento (trace_id = ABC)
  |
  +--> Estoque    (trace_id = ABC)
```

Se a aplicação utilizar OpenTelemetry, a propagação deve ser automática pela instrumentação HTTP, evitando implementação manual inconsistente.

### LGPD nos Logs

Logs e traces não devem conter dados pessoais desnecessários. Devem ficar fora dos registros: nome completo, CPF, e-mail, telefone, endereço, dados de cartão, CVV, senha, access token, refresh token, cookies de sessão, dados bancários e payloads completos com dados pessoais.

Quando um identificador precisar aparecer, usar:

- ID interno não significativo;
- pseudonimização;
- hash quando o valor não precisar ser recuperado;
- mascaramento parcial quando necessário para suporte.

O acesso aos registros deve seguir RBAC, menor privilégio, autenticação forte, trilha de auditoria, retenção limitada e segregação por ambiente.

---

## 2. Métricas

### Visão do Cliente (RED)

Para o Checkout, o painel deve seguir o método **RED**:

- **Rate**: taxa de requisições;
- **Errors**: taxa de erros;
- **Duration**: duração das requisições.

Métricas principais:

```text
checkout_requests_total
checkout_errors_total
checkout_request_duration_seconds
checkout_dependency_duration_seconds
checkout_timeouts_total
```

Como o incidente derruba conversão sem necessariamente gerar HTTP 500, também monitoraria:

```text
checkout_started_total
checkout_completed_total
checkout_abandoned_total
checkout_conversion_rate
```

O ponto crítico é **não confiar na média**. A média de 120 ms pode esconder uma minoria de requisições com vários segundos de duração. Por isso, medir percentis e histogramas:

```text
p50
p95
p99
p99.9
```

Exemplo:

```text
Média = 120 ms
p50 = 90 ms
p95 = 180 ms
p99 = 2.8 s
p99.9 = 5.2 s
```

A média parece saudável, mas o `p99` revela o problema.

### Visão dos Recursos (USE)

Para recursos, usar o método **USE**:

- **Utilization**;
- **Saturation**;
- **Errors**.

Monitoraria:

- CPU, memória e GC;
- threads/goroutines conforme a stack;
- filas internas;
- file descriptors;
- pool de conexões;
- conexões em uso/livres;
- tempo esperando conexão;
- locks e queries lentas no PostgreSQL;
- I/O e cache hit do banco;
- latência e timeout de Pagamento e Estoque;
- circuit breaker;
- rede e retransmissões.

Um sinal de saturação pode aparecer antes do primeiro 500:

```text
pool = 100/100
fila de espera crescendo
p99 aumentando
HTTP 500 = 0
```

Portanto, CPU baixa não significa sistema saudável.

### Labels

Labels devem ter **baixa cardinalidade**.

Permitidos:

```text
service="checkout"
environment="prod"
method="POST"
route="/checkout/{id}"
status_class="2xx"
dependency="pagamento"
result="success"
version="v2"
```

Proibidos como labels:

```text
user_id
order_id
request_id
trace_id
email
cpf
product_id
full_url
timestamp
error_message livre
```

Cada valor único gera uma nova série temporal. Alta cardinalidade aumenta memória, disco, custo de ingestão e tempo de consulta. IDs únicos devem ir para logs/traces, não para labels de métricas.

---

## 3. Traces e Amostragem

### A Escolha

- ( ) Head Sampling
- (**X**) Tail Sampling
- ( ) Combinação das duas

### Justificativa

Escolho **Tail Sampling** porque o problema importante só é conhecido depois que a requisição termina ou estoura timeout.

Com Head Sampling, a decisão de descartar pode ocorrer antes de sabermos que aquela requisição seria lenta.

Com Tail Sampling, a infraestrutura observa o trace completo antes de decidir se será retido.

Regra de retenção:

- manter 100% dos traces com erro;
- manter 100% dos traces com timeout;
- manter 100% dos traces acima de um limite de latência;
- manter 100% das falhas em Pagamento/Estoque;
- amostrar apenas 1% a 5% dos traces saudáveis.

```text
Trace terminou
    |
    v
Erro? ------ sim --> guardar
  | não
  v
Timeout? --- sim --> guardar
  | não
  v
Duração > 2s? sim -> guardar
  | não
  v
Guardar pequena amostra saudável
```

Trade-off: Tail Sampling exige buffering e collectors dimensionados para 5.000 TPS e alta disponibilidade.

---

## 4. Estratégia de Testes

### Qual a nossa estratégia para testar isso e garantir que funciona?

Antes da próxima Black Friday, precisamos provar que a observabilidade detecta o incidente.

#### 1. Testar logs estruturados

Validar automaticamente que logs possuem `trace_id`, `span_id`, `service`, `version`, `operation`, `duration_ms` e `outcome`, além de garantir que CPF, tokens e outros dados sensíveis não aparecem.

#### 2. Testar propagação de trace

Executar uma chamada que passe por Checkout, Pagamento e Estoque e validar que os três spans pertencem ao mesmo `trace_id`.

#### 3. Injetar latência em Pagamento

Simular exatamente o incidente:

```text
Pagamento normal: 100 ms
Pagamento degradado: 3 s
```

Sem gerar HTTP 500. Esperado:

```text
HTTP 500 -> continua baixo
p99 -> sobe
timeouts -> sobem
conversão -> cai
trace -> aponta Pagamento
alerta -> dispara
```

#### 4. Testar saturação

Simular pool esgotado, banco lento, fila crescendo, limite de threads e dependência lenta. O objetivo é confirmar alertas antes da falha explícita.

#### 5. Testar dashboards sob carga

Executar cargas progressivas:

```text
500 TPS
1.000 TPS
2.500 TPS
5.000 TPS
```

Validar Rate, Errors, p50, p95, p99, p99.9, timeouts, pool de conexões, dependências e conversão.

#### 6. Testar alertas

Disparar intencionalmente condições como:

```text
p99 > 2 s por 5 min
timeout rate > 1%
queda de conversão > limite
pool > 90% por 3 min
```

E verificar se o canal correto recebe o alerta.

#### 7. Testar Tail Sampling

Gerar traces saudáveis, lentos, com erro e com timeout. Validar que saudáveis ficam apenas em pequena amostra, enquanto erros, timeouts e lentos são preservados em 100%.

#### 8. Game Day / Chaos Engineering

Simular o incidente completo antes da Black Friday:

```text
5.000 TPS
   |
Checkout v2 Canary
   |
Pagamento com latência artificial
   |
sem HTTP 500
   |
clientes estouram timeout
```

A equipe deve diagnosticar usando apenas dashboards, alertas, logs e traces.

Medir:

- MTTD — Mean Time To Detect;
- MTTR — Mean Time To Recover.

---

## Resumo das Decisões

| Tema | Decisão |
|---|---|
| Logs | Estruturados em JSON |
| Correlação | `trace_id`, `span_id`, `request_id` |
| Propagação | W3C Trace Context |
| Dados pessoais | Não logar; pseudonimizar quando necessário |
| Acesso aos logs | RBAC + menor privilégio + auditoria |
| Métricas da aplicação | RED |
| Recursos | USE |
| Latência | Histogramas + p50/p95/p99/p99.9 |
| Métricas de negócio | Conversão e abandono do Checkout |
| Labels | Apenas baixa cardinalidade |
| IDs únicos | Logs/traces, nunca labels |
| Tracing | Distribuído entre Checkout, Pagamento e Estoque |
| Sampling | Tail Sampling |
| Retenção | 100% erros/timeouts/lentos + pequena amostra saudável |
| Testes | Falha injetada + carga + alertas + tracing |
| Validação final | Game Day / Chaos Engineering |

## Fluxo de Diagnóstico Esperado

```text
Alerta: Checkout p99 aumentou
        |
        v
Dashboard RED
        |
        +--> HTTP 500 normal
        +--> p99 = 3,5 s
        +--> timeout crescendo
        +--> conversão caindo
        |
        v
Trace lento (trace_id = ABC)
        |
        v
Checkout 3,4 s
        |
        +--> Estoque 40 ms
        |
        +--> Pagamento 3,2 s
                     |
                     v
              gargalo encontrado
```

Esse desenho resolve a limitação central do cenário: o sistema deixa de depender apenas de média de latência, CPU e HTTP 500 para afirmar que está saudável e passa a observar a experiência real do cliente, saturação de recursos e o caminho completo da requisição distribuída.
