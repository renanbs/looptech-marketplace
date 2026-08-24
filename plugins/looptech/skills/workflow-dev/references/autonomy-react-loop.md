# Loop de Autonomia / Self-Correction (ReAct) + Trava de Segurança

Bloco embutido em todo prompt de subagente de investigação, desenvolvimento ou debug
despachado pelo `workflow-dev`.

## O bloco a colar no prompt do subagente

```markdown
# INSTRUÇÕES DE AUTONOMIA E REPROCESSAMENTO
Você tem autonomia para executar e reexecutar ferramentas em loop até os
`/goal` colados no Objetivo Final estarem verdes (done_when = sim + evidence
rodada).

## Processo iterativo (por tentativa):
1. PENSAMENTO: estado atual, o que falta no `/goal`, por que a tentativa
   anterior falhou.
2. AÇÃO: execute a ferramenta adequada. Na primeira iteração de impl, a
   ação é emitir `## /plan` — ainda sem Write/Edit.
3. OBSERVAÇÃO: analise o resultado retornado.
4. VALIDAÇÃO: cada `/goal` da task está verde? Se sim, finalize. Se não,
   ajuste e reinicie o ciclo. Não invente outro critério de parada.

## Restrições (trava de segurança):
- Máximo de 12 iterações por task de impl; 5 por subtarefa de investigação.
- Ao atingir o limite sem o `/goal` verde, PARE e reporte ao orquestrador:
  motivo, o que foi tentado, estado final. Ele respawna com o mesmo `/goal`.
- Nunca declare sucesso com teste skipped ou sem colar a evidence.
```

## O ciclo, explicado

- **PENSAMENTO** — articula o estado e a causa da falha anterior. Nunca
  repete ação idêntica sem hipótese nova.
- **AÇÃO** — uma ferramenta por iteração. Em impl, a primeira ação é
  emitir `## /plan`.
- **OBSERVAÇÃO** — lê o resultado bruto antes de concluir.
- **VALIDAÇÃO** — compara contra o `done_when` de cada `/goal` colado.

## Trava de segurança

Impl: 12 iterações por task. Investigação/debug: 5. Ao limite sem `/goal`
verde, o subagente **para** e monta relatório de **falha**:

1. **Motivo** — causa-raiz com evidência.
2. **O que foi tentado** — iterações em ordem.
3. **Estado final** — o que o código/dado ficou.
4. **`/goal` ainda vermelhos** — ids.

O relatório segue `subagent-handoff.md`. O orquestrador **respawna** o
mesmo expert com este relatório + os mesmos `/goal`. Só o humano cancela
um `/goal`.

## Critério de Parada é o /goal

A VALIDAÇÃO compara contra o `done_when` colado. "Tente até ficar bom"
não é critério. Sem `/goal` binário, volte a `goals.md` antes de
despachar.
