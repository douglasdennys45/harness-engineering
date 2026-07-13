---
name: python-testing-best-practices
description: Aplica melhores praticas de testes em Python cobrindo a piramide completa com pytest — unitarios com funcoes de teste, AAA e pytest.mark.parametrize, dublês com fakes tipados (implementando Protocols) injetados por constructor (dependency-injector), unittest.mock/AsyncMock, freezegun para tempo, integracao com testcontainers (PostgreSqlContainer, NATS) e E2E de API subindo a API Robyn em porta livre com httpx. Use sempre que escrever, revisar ou refatorar arquivos test_*.py, ao discutir estrategia de testes, mocks, fixtures, containers de teste, cobertura (pytest-cov) ou mutation testing (mutmut). Acione ao montar suites pytest, ao subir dependencias reais via testcontainers ou ao configurar pytest no pyproject.toml.
---

# Python Testing Best Practices (pytest · testcontainers · mutmut)

Skill baseada na documentacao oficial:

- [docs.pytest.org](https://docs.pytest.org/) (`test_*`, fixtures, `parametrize`, `pytest.raises`)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/) (`asyncio_mode = "auto"`)
- [testcontainers-python](https://testcontainers-python.readthedocs.io/) (`PostgresContainer`, `DockerContainer`)
- [mutmut](https://mutmut.readthedocs.io/) (mutation testing)

Toda recomendacao aqui e a forma idiomatica prescrita pelas fontes oficiais, adaptada a arquitetura do projeto (Clean Architecture, DI por constructor com dependency-injector, 1 use case = 1 Protocol = metodo `perform`). Combine com [[python-best-practices]] para convencoes gerais da linguagem.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer `test_*.py`
- Ao criar fakes/mocks/stubs ou estruturar suites
- Ao desenhar a estrategia de testes (unit, integracao, E2E)
- Ao subir dependencias reais (Postgres, NATS) via testcontainers
- Ao discutir cobertura, fixtures, tempo determinístico ou mutation testing

## 0. A Piramide de Testes no Projeto

| Camada | Velocidade | Escopo | Ferramenta principal |
|---|---|---|---|
| **Unitario** | ms | Use case/entidade isolados, sem I/O | pytest + fakes/`AsyncMock` |
| **Integracao** | ~s | Repositorio/publisher contra dependencia **real** em container | pytest + `testcontainers` |
| **E2E de API** | s | HTTP -> controller -> use case -> banco real, via `httpx` | pytest + API Robyn em porta livre |

Regras inegociaveis:
- Muitos unitarios -> poucos integracao -> minimo de E2E.
- Toda camada usa **pytest**. Diferencie por **diretorio + marker**, nunca por framework paralelo.
- Testes de integracao marcados com `@pytest.mark.integration` — nao rodam por padrao (`addopts = "-m 'not integration'"`); `make test-integration` roda so eles.

## 1. Convencoes Gerais para `test_*.py`

### Layout de arquivos

| Camada | Localizacao | Prefixo |
|---|---|---|
| Unitario | `tests/unit/` do app, espelhando `src/` | `test_` |
| Integracao | `tests/integration/` do app | `test_` + `pytestmark = pytest.mark.integration` |
| E2E de API | `tests/integration/` do app | `test_` + marker `integration` |

- Unitarios espelham a estrutura de `src/` dentro de `tests/unit/` (ex: `tests/unit/application/usecase/user/test_create_usecase.py`).
- Integracao/E2E em `tests/integration/` com helpers em `tests/integration/testhelper/`.

### Configuracao pytest (`pyproject.toml`)

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"                 # testes async sem @pytest.mark.asyncio explicito
markers = ["integration: testes que sobem containers reais"]
addopts = "-m 'not integration'"      # unitarios por padrao; integracao via -m integration
testpaths = ["apps"]

[tool.coverage.run]
source = ["apps"]
omit = ["*/bin/*", "*/tests/*"]

[tool.coverage.report]
fail_under = 80
```

Rode assim:

```bash
uv run pytest                         # unitarios (marker 'not integration')
uv run pytest -m integration          # integracao + E2E (containers)
uv run pytest --cov                   # cobertura (pytest-cov)
uv run pytest apps/user/tests/unit    # diretorio especifico
```

### Estrutura de teste + AAA

- Nome da funcao descreve **comportamento** em linguagem de dominio: `test_creates_user_and_publishes_event`, `test_raises_user_already_exists_when_email_taken`.
- Todo teste segue **AAA** (Arrange, Act, Assert) — separe os blocos com linha em branco, sem comentarios redundantes.
- Agrupe cenarios relacionados numa `class Test<Unidade>` quando ajudar (sem `__init__`).

```python
import pytest

from user.application.usecase.user.create_usecase import CreateUseCase
from user.domain.entity.errors import UserAlreadyExistsError
from user.domain.usecase.user.create import CreateInput


class TestCreateUseCase:
    async def test_creates_user_and_publishes_event(self) -> None:
        # Arrange
        repository = FakeUserRepository()
        publisher = FakeUserEventPublisher()
        sut = CreateUseCase(user_repository=repository, user_event=publisher)

        # Act
        output = await sut.perform(CreateInput(name="Ana", email="ana@example.com"))

        # Assert
        assert output.user.email == "ana@example.com"
        assert await repository.find_by_email("ana@example.com") is not None
        assert len(publisher.published) == 1
```

### Asercoes essenciais

| Cenario | Use |
|---|---|
| Igualdade | `assert a == b` |
| Identidade / None | `assert x is None` / `assert x is not None` |
| Excecao async | `with pytest.raises(UserAlreadyExistsError): await sut.perform(...)` |
| Excecao com match | `pytest.raises(DomainError, match="not found")` |
| Colecoes | `assert item in coll`, `assert len(coll) == 3` |
| Parcial (dict) | `assert body["email"] == "..."` (asserte campos, nao o dict todo com ruido) |

Regras:
- `assert` simples do pytest (ele reescreve para diagnostico rico) — nunca `unittest.TestCase`.
- Erro esperado **sempre** via `pytest.raises(ClasseDeErro)` — asserta o tipo, nao so a mensagem.
- Nunca `try/except` manual para assertar erro.

### Table-driven com `pytest.mark.parametrize`

Sempre que houver >= 3 casos da mesma regra:

```python
@pytest.mark.parametrize(
    ("email", "valid"),
    [
        ("ana@example.com", True),
        ("sem-arroba.com", False),
        ("", False),
        ("a@b.co", True),
    ],
)
def test_email_validation(email: str, valid: bool) -> None:
    assert is_valid_email(email) is valid
```

- IDs legiveis: o pytest gera automaticamente; use `ids=[...]` se precisar clareza.
- Cada linha **independente** — nenhum caso depende do estado deixado por outro.
- Para casos que esperam erro, inclua a classe de erro esperada na tabela (`expected_error`), nao duplique o teste.

### Fixtures e isolamento

- Estado novo por teste: crie fakes/SUT numa `@pytest.fixture` ou factory local (`make_sut()`), nunca em variavel de modulo mutavel compartilhada.
- Fixtures compartilhadas de suite vao em `conftest.py` (do diretorio apropriado).
- Padrao factory (deixa o Arrange explicito):

```python
@dataclass
class Sut:
    use_case: CreateUseCase
    repository: FakeUserRepository
    publisher: FakeUserEventPublisher


def make_sut() -> Sut:
    repository = FakeUserRepository()
    publisher = FakeUserEventPublisher()
    return Sut(CreateUseCase(user_repository=repository, user_event=publisher), repository, publisher)
```

## 2. Dublês de Teste: Fake tipado > mock de modulo

O projeto usa **dependency-injector** — toda dependencia entra pelo constructor como Protocol de dominio. Isso torna mock de modulo (`unittest.mock.patch` no caminho de import) **desnecessario e indesejado** na maior parte dos casos:

| Tecnica | Quando usar |
|---|---|
| **Fake tipado** (satisfaz o Protocol de dominio) | Padrao para use cases — comportamento realista in-memory |
| **`AsyncMock`** (spec do Protocol) | Verificar interacao pontual (foi chamado? com que args?) |
| **`unittest.mock.patch`** | **Ultimo recurso** — funcao top-level sem injecao possivel |

Por que preferir fake a patch:
- `patch` acopla o teste ao **caminho do import** e quebra silenciosamente em refactors.
- Fake tipado e verificado pelo mypy contra o Protocol — se o Protocol muda, o fake nao passa no typecheck.

### Fake tipado — padrao do projeto

Fakes vivem em `tests/fakes/`:

```python
# tests/fakes/fake_user_repository.py
from user.domain.entity.user import User


class FakeUserRepository:
    """Satisfaz UserRepository (Protocol) — verificado pelo mypy no ponto de uso."""

    def __init__(self) -> None:
        self._users: dict[str, User] = {}

    async def create(self, user: User) -> None:
        if not user.id:
            user.id = f"id-{len(self._users) + 1}"
        self._users[user.id] = user

    async def find_by_id(self, id: str) -> User | None:
        return self._users.get(id)

    async def find_by_email(self, email: str) -> User | None:
        return next((u for u in self._users.values() if u.email == email), None)
```

- O fake **respeita o contrato de None-safety**: nao encontrado -> `None`, nunca raise.
- Para garantir conformidade em compile time, adicione um teste de tipo simples ou use o fake num contexto que exija o Protocol (o mypy pega).

### `AsyncMock` — verificacao de interacao

Quando o que importa e "chamou X com Y":

```python
from unittest.mock import AsyncMock

from user.domain.repository.user_repository import UserRepository


def make_user_repository_mock() -> UserRepository:
    mock = AsyncMock(spec=UserRepository)
    mock.find_by_email.return_value = None
    return mock


async def test_does_not_create_when_email_exists() -> None:
    repository = make_user_repository_mock()
    repository.find_by_email.return_value = existing_user
    sut = CreateUseCase(user_repository=repository, user_event=AsyncMock())

    with pytest.raises(UserAlreadyExistsError):
        await sut.perform(CreateInput(name="Ana", email="ana@example.com"))

    repository.create.assert_not_awaited()
```

Regras:
- `AsyncMock(spec=Protocol)` — o `spec` garante que so metodos reais sao mockaveis.
- `return_value` para corrotinas; `side_effect=Err()` para simular falha.
- `assert_awaited_once_with(...)` / `assert_not_awaited()` para interacao async.
- Verifique a interacao **relevante**, nao todas — teste comportamento, nao implementacao.

### O que **nao** fazer com dublês

- Nao use `patch("asyncpg.connect")` para testar repositorio — isso testa sua imaginacao do driver. Repositorio se testa com **container real** (secao 4).
- Nao asserte chamadas internas irrelevantes.
- Nao compartilhe mocks entre testes sem fixture que os recrie.

## 3. Tempo Determinístico

Use `freezegun` para logica com `datetime.now`:

```python
from freezegun import freeze_time

@freeze_time("2026-01-01T00:00:00Z")
def test_uses_frozen_time_in_created_at() -> None:
    user = User.create("Ana", "ana@example.com")
    assert user.created_at.isoformat() == "2026-01-01T00:00:00+00:00"
```

- Alternativa de arquitetura: injete um `Clock` (Protocol) e use um `FixedClock` no teste — preferivel quando a entidade precisa de data determinística sem monkeypatch global.
- Nunca `asyncio.sleep(...)` real em unitario — controle o tempo ou espere por condicao.

## 4. Testes de Use Case (Unitarios)

Use case recebe Protocols de dominio pelo constructor (kwargs = providers do dependency-injector); fakes/mocks entram direto — **o container nao participa de teste unitario**.

```python
# tests/unit/application/usecase/user/test_create_usecase.py
import pytest

from user.application.usecase.user.create_usecase import CreateUseCase
from user.domain.entity.errors import UserAlreadyExistsError
from user.domain.usecase.user.create import CreateInput


def make_sut():
    repository = FakeUserRepository()
    publisher = FakeUserEventPublisher()
    return CreateUseCase(user_repository=repository, user_event=publisher), repository, publisher


async def test_persists_user_and_returns_output_with_id() -> None:
    sut, repository, _ = make_sut()

    output = await sut.perform(CreateInput(name="Ana", email="ana@example.com"))

    assert output.user.id
    assert await repository.find_by_id(output.user.id) is not None


async def test_raises_when_email_already_taken() -> None:
    sut, _, _ = make_sut()
    await sut.perform(CreateInput(name="Ana", email="ana@example.com"))

    with pytest.raises(UserAlreadyExistsError):
        await sut.perform(CreateInput(name="Bia", email="ana@example.com"))
```

Regras:
- Nomeie por comportamento: `test_<acao>_when_<condicao>` / `test_raises_<erro>_when_<condicao>`.
- Um comportamento por teste. Multiplas asercoes sobre o **mesmo** resultado sao OK.
- Teste os caminhos de erro de infra: injete mock com `side_effect=RuntimeError("db down")` e asserte que o use case propaga com `from` ou converte no erro de dominio correto.

## 5. Testes de Integracao com `testcontainers`

Referencia: https://testcontainers-python.readthedocs.io/

Sobem **dependencias reais em containers** (PostgreSQL 18, NATS 2.x) para testar repositorios, publishers e subscribers contra o engine de verdade. Ver [[testcontainers-python]] para o guia completo. Resumo do padrao:

```python
# tests/integration/test_user_repository.py
import asyncpg
import pytest

from user.infrastructure.repository.user_repository import PostgresUserRepository
from user.domain.entity.user import User

pytestmark = pytest.mark.integration


async def test_persists_and_reads_user(pg_pool: asyncpg.Pool) -> None:
    # pg_pool vem de uma fixture que sobe PostgreSqlContainer + aplica migrations (dbmate)
    repository = PostgresUserRepository(FakeDatabase(pg_pool))

    user = User.create("Ana", "ana@example.com")
    await repository.create(user)

    found = await repository.find_by_id(user.id)
    assert found is not None
    assert found.email == "ana@example.com"


async def test_returns_none_when_not_found(pg_pool: asyncpg.Pool) -> None:
    repository = PostgresUserRepository(FakeDatabase(pg_pool))
    assert await repository.find_by_id("00000000-0000-0000-0000-000000000000") is None
```

Regras criticas (detalhadas em [[testcontainers-python]]):
- Container compartilhado por **modulo/sessao** via fixture (`scope="session"`/`"module"`), isolamento por `TRUNCATE` em fixture `function`.
- **Nunca** hardcode porta — use `container.get_exposed_port(5432)` / `get_connection_url()`.
- Feche pool (`await pool.close()`) **antes** de parar o container.
- `event_loop` scope compativel com o dos containers (evitar "attached to different loop").

## 6. Testes E2E de API (Robyn + httpx)

Sobem a API real em **porta livre** (subprocess), com container de Postgres/NATS por tras, e exercitam por HTTP com `httpx`. Ver [[robyn-best-practices]] §11 para o helper de subprocess. Resumo:

```python
# tests/integration/test_user_api.py
import httpx
import pytest

pytestmark = pytest.mark.integration


def test_returns_201_with_created_user(api_base_url: str) -> None:
    resp = httpx.post(
        f"{api_base_url}/v1/users",
        json={"name": "Ana", "email": "ana@example.com"},
    )
    assert resp.status_code == 201
    body = resp.json()
    assert body["email"] == "ana@example.com"
    assert body["id"]


def test_returns_409_when_email_exists(api_base_url: str) -> None:
    httpx.post(f"{api_base_url}/v1/users", json={"name": "Ana", "email": "ana@example.com"})

    resp = httpx.post(f"{api_base_url}/v1/users", json={"name": "Bia", "email": "ana@example.com"})
    assert resp.status_code == 409


def test_returns_400_on_invalid_body(api_base_url: str) -> None:
    resp = httpx.post(f"{api_base_url}/v1/users", json={"name": "A"})
    assert resp.status_code == 400
```

Regras:
- **Porta livre sempre** (bind `:0`), URL montada com a porta real.
- `httpx` nativo — sem libs HTTP extras.
- E2E cobre o **contrato HTTP**: status, shape do JSON (camelCase), mapeamento erro de dominio -> status. Regra de negocio fina fica no unitario.
- Teste os 3 tipos de resposta de cada endpoint critico: sucesso, erro de dominio mapeado (404/409/422), erro de validacao (400).
- `proc.terminate()` no teardown da fixture.

## 7. Cobertura e Mutation Testing

### Cobertura com pytest-cov

```bash
uv run pytest --cov --cov-report=term-missing
```

- Configurado no `[tool.coverage.run]` (secao 1) — exclui `bin/` (composicao) e testes.
- Cobertura e **detector de buraco**, nao meta de vaidade: 100% de linha com asercoes fracas e pior que 85% com asercoes fortes — por isso o projeto usa mutation testing.

### Mutation testing com mutmut

```bash
uv run mutmut run --paths-to-mutate apps/user/src/user/domain,apps/user/src/user/application
uv run mutmut results
```

Regras:
- Mute **dominio e aplicacao** (onde vive a regra de negocio); nao mute `infrastructure/` nem `bin/` — mutantes de SQL/wiring geram ruido.
- Roda contra os **unitarios** (rapidos) — mutation testing sobre containers e inviavel.
- Mutante sobrevivente = teste fraco: fortaleca a **asercao** (valores exatos, erro exato), nao escreva teste que apenas executa a linha.
- **Threshold: >= 80%** de mutantes mortos nos modulos da feature.

## 8. Convencoes Transversais

### Builders de fixture

Para entidades com muitos campos, use builder com defaults validos e override pontual:

```python
# tests/builders/user_builder.py
from uuid import uuid4
from datetime import UTC, datetime

from user.domain.entity.user import User


def make_user(**overrides: object) -> User:
    defaults: dict[str, object] = {
        "id": str(uuid4()),
        "name": "Ana",
        "email": f"{uuid4()}@example.com",
        "created_at": datetime(2026, 1, 1, tzinfo=UTC),
        "updated_at": datetime(2026, 1, 1, tzinfo=UTC),
    }
    return User(**{**defaults, **overrides})  # type: ignore[arg-type]
```

- Defaults sempre **validos**; o teste sobrescreve apenas o relevante.
- Email/ID aleatorios por default — evita colisao de indice unico em integracao.

### Nao teste a implementacao, teste o comportamento

- ❌ "verifica que `find_by_email` foi chamado 1x antes de `create`"
- ✅ "depois de `perform`, `find_by_email` retorna o usuario criado"

### Determinismo

- Nada de `datetime.now()`/`random` cru no SUT sem seam (freezegun, `Clock` injetado ou builder).
- Nenhum teste depende de ordem de execucao — cada `test_` recria estado.
- Nenhum `sleep` real para sincronizar: unitario controla o tempo; integracao espera por condicao (poll com timeout).

## 9. Checklist de Revisao

Ao escrever ou revisar testes, verifique:

### Geral
- [ ] Unitario em `tests/unit/` espelhando `src/`; integracao/E2E em `tests/integration/` com marker `integration`
- [ ] `asyncio_mode = "auto"` e `addopts = "-m 'not integration'"` no pyproject
- [ ] AAA com blocos separados; um comportamento por teste
- [ ] Nomes em linguagem de dominio (`test_raises_x_when_y`)
- [ ] `pytest.mark.parametrize` quando houver >= 3 casos
- [ ] Nenhum teste depende de ordem/estado de outro

### Dublês
- [ ] Fake/mock **injetado por constructor** — `patch` so como ultimo recurso
- [ ] Fakes satisfazem o Protocol e contrato de `None` para nao encontrado
- [ ] `AsyncMock(spec=Protocol)`; `assert_awaited*` para interacao
- [ ] Asercao de interacao apenas quando a interacao **e** o comportamento

### Tempo
- [ ] `freeze_time` ou `Clock` injetado para datas determinísticas
- [ ] Nenhum sleep real em unitario

### Integracao (testcontainers)
- [ ] Container compartilhado por sessao/modulo; isolamento por `TRUNCATE` em fixture function
- [ ] Modulo oficial (`PostgresContainer`) quando existe
- [ ] `get_exposed_port`/`get_connection_url` — nunca porta hardcoded
- [ ] Pool fechado antes do `container.stop()`; event loop scope compativel
- [ ] Marker `integration` e timeout generoso

### E2E
- [ ] API em porta livre (`:0`), URL com a porta real, poll no `/health`
- [ ] `httpx` nativo — sem lib HTTP extra
- [ ] Cobre sucesso, erro de dominio mapeado e erro de validacao por endpoint critico
- [ ] `proc.terminate()` no teardown

### Cobertura e mutacao
- [ ] Coverage excluindo `bin/` e testes; `fail_under = 80`
- [ ] mutmut mutando apenas `domain/` e `application/`, rodando os unitarios
- [ ] Mutante sobrevivente tratado fortalecendo asercao

## 10. Fontes Oficiais

- https://docs.pytest.org/en/stable/
- https://pytest-asyncio.readthedocs.io/
- https://docs.python.org/3/library/unittest.mock.html (AsyncMock)
- https://testcontainers-python.readthedocs.io/
- https://mutmut.readthedocs.io/
- https://www.python-httpx.org/

Combine sempre com [[python-best-practices]] para convencoes gerais da linguagem (async, erros com `from`, None-safety, tipos).
