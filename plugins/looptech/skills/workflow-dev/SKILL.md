---
name: workflow-dev
description: Use ao iniciar QUALQUER tarefa de desenvolvimento (feature, bugfix, hotfix, refactor) em um projeto que declara um Project Profile no CLAUDE.md ou AGENTS.md. Orquestra o ciclo completo — discover → dual brainstorm → /goals → spec+plan → env → /plan por task → impl async multi-agente → review → expert-security → testes (unit+e2e/Playwright) → PR. Spawna agentes nomeados (plan, expert-backend-go/python, expert-frontend-react/vue/pwa/web, expert-database, review, expert-security) com a classe de modelo do Profile (reasoning/code/critique/security), nunca slug de marca. NÃO use para projetos sem Project Profile.
---

# Workflow Dev — Orquestrador Agnóstico de Desenvolvimento

## Overview

Ciclo para qualquer projeto com **Project Profile** no `CLAUDE.md` e/ou
`AGENTS.md`: **discover → dual brainstorm → /goals → spec+plan → ambiente →
/plan por task → impl async → review → security → testes → PR**. Processo e
disciplina aqui; fatos (paths, comandos, branches, IDs de modelo) vêm do
Profile. Hosts: Claude Code, Codex, Cursor — tools em
[`references/host-compat.md`](references/host-compat.md), papéis em
[`references/agent-roles.md`](references/agent-roles.md).

**Announce at start:** "Estou usando a skill workflow-dev para guiar esta tarefa."

**Escopo:** só sub-projetos mapeados no Profile. Fora disso, ou sem Profile,
**pare** e diga que esta skill não cobre o caso.

**Princípios:**
- Branch base atualizada, isolamento por sub-projeto, testes como prova,
  entrada via PR.
- Specs/plans/**brainstorm**/**goals** vivem na **Regra de Destino** — nunca
  no código do sub-projeto.
- Cerimônia pela lane (`references/lanes.md`). Review de correção **e**
  segurança antes de todo commit.
- **Documento não implementa.** O que faz o agente ir até o fim é o `/goal`
  colado no handoff (`references/goals.md`).
- **Uma task por vez, com fatias independentes, é erro.** Fan-out é o
  padrão (`references/async-dispatch.md`).

---

## Fase 0 — Resolver o Project Profile · main

Detecte o host e leia o Profile em `CLAUDE.md` **e** `AGENTS.md`. Monte
`path → stack → comandos → convenções` antes de tocar código. Contrato:
[`references/project-profile.md`](references/project-profile.md).

- Path ausente: infira pelo manifesto e **avise**. Sem Profile: pare.
- Todo comando de teste/lint/build/e2e vem do Profile.
- Resolva papéis (`agent-roles.md`) e anuncie
  `plan/impl/review/security → id`.
- Destino de spec/memória: bloco `memory:` → vault +
  `memory-graph:memory-vault`; senão `specs_dir`; senão `.specs/`.

---

## DELEGATION MANDATE — puro orquestrador

O agente principal **não** analisa, implementa nem ajusta código — delega.

| Ação | Quem |
|------|------|
| Mapa / blast radius / spec+plan / goals | `plan` |
| Brainstorm de produto (2a) | orquestrador (precisa da conversa) |
| Brainstorm de código + modelos UX | `plan` + expert de UX (sem código de produto) |
| Impl e ajuste pós-review | expert da stack |
| Review de correção | `review` |
| Review de segurança | `expert-security` |
| Edição trivial ≤ 100 caracteres | orquestrador pode |
| Coordenação, git, gate de teste do Profile | orquestrador executa; falha → expert |

Lane S também vai para subagente. Loop pós-review: `CHANGES-REQUESTED` ou
`ISSUES-FOUND` → novo `impl` → os dois reviews de novo. Orquestrador nunca
aplica o fix.

---

## Lanes, papéis, handoff

Sizing: [`references/lanes.md`](references/lanes.md). Papéis por **classe**,
nunca slug: [`references/agent-roles.md`](references/agent-roles.md).
Orquestrador fica no `coord` da sessão.

