# Decisões arquiteturais — Padrão Saga

## ADR-001 — Orquestração e consistência eventual

**Decisão:** usar Saga orquestrada, com comandos/respostas assíncronos em broker e estado durável em Saga DB. Cada serviço tem banco próprio. **Motivo:** ordem de execução explícita, compensação determinística, observabilidade central e menor dispersão da lógica de negócio. **Trade-off:** coordenador adiciona componente crítico; mitigar com instâncias stateless, lock/lease por saga, versionamento otimista e banco persistente altamente disponível.

## ADR-002 — Modelo de dados e transições

Tabela `sagas`: `saga_id (PK)`, `order_id (UNIQUE)`, `state`, `current_step`, `version`, `deadline_at`, `retry_count`, `last_error`, `created_at`, `updated_at`.

Tabela `saga_steps`: `saga_id`, `step_id`, `status`, `command_message_id`, `business_idempotency_key`, `attempts`, `completed_at`; chave única `(saga_id, step_id)`.

Tabela `inbox`: `consumer_name`, `message_id`, `processed_at`; chave única `(consumer_name, message_id)`.

Tabela `outbox`: `event_id (PK)`, `aggregate_id`, `type`, `payload`, `published_at`, `attempts`.

Estados válidos: `STARTED -> ORDER_CREATED -> STOCK_RESERVED -> PAYMENT_AUTHORIZED -> CONFIRMING -> COMPLETED`. Em falha definitiva após reserva: `STOCK_RESERVED -> COMPENSATING -> COMPENSATED`; `MANUAL_REVIEW` se não houver recuperação automática segura. A Saga só é `COMPENSATED` após confirmações de `StockReleased` e `OrderCancelled`.

## ADR-003 — Matriz de decisão, idempotência e timeout

| Situação | Detecção | Decisão | Garantia de idempotência |
|---|---|---|---|
| `CreateOrder` duplicado | Mesmo `orderId` / `stepId` | Retornar pedido existente | Unique `orderId`, inbox transacional |
| `ReserveStock` duplicado | Mesmo `reservationId` | Não descontar novamente | Unique `reservationId`, atualização atômica de estoque |
| `AuthorizePayment` repetido | Mesmo `paymentKey` | Consultar/reusar resultado do gateway | Chave idempotente no gateway + registro local |
| `ReleaseStock` repetido | Reserva já `RELEASED` | Retornar `StockReleased` sem somar estoque | Transição condicional `RESERVED -> RELEASED` |
| `CancelOrder` repetido | Pedido já `CANCELLED` | Retornar `OrderCancelled` | Transição condicional de status |
| Resposta duplicada ou fora de ordem | `messageId` / `stepId` / estado | Ignorar ou registrar sem avançar indevidamente | Inbox + validação da máquina de estados |
| Timeout após comando de pagamento | Resposta ausente | Consultar gateway antes de retry/compensação | Mesma `paymentKey`; nunca assumir recusa |
| Broker indisponível | Outbox pendente | Retry de publicação | Outbox + deduplicação no destino |
| Pagamento recusado definitivamente | `PaymentDeclined` validado | `ReleaseStock`, depois `CancelOrder` | Chaves estáveis em compensações |
| Compensação sem resposta | Deadline expirado | Reconsultar estado e reenviar | Compensações idempotentes |
| Gateway capturou fundos | Status financeiro confirmado | Executar `RefundPayment` e reconciliar | Idempotency key para estorno; auditoria |

## ADR-004 — Política de erros e resiliência

**Retry:** usar backoff exponencial com jitter para erros transitórios, por exemplo 1 s, 2 s, 4 s, 8 s, limitado por tentativas e prazo de negócio. Recusa de cartão é falha de negócio definitiva, **não** passível de retry automático. Reenvios preservam as mesmas chaves de idempotência.

**Circuit Breaker:** no adaptador HTTP do gateway financeiro, abrir após falhas repetidas e permitir recuperação em half-open. Quando aberto, a Saga permanece em estado recuperável, sem inventar resultado do pagamento.

**Timeout:** comandos recebem deadline persistente. Um worker reavalia timeouts mesmo após reinício do orquestrador. Antes de compensar operações com resultado incerto, consultar o estado real no serviço/gateway. Se impossível, usar reconciliação e `MANUAL_REVIEW`.

**Dead-letter queue (DLQ):** encaminhar mensagens com falhas persistentes após retries limitados, preservando `messageId`, `sagaId`, motivo e histórico. Emitir alerta, disponibilizar inspeção, correção de causa-raiz e replay controlado com as mesmas chaves. **DLQ não equivale a compensação concluída**; Saga permanece pendente ou vai para revisão manual.

**Observabilidade:** `correlationId`, tracing distribuído, logs estruturados, métricas de duração da Saga, percentual de compensações, sagas travadas, tamanho da DLQ, retries e taxa de falha do gateway. Alarmes para Sagas acima do SLA.

## ADR-005 — Integridade, concorrência e segurança

- `ReserveStock` deve reservar atomicamente somente quantidade disponível, sem overselling. Reservas têm TTL de negócio; expiração automática deve ser coordenada com o estado da Saga.
- Consumidor grava inbox e mutação de domínio em uma transação local; outbox garante entrega eventual sem dual write inconsistente.
- Atualizações da Saga usam controle de versão/lease para impedir que dois workers executem transições incompatíveis.
- Não registrar dados sensíveis de cartão no broker; utilizar tokenização, autenticação serviço a serviço, criptografia em trânsito e acesso mínimo.
- Não prometer exactly-once no transporte: entrega at-least-once + processamento idempotente.
- `AuthorizePayment` representa autorização, não necessariamente captura; se o gateway fizer captura, prever `RefundPayment` e eventual conciliação financeira.

## Riscos mitigados

| Risco | Mitigação |
|---|---|
| Estoque reservado com pagamento recusado | `ReleaseStock` antes de `CancelOrder` |
| Pedido confirmado sem pagamento | Confirmar somente após `PaymentAuthorized` |
| Cobrança duplicada | `paymentKey` estável + consulta de status |
| Perda de evento entre banco e broker | Transactional Outbox |
| Duplicação e entrega fora de ordem | Inbox + máquina de estados + versionamento |
| Falha persistente na compensação | Retry + DLQ + alerta + reconciliação |
| Indisponibilidade do coordenador | Persistência + workers redundantes + retomada |

## Exemplo de percurso auditável

`SAGA-001`: `STARTED` → `ORDER_CREATED` → `STOCK_RESERVED` → `PaymentDeclined` → `COMPENSATING` → `StockReleased` → `OrderCancelled` → `COMPENSATED`.

O pedido termina `CANCELLED`, a reserva `RELEASED` e nenhuma cobrança deve permanecer confirmada. Para falhas ambíguas do gateway, reconciliar o resultado financeiro antes de encerrar a Saga.
