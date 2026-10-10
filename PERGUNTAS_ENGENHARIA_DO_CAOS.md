# Aula 02 — APIs Resilientes
## Engenharia do Caos: Perguntas de Risco

**Cenário:** API de Checkout integrada a um parceiro externo de pagamentos, com suporte a 10.000 usuários simultâneos.

### 1. E se o parceiro processar a cobrança, mas ocorrer um timeout antes de retornar a resposta?

**Resposta do sistema:**

O pagamento será marcado com status `UNKNOWN`, pois não podemos assumir que a cobrança falhou.

A aplicação utilizará a `Idempotency-Key` e um processo de reconciliação para consultar o resultado da transação junto ao parceiro, evitando cobranças duplicadas.

**Padrões utilizados:** Idempotência, Timeout e Reconciliação.

### 2. E se o banco de dados ficar indisponível durante o processamento do pagamento?

**Resposta do sistema:**

A aplicação deverá impedir novas cobranças quando não conseguir registrar a operação com segurança.

Se o pagamento já tiver sido enviado ao parceiro, seu resultado será verificado posteriormente por um processo de reconciliação.

**Padrões utilizados:** Fail Fast, Persistência Transacional e Reconciliação.

### 3. E se 10.000 usuários tentarem realizar pagamentos simultaneamente e a API externa ficar sobrecarregada?

**Resposta do sistema:**

Utilizaremos Bulkhead para limitar a quantidade de chamadas simultâneas, RateLimiter para controlar a taxa de requisições e Circuit Breaker para interromper temporariamente chamadas quando o parceiro apresentar falhas.

Isso permitirá que nossa aplicação continue estável, mesmo diante da indisponibilidade do serviço externo.

**Padrões utilizados:** Bulkhead, RateLimiter e Circuit Breaker.

---

## Conclusão

A arquitetura deve priorizar três características:

- **Consistência:** impedir cobranças duplicadas.
- **Resiliência:** lidar com falhas de serviços externos.
- **Escalabilidade:** suportar alta concorrência sem comprometer a estabilidade.