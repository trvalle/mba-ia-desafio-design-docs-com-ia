# Tracker — Matriz de Rastreabilidade

Mapeia cada item identificável nos documentos da feature de webhooks à sua origem
(transcrição da reunião ou código do repositório). **Não é backlog nem quebra de
sprints.**

- **Fonte = TRANSCRICAO** → Localização = `[hh:mm] Nome`.
- **Fonte = CODIGO** → Localização = caminho real do arquivo.
- Itens sem origem rastreável direta estão marcados com ⚠ e detalhados na
  seção [Itens sem origem rastreável direta](#itens-sem-origem-rastreável-direta).

**Cobertura:** 79 itens mapeados · 74 com origem concreta (**94%**) · 66 linhas
`TRANSCRICAO` (**84%**) · 8 linhas `CODIGO` · 5 linhas ⚠ sinalizadas.

---

## Requisitos Funcionais (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (POST): URL + status; secret gerada por nós e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar (PATCH) e remover (DELETE) webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Assinar apenas status desejados; filtro aplicado na geração do evento | TRANSCRICAO | [09:33] Marcos / [09:34] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Consultar histórico de entregas (sucesso/falha, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Identificador único de evento (X-Event-Id) para dedup no cliente | TRANSCRICAO | [09:25] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Rotacionar secret via API; antiga válida por 24h (grace period) | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Replay de entrega em DLQ, restrito a ADMIN e auditado | TRANSCRICAO | [09:35] Diego / [09:36] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Evento gerado automaticamente quando o status do pedido muda | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Payload enxuto (sem `items`); detalhe via GET /orders/:id | TRANSCRICAO | [09:43] Diego |

## Requisitos Não Funcionais (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação < 10s no caso comum | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório: URL deve ser `https`; `http` recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB; acima disso, erro (não trunca) | TRANSCRICAO | [09:24] Diego / Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Autenticidade/integridade via HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Semântica at-least-once (cliente pode receber duplicado) | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Não degradar a transação de mudança de status | TRANSCRICAO | [09:04] Bruno |

## Objetivos / Métricas (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-MET-01 | docs/PRD.md | Métrica de Sucesso | Latência de notificação < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-MET-02 | docs/PRD.md | Métrica de Sucesso | Taxa de entrega bem-sucedida ≥ 99% dentro da janela de retry | ⚠ | ⚠ meta quantitativa definida no PRD |
| PRD-MET-03 | docs/PRD.md | Métrica de Sucesso | Redução do polling em GET /orders pelos clientes-piloto | TRANSCRICAO | [09:00] Marcos |

## Decisões (ADRs)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Transactional Outbox no MySQL, transação atômica, snapshot na inserção | TRANSCRICAO | [09:06] Diego / [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Retry backoff 1m/5m/30m/2h/12h, 5 tentativas, DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa / [09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | [09:20]–[09:22] Sofia |
| ADR-004 | docs/adrs/ADR-004-entrega-at-least-once-x-event-id.md | Decisão | Entrega at-least-once, dedup por X-Event-Id | TRANSCRICAO | [09:24]–[09:25] Diego (formalizada [09:26] Larissa) |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo separado, polling 2s, single-worker | TRANSCRICAO | [09:09]–[09:11] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso de AppError, Pino, error middleware, padrão de módulos, prefixo WEBHOOK_ | TRANSCRICAO | [09:27]–[09:30] Bruno / Larissa |
| ADR-DET-01 | docs/adrs/ADR-001-...md | Decisão | Id da outbox é UUID (padrão do projeto) | TRANSCRICAO | [09:51] Larissa |
| ADR-DET-02 | docs/adrs/ADR-001-...md | Decisão | Outbox guarda payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa / Diego / Bruno |
| ADR-DET-03 | docs/adrs/ADR-002-...md | Decisão | Replay manual via endpoint admin (recoloca como pendente) | TRANSCRICAO | [09:18] Diego / [09:35] Diego |
| ADR-DET-04 | docs/adrs/ADR-001-...md | Decisão | Filtro por status na inserção do outbox (não insere se ninguém assina) | TRANSCRICAO | [09:34] Bruno / Diego |
| ADR-DET-05 | docs/adrs/ADR-006-...md | Decisão | Integração via função pura publishWebhookEvent(tx, ...) | TRANSCRICAO | [09:41] Bruno / Diego |

## Alternativas Descartadas (RFC / ADRs)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Envio síncrono no changeStatus (trava status; sem rollback viável) | TRANSCRICAO | [09:04] Bruno / [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / Redis Cluster (overengineering p/ time pequeno) | TRANSCRICAO | [09:07] Larissa / Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco reativa (MySQL sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Exactly-once (exige coordenação dos dois lados) | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/adrs/ADR-002-...md | Alternativa Descartada | 3 tentativas de retry (agressivo demais) | TRANSCRICAO | [09:16] Bruno / Diego |
| RFC-ALT-06 | docs/adrs/ADR-002-...md | Alternativa Descartada | Retry indefinido (evento pendurado pra sempre) | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-07 | docs/adrs/ADR-002-...md | Alternativa Descartada | DLQ como flag `failed` na própria outbox | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-08 | docs/adrs/ADR-003-...md | Alternativa Descartada | Secret global da plataforma (vaza uma, vaza tudo) | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-09 | docs/adrs/ADR-006-...md | Alternativa Descartada | Injetar WebhookRepository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |

## Itens Fora de Escopo / Adiados (PRD / RFC)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | Notificação por email quando webhook falha (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Rate limiting de saída (observar e decidir depois) | TRANSCRICAO | [09:39] Diego / Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Dashboard/painel visual (projeto do time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Arquivamento de linhas entregues da outbox | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Escala multi-worker / ordering global | TRANSCRICAO | [09:13] Diego |

## Questões em Aberto (RFC)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-OQ-01 | docs/RFC.md | Questão em Aberto | customer_id no body vs. path (não vem do JWT) | TRANSCRICAO | [09:32] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em Aberto | Associação usuário autenticado ↔ customer_id / autorização por recurso | TRANSCRICAO | [09:32] Bruno / Marcos |
| RFC-OQ-03 | docs/RFC.md | Questão em Aberto | Rate limiting de saída (mesmo item que PRD-OOS-02) | TRANSCRICAO | [09:39] Diego / Larissa |
| RFC-OQ-04 | docs/RFC.md | Questão em Aberto | Armazenamento seguro da secret durante rotação (não pode ser só hash) | ⚠ | ⚠ ver nota — inferido, não falado |

## Riscos (PRD / RFC / FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente offline por longos períodos perde eventos | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não implementa dedup e processa duplicado | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Atraso compromete prazo comercial com a Atlas | TRANSCRICAO | [09:00] Marcos / [09:45] Marcos |
| PRD-RISK-04 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-05 | docs/RFC.md | Risco | Contenção de conexões MySQL (API + worker) | TRANSCRICAO | [09:29] Diego / [09:30] Bruno |
| RFC-RISK-06 | docs/RFC.md | Risco | Latência mínima de 2s por design | TRANSCRICAO | [09:10] Larissa |

## Contratos Públicos (FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CONTRATO-01 | docs/FDD.md | Contrato de API | POST /webhooks — criar (secret devolvida na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato de API | GET /webhooks — listar por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato de API | PATCH /webhooks/:id — editar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato de API | DELETE /webhooks/:id — remover | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato de API | POST /webhooks/:id/rotate-secret — rotação (grace 24h) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato de API | GET /webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato de API | POST /admin/webhooks/dead-letter/:id/replay — ADMIN | TRANSCRICAO | [09:35] Diego / [09:36] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato de API | Entrega outbound: headers X-Event-Id/X-Signature/X-Timestamp/X-Webhook-Id + payload | TRANSCRICAO | [09:44]–[09:45] Diego / Sofia |

## Matriz de Erros WEBHOOK_* (FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-ERR-01 | docs/FDD.md | Restrição / Erro | `WEBHOOK_NOT_FOUND` (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição / Erro | `WEBHOOK_INVALID_URL` (400, URL não-https) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição / Erro | `WEBHOOK_SECRET_REQUIRED` (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição / Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (422, > 64KB) | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-05 | docs/FDD.md | Restrição / Erro | `WEBHOOK_DELIVERY_TIMEOUT` (motivo interno, 10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-06 | docs/FDD.md | Restrição / Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404, replay) | ⚠ | ⚠ segue prefixo [09:29]; código específico é adição do FDD |
| FDD-ERR-07 | docs/FDD.md | Restrição / Erro | `WEBHOOK_INVALID_STATUS` (400, status fora do enum) | ⚠ | ⚠ segue prefixo [09:29]; código específico é adição do FDD |
| FDD-ERR-08 | docs/FDD.md | Restrição / Erro | `WEBHOOK_ALREADY_INACTIVE` (409) | ⚠ | ⚠ segue prefixo [09:29]; código específico é adição do FDD |

## Integração com o Sistema Existente (FDD → CODIGO)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-INT-01 | docs/FDD.md | Integração | Estender OrderService.changeStatus para chamar publishWebhookEvent(tx,...) dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reusar a máquina de estados (canTransition) para emitir só transições válidas | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Criar erros WEBHOOK_* estendendo AppError e as classes HTTP | CODIGO | src/shared/errors/http-errors.ts, src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware trata erros WEBHOOK_* sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Reusar authenticate + requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Wiring do módulo: DI manual e registro de router | CODIGO | src/app.ts, src/routes/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Worker reusa config de env e logger Pino | CODIGO | src/config/env.ts, src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração / Padrão | Módulo webhooks segue o padrão controller/service/repository/routes/schemas | CODIGO | src/modules/orders/ (padrão de referência) |

---

## Itens sem origem rastreável direta

Estes itens **não** têm um timestamp único na transcrição nem um caminho de código
que os origine. Estão sinalizados para você decidir **remover, ajustar ou manter
como decisão de documento**:

1. **PRD-MET-02** — a meta de **taxa de entrega ≥ 99%** foi introduzida no PRD como
   alvo quantitativo. A reunião definiu a política de retry/DLQ, mas **não cravou
   um número de SLA de entrega**. Sugestão: validar o 99% com Marcos/Larissa ou
   marcá-lo como "meta proposta, a confirmar".

2. **RFC-OQ-04 / (relacionado a ADR-003)** — a questão de **armazenamento seguro da
   secret durante a rotação** (o fato de a secret não poder ser guardada só como
   hash irreversível, pois precisa reassinar) foi **inferida no design**, não dita
   na reunião. O mais próximo na transcrição é a decisão de rotação com grace de
   24h ([09:21]–[09:22] Sofia) e o caso de vazamento em log ([09:22] Diego), mas o
   **problema de storage em si não foi discutido**. Sugestão: manter como questão
   em aberto explicitamente marcada como "levantada no design", não como decisão da
   reunião.

3. **FDD-ERR-06, FDD-ERR-07, FDD-ERR-08** (`WEBHOOK_DEAD_LETTER_NOT_FOUND`,
   `WEBHOOK_INVALID_STATUS`, `WEBHOOK_ALREADY_INACTIVE`) — seguem a **convenção de
   prefixo `WEBHOOK_`** decidida na reunião ([09:29] Larissa/Bruno), mas os
   **códigos específicos são complementos do FDD**, não enumerados na transcrição
   (lá só foram citados `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`,
   `WEBHOOK_SECRET_REQUIRED`). São adições legítimas de design; ficam marcados
   apenas para transparência de origem.

> Todos os demais 53 itens têm origem concreta (timestamp da transcrição ou
> caminho de arquivo real confirmado por leitura do repositório).
