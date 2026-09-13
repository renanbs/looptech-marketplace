---
name: expert-backend-python
description: Disciplina de ENGENHARIA backend em Python — arquitetura Clean + Hexagonal (Ports & Adapters) e desacoplamento por interfaces (ABC/ports), disciplina SQLAlchemy 2.0 async (select() tipado, zero SELECT *, valores só como parâmetro ligado, text() apenas com :param, domínio puro sem ORM/Pydantic), pirâmide de testes (unit com ports mockados + integração com testcontainers + injection tests), e gate ruff + mypy strict espelhando o CI. Carregue ao trabalhar em qualquer sub-projeto cuja stack seja Python (pyproject.toml / requirements presente). Despachada pelo workflow-dev; comandos concretos vêm do Project Profile. Execução de query/migration é delegada a expert-database.
---

# Expert Backend Python — Disciplina de Engenharia

Esta skill carrega **processo e disciplina transferível** para qualquer backend Python
(FastAPI/Litestar/Starlette + SQLAlchemy async + Pydantic v2), independentemente do produto.
Fatos concretos (paths, comandos exatos, nomes de tabela, nomes de função de negócio, TTL de
token, allowlist de CORS) vêm do Project Profile (`CLAUDE.md` / `AGENTS.md`) do projeto — nunca desta skill.

> **Escopo:** só engenharia (arquitetura, disciplina de acesso a dados, testes, lint/typecheck,
> segurança em princípio). A **execução** de query/migration (discovery de schema, gate de
> produção, ordem de aplicação de migration) é responsabilidade de `looptech:expert-database` —
> delegue para lá.

---

## 1. Arquitetura Clean + Hexagonal (Ports & Adapters)

- Separe o código em camadas explícitas com **a regra de dependência apontando só para dentro**:

  ```
  presentation  →  application  →  domain
                        ↓
                     ports  ←  infrastructure (implementa)
  ```

  | Camada | Responsabilidade | Pode importar de |
  |---|---|---|
  | `domain/` | Entidades, value objects, exceções de negócio | **nada** externo — só stdlib leve (dataclass/enum/Decimal/uuid/datetime) |
  | `application/` | Use cases, commands, queries — orquestra o domínio | `domain/`, `ports/` |
  | `ports/` | Contratos abstratos (`ABC`) | `domain/` |
  | `infrastructure/` | Implementações concretas (Postgres, HTTP externo, JWT, hashing) | `ports/`, libs externas |
  | `presentation/` | Routers HTTP, schemas Pydantic (DTOs), DI via `Depends()` | `application/`, schemas |

- **Desacople por interfaces (ports).** Toda dependência externa (repositório, gateway de
  pagamento, provedor de e-mail, fila, cliente HTTP externo) é consumida via uma `ABC` definida
  em `ports/` (convenção `I<Nome>`), com a implementação concreta (`<Provider><Nome>`) vivendo em
  `infrastructure/`. Isso permite trocar a implementação — ou mockar em teste — sem tocar a regra
  de negócio.

- **Injeção de dependência por construtor.** O use case recebe as ports de que precisa no
  `__init__` e **nunca instancia infra diretamente** nem lê de singleton global. O wiring concreto
  vive na camada de presentation (ex.: providers `get_*_use_case` num módulo de `dependencies.py`,
  entregues via `Depends()`).

  ```python
  # ports/repositories/user_repository.py
  class IUserRepository(ABC):
      @abstractmethod
      async def get_by_email(self, email: str) -> User | None: ...
      @abstractmethod
      async def save(self, user: User) -> User: ...

  # application/use_cases/auth/register.py
  class RegisterUseCase:
      def __init__(self, users: IUserRepository, hasher: IPasswordHasher) -> None:
          self._users = users
          self._hasher = hasher

      async def execute(self, email: str, password: str, name: str) -> User:
          if await self._users.get_by_email(email):
              raise UserAlreadyExistsError(email)
          user = User.create(email=email, name=name,
                             hashed_password=self._hasher.hash(password))
          return await self._users.save(user)
  ```

