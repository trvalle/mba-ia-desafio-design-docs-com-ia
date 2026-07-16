# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Aprovado para desenvolvimento |
| **Data** | 2026-07-16 |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Prazo alvo** | Fim de novembro (3 sprints) |
| **Documentos técnicos** | [RFC.md](RFC.md) · [FDD.md](FDD.md) · [ADRs](adrs/) |

---

## 1. Resumo e contexto da feature

Vamos oferecer aos clientes B2B a capacidade de **receber notificações
automáticas (webhooks) quando o status de seus pedidos muda** na nossa plataforma,
em vez de consultarem repetidamente a API. É uma feature de integração voltada a
clientes que operam volume e precisam reagir a mudanças de status (pago, enviado,
entregue etc.) sem intervenção manual.

## 2. Problema e motivação

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram
formalmente notificação em tempo real das mudanças de status de seus pedidos
([09:00] Marcos). Hoje eles fazem **polling recorrente em `GET /orders`** para
descobrir se algo mudou, o que torna a integração **lenta e cara** para eles.

Há **urgência comercial**: a Atlas sinalizou que, se a entrega não sair **até o fim
do trimestre**, pode migrar para um concorrente ([09:00] Marcos). Resolver isso
protege receita e melhora a experiência de integração dos clientes estratégicos.

## 3. Público-alvo e cenários de uso

**Público-alvo:** clientes B2B integradores (a começar por Atlas, MaxDistribuição e
Nova Cargo) que consomem nossa API e mantêm sistemas próprios reagindo a pedidos.

**Cenários de uso:**
- *Logística reativa:* o cliente quer ser avisado assim que um pedido vira
  `SHIPPED`/`DELIVERED` para acionar seu próprio fluxo, sem ficar consultando a API
  ([09:33] Marcos).
- *Conciliação financeira:* o cliente assina `PAID` para conciliar pagamentos.
- *Diagnóstico de integração:* o cliente consulta o histórico de entregas para
  entender falhas do lado dele ([09:34] Marcos).
- *Rotação de credenciais:* o cliente troca a secret periodicamente por segurança,
  sem downtime, graças ao período de convivência ([09:21] Sofia).

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Notificar quase em tempo real | Latência da notificação (mudança de status → entrega) | **< 10s** no caso comum ([09:02] Marcos) |
| Entrega confiável | Taxa de entrega bem-sucedida (sem cair em DLQ) | **≥ 99%** dos eventos entregues dentro da janela de retry |
| Reduzir polling dos clientes | Redução de chamadas a `GET /orders` pelos 3 clientes-piloto | Queda relevante após adoção (baseline medido no piloto) |
| Não regredir pedidos | Impacto na latência de `changeStatus` | Sem degradação perceptível na mudança de status |

## 5. Escopo

**Incluso:**
- CRUD de configuração de webhook (criar, listar, editar, remover).
- Assinatura por status (o cliente escolhe quais status ouvir).
- Entrega assinada (HMAC) com retry e DLQ.
- Histórico de entregas consultável pelo cliente.
- Rotação de secret com período de convivência.
- Endpoint administrativo para reprocessar entregas mortas (DLQ).

**Fora de escopo (adiado/descartado na reunião):**
- **Notificação por email ao cliente quando o webhook falha** — adiado para a
  próxima fase, após medir impacto ([09:37] Larissa).
- **Rate limiting de saída** (evitar bombardear o cliente com muitas chamadas) —
  "observar e decidir depois" ([09:39] Diego/Larissa).
- **Dashboard/painel visual** para o cliente — projeto separado do time de
  frontend; nesta fase só endpoints ([09:40] Larissa).
- **Arquivamento de linhas de eventos já entregues** — fora do escopo desta
  feature ([09:08] Diego).
- **Escala com múltiplos workers / ordering global** — adiado como "problema do
  futuro" ([09:13] Diego).

## 6. Requisitos funcionais

| # | Requisito | Origem |
|---|---|---|
| RF-01 | Cliente pode **cadastrar um webhook** (endpoint POST) informando URL e a lista de status desejados; a secret é gerada por nós e devolvida na criação | [09:31] Marcos |
| RF-02 | Cliente pode **editar (PATCH)** e **remover (DELETE)** um webhook | [09:33] Bruno |
| RF-03 | Cliente pode **listar os webhooks** de um customer (GET) | [09:33] Bruno |
| RF-04 | Cada webhook pode **assinar apenas os status que deseja receber** (ex.: só `SHIPPED` e `DELIVERED`); o filtro é aplicado ao gerar o evento | [09:33] Marcos / [09:34] Bruno |
| RF-05 | Cliente pode **consultar o histórico de entregas** de um webhook (sucesso/falha, payload, resposta, tempo de resposta) | [09:34] Marcos |
| RF-06 | Cada entrega inclui um **identificador único de evento (X-Event-Id)** para o cliente deduplicar | [09:25] Diego |
| RF-07 | Cliente pode **rotacionar a secret** via API; a secret antiga continua válida por **24h** durante a migração | [09:21] Sofia |
| RF-08 | Administrador pode **reprocessar (replay) uma entrega que caiu na DLQ**, restrito à role ADMIN e auditado | [09:35] Diego / [09:36] Sofia |
| RF-09 | O evento de mudança de status é **gerado automaticamente quando o status do pedido muda**, de forma consistente com a mudança | [09:40] Bruno |
| RF-10 | A entrega envia **payload enxuto** com os dados essenciais do pedido (sem a lista de itens); detalhes o cliente busca em `GET /orders/:id` | [09:43] Diego |