Todo spawn segue [`references/subagent-handoff.md`](references/subagent-handoff.md):
Objetivo Final = `/goal` colado, Estado Atual colado, Variáveis, 5–15 linhas
do expert. `impl` embute
[`references/autonomy-react-loop.md`](references/autonomy-react-loop.md) e
**começa** com [`references/plan-before-impl.md`](references/plan-before-impl.md).

---

## /goal e critérios de sucesso

Nenhuma spec, plan ou handoff começa sem `/goal` testável. Disciplina:
[`references/success-criteria.md`](references/success-criteria.md) e
[`references/goals.md`](references/goals.md).

**Gate:** Fase 6 só existe depois de `Goals - <Título da feature>` com
todo `owner_task` preenchido. Código de produto com `/goal` vazio → pare.

---

## Fases

Paths e comandos vêm do Profile. Nada hardcoded.

### Fase 1 — Discover & sync · main

Identifique sub-projetos e o expert de cada stack. Atualize a partir de
`vcs.base` (ou hotfix). Área desconhecida → **Fase 1b**: `plan` com
handoff rico; orquestrador não varre o repo sozinho.

### Fase 2 — Dual brainstorm · main + `plan` / UX

Contrato e templates: [`references/brainstorm.md`](references/brainstorm.md).

- **2a produto** (orquestrador): problema, users, success_criteria,
  constraints, non_goals, invariants, unknowns, risks. Grave
  `Brainstorm - <Título> (produto)`.
- **2b código** (`plan`): lane, blast radius, construção, decisions,
  code_success_criteria, async_hint. Grave
  `Brainstorm - <Título> (codigo)`.
- UI/UX: despache `expert-frontend-pwa` e/ou `expert-frontend-web` **em
  paralelo**, só para modelos de decisão — sem código de produto.
- `unknowns` que mudam o desenho bloqueiam a Fase 3.

Lane S: 2a obrigatório (pode ser um critério); 2b pode caber no handoff.

### Fase 3 — Spec + Plan + /goals · `plan` (`reasoning`) · M/L

Paralelo à preparação de ambiente. O `plan` lê os dois brainstorms e
produz, no mesmo destino:

```
<destino>/<feature>/
├── Brainstorm - <Título> (produto).md
├── Brainstorm - <Título> (codigo).md
├── Goals - <Título>.md
├── Spec - <Título>.md
├── Design - <Título>.md          # só se houver decisão de arquitetura
├── Tasks - <Título>.md
└── Plan - <Título>.md
```

Spec: requisitos rastreáveis a `G-`. Design: só decisão genuína. Tasks
atômicas, cada uma com:

```
id, what, where, depends_on, async, goals, reuse, done_when, tests, commit
```

`async: true` quando `depends_on` está vazio e não há colisão de escrita.
**Desenvolvimento não começa** enquanto algum `/goal` estiver sem
`owner_task`.

#### Regra de Destino

1. Bloco `memory:` → vault `<vault>/70-Specs/<feature>/` via `obsidian`
   CLI. Carregue `memory-graph:memory-vault`.
2. Senão `specs_dir` do Profile.
3. Senão `.specs/<feature>/` e avise que o projeto ganharia um vault.

Nome: `<Tipo> - <Título da feature>`. Tipos: `Brainstorm` · `Goals` ·
`Spec` · `Design` · `Tasks` · `Plan`. Nunca `spec.md`. Sanitize
`\ / : * ? " < > | # ^ [ ]`. CLI do Obsidian mente no exit code —
confira stdout e releia.

### Ambiente · main · paralelo à Fase 3

Isolamento por sub-projeto, branch pela convenção `vcs`, cache de
dependências se o lock não mudou.

### Fase 6-S — Lane S · `impl` + `review` + `security`

Handoff com o `/goal`. Primeira ação do `impl` = `## /plan`. Depois
código + testes da
[`references/verification.md`](references/verification.md). No mesmo
diff, em paralelo: `review` e `expert-security`. Commit só com
`APPROVE` + `SECURE`.

### Fase 6 — Lanes M/L · ondas async

[`references/async-dispatch.md`](references/async-dispatch.md). Uma
onda = um `tasks[]`. Cada `impl` faz `/plan` da **sua** task e só então
edita. Review+security encadeiam no diff da task; não bloqueiam o impl
da próxima onda.

