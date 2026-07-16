# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-07-16
- **Decisores:** Bruno (Eng. Pleno/Pedidos), Larissa (Tech Lead), Diego (Eng. Sênior/Plataforma), Sofia (Eng. Segurança)

## Contexto

A codebase já tem convenções maduras e consistentes. Em vez de introduzir novos
padrões para a feature de webhooks, decidiu-se reaproveitar ao máximo o que
existe.

- [09:27] Bruno — cada domínio é um **módulo em `src/modules/`** com
  `controller`, `service`, `repository`, `routes` e `schemas`; webhook segue igual
  em `src/modules/webhooks`. (Confirmado no repo: `src/modules/orders/*`,
  `src/modules/products/*`, etc.)
- [09:28] Bruno — erros seguem o padrão existente: classe base `AppError` e
  subclasses específicas como `InsufficientStockError`,
  `InvalidStatusTransitionError`, com códigos em `SCREAMING_SNAKE_CASE`.
- [09:29] Bruno / Larissa — os códigos de erro do módulo usam **prefixo
  `WEBHOOK_`** (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`,
  `WEBHOOK_SECRET_REQUIRED`, ...).
- [09:29] Bruno — logger **Pino** já é global; o **middleware de erro
  centralizado** já trata `AppError`, `ZodError` e erros do Prisma, então pega os
  novos erros sem alteração.
- [09:30] Larissa — decisão de **reuso máximo**: `AppError`, Pino, error
  middleware, padrão de módulos, schemas Zod, padrão de códigos de erro.
- [09:36] Sofia / Larissa — o endpoint admin de replay reusa o **`requireRole`**
  existente para exigir role `ADMIN`.
- [09:40] Bruno / [09:41] Diego — a integração no `OrderService.changeStatus` se
  dá por uma **função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`**
  que recebe o `tx` da transação atual, evitando injetar um repository inteiro no
  `OrderService`.

## Decisão

O módulo `src/modules/webhooks` segue **estritamente os padrões existentes**:

- Estrutura de módulo `controller / service / repository / routes / schemas`,
  idêntica a `src/modules/orders/`.
- Erros estendendo `AppError` (`src/shared/errors/app-error.ts`) e as subclasses
  de `src/shared/errors/http-errors.ts`, com códigos prefixados por `WEBHOOK_`.
- Sem novo logging: reuso do Pino (`src/shared/logger/index.ts`) e do
  `errorMiddleware` (`src/middlewares/error.middleware.ts`), que já serializa
  `AppError`/`ZodError`/`Prisma`.
- Validação via `validate(...)` (`src/middlewares/validate.middleware.ts`) com
  schemas Zod por rota.
- Autorização via `authenticate` e `requireRole('ADMIN')`
  (`src/middlewares/auth.middleware.ts`) no endpoint de replay de DLQ.
- Integração com pedidos via função pura `publishWebhookEvent(tx, ...)` chamada de
  dentro do `this.prisma.$transaction` em
  `src/modules/orders/order.service.ts` (`OrderService.changeStatus`).

## Alternativas Consideradas

- **Injetar um `WebhookRepository` completo no `OrderService`** — preterido em
  [09:41] Diego em favor de uma **função pura recebendo o `tx`**, mais simples e
  sem acoplar o ciclo de vida de um repository ao serviço de pedidos.
- **Introduzir stack/abstrações novas** (novo logger, novo formato de erro,
  container de DI) — descartado em [09:30] Larissa: o projeto usa DI manual em
  `src/app.ts` (`buildControllers`) e já tem convenções suficientes; divergir
  aumentaria carga cognitiva e retrabalho.

## Consequências

**Positivas**
- Consistência com o restante da codebase; menor curva de aprendizado e revisão.
- O `errorMiddleware` já trata os novos erros — **zero alteração** na camada de
  erro.
- Função pura `publishWebhookEvent(tx, ...)` mantém o `OrderService` enxuto e
  respeita a atomicidade da transação (ver
  [ADR-001](ADR-001-outbox-no-mysql.md)).

**Negativas / trade-offs**
- Ficamos **presos às limitações dos padrões atuais** (ex.: DI manual em
  `src/app.ts` cresce a cada módulo; sem container automático).
- O `OrderService.changeStatus`, hoje responsável por order + history + estoque,
  passa a ter também a responsabilidade de publicar o evento — aumento de
  responsabilidade concentrado em um único método já crítico.

## Referências (arquivos reais do repositório)

- `src/modules/orders/{order.controller,order.service,order.repository,order.routes,order.schemas}.ts`
  — padrão de módulo replicado.
- `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` — base de
  erros e subclasses (`InsufficientStockError`, `InvalidStatusTransitionError`).
- `src/middlewares/error.middleware.ts` — tratamento centralizado de
  `AppError`/`ZodError`/`Prisma`.
- `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`.
- `src/middlewares/validate.middleware.ts` — `validate(...)` com Zod.
- `src/shared/logger/index.ts` — logger Pino global.
- `src/app.ts` — DI manual (`buildControllers`); `src/config/env.ts` — env via Zod.
