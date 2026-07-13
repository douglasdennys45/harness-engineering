# Arquitetura da Aplicacao

Este documento define a arquitetura do projeto, suas camadas, regras de dependencia, convencoes de nomenclatura e o guia passo a passo para adicionar novas features.

## Visao Geral

O projeto segue **Clean Architecture** com inversao de dependencia em uma estrutura de **monorepo** com multiplos microservicos. Camadas internas nunca importam camadas externas. Injecao de dependencia e gerenciada pelo **dependency-injector** (containers declarativos, injecao por constructor com argumentos nomeados). O monorepo suporta **N apps independentes** (APIs e Consumers) que compartilham bibliotecas via `pkg/` (workspace member `org-pkg`, modulo `org_pkg`).

```
apps/<app-name>/src/<app>/bin/<binary>.py   Entry points (um por binario)
       |
       v
apps/<app-name>/src/<app>/domain/           Entidades, Protocols (sem dependencias externas)
apps/<app-name>/src/<app>/application/      Implementacoes de use cases (depende apenas de domain)
apps/<app-name>/src/<app>/infrastructure/   Controllers, repos, publisher, subscriber, config
       |
       v
pkg/                                        Bibliotecas reutilizaveis compartilhadas entre apps
```

### Stack Tecnologica

| Camada | Tecnologia | Versao |
|---|---|---|
| Runtime | Python | **3.13+** |
| HTTP | Robyn (runtime Rust / tokio) | `robyn` **0.86+** |
| Banco de Dados | PostgreSQL (via `asyncpg`, SQL manual sem ORM) | **18** |
| Mensageria | NATS JetStream (via `nats-py`) | **2.12** |
| DI | dependency-injector (DeclarativeContainer, constructor injection) | latest |
| Logging | structlog (JSON estruturado) | latest |
| Validacao | Pydantic | **v2** |
| Documentacao | OpenAPI nativa do Robyn (`/docs`, `/openapi.json`) | - |
| Config | python-dotenv + variaveis de ambiente | - |
| Migrations | dbmate (SQL puro) | latest |
| Testes | pytest + pytest-asyncio (unit), testcontainers (integracao), mutmut (mutation) | latest |
| Typecheck | mypy (`--strict`) | latest |
| Lint / Format | ruff (`ruff check` + `ruff format`) | latest |
| Monorepo | uv workspaces | latest |

### Tipos de Binarios

| Tipo | Proposito | Exemplo |
|---|---|---|
| **API** | Servidor HTTP expondo endpoints REST (Robyn) | `apps/user/src/user/bin/api.py` |
| **Consumer** | Processo em background consumindo eventos NATS JetStream (asyncio puro, sem Robyn) | `apps/user/src/user/bin/consumer.py` |

Em desenvolvimento a API roda com hot reload: `uv run robyn apps/user/src/user/bin/api.py --dev`. Em producao: `uv run python -m user.bin.api` (1 processo por replica de container; escala horizontal via replicas — ver secao de escala do Robyn na skill `robyn-best-practices`).

### Regra de Dependencia

```
Domain  <--  Application  <--  Infrastructure  <--  src/<app>/bin/
  ^                                  |
  |                                  v
  +--- nunca depende de -------->  pkg/
```

- `domain/` nao importa nenhuma biblioteca externa — apenas stdlib (`dataclasses`, `datetime`, `typing`). Excecao documentada: `if TYPE_CHECKING: from asyncpg import Connection` no `Transactor` (import somente de tipo, sem codigo em runtime — ver secao de transactions).
- `application/` importa apenas `domain/`.
- `infrastructure/` importa `domain/`, `org_pkg` e bibliotecas externas.
- `pkg/` nunca importa `src/` de nenhum app.
- `src/<app>/bin/<binary>.py` importa tudo para compor o container dependency-injector **daquele binario especifico** (composition root).

---

## Estrutura de Diretorios

```
/
├── pyproject.toml                                   # workspace root (uv) + config de ruff/mypy/pytest
├── uv.lock
├── Makefile                                         # gates raiz: typecheck, lint, format-check, test
├── docker-compose.yml
│
├── pkg/                                             # workspace member: org-pkg (modulo org_pkg)
│   ├── pyproject.toml
│   └── src/org_pkg/
│       ├── __init__.py
│       ├── postgres/
│       │   ├── conn.py                              # Database (wrapper do pool asyncpg)
│       │   ├── tx.py                                # run_in_tx helper
│       │   └── health.py                            # ping health check
│       ├── nats/
│       │   ├── conn.py                              # NatsClient (wrapper de conexao + reconnect)
│       │   ├── stream.py                            # ensure_stream
│       │   ├── publisher.py                         # Publisher generico
│       │   └── subscriber.py                        # subscribe com consumer duravel (pull)
│       ├── controller/
│       │   ├── base_controller.py                   # BaseController (template method)
│       │   ├── request_context.py                   # RequestContext
│       │   ├── validation.py                        # format_validation_error + parse_query (Pydantic)
│       │   ├── responses.py                         # json_response (Response JSON padronizada)
│       │   ├── errors.py                            # HttpError, ValidationError
│       │   └── error_mapper.py                      # ErrorMapper
│       ├── logger/
│       │   └── logger.py                            # structlog JSON setup com service e version
│       └── event/                                   # contratos de eventos compartilhados
│           ├── envelope.py                          # Envelope base (Pydantic)
│           ├── user.py                              # subjects + data models
│           └── billing.py                           # subjects + data models
│
├── migrations/                                      # migrations SQL por app (dbmate)
│   ├── user/
│   │   └── 0001_create_users.sql                    # -- migrate:up / -- migrate:down
│   └── billing/
│       └── 0001_create_accounts.sql
│
├── apps/
│   ├── user/                                        # workspace member: org-user (modulo user)
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── src/user/
│   │   │   ├── __init__.py
│   │   │   ├── bin/
│   │   │   │   ├── api.py                           # API HTTP — composition root (Robyn)
│   │   │   │   └── consumer.py                      # Consumer NATS — composition root (asyncio)
│   │   │   ├── domain/
│   │   │   │   ├── entity/
│   │   │   │   │   ├── user.py                      # Dataclass de dominio + classmethod create
│   │   │   │   │   └── errors.py                    # Classes de erro de dominio
│   │   │   │   ├── usecase/
│   │   │   │   │   └── user/                        # Subpasta por contexto de dominio
│   │   │   │   │       ├── create.py                # CreateUseCase (Protocol)
│   │   │   │   │       ├── get_by_id.py             # GetByIdUseCase (Protocol)
│   │   │   │   │       └── list.py                  # ListUseCase (Protocol)
│   │   │   │   ├── repository/
│   │   │   │   │   └── user_repository.py           # Protocol de repositorio
│   │   │   │   ├── event/
│   │   │   │   │   └── user_event.py                # Protocol de eventos/mensageria
│   │   │   │   └── <lib>/                           # Protocols para libs externas
│   │   │   │       └── <name>.py                    # Interface de inversao de dependencia
│   │   │   ├── application/
│   │   │   │   └── usecase/
│   │   │   │       └── user/                        # Subpasta por contexto de dominio
│   │   │   │           ├── create_usecase.py        # Implementacao do use case
│   │   │   │           ├── get_by_id_usecase.py
│   │   │   │           └── list_usecase.py
│   │   │   └── infrastructure/
│   │   │       ├── config/
│   │   │       │   ├── config.py                    # AppConfig + load_config
│   │   │       │   └── container.py                 # InfraContainer (providers de infra compartilhada)
│   │   │       ├── controller/
│   │   │       │   └── user_controller.py           # Controllers HTTP (Robyn) — apenas APIs
│   │   │       ├── repository/
│   │   │       │   └── user_repository.py           # Implementacao PostgreSQL (asyncpg)
│   │   │       ├── publisher/
│   │   │       │   └── user_publisher.py            # Implementacao NATS JetStream
│   │   │       ├── subscriber/
│   │   │       │   └── user_subscriber.py           # Consumer NATS — apenas Consumers
│   │   │       └── adapter/
│   │   │           └── <lib>_adapter.py             # Implementacoes de Protocols de libs externas
│   │   └── tests/
│   │       ├── fakes/                               # Fakes tipados das interfaces de dominio
│   │       ├── unit/                                # Testes unitarios (pytest, espelham src/)
│   │       │   └── application/usecase/user/
│   │       │       └── test_create_usecase.py
│   │       └── integration/
│   │           ├── testhelper/
│   │           │   ├── postgres.py                  # Container PG compartilhado (testcontainers)
│   │           │   ├── nats.py                      # Container NATS compartilhado
│   │           │   └── api.py                       # Sobe a API real em porta livre (subprocess)
│   │           └── test_user_repository.py
│   │
│   └── billing/                                     # mesma estrutura interna (org-billing, modulo billing)
│       ├── pyproject.toml
│       ├── Dockerfile
│       ├── src/billing/
│       │   ├── bin/
│       │   ├── domain/
│       │   ├── application/
│       │   └── infrastructure/
│       └── tests/
```

### Workspace uv (`pyproject.toml` raiz)

```toml
[project]
name = "org-project"
version = "0.0.0"
requires-python = ">=3.13"

[tool.uv.workspace]
members = ["pkg", "apps/*"]

[tool.uv.sources]
org-pkg = { workspace = true }

[dependency-groups]
dev = [
    "mypy",
    "ruff",
    "pytest",
    "pytest-asyncio",
    "pytest-cov",
    "testcontainers[postgres]",
    "mutmut",
    "httpx",
]

[tool.ruff]
line-length = 100
src = ["pkg/src", "apps/*/src"]

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "N", "ASYNC", "RUF"]

[tool.mypy]
strict = true
python_version = "3.13"

[tool.pytest.ini_options]
asyncio_mode = "auto"
markers = ["integration: testes que sobem containers reais"]
addopts = "-m 'not integration'"
```

**`Makefile` raiz** — alvos que servem de **gates de validacao** para todo o monorepo:

```makefile
.PHONY: typecheck lint format format-check test test-integration check

typecheck:
	uv run mypy .

lint:
	uv run ruff check .

format:
	uv run ruff format .

format-check:
	uv run ruff format --check .

test:
	uv run pytest

test-integration:
	uv run pytest -m integration

check: typecheck lint format-check test
```

| Gate | Comando | O que valida |
|---|---|---|
| Typecheck | `make typecheck` | `mypy --strict` em todos os workspaces |
| Lint | `make lint` | `ruff check` (E, F, I, B, UP, N, ASYNC, RUF) |
| Formatacao | `make format-check` | `ruff format --check` |
| Testes | `make test` | `pytest` (unitarios; integracao via `make test-integration`) |
| Lockfile | `uv sync --locked` | Integridade do `uv.lock` |

**`pkg/pyproject.toml`** — biblioteca compartilhada:

```toml
[project]
name = "org-pkg"
version = "0.0.0"
requires-python = ">=3.13"
dependencies = [
    "robyn>=0.86",
    "nats-py>=2",
    "asyncpg>=0.30",
    "structlog>=24",
    "pydantic>=2",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/org_pkg"]
```

Cada app declara `org-pkg` como dependencia (resolvida via `[tool.uv.sources]` do workspace):

```toml
# apps/user/pyproject.toml
[project]
name = "org-user"
version = "0.0.0"
requires-python = ">=3.13"
dependencies = [
    "org-pkg",
    "robyn>=0.86",
    "asyncpg>=0.30",
    "nats-py>=2",
    "pydantic>=2",
    "structlog>=24",
    "dependency-injector>=4.42",
    "python-dotenv>=1",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/user"]
```

**Regras do workspace:**
- Layout `src/` obrigatorio em todo member (`pkg/src/org_pkg/`, `apps/user/src/user/`) — evita imports acidentais do diretorio de trabalho.
- **Imports absolutos sempre**: `from user.domain.entity.user import User`, `from org_pkg.postgres import Database`. Nunca imports relativos entre camadas (`from ...domain import`) — o ruff (`TID`) pode reforcar.
- Um unico ambiente virtual na raiz (`uv sync` instala todos os members em editable) — `uv run` na raiz enxerga todos os modulos.
- `uv sync --locked` em CI garante que o `uv.lock` esta consistente com os `pyproject.toml`.
- Todos os modulos publicos com type hints completos — `mypy --strict` e gate.