## 7. Requisitos não funcionais

| # | Requisito | Origem |
|---|---|---|
| RNF-01 | **Latência de notificação < 10s** no caso comum | [09:02] Marcos |
| RNF-02 | **TLS obrigatório**: a URL do webhook deve ser `https`; `http` é recusado | [09:23] Sofia |
| RNF-03 | **Limite de payload de 64KB**; acima disso, erro (não trunca) | [09:24] Diego/Larissa |
| RNF-04 | **Autenticidade e integridade** de cada entrega via assinatura (HMAC-SHA256) | [09:20] Sofia |
| RNF-05 | **Timeout de 10s** por tentativa de entrega | [09:42] Diego |
| RNF-06 | **Semântica at-least-once** — o cliente pode receber o mesmo evento mais de uma vez | [09:24] Diego |
| RNF-07 | A feature **não pode degradar** a transação de mudança de status dos pedidos | [09:04] Bruno |

## 8. Decisões e trade-offs principais

As decisões arquiteturais estão registradas nos ADRs (não repetidas aqui):

- Emissão desacoplada e atômica via outbox — [ADR-001](adrs/ADR-001-outbox-no-mysql.md)
- Confiabilidade por retry/backoff + DLQ — [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)
- Segurança por HMAC-SHA256 e secret por endpoint — [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- Entrega at-least-once com X-Event-Id (dedup no cliente) — [ADR-004](adrs/ADR-004-entrega-at-least-once-x-event-id.md)
- Worker em processo separado (polling 2s) — [ADR-005](adrs/ADR-005-worker-processo-separado-polling.md)
- Reuso dos padrões existentes do projeto — [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)

**Trade-off de produto central:** optamos por **at-least-once com dedup no cliente**
em vez de exactly-once. Isso transfere a responsabilidade de deduplicação ao
cliente ([09:25] Sofia), mitigada com **documentação destacada no portal do
desenvolvedor** ([09:26] Marcos). É o padrão de mercado (Stripe, GitHub).

## 9. Dependências

- **Clientes B2B:** cada cliente precisa expor um endpoint `https` e implementar
  verificação de assinatura e deduplicação por `X-Event-Id`.
- **Documentação:** portal do desenvolvedor atualizado por produto para orientar a
  integração ([09:40] Marcos).
- **Revisão de segurança:** revisão obrigatória de Sofia antes do deploy (HMAC e
  geração de secret), com ≥2 dias úteis reservados ([09:46] Sofia).
- **Técnicas:** infraestrutura existente (MySQL, API) — detalhes no
  [RFC](RFC.md)/[FDD](FDD.md).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Cliente offline por longos períodos deixa de receber eventos | Média | Alto (cliente perde notificações) | Retry com backoff em janela de ~15h + DLQ com replay manual ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)) |
| Cliente não implementa dedup e processa evento duplicado | Média | Médio (efeito colateral no sistema do cliente) | Documentação clara no portal + `X-Event-Id` estável ([09:26] Marcos) |
| Atraso na entrega compromete o prazo comercial com a Atlas | Média | Alto (risco de churn) | Escopo enxuto (só endpoints, sem dashboard/email), 3 sprints estimadas e prazo confirmado com o cliente ([09:47] Marcos) |
| Vazamento de secret pelo cliente | Baixa | Alto | Secret por endpoint (blast radius contido) + rotação com grace de 24h ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)) |

## 11. Critérios de aceitação

- [ ] Cliente consegue cadastrar, listar, editar e remover webhooks pela API.
- [ ] Cliente recebe notificação de mudança de status em < 10s no caso comum.
- [ ] Cliente só recebe os status que assinou.
- [ ] Cada entrega chega assinada e com identificador único de evento.
- [ ] Cliente consegue consultar o histórico de entregas de um webhook.
- [ ] Cliente consegue rotacionar a secret sem perder entregas durante a migração.
- [ ] Entregas que falham repetidamente vão para a DLQ e podem ser reprocessadas
      por um administrador.
- [ ] URLs `http` são recusadas no cadastro.
- [ ] A mudança de status dos pedidos continua funcionando sem degradação.

## 12. Estratégia de testes e validação

- **Testes automatizados** seguindo a suíte existente (`tests/`, Vitest):
  emissão atômica do evento na transação de status, filtro por status assinado,
  retry/backoff, movimentação para DLQ, replay admin (com checagem de role) e
  validação de URL `https`.
- **Validação de segurança:** revisão dedicada de Sofia sobre HMAC e geração de
  secret antes do deploy ([09:46] Sofia).
- **Piloto com os 3 clientes** (Atlas, MaxDistribuição, Nova Cargo): medir latência
  real de notificação, taxa de entrega bem-sucedida e redução de polling em
  `GET /orders` contra a baseline.
- **Critério de liberação:** metas da seção 4 atingidas no piloto e checklist de
  aceitação (seção 11) completo.
