# ADR-002 — Política de retry com backoff exponencial e Dead Letter Queue

- **Status:** Aceito
- **Data:** 2026-07-16
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior/Plataforma), Bruno (Eng. Pleno/Pedidos), Marcos (PM)

## Contexto

Os clientes que recebem webhooks podem estar temporariamente offline (inclusive
manutenções planejadas de horas). Precisávamos de uma política de reentrega que
cobrisse indisponibilidades reais sem deixar eventos pendurados para sempre.

- [09:16] Diego — houve caso real de cliente com **indisponibilidade de duas
  horas** em manutenção planejada; a política tem que cobrir essa janela.
- [09:17] Larissa — decidido: **5 tentativas** com backoff exponencial
  **1m / 5m / 30m / 2h / 12h** (~15h entre a primeira falha e a última tentativa).
- [09:17] Marcos — um cliente offline por 15h "já tem um problema sério dele";
  a janela é aceitável do ponto de vista de produto.
- [09:18] Diego / Larissa — esgotadas as tentativas, o evento vai para uma
  **DLQ em tabela separada** (`webhook_dead_letter`) contendo payload, motivo da
  falha e timestamp.
- [09:18] Diego / [09:35] Diego / [09:36] Sofia — reprocessamento **manual** via
  endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o
  evento na outbox como pendente, restrito a role `ADMIN` e com log de auditoria
  de quem executou.
- [09:42] Diego — o HTTP call do worker tem **timeout de 10s**; sem resposta é
  tratado como falha e agenda retry.

## Decisão

Adotar **backoff exponencial com 5 tentativas** (`1m, 5m, 30m, 2h, 12h`). Após a
quinta falha, mover o evento para a tabela **`webhook_dead_letter`** (payload,
motivo, timestamp). O reprocessamento é **manual**, via
`POST /admin/webhooks/dead-letter/:id/replay`, exigindo role `ADMIN` e registrando
auditoria. Falha de entrega inclui timeout de **10 segundos** sem resposta.

## Alternativas Consideradas

- **3 tentativas (mais agressivo)** — descartado em [09:16] Bruno / Diego: três
  retries em ~30 minutos matariam eventos de clientes com indisponibilidade
  matinal ou manutenção de horas.
- **Retry indefinido com backoff** — descartado em [09:15] Diego: mantém eventos
  pendurados para sempre quando o cliente simplesmente sumiu, sem ponto de corte.
- **DLQ como flag `failed` na própria outbox** (em vez de tabela separada) —
  descartado em [09:18] Diego em favor de tabela dedicada, que mantém a leitura da
  outbox principal limpa e serve de evidência para debug/reprocessamento.

## Consequências

**Positivas**
- Cobre indisponibilidades reais (até ~15h) sem intervenção humana.
- Ponto de corte determinístico evita acúmulo infinito de retries.
- DLQ separada dá observabilidade e um caminho de recuperação auditável.

**Negativas / trade-offs**
- Um evento pode levar até ~15h para ser entregue no pior caso — aceito
  explicitamente por produto ([09:17] Marcos).
- Reprocessamento manual exige operador com role `ADMIN`; não há auto-replay
  (decisão consciente para manter controle e auditoria).
- Mais uma tabela e um endpoint admin a manter e monitorar.

## Referências

- Reaproveita `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` para
  proteger o endpoint de replay (ver
  [ADR-006](ADR-006-reuso-padroes-existentes.md)).
- Relacionado: [ADR-001](ADR-001-outbox-no-mysql.md),
  [ADR-005](ADR-005-worker-processo-separado-polling.md).