### Convencao de Nomes para `apps/` e `src/<app>/bin/`

- **Apps**: `apps/<app-name>/` — lowercase (ex: `apps/user/`, `apps/billing/`); pacote Python homonimo em `src/<app-name>/`.
- **APIs**: `apps/<app>/src/<app>/bin/api.py`.
- **Consumers**: `apps/<app>/src/<app>/bin/consumer.py`. Se houver multiplos consumers, usar `bin/consumer_<dominio>.py`.
- Cada arquivo em `bin/` e um **composition root completo**: carrega config, monta o container dependency-injector, inicia o processo e trata shutdown. Nenhuma logica de negocio.

---

## Camada de Dominio (`src/<app>/domain/`)

A camada de dominio contem **entidades** e **contratos** (Protocols). Nao possui dependencias externas — apenas Python puro e stdlib (`dataclasses`, `datetime`, `uuid` quando estritamente necessario). **Isolada dentro de cada app.**

### entity/

Dataclasses representando conceitos de dominio. Cada entidade e um **`@dataclass`** com um classmethod `create(...)` para novas instancias e o constructor gerado (usado para rehidratacao a partir do banco).

**Padrao:**

```python
# apps/user/src/user/domain/entity/user.py
from dataclasses import dataclass
from datetime import UTC, datetime


@dataclass
class User:
    id: str
    name: str
    email: str
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create(cls, name: str, email: str) -> "User":
        now = datetime.now(UTC)
        return cls(
            id="",  # gerado pelo banco via INSERT ... RETURNING "id"
            name=name,
            email=email,
            created_at=now,
            updated_at=now,
        )
```

**Regras:**
- Arquivo: `<name>.py` em snake_case (ex: `audit_log.py`).
- Classe exportada em PascalCase; atributos em snake_case (a serializacao camelCase acontece na borda HTTP, via Pydantic — ver controllers).
- `@classmethod create(...)` para criar novas instancias com regras de negocio; o constructor do dataclass para rehidratar do banco.
- Timestamps sempre em UTC: `datetime.now(UTC)` — datetimes **sempre aware** (`TIMESTAMPTZ` no banco devolve aware via asyncpg); nunca `datetime.utcnow()` (naive, deprecated).
- Sem logica de infraestrutura (sem imports de drivers, frameworks ou bibliotecas).
- Metodos de negocio da entidade (ex: `cancel()`) vivem na propria classe e lancam erros de dominio.

**Erros de dominio** (cada app define os seus em `entity/errors.py`):

```python
# apps/user/src/user/domain/entity/errors.py


class DomainError(Exception):
    """Classe base para todos os erros de dominio do app."""


class InvalidInputError(DomainError):
    def __init__(self, message: str = "invalid input") -> None:
        super().__init__(message)


class UserNotFoundError(DomainError):
    def __init__(self, message: str = "user not found") -> None:
        super().__init__(message)


class UserAlreadyExistsError(DomainError):
    def __init__(self, message: str = "email already in use") -> None:
        super().__init__(message)


class InvalidRoleError(DomainError):
    def __init__(self, message: str = "invalid role") -> None:
        super().__init__(message)


class ForbiddenError(DomainError):
    def __init__(self, message: str = "forbidden") -> None:
        super().__init__(message)


# Para manter rastreabilidade, encadear a causa original com `from`:
#   raise UserNotFoundError(f"user {user_id} not found")
#   raise RuntimeError("create user failed") from err
# O ErrorMapper usa isinstance() para mapear a classe -> HTTP status.
```

### usecase/

Protocols definindo os use cases de dominio. Cada use case e representado por **um unico Protocol com um unico metodo `perform`**. Protocols sao organizados em **subpastas por contexto de dominio**.

**Principio:** 1 acao = 1 Protocol = 1 arquivo. Isso garante interfaces coesas, facilita testes (fakes/mocks granulares) e respeita o Interface Segregation Principle.

**Estrutura de diretorios:**

```
src/user/domain/usecase/
└── user/
    ├── create.py             # CreateUseCase (Protocol)
    ├── get_by_id.py          # GetByIdUseCase (Protocol)
    └── list.py               # ListUseCase (Protocol)
```

**Padrao:**

```python
# apps/user/src/user/domain/usecase/user/create.py
from dataclasses import dataclass
from typing import Protocol

from user.domain.entity.user import User


@dataclass(frozen=True)
class CreateInput:
    name: str
    email: str


@dataclass(frozen=True)
class CreateOutput:
    user: User


class CreateUseCase(Protocol):
    async def perform(self, input: CreateInput) -> CreateOutput: ...
```

```python
# apps/user/src/user/domain/usecase/user/get_by_id.py
from dataclasses import dataclass
from typing import Protocol

from user.domain.entity.user import User


@dataclass(frozen=True)
class GetByIdInput:
    id: str


class GetByIdUseCase(Protocol):
    async def perform(self, input: GetByIdInput) -> User | None: ...
```

```python
# apps/user/src/user/domain/usecase/user/list.py
from dataclasses import dataclass
from typing import Protocol

from user.domain.entity.user import User


@dataclass(frozen=True)
class ListInput:
    cursor: str
    limit: int


@dataclass(frozen=True)
class ListOutput:
    users: list[User]
    next_cursor: str


class ListUseCase(Protocol):
    async def perform(self, input: ListInput) -> ListOutput: ...
```

**Regras:**
- Organizacao: `domain/usecase/<contexto>/<acao>.py` (ex: `domain/usecase/user/create.py`).
- Cada arquivo contem **1 Protocol** com **1 metodo**: `perform`, sempre `async`.
- Protocol exportado: `<Action>UseCase` (ex: `CreateUseCase`, `GetByIdUseCase`).
- O metodo `perform` recebe **um unico parametro** `input` (dataclass frozen) e retorna o output.
- Dataclasses de input (`<Action>Input`) e output (`<Action>Output`) sao definidas no mesmo arquivo do Protocol, sempre `frozen=True`.
- Retorna entidades de dominio, outputs e/ou lanca erros de dominio. Ausencia e representada por `None`.

### repository/

Protocols definindo contratos de persistencia.

**Padrao:**

```python
# apps/user/src/user/domain/repository/user_repository.py
from typing import Protocol

from user.domain.entity.user import User


class UserRepository(Protocol):
    async def create(self, user: User) -> None: ...
    async def find_by_id(self, id: str) -> User | None: ...
    async def find_by_email(self, email: str) -> User | None: ...
    async def list(self, cursor: str, limit: int) -> tuple[list[User], str]: ...
    async def update(self, user: User) -> None: ...
    async def delete(self, id: str) -> None: ...
```

**Regras:**
- Arquivo: `<name>_repository.py` (ex: `user_repository.py`).
- Protocol exportado: `<Name>Repository`.
- Todos os metodos sao `async`.
- `find_by_id` retorna `User | None` — retorna `None` quando nao encontrado (ausencia nao e erro).

### event/

Protocols definindo contratos de mensageria/eventos.

**Padrao:**

```python
# apps/user/src/user/domain/event/user_event.py
from typing import Protocol

from user.domain.entity.user import User


class UserEvent(Protocol):
    async def publish_created(self, user: User) -> None: ...
    async def publish_updated(self, user: User) -> None: ...
```

**Regras:**
- Arquivo: `<name>_event.py` (ex: `user_event.py`).
- Protocol exportado: `<Name>Event`.
- Metodos nomeados por acao: `publish_<action>`.

### Protocols para Bibliotecas Externas (`domain/<lib>/`)

Toda biblioteca externa utilizada no projeto **deve ser acessada via inversao de dependencia**. O Protocol e definido na camada de dominio em uma subpasta nomeada pelo conceito/capacidade que a lib fornece. A implementacao concreta fica em `infrastructure/adapter/`.

**Principio:** O dominio nunca conhece a lib concreta. Ele define **o que precisa** (Protocol), e a infraestrutura fornece **como fazer** (implementacao com a lib).

**Estrutura de diretorios:**

```
src/user/domain/
├── crypt/
│   └── hasher.py              # Protocol para hashing de senhas
├── token/
│   └── manager.py             # Protocol para geracao/validacao de tokens
└── mailer/
    └── sender.py              # Protocol para envio de emails
```

**Padrao:**

```python
# apps/user/src/user/domain/crypt/hasher.py
from typing import Protocol


class Hasher(Protocol):
    async def hash(self, plain: str) -> str: ...
    async def compare(self, hashed: str, plain: str) -> bool: ...
```

**Regras:**
- Subpasta nomeada pelo **conceito/capacidade**, nao pelo nome da lib (ex: `crypt/`, nao `bcrypt/`; `token/`, nao `jwt/`).
- Protocol exportado com nome descritivo da capacidade (ex: `Hasher`, `Manager`, `Sender`).
- **Nenhum import de lib externa** — apenas Python puro.
- As implementacoes concretas ficam em `infrastructure/adapter/`.

---

## Camada de Aplicacao (`src/<app>/application/`)

Contem a **implementacao** dos use cases. Orquestra entidades, repositorios e eventos. **Isolada dentro de cada app.**

### usecase/

Implementacoes dos use cases, organizadas em **subpastas por contexto de dominio**, espelhando a estrutura de `domain/usecase/`.

**Estrutura de diretorios:**

```
src/user/application/usecase/
└── user/
    ├── create_usecase.py           # CreateUseCase class + perform
    ├── get_by_id_usecase.py
    └── list_usecase.py

tests/unit/application/usecase/user/
└── test_create_usecase.py          # Teste unitario (pytest + fakes)
```

**Padrao:**

```python
# apps/user/src/user/application/usecase/user/create_usecase.py
import contextlib

from user.domain.entity.errors import UserAlreadyExistsError
from user.domain.entity.user import User
from user.domain.event.user_event import UserEvent
from user.domain.repository.user_repository import UserRepository
from user.domain.usecase.user.create import CreateInput, CreateOutput


class CreateUseCase:
    def __init__(self, user_repository: UserRepository, user_event: UserEvent) -> None:
        self._user_repository = user_repository
        self._user_event = user_event

    async def perform(self, input: CreateInput) -> CreateOutput:
        existing = await self._user_repository.find_by_email(input.email)
        if existing is not None:
            raise UserAlreadyExistsError()

        user = User.create(input.name, input.email)

        await self._user_repository.create(user)

        # publica evento DEPOIS do commit no banco.
        # falha na publicacao nao desfaz a operacao de negocio.
        with contextlib.suppress(Exception):
            await self._user_event.publish_created(user)

        return CreateOutput(user=user)
```

**Regras:**
- Organizacao: `application/usecase/<contexto>/<acao>_usecase.py`.
- Classe: `<Action>UseCase` (mesmo nome do Protocol, sem sufixo `Impl`). Protocols sao **estruturais** — a classe satisfaz o contrato sem herdar dele; o mypy verifica a conformidade no ponto de injecao.
- Dependencias entram **pelo constructor** com nomes que coincidem com os providers do container (`user_repository`, `user_event`, `hasher`, ...), guardadas como atributos privados (`self._user_repository`).
- Dependencias sao **Protocols de dominio** (nunca tipos concretos).
- O metodo implementado e sempre `async def perform(self, input) -> output`.
- Depende apenas de `domain/`. Nunca importa `infrastructure/` ou `org_pkg`.
- Erros re-lancados com contexto usando `raise ... from err` — ou simplesmente propagados (excecoes ja propagam).
- Evento publicado **apos** persistencia no banco, nunca antes. Falha de publicacao e suprimida (com log na implementacao do publisher), nunca desfaz o commit.

---

## Camada de Infraestrutura (`src/<app>/infrastructure/`)

Implementa contratos de dominio usando tecnologias concretas e gerencia a configuracao da aplicacao. Cada binario conecta apenas o que precisa.

### controller/ (apenas APIs)

