# ADR-001 — Padrão Outbox no MySQL para disparo de webhooks

- **Status:** Aceito
- **Data:** 2026-07-16
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior/Plataforma), Bruno (Eng. Pleno/Pedidos)

## Contexto

Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação em
tempo real (definido como latência **abaixo de 10s** — [09:02] Marcos) quando o
status de seus pedidos muda. Hoje eles fazem polling em `GET /orders`.

A mudança de status já é uma operação transacional pesada: em
`src/modules/orders/order.service.ts`, o método `OrderService.changeStatus`
executa, dentro de um único `this.prisma.$transaction`, o `UPDATE` da order, o
`INSERT` em `OrderStatusHistory` e o débito/reposição de `stockQuantity` dos
produtos. Precisávamos decidir se o webhook seria disparado sincronamente dentro
desse fluxo ou desacoplado.

- [09:06] Diego — adotado o **padrão Outbox**: dentro da mesma transação SQL que
  atualiza `orders` e `order_status_history`, inserir uma linha em uma tabela
  `webhook_outbox` com o evento. Se a transação commita, o evento existe; se dá
  rollback, o evento some junto — sem inconsistência possível.
- [09:08] Larissa — a outbox fica no **MySQL já existente**, com índice em status
  e em `created_at`.
- [09:52] Larissa / Diego / Bruno — a outbox guarda o **payload renderizado
  (snapshot) no momento da inserção**, para o evento refletir o estado de quando
  o status mudou.
- [09:51] Larissa — o id da outbox é **UUID**, seguindo o padrão do projeto
  (todas as tabelas usam `@id @default(uuid()) @db.Char(36)` em
  `prisma/schema.prisma`).

## Decisão

Implementar o padrão **Transactional Outbox** sobre o MySQL existente. A escrita
do evento (`webhook_outbox`) ocorre **na mesma transação** de
`OrderService.changeStatus`, garantindo atomicidade entre a mudança de status e o
registro do evento. A linha da outbox armazena o **payload já renderizado
(snapshot)**, id UUID, e possui índice em status e `created_at` para leitura
eficiente pelo worker.

## Alternativas Consideradas

- **Disparo síncrono do HTTP dentro de `changeStatus`** — descartado em
  [09:04] Bruno / [09:06] Diego. Um HTTP call no meio da transação faria um
  cliente lento travar a mudança de status de outros pedidos, e não há rollback
  aceitável se o cliente estiver offline.
- **Fila externa (Redis Streams / Redis Cluster)** — descartado em
  [09:07] Larissa / Diego como overengineering para um time pequeno; exigiria
  subir e operar infraestrutura nova sem ganho proporcional.
- **Guardar apenas `order_id` e renderizar o payload no envio** — descartado em
  [09:52] Larissa; poderia refletir um estado posterior do pedido em vez do
  estado no instante da transição.

## Consequências

**Positivas**
- Atomicidade forte: nunca ocorre "status mudou mas evento não saiu" — a garantia
  vem do próprio commit/rollback da transação.
- Zero infraestrutura nova: reaproveita o MySQL e o `PrismaClient` já usados.
- Snapshot na inserção elimina a classe de bugs de "evento reflete estado errado".

**Negativas / trade-offs**
- Acopla a escrita do evento à transação de pedidos: uma falha ao inserir na
  outbox provoca **rollback da mudança de status** (aceito conscientemente —
  [09:40] Bruno / [09:41] Diego).
- Payload renderizado ocupa mais espaço por linha do que guardar só o `order_id`;
  mitigado pela política de arquivamento de linhas entregues (fora do escopo
  desta fase — [09:08] Diego).
- Polling da tabela adiciona carga de leitura recorrente ao MySQL (ver
  [ADR-005](ADR-005-worker-processo-separado-polling.md)).

## Referências

- Código: `src/modules/orders/order.service.ts` (`OrderService.changeStatus`),
  `prisma/schema.prisma` (`OrderStatusHistory`, convenção `uuid`/`Char(36)`).
- Relacionado: [ADR-005](ADR-005-worker-processo-separado-polling.md),
  [ADR-006](ADR-006-reuso-padroes-existentes.md).
