# Design Docs com IA — Sistema de Webhooks de Notificação de Pedidos

Este repositório é a entrega do desafio de **produção de documentação técnica
assistida por IA**. A partir de uma base de código existente (uma API REST de
gestão de pedidos) e da transcrição de uma reunião técnica, foi produzido um
conjunto documental completo para a feature de **webhooks outbound de notificação
de mudança de status de pedidos**.

---

## Sobre o desafio

O ponto de partida é um cenário realista: um time acabou de decidir, numa reunião
de ~55 minutos, como vai construir um sistema de webhooks para avisar clientes B2B
quando o status dos pedidos deles muda. Essa reunião foi transcrita
([`TRANSCRICAO.md`](TRANSCRICAO.md)), e existe uma codebase real com padrões já
estabelecidos (módulos em `src/modules/`, tratamento de erro centralizado, uma
transação crítica de mudança de status). O objetivo do desafio é transformar essa
matéria-prima — conversa informal + código — em documentação de engenharia
rastreável e acionável.

O trabalho não foi "pedir para a IA escrever documentos", e sim conduzir a IA por
um processo: primeiro entender o código e a reunião, depois destilar decisões e
descartes com rastreabilidade (quem decidiu o quê e quando), e só então escrever
cada documento no nível de abstração certo — de decisões atômicas (ADRs) até visão
de produto (PRD), sempre citando a origem de cada afirmação (timestamp da reunião
ou caminho de arquivo real). A ênfase esteve em **não inventar**: quando um item
não tinha origem rastreável, isso foi sinalizado em vez de mascarado.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
|---|---|
| **Claude Code (Opus 4.8)** | Agente principal. Explorou o repositório, leu a transcrição, destilou decisões/descartes e escreveu todos os documentos. |
| **Ferramentas de leitura/busca do agente** (Read, Glob, Grep, Bash `ls`) | Confirmação de fatos no código real — estrutura de módulos, hierarquia de erros, e a assinatura atual de `OrderService.changeStatus` — antes de escrever qualquer seção de integração. |
| **Ferramenta de escrita/edição** (Write, Edit) | Materialização dos documentos em `docs/` e ajustes pontuais de correção. |

> Nenhuma decisão de arquitetura foi criada pela IA: todas vêm da reunião. O papel
> da IA foi organizar, estruturar e rastrear — com validação humana a cada etapa.

---

## Workflow adotado

A produção seguiu uma ordem deliberada, em **prompts sequenciais e validados** — a
saída de cada etapa foi conferida antes de seguir para a próxima, e o
encadeamento foi do concreto (código/decisões) para o abstrato (produto) e, por
fim, para a rastreabilidade transversal:

1. **Exploração inicial** — leitura do repositório (estrutura, `package.json`,
   schema Prisma, módulos, máquina de estados de pedidos) e da transcrição
   completa. Saída: 4 listas — padrões de código, decisões fechadas, itens
   descartados/adiados e pontos ambíguos — **sem gerar nenhum documento ainda**.
2. **ADRs** ([`docs/adrs/`](docs/adrs/)) — 6 decisões arquiteturais, uma por
   arquivo, em formato MADR, cada uma citando o timestamp da decisão e a
   alternativa descartada correspondente.
3. **RFC** ([`docs/RFC.md`](docs/RFC.md)) — a proposta técnica em nível de
   arquitetura, submetida à equipe, linkando os ADRs.
4. **FDD** ([`docs/FDD.md`](docs/FDD.md)) — a especificação de implementação
   (fluxos, contratos, matriz de erros, integração com o código real).
5. **PRD** ([`docs/PRD.md`](docs/PRD.md)) — a visão de produto/negócio
   (problema, público-alvo, requisitos, métricas), acima do detalhe técnico.
6. **Tracker** ([`docs/TRACKER.md`](docs/TRACKER.md)) — a matriz de
   rastreabilidade que amarra cada item de cada documento à sua origem.
7. **README** (este arquivo) — a documentação do processo.

> A ordem ADR→RFC→FDD→PRD foi intencional: fixar primeiro as decisões atômicas e a
> arquitetura evita que o PRD "invente" requisitos que a engenharia não sustenta.
> O PRD, escrito por último entre os documentos de conteúdo, pôde então referenciar
> decisões já consolidadas em vez de antecipá-las.

---

## Prompts customizados

Dois exemplos reais de prompts usados nesta sessão.

**1. Contextualização inicial (exploração antes de qualquer escrita):**

```
Leia o repositório completo: estrutura de pastas, package.json, schema do Prisma,
módulos em src/ (auth, users, clients, products, orders), e a máquina de estados
de pedidos em order.service.ts (ou equivalente).

Depois leia TRANSCRICAO.md por completo.

Não gere nenhum documento ainda. Apenas me devolva:
1. Lista dos principais padrões de código já existentes
2. Lista de decisões técnicas fechadas na reunião (com timestamp de cada uma)
3. Lista de itens explicitamente descartados ou adiados na reunião (com timestamp)
4. Pontos da transcrição que ficaram ambíguos ou não decididos

Cite timestamps no formato [hh:mm] Nome para cada item das listas 2, 3 e 4.
```