Controllers HTTP usando Robyn. O projeto utiliza um **BaseController** em `pkg/controller/` que implementa o template method pattern: extrai dados comuns da request (request_id, IP, headers), executa validacao e delega para o metodo concreto. Cada controller **estende** o `BaseController` e so implementa a logica especifica. **Usado apenas por binarios de API.**

#### BaseController — `pkg/controller/` (compartilhado entre todos os apps)

O `BaseController` vive no `pkg/` porque e infraestrutura generica **sem nenhuma logica de dominio**. Todos os apps reutilizam a mesma base, evitando duplicacao.

Ele centraliza:
- Extracao automatica de `request_id`, `parent_id`, `ip`, `user_agent`.
- Bind + validacao do body (Pydantic) com resposta padronizada.
- Tratamento uniforme de erros (dominio -> HTTP status via `ErrorMapper`).
- Logging estruturado com contexto da request (logger structlog com bind).

**Estrutura do `pkg/controller/`:**

```
pkg/src/org_pkg/controller/
├── base_controller.py      # BaseController + handle/bind_and_handle (template method)
├── request_context.py      # RequestContext dataclass
├── validation.py           # format_validation_error + parse_query
├── responses.py            # json_response
├── errors.py               # HttpError, ValidationError
└── error_mapper.py         # ErrorMapper
```

**`pkg/controller/request_context.py` — RequestContext:**

```python
# pkg/src/org_pkg/controller/request_context.py
from dataclasses import dataclass

from structlog.stdlib import BoundLogger


@dataclass(frozen=True)
class RequestContext:
    """Dados extraidos automaticamente de toda request.

    Handlers recebem este objeto ja preenchido, sem precisar extrair manualmente.
    """

    request_id: str
    parent_id: str
    ip: str
    user_agent: str
    logger: BoundLogger
```

**`pkg/controller/errors.py` — Erros HTTP genericos:**

```python
# pkg/src/org_pkg/controller/errors.py


class HttpError(Exception):
    """Erro com status HTTP explicito (equivalente a um 'erro de framework')."""

    def __init__(self, status: int, message: str) -> None:
        super().__init__(message)
        self.status = status


class ValidationError(HttpError):
    """Erro de validacao de input (bind do body ou query). Sempre vira 400."""

    def __init__(self, message: str) -> None:
        super().__init__(400, message)
```

**`pkg/controller/responses.py` — resposta JSON padronizada:**

```python
# pkg/src/org_pkg/controller/responses.py
import json
from datetime import datetime
from typing import Any
from uuid import UUID

from robyn import Headers, Response


def _default(value: Any) -> str:
    if isinstance(value, datetime):
        return value.isoformat()
    if isinstance(value, UUID):
        return str(value)
    raise TypeError(f"not json serializable: {type(value)!r}")


def json_response(status_code: int, data: Any) -> Response:
    """Serializa `data` como JSON e monta a Response do Robyn."""
    return Response(
        status_code=status_code,
        headers=Headers({"Content-Type": "application/json"}),
        description=json.dumps(data, default=_default),
    )
```

**`pkg/controller/base_controller.py` — BaseController (template method):**

```python
# pkg/src/org_pkg/controller/base_controller.py
import functools
from collections.abc import Awaitable, Callable
from uuid import uuid4

from pydantic import BaseModel, ValidationError as PydanticValidationError
from robyn import Request, Response
from structlog.stdlib import BoundLogger

from org_pkg.controller.error_mapper import ErrorMapper
from org_pkg.controller.errors import ValidationError
from org_pkg.controller.request_context import RequestContext
from org_pkg.controller.responses import json_response
from org_pkg.controller.validation import format_validation_error

# HandlerFn e a assinatura que handlers concretos implementam.
HandlerFn = Callable[[Request, RequestContext], Awaitable[Response]]
RouteHandler = Callable[[Request], Awaitable[Response]]


class BaseController:
    """Centraliza extracao de contexto, logging e error handling.

    Todo controller do projeto estende esta classe.
    """

    def __init__(self, error_mapper: ErrorMapper, logger: BoundLogger) -> None:
        self._error_mapper = error_mapper
        self._logger = logger

    def handle(self, handler: HandlerFn) -> RouteHandler:
        """Template method: extrai o contexto comum e delega para o handler concreto.

        Todo handler do projeto deve ser registrado via este metodo (ou bind_and_handle).
        """

        @functools.wraps(handler)  # preserva nome/docstring para o OpenAPI do Robyn
        async def wrapper(request: Request) -> Response:
            rc = self._extract_context(request)
            try:
                return await handler(request, rc)
            except Exception as err:  # noqa: BLE001 — boundary HTTP: mapeia e responde
                return self._handle_error(rc, err)

        return wrapper

    def bind_and_handle[T: BaseModel](
        self,
        model: type[T],
        handler: Callable[[Request, RequestContext, T], Awaitable[Response]],
    ) -> RouteHandler:
        """Faz parse do JSON + validacao Pydantic do body antes de chamar o handler.

        O handler so e executado se o body for valido — recebe `input` ja tipado.
        """

        @functools.wraps(handler)
        async def wrapper(request: Request) -> Response:
            rc = self._extract_context(request)
            try:
                try:
                    payload = request.json()
                except Exception as err:
                    raise ValidationError("invalid json body") from err

                try:
                    input_ = model.model_validate(payload)
                except PydanticValidationError as err:
                    # erro de parse OU de validacao — ambos retornam 400
                    raise ValidationError(format_validation_error(err)) from err

                return await handler(request, rc, input_)
            except Exception as err:  # noqa: BLE001
                return self._handle_error(rc, err)

        return wrapper

    def _extract_context(self, request: Request) -> RequestContext:
        request_id = str(uuid4())
        parent_id = request.headers.get("x-request-id") or ""

        logger = self._logger.bind(
            request_id=request_id,
            parent_id=parent_id,
            method=request.method,
            route=request.url.path,
        )

        return RequestContext(
            request_id=request_id,
            parent_id=parent_id,
            ip=request.ip_addr or "",
            user_agent=request.headers.get("user-agent") or "",
            logger=logger,
        )

    def _handle_error(self, rc: RequestContext, err: Exception) -> Response:
        status, message = self._error_mapper.map(err)

        if status >= 500:
            rc.logger.error("request failed", status_code=status, error=str(err))
        else:
            rc.logger.warning("request rejected", status_code=status, error=message)

        return json_response(status, {"error": message, "status": status})
```

**`pkg/controller/validation.py` — formatacao de erros Pydantic + query params:**

O projeto utiliza **Pydantic v2** como engine de validacao. Todo body e validado com `model_validate` dentro do `bind_and_handle` — o handler nunca ve input invalido.

```python
# pkg/src/org_pkg/controller/validation.py
from pydantic import BaseModel, ValidationError as PydanticValidationError
from robyn import Request

from org_pkg.controller.errors import ValidationError


def format_validation_error(error: PydanticValidationError) -> str:
    """Converte os issues do Pydantic em uma mensagem legivel.

    Nunca expor o erro cru do Pydantic para o usuario.
    """
    messages = []
    for issue in error.errors():
        field = ".".join(str(part) for part in issue["loc"]) or "body"
        messages.append(f"{field}: {issue['msg']}")
    return f"validation failed: {'; '.join(messages)}"


def parse_query[T: BaseModel](model: type[T], request: Request) -> T:
    """Valida query parameters com Pydantic. Lanca ValidationError (400) se invalido."""
    raw = {key: values[0] for key, values in request.query_params.to_dict().items() if values}
    try:
        return model.model_validate(raw)
    except PydanticValidationError as err:
        raise ValidationError(format_validation_error(err)) from err
```

### Validacao com Pydantic — Referencia Rapida

Validacoes mais usadas nos modelos de input (equivalencia com validadores classicos):

| Validacao | Pydantic v2 | Exemplo |
|---|---|---|
| Campo obrigatorio | campo sem default | `name: str` |
| Email valido | `EmailStr` | `email: EmailStr` |
| Tamanho minimo (string) | `Field(min_length=n)` | `name: str = Field(min_length=3)` |
| Tamanho maximo (string) | `Field(max_length=n)` | `name: str = Field(max_length=100)` |
| Maior que (numeros) | `Field(gt=n)` | `total_cents: int = Field(gt=0)` |
| Maior ou igual | `Field(ge=n)` | `limit: int = Field(ge=1)` |
| Um dos valores listados | `Literal[...]` | `currency: Literal["BRL", "USD", "EUR"]` |
| UUID valido | `UUID` (tipo) | `user_id: UUID` |
| URL valida | `HttpUrl` | `callback: HttpUrl` |
| Valida cada elemento do array | `list[Model]` | `items: list[ItemInput] = Field(min_length=1)` |
| So valida se preenchido | `X \| None = None` | `notes: str \| None = None` |
| Query string -> numero | coercao padrao do Pydantic | `limit: int = 20` (str "20" vira 20) |

**Padrao para modelos de input** (definidos no arquivo do controller, com alias camelCase na borda):

```python
from typing import Literal

from pydantic import BaseModel, ConfigDict, EmailStr, Field
from pydantic.alias_generators import to_camel


class ApiModel(BaseModel):
    """Base dos modelos da borda HTTP: aceita e emite camelCase."""

    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)


class OrderItemInput(ApiModel):
    product_id: UUID
    quantity: int = Field(gt=0)
    unit_price_cents: int = Field(gt=0)


class CreateOrderInput(ApiModel):
    user_id: UUID
    total_cents: int = Field(gt=0)
    currency: Literal["BRL", "USD", "EUR"]
    notes: str | None = Field(default=None, max_length=500)
    items: list[OrderItemInput] = Field(min_length=1)
```

**Regras de validacao:**
- **Sempre** validar input com modelo Pydantic via `bind_and_handle`/`parse_query`. Nunca validar manualmente no handler.
- **`UUID`** como tipo em campos que sao IDs.
- **`Literal[...]`** para enumeracoes (em vez de validar no use case).
- **`list[Model]` + `Field(min_length=1)`** valida cada elemento do array individualmente.
- **`X | None = None`** em campos opcionais que so devem ser validados se preenchidos.
- Coercao de query parameters (chegam sempre como string) e o comportamento padrao do Pydantic — `limit: int` aceita `"20"`.
- **Borda camelCase**: modelos de input/output estendem `ApiModel` (`alias_generator=to_camel`); internamente tudo e snake_case.
- Mensagens de erro formatadas pelo `format_validation_error` — nunca expor o erro cru do Pydantic para o usuario.

**`pkg/controller/error_mapper.py` — Mapeamento extensivel de erros:**

```python
# pkg/src/org_pkg/controller/error_mapper.py
from org_pkg.controller.errors import HttpError


class ErrorMapper:
    """Mapeamento de erros de dominio -> HTTP status.

    Cada app registra suas proprias classes de erro no mapper.
    """

    def __init__(self) -> None:
        self._mappings: list[tuple[type[Exception], int]] = []

    def register(self, target: type[Exception], status_code: int) -> "ErrorMapper":
        self._mappings.append((target, status_code))
        return self

    def map(self, err: Exception) -> tuple[int, str]:
        """Retorna (status, message) para um erro.

        HttpError usa o proprio status. Erros nao mapeados viram 500.
        """
        if isinstance(err, HttpError):
            return err.status, str(err)

        for target, status_code in self._mappings:
            if isinstance(err, target):
                return status_code, str(err)

        return 500, "internal server error"
```

**Registro dos erros no app (wiring no `bin/api.py`):**

```python
# apps/user/src/user/bin/api.py
from org_pkg.controller import ErrorMapper

from user.domain.entity.errors import (
    ForbiddenError,
    InvalidInputError,
    InvalidRoleError,
    UserAlreadyExistsError,
    UserNotFoundError,
)


def provide_error_mapper() -> ErrorMapper:
    """Configura o mapeamento de erros DESTE app."""
    return (
        ErrorMapper()
        .register(InvalidInputError, 400)
        .register(UserNotFoundError, 404)
        .register(UserAlreadyExistsError, 409)
        .register(InvalidRoleError, 422)
        .register(ForbiddenError, 403)
    )


# registrado no container dependency-injector:
#   error_mapper = providers.Singleton(provide_error_mapper)
```

