# FDD — Sistema de Webhooks de Notificação de Pedidos

> **Feature Design Document** — especificação de implementação.
> Fonte de verdade das decisões: [RFC.md](RFC.md) e os ADRs em [`adrs/`](adrs/).
> As decisões arquiteturais já estão fechadas; este documento não as reabre.

| Campo | Valor |
|---|---|
| **Status** | Pronto para implementação |
| **Data** | 2026-07-16 |
| **Autor** | Larissa (Tech Lead) |
| **Módulo** | `src/modules/webhooks` |

---

## 1. Contexto e motivação técnica

Clientes B2B precisam ser notificados (latência **< 10s**) quando o status de seus
pedidos muda, substituindo o polling atual em `GET /orders`. A mudança de status
já é uma transação crítica em `OrderService.changeStatus` (update do pedido +
`OrderStatusHistory` + estoque). A feature adiciona a emissão de eventos **sem
acoplar disponibilidade externa a essa transação**, via padrão Outbox
([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) consumido por um worker separado
([ADR-005](adrs/ADR-005-worker-processo-separado-polling.md)).

## 2. Objetivos técnicos

- Emitir evento de mudança de status de forma **atômica** com a transação do pedido.
- Entregar via HTTP com **at-least-once**, retry/backoff e DLQ.
- Autenticar cada entrega com **HMAC-SHA256** e secret por endpoint.
- Expor CRUD de configuração, histórico de entregas e replay de DLQ.
- Reaproveitar os padrões existentes do projeto (módulos, `AppError`, Pino, error
  middleware, `requireRole`).

## 3. Escopo e exclusões

**No escopo:** tabela outbox, tabela DLQ, tabela de configuração de webhook,
tabela de deliveries; worker de entrega; HMAC + rotação de secret com grace de
24h; endpoints CRUD + deliveries + replay; filtro por status na inserção.

**Fora de escopo (fases futuras):** notificação por email quando webhook falha
([09:37]), rate limiting de saída ([09:39]), dashboard visual ([09:40]),
arquivamento de linhas entregues da outbox ([09:08]), escala multi-worker /
ordering global ([09:13]).

---

## 4. Fluxos detalhados

### 4.1 Criação do evento na outbox (com filtro por status)

Ocorre **dentro** da transação de `OrderService.changeStatus`, após o update do
pedido e o `OrderStatusHistory`:

1. Após persistir a transição `from → to`, chamar
   `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
2. A função consulta as **configurações de webhook ativas** do `customer_id` do
   pedido cujo array de status assinados **contém `toStatus`**.
3. **Filtro na inserção:** se nenhum webhook do cliente assina `toStatus`, **não
   insere nada** na outbox (economiza linhas — [09:34] Bruno/Diego).
4. Para cada webhook elegível, renderiza o **payload snapshot** e insere uma linha
   em `webhook_outbox` com: `id` (UUID), `webhookId`, `eventId` (UUID),
   `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`, `payload` (JSON
   snapshot), `createdAt`.
5. Se qualquer insert falhar, a transação inteira faz **rollback** — status não
   muda e evento não sai ([09:40] Bruno / [09:41] Diego).

### 4.2 Processamento pelo worker

Processo separado (`src/worker.ts`), loop de **polling a cada 2s**
([ADR-005](adrs/ADR-005-worker-processo-separado-polling.md)):

1. Seleciona um batch pequeno de linhas `status = PENDING` e
   `nextAttemptAt <= now()`, ordenado por `createdAt` asc (ordenação por pedido
   preservada em single-worker).
2. Marca a linha como `PROCESSING` (evita reprocessamento no próximo tick).
3. Monta os headers (incluindo assinatura HMAC — ver §7 / ADR-003) e faz `POST`
   para a `url` do webhook, com **timeout de 10s**.
4. Registra o resultado em `webhook_deliveries` (status, HTTP code, tempo de
   resposta, corpo truncado da resposta).
5. **Sucesso** (2xx): marca a outbox como `DELIVERED`.
6. **Falha** (não-2xx, timeout, erro de conexão): ver §4.3.

### 4.3 Retry e backoff

([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md))

- Incrementa `attempts`. Se `attempts < 5`, agenda o próximo retry definindo
  `nextAttemptAt = now() + backoff[attempts]`, com
  `backoff = [1m, 5m, 30m, 2h, 12h]`, e volta a linha para `PENDING`.
- Latência mínima do primeiro envio permanece ~2s (tick do worker).

### 4.4 DLQ (Dead Letter Queue)

- Após a **5ª falha**, move o evento para `webhook_dead_letter` (payload, último
  motivo/HTTP code, timestamp) e marca a outbox original como `FAILED`.
- Reprocessamento **manual** via `POST /admin/webhooks/dead-letter/:id/replay`
  (role `ADMIN`, auditado): recria a linha na outbox como `PENDING`,
  `attempts = 0`.

---

## 5. Contratos públicos

Base: `/api/v1`. Todos os endpoints exigem `Authorization: Bearer <jwt>`. O
`customer_id` **não** vem do JWT (ver Questões em aberto no RFC). Respostas de erro
seguem o formato do `errorMiddleware`: `{ "error": { "code", "message", "details"? } }`.

### 5.1 `POST /api/v1/webhooks` — criar webhook

Cria a configuração; a **secret é gerada por nós e devolvida apenas na criação**.

Request:
```json
{
  "customerId": "0b8f...uuid",
  "url": "https://client.example.com/hooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```
Response `201 Created`:
```json
{
  "id": "6a1c...uuid",
  "customerId": "0b8f...uuid",
  "url": "https://client.example.com/hooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9d3f...onlyShownOnce",
  "createdAt": "2026-07-16T12:00:00.000Z"
}
```
Status: `201` sucesso · `400` `WEBHOOK_INVALID_URL` / `VALIDATION_ERROR` · `401`.

### 5.2 `GET /api/v1/webhooks?customerId=...` — listar webhooks

Response `200 OK` (paginado, no formato `{ data, pagination }` já usado no projeto):
```json
{
  "data": [
    {
      "id": "6a1c...uuid",
      "customerId": "0b8f...uuid",
      "url": "https://client.example.com/hooks/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-16T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
> A `secret` **nunca** é retornada em GET/PATCH — só na criação e na rotação.

### 5.3 `PATCH /api/v1/webhooks/:id` — editar webhook

Request (campos parciais):
```json
{ "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```
Response `200 OK`: o webhook atualizado (sem `secret`).
Status: `200` · `400` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 5.4 `DELETE /api/v1/webhooks/:id` — remover webhook

Response `204 No Content`. Status: `204` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 5.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Gera nova secret; a antiga permanece válida por **24h** (grace period —
[ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).

Response `200 OK`:
```json
{
  "id": "6a1c...uuid",
  "secret": "whsec_new...onlyShownOnce",
  "previousSecretValidUntil": "2026-07-17T12:00:00.000Z"
}
```

### 5.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Últimas entregas do endpoint (sucesso/falha, payload, response, tempo).
Response `200 OK` (paginado):
```json
{
  "data": [
    {
      "id": "de11...uuid",
      "eventId": "e5a2...uuid",
      "status": "SUCCESS",
      "httpStatus": 200,
      "responseTimeMs": 143,
      "attempt": 1,
      "deliveredAt": "2026-07-16T12:00:03.100Z"
    },
    {
      "id": "de12...uuid",
      "eventId": "e5a3...uuid",
      "status": "FAILED",
      "httpStatus": 503,
      "responseTimeMs": 10000,
      "attempt": 5,
      "deliveredAt": "2026-07-16T12:31:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```
Status: `200` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ

**Requer role `ADMIN`** (`requireRole('ADMIN')`), auditado ([09:36] Sofia).
Recoloca o evento na outbox como pendente.
Response `202 Accepted`:
```json
{ "deadLetterId": "dl77...uuid", "outboxId": "ob90...uuid", "status": "REQUEUED" }
```
Status: `202` · `401` · `403` `FORBIDDEN` · `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 5.8 Contrato da entrega ao cliente (outbound `POST`)

Headers ([09:44]–[09:45]): `X-Event-Id`, `X-Signature` (HMAC-SHA256),
`X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`.
Payload (snapshot enxuto, sem `items` — [09:43] Diego):
```json
{
  "event_id": "e5a2...uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-16T12:00:01.000Z",
  "order_id": "or33...uuid",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "0b8f...uuid",
  "total_cents": 158900
}
```

---

## 6. Matriz de erros previstos

Todos estendem `AppError` e seguem o padrão de códigos `WEBHOOK_*`
([ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)). Formato de resposta pelo
`errorMiddleware` existente.

| Código | HTTP | Quando | Classe sugerida |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | webhook inexistente em GET/PATCH/DELETE/deliveries | estende `NotFoundError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | id de DLQ inexistente no replay | estende `NotFoundError` |
| `WEBHOOK_INVALID_URL` | 400 | URL não-`https` ou malformada | estende `ValidationError`/`BadRequestError` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | operação que exige secret sem secret disponível | estende `BadRequestError` |
| `WEBHOOK_INVALID_STATUS` | 400 | `subscribedStatuses` com valor fora do enum `OrderStatus` | estende `ValidationError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | payload renderizado > 64KB ([09:24]) | estende `UnprocessableEntityError` |
| `WEBHOOK_ALREADY_INACTIVE` | 409 | ativar/desativar em estado incompatível | estende `ConflictError` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | timeout de 10s no envio; não é resposta HTTP, é motivo de falha registrado em deliveries/DLQ | — |

> Erros de validação de schema (Zod) e violações de unicidade do Prisma continuam
> tratados pelo `errorMiddleware` existente (`VALIDATION_ERROR`, `CONFLICT`).

---

## 7. Estratégias de resiliência

- **Timeout HTTP:** 10s por tentativa ([09:42] Diego); ao estourar, conta como
  falha e agenda retry.
- **Retry/backoff:** 5 tentativas, `1m/5m/30m/2h/12h`
  ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)).
- **DLQ:** após esgotar, persiste em `webhook_dead_letter`; replay manual admin.
- **Atomicidade:** inserção na outbox dentro da transação de `changeStatus`;
  rollback conjunto ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
- **Idempotência:** `X-Event-Id` estável entre retries; cliente deduplica
  ([ADR-004](adrs/ADR-004-entrega-at-least-once-x-event-id.md)).
- **HMAC-SHA256** por endpoint + rotação com grace de 24h; `X-Timestamp` para
  detecção de replay pelo cliente
  ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).
- **Segurança de payload:** limite de 64KB → erro (`WEBHOOK_PAYLOAD_TOO_LARGE`).
- **Fallback:** não há fallback por email nesta fase (fora de escopo — [09:37]);
  o fallback efetivo é a DLQ + replay manual.

---

## 8. Observabilidade

Reusar o logger **Pino** já global (`src/shared/logger/index.ts`). Cada etapa loga
com `eventId`, `webhookId`, `orderId` e `attempt` para correlação
outbox→worker→entrega.

**Logs por etapa**
- *Inserção na outbox:* `info` — `webhook.outbox.enqueued { eventId, webhookId, orderId, toStatus }`; se filtrado, `debug` — `webhook.outbox.skipped_no_subscriber`.
- *Worker pega batch:* `debug` — `webhook.worker.batch { size }`.
- *Tentativa de envio:* `info` — `webhook.delivery.attempt { eventId, webhookId, attempt }`.
- *Sucesso:* `info` — `webhook.delivery.success { eventId, httpStatus, responseTimeMs }`.
- *Falha/retry:* `warn` — `webhook.delivery.retry { eventId, attempt, httpStatus, nextAttemptAt }`.
- *Movido p/ DLQ:* `error` — `webhook.deadletter.moved { eventId, webhookId, reason }`.
- *Replay admin:* `info` — `webhook.deadletter.replay { deadLetterId, actorUserId }` (auditoria — [09:36] Sofia).

**Métricas (exemplos concretos)**
- `webhook_outbox_pending_total` (gauge) — profundidade da fila pendente.
- `webhook_delivery_attempts_total{result="success|failure"}` (counter).
- `webhook_delivery_latency_ms` (histogram) — tempo de resposta do cliente.
- `webhook_deadletter_total` (counter) — eventos que caíram em DLQ.
- `webhook_worker_tick_duration_ms` (histogram) — duração do processamento do batch.

**Tracing:** propagar `eventId` como correlation id em todo o fluxo; opcionalmente
enviar `traceparent` no request outbound se/quando houver tracing distribuído
(não obrigatório nesta fase).

---

## 9. Dependências e compatibilidade

- **Runtime:** Node ≥ 20, ESM (imports `.js`), TypeScript — inalterado.
- **Banco:** MySQL existente; novas tabelas via migração Prisma
  (`webhook`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`).
  IDs UUID `Char(36)` seguindo o padrão do schema.
- **Libs já no projeto:** `@prisma/client`, `express`, `zod`, `jsonwebtoken`,
  `pino`. HMAC via `crypto` nativo do Node (sem dependência nova).
- **Processos:** novo entry-point `src/worker.ts` + script `npm run worker`;
  `PrismaClient` próprio no worker (mesma `DATABASE_URL`).
- **Compatibilidade:** feature aditiva; nenhum contrato existente muda. A única
  alteração de comportamento é a emissão de eventos dentro de `changeStatus`.

---

## 10. Critérios de aceite técnicos

- [ ] Mudar status de um pedido cujo cliente assina aquele status insere 1 linha
      em `webhook_outbox` **na mesma transação**; rollback da transação não deixa
      linha órfã.
- [ ] Status não assinado por nenhum webhook do cliente **não** gera linha na outbox.
- [ ] Worker entrega o evento em ≤ ~2s no caminho feliz; cliente recebe headers
      `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
- [ ] Assinatura HMAC-SHA256 do corpo verifica corretamente com a secret do endpoint.
- [ ] Cliente offline gera 5 tentativas com os intervalos `1m/5m/30m/2h/12h` e, ao
      esgotar, uma linha em `webhook_dead_letter`.
- [ ] Replay admin recoloca o evento como `PENDING` e exige role `ADMIN` (403 sem).
- [ ] Rotação de secret mantém a antiga válida por 24h e depois a invalida.
- [ ] `GET /webhooks/:id/deliveries` retorna histórico com status, httpStatus e tempo.
- [ ] Todos os erros do módulo retornam códigos `WEBHOOK_*` via `errorMiddleware`.
- [ ] Timeout de 10s no envio é tratado como falha e registrado.

---

## 11. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Rollback de `changeStatus` por falha na outbox amplia superfície de falha do método crítico | Insert simples e coberto por testes de transação; monitorar `webhook_outbox` |
| Contenção de conexões MySQL (API + worker) | Dimensionar pool; batch pequeno no worker |
| Sem ordering global / single-worker é ponto único | Documentado como limitação ([ADR-005]); caminho futuro por particionamento/lock |
| Armazenamento seguro da secret durante rotação (não pode ser só hash) | **Questão em aberto** (RFC); definir antes de codar a rotação; revisão de Sofia |
| Cliente não deduplica (at-least-once) | Documentação destacada no portal ([09:26] Marcos) |
| Payload > 64KB | Validação → `WEBHOOK_PAYLOAD_TOO_LARGE` |

---

## 12. Integração com o sistema existente

> Caminhos confirmados por leitura do repositório nesta sessão.

1. **`src/modules/orders/order.service.ts` — `OrderService.changeStatus`**
   Assinatura atual confirmada:
   `async changeStatus(id: string, input: UpdateOrderStatusInput, userId: string): Promise<OrderWithRelations>`,
   executada dentro de `this.prisma.$transaction(async (tx) => { ... })` que já faz
   `tx.order.update`, `tx.orderStatusHistory.create` e o débito/reposição de
   estoque. **Integração:** logo após o `orderStatusHistory.create`, dentro do
   mesmo `tx`, chamar `publishWebhookEvent(tx, order, from, to)` — função pura que
   recebe o `tx` client ([09:41] Bruno/Diego), sem injetar repository no
   `OrderService`. Se a função lançar, a transação existente já propaga o rollback.

2. **`src/modules/orders/order.status.ts`**
   Já expõe a máquina de estados (`canTransition`, `allowedTransitions`,
   `isTerminal`) e as regras `shouldDebitStock`/`shouldReplenishStock`.
   **Integração:** `publishWebhookEvent` usa o par `from → to` já validado por
   `canTransition` — o webhook só é emitido para transições legítimas, sem
   reimplementar a máquina de estados. O `toStatus` do evento é o mesmo enum
   `OrderStatus` usado nos `subscribedStatuses`.

3. **`src/shared/errors/http-errors.ts` + `src/shared/errors/app-error.ts` +
   `src/shared/errors/index.ts`**
   `app-error.ts` define a base `AppError` (`statusCode`, `errorCode`, `details`);
   `http-errors.ts` traz `NotFoundError`, `ValidationError`, `BadRequestError`,
   `ConflictError`, `UnprocessableEntityError` e exemplos de erros de domínio
   (`InsufficientStockError`, `InvalidStatusTransitionError`); `index.ts`
   reexporta todos. **Integração:** criar as classes `WEBHOOK_*` (ex.:
   `WebhookNotFoundError extends NotFoundError`,
   `WebhookInvalidUrlError extends ValidationError`) e reexportá-las em `index.ts`,
   exatamente como `InvalidStatusTransitionError` estende `ConflictError` hoje.

4. **`src/middlewares/error.middleware.ts`**
   Já serializa `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError`
   para `{ error: { code, message, details? } }`. **Integração:** **nenhuma
   alteração** — como os erros `WEBHOOK_*` estendem `AppError`, são tratados
   automaticamente ([09:29] Bruno).

5. **`src/middlewares/auth.middleware.ts`**
   Expõe `authenticate` e `requireRole(...roles)`. **Integração:** o router do
   módulo reusa `authenticate` em todo o CRUD e
   `requireRole('ADMIN')` no endpoint de replay de DLQ ([09:36] Sofia/Larissa),
   idêntico ao uso já existente em `src/modules/users/user.routes.ts`.

6. **`src/app.ts` (`buildControllers`) e `src/routes/index.ts`**
   DI manual e wiring de routers por domínio. **Integração:** instanciar
   `WebhookRepository → WebhookService → WebhookController` em `buildControllers` e
   registrar `router.use('/webhooks', buildWebhookRouter(...))` (e a rota admin)
   em `buildApiRouter`, seguindo o padrão dos demais módulos. O worker
   (`src/worker.ts`) reusa `src/config/env.ts` e `src/shared/logger/index.ts`.

---

## 13. Decisões relacionadas

- [RFC.md](RFC.md)
- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry/backoff e DLQ](adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once com X-Event-Id](adrs/ADR-004-entrega-at-least-once-x-event-id.md)
- [ADR-005 — Worker em processo separado (polling)](adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-padroes-existentes.md)