`/goal` vermelho no limite de iterações → relatório de falha →
**respawn** do mesmo expert com o mesmo `/goal`. Só o humano cancela.

### Fase 7 — Gate de testes · main

Testes são a prova do `/goal`, não um adorno. Rode **todo** comando do
Profile (unit, integ, e2e/Playwright, lint, types, build) nas stacks
tocadas, na forma do CI (`ci_gotchas`). Sem evidência de cada `/goal`,
não abre PR. Falha → expert da stack, sem enfraquecer a asserção.

### Fase 8 — Commit & PR · main

Commits específicos, hooks ligados, sync com a base antes do PR. Um PR
por sub-projeto. Descrição lista G1…Gn e a evidência. Sem segredo no
diff. Depois do merge, limpe o isolamento.

---

## Fechamento · main

Com bloco `memory:`: nota no vault + linha em `90-Log/AAAA-MM.md` via
CLI. Sem memória, pule. Protocolo: `memory-graph:memory-vault`.

---

## Dispatch de expert

Spawne o **agente nomeado**. Cole 5–15 linhas da skill; ele lê o
`SKILL.md` na primeira ação.

| Stack / eixo | Agente | Classe |
|---|---|---|
| `expert-backend-go` | `expert-backend-go` | `code` |
| `expert-backend-python` | `expert-backend-python` | `code` |
| `expert-frontend-react` | `expert-frontend-react` | `code` |
| `expert-frontend-vue` | `expert-frontend-vue` | `code` |
| UX mobile-first | composto no handoff, ou `expert-frontend-pwa` se só UX | `code` |
| UX web-first | idem `expert-frontend-web` | `code` |
| persistência | também `expert-database` | `code` |
| correção | `review` | `critique` |
| segurança | `expert-security` | `security` |
| spec/plan/goals/1b | `plan` | `reasoning` |

Persistência → carregue também `expert-database`. Frontend → engenharia
**mais** UX da área; cruzou áreas → as duas UX, fronteira no handoff.

---

## Quick Reference

```
FASE 0: Profile + papéis
LANE: S | M | L
MANDATO: orquestrador não implementa (>100 chars)

1. Discover     → coord · sync base · 1b = plan
2a. Feature     → coord · Brainstorm (produto) · success_criteria
2b. Código      → plan + UX experts · Brainstorm (codigo) · decisions
   GATE         → Goals - <Título> com owner_task em todo G-
3. Spec+Plan    → plan · tasks com async/depends_on/goals
Env             → coord · paralelo à 3
6. Impl         → ondas tasks[] · cada impl começa com /plan
                → review + expert-security no diff · APPROVE+SECURE
7. Testes       → coord · unit + e2e/Playwright · todo /goal com evidência
8. PR           → coord · lista G1…Gn
```

---

## Red Flags — PARE

- Workflow sem Profile, ou em path fora do Profile
- Orquestrador implementando/analisando/ajustando >100 chars
- Fase 0 pulada; lane não classificada
- Código de produto antes de `/goal` com `owner_task` (M/L) ou sem
  `/goal` (S)
- Spec/plan/brainstorm/goals dentro do código do sub-projeto
- `.specs/` em projeto com `memory:`
- Nome `spec.md` / `tasks.md` em vez de `<Tipo> - <Título>`
- Vault com `Write`/`Edit` ou sem checar stdout do CLI
- Handoff “leia a skill e o plan” em vez de colar `/goal` + recorte
- Review sem diff colado
- **Uma task por vez** com `async: true` na lista
- `impl` que edita sem emitir `## /plan`
- Commit sem `APPROVE` + `SECURE`, ou `SECURE` sem evidência
- Frontend novo sem spec Playwright; API nova sem matriz unitária
- `/goal` fechado com `skip` no teste que era a `evidence`
- Pedir exploit / PoC / probe em prod ao `expert-security`
- Carregar skill ofensiva da biblioteca de cybersecurity
- Spawnar `impl` genérico quando existe expert da stack
- Hardcodar slug de LLM, comando ou path que deveria vir do Profile
