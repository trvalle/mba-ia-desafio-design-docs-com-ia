# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Título** | Webhooks outbound para notificação de mudança de status de pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-07-16 |
| **Revisores** | Diego (Eng. Sênior/Plataforma), Bruno (Eng. Pleno/Pedidos), Sofia (Eng. Segurança), Marcos (Product Manager) |
| **Prazo alvo** | Fim de novembro — estimativa de 3 sprints, incluindo revisão de segurança |

---

## Resumo executivo (TL;DR)

Clientes B2B precisam ser notificados em tempo real (latência **< 10s**) quando o
status de seus pedidos muda, em vez de fazer polling em `GET /orders`. Propomos um
sistema de **webhooks outbound** baseado no **padrão Transactional Outbox** sobre o
MySQL existente: a mudança de status grava o evento **na mesma transação** do
pedido, e um **worker em processo separado** faz polling da outbox a cada 2s e
entrega via HTTP. A confiabilidade vem de **retry com backoff exponencial (5
tentativas) + Dead Letter Queue**; a segurança, de **HMAC-SHA256 com secret por
endpoint** e TLS obrigatório; e a semântica de entrega é **at-least-once** com
deduplicação por `X-Event-Id` no lado do cliente. A solução **reaproveita os
padrões já existentes** do projeto (módulos em `src/modules/`, `AppError`, Pino,
error middleware) e não introduz infraestrutura nova.

---

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram
formalmente notificação em tempo real de mudanças de status de seus pedidos. Hoje
eles fazem polling recorrente em `GET /orders`, o que torna a integração **lenta e
cara** para eles. A Atlas indicou que pode migrar para um concorrente se a entrega
não sair até o fim do trimestre.

Para o produto, "tempo real" foi definido como **qualquer latência abaixo de 10
segundos** ([09:02] Marcos) — o requisito é não depender de atualização manual, e
não latência instantânea.

A restrição técnica central é que a mudança de status já é uma transação crítica e
pesada: em `src/modules/orders/order.service.ts`, `OrderService.changeStatus`
atualiza o pedido, insere em `OrderStatusHistory` e ajusta o estoque, tudo dentro
de uma única transação. Qualquer solução precisa preservar essa atomicidade sem
acoplar a latência/disponibilidade de sistemas externos ao fluxo de pedidos.

O escopo é estritamente **outbound** — só enviamos eventos aos clientes; eles não
enviam eventos para nós ([09:02]–[09:03] Sofia/Marcos).

---

## Proposta técnica (visão geral)

### 1. Transactional Outbox (MySQL)

Quando o status de um pedido muda, dentro da **mesma transação SQL** que atualiza
`orders` e `order_status_history`, gravamos o evento numa tabela `webhook_outbox`.
Se a transação commita, o evento existe; se dá rollback, o evento some junto —
eliminando inconsistência entre "status mudou" e "evento registrado". O evento é
persistido como **snapshot renderizado** no momento da inserção, para refletir o
estado do pedido naquele instante. Um **filtro por status** é aplicado **na
inserção**: se nenhum webhook do cliente assina aquele status, o evento nem entra
na outbox. Detalhes de schema e contrato ficam para o FDD.

### 2. Worker em processo separado (polling)

Um **worker dedicado**, rodando como processo separado da API (novo entry-point,
análogo ao `src/server.ts` atual), faz **polling da outbox a cada 2 segundos**,
lê os pendentes mais antigos em batch pequeno, entrega via HTTP e marca o
resultado. Roda em processo próprio para não ser derrubado por restarts da API, e
usa uma instância própria de `PrismaClient` sobre o **mesmo banco**. Nesta fase é
**single-worker**, o que preserva ordenação por pedido; ordering global não é
garantido.

### 3. Retry com backoff + Dead Letter Queue

Entregas que falham (incluindo timeout de HTTP) são reentregues com **backoff
exponencial**, num total de **5 tentativas** (1m / 5m / 30m / 2h / 12h,
~15h de janela total). Esgotadas as tentativas, o evento vai para uma tabela
**Dead Letter Queue** separada, com payload, motivo e timestamp. O reprocessamento
é **manual**, por endpoint administrativo restrito a role `ADMIN` e auditado.

### 4. Segurança

Cada request é assinado com **HMAC-SHA256** sobre o corpo, com **secret única por
endpoint** (não global), **rotacionável** com **grace period de 24h**. A URL do
webhook deve ser **`https`** (TLS obrigatório), validada no cadastro. A revisão de
segurança da implementação (HMAC e geração de secret) é pré-requisito de deploy.

### 5. Semântica de entrega

Garantimos **at-least-once**: um evento pode ser entregue mais de uma vez. Cada
evento carrega um identificador único (`X-Event-Id`) estável entre retries, e o
cliente deduplica por ele. É o padrão adotado por provedores de referência
(Stripe, GitHub).