- **Domínio puro.** Nada de SQLAlchemy, Pydantic, FastAPI ou httpx dentro de `domain/` — isso vaza
  infra no core. O mapeamento entre o `Model` SQLAlchemy e a entidade de domínio é explícito
  (`Model.to_entity()` / `Model.from_entity(entity)`), na fronteira da infra.

- **Use case não conhece a casca web.** Nada de `Request`, `Depends`, `HTTPException` ou `status`
  dentro de `application/` — isso vaza FastAPI para a aplicação. O use case levanta **exceções de
  domínio tipadas** (`UserAlreadyExistsError`, `EntityNotFoundError`) e a borda HTTP as traduz em
  status code. Erro genérico (`raise Exception(...)`) espalhado impede tratamento diferenciado por
  camada (ex.: 404 vs 409 vs 500).

- **Um use case por arquivo.** Sem god-class `service.py` de 500 linhas. Schemas Pydantic são
  **DTOs de I/O** — sem regra de negócio dentro deles.

- **Async em toda a stack de I/O.** DB, HTTP externo, repositórios, use cases que fazem I/O são
  `async`. **Nunca** chame função síncrona bloqueante dentro de handler async (`time.sleep`,
  `requests.get`, driver DB síncrono) — o lint `ASYNC` do ruff existe para pegar isso.

---

## 2. Disciplina de acesso a dados — SQLAlchemy 2.0 async

Estas regras valem para todo código de repositório que compõe/executa SQL. São
**não-negociáveis** — todo PR que toca a camada de dados deve respeitá-las antes de merge.

### 2.1 Estilo 2.0 tipado, mapeamento model ↔ entidade

- Use a API 2.0: `select()` + `session.execute(stmt)` + `.scalar_one_or_none()` / `.scalars().all()`,
  models com `Mapped[...]` / `mapped_column(...)`. Para lookup por PK, `session.get(Model, id)`.
- O repositório recebe a `AsyncSession` no `__init__`, implementa a `ABC` do port, e **converte
  Model → entidade de domínio na saída** (nunca retorna o `Model` SQLAlchemy para cima — isso
  vazaria infra na aplicação).
- Caminho *not-found* retorna `None` (`row.to_entity() if row else None`), não levanta exceção — a
  decisão de virar 404 é da borda, não do repositório.

### 2.2 Valores só como parâmetro ligado — nunca interpolados

Todo valor de origem externa ou controlado pelo usuário é vinculado como **parâmetro**, nunca
concatenado/f-string dentro da string SQL. Com o construtor `select()`/`where()` isso é automático
(`where(Model.email == email)` gera `= :email`). Quando `text()` for realmente necessário:

- Use **parâmetros nomeados** (`:campo`) e passe os valores no `.bindparams()` / dict de execução —
  nunca `text(f"... {valor} ...")`.
- Para cast de tipo (ex.: `uuid`), use `CAST(:cid AS uuid)` e não `:cid::uuid` — o parser de
  `text()` lê `::uuid` como parte do nome do parâmetro.
- A **única** interpolação tolerável em SQL cru é de *identificador* fixo controlado por código
  (nome de tabela/coluna vindo de allowlist do próprio repositório) — nunca de valor. Se você está
  interpolando algo que veio do request, está errado.

```python
async def get_by_email(self, email: str) -> User | None:
    stmt = select(UserModel).where(UserModel.email == email)   # email → :param ligado
    row = (await self._session.execute(stmt)).scalar_one_or_none()
    return row.to_entity() if row else None
```

### 2.3 Zero `SELECT *` — colunas sempre explícitas