#### Controller Concreto — Exemplo Simples (sem transaction)

```python
# apps/user/src/user/infrastructure/controller/user_controller.py
from pydantic import BaseModel, ConfigDict, EmailStr, Field
from pydantic.alias_generators import to_camel
from robyn import Request, Robyn, SubRouter
from structlog.stdlib import BoundLogger

from org_pkg.controller import (
    BaseController,
    ErrorMapper,
    RequestContext,
    json_response,
    parse_query,
)

from user.domain.entity.errors import UserNotFoundError
from user.domain.entity.user import User
from user.domain.usecase.user.create import CreateInput, CreateUseCase
from user.domain.usecase.user.get_by_id import GetByIdInput, GetByIdUseCase
from user.domain.usecase.user.list import ListInput, ListUseCase


class ApiModel(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)


class CreateUserBody(ApiModel):
    name: str = Field(min_length=2, max_length=100)
    email: EmailStr


class ListUsersQuery(ApiModel):
    cursor: str = ""
    limit: int = Field(default=20, ge=1, le=100)


class UserResponse(ApiModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True, from_attributes=True)

    id: str
    name: str
    email: str
    created_at: object  # datetime — serializado como ISO-8601 pelo model_dump(mode="json")
    updated_at: object


def user_to_json(user: User) -> dict[str, object]:
    return UserResponse.model_validate(user).model_dump(by_alias=True, mode="json")


class UserController(BaseController):
    def __init__(
        self,
        error_mapper: ErrorMapper,          # injetado via dependency-injector
        logger: BoundLogger,
        create_user_use_case: CreateUseCase,
        get_user_by_id_use_case: GetByIdUseCase,
        list_users_use_case: ListUseCase,
    ) -> None:
        super().__init__(error_mapper, logger)
        self._create_user_use_case = create_user_use_case
        self._get_user_by_id_use_case = get_user_by_id_use_case
        self._list_users_use_case = list_users_use_case

    def register_routes(self, app: Robyn) -> None:
        router = SubRouter(__file__, prefix="/v1")
        router.post("/users", openapi_tags=["users"])(
            self.bind_and_handle(CreateUserBody, self._create)
        )
        router.get("/users/:id", openapi_tags=["users"])(self.handle(self._get_by_id))
        router.get("/users", openapi_tags=["users"])(self.handle(self._list))
        app.include_router(router)

    async def _create(self, request: Request, rc: RequestContext, body: CreateUserBody):
        """Cria um usuario. 201 criado | 400 input invalido | 409 email ja em uso."""
        # body ja foi parseado e validado pelo bind_and_handle
        # request_id, parent_id, ip, logger ja estao no rc — zero boilerplate
        output = await self._create_user_use_case.perform(
            CreateInput(name=body.name, email=str(body.email))
        )
        # erros lancados sobem para BaseController._handle_error -> ErrorMapper -> HTTP status

        rc.logger.info("user created", user_id=output.user.id)
        return json_response(201, user_to_json(output.user))

    async def _get_by_id(self, request: Request, rc: RequestContext):
        """Busca um usuario por id. 200 | 404 nao encontrado."""
        user_id = request.path_params["id"]

        user = await self._get_user_by_id_use_case.perform(GetByIdInput(id=user_id))
        if user is None:
            raise UserNotFoundError()

        return json_response(200, user_to_json(user))

    async def _list(self, request: Request, rc: RequestContext):
        """Lista usuarios com paginacao por cursor."""
        query = parse_query(ListUsersQuery, request)

        output = await self._list_users_use_case.perform(
            ListInput(cursor=query.cursor, limit=query.limit)
        )

        return json_response(
            200,
            {
                "users": [user_to_json(u) for u in output.users],
                "nextCursor": output.next_cursor,
            },
        )
```

**Nota sobre `self`:** os handlers sao metodos comuns (`self._create`) passados como referencia para `handle`/`bind_and_handle` — bound methods em Python preservam o `self` automaticamente (diferente de JavaScript, nenhum truque e necessario).

#### Controller com Transaction — Exemplo Completo

Quando um endpoint precisa executar **multiplas operacoes atomicas** (ex: criar order + items + atualizar estoque), o controller orquestra a transaction via `Transactor`:

```python
# apps/billing/src/billing/infrastructure/controller/order_controller.py
from asyncpg import Connection
from robyn import Request, Robyn, SubRouter

from org_pkg.controller import BaseController, RequestContext, json_response

from billing.domain.entity.errors import OrderNotFoundError
from billing.domain.entity.order import Order, OrderItem


class OrderController(BaseController):
    def __init__(
        self,
        error_mapper: ErrorMapper,           # injetado via dependency-injector
        logger: BoundLogger,
        transactor: Transactor,
        order_repository: OrderRepository,
        stock_repository: StockRepository,
        order_event: OrderEvent,
    ) -> None:
        super().__init__(error_mapper, logger)
        self._transactor = transactor
        self._order_repository = order_repository
        self._stock_repository = stock_repository
        self._order_event = order_event

    def register_routes(self, app: Robyn) -> None:
        router = SubRouter(__file__, prefix="/v1")
        router.post("/orders")(self.bind_and_handle(CreateOrderBody, self._create))
        router.post("/orders/:id/cancel")(self.handle(self._cancel))
        app.include_router(router)

    async def _create(self, request: Request, rc: RequestContext, body: CreateOrderBody):
        """Cria order + items + debita estoque atomicamente (PRECISA de transaction)."""

        async def tx(conn: Connection) -> Order:
            # 1. criar order
            created = await self._order_repository.create_with_tx(
                conn, Order.create(str(body.user_id), body.total_cents)
            )

            # 2. criar items (mesma connection/tx)
            for item in body.items:
                await self._order_repository.create_item_with_tx(
                    conn,
                    OrderItem.create(
                        created.id, str(item.product_id), item.quantity, item.unit_price_cents
                    ),
                )

            # 3. debitar estoque (mesma connection/tx)
            for item in body.items:
                await self._stock_repository.debit_with_tx(
                    conn, str(item.product_id), item.quantity
                )

            return created  # COMMIT — order, items e estoque persistidos atomicamente

        # === TRANSACTION: tudo dentro e atomico ===
        order = await self._transactor.run_in_tx(tx)
        # erro dentro do run_in_tx => ROLLBACK e o erro sobe para o ErrorMapper

        # === DEPOIS DO COMMIT: publicar evento ===
        # Nunca publicar evento dentro da transaction
        with contextlib.suppress(Exception):
            await self._order_event.publish_created(order)

        rc.logger.info("order created", order_id=order.id)
        return json_response(201, order_to_json(order))

    async def _cancel(self, request: Request, rc: RequestContext):
        """Cancela uma order. Operacao SIMPLES, sem transaction (um unico UPDATE)."""
        order_id = request.path_params["id"]

        order = await self._order_repository.find_by_id(order_id)
        if order is None:
            raise OrderNotFoundError()

        order.cancel()  # regra de negocio (ex: lanca erro se ja cancelada)

        await self._order_repository.update(order)

        with contextlib.suppress(Exception):
            await self._order_event.publish_cancelled(order)

        rc.logger.info("order cancelled", order_id=order.id)
        return json_response(200, order_to_json(order))
```

#### Transactor — Protocol de Dominio

O `Transactor` e a **unica excecao documentada** de import externo no dominio: importa **apenas o tipo** `Connection` do `asyncpg` sob `TYPE_CHECKING` (sem codigo em runtime), cumprindo o papel que um handle de transaction da stdlib cumpriria.

```python
# apps/billing/src/billing/domain/repository/transactor.py
from collections.abc import Awaitable, Callable
from typing import TYPE_CHECKING, Protocol

if TYPE_CHECKING:
    from asyncpg import Connection


class Transactor(Protocol):
    async def run_in_tx[T](self, fn: "Callable[[Connection], Awaitable[T]]") -> T: ...
```

```python
# apps/billing/src/billing/infrastructure/repository/transactor.py
from collections.abc import Awaitable, Callable

from asyncpg import Connection

from org_pkg.postgres import Database, run_in_tx


class PostgresTransactor:
    def __init__(self, database: Database) -> None:
        self._database = database

    async def run_in_tx[T](self, fn: Callable[[Connection], Awaitable[T]]) -> T:
        return await run_in_tx(self._database.pool, fn)
```

**Helper compartilhado (`pkg/postgres/tx.py`):**

```python
# pkg/src/org_pkg/postgres/tx.py
from collections.abc import Awaitable, Callable

from asyncpg import Connection, Pool


async def run_in_tx[T](pool: Pool, fn: Callable[[Connection], Awaitable[T]]) -> T:
    """Adquire uma connection dedicada e executa fn dentro de uma transaction.

    COMMIT se fn retorna; ROLLBACK se fn lanca. A connection sempre volta para o pool.
    """
    async with pool.acquire() as conn:
        async with conn.transaction():
            return await fn(conn)
```

#### Repositorio com Metodos `with_tx`

Quando um repositorio participa de uma transaction externa, expoe metodos que recebem `Connection`:

```python
# apps/billing/src/billing/domain/repository/order_repository.py
from typing import TYPE_CHECKING, Protocol

from billing.domain.entity.order import Order, OrderItem

if TYPE_CHECKING:
    from asyncpg import Connection


class OrderRepository(Protocol):
    # Metodos normais (sem transaction, autocommit via pool)
    async def find_by_id(self, id: str) -> Order | None: ...
    async def update(self, order: Order) -> None: ...

    # Metodos with_tx (participam de transaction externa)
    async def create_with_tx(self, conn: "Connection", order: Order) -> Order: ...
    async def create_item_with_tx(self, conn: "Connection", item: OrderItem) -> OrderItem: ...
```

```python
# apps/billing/src/billing/infrastructure/repository/order_repository.py


class PostgresOrderRepository:
    def __init__(self, database: Database) -> None:
        self._database = database

    async def create_with_tx(self, conn: Connection, order: Order) -> Order:
        query = """
            INSERT INTO "orders" ("userId", "status", "totalCents", "createdAt", "updatedAt")
            VALUES ($1, $2, $3, $4, $5)
            RETURNING "id"
        """

        order_id = await conn.fetchval(
            query, order.user_id, order.status, order.total_cents, order.created_at, order.updated_at
        )

        order.id = str(order_id)
        return order

    async def find_by_id(self, id: str) -> Order | None:
        query = """
            SELECT "id", "userId", "status", "totalCents", "createdAt", "updatedAt"
            FROM "orders"
            WHERE "id" = $1
        """

        row = await self._database.pool.fetchrow(query, id)
        if row is None:
            return None
        return self._to_entity(row)
```

#### Quando Usar Transaction no Controller

| Cenario | Transaction? | Pattern |
|---|---|---|
| GET (leitura simples) | **Nao** | `self.handle(self._get_by_id)` |
| POST com 1 INSERT | **Nao** | `self.bind_and_handle(Model, self._create)` — use case faz INSERT unico |
| POST com N INSERTs relacionados | **Sim** | `transactor.run_in_tx` no handler |
| PUT/PATCH simples | **Nao** | Use case faz UPDATE unico |
| PUT que move saldo (debito + credito) | **Sim** | `transactor.run_in_tx` no handler |
| DELETE com cascade manual | **Sim** | `transactor.run_in_tx` no handler |
| Operacao + publicacao de evento | **Evento FORA do tx** | `run_in_tx(...)` depois `event.publish(...)` |

#### Regras do Controller

