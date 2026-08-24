# Handoff de Subagente — Entrada Rica + Isolamento + Retorno Estruturado

Referenciado por **todo** template de subagente em `workflow-dev`. Nenhum subagente é
despachado sem seguir este protocolo.

## Princípio

O orquestrador nunca despacha um subagente "para descobrir". O orquestrador já sabe o
suficiente do Project Profile, da skill expert, do `/goal` e da investigação prévia para
**colar** esse conhecimento no prompt. O subagente **recebe → emite `/plan` → executa**;
ele não gasta iterações remontando contexto que o orquestrador já tinha.

## Entrada — o orquestrador SEMPRE cola os quatro blocos

### 1. Objetivo Final = `/goal`
Cole o(s) `/goal` da task (`id`, `kind`, `statement`, `done_when`, `evidence`) de
`goals.md` — texto, não ponteiro. Isso é a condição de parada. Nunca despachar com
"melhora isso aí". `impl` ainda não autorizado a editar: primeira ação é o
`## /plan` de `plan-before-impl.md`.

### 2. Estado Atual
Referências **exatas** — arquivos, dados, trechos de código **colados no prompt**, não
apontados por caminho para o subagente "ir ler depois". Se o orquestrador já leu o arquivo
relevante, o conteúdo relevante vai no prompt. Isso inclui:
- Caminhos absolutos dos arquivos/diretórios envolvidos.
- Trechos de código/config/dados já coletados.
- Resultado de comandos já executados (test/lint/build, queries, etc.).

> **Nunca "leia o §X de um documento grande".** Colar o trecho exato de que o subagente
> precisa é o protocolo; mandá-lo abrir uma spec/design de centenas de linhas para extrair
> sua parte força re-leitura — o mesmo documento relido por N subagentes em paralelo — e é
> exatamente o "redescobrir" que este protocolo existe para matar. Cole o texto-fonte,
> não o ponteiro. Só peça a um subagente dev que abra o **arquivo-fonte que ele próprio vai
> editar**.

### 3. Variáveis Críticas
Parâmetros de negócio relevantes à tarefa — regras não-negociáveis, valores/limites que não
podem ser inferidos errado, convenções do projeto que mudam o resultado se ignoradas.

### 4. Conhecimento expert da arquitetura
As 5–15 linhas da skill expert (stack + Project Profile) que impõem arquitetura,
desacoplamento e convenções nos arquivos tocados. O subagente **respeita a arquitetura do
projeto** porque já recebeu a regra — não porque foi descobrir sozinho lendo o repositório
inteiro.

## Granularidade e fan-out — 1 subagente por unidade independente

Cada subagente recebe **somente o que é da sua responsabilidade**. A decomposição
está em `Tasks - <Título>` (`async`, `depends_on`, `goals`). Protocolo de ondas:
`async-dispatch.md`.

- **Delegar é o padrão.** Na dúvida, delegue.
- **Uma unidade independente = um subagente, na mesma onda.** Despache a onda
  numa única chamada `tasks[]`. Uma task por vez com `async: true` é red flag.
- **Colisão ⇒ serialize.** Mesmo arquivo, mesmo recurso, ou `depends_on`
  não vazio: espere a anterior. Duas escritas no mesmo arquivo corrompem
  o resultado.
- **Nunca empacote N arquivos independentes num único subagente** — isso
  serializa o que deveria ser uma onda.
- **Só agrupe no mesmo subagente** tasks genuinamente acopladas.
- **Tarefas pequenas — respeite a janela do subagente.** Cada subagente tem janela finita (na
  ordem de ~350k tokens). Dimensione cada subtask para caber com folga; se exigir ler/escrever
  muito, **quebre em unidades menores antes de despachar**. Task grande demais estoura o
  contexto no meio, gera retrabalho e entrega pela metade. O checkpoint de progresso é rede de
  segurança, não desculpa para task gigante.
- **Primeiro prompt completo.** Faça o recon (grep de paths, edge cases, estrutura) **antes**
  de despachar, e coloque tudo no primeiro prompt — evita rodadas de follow-up, cada uma
  pagando cold-start de novo.

## Isolamento de contexto

Todo ruído da execução — buscas exploratórias, falhas de leitura, tentativa-e-erro,
iterações do loop de autonomia — fica **apenas** na janela de contexto do subagente. Nada
disso consome a janela do agente principal. Ao terminar, o subagente devolve **só o
resultado sintetizado**, nunca o rastro bruto de exploração.

## Checkpoint de progresso — o subagente é dono do próprio contexto

Todo subagente de **tarefa longa** (múltiplos passos, edição de vários pontos, investigação
extensa) mantém o **seu próprio arquivo de checkpoint** em memória/scratch — não depende da
janela de contexto sobreviver inteira até o fim.

- **O que registrar:** o que já foi feito, o estado atual, e o **próximo passo concreto** —
  atualizado a cada marco relevante, não só no fim.
- **Por quê:** se o contexto do subagente estourar no meio da tarefa, ele **retoma a partir
  do checkpoint** e continua até finalizar — sabe exatamente onde parou e o que falta, sem
  recomeçar do zero nem devolver trabalho pela metade.
- **Onde:** um path de scratch/memória próprio do subagente (não o do orquestrador); o
  subagente cita esse path no retorno para rastreabilidade.
- **Condição de encerramento:** o subagente só se dá por concluído quando
  cada `/goal` da task está verde (`goals.md`). O checkpoint garante que
  a tarefa longa chega ao fim mesmo cruzando limite de contexto.

## Retorno estruturado (obrigatório)

Todo subagente encerra sua resposta com este template Markdown exato — três seções, nesta
ordem, sem variação de título:

```markdown
## O que foi feito
<rastro auditável; impl inclui o ## /plan emitido>

## Evidências / Resultados
<saída de teste / Playwright / comando do Profile; /goal Gn: verde|vermelho>

## Próximos Passos
<o que o agente principal precisa fazer com isso>
```

- **O que foi feito** — resumo do trabalho realizado, em ordem, auditável.
- **Evidências / Resultados** — a prova concreta (saída de comando, trecho de código,
  contagem de linhas, diff) que sustenta a conclusão. Nunca uma afirmação sem evidência.
- **Próximos Passos** — a ação que o agente principal (ou o próximo subagente) precisa tomar
  a partir daqui.

Subagente `review`: em Evidências, a última linha é `VEREDITO: APPROVE` ou
`VEREDITO: CHANGES-REQUESTED`. Subagente `security`: última linha `VEREDITO: SECURE`
ou `VEREDITO: ISSUES-FOUND` (ver `security-review.md`). Sem essa linha o orquestrador
trata o review como inválido e não commita.

## Red flag

Se um subagente está **"descobrindo"** algo que já era sabido pelo orquestrador — relendo um
arquivo que o orquestrador já tinha aberto, redescobrindo uma convenção que a skill expert já
declarava — o contexto entregue **estava incompleto**. A correção é no **prompt** do próximo
despacho, nunca uma cobrança ao subagente. Um subagente que investiga do zero é sintoma de
handoff malfeito, não de subagente preguiçoso.