Prefira o `select(Model)` tipado (o mapper materializa só as colunas mapeadas). Em SQL cru via
`text()`, **liste as colunas** — nunca `SELECT *`. Motivo: um `SELECT *` acopla o schema físico ao
deploy — uma migration que adiciona coluna antes do código pode quebrar o scan de resultado, e
`SELECT *` mascara qual coluna cada leitura realmente depende.

### 2.4 Nullable é `Mapped[T | None]` — sem `COALESCE` em coluna indexada

- Campo nullable no model é `Mapped[T | None]` / `mapped_column(..., nullable=True)`; o binding
  mapeia `NULL → None` nativamente. Na entidade de domínio, o campo é `T | None`.
- **Não** envolva coluna nullable em `COALESCE` quando o campo já é opcional — além de redundante,
  `COALESCE` sobre coluna indexada impede o planner de usar o índice. `COALESCE` continua ok sobre
  agregação (`COALESCE(SUM(x), 0)`).

### 2.5 Fronteira de commit — repositório faz `flush()`, não `commit()`

- Repositórios fazem `session.add(...)` + `await session.flush()` para materializar dentro da
  transação corrente, mas **não** dão `commit()`. A fronteira de commit é única e vive na borda da
  request (dependency de sessão que faz um `commit` no teardown, ou um Unit of Work explícito).
- Ler um valor e decidir sobre ele **e** efetivar a mutação devem estar na **mesma transação**
  (mesma sessão) — nunca ler numa transação e escrever em outra. Um `commit()` intermediário
  escondido no meio de um fluxo é a origem clássica de corrida (ver §5).

### 2.6 Migration parity (execução delegada a `expert-database`)

- Toda coluna/tabela nova exige (a) o `Model` SQLAlchemy atualizado, (b) o `Model` **importado**
  no agregador de models (`models/__init__.py` + `__all__`) — sem isso o autogenerate do Alembic
  gera revisão vazia, e (c) a migration correspondente, tudo na **mesma PR**.
- Todo arquivo de migration autogerado é **revisado à mão** antes de aplicar: `upgrade()` e
  `downgrade()` ambos preenchidos, enums Postgres corretos, rename tratado como `alter` (não
  `drop+add`), `server_default` para coluna NOT NULL em tabela com dados, `ondelete=` explícito em
  FK, índices grandes com `concurrently` quando fizer sentido.
- A **execução** em si (ordem, gate de produção, `upgrade head`, resolução de múltiplos heads,
  expand→contract) é responsabilidade de `looptech:expert-database` — delegue.

### 2.7 Isolamento multi-tenant — métodos tenant-safe vs RAW

Quando o produto é multi-tenant, o repositório costuma expor dois grupos de método (o nome
concreto do tenant — `company`, `org`, `account` — vem do domínio do produto):

- **tenant-safe** (`get_by_id_for_<tenant>(id, tenant_id)`): filtra pelo tenant no `WHERE`;
  ID de outro tenant é indistinguível de *not-found* (retorna `None`). Use **sempre** que a entrada
  vem do request HTTP.
- **RAW/sistema** (`get_by_id(id)`): sem guarda de tenant — correto só para workers internos onde o
  escopo já é conhecido por contexto. **Nunca** chame um método RAW num router/use case
  tenant-facing sem uma checagem de ownership explícita **imediatamente após** a leitura (ver §5).

---

## 3. Pirâmide de testes

| Camada | Tipo | Ferramenta |
|---|---|---|
| Domínio + use cases | Unit | `pytest`, ports mockados (subclasse da ABC) |
| Use case / API | Matriz de casos | `pytest` parametrizado |
| Repositório | Integração | testcontainers + `alembic upgrade head` |
| Cada `/goal` de produto da API | E2E | `TestClient` / cliente HTTP async |
| HTTP externo | Unit/integração | `respx` |

