# ADR-004 — Entrega at-least-once com idempotência via X-Event-Id

- **Status:** Aceito
- **Data:** 2026-07-16
- **Decisores:** Diego (Eng. Sênior/Plataforma), Larissa (Tech Lead), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

Com outbox + retry (ver [ADR-001](ADR-001-outbox-no-mysql.md) e
[ADR-002](ADR-002-retry-backoff-e-dlq.md)), é possível que um mesmo evento seja
entregue mais de uma vez (por exemplo, o cliente recebe e processa, mas a
confirmação se perde e o worker retenta). Precisávamos definir a garantia de
entrega e como o cliente lida com duplicatas.

- [09:24] Diego — a garantia será **at-least-once**: o cliente pode receber o
  mesmo evento duas vezes e precisa estar preparado.
- [09:25] Diego — enviamos um **`X-Event-Id`** (UUID gerado quando o evento entra
  na outbox, único por evento); o cliente **dedupica pelo `event_id`** do lado
  dele.
- [09:25] Diego — é o padrão de mercado (Stripe, GitHub fazem assim); garantir
  exactly-once exigiria coordenação dos dois lados.
- [09:25] Sofia — reconhecido que isso **transfere responsabilidade ao cliente**.
- [09:26] Marcos — o comportamento será documentado com destaque no portal do
  desenvolvedor.

## Decisão

Garantir entrega **at-least-once**. Cada evento carrega um **`X-Event-Id`** (UUID,
gerado na inserção na outbox e estável entre retries), permitindo ao cliente
**deduplicar** por esse id. O contrato de duplicação possível é documentado
publicamente no portal do desenvolvedor.

## Alternativas Consideradas

- **Exactly-once** — descartado em [09:25] Diego: exigiria coordenação
  transacional entre nós e o cliente (confirmação/commit de dois lados),
  aumentando muito a complexidade para um ganho marginal frente a at-least-once +
  dedup por id.
- **Não enviar identificador de evento** (deixar o cliente inferir duplicatas por
  conteúdo) — implicitamente rejeitado em [09:25] Diego ao definir o `X-Event-Id`
  explícito, que torna a deduplicação trivial e confiável.

## Consequências

**Positivas**
- Simplicidade operacional: sem protocolo de commit distribuído.
- Alinhado a provedores de referência, reduzindo fricção para clientes que já
  integram webhooks de terceiros.
- O `X-Event-Id` estável entre retries torna a dedup do cliente determinística.

**Negativas / trade-offs**
- **Transfere a responsabilidade de deduplicação ao cliente** ([09:25] Sofia);
  clientes que não deduplicam podem processar o mesmo evento mais de uma vez.
- Exige documentação clara e destacada para os clientes ([09:26] Marcos).

## Referências

- O `X-Event-Id` (UUID) segue a convenção de identificadores UUID do projeto
  (`prisma/schema.prisma`, `@default(uuid())`) e é gerado na inserção da outbox
  (ver [ADR-001](ADR-001-outbox-no-mysql.md)).
- Enviado junto de `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e
  `Content-Type: application/json` ([09:44]–[09:45] Diego/Sofia).
