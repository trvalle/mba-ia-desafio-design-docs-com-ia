# ADR-005 — Worker em processo separado consumindo a outbox por polling

- **Status:** Aceito
- **Data:** 2026-07-16
- **Decisores:** Diego (Eng. Sênior/Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno/Pedidos), Marcos (PM)

## Contexto

Definido o padrão outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), faltava decidir
**como** os eventos pendentes são lidos e entregues, e **onde** esse consumidor
roda.

- [09:09] Diego — leitura por **polling em loop**: a cada 2 segundos, busca os
  eventos pendentes mais antigos, processa e marca.
- [09:10] Marcos / Larissa — 2s atende o requisito de "abaixo de 10s"; a latência
  mínima passa a ser 2s no pior caso, e isso é aceito.
- [09:11] Diego / Larissa — o worker deve rodar como **processo separado**, não
  dentro da instância da API; se a API reinicia, não se perde o worker. Prevê-se
  um entry-point novo `src/worker.ts` + script `npm run worker`, análogo ao
  `src/server.ts` existente.
- [09:11] Bruno / [09:30] Bruno — o worker usa uma **instância própria de
  `PrismaClient`** (mesmo banco/`DATABASE_URL`), pois `PrismaClient` é por
  processo.
- [09:12] Diego / [09:13] Larissa — por ora **single-worker**, com ordenação por
  `order_id` / `created_at` da outbox; sem garantia de ordering global.
- [09:28] Bruno / Diego — a lógica de processamento vive dentro do módulo
  (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`).

## Decisão

Executar um **worker dedicado em processo separado**, com entry-point
`src/worker.ts` e script `npm run worker`, que faz **polling da outbox a cada 2
segundos** buscando os pendentes mais antigos em batch pequeno. O worker abre sua
**própria instância de `PrismaClient`** apontando para o mesmo banco. Opera como
**single-worker**, com ordenação implícita por `order_id`/`created_at`; ordering
global **não** é garantido nesta fase.

## Alternativas Consideradas

- **Worker dentro da mesma instância/processo da API** — descartado em
  [09:11] Diego/Larissa: um restart da API derrubaria o worker e o processamento
  de eventos ficaria acoplado ao ciclo de vida do servidor HTTP.
- **Trigger de banco para notificação reativa** — descartado em [09:09] Diego:
  MySQL não tem `NOTIFY`/`LISTEN` como o Postgres; uma trigger só executa SQL e
  não avisa processo externo, exigindo gambiarra (arquivo/endpoint).
- **Múltiplos workers em paralelo (para ordering global / escala)** — adiado em
  [09:13] Diego/Larissa como "problema do futuro" (particionar por `order_id` ou
  lock pessimista quando necessário), pois quebraria a ordenação implícita do
  single-worker.

## Consequências

**Positivas**
- Desacoplamento do ciclo de vida da API: reiniciar o servidor HTTP não afeta a
  entrega de eventos.
- Polling de 2s é simples, sem infraestrutura de mensageria, e cumpre o SLA de
  <10s com folga.
- Single-worker garante ordenação por pedido sem coordenação distribuída.

**Negativas / trade-offs**
- **Latência mínima de 2s** por design (aceito — [09:10] Larissa).
- **Sem ordering global** e **sem escala horizontal** nesta fase; um único worker
  é ponto único de processamento (mitigável no futuro por particionamento/lock).
- Polling gera carga de leitura recorrente no MySQL, mesmo sem eventos.
- Um `PrismaClient` adicional consome conexões do banco — atenção ao
  dimensionamento do pool (API + worker no mesmo MySQL).

## Referências

- Espelha o padrão de entry-point já existente em `src/server.ts`; reaproveita
  `src/config/env.ts` e o logger Pino (`src/shared/logger/index.ts`) — ver
  [ADR-006](ADR-006-reuso-padroes-existentes.md).
- Relacionado: [ADR-001](ADR-001-outbox-no-mysql.md),
  [ADR-002](ADR-002-retry-backoff-e-dlq.md).
