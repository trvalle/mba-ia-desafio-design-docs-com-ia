# ADR-003 — Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

- **Status:** Aceito
- **Data:** 2026-07-16
- **Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead), Diego (Eng. Sênior/Plataforma), Bruno (Eng. Pleno/Pedidos)

## Contexto

Passaremos a enviar eventos com dados de pedidos para endpoints **fora da nossa
infraestrutura**. O cliente precisa validar que a requisição veio realmente de nós
e que o payload não foi adulterado em trânsito.

- [09:19] Sofia — o cliente tem que conseguir provar a origem e a integridade de
  cada requisição.
- [09:20] Sofia — padrão **HMAC**: assinamos o payload com uma secret
  compartilhada e enviamos a assinatura no header **`X-Signature`**; o cliente
  verifica do lado dele.
- [09:20] Sofia — algoritmo **SHA-256** (HMAC-SHA256), padrão de mercado com
  bibliotecas prontas em qualquer stack séria.
- [09:21] Sofia — **secret única por endpoint** (não global): se uma vaza, não
  compromete todos os clientes.
- [09:21] Sofia / [09:22] Sofia — a secret é **rotacionável** via API; ao
  rotacionar, a **secret antiga permanece válida por 24h em paralelo** (grace
  period) para o cliente migrar, e depois é revogada. Motivada por caso real de
  cliente que vazou secret em log de aplicação ([09:22] Diego).
- [09:23] Sofia — **TLS obrigatório**: a URL do webhook deve ser `https`; `http` é
  recusado por validação de schema.
- [09:44] Diego / Sofia — além de `X-Signature`, enviamos `X-Timestamp` (permite
  ao cliente detectar replay attack) e demais headers de identificação.

## Decisão

Assinar o corpo de cada request com **HMAC-SHA256**, enviando a assinatura em
`X-Signature` e o horário de envio em `X-Timestamp`. Cada endpoint de webhook tem
uma **secret própria**, **rotacionável** com **grace period de 24h** (secret
antiga e nova válidas simultaneamente durante a janela). A URL cadastrada deve ser
**`https`**, validada no schema Zod de criação/edição do webhook.

## Alternativas Consideradas

- **Secret global da plataforma** (uma única secret para todos os clientes) —
  descartado em [09:21] Sofia: o vazamento de uma secret comprometeria todos os
  endpoints de todos os clientes.
- **Sem grace period na rotação** (revogar a secret antiga imediatamente) —
  descartado implicitamente em [09:21] Sofia em favor da janela de 24h, para não
  quebrar a integração do cliente durante a troca.
- **mTLS / troca de certificados** — não levantado como necessário; HMAC-SHA256
  com secret compartilhada atende ao requisito de origem+integridade com menor
  fricção de integração ([09:20] Sofia).

## Consequências

**Positivas**
- Origem e integridade verificáveis pelo cliente com bibliotecas padrão.
- O blast radius de um vazamento fica contido a um único endpoint.
- Grace period de 24h permite rotação sem downtime da integração do cliente.
- `X-Timestamp` habilita detecção de replay no lado do cliente.

**Negativas / trade-offs**
- Precisamos **armazenar e gerenciar com segurança** as secrets por endpoint,
  incluindo o estado de duas secrets válidas durante a janela de rotação — como a
  secret precisa ser reutilizável para assinar, não pode ser guardada apenas como
  hash irreversível (**questão em aberto** a resolver no design de armazenamento).
- A verificação fica a cargo do cliente; má implementação do lado dele reduz a
  proteção efetiva.
- Lógica extra de rotação (dupla validade + expiração) aumenta a complexidade.

## Referências

- A validação de `https` segue o padrão de validação com Zod já usado no projeto
  via `validate(...)` (`src/middlewares/validate.middleware.ts`) e schemas por
  módulo (ver [ADR-006](ADR-006-reuso-padroes-existentes.md)).
- A revisão deste ADR pela Segurança (HMAC e geração de secret) é pré-requisito de
  deploy — reservar ≥2 dias úteis ([09:46] Sofia).
