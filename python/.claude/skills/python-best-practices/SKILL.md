---
name: python-best-practices
description: Aplica boas praticas modernas de Python 3.13 com type hints estritos (mypy --strict) e asyncio ao escrever, revisar ou refatorar codigo Python. Use ao criar/editar arquivos .py, ao revisar codigo Python, ao discutir mypy strict, type hints (Protocol, generics PEP 695, sintaxe X | None), async/await (corrotinas nao aguardadas, asyncio.gather, TaskGroup, timeouts, cancelamento), tratamento de erros (excecoes encadeadas com raise from, classes de erro de dominio), dataclasses, imutabilidade, event loop e nunca bloquear com codigo sincrono. Acione sempre que o trabalho envolver Python.
---

# Python Best Practices (Python 3.13 · mypy strict · asyncio)

Skill baseada na documentacao oficial do Python (docs.python.org), no mypy (mypy.readthedocs.io) e nas PEPs relevantes. Todas as recomendacoes aqui devem ser seguidas ao escrever, revisar ou refatorar codigo Python neste harness.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.py`
- Ao discutir design, nomenclatura ou arquitetura de aplicacoes Python
- Ao tratar de async/await, cancelamento, erros ou type hints
- Ao gerar exemplos de codigo Python em respostas

## 1. mypy Strict — Configuracao Canonica

O projeto usa **mypy em modo strict** com Python 3.13. Esta e a configuracao base (no `pyproject.toml` raiz do workspace):

```toml
[tool.mypy]
python_version = "3.13"
strict = true
# strict ja liga: disallow_untyped_defs, disallow_any_generics,
# warn_return_any, no_implicit_optional, warn_unused_ignores, etc.
warn_unreachable = true
disallow_untyped_decorators = true

[[tool.mypy.overrides]]
module = ["robyn.*"]
# libs sem stubs completos: nunca `ignore_errors`; use ignores pontuais e justificados
```

### O que `strict` obriga na pratica

| Regra (parte de strict) | Efeito | Consequencia pratica |
|---|---|---|
| `disallow_untyped_defs` | Toda funcao/metodo tem tipos completos | Sem funcao sem anotacao |
| `no_implicit_optional` | `x: str = None` e erro | Escreva `x: str | None = None` |
| `warn_return_any` | Retornar `Any` de funcao tipada alerta | Fronteiras usam narrowing, nao vazam `Any` |
| `disallow_any_generics` | `list` sem parametro e erro | Sempre `list[str]`, `dict[str, int]` |
| `warn_unused_ignores` | `# type: ignore` desnecessario alerta | Ignores sempre justificados e minimos |

- **`mypy .` (strict) e gate de CI** — zero erros. `# type: ignore` apenas com codigo de erro e comentario (`# type: ignore[arg-type]  # robyn sem stub`).
- Nunca `Any` para "fazer passar" — use `object` + narrowing, `TypeVar`, ou Protocol.

## 2. Sintaxe Moderna de Tipos (3.13)

O projeto usa a sintaxe moderna — **nunca** importe de `typing` o que ja e builtin:

```python
# ✅ moderno (3.10+)
def f(items: list[str], meta: dict[str, int]) -> str | None: ...

# ❌ legado — nao usar
from typing import List, Dict, Optional
def f(items: List[str], meta: Dict[str, int]) -> Optional[str]: ...
```

### Generics PEP 695 (3.12+)

```python
# ✅ sintaxe nova de generics — sem TypeVar explicito
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

class Repository[T]:
    async def get(self, id: str) -> T | None: ...

# type alias novo
type Json = dict[str, "Json"] | list["Json"] | str | int | float | bool | None
```

### Uniao com `|`, nunca `Optional`/`Union`

```python
def find(id: str) -> User | None: ...          # ✅
def parse(x: str | bytes) -> dict[str, str]: ...  # ✅
# Optional[User] / Union[str, bytes] — evitar (legado)
```

### `Protocol` para contratos estruturais

O projeto define contratos de dominio como **Protocols** (structural typing) — as implementacoes satisfazem sem herdar:

```python
from typing import Protocol


class UserRepository(Protocol):
    async def find_by_id(self, id: str) -> "User | None": ...
    async def create(self, user: "User") -> None: ...


# implementacao NAO herda de UserRepository — apenas satisfaz a forma.
# o mypy verifica a conformidade no ponto de injecao/uso.
class PostgresUserRepository:
    async def find_by_id(self, id: str) -> User | None: ...
    async def create(self, user: User) -> None: ...
```

- Protocols em `domain/` sao a versao Python das interfaces — sem prefixo `I`, PascalCase.
- Use `@runtime_checkable` **apenas** se precisar de `isinstance` (raro — evite).