- **Unit** — sem DB, sem rede. Mock = subclasse da `ABC`, não MagicMock solto.
- **Matriz de API obrigatória** (omite um caso só no `## /plan`): feliz;
  validação de cada campo; 401; 404 de outro tenant; not-found;
  conflito/idempotência; fronteira; corrida se dinheiro/estado. Ver
  `workflow-dev/references/verification.md`.
- **Integração** — um container por arquivo de suíte; write→read, nullable,
  not-found.
- **Injection** — `str` externo: busca não casa linha e não erra SQL; escrita
  persiste o payload literal.
- **E2E** — um teste HTTP por `/goal` de produto. Skip no caminho = vermelho.
- Primeira ação de impl: `## /plan`.
- Marque `@pytest.mark.unit/integration/e2e` com `--strict-markers`.

---

## 4. Lint + typecheck — espelhando o CI

O gate correto é o que o CI real roda, tipicamente agregado num alvo único (ex.: `make check` =
lint + typecheck + testes). Antes de considerar a task pronta, os três precisam imprimir verde.

- **`ruff check`** (lint) + **`ruff format`** (format) — o conjunto de regras (ex.: `E/F/W/I/B/UP/
  ASYNC/RUF`) e a `line-length` vêm da config do projeto (`pyproject.toml`). A regra `ASYNC` pega
  chamada bloqueante em código async; `B` (bugbear) pega armadilhas comuns. Não rebaixe uma regra
  no `pyproject` para "passar" — conserte o código.
- **`mypy`** em modo **strict** — sem `Any` implícito, sem `# type: ignore` **sem justificativa**
  inline. Type hints completos em toda função de produção (o relaxamento de tipagem em `tests/*` é
  aceitável e explícito na config, não uma desculpa para o código de produção).
- O **comando exato** (nome do alvo, se roda coverage, path base) vem do Project Profile — esta
  skill não hardcoda comando, só o princípio de que lint + typecheck + testes são um gate único e
  devem estar verdes antes do PR.

---

## 5. Segurança — em princípio (product-neutral)

Princípios de engenharia backend válidos para qualquer produto. Regras **específicas do produto**
(nome de lock, nome de variável de ambiente de scope, allowlist de CORS, papéis administrativos,
rate-limit) ficam no Project Profile do projeto, não nesta skill.

- **Ownership em todo acesso a recurso.** Toda rota que acessa um recurso por ID verifica que o
  dono é o usuário/tenant autenticado (ou que o chamador tem papel administrativo explícito) —
  nunca confiar que um ID "difícil de adivinhar" seja controle de acesso. Em falha, retorne
  **not-found** (`EntityNotFoundError` → 404), **não** 403 — 403 vaza a existência do recurso ao
  chamador errado. Se um método RAW (§2.7) foi usado, a guarda de ownership vem **na linha
  seguinte**, sem código de negócio no meio.

- **Serializar mutações críticas/financeiras com lock de banco.** Qualquer operação que lê um estado
  e decide sobre ele (cobrar, cancelar, estornar, alocar número sequencial) serializa por chave de
  negócio usando um lock transacional do Postgres — `pg_advisory_xact_lock(...)` (auto-liberado no
  commit/rollback) ou `SELECT ... FOR UPDATE [SKIP LOCKED]` — com **leitura e escrita na mesma
  transação/sessão**. Dois detalhes que quebram a garantia silenciosamente:
  - **Isolamento READ COMMITTED** (default do Postgres/engine) é premissa: a re-leitura pós-lock só
    enxerga o commit do vencedor porque cada statement pega um snapshot fresco. Subir o default para
    REPEATABLE READ/SERIALIZABLE reintroduz a corrida.
  - **Identity map do SQLAlchemy**: se a sessão já leu a linha antes do lock, faça `expire_all()` /
    `expire(obj)` após adquirir o lock, senão a re-leitura devolve o snapshot em cache (stale) em
    vez da linha commitada pelo vencedor.

