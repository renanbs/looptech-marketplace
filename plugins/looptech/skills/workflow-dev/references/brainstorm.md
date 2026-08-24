# Dual Brainstorm — Feature + Código

A Fase 2 não é conversa solta. São **dois artefatos** gravados no destino da
Regra de Destino. Sem os dois (lane M/L) o `plan` **não** abre spec. Sem o
primeiro (toda lane) **não** existem `/goal`.

O orquestrador conduz no agente principal — precisa do contexto da conversa.
Ambiguidade de produto resolve com o usuário **aqui**, não no meio do impl.

## 2a — Brainstorm da feature (produto)

Responde: o que é sucesso, o que a feature resolve, e quando ela está
**100% implementada**. Não descreve arquivos.

Grave `Brainstorm - <Título da feature> (produto).md` com este bloco
preenchido (YAML dentro de fence, mais prosa só se um campo precisar de
contexto):

```yaml
problem:
  what: "O que estamos resolvendo?"
  why: "Por que isso precisa existir agora?"

users:
  - role: "<quem>"
    job: "<o que precisa conseguir>"

success_criteria:
  - id: SC1
    statement: "<observável, binário>"
    evidence: "<comando, tela, contrato ou dado que prova>"

constraints:
  - "<não-negociável de negócio, prazo, compliance, stack>"

non_goals:
  - "<o que explicitamente NÃO entra>"

invariants:
  - "<o que não pode quebrar enquanto entregamos isto>"

unknowns:
  - "<o que ainda não sabemos; quem decide; até quando>"

risks:
  - "<o que pode falhar e o prejuízo>"
```

Regras:

- Cada `success_criteria` vira um `/goal` de produto em `goals.md`.
- Critério sem `evidence` é inválido — volte e complete.
- `unknowns` abertos que mudam o desenho **bloqueiam** a Fase 3 até o
  usuário decidir ou o item virar `constraint` / `non_goal`.
- Lane S pode caber num único critério; ainda assim o bloco existe.

## 2b — Brainstorm do código (construção)

Responde: como o código entrega o 2a, em que tamanho, em quais stacks,
quais decisões de construção já estão tomadas. Não implementa.

Grave `Brainstorm - <Título da feature> (codigo).md`:

```yaml
lane: S | M | L
subprojects:
  - path: "<path do Profile>"
    stack: "<expert-*>"
    ux: "<expert-frontend-pwa|expert-frontend-web|nenhuma>"
blast_radius:
  - "<módulo / contrato / tabela / rota tocados>"
construction:
  approach: "<como vamos construir, em 3–8 linhas>"
  size: "<arquivos estimados, cruzamento de sub-projeto, risco>"
  reuse:
    - "<símbolo / padrão existente a reusar — não inventar o segundo>"
decisions:
  - id: D1
    question: "<decisão de construção>"
    options:
      - id: A
        what: "<opção>"
        tradeoff: "<custo>"
      - id: B
        what: "<opção>"
        tradeoff: "<custo>"
    recommended: A
    why: "<por que esta, não a outra>"
    expert: "<agente que validou, se houver>"
code_success_criteria:
  - id: CC1
    statement: "<binário de construção — teste, contrato, fronteira>"
    evidence: "<comando do Profile ou asserção observável>"
async_hint:
  - "<fatias que já nascem independentes>"
```

Regras:

- Cada `code_success_criteria` vira um `/goal` de construção.
- `decisions` sem `options` (≥2) e `recommended` é inválida.
- Se a área for desconhecida, o orquestrador despacha `plan` (Fase 1b)
  **antes** de fechar o 2b — não explora o repo inteiro no próprio contexto.

## UI/UX — modelos de decisão obrigatórios

Se a feature toca interface (tela, fluxo, formulário, tabela, PWA, admin):

1. Resolva o eixo de UX pelo Profile (`ux_default` / `ux_overrides`).
2. Despache o expert de UX (`expert-frontend-pwa` e/ou
   `expert-frontend-web`) **em paralelo** com o `plan` do 2b, só para
   produzir modelos de decisão — **sem escrever código de produto**.
3. Cole no 2b um `decisions[]` por pergunta de interação. Cada modelo
   cobre: layout, densidade, caminho sem hover, alvo de toque **ou**
   teclado, estado vazio/erro/loading, e acessibilidade.
4. Cruzou mobile-first e web-first → os dois experts, fronteira explícita.

O expert de engenharia (React/Vue) **não** substitui este passo. UX decide
interação; engenharia decide componente/tipo/teste.

## Aceite

2a e 2b (M/L) ou 2a (S) estão aceitos quando:

- todo `success_criteria` / `code_success_criteria` é binário;
- nenhum `unknown` crítico está aberto;
- UI, se houver, tem `decisions[]` com expert nomeado;
- o orquestrador anunciou a lane.

Só então a Fase 3 lê estes arquivos e deriva `/goal`.