## 3. Dataclasses e Modelos

### Entidades de dominio: `@dataclass`

```python
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
        return cls(id="", name=name, email=email, created_at=now, updated_at=now)
```

- `@dataclass` para entidades de dominio (sem dependencia externa).
- `@dataclass(frozen=True)` para inputs/outputs de use case e value objects imutaveis.
- `@classmethod create(...)` para construir com regras de negocio; o `__init__` gerado rehidrata do banco.
- **Pydantic `BaseModel`** apenas na **fronteira** (validacao HTTP em `infrastructure/controller/`, eventos em `pkg/event/`) — nunca no dominio.

### `slots=True` em entidades quentes

```python
@dataclass(slots=True)
class OrderItem:
    product_id: str
    quantity: int
```

Reduz memoria e acelera acesso a atributos em objetos criados em volume.

## 4. Tratamento de Erros

### Lance apenas `Exception` (ou subclasses) — nunca strings

```python
# ❌ Python nao permite `raise "erro"` — mas o equivalente ruim e generico demais
raise Exception("user not found")   # generico, dificil de capturar seletivamente

# ✅ classe de erro de dominio especifica
raise UserNotFoundError(user_id)
```

### Classes de erro de dominio

Cada app define seus erros em `domain/entity/errors.py` (equivalente aos sentinel errors):

```python
# domain/entity/errors.py
class DomainError(Exception):
    """Classe base para todos os erros de dominio do app."""


class UserNotFoundError(DomainError):
    def __init__(self, id: str) -> None:
        super().__init__(f"user {id}: not found")


class UserAlreadyExistsError(DomainError):
    def __init__(self, email: str) -> None:
        super().__init__(f"email {email}: already in use")
```

Verificacao com `isinstance` (o equivalente de `errors.Is`), usada pelo `ErrorMapper`:

```python
if isinstance(err, UserNotFoundError):
    return 404, str(err)
```

### `raise ... from` — a cadeia de excecoes (equivalente ao `%w`)

Ao adicionar contexto sem perder o erro original, use `from`:

```python
try:
    await conn.execute(sql, *params)
except asyncpg.PostgresError as err:
    raise RuntimeError("create user: insert failed") from err
```

Regras:
- **Sempre** encadeie com `from err` ao re-lancar com contexto novo. Nunca `raise RuntimeError(str(err))` — perde o traceback e o tipo original.
- Erros de dominio embrulham erros de infra via `from` quando a origem importa para diagnostico.
- Mensagens comecam com minuscula, sem ponto final, identificando a operacao: `"create user: insert failed"`.

### `except` estreito — nunca engula

```python
# ❌ except amplo que engole tudo (mascara bugs, KeyboardInterrupt, etc.)
try:
    await do_work()
except Exception:
    pass

# ✅ capture o especifico, ou re-lance o que nao conhece
try:
    await do_work()
except DomainError as err:
    logger.warning("request rejected", error=str(err))
    raise
```

- `except Exception:` apenas em **boundaries** (handler HTTP, loop de consumo de mensagem) — e sempre com log + acao explicita (responder erro, `nak`, etc.), nunca `pass`.
- Nunca capture `BaseException` (pega `KeyboardInterrupt`/`SystemExit`/`asyncio.CancelledError`).
- **Nunca engula `CancelledError`**: se capturar em um `except Exception` amplo, re-lance — engolir quebra o cancelamento cooperativo do asyncio.

### `contextlib.suppress` para fire-and-forget explicito

```python
import contextlib

# publica evento apos commit — falha de publicacao nao desfaz o negocio
with contextlib.suppress(Exception):
    await self._user_event.publish_created(user)
```

Use apenas quando ignorar a falha e **intencional e documentado** (com log dentro do publisher).

## 5. Async/Await

### Nunca deixe corrotinas sem await

Toda corrotina deve ser aguardada, agendada como task rastreada, ou explicitamente tratada. Corrotina criada e nao aguardada e um bug (`RuntimeWarning: coroutine was never awaited`).

```python
# ❌ corrotina solta — nunca executa, warning silencioso
repository.save(user)

# ✅ aguarde
await repository.save(user)

# ✅ fire-and-forget INTENCIONAL: task rastreada + tratamento de erro
task = asyncio.create_task(publisher.publish_created(user))
task.add_done_callback(lambda t: t.exception() and logger.error("publish failed", error=str(t.exception())))
```

- **Guarde referencia** a toda task de `create_task` — o event loop so mantem weakref; tasks orfas podem ser coletadas no meio.
- Configure lint (`ruff` `ASYNC`, ou `RUF006`) para detectar `create_task` sem referencia.

### `asyncio.gather` para paralelismo; `TaskGroup` (3.11+) quando precisa de cancelamento estruturado

```python
# ❌ sequencial sem necessidade — 3x mais lento
user = await user_repository.find_by_id(id)
orders = await order_repository.list_by_user(id)
balance = await balance_repository.get_by_user(id)

# ✅ operacoes independentes em paralelo
user, orders, balance = await asyncio.gather(
    user_repository.find_by_id(id),
    order_repository.list_by_user(id),
    balance_repository.get_by_user(id),
)

# ✅ TaskGroup: se uma falha, as outras sao canceladas (concorrencia estruturada)
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(step_one())
    t2 = tg.create_task(step_two())
result = t1.result(), t2.result()
```

- `gather(..., return_exceptions=False)` (default) propaga a primeira excecao — as demais tasks continuam ao fundo; use `TaskGroup` quando falha parcial deve cancelar o resto.
- `gather(..., return_exceptions=True)` quando voce precisa do resultado de todas mesmo com falhas.

### Timeouts com `asyncio.timeout` (3.11+)

```python
# ✅ context manager de timeout — cancela o bloco ao estourar
async with asyncio.timeout(5):
    result = await slow_operation()

# ✅ timeout pontual
result = await asyncio.wait_for(slow_operation(), timeout=5)
```

- Prefira `asyncio.timeout(...)` a `wait_for` para blocos com multiplos awaits.
- Toda operacao de I/O externa (DB, NATS, HTTP) num handler deve ter timeout.

### Cancelamento cooperativo

```python
# CancelledError propaga o cancelamento — NUNCA engula
try:
    await long_running()
except asyncio.CancelledError:
    await cleanup()   # cleanup rapido permitido
    raise             # re-lance SEMPRE
```

### `async` ate o fim

- Nao misture codigo sincrono bloqueante com async — ver secao 8.
- `async for` para iterar async iterators (mensagens NATS, cursores); `async with` para recursos async (pool, conexao, transaction).
- Nunca chame `asyncio.run()` fora do entrypoint (`bin/`) — cria e destroi event loop.

## 6. Type Narrowing e `None`

Padronizacao do projeto (espelha o `nil, nil` dos harnesses Go/Node):

| Situacao | Valor |
|---|---|
| Consulta que nao encontrou (`find_by_id`) | **retorna `None`** |
| Parametro opcional ausente | `None` (default) |
| Campo de banco nullable | `T | None` no tipo da entidade |

```python
async def find_by_id(self, id: str) -> User | None:
    # "nao encontrado" NAO e erro — e None
    ...

# Caller decide se ausencia e erro no SEU contexto:
user = await self._user_repository.find_by_id(input.id)
if user is None:
    raise UserNotFoundError(input.id)
```

Regras:
- **Nunca** lance excecao para "nao encontrado" dentro do repositorio — ausencia e dado, nao falha.
- Comparacao com `None` sempre por identidade: `is None` / `is not None`. Nunca `== None`.
- Narrowing explicito antes de usar `T | None` — o mypy strict exige.
- `x or default` engole `0`/`""`/`False`; para "so quando None" use `x if x is not None else default`.

## 7. Imutabilidade e Constantes

```python
from typing import Final

# constantes de modulo
MAX_PAGE_SIZE: Final = 100
SUBJECT_USER_CREATED: Final = "events.user.created"

# tuplas para sequencias fixas; frozenset para conjuntos
ALLOWED_ROLES: Final = frozenset({"admin", "editor", "viewer"})

# dataclass frozen para value objects
@dataclass(frozen=True)
class Money:
    cents: int
    currency: str
```

- Prefira criar novo objeto (`dataclasses.replace(user, name=new_name)`) a mutar in-place quando o valor e conceitualmente imutavel.
- Nunca use argumento default mutavel (`def f(x: list = [])`) — bug classico; use `None` e crie dentro.
- `Final` em constantes de modulo; `frozen=True` em value objects.

## 8. Event Loop, CPU-Bound e I/O Bloqueante

### Nunca bloqueie o event loop

Asyncio atende tudo em uma thread. Qualquer chamada sincrona longa trava **todas** as corrotinas:

- **Proibido** em codigo de request: `time.sleep`, `requests.get`, I/O de arquivo sincrono, `bcrypt.hashpw` direto, parsing pesado, loops CPU-bound grandes.
- I/O sincrono inevitavel de lib externa vai para thread: `await asyncio.to_thread(blocking_fn, arg)`.
- CPU-bound real (hash de senha, compressao, parsing de arquivo grande) vai para `asyncio.to_thread` (libera o GIL se a lib for C) ou `ProcessPoolExecutor` via `loop.run_in_executor`.

