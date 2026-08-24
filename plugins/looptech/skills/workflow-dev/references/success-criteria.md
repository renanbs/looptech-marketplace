# Critérios de Sucesso Obrigatórios

Imposto na **entrada** do dual brainstorm e materializado como `/goal`
(`goals.md`) antes de qualquer impl. Propagado a todo handoff
(`subagent-handoff.md`). Nenhuma fase avança sem isto definido primeiro.

## Regra

- **Brainstorm primeiro.** `success_criteria` (produto) e
  `code_success_criteria` (construção) nascem na Fase 2
  (`brainstorm.md`), **antes** da spec. Nada é escrito
  retroativamente para justificar código já feito.
- **Fase 3 promove critério → `/goal`.** Cada critério ganha `id`
  `G-`, `done_when`, `evidence` e, ao fechar as tasks, `owner_task`.
  Spec sem essa promoção está incompleta.
- **Cada task herda os `/goal` que ela dona.** `done_when` da task é
  a conjunção dos `done_when` dos seus `G-` — não uma reafirmação
  vaga do objetivo geral.
- **O `/goal` é a condição de parada do ReAct**
  (`autonomy-react-loop.md`). O subagente não inventa outro critério
  em tempo de execução.
- **Teste é a evidência** (`verification.md`). Compilar não fecha
  `/goal`.

## O que torna um critério / `/goal` válido

Binário (sim/não), sem julgamento subjetivo:

- **Válido:** "POST /orders retorna 201 com `id` e o GET seguinte
  devolve o mesmo corpo; `go test ./internal/orders -count=1` passa."
- **Válido:** "Playwright `e2e/checkout.spec.ts` passa, sem skip."
- **Inválido:** "a funcionalidade está funcionando bem."
- **Inválido:** "o código ficou mais limpo."

## Onde isso se conecta

- **2a/2b** — origem dos critérios.
- **`Goals - <Título>`** — contrato que o orquestrador cola.
- **Handoff** — Objetivo Final = `/goal` da task, não um ponteiro.
- **`/plan` da task** — serve o `/goal`; não o encolhe.
- **Review de correção** — diff contra o `done_when`, inclusive
  presença da matriz de testes / Playwright.
- **Review de segurança** — mesmo diff contra `security-review.md`.
- **Fase 7 / PR** — lista G1…Gn com evidência rodada.