- **Gatear endpoints internos/de debug por ambiente.** Rota que expõe schema interno, spec de API
  ou ferramental de debug é condicionada a uma variável de ambiente de escopo — nunca exposta por
  padrão em staging/produção.

- **Nunca confiar em `role`/papel vindo do input.** Campo de papel recebido em registro/atualização
  é validado contra uma allowlist fechada de papéis não-administrativos; contas privilegiadas são
  criadas fora desse fluxo.

- **Sem segredo hardcoded, sem PII em log.** `JWT_SECRET`, API keys, senhas, URLs de produção vêm
  de `Settings`/env (pydantic-settings), nunca do código. Não logue CPF/CNPJ/e-mail/dado fiscal —
  redija na borda de logging. Duas formas concretas: prefira logar o **ID** do registro (pseudônimo
  que leva ao dado sob acesso controlado) em vez do dado; e **nunca serialize entidade, DTO ou
  request inteiro** num log (`logger.info(f"{payload!r}")`, `model_dump()` direto) — o conjunto de
  campos cresce e o log nunca é revisitado. Implemente `__repr__` do tipo sensível já redigido, para
  que o erro deixe de ser possível em vez de depender de disciplina.

### 5.1 Superfície operacional

O serviço expõe mais do que as rotas de produto. Métricas, health checks, spec de API e sessões de
suporte são superfície real, e costumam ser revisadas com menos rigor justamente porque "não são
features".

- **Comparar segredo em tempo constante.** Token de scrape, assinatura de webhook e API key usam
  `hmac.compare_digest`, nunca `==`. A comparação de string do runtime retorna no primeiro byte
  divergente, e a diferença de tempo é mensurável pela rede — o bastante para recuperar o segredo
  byte a byte.
  ```python
  import hmac

  if not hmac.compare_digest(provided_token, expected_token):
      raise HTTPException(status_code=401)
  ```

- **Endpoint de telemetria é superfície pública quando exposto.** `/metrics` entrega nomes de rota,
  versão em execução, volume por endpoint e distribuição de latência — um mapa do sistema para quem
  estiver medindo. Quando a porta é alcançável fora da rede interna, exija autenticação, e mantenha
  a rota **fora de middlewares que alteram o corpo** (compressão, reescrita), que quebram scrapers.

- **Nunca use PII como label de métrica.** Além de vazar dado pessoal para um sistema sem controle
  de acesso equivalente, cada valor distinto cria uma série temporal nova — cardinalidade ilimitada
  que esgota o servidor de métricas. Use o **template** da rota (`request.scope["route"].path`),
  nunca o path concretizado com IDs, e um fallback fixo para requisição sem rota casada.

- **Gate que degrada aberto precisa falhar o boot.** É comum — e conveniente — um middleware de
  proteção virar no-op quando a variável correspondente está vazia, para não atrapalhar o ambiente
  local. O risco é que um deploy com a variável faltando **não produz erro nenhum**: o serviço sobe
  saudável e a proteção não existe. Se adotar essa forma, valide em `Settings` na inicialização e
  **recuse subir** fora do ambiente local sem o segredo — um validator do pydantic-settings resolve.

- **Acesso a dado de cliente por operador é somente leitura e auditado.** Impersonação e ferramenta
  administrativa de suporte restringem-se a métodos de leitura — escrita impersonada destrói a
  confiabilidade do histórico da conta como prova do que o cliente fez. E **toda** requisição
  impersonada gera registro de auditoria (quem, sobre quem, quando, o quê), não só as negadas: é o
  que permite responder quem da equipe acessou os dados de um titular. O log de auditoria é separado
  do operacional, com retenção maior e acesso mais restrito.

---

## 6. Delegação — o que NÃO é desta skill

- **Execução de query/migration** (discovery de schema ao vivo, preflight de conexão, gate de
  produção, ordem de aplicação por existência, resolução de heads) → `looptech:expert-database`.