- `BaseController`, `RequestContext`, `bind_and_handle`, `ErrorMapper` e `json_response` vivem em **`pkg/controller/`**. Nenhum app reimplementa essa logica.
- Arquivo: `<name>_controller.py` (ex: `user_controller.py`) em `infrastructure/controller/`.
- Classe: `<Name>Controller` que **estende** `BaseController`.
- Constructor: `(error_mapper, logger, ...use_cases)` chamando `super().__init__(error_mapper, logger)` — resolvido via dependency-injector (kwargs dos providers = nomes dos parametros).
- Dependencias sao **Protocols de dominio de use case**, **repositorios** (quando precisa de tx) e **Transactor**.
- Deve implementar `register_routes(app: Robyn) -> None` criando um `SubRouter(__file__, prefix="/v1")` e montando via `app.include_router(router)`.
- Rotas registradas via `self.handle(self._method)` ou `self.bind_and_handle(Model, self._method)` — os decorators do Robyn sao chamados como funcao: `router.post("/users")(wrapper)`.
- **Todas as rotas registradas ANTES do `app.start()`** — o Robyn compila o roteamento no startup.
- Handlers recebem `RequestContext` ja populado — nunca extrair request_id, IP, etc. manualmente.
- Handlers delegam logica de negocio para use cases. Sem logica de negocio no controller.
- Docstrings dos handlers viram a descricao do endpoint no OpenAPI nativo do Robyn (o `functools.wraps` do BaseController preserva nome e docstring); tags via `openapi_tags` no decorator.
- Todo body e validado com **modelo Pydantic** via `bind_and_handle`; query parameters via `parse_query`. Nunca `request.json()` manual nem acesso direto a body sem validacao.
- Path parameters lidos de `request.path_params`; query parameters de `request.query_params` (sempre via `parse_query`).
- Evento publicado **DEPOIS** do commit da transaction, nunca dentro.
- Erros de dominio propagados com `raise` — o `BaseController._handle_error` mapeia para HTTP status via `ErrorMapper`.
- Cada app registra suas classes de erro de dominio no `ErrorMapper` via `provide_error_mapper()` no `bin/api.py`. O `pkg/controller` nao conhece nenhum erro de dominio — ele so recebe o mapeamento.

### subscriber/ (apenas Consumers)

Handlers de consumo NATS JetStream. Cada subscriber consome de um subject especifico e delega o processamento para use cases. **Usado apenas por binarios de Consumer.**

**Padrao:**

```python
# apps/billing/src/billing/infrastructure/subscriber/user_subscriber.py
import json
from uuid import uuid4

from nats.aio.msg import Msg
from pydantic import BaseModel, EmailStr, ValidationError
from structlog.stdlib import BoundLogger

from billing.domain.usecase.billing.create_account import CreateAccountInput, CreateAccountUseCase


class UserCreatedData(BaseModel):
    user_id: str
    name: str
    email: EmailStr


class UserSubscriber:
    def __init__(self, create_account_use_case: CreateAccountUseCase, logger: BoundLogger) -> None:
        self._create_account_use_case = create_account_use_case
        self._logger = logger

    async def handle_user_created(self, msg: Msg) -> None:
        request_id = str(uuid4())
        parent_id = (msg.headers or {}).get("Nats-Msg-Id", "")

        logger = self._logger.bind(
            request_id=request_id,
            parent_id=parent_id,
            subject=msg.subject,
        )

        try:
            envelope = json.loads(msg.data)
            data = UserCreatedData.model_validate(envelope["data"])
        except (json.JSONDecodeError, KeyError, ValidationError) as err:
            logger.error("invalid event payload", error=str(err))
            await msg.term()  # mensagem corrompida — nao reenviar
            return

        logger.info("processing user created event")

        try:
            await self._create_account_use_case.perform(
                CreateAccountInput(user_id=data.user_id, name=data.name, email=str(data.email))
            )
        except Exception as err:  # noqa: BLE001 — boundary de mensageria
            logger.error("create account failed", error=str(err))
            await msg.nak()  # redelivery com backoff
            return

        logger.info("account created successfully")
        await msg.ack()
```

**Regras:**
- Arquivo: `<name>_subscriber.py` (ex: `user_subscriber.py`).
- Classe: `<Name>Subscriber`.
- Dependencias sao **Protocols de dominio de use case** (+ `logger`), injetadas pelo constructor via dependency-injector.
- Handlers nomeados: `handle_<event_name>` (ex: `handle_user_created`), com assinatura `async (msg: Msg) -> None`.
- Payload decodificado com `json.loads(msg.data)` e **validado com Pydantic** antes de processar.
- `await msg.ack()` apenas apos processamento com sucesso.
- `await msg.nak()` para erros recuperaveis (JetStream faz redelivery com backoff).
- `await msg.term()` para mensagens corrompidas/invalidas (nao reenviar).
- Handler **nunca deixa excecao escapar**: todo caminho termina em `ack`, `nak` ou `term`.
- Logging estruturado com `request_id` e `parent_id` em todo handler.

### repository/

Implementacoes de repositorio usando PostgreSQL via `asyncpg`. Queries SQL escritas manualmente, sem ORM.

**Padrao:**

```python
# apps/user/src/user/infrastructure/repository/user_repository.py
from asyncpg import Record

from org_pkg.postgres import Database

from user.domain.entity.user import User


class PostgresUserRepository:
    def __init__(self, database: Database) -> None:
        self._database = database

    async def create(self, user: User) -> None:
        query = """
            INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
            VALUES ($1, $2, $3, $4)
            RETURNING "id"
        """

        user_id = await self._database.pool.fetchval(
            query, user.name, user.email, user.created_at, user.updated_at
        )
        user.id = str(user_id)

    async def find_by_id(self, id: str) -> User | None:
        query = """
            SELECT "id", "name", "email", "createdAt", "updatedAt"
            FROM "users"
            WHERE "id" = $1
        """

        row = await self._database.pool.fetchrow(query, id)
        if row is None:
            return None  # ausencia nao e erro
        return self._to_entity(row)

    async def find_by_email(self, email: str) -> User | None:
        query = """
            SELECT "id", "name", "email", "createdAt", "updatedAt"
            FROM "users"
            WHERE "email" = $1
        """

        row = await self._database.pool.fetchrow(query, email)
        if row is None:
            return None
        return self._to_entity(row)

    async def list(self, cursor: str, limit: int) -> tuple[list[User], str]:
        # list usa paginacao por cursor (sem OFFSET); busca limit + 1 linhas
        if cursor == "":
            query = """
                SELECT "id", "name", "email", "createdAt", "updatedAt"
                FROM "users"
                ORDER BY "id" ASC
                LIMIT $1
            """
            rows = await self._database.pool.fetch(query, limit + 1)
        else:
            query = """
                SELECT "id", "name", "email", "createdAt", "updatedAt"
                FROM "users"
                WHERE "id" > $1
                ORDER BY "id" ASC
                LIMIT $2
            """
            rows = await self._database.pool.fetch(query, cursor, limit + 1)

        users = [self._to_entity(row) for row in rows]

        next_cursor = ""
        if len(users) > limit:
            next_cursor = users[limit].id
            users = users[:limit]

        return users, next_cursor

    def _to_entity(self, row: Record) -> User:
        return User(
            id=str(row["id"]),
            name=row["name"],
            email=row["email"],
            created_at=row["createdAt"],
            updated_at=row["updatedAt"],
        )
```

**Regras:**
- Arquivo: `<name>_repository.py` (ex: `user_repository.py`).
- Classe: `Postgres<Name>Repository` satisfazendo o Protocol de dominio `<Name>Repository`.
- Recebe `Database` (wrapper do pool asyncpg, do `org_pkg.postgres`) pelo constructor (provider `database` no container).
- SQL escrito manualmente com placeholders posicionais (`$1`, `$2`). Sem ORM. **Nunca** interpolar valores na string da query (f-string em SQL e proibido).
- Sem `SELECT *`. Listar colunas explicitamente, entre aspas duplas (`"createdAt"`).
- `INSERT ... RETURNING "id"` + `fetchval` para obter ID gerado pelo banco.
- Zero linhas (`fetchrow` retorna `None`) -> retornar `None` (nao e erro, e ausencia).
- Paginacao por cursor, nunca `OFFSET` (para tabelas grandes). Buscar `limit + 1` linhas para descobrir o `next_cursor`.
- Queries como constante local dentro do metodo (triple-quoted string).
- Converter a `Record` para entidade em um metodo `_to_entity` (UUID -> `str(row["id"])`; TIMESTAMPTZ ja vem como `datetime` aware).
- Erros do driver propagam com `raise`; quando precisar de contexto, re-lancar com `raise RuntimeError("find user by id failed") from err`.

### publisher/

Implementacoes de eventos usando NATS JetStream.

**Padrao:**

```python
# apps/user/src/user/infrastructure/publisher/user_publisher.py
from org_pkg.event.user import SUBJECT_USER_CREATED, SUBJECT_USER_UPDATED
from org_pkg.nats import Publisher

from user.domain.entity.user import User


class UserPublisher:
    def __init__(self, publisher: Publisher) -> None:
        self._publisher = publisher

    async def publish_created(self, user: User) -> None:
        await self._publisher.publish(
            SUBJECT_USER_CREATED,
            msg_id=user.id,
            data={"userId": user.id, "name": user.name, "email": user.email},
        )

    async def publish_updated(self, user: User) -> None:
        await self._publisher.publish(
            SUBJECT_USER_UPDATED,
            msg_id=user.id,
            data={"userId": user.id, "name": user.name, "email": user.email},
        )
```

**Publisher generico compartilhado (`pkg/nats/publisher.py`):**

```python
# pkg/src/org_pkg/nats/publisher.py
import json
from datetime import UTC, datetime
from typing import Any
from uuid import uuid4

from org_pkg.nats.conn import NatsClient


class Publisher:
    def __init__(self, nats_client: NatsClient, source: str) -> None:
        self._nats_client = nats_client
        self._source = source

    async def publish(self, subject: str, msg_id: str, data: Any) -> None:
        """Envia um Envelope para o subject com msg_id para deduplicacao server-side."""
        envelope = {
            "id": str(uuid4()),
            "type": subject,
            "source": self._source,
            "timestamp": datetime.now(UTC).isoformat(),
            "data": data,
        }

        # Nats-Msg-Id — JetStream descarta duplicatas na janela de dedup
        await self._nats_client.js.publish(
            subject,
            json.dumps(envelope).encode(),
            headers={"Nats-Msg-Id": msg_id},
        )
```

**Regras:**
- Arquivo: `<name>_publisher.py` (ex: `user_publisher.py`).
- Classe: `<Name>Publisher` satisfazendo o Protocol de dominio `<Name>Event`.
- Registrado no container com o **nome do Protocol de dominio**: `user_event = providers.Singleton(UserPublisher, publisher=publisher)` — assim os use cases recebem `user_event` sem conhecer a implementacao.
- `msg_id` = ID da entidade para deduplicacao no JetStream (`Nats-Msg-Id`).
- Payload do envelope com chaves camelCase (contrato compartilhado entre apps de qualquer stack).
- Publicar DEPOIS do commit no banco, nunca antes.

### adapter/

Implementacoes concretas dos Protocols de bibliotecas externas definidos em `domain/<lib>/`.

**Padrao:**

```python
# apps/user/src/user/infrastructure/adapter/bcrypt_adapter.py
import asyncio

import bcrypt


class BcryptAdapter:
    _COST = 12

    async def hash(self, plain: str) -> str:
        # bcrypt e CPU-bound e sincrono — roda em thread para nao bloquear o event loop
        def _hash() -> str:
            return bcrypt.hashpw(plain.encode(), bcrypt.gensalt(self._COST)).decode()

        return await asyncio.to_thread(_hash)

    async def compare(self, hashed: str, plain: str) -> bool:
        def _compare() -> bool:
            return bcrypt.checkpw(plain.encode(), hashed.encode())

        return await asyncio.to_thread(_compare)
```

**Regras:**
- Arquivo: `<lib_name>_adapter.py` (ex: `bcrypt_adapter.py`, `jwt_adapter.py`).
- Classe: `<LibName>Adapter` satisfazendo o Protocol de dominio.
- Registrado no container com o **nome da capacidade de dominio**: `hasher = providers.Singleton(BcryptAdapter)`.
- O nome do arquivo/classe reflete a **lib concreta**, pois esta na camada de infraestrutura.
- Operacoes CPU-bound ou sincronas de I/O rodam via `asyncio.to_thread` — nunca bloquear o event loop.

### config/

Configuracao da aplicacao e container de infraestrutura compartilhada.