### 6. Reuso dos padrões do projeto

O módulo `src/modules/webhooks` segue a mesma estrutura dos demais domínios
(`controller/service/repository/routes/schemas`), reutiliza a hierarquia de erros
`AppError`, o middleware de erro centralizado, o logger Pino e a autorização por
`requireRole`. A integração no fluxo de pedidos se dá por uma função que participa
da transação existente, sem reescrever `OrderService.changeStatus` nem introduzir
stack nova.

---

## Alternativas consideradas

- **Envio síncrono do HTTP dentro de `changeStatus`** — descartado
  ([09:04] Bruno / [09:06] Diego). *Trade-off:* um HTTP call no meio da transação
  faria um cliente lento travar a mudança de status de outros pedidos, e não há
  rollback aceitável se o cliente estiver offline. O acoplamento de
  disponibilidade externa ao fluxo de pedidos é inaceitável.

- **Fila externa com Redis Streams / Redis Cluster** — descartado
  ([09:07] Larissa / Diego). *Trade-off:* exigiria subir e operar infraestrutura
  nova para um time pequeno, sem ganho proporcional; o outbox sobre o MySQL já
  existente entrega a mesma garantia com custo operacional muito menor
  (overengineering).

- **Trigger de banco para consumo reativo (em vez de polling)** — descartado
  ([09:09] Diego). *Trade-off:* MySQL não tem `NOTIFY`/`LISTEN` como o Postgres;
  uma trigger só executa SQL e não avisa processo externo, exigindo gambiarra
  (arquivo/endpoint). Polling de 2s atende o requisito de <10s com folga.

- **Entrega exactly-once** — descartado ([09:25] Diego). *Trade-off:* exigiria
  coordenação transacional entre nós e o cliente, aumentando muito a complexidade;
  at-least-once + dedup por `X-Event-Id` resolve a grande maioria dos casos com
  fração do custo.

---

## Questões em aberto

- **`customer_id`: body vs. path** — decidiu-se que o `customer_id` **não** vem do
  JWT (que representa o usuário operador, não o cliente), mas ficou em aberto se
  será passado no body ou no path ([09:32] Larissa). Relacionado: a associação
  entre usuário autenticado e `customer_id`, e como garantir que um usuário só
  gerencie/leia webhooks do próprio cliente, ainda não foi definida
  ([09:32] Bruno/Marcos).

- **Armazenamento seguro da secret durante rotação** — decidiu-se HMAC com secret
  por endpoint e grace period de 24h ([09:22] Sofia), mas como a secret precisa
  ser reutilizável para reassinar, ela **não** pode ser guardada apenas como hash
  irreversível; o mecanismo de armazenamento e de convivência das duas secrets
  válidas na janela de rotação ainda precisa ser definido no design.

- **Rate limiting de saída** — se um cliente tiver muitos pedidos mudando de
  status em curto intervalo, podemos bombardeá-lo com muitas chamadas. Ficou como
  **"observar e decidir depois"** ([09:39] Diego / Larissa), fora do escopo desta
  fase.

---

## Impacto e riscos

**Impacto no sistema existente**
- Alteração pontual, porém **crítica**, em `OrderService.changeStatus`: a
  inserção na outbox passa a participar da transação de mudança de status. Uma
  falha ao gravar o evento provoca rollback da mudança de status (comportamento
  desejado, mas amplia a superfície de falha desse método já central).
- Novo processo operacional (worker) a implantar, monitorar e dimensionar.

**Riscos e mitigações**
- *Latência mínima de 2s por design* — aceito pelo produto ([09:10] Larissa),
  dentro do SLA de <10s.
- *Ausência de ordering global e de escala horizontal* nesta fase (single-worker)
  — documentado como limitação conhecida; caminho futuro por particionamento por
  `order_id` ou lock pessimista ([09:13] Diego).
- *Contenção de conexões no MySQL* — API e worker compartilham o mesmo banco com
  `PrismaClient` distintos; requer atenção ao dimensionamento do pool.
- *Responsabilidade de deduplicação transferida ao cliente* (at-least-once) —
  mitigado por documentação destacada no portal do desenvolvedor
  ([09:26] Marcos).
- *Segurança de HMAC/secret* — mitigado por revisão de segurança obrigatória antes
  do deploy, com ≥2 dias úteis reservados ([09:46] Sofia).

**Fora de escopo nesta fase** (registrado para fases futuras): notificação por
email ao cliente quando o webhook falha ([09:37] Larissa), rate limiting de saída
([09:39]), e dashboard/painel visual ([09:40] Larissa).

---

## Decisões relacionadas (ADRs)

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — Entrega at-least-once com X-Event-Id](adrs/ADR-004-entrega-at-least-once-x-event-id.md)
- [ADR-005 — Worker em processo separado (polling)](adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-padroes-existentes.md)