- **Fatos concretos** (comandos exatos de test/lint/typecheck, nome de conexão de banco, convenção
  de branch/PR, TTL de token, `specs_dir`) → Project Profile no `CLAUDE.md` / `AGENTS.md`,
  resolvido pelo `looptech:workflow-dev`.
- **Regras de segurança específicas do produto** (nome de lock/função, allowlist de CORS concreta,
  papéis administrativos, gating de endpoint sensível) → Project Profile do projeto.

---

## Checklist de PR (engenharia backend Python)

- [ ] Regra de dependência respeitada — `domain/` puro (sem SQLAlchemy/Pydantic/FastAPI), `application/` só conhece `domain/` + `ports/`
- [ ] Dependência externa desacoplada por `ABC` (port); use case recebe a port no `__init__`, DI via `Depends()` no presentation
- [ ] Model ↔ entidade convertidos na fronteira da infra — repositório não vaza `Model` para cima
- [ ] Todo valor vai como parâmetro ligado — nenhuma interpolação/f-string de valor em SQL; `text()` só com `:param`
- [ ] Zero `SELECT *` — `select(Model)` tipado ou colunas explícitas
- [ ] Campo nullable é `Mapped[T | None]`; sem `COALESCE` em coluna indexada
- [ ] Repositório faz `flush()`, não `commit()`; leitura+escrita de decisão na mesma transação
- [ ] Coluna/tabela nova: `Model` atualizado **e** importado no agregador **e** migration na mesma PR (execução delegada a `expert-database`)
- [ ] Entrada tenant-facing usa método tenant-safe (`*_for_<tenant>`); método RAW só com guarda de ownership na linha seguinte
- [ ] Unit test do use case roda sem DB e sem HTTP (ports mockados como subclasse da `ABC`)
- [ ] Integração cobre write→read, nullable, not-found — testcontainers + `alembic upgrade head`
- [ ] Injection test cobre todo parâmetro `str` externo de repositório
- [ ] Matriz de API + E2E de cada `/goal` de produto passam, sem skip
- [ ] Mutação financeira/crítica serializada por lock de banco, na mesma transação
- [ ] Comparação de segredo usa `hmac.compare_digest`, não `==`
- [ ] Endpoint de telemetria autenticado quando alcançável fora da rede interna, e fora de middleware que altera o corpo
- [ ] Nenhuma PII como label de métrica; label de rota usa o template, não o path com IDs
- [ ] Gate que degrada aberto valida em `Settings` no boot e recusa subir sem o segredo fora do local
- [ ] Impersonação/ferramenta de suporte é somente leitura e audita toda requisição
- [ ] Nenhum log serializa entidade/DTO/request inteiro; PII redigida na borda de logging
- [ ] `ruff check` + `ruff format` + `mypy` strict verdes; sem `# type: ignore` sem justificativa

## Red flags

- `import` da camada de `infrastructure` dentro de `application/` ou `domain/`
- SQLAlchemy/Pydantic/`Request`/`Depends` dentro de `domain/` ou `application/`
- Valor de runtime interpolado (f-string/`+`) dentro de string SQL; `text(f"...")`
- `SELECT *` em qualquer repositório
- Repositório dando `commit()` no meio de um fluxo
- Método de repositório novo sem integração; `str` externo sem injection test
- Mutação financeira fora de lock, ou lock sem `expire_all()` na re-leitura
- Método RAW em fluxo tenant-facing sem guarda de ownership
- Chamada síncrona bloqueante dentro de handler async
- Use case importando implementação concreta em vez da `ABC`
- Segredo comparado com `==` em vez de `hmac.compare_digest`
- Middleware de proteção que vira no-op com env vazia, sem validação de boot
- Escrita permitida em sessão impersonada, ou impersonação sem log de auditoria
- `model_dump()`/`repr` de entidade, DTO ou erro cru de driver caindo em log
- `# type: ignore` / `Any` sem justificativa; regra de ruff rebaixada para "passar"