**`config.py`** — Dataclass de configuracao e carregamento de variaveis de ambiente:

```python
# apps/user/src/user/infrastructure/config/config.py
import os
from dataclasses import dataclass


@dataclass(frozen=True)
class AppConfig:
    app_host: str
    app_port: int
    db_host: str
    db_port: int
    db_user: str
    db_password: str
    db_name: str
    db_pool_max: int
    nats_url: str
    service_name: str
    service_version: str
    log_level: str


def load_config() -> AppConfig:
    return AppConfig(
        app_host=_env("APP_HOST", "0.0.0.0"),
        app_port=_env_int("APP_PORT", 8080),
        db_host=_env("DB_HOST", "localhost"),
        db_port=_env_int("DB_PORT", 5432),
        db_user=_env("DB_USER", "app"),
        db_password=_env("DB_PASSWORD", ""),
        db_name=_env("DB_NAME", "users"),
        db_pool_max=_env_int("DB_POOL_MAX", 10),
        nats_url=_env("NATS_URL", "nats://localhost:4222"),
        service_name=_env("SERVICE_NAME", "user-api"),
        service_version=_env("SERVICE_VERSION", "dev"),
        log_level=_env("LOG_LEVEL", "info"),
    )


def _env(key: str, fallback: str) -> str:
    value = os.environ.get(key, "")
    return value if value != "" else fallback


def _env_int(key: str, fallback: int) -> int:
    try:
        return int(os.environ.get(key, ""))
    except ValueError:
        return fallback
```

O `.env` e carregado no **topo do `main()` de cada binario** com `load_dotenv()` (python-dotenv) — nunca dentro de `load_config`.

**`container.py`** — Container de infraestrutura compartilhada (dependency-injector):

```python
# apps/user/src/user/infrastructure/config/container.py
from dependency_injector import containers, providers

from org_pkg.logger import create_logger
from org_pkg.nats import NatsClient, Publisher
from org_pkg.postgres import Database

from user.infrastructure.config.config import load_config


class InfraContainer(containers.DeclarativeContainer):
    """Providers de infraestrutura compartilhada entre todos os binarios do app.

    Database e NatsClient sao wrappers com ciclo de vida explicito:
    `await connect()` no startup e `await disconnect()/drain()` no shutdown —
    as conexoes precisam ser criadas NO event loop do processo, nunca no import.
    """

    config = providers.Singleton(load_config)

    logger = providers.Singleton(
        create_logger,
        service=config.provided.service_name,
        version=config.provided.service_version,
        level=config.provided.log_level,
    )

    database = providers.Singleton(Database, config=config)

    nats_client = providers.Singleton(NatsClient, config=config)

    publisher = providers.Singleton(
        Publisher,
        nats_client=nats_client,
        source=config.provided.service_name,
    )
```

**Regras:**
- `InfraContainer` fornece **apenas** infraestrutura compartilhada (`config`, `logger`, `database`, `nats_client`, `publisher`).
- Registro de rotas e startup do servidor ficam no **`bin/api.py`**, NAO no `container.py`.
- Logica de startup do subscriber fica no **`bin/consumer.py`**.
- Os wrappers `Database`/`NatsClient` sao construidos de forma **sincrona e barata**; a conexao real acontece em `await connect()` dentro do startup do binario (no event loop do processo).
- **Nunca criar conexao no import ou no constructor** — pool asyncpg e conexao NATS ficam presos ao event loop em que nasceram.

---

## Camada de Pacotes (`pkg/` — workspace member `org-pkg`, modulo `org_pkg`)

Bibliotecas reutilizaveis compartilhadas entre todos os apps. Nunca importam `src/` de nenhum app.

| Modulo | Import | Responsabilidade |
|---|---|---|
| `postgres` | `from org_pkg.postgres import ...` | `Database` (pool asyncpg + ciclo de vida), `run_in_tx`, health check |
| `nats` | `from org_pkg.nats import ...` | `NatsClient`, `ensure_stream`, `Publisher`, `subscribe` |
| `controller` | `from org_pkg.controller import ...` | `BaseController`, `RequestContext`, `ErrorMapper`, `json_response`, `parse_query` |
| `logger` | `from org_pkg.logger import ...` | `create_logger()` com structlog JSON + `service` e `version` na base |
| `event` | `from org_pkg.event import ...` | `Envelope` base e contratos de eventos compartilhados |

**`pkg/logger/logger.py`:**

```python
# pkg/src/org_pkg/logger/logger.py
import logging

import structlog
from structlog.stdlib import BoundLogger


def create_logger(service: str, version: str, level: str) -> BoundLogger:
    structlog.configure(
        processors=[
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, level.upper(), logging.INFO)
        ),
        cache_logger_on_first_use=True,
    )
    return structlog.get_logger().bind(service=service, version=version)
```

**`pkg/postgres/conn.py`:**

```python
# pkg/src/org_pkg/postgres/conn.py
from dataclasses import dataclass

import asyncpg


@dataclass(frozen=True)
class PostgresConfig:
    db_host: str
    db_port: int
    db_user: str
    db_password: str
    db_name: str
    db_pool_max: int


class Database:
    """Wrapper do pool asyncpg com ciclo de vida explicito.

    Construido de forma sincrona no composition root; `connect()` roda no
    startup do binario (dentro do event loop) e `disconnect()` no shutdown.
    """

    def __init__(self, config: PostgresConfig) -> None:
        self._config = config
        self._pool: asyncpg.Pool | None = None

    async def connect(self) -> None:
        self._pool = await asyncpg.create_pool(
            host=self._config.db_host,
            port=self._config.db_port,
            user=self._config.db_user,
            password=self._config.db_password,
            database=self._config.db_name,
            min_size=1,
            max_size=self._config.db_pool_max,
            command_timeout=10,
        )

    @property
    def pool(self) -> asyncpg.Pool:
        if self._pool is None:
            raise RuntimeError("database not connected — call connect() on startup")
        return self._pool

    async def disconnect(self) -> None:
        if self._pool is not None:
            await self._pool.close()
            self._pool = None
```

**`pkg/nats/conn.py` e `pkg/nats/stream.py`:**

```python
# pkg/src/org_pkg/nats/conn.py
from dataclasses import dataclass

import nats
from nats.aio.client import Client
from nats.js import JetStreamContext, JetStreamManager


@dataclass(frozen=True)
class NatsConfig:
    nats_url: str
    service_name: str


class NatsClient:
    """Wrapper da conexao NATS com ciclo de vida explicito (connect/drain)."""

    def __init__(self, config: NatsConfig) -> None:
        self._config = config
        self._nc: Client | None = None

    async def connect(self) -> None:
        self._nc = await nats.connect(
            servers=self._config.nats_url,
            name=self._config.service_name,
            max_reconnect_attempts=-1,  # reconectar para sempre
            reconnect_time_wait=2,
        )

    @property
    def nc(self) -> Client:
        if self._nc is None:
            raise RuntimeError("nats not connected — call connect() on startup")
        return self._nc

    @property
    def js(self) -> JetStreamContext:
        return self.nc.jetstream()

    async def jsm(self) -> JetStreamManager:
        return self.nc.jsm()

    async def drain(self) -> None:
        if self._nc is not None:
            await self._nc.drain()
            self._nc = None
```

```python
# pkg/src/org_pkg/nats/stream.py
from dataclasses import dataclass

from nats.js import JetStreamManager
from nats.js.api import RetentionPolicy, StorageType, StreamConfig
from nats.js.errors import NotFoundError


@dataclass(frozen=True)
class StreamOptions:
    name: str
    subjects: list[str]
    max_age_days: int
    max_bytes: int
    replicas: int


async def ensure_stream(jsm: JetStreamManager, opts: StreamOptions) -> None:
    """Cria o stream se nao existir, ou atualiza a configuracao."""
    config = StreamConfig(
        name=opts.name,
        subjects=opts.subjects,
        retention=RetentionPolicy.LIMITS,
        storage=StorageType.FILE,
        max_age=opts.max_age_days * 24 * 60 * 60,  # segundos (nats-py converte para nanos)
        max_bytes=opts.max_bytes,
        num_replicas=opts.replicas,
    )

    try:
        await jsm.stream_info(opts.name)
        await jsm.update_stream(config)
    except NotFoundError:
        await jsm.add_stream(config)
```

**`pkg/nats/subscriber.py`:**

```python
# pkg/src/org_pkg/nats/subscriber.py
import asyncio
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

import nats.errors
from nats.aio.msg import Msg
from nats.js import JetStreamContext, JetStreamManager
from nats.js.api import AckPolicy, ConsumerConfig


@dataclass(frozen=True)
class ConsumerOptions:
    stream: str
    consumer: str          # nome do consumer duravel
    filter_subject: str
    max_deliver: int
    ack_wait_seconds: int


class ConsumerLoop:
    """Loop de consumo de um pull consumer duravel. `stop()` encerra graciosamente."""

    def __init__(self) -> None:
        self._stop = asyncio.Event()
        self._task: asyncio.Task[None] | None = None

    def stop(self) -> None:
        self._stop.set()

    async def wait(self) -> None:
        if self._task is not None:
            await self._task

    def _start(self, coro: Awaitable[None]) -> None:
        self._task = asyncio.ensure_future(coro)


async def subscribe(
    js: JetStreamContext,
    jsm: JetStreamManager,
    opts: ConsumerOptions,
    handler: Callable[[Msg], Awaitable[None]],
) -> ConsumerLoop:
    """Garante o consumer duravel e inicia o loop de consumo (pull + fetch).

    Retorna ConsumerLoop — chamar `.stop()` no shutdown graceful.
    """
    await jsm.add_consumer(
        opts.stream,
        ConsumerConfig(
            durable_name=opts.consumer,
            ack_policy=AckPolicy.EXPLICIT,
            filter_subject=opts.filter_subject,
            max_deliver=opts.max_deliver,
            ack_wait=opts.ack_wait_seconds,  # segundos
        ),
    )

    psub = await js.pull_subscribe(
        subject=opts.filter_subject, durable=opts.consumer, stream=opts.stream
    )

    loop = ConsumerLoop()

    async def run() -> None:
        while not loop._stop.is_set():
            try:
                msgs = await psub.fetch(batch=16, timeout=5)
            except nats.errors.TimeoutError:
                continue  # sem mensagens na janela — segue o loop
            for msg in msgs:
                # o handler e responsavel por ack/nak/term e nunca lanca
                await handler(msg)

    loop._start(run())
    return loop
```

**Regras:**
- Sem estado global de modulo (sem conexao criada no import — tudo instanciado por classe/funcao e injetado).
- Funcoes e classes recebem e retornam dependencias explicitamente.
- Nunca importa nada de `apps/*/src/`.
- Pode ser usado por outros projetos.

---

## Injecao de Dependencia com dependency-injector

Cada binario (`bin/api.py`, `bin/consumer.py`) e um **composition root** com seu proprio container declarativo, que **herda** de `InfraContainer` e registra apenas o que precisa. Os kwargs dos providers correspondem exatamente aos nomes dos parametros dos constructors.

| Provider | Uso |
|---|---|
| `providers.Singleton(load_config)` | Valores construidos uma unica vez (config, logger) |
| `providers.Singleton(Classe, dep=outro_provider)` | Classes com constructor injection (repos, use cases, controllers, subscribers) |
| `providers.Singleton(provide_error_mapper)` | Factories nomeadas (error mapper do app) |

### Composicao no `bin/api.py` da API

