# Dispatch async — multi-agente é o padrão

Uma task por vez, quando as tasks são independentes, é o anti-padrão
que este workflow existe para matar. O orquestrador **força** fan-out.

## Regra

1. A Fase 3 marca cada task com `async: true|false` e `depends_on: []`.
2. Monte **ondas**. Uma onda = todas as tasks com `async: true` cujas
   dependências já terminaram.
3. Despache a onda em **uma** chamada `Task` com `tasks[]` — um
   subagente por item, experts da stack, handoff completo em cada
   `task`.
4. `review` + `expert-security` da task que já entregou rodam **em
   paralelo** com o `impl` da próxima onda, desde que não colidam no
   mesmo arquivo.
5. Só serialize quando `depends_on` não está vazio **ou** duas tasks
   escrevem o mesmo arquivo / o mesmo recurso.

```
onda 1:  impl T1  impl T3  impl T4     ← um tasks[] só
           │        │        │
           ▼        ▼        ▼
         rev+sec  rev+sec  rev+sec     ← paralelo ao que já pode
onda 2:           impl T2 (depends T1)
```

## O que conta como independente

Independente = não compartilham arquivo de escrita, não compartilham
estado mutável (mesma tabela em migration conflitante, mesmo módulo de
wiring), e nenhuma precisa do *resultado* da outra para começar.

Leitura do mesmo arquivo é ok em paralelo. Duas escritas no mesmo
arquivo → serialize. UX e engenharia no mesmo SFC → um writer
(handoff composto), não dois `impl` no mesmo path.

## Tamanho da onda

- Cabe no cap de concorrência do host (não invente mais de 32).
- Cada item é uma unidade que cabe na janela do subagente.
- N arquivos independentes = N agentes, não 1 agente editando em
  sequência.
- Backend + frontend sem contrato cruzado instável = onda única.
- Contrato de API ainda não mergeado + client que depende dele =
  backend na onda 1, frontend na onda 2.

## Proibido

- Spawnar T1, esperar, spawnar T2 “para ficar simples”.
- Um `impl` genérico quando existe `expert-backend-go` /
  `expert-frontend-react` / etc.
- Empacotar a feature inteira num único subagente “porque é uma
  feature”.
- Esperar o review de T1 para *começar* T3 quando T3 não depende de T1.

Se o orquestrador se pega fazendo uma task por vez com `async: true`
na lista, **para**, remonta a onda e despacha de novo.

## Handoff da onda

O `context` do batch carrega o que é comum (Profile, `/goal` da
feature, convenções). Cada `task` carrega só o recorte: `/goal` da
task, files, Done when, testes, 5–15 linhas do expert. Ver
`subagent-handoff.md`.
