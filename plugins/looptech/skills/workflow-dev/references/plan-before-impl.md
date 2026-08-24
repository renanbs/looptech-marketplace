# /plan — Plan mode antes de cada task de impl

Receber uma task **não** autoriza escrever código. O primeiro ato do
`impl` (e de qualquer ajuste pós-review) é um `/plan` da **task**, não
da feature inteira. A spec/plan da Fase 3 já existe; este passo é o
desenho tático de *esta* unidade, contra o `/goal` colado.

Isto é o “plan mode” do workflow: host-agnóstico. Em host com modo
`/plan` nativo, use-o se o orquestrador estiver no nível da feature;
**dentro do subagente** o protocolo abaixo é obrigatório mesmo assim.

## Primeira ação do impl (antes de Write/Edit)

Emitir, no próprio transcript (e no checkpoint se a task for longa):

```markdown
## /plan
goal: G1, G2
approach: <como esta task cumpre o done_when, 3–8 linhas>
files:
  - <path exato que vai mudar e por quê>
reuse:
  - <símbolo / padrão existente>
tests:
  - <caso unitário / e2e / Playwright que prova o /goal>
risks:
  - <o que pode quebrar fora do escopo>
out_of_scope:
  - <o que esta task não toca>
```

Só depois disso: editar código. Write/Edit sem `## /plan` no transcript
é handoff inválido — o orquestrador manda refazer, não commita.

## O que o /plan da task não é

- Não reescreve a spec.
- Não reabre `non_goals` do brainstorm.
- Não agrupa a próxima task independente “já que estou aqui”.
- Não substitui o `/goal`. O plano serve o goal; o goal não se
  encolhe para caber no plano.

## Orquestrador

No prompt de cada `impl`, além dos quatro blocos de
`subagent-handoff.md`:

1. Cole o `/goal` da task.
2. Ordene: “primeira ação = `## /plan`; zero edições antes”.
3. Cole `verification.md` (matriz de testes da stack).
4. Não peça “carregue a skill e leia o plan da feature” — cole o
   recorte da task.

Tasks independentes: cada `impl` faz o **seu** `/plan` em paralelo.
Não serialize plan→impl→plan→impl entre fatias sem dependência.

## Lane S

Vale igual. O `/plan` cabe em poucos campos; a ausência não é
permitida só porque a mudança é pequena.