```python
# apps/user/src/user/bin/api.py
from dependency_injector import providers
from dotenv import load_dotenv
from robyn import Robyn

from org_pkg.controller import ErrorMapper

from user.application.usecase.user.create_usecase import CreateUseCase
from user.application.usecase.user.get_by_id_usecase import GetByIdUseCase
from user.application.usecase.user.list_usecase import ListUseCase
from user.domain.entity.errors import (
    ForbiddenError,
    InvalidInputError,
    InvalidRoleError,
    UserAlreadyExistsError,
    UserNotFoundError,
)
from user.infrastructure.adapter.bcrypt_adapter import BcryptAdapter
from user.infrastructure.config.container import InfraContainer
from user.infrastructure.controller.user_controller import UserController
from user.infrastructure.publisher.user_publisher import UserPublisher
from user.infrastructure.repository.user_repository import PostgresUserRepository


def provide_error_mapper() -> ErrorMapper:
    return (
        ErrorMapper()
        .register(InvalidInputError, 400)
        .register(UserNotFoundError, 404)
        .register(UserAlreadyExistsError, 409)
        .register(InvalidRoleError, 422)
        .register(ForbiddenError, 403)
    )


class ApiContainer(InfraContainer):
    """Composition root da API: infraestrutura herdada + providers especificos."""

    error_mapper = providers.Singleton(provide_error_mapper)

    # repositories
    user_repository = providers.Singleton(PostgresUserRepository, database=InfraContainer.database)

    # publishers (registrados pelo nome do Protocol de dominio)
    user_event = providers.Singleton(UserPublisher, publisher=InfraContainer.publisher)

    # adapters (registrados pelo nome da capacidade de dominio)
    hasher = providers.Singleton(BcryptAdapter)

    # use cases (nome do provider: <acao>_<contexto>_use_case)
    create_user_use_case = providers.Singleton(
        CreateUseCase, user_repository=user_repository, user_event=user_event
    )
    get_user_by_id_use_case = providers.Singleton(GetByIdUseCase, user_repository=user_repository)
    list_users_use_case = providers.Singleton(ListUseCase, user_repository=user_repository)

    # controllers
    user_controller = providers.Singleton(
        UserController,
        error_mapper=error_mapper,
        logger=InfraContainer.logger,
        create_user_use_case=create_user_use_case,
        get_user_by_id_use_case=get_user_by_id_use_case,
        list_users_use_case=list_users_use_case,
    )


def main() -> None:
    load_dotenv()

    container = ApiContainer()
    config = container.config()
    logger = container.logger()

    app = Robyn(__file__)

    # Rotas registradas ANTES do app.start (o Robyn compila o roteamento no startup)
    container.user_controller().register_routes(app)

    # Conexoes criadas/derrubadas NO event loop do servidor
    @app.startup_handler
    async def on_startup() -> None:
        await container.database().connect()
        await container.nats_client().connect()
        logger.info("api started", port=config.app_port)

    @app.shutdown_handler
    async def on_shutdown() -> None:
        logger.info("shutting down")
        # 1. NATS drain (aguarda publishes pendentes), depois pool PG close
        await container.nats_client().drain()
        await container.database().disconnect()
        logger.info("shutdown complete")

    # safety net: erro que escapou do BaseController vira 500 padronizado
    @app.exception
    def handle_uncaught(error: Exception):
        from org_pkg.controller import json_response

        return json_response(500, {"error": "internal server error", "status": 500})

    app.start(host=config.app_host, port=config.app_port)


if __name__ == "__main__":
    main()
```

### Composicao no `bin/consumer.py` do Consumer

Cada binario de consumer compoe seu proprio container. **Sem Robyn, sem controllers, sem rotas** — asyncio puro.

```python
# apps/billing/src/billing/bin/consumer.py
import asyncio
import signal

from dependency_injector import providers
from dotenv import load_dotenv

from org_pkg.nats import ConsumerOptions, StreamOptions, ensure_stream, subscribe

from billing.application.usecase.billing.create_account_usecase import CreateAccountUseCase
from billing.infrastructure.config.container import InfraContainer
from billing.infrastructure.repository.account_repository import PostgresAccountRepository
from billing.infrastructure.subscriber.user_subscriber import UserSubscriber


class ConsumerContainer(InfraContainer):
    """Composition root do consumer: apenas o necessario para consumir."""

    account_repository = providers.Singleton(
        PostgresAccountRepository, database=InfraContainer.database
    )
    create_account_use_case = providers.Singleton(
        CreateAccountUseCase, account_repository=account_repository
    )
    user_subscriber = providers.Singleton(
        UserSubscriber,
        create_account_use_case=create_account_use_case,
        logger=InfraContainer.logger,
    )


async def run() -> None:
    load_dotenv()

    container = ConsumerContainer()
    logger = container.logger()

    await container.database().connect()
    await container.nats_client().connect()

    js = container.nats_client().js
    jsm = await container.nats_client().jsm()

    # Garante o stream antes de consumir
    await ensure_stream(
        jsm,
        StreamOptions(
            name="EVENTS_USER",
            subjects=["events.user.>"],
            max_age_days=7,
            max_bytes=1 * 1024 * 1024 * 1024,
            replicas=1,
        ),
    )

    # Inicia o consumer duravel
    user_subscriber = container.user_subscriber()
    consumer_loop = await subscribe(
        js,
        jsm,
        ConsumerOptions(
            stream="EVENTS_USER",
            consumer="billing-on-user-created",
            filter_subject="events.user.created",
            max_deliver=5,
            ack_wait_seconds=30,
        ),
        user_subscriber.handle_user_created,
    )

    logger.info("consumer started")

    # === Shutdown graceful ===
    stop = asyncio.Event()
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, stop.set)

    await stop.wait()
    logger.info("shutting down")

    # 1. para de consumir novas mensagens (a mensagem em andamento conclui)
    consumer_loop.stop()
    await consumer_loop.wait()

    # 2. NATS drain (aguarda acks pendentes), depois pool PG close
    await container.nats_client().drain()
    await container.database().disconnect()

    logger.info("shutdown complete")


def main() -> None:
    asyncio.run(run())


if __name__ == "__main__":
    main()
```

### Regras de Composicao

1. **`InfraContainer`** e a base de todo container de binario. Fornece `config`, `logger`, `database`, `nats_client` e `publisher`.
2. **Cada binario registra apenas o que precisa.** Uma API registra controllers; um consumer registra subscribers. Nenhum registra as dependencias do outro.
3. **Startup do servidor HTTP** (`Robyn` + `register_routes` + `app.start`) fica no `bin/api.py`, nao em um modulo compartilhado.
4. **`ensure_stream` e `subscribe`** ficam no `bin/consumer.py`, nao em um modulo compartilhado.
5. **Kwargs explicitos**: os providers passam dependencias por **keyword argument com o mesmo nome do parametro do constructor** (`user_repository=user_repository`) — o grafo fica legivel e verificado pelo mypy.
6. **Nomes de providers** sao snake_case e unicos no container: use cases usam `<acao>_<contexto>_use_case` (ex: `create_user_use_case`), implementacoes de contratos de dominio usam o **nome do contrato** (`user_repository`, `user_event`, `hasher`, `transactor`).
7. **Tudo `providers.Singleton`**: repos, use cases, controllers e subscribers nao guardam estado por request.
8. **Conexoes nascem no event loop**: `Database.connect()`/`NatsClient.connect()` rodam no `startup_handler` (API) ou no inicio do `run()` (consumer) — **nunca** no import nem no constructor.
9. **Nunca usar o container como service locator** dentro das camadas: `container.<provider>()` so aparece no composition root (`bin/`).

### Ordem de Shutdown

**API (Robyn `shutdown_handler`, disparado por SIGTERM/SIGINT):**
1. O Robyn para de aceitar novas conexoes.
2. `nats_client.drain()` — aguarda mensagens pendentes.
3. `database.disconnect()` — fecha o pool asyncpg.

**Consumer:**
1. `consumer_loop.stop()` + `wait()` — subscriber para de consumir; mensagem em andamento conclui.
2. `nats_client.drain()`.
3. `database.disconnect()`.

### Documentacao OpenAPI (apenas APIs)

- O Robyn gera OpenAPI **nativamente**: Swagger UI em `/docs` e spec em `/openapi.json`, sem lib extra.
- Metadados globais via `Robyn(__file__, openapi=OpenAPI(info=OpenAPIInfo(title=..., version=...)))`.
- Por rota: `openapi_name` e `openapi_tags` nos decorators; a **docstring do handler** vira a descricao do endpoint (o `functools.wraps` do BaseController preserva nome e docstring do metodo concreto).
- Os **schemas de request/response sao os modelos Pydantic** definidos no arquivo do controller — eles sao a fonte de verdade do contrato; o QA valida o comportamento real contra esses modelos e contra o `/openapi.json`.
- Exportar o spec para arquivo quando necessario: `curl -s http://localhost:8080/openapi.json > docs/openapi.json`.

---

## Convencoes de Nomenclatura

### Arquivos

| Localizacao | Padrao | Exemplo |
|---|---|---|
| `src/<app>/bin/` | `<binary>.py` | `api.py`, `consumer.py` |
| `domain/entity/` | `<name>.py` | `user.py` |
| `domain/usecase/<context>/` | `<action>.py` | `user/create.py` |
| `domain/repository/` | `<name>_repository.py` | `user_repository.py` |
| `domain/event/` | `<name>_event.py` | `user_event.py` |
| `domain/<lib>/` | `<name>.py` | `crypt/hasher.py` |
| `application/usecase/<context>/` | `<action>_usecase.py` | `user/create_usecase.py` |
| `infrastructure/controller/` | `<name>_controller.py` | `user_controller.py` |
| `infrastructure/repository/` | `<name>_repository.py` | `user_repository.py` |
| `infrastructure/publisher/` | `<name>_publisher.py` | `user_publisher.py` |
| `infrastructure/subscriber/` | `<name>_subscriber.py` | `user_subscriber.py` |
| `infrastructure/adapter/` | `<lib>_adapter.py` | `bcrypt_adapter.py` |

- Nomes de arquivo e modulo sempre em **snake_case** (padrao Python — kebab-case nao e importavel).
- Testes: `tests/unit/**/test_<file>.py` (unitarios, pytest) e `tests/integration/test_<file>.py` (integracao, testcontainers).

### Diretorios

Sempre **lowercase** e **singular**: `entity`, `usecase`, `repository`, `event`, `crypt`, `token`, `controller`, `publisher`, `subscriber`, `adapter`, `config`, `bin`.

### Tipos, Protocols e Classes

| Elemento | Padrao | Exemplo |
|---|---|---|
| Entidade | `<Name>` (dataclass) | `User` |
| Protocol UC | `<Action>UseCase` | `CreateUseCase` |
| Protocol Repo | `<Name>Repository` | `UserRepository` |
| Protocol Event | `<Name>Event` | `UserEvent` |
| Protocol Lib | `<Capability>` | `Hasher`, `Manager` |
| Classe Impl UC | `<Action>UseCase` (sem `Impl`) | `CreateUseCase` |
| Classe Impl Repo | `Postgres<Name>Repository` | `PostgresUserRepository` |
| Controller | `<Name>Controller` | `UserController` |
| Publisher | `<Name>Publisher` | `UserPublisher` |
| Subscriber | `<Name>Subscriber` | `UserSubscriber` |
| Adapter | `<LibName>Adapter` | `BcryptAdapter` |

- Classes e Protocols em **PascalCase**. Protocols **sem** prefixo `I`.
- Quando a classe de aplicacao tem o mesmo nome do Protocol de dominio (`CreateUseCase`), eles vivem em **modulos diferentes** (`domain.usecase.user.create` vs `application.usecase.user.create_usecase`) — o import qualificado desambigua; nunca importar os dois no mesmo namespace sem alias.

### Nomes de Providers no Container (dependency-injector)

| Elemento | Nome do provider | Exemplo |
|---|---|---|
| Config | `config` | `providers.Singleton(load_config)` |
| Logger | `logger` | `providers.Singleton(create_logger, ...)` |
| Database (pool PG) | `database` | `providers.Singleton(Database, config=config)` |
| Conexao NATS | `nats_client` | `providers.Singleton(NatsClient, config=config)` |
| Publisher generico | `publisher` | `providers.Singleton(Publisher, ...)` |
| ErrorMapper | `error_mapper` | `providers.Singleton(provide_error_mapper)` |
| Repositorio | `<name>_repository` | `user_repository = providers.Singleton(PostgresUserRepository, ...)` |
| Evento (publisher) | `<name>_event` | `user_event = providers.Singleton(UserPublisher, ...)` |
| Lib externa (adapter) | `<capability>` | `hasher = providers.Singleton(BcryptAdapter)` |
| Transactor | `transactor` | `transactor = providers.Singleton(PostgresTransactor, ...)` |
| Use case | `<acao>_<contexto>_use_case` | `create_user_use_case = providers.Singleton(CreateUseCase, ...)` |
| Controller | `<name>_controller` | `user_controller = providers.Singleton(UserController, ...)` |
| Subscriber | `<name>_subscriber` | `user_subscriber = providers.Singleton(UserSubscriber, ...)` |

