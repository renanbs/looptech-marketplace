# Lanes (S/M/L) e o Delegation Mandate

O `workflow-dev` mantém o esqueleto — discover → dual brainstorm → `/goals` →
spec+plan → env → `/plan` por task → impl async → review → security →
testes → PR — mas **zero** paths/comandos/slugs hardcoded: tudo via
`project-profile.md` e `agent-roles.md`. Este documento cobre o sizing
por lane e quem executa cada ação.

## Fase 0 — Resolver Project Profile

Antes de classificar a lane, o orquestrador lê o Project Profile (`CLAUDE.md` / `AGENTS.md`), monta
`path → stack → comandos` a partir do Project Profile e detecta stacks ausentes por
manifesto (ver `project-profile.md`). Nenhuma lane é classificada sem essa resolução prévia.

## Dispatch de expert

Para cada sub-projeto tocado pela task:

- Carrega a skill expert da stack resolvida.
- Se a task tocar a camada de persistência/banco de dados, carrega **também** a skill de
  banco de dados agnóstica.
- Para frontend, carrega a skill de **engenharia** (ex.: `expert-frontend-react`) **+** a
  skill de **UX** resolvida por área (`ux_overrides`/`ux_default` → skill mobile-first ou
  web-first; ambas se a task cruzar áreas — ver `project-profile.md`).

Todo template de subagente despachado a partir daqui referencia `subagent-handoff.md` e,
para tarefas de dev/debug/investigação, também `autonomy-react-loop.md` — colando o
conhecimento expert de arquitetura relevante. Critérios de sucesso (`success-criteria.md`)
são obrigatórios na entrada de toda spec/plan.

A fase de lint/tests roda os comandos declarados no Project Profile — nunca comandos
hardcoded do plugin — mantendo dois princípios: **reproduza localmente cada check que o CI
roda** e **lint só-linhas-novas quando for essa a forma que o CI usa**.

---

## Delegation Mandate — o agente principal é PURO ORQUESTRADOR

Regra dura, acima de qualquer lane: **o agente principal nunca faz análise de código,
implementação, ajuste ou ajuste pós-review por conta própria — tudo é delegado a
subagente.**

| Ação | Quem faz |
|------|----------|
| Análise/investigação de código, mapeamento, blast radius | **subagente** (handoff rico + ReAct) |
| Implementação de qualquer task (inclusive lane S) | **subagente** dev |
| Ajustes pós-review (aplicar CHANGES-REQUESTED) | **subagente** dev |
| Code review de correção de todo diff | **subagente** `review` (`critique`) |
| Review de segurança de todo diff de código | **subagente** `security` |
| **Edição trivial ≤ 100 caracteres** (typo, bump de versão, uma linha de config) | orquestrador pode fazer direto |
| Coordenação: ler Project Profile, classificar lane, colar contexto, despachar, coletar síntese | orquestrador |
| Rodar comandos de verificação/git (test/lint/build, commit, push, abertura de PR) | orquestrador (executa o comando; **a análise de falha e o fix vão para subagente**) |

A única exceção ao mandato é a **edição trivial ≤ 100 caracteres** — um typo, um bump de
versão, uma linha de config isolada. Qualquer coisa além disso, mesmo que pareça pequena,
vai para um subagente dev com handoff rico.

## Sizing das lanes

O tamanho da lane define a **ceremônia** (quanto planejamento formal precede a implementação)
— nunca quem implementa. Implementação é **sempre** delegada a subagente, em toda lane.

### Lane S — mudança pequena e localizada
Poucos arquivos, sem cruzar sub-projeto, sem decisão de arquitetura em aberto. Mesmo assim:

- **Não existe "orquestrador implementa direto".** A implementação vai para **um
  subagente** expert da stack.
- Review de correção **e** de segurança obrigatórios (security só pula em diff
  100% não-runtime).
- 2a (brainstorm de produto) + pelo menos um `/goal`. Spec/plan podem viver no
  handoff. `/plan` da task e testes da `verification.md` continuam obrigatórios.

### Lane M — mudança de escopo médio
Múltiplos arquivos, possivelmente cruzando sub-projetos ou tocando dados. Dual
brainstorm (2a+2b), `Goals - <Título>`, spec e plan formais, tasks com
`async`/`depends_on`. Ondas em `tasks[]`. Cada task herda seus `G-`.

### Lane L — mudança de escopo grande
Nova feature, mudança estrutural, ou risco alto (financeiro, segurança, dado
sensível). Pipeline completo: discover → 2a+2b (UX experts se houver UI) →
`/goals` → spec+plan → env → ondas de impl com `/plan` por task → review+security
por diff → gate de testes → PR. Feature 100% = todos os `/goal` verdes.

## Loop de ajuste pós-review

Quando o `review` devolve `CHANGES-REQUESTED` **ou** o `security` devolve `ISSUES-FOUND`:

1. **Novo subagente `impl`** aplica os ajustes, recebendo o diff + os pontos (handoff rico).
2. Novo ciclo de **correção e segurança** sobre o diff ajustado.
3. Só após `APPROVE` **e** `SECURE` o commit acontece.

O orquestrador **nunca** aplica o fix pós-review ele mesmo — mesmo que o ajuste pareça
trivial, a menos que caia dentro da exceção de ≤ 100 caracteres.

## Planejamento do plugin

O contrato de brainstorm, `/goal`, `/plan` por task, async e verificação vive
nas referências deste plugin. Se o projeto declarar uma skill extra de
planejamento no Profile, o `plan` ainda assim **tem** de emitir `Goals -` e
tasks com `async`/`depends_on` — senão a Fase 6 não abre.
