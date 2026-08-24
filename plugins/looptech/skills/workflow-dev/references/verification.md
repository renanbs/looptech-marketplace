# Verificação — testes são a prova do /goal

Nenhuma task está feita porque o código compila. O `/goal` só fica
verde com evidência executada. Esta página é a disciplina; os
**comandos** vêm do Project Profile (`commands.<stack>.test` /
`.integ` / `.e2e`).

## Pirâmide por eixo

| Eixo | Obrigatório na task que toca esse eixo | Ferramenta |
|---|---|---|
| Backend (domínio / use case) | Unitário da matriz de casos da API | `go test` / `pytest` (Profile) |
| Backend (repositório) | Integração + injection | testcontainers |
| Backend (feature) | E2E HTTP cobrindo **cada** `/goal` de produto da API | cliente HTTP real da suíte do projeto |
| Frontend (hook/composable / componente) | Unit + component no comportamento visível | vitest + Testing Library |
| Frontend (feature) | E2E Playwright cobrindo **cada** `/goal` de produto de UI | [Playwright](https://playwright.dev) |
| Persistência | Integração + injection (skill `expert-database`) | container real |

“Golden path só” **não** fecha feature. Edge cases ficam no unitário;
o E2E fecha os `/goal` de produto, um a um.

## Matriz unitária de API (backend)

Todo endpoint / use case novo ou alterado precisa de testes
parametrizados (table-driven) cobrindo **todos** os casos abaixo que
se aplicam. Omitir um caso exige uma linha no `/plan` dizendo por que
não se aplica.

| Caso | O que asserir |
|---|---|
| Feliz | status de sucesso + corpo contratado + efeito persistido |
| Validação | cada campo obrigatório / invariante — 4xx, sem efeito colateral |
| Não autenticado | 401 (ou o contrato do projeto), sem vazamento |
| Não autorizado / outro tenant | 404 (não 403 que vaza existência), sem mutação |
| Não encontrado | vazio/404 conforme o contrato |
| Conflito / idempotência | replay do mesmo comando não duplica efeito |
| Fronteira | limites numéricos, string vazia, null, tamanho máximo |
| Corrida (se dinheiro/estado) | dois concorrentes, um vence, invariante de saldo/estoque |

Injection de string em repositório continua obrigatória (skills
backend + `expert-database`).

O `impl` roda o comando de teste do Profile **na pasta da stack**
antes de devolver. Sem a saída colada em Evidências, o `/goal` não
está verde.

## E2E da feature (100%)

Para cada `/goal` de produto (`kind: produto`):

- Backend: um teste E2E HTTP que exercita o fluxo real (servidor de
  teste + banco de teste). Não mocka o use case por dentro.
- Frontend: um spec Playwright que percorre o fluxo na UI real
  (queries por papel/label, não `data-testid` como padrão).
- Feature full-stack: os dois. O `/goal` de UI não se prova com
  `go test`; o `/goal` de API não se prova só com Playwright.

Skip / `xit` / `t.Skip` no caminho do `/goal` = `/goal` vermelho.

## Playwright (frontend)

Playwright é a ferramenta de E2E de UI deste workflow
([documentação](https://playwright.dev)). Comando concreto no Profile
(`commands.<frontend>.e2e`). Não abra uma segunda suíte (Cypress etc.)
em feature nova. Projeto legado sem Playwright: a task que introduz o
primeiro fluxo de UI **adiciona** Playwright; não “equivale” com
clique manual.

O `impl` frontend **roda** o spec que ele escreveu. Afirmar que o
fluxo existe sem a saída do Playwright é review `CHANGES-REQUESTED`.

## Fase 7

Depois de todas as tasks commitadas, o orquestrador roda **todo** o
gate do Profile (unit + integ + e2e + lint + types + build) nas
stacks tocadas. Um `/goal` sem evidência nesta fase bloqueia o PR.

Falha de teste → expert da stack, com o `/goal` e a saída colados;
nunca “ajustar o teste para passar” apagando a asserção do `done_when`.