> Os **parametros do constructor** de qualquer classe registrada devem usar exatamente esses nomes — os providers injetam por keyword argument.

### Imports

Organizados em 3 blocos separados por linhas em branco (ruff `I` / isort):

```python
# 1. stdlib
import json
from uuid import uuid4

# 2. third-party
from pydantic import BaseModel
from robyn import Request, Robyn

# 3. first-party (org_pkg antes do modulo do app)
from org_pkg.controller import BaseController

from user.domain.entity.user import User
from user.domain.repository.user_repository import UserRepository
```

- Imports **absolutos sempre** — nunca relativos entre camadas.
- `from __future__ import annotations` desnecessario no Python 3.13 para os padroes deste projeto; usar `TYPE_CHECKING` para imports somente de tipo (caso Transactor).

---

## Comunicacao entre Servicos

Cada app tem seu proprio banco (database-per-service). Comunicacao entre apps e feita via NATS JetStream — **nunca** por chamadas HTTP diretas nem por acesso ao banco do outro app.

```
┌──────────┐  events.user.created  ┌──────────────┐
│ apps/user│ ────────────────────► │ apps/billing │
│  (API)   │    NATS JetStream     │  (Consumer)  │
└────┬─────┘                       └──────┬───────┘
     │                                     │
     v                                     v
 ┌────────┐                          ┌──────────┐
 │users_db│                          │billing_db│
 └────────┘                          └──────────┘
```

### Contratos de Eventos Compartilhados (`pkg/event/`)

```python
# pkg/src/org_pkg/event/envelope.py
from pydantic import BaseModel


class Envelope[T](BaseModel):
    """Envelope base de todo evento publicado no JetStream."""

    id: str         # uuid do evento
    type: str       # subject (ex: events.user.created)
    source: str     # nome do servico que publicou
    timestamp: str  # ISO-8601 UTC
    data: T
```

```python
# pkg/src/org_pkg/event/user.py
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

SUBJECT_USER_CREATED = "events.user.created"
SUBJECT_USER_UPDATED = "events.user.updated"


class UserCreatedData(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)

    user_id: str
    name: str
    email: str
```

**Regras:**
- Subjects hierarquicos: `events.<dominio>.<acao>` (ex: `events.user.created`).
- Um stream por dominio: `EVENTS_USER`, `EVENTS_BILLING`.
- Publisher usa `Nats-Msg-Id` = ID da entidade para deduplicacao server-side.
- Consumer duravel com ack explicito, backoff e `max_deliver`.
- Idempotencia no consumer: `INSERT ... ON CONFLICT DO NOTHING`.
- Payload dos eventos com chaves **camelCase** (contrato neutro entre stacks); os modelos Pydantic usam `alias_generator=to_camel`.

---

## Guia Passo a Passo: Adicionando uma Nova Feature

Exemplo: adicionando o dominio `Order` no app `billing`.

### Passo 1 — Entidade de Dominio

Crie `apps/billing/src/billing/domain/entity/order.py`:

```python
from dataclasses import dataclass
from datetime import UTC, datetime


@dataclass
class Order:
    id: str
    user_id: str
    amount: int  # centavos (inteiro)
    status: str
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create(cls, user_id: str, amount: int) -> "Order":
        now = datetime.now(UTC)
        return cls(
            id="",
            user_id=user_id,
            amount=amount,
            status="pending",
            created_at=now,
            updated_at=now,
        )
```

### Passo 2 — Protocols de Dominio (Use Cases)

Crie um Protocol por use case, organizados por contexto.

Crie `apps/billing/src/billing/domain/usecase/order/create.py`:

```python
from dataclasses import dataclass
from typing import Protocol

from billing.domain.entity.order import Order


@dataclass(frozen=True)
class CreateInput:
    user_id: str
    amount: int


@dataclass(frozen=True)
class CreateOutput:
    order: Order


class CreateUseCase(Protocol):
    async def perform(self, input: CreateInput) -> CreateOutput: ...
```

### Passo 2b — Protocols de Dominio (Repository e Event)

Crie `apps/billing/src/billing/domain/repository/order_repository.py`:

```python
from typing import Protocol

from billing.domain.entity.order import Order


class OrderRepository(Protocol):
    async def create(self, order: Order) -> None: ...
    async def find_by_id(self, id: str) -> Order | None: ...
```

Crie `apps/billing/src/billing/domain/event/order_event.py` (se aplicavel):

```python
from typing import Protocol

from billing.domain.entity.order import Order


class OrderEvent(Protocol):
    async def publish_created(self, order: Order) -> None: ...
```

### Passo 3 — Use Case de Aplicacao

Crie `apps/billing/src/billing/application/usecase/order/create_usecase.py`:

```python
from billing.domain.entity.order import Order
from billing.domain.repository.order_repository import OrderRepository
from billing.domain.usecase.order.create import CreateInput, CreateOutput


class CreateUseCase:
    def __init__(self, order_repository: OrderRepository) -> None:
        self._order_repository = order_repository

    async def perform(self, input: CreateInput) -> CreateOutput:
        order = Order.create(input.user_id, input.amount)
        await self._order_repository.create(order)
        return CreateOutput(order=order)
```

### Passo 4 — Repositorio de Infraestrutura

Crie `apps/billing/src/billing/infrastructure/repository/order_repository.py` (`PostgresOrderRepository`, queries manuais com `$1`, `RETURNING "id"` + `fetchval`, `None` quando nao encontrado).

### Passo 5 — Publisher de Infraestrutura (se aplicavel)

Crie `apps/billing/src/billing/infrastructure/publisher/order_publisher.py` (`OrderPublisher` satisfazendo `OrderEvent`; subject e data model em `pkg/src/org_pkg/event/billing.py`).

### Passo 6a — Controller (se a API precisar)

Crie `apps/billing/src/billing/infrastructure/controller/order_controller.py` (estende `BaseController`, modelos Pydantic no proprio arquivo, `register_routes`).

### Passo 6b — Subscriber (se o Consumer precisar)

Crie `apps/billing/src/billing/infrastructure/subscriber/order_subscriber.py` (`handle_<event_name>` com `ack`/`nak`/`term`).

### Passo 7 — Migration SQL

Crie `migrations/billing/0002_create_orders.sql` (dbmate) seguindo as boas praticas de PostgreSQL 18:

```sql
-- migrate:up
CREATE TABLE "orders" (
    "id"         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId"     UUID        NOT NULL,
    "status"     TEXT        NOT NULL DEFAULT 'pending',
    "amount"     BIGINT      NOT NULL CHECK ("amount" > 0),
    "createdAt"  TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt"  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX "idx_orders_userId" ON "orders" ("userId");

-- migrate:down
DROP TABLE "orders";
```

Aplicar com `dbmate --migrations-dir migrations/billing up` (a `DATABASE_URL` de cada app aponta para seu proprio banco).

### Passo 8 — Composicao dependency-injector

**Para a API** — atualize o `ApiContainer` em `apps/billing/src/billing/bin/api.py`:

```python
class ApiContainer(InfraContainer):
    # ... existentes ...
    order_repository = providers.Singleton(PostgresOrderRepository, database=InfraContainer.database)
    order_event = providers.Singleton(OrderPublisher, publisher=InfraContainer.publisher)
    create_order_use_case = providers.Singleton(CreateUseCase, order_repository=order_repository)
    order_controller = providers.Singleton(
        OrderController,
        error_mapper=error_mapper,
        logger=InfraContainer.logger,
        create_order_use_case=create_order_use_case,
    )
```

Registre as rotas no mesmo arquivo (antes do `app.start`):

```python
container.order_controller().register_routes(app)
```

**Para o Consumer** — atualize `apps/billing/src/billing/bin/consumer.py` registrando o subscriber e o `subscribe(...)` do novo consumer duravel.

### Passo 9 — Testes

- Unitario: `apps/billing/tests/unit/application/usecase/order/test_create_usecase.py` (pytest, fakes tipados das interfaces de dominio).
- Integracao: `apps/billing/tests/integration/test_order_repository.py` (testcontainers com PostgreSQL real, `pytestmark = pytest.mark.integration`).

### Passo 10 — OpenAPI (apenas APIs)

Escreva docstrings nos handlers, defina `openapi_tags` nas rotas e verifique `http://localhost:8080/docs` / exporte o spec com `curl -s http://localhost:8080/openapi.json > docs/openapi.json`.

### Checklist — Nova Feature

- [ ] `domain/entity/<name>.py` — Entidade (dataclass com `classmethod create`)
- [ ] `domain/usecase/<context>/<action>.py` — 1 Protocol por use case (metodo `perform`)
- [ ] `domain/repository/<name>_repository.py` — Protocol de repositorio
- [ ] `domain/event/<name>_event.py` — Protocol de evento (se aplicavel)
- [ ] `domain/<lib>/<name>.py` — Protocol para lib externa (se aplicavel)
- [ ] `application/usecase/<context>/<action>_usecase.py` — Implementacao do use case
- [ ] `tests/unit/application/usecase/<context>/test_<action>_usecase.py` — Teste unitario (pytest + fakes)
- [ ] `infrastructure/repository/<name>_repository.py` — Implementacao PostgreSQL (asyncpg)
- [ ] `tests/integration/test_<name>_repository.py` — Teste de integracao (testcontainers)
- [ ] `infrastructure/publisher/<name>_publisher.py` — Implementacao NATS (se aplicavel)
- [ ] `infrastructure/adapter/<lib>_adapter.py` — Implementacao de lib externa (se aplicavel)
- [ ] `infrastructure/controller/<name>_controller.py` — Controller Robyn + modelos Pydantic (se API)
- [ ] `infrastructure/subscriber/<name>_subscriber.py` — Subscriber NATS (se Consumer)
- [ ] `migrations/<app>/NNNN_<descricao>.sql` — Migration dbmate (`-- migrate:up` / `-- migrate:down`)
- [ ] `apps/<app>/src/<app>/bin/api.py` — Registrar no container e chamar `register_routes` (se API)
- [ ] `apps/<app>/src/<app>/bin/consumer.py` — Registrar subscriber e `subscribe(...)` (se Consumer)
- [ ] Docstrings + `openapi_tags` nos handlers e spec verificado em `/docs` (se API)
- [ ] Rodar os gates: `make typecheck`, `make lint`, `make format-check`, `make test`

### Checklist — Novo App

- [ ] Criar `apps/<app-name>/pyproject.toml` (`org-<app-name>`, dependencia `org-pkg`, hatchling com `src/<app-name>`)
- [ ] Confirmar que o `[tool.uv.workspace]` da raiz cobre o novo app (`apps/*`) e rodar `uv sync`
- [ ] Criar `apps/<app-name>/src/<app-name>/bin/api.py` e/ou `bin/consumer.py`
- [ ] Criar `infrastructure/config/config.py` (`AppConfig` + `load_config`) e `container.py` (`InfraContainer`)
- [ ] Herdar de `InfraContainer` no container do binario e registrar apenas repositorios, publishers, adapters, use cases e handlers necessarios
- [ ] Definir startup (`Robyn` + `register_routes` + `startup_handler` + `app.start` para API; `ensure_stream` + `subscribe` para Consumer)
- [ ] Registrar shutdown graceful (`shutdown_handler` na API; `add_signal_handler` + `consumer_loop.stop()` no Consumer) com ordem NATS drain -> PG disconnect
- [ ] Criar `migrations/<app-name>/` com migrations iniciais (dbmate)
- [ ] Criar `Dockerfile` (base `python:3.13-slim`, `uv sync --locked`, runtime `python -m <app>.bin.<binary>`)
- [ ] Adicionar ao `docker-compose.yml`
