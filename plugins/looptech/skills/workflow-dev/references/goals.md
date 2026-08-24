# /goal — Contrato que faz o agente trabalhar até o sucesso

Spec e plan **não** são documentação decorativa. Eles existem para
materializar `/goal`. O agente para quando o `/goal` está satisfeito —
não quando “parece pronto”, não no fim da janela, não depois de um
único passe.

## Por que isto existe

Subagente começa sem histórico. Sem um objetivo binário colado no
handoff, ele entrega um recorte e encerra. O `/goal` é a condição de
parada do loop ReAct (`autonomy-react-loop.md`) **e** a condição de
parada do orquestrador (respawn até cumprir ou o humano abortar).

## Artefato

Grave `Goals - <Título da feature>.md` no mesmo destino da spec.
Deriva **somente** do dual brainstorm (`brainstorm.md`). Nada entra
aqui que não esteja no 2a/2b.

```yaml
feature: "<Título da feature>"
source:
  produto: "[[Brainstorm - <Título> (produto)]]"
  codigo: "[[Brainstorm - <Título> (codigo)]]"
goals:
  - id: G1
    kind: produto | construcao
    statement: "<o que tem de ser verdade>"
    source_id: SC1   # ou CC1 / D1
    done_when: "<asserção binária, testável>"
    evidence: "<comando do Profile, teste, tela, query>"
    owner_task: T1   # preenchido na Fase 3; vazio = spec incompleta
    async_ok: true   # a task dona pode rodar em paralelo
```

## O que torna um /goal válido

| Válido | Inválido |
|---|---|
| `done_when`: "POST /x retorna 201 com `id` e o GET seguinte devolve o mesmo corpo" | "API funcionando" |
| `evidence`: `go test ./internal/foo -count=1` (comando do Profile) | "testar localmente" |
| `done_when`: "Playwright `e2e/checkout.spec.ts` passa sem skip" | "UX ok" |
| Um `owner_task` depois da Fase 3 | Goal órfão sem task |

Um `/goal` de produto cobre o critério de sucesso do usuário. Um
`/goal` de construção cobre fronteira de código (contrato, teste,
migração, CSP). Os dois tipos convivem. Feature 100% = **todos** os
`/goal` verdes.

## Gate — desenvolvimento só depois

| Fase | Exige |
|---|---|
| 2a/2b | critérios escritos |
| 3 | cada critério virou `/goal`; cada `/goal` ganhou `owner_task` |
| 6 / 6-S | arquivo `Goals - …` existe; **zero** `owner_task` vazio |
| commit | `/goal` da task com evidência colada no retorno do `impl` |
| PR | orquestrador declara a lista G1…Gn e a evidência de cada um |

Começar a escrever código de produto com `/goal` indefinido é red flag
— o orquestrador para e volta à Fase 2/3.

Lane S: pelo menos um `/goal`. Pode viver no handoff se o destino ainda
não tiver arquivo; o texto do contrato é o mesmo.

## Como o /goal entra no agente

O orquestrador **cola** no bloco Objetivo Final de
`subagent-handoff.md` o(s) `/goal` daquela task — `id`, `statement`,
`done_when`, `evidence` — não um ponteiro para o arquivo.

O `impl`:

1. Emite o `/plan` da task (`plan-before-impl.md`) contra esses `/goal`.
2. Implementa.
3. Roda a `evidence`.
4. Só se dá por concluído quando cada `done_when` da task é sim.

Se o limite de iterações estoura sem o `/goal` verde, o subagente
**reporta falha** (não sucesso disfarçado). O orquestrador **respawna**
o mesmo expert com o relatório + os mesmos `/goal`. Não troca o
objetivo no meio do caminho. Só o humano cancela um `/goal`.

## Rastreio

- Spec referencia `G-` em cada requisito.
- Tasks carregam `goals: [G1, G2]`.
- Review de correção compara o diff ao `done_when` dos `/goal` da task,
  não a uma impressão de qualidade.
- Fase 7 falha se algum `/goal` da feature não tiver evidência rodada.