```python
import asyncio
import bcrypt

# hash e CPU-bound e sincrono — roda em thread para nao travar o loop
async def hash_password(plain: str) -> str:
    return await asyncio.to_thread(
        lambda: bcrypt.hashpw(plain.encode(), bcrypt.gensalt(12)).decode()
    )
```

- Use sempre drivers **async** (`asyncpg`, `nats-py`, `httpx`) — nunca `psycopg2` sincrono ou `requests` no caminho async.

## 9. Organizacao de Modulos e Naming

### Arquivos e diretorios

- Arquivos e modulos em **snake_case**: `create_user_usecase.py`, `user_repository.py` (kebab-case nao e importavel em Python).
- 1 use case = 1 Protocol = 1 arquivo, metodo `perform(input)`.
- Camadas: `domain/` -> `application/` -> `infrastructure/` -> `bin/`. Camada interna nunca importa externa.
- `pkg/` (`org_pkg`) nunca importa `apps/`.
- Layout `src/` obrigatorio; **imports absolutos sempre** (`from user.domain.entity.user import User`), nunca relativos entre camadas.

### Identificadores

| Elemento | Convencao | Exemplo |
|---|---|---|
| Classes, Protocols, type aliases | `PascalCase` | `CreateUseCase`, `UserRepository` |
| Funcoes, metodos, variaveis | `snake_case` | `find_by_id`, `next_cursor` |
| Constantes de modulo | `SCREAMING_SNAKE_CASE` | `MAX_PAGE_SIZE` |
| Atributo/metodo "privado" | prefixo `_` | `self._pool`, `_to_entity` |
| Booleanos | prefixo `is`/`has`/`can` | `is_active`, `has_pending_orders` |

- Sem prefixo `I` em Protocols (`UserRepository`, nao `IUserRepository`). A implementacao concreta ganha o qualificador: `PostgresUserRepository`.
- Sem sufixo `Impl`.
- Modulos publicos exportam via nome (sem `__all__` obrigatorio, mas util em `pkg/`).

## 10. Ferramentas

| Ferramenta | Papel | Regra |
|---|---|---|
| `mypy --strict` | Typecheck no CI | Zero erros; `# type: ignore[code]` so justificado |
| `ruff check` | Lint | Regras `E,F,I,B,UP,N,ASYNC,RUF`; sem `noqa` sem codigo+motivo |
| `ruff format` | Formatacao | Sem debate de estilo; roda no pre-commit e CI (`--check`) |
| `pytest` + `pytest-asyncio` | Testes | Ver [[python-testing-best-practices]] — `asyncio_mode = "auto"` |
| `structlog` | Logging JSON estruturado | Nunca `print` em codigo de aplicacao |
| `uv` | Workspace e deps | `uv run`, `uv sync --locked`, `[tool.uv.workspace]` |

## Checklist de Revisao

Ao escrever ou revisar codigo Python, verifique:

- [ ] Imports absolutos; organizados em stdlib -> third-party -> first-party
- [ ] Sintaxe moderna de tipos (`list[str]`, `X | None`, generics PEP 695) — sem `List`/`Optional`/`Union`
- [ ] Toda funcao/metodo com type hints completos (mypy strict passa)
- [ ] Nenhum `Any`; fronteiras usam Pydantic/narrowing
- [ ] Contratos de dominio como `Protocol` (sem prefixo `I`)
- [ ] Entidades como `@dataclass`; inputs/outputs `frozen=True`
- [ ] `raise` apenas de `Exception`/subclasse; erros de dominio como classes
- [ ] Re-throw com contexto usa `raise ... from err`
- [ ] `except` estreito; boundary com `except Exception` sempre loga + age; `CancelledError` re-lancado
- [ ] Nenhuma corrotina sem await; fire-and-forget com task rastreada
- [ ] Operacoes independentes com `asyncio.gather`/`TaskGroup`
- [ ] I/O externa com `asyncio.timeout`/`wait_for`
- [ ] "Nao encontrado" retorna `None`, nunca excecao do repositorio
- [ ] Comparacao com `None` via `is`/`is not`
- [ ] Sem default mutavel; `Final`/`frozen` em dados que nao mutam
- [ ] Nenhum I/O sincrono bloqueante no caminho async; CPU-bound em `to_thread`
- [ ] `print` ausente — structlog em todo logging

## Fontes Oficiais

Toda recomendacao acima vem destas fontes — consulte-as quando houver duvida:

- https://docs.python.org/3/library/asyncio.html
- https://docs.python.org/3/library/asyncio-task.html (gather, TaskGroup, timeout, cancelamento)
- https://docs.python.org/3/library/dataclasses.html
- https://docs.python.org/3/library/typing.html
- https://mypy.readthedocs.io/en/stable/
- https://docs.astral.sh/ruff/
- https://peps.python.org/pep-0695/ (type parameter syntax)