**2. Geração do FDD com exigência de confirmar caminhos reais antes de escrever:**

```
Gere docs/FDD.md ... acionável o suficiente para um desenvolvedor começar a codar.
[...]
SEÇÃO OBRIGATÓRIA "Integração com o sistema existente": antes de escrever esta
seção, rode uma busca real no código (liste os arquivos de src/modules/orders/ e
src/shared/errors/ novamente, e confirme a assinatura atual de
OrderService.changeStatus) e só então nomeie pelo menos 4 caminhos de arquivo
REAIS, descrevendo como o módulo de webhooks se integra com cada um.

Use as decisões já fechadas nos ADRs e no RFC como fonte de verdade — não reabra
decisões já tomadas.
```

> Padrão comum aos dois: **restringir a IA a fatos verificáveis** (timestamps reais,
> caminhos reais) e **separar exploração de escrita**, para reduzir alucinação.

---

## Iterações e ajustes

O processo teve correções reais — momentos em que a saída da IA foi verificada,
apontada como imprecisa e ajustada antes de prosseguir:

- **Separação de decisões fechadas × descartadas na exploração inicial.** Na
  primeira versão das listas, alguns itens descartados ficaram misturados na lista
  de decisões fechadas (ex.: "síncrono descartado; adotado outbox" aparecia nas
  duas). Foi pedida a reescrita separando estritamente **o que FOI decidido**
  (Lista 2) de **o que NÃO entra / fica pra depois** (Lista 3). No mesmo passo, um
  timestamp fabricado (`[09:56 tabela]`) foi corrigido — a reunião termina em
  [09:53], e o item foi remapeado para os trechos reais ([09:21] e [09:33]–[09:34]).

- **Verificação de `docs/` antes de gerar.** Antes de escrever qualquer documento,
  foi confirmado por `ls docs/` que os arquivos existentes eram apenas *stubs*
  vazios — evitando sobrescrever conteúdo real sem querer.

- **Confirmação de código real antes da seção de integração do FDD.** Em vez de
  descrever a integração de memória, a assinatura atual de
  `OrderService.changeStatus` e a lista de `src/modules/orders/` e
  `src/shared/errors/` foram relidas do repositório, e só então os caminhos foram
  citados no FDD.

- **Tracker sinalizando itens sem origem rastreável, com correção no PRD.** Ao
  montar a matriz de rastreabilidade, 3 itens foram identificados como **sem origem
  concreta na reunião**: (1) a meta de **≥ 99% de taxa de entrega**, (2) o
  **armazenamento seguro da secret durante a rotação**, e (3) três **códigos de erro
  `WEBHOOK_*` complementares** (`WEBHOOK_DEAD_LETTER_NOT_FOUND`,
  `WEBHOOK_INVALID_STATUS`, `WEBHOOK_ALREADY_INACTIVE`) que seguem a convenção de
  prefixo decidida, mas não foram enumerados na transcrição. Em vez de forjar uma
  origem, esses itens ficaram marcados com ⚠ no Tracker. Como consequência, o
  **PRD foi ajustado**: a meta de 99% passou a ser rotulada explicitamente como
  _"meta proposta — a confirmar com produto; não foi cravada na reunião"_, para o
  documento não afirmar como decidido algo que foi apenas proposto.

---

## Como navegar a entrega

**Ordem de leitura sugerida** (do "porquê" ao "como" e à rastreabilidade):

1. **[README.md](README.md)** — você está aqui: o processo.
2. **[docs/PRD.md](docs/PRD.md)** — o problema, o público e os requisitos (produto).
3. **[docs/RFC.md](docs/RFC.md)** — a proposta técnica em nível de arquitetura.
4. **[docs/adrs/](docs/adrs/)** — as 6 decisões arquiteturais, uma por arquivo.
5. **[docs/FDD.md](docs/FDD.md)** — a especificação de implementação (para codar).
6. **[docs/TRACKER.md](docs/TRACKER.md)** — a matriz que amarra cada item à origem.

**Mapa de arquivos:**

```
docs/
├── PRD.md                                  Product Requirements Document
├── RFC.md                                  Request for Comments (proposta técnica)
├── FDD.md                                  Feature Design Document (implementação)
├── TRACKER.md                              Matriz de rastreabilidade
└── adrs/
    ├── ADR-001-outbox-no-mysql.md
    ├── ADR-002-retry-backoff-e-dlq.md
    ├── ADR-003-hmac-sha256-secret-por-endpoint.md
    ├── ADR-004-entrega-at-least-once-x-event-id.md
    ├── ADR-005-worker-processo-separado-polling.md
    └── ADR-006-reuso-padroes-existentes.md

TRANSCRICAO.md                              Transcrição da reunião (matéria-prima)
DESAFIO_DESIGN_DOCS.md                      Enunciado original do desafio
```

---

## Enunciado original

O enunciado completo do desafio está em
[`DESAFIO_DESIGN_DOCS.md`](DESAFIO_DESIGN_DOCS.md), e a transcrição da reunião que
serviu de base em [`TRANSCRICAO.md`](TRANSCRICAO.md).
