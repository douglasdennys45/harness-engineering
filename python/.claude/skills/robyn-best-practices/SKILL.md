---
name: robyn-best-practices
description: Aplica melhores praticas oficiais do Robyn (robyn.tech, github.com/sparckles/Robyn) ao escrever, revisar ou refatorar APIs HTTP em Python. Cobre inicializacao do `Robyn(__file__)`, roteamento com decorators (@app.get/post/put/patch/delete, path params `:param`, wildcards, SubRouter com prefix e app.include_router, ordem de registro antes do app.start), handlers sync e async, o objeto Request (json(), body, path_params, query_params, headers, ip_addr, method, form_data, files), tipos de resposta (str, dict, tupla (body, status), Response com status_code/headers/description), middlewares (@app.before_request/@app.after_request globais e por rota, short-circuit ao retornar Response), lifecycle (@app.startup_handler/@app.shutdown_handler), exception handler global (@app.exception), dependency injection nativa (app.inject/inject_global e parametros reservados), validacao de input com Pydantic no boundary (model_validate + 400 padronizado via bind_and_handle), geracao nativa de OpenAPI (/docs, /openapi.json, OpenAPIInfo, openapi_tags, Body/QueryParams), escala multiprocesso (--processes/--workers, shared-nothing, --fast, --dev) e testes subindo a API em porta livre com httpx. Use sempre que criar/editar arquivos .py que importam robyn, ao desenhar rotas, controllers, middlewares ou exception handlers HTTP, ao validar input com Pydantic no boundary HTTP, ao configurar startup/shutdown da API, ou ao revisar PRs que tocam a camada HTTP de um servico Python.
---

# Robyn Best Practices (Documentacao Oficial)

Skill baseada na documentacao oficial em https://robyn.tech e https://github.com/sparckles/Robyn. Todas as recomendacoes aqui devem ser seguidas ao escrever, revisar ou refatorar codigo que use o pacote `robyn`.

Robyn e um framework web Python **async com runtime em Rust** (tokio). A API e baseada em decorators (estilo Flask/FastAPI), mas o transporte e o event loop de I/O rodam em Rust — isso muda a estrategia de escala (multiprocesso shared-nothing), o modelo de OpenAPI (nativo) e os testes (porta real + httpx).

Esta skill **complementa** [[python-best-practices]] (mypy strict, asyncio, tipos) e [[python-testing-best-practices]] (pytest). Sempre carregue-as em conjunto ao trabalhar com a camada HTTP.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.py` que importe `robyn`
- Ao desenhar rotas, controllers, middlewares ou exception handlers
- Ao implementar parsing/validacao de request com Pydantic, ou tratamento centralizado de erros
- Ao configurar startup/shutdown da API
- Ao revisar PRs que tocam a camada HTTP de um servico Python

## Pre-requisitos

- **Python 3.13** (Robyn suporta 3.10–3.14)
- Pacote: `robyn` (traz o runtime Rust compilado)
- mypy strict, imports absolutos

```python
from robyn import Robyn, Request, Response
```

## 1. Inicializacao do App

Referencia: https://robyn.tech/documentation/en/api_reference/getting_started

### Padrao idiomatico

```python
from robyn import Robyn

app = Robyn(__file__)   # __file__ e obrigatorio — Robyn usa para resolver paths/hot-reload

# ... registrar rotas, middlewares, handlers ANTES de start ...

app.start(host="0.0.0.0", port=8080)
```

- `Robyn(__file__)` — o primeiro argumento posicional e sempre `__file__`.
- `app.start(host, port)` inicia o servidor (bloqueante). Em dev, use a CLI com `--dev` para hot reload: `uv run robyn apps/user/src/user/bin/api.py --dev`.
- Com OpenAPI, passe `openapi=OpenAPI(...)` no construtor (ver §9).

### Regras absolutas

- **Registre TODAS as rotas, middlewares e handlers ANTES de `app.start()`.** O roteamento e compilado no startup — nao ha registro dinamico em runtime.
- **Sempre configure `@app.exception`** (ver §6) — sem ele, erro nao tratado vira 500 generico do runtime.
- `app.start()` e **bloqueante** e chamado uma unica vez no `main()` do binario (`bin/api.py`).

## 2. Roteamento

Referencia: https://robyn.tech/documentation/en/api_reference/routing

### Metodos HTTP (decorators)

```python
@app.get("/users/:id")
async def get_user(request: Request):
    return {"id": request.path_params["id"]}

@app.post("/users")
async def create_user(request: Request):
    data = request.json()
    return {"created": data}, 201

@app.put("/users/:id")
async def update_user(request: Request): ...

@app.patch("/users/:id")
async def patch_user(request: Request): ...

@app.delete("/users/:id")
async def delete_user(request: Request): ...
```

- Decorators: `@app.get/post/put/patch/delete/head/options`.
- Handlers podem ser **`async def`** (padrao do projeto — toda camada e async) ou `def` sync (Robyn roda sync handlers em thread pool). **Use sempre `async def`** neste projeto.

### Path params e wildcards

| Sintaxe | Significado | Acesso |
|---|---|---|
| `:name` | Parametro nomeado | `request.path_params["name"]` |
| `*rest` | Wildcard catch-all | `request.path_params["rest"]` (ex: `/files/*rest` casa `/files/a/b/c`) |

```python
@app.get("/users/:id/orders/:order_id")
async def handler(request: Request):
    user_id = request.path_params["id"]
    order_id = request.path_params["order_id"]
```

### Ordem de registro importa

Declare **rotas especificas antes** de wildcards e genericas, e registre **tudo antes de `app.start()`**:

```python
@app.get("/users/me")        # especifica primeiro
async def current_user(request: Request): ...

@app.get("/users/:id")       # generica depois
async def user_by_id(request: Request): ...
```

### `SubRouter`: composicao modular

Cada contexto de dominio expoe seu proprio `SubRouter` com prefixo; o binario monta tudo via `include_router`:

```python
from robyn import Robyn, SubRouter

def create_user_router() -> SubRouter:
    router = SubRouter(__file__, prefix="/v1/users")

    @router.get("/:id")
    async def get_by_id(request: Request): ...

    @router.post("/")
    async def create(request: Request): ...

    return router
```

```python
# bin/api.py — composicao
app.include_router(create_user_router())   # -> GET /v1/users/:id, POST /v1/users/
```

- `SubRouter(__file__, prefix="/v1/users")` — `__file__` obrigatorio, `prefix` acumula no path.
- No projeto, o `register_routes(app)` do controller cria o SubRouter e chama `app.include_router(...)` (ver §Controller).
- **O decorator do SubRouter tambem pode ser chamado como funcao**: `router.post("/")(handler)` — o padrao do `BaseController` usa essa forma para passar o handler ja embrulhado por `handle`/`bind_and_handle`.

## 3. Request

Referencia: https://robyn.tech/documentation/en/api_reference/request_object

### Atributos e metodos

| Membro | Tipo | Descricao |
|---|---|---|
| `request.method` | `str` | Metodo HTTP |
| `request.url` | `Url` | Objeto de URL — `request.url.path` para o path |
| `request.headers` | `Headers` | `request.headers.get("x-request-id")` (case-insensitive) |
| `request.path_params` | `dict[str, str]` | Path params (`:name`) |
| `request.query_params` | `QueryParams` | Query params — `.get(key)` ou `.to_dict()` (valores em `list[str]`) |
| `request.body` | `str | bytes` | Body cru |
| `request.json()` | `dict/list/...` | **Parseia** o body como JSON (preserva tipos: null->None, number->int/float) |
| `request.ip_addr` | `str | None` | IP da conexao |
| `request.form_data` | `dict` | Form url-encoded |
| `request.files` | `dict[str, bytes]` | Uploads multipart |

### Body: `request.json()`

```python
@app.post("/users")
async def create(request: Request):
    data = request.json()   # dict com tipos preservados
    return {"received": data}
```

- **`request.json()` lanca em JSON invalido** — sempre capture e converta em 400 (o `bind_and_handle` do projeto faz isso e valida com Pydantic em seguida).
- No projeto, **nunca** use `request.json()` cru no handler de negocio — sempre via `bind_and_handle(Model, handler)` que parseia + valida com Pydantic.

### Query params

```python
@app.get("/users")
async def list_users(request: Request):
    raw = {k: v[0] for k, v in request.query_params.to_dict().items() if v}
    # no projeto: parse_query(ListUsersQuery, request) valida com Pydantic
```

- `request.query_params.to_dict()` retorna `dict[str, list[str]]` — pegue o primeiro valor de cada.
- Coercao de tipos (str -> int) e responsabilidade do Pydantic no `parse_query`.

### Regras
- Headers via `request.headers.get(...)` (case-insensitive).
- Path params sempre `str` — converta/valide via Pydantic quando numericos.
- Leia o body uma vez (`request.json()` / `request.body`).

## 4. Response

Referencia: https://robyn.tech/documentation/en/api_reference/response-objects

### Tipos de retorno de handler

Robyn aceita varios tipos de retorno — o projeto **padroniza `Response` via helper `json_response`** para consistencia (status + Content-Type + serializacao):

```python
# 1. str -> 200 text/plain
return "hello"

# 2. dict -> 200 application/json (auto)
return {"status": "ok"}

# 3. tupla (body, status_code)
return {"created": True}, 201

# 4. Response explicito (PADRAO DO PROJETO)
from robyn import Response, Headers
return Response(
    status_code=201,
    headers=Headers({"Content-Type": "application/json"}),
    description=json.dumps(payload),   # `description` e o corpo da resposta
)
```

**Atencao ao `Response`:** o corpo vai no parametro **`description`** (nome do campo no Robyn), e `headers` e um objeto `Headers`. O helper `json_response` do `pkg/controller/responses.py` encapsula isso — use-o sempre:

```python
from org_pkg.controller import json_response

return json_response(201, {"id": user.id, "name": user.name})
```

### Regras
- **Padronize `json_response(status, data)`** em todos os handlers — nunca monte `Response` cru espalhado.
- Serializacao de `datetime`/`UUID` fica no `json_response` (via `json.dumps(default=...)`).
- Um handler sempre retorna algo (Response/dict/tupla) — nunca `None` implicito.

## 5. Middlewares

Referencia: https://robyn.tech/documentation/en/api_reference/middlewares

### `before_request` e `after_request`

```python
@app.before_request()
async def add_request_id(request: Request):
    # pode modificar o request (ex: injetar em request.headers)
    return request   # DEVE retornar o Request para a cadeia continuar

@app.after_request()
async def add_headers(response: Response):
    response.headers.set("X-Content-Type-Options", "nosniff")
    return response  # DEVE retornar o Response
```

- `before_request` recebe e **retorna** `Request`. Se retornar um `Response`, a cadeia curto-circuita e o handler nao roda (util para auth: retornar 401 direto).
- `after_request` recebe e **retorna** `Response` — roda sobre a resposta de saida.
- Por rota: `@app.before_request("/v1/admin")` aplica so ao path.

### Padrao do projeto: contexto e error handling ficam no BaseController

Neste harness, **request context (request_id, logger), validacao e mapeamento de erro vivem no `BaseController`** (template method), nao em middlewares globais — isso mantem o fluxo tipado e testavel por controller. Middlewares globais ficam para concerns transversais simples (security headers, CORS, access log).

### Regras
- `before_request` **sempre retorna** `Request` (ou um `Response` para curto-circuitar).
- `after_request` **sempre retorna** `Response`.
- Middlewares async (padrao do projeto). Nao coloque logica de negocio em middleware.
- Auth: `before_request` que valida o header e retorna `Response(401)` quando invalido (curto-circuita).

## 6. Exception Handler Global

Referencia: https://robyn.tech/documentation/en/api_reference/exceptions

### Filosofia: lance o erro de dominio, o BaseController mapeia

No projeto, o **`BaseController._handle_error`** (via `ErrorMapper`) e a primeira linha: handlers lancam erros de dominio, o BaseController captura e responde o status certo. O `@app.exception` e a **rede de seguranca** para o que escapar.

```python
from robyn import Response

@app.exception
def handle_uncaught(error: Exception) -> Response:
    # safety net — erro que escapou do BaseController vira 500 padronizado.
    # NUNCA vaze detalhes internos em 5xx.
    return json_response(500, {"error": "internal server error", "status": 500})
```

- `@app.exception` recebe a `Exception` e **retorna** um `Response`.
- Registrado uma vez no bootstrap (`bin/api.py`).
- O mapeamento fino de erros de dominio -> HTTP status e feito pelo `ErrorMapper` **dentro** do `BaseController` (ver `architecture.md`), nao aqui — o `@app.exception` so pega o que vazou.

## 7. Validacao com Pydantic no Boundary

Todo input HTTP e validado com **Pydantic v2** no boundary, antes de chegar ao use case. O padrao do harness e o `bind_and_handle` do `BaseController`:

```python
from pydantic import BaseModel, EmailStr, Field

class CreateUserBody(BaseModel):
    name: str = Field(min_length=2, max_length=100)
    email: EmailStr

# no controller:
router.post("/")(self.bind_and_handle(CreateUserBody, self._create))

async def _create(self, request, rc, body: CreateUserBody):
    # body ja validado e tipado — 400 automatico se invalido
    output = await self._create_user_use_case.perform(CreateInput(name=body.name, email=str(body.email)))
    return json_response(201, user_to_json(output.user))
```

O `bind_and_handle` (em `pkg/controller/base_controller.py`) faz:
1. `request.json()` com try/except -> `ValidationError` (400) se JSON invalido.
2. `Model.model_validate(payload)` -> `ValidationError` (400) com mensagem formatada se schema invalido.
3. Chama o handler com o modelo ja tipado.

### Query params

```python
class ListUsersQuery(BaseModel):
    cursor: str = ""
    limit: int = Field(default=20, ge=1, le=100)

query = parse_query(ListUsersQuery, request)   # 400 se invalido
```

### Regras
- **Sempre valide via Pydantic** no boundary (`bind_and_handle`/`parse_query`) — nunca `request.json()` cru no handler de negocio.
- Modelos de input/output na **fronteira camelCase**: estendem uma base com `alias_generator=to_camel` (ver `architecture.md`); internamente snake_case.
- `EmailStr`, `UUID`, `Literal[...]`, `Field(min_length=, gt=, ...)` para as regras.
- Nunca exponha o erro cru do Pydantic — formate com `format_validation_error`.
- Schemas vivem no arquivo do controller (infrastructure), nunca no domain.

## 8. Startup, Shutdown e Ciclo de Vida

Referencia: https://robyn.tech/documentation/en/api_reference/middlewares (lifecycle handlers)

Conexoes (pool asyncpg, NATS) sao criadas **no event loop do servidor**, via startup handler — nunca no import:

```python
@app.startup_handler
async def on_startup() -> None:
    await container.database().connect()
    await container.nats_client().connect()
    logger.info("api started", port=config.app_port)

@app.shutdown_handler
async def on_shutdown() -> None:
    # ordem: NATS drain (aguarda publishes), depois pool PG close
    await container.nats_client().drain()
    await container.database().disconnect()
    logger.info("shutdown complete")
```

- `@app.startup_handler` (async) — abre conexoes/streams **depois** que o event loop do Robyn subiu.
- `@app.shutdown_handler` (async ou sync) — fecha na ordem: mensageria (drain) -> banco (close).
- **Nunca** crie pool asyncpg / conexao NATS no import ou no constructor do container — eles ficam presos ao event loop errado.

## 9. OpenAPI Nativo

Referencia: https://robyn.tech/documentation/en/api_reference/openapi

Robyn gera OpenAPI **nativamente** — sem lib extra. Endpoints automaticos: `/docs` (Swagger UI) e `/openapi.json`.

```python
from robyn import Robyn
from robyn.openapi import OpenAPI, OpenAPIInfo, Contact, License

app = Robyn(
    __file__,
    openapi=OpenAPI(
        info=OpenAPIInfo(
            title="User API",
            description="Servico de usuarios",
            version="1.0.0",
            contact=Contact(name="Squad", email="squad@example.com"),
            license=License(name="MIT"),
        ),
    ),
)
```

### Documentando rotas

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str

@app.post("/users", openapi_tags=["users"])
async def create_user(request: Request, body: UserCreate) -> dict:
    """Cria um novo usuario."""   # docstring vira a descricao no OpenAPI
    return {"name": body.name}
```

- **A docstring do handler** vira a descricao do endpoint. No projeto, o `BaseController` usa `functools.wraps` para preservar nome + docstring do metodo concreto ao embrulhar em `handle`/`bind_and_handle`.
- `openapi_tags=[...]` agrupa endpoints; `openapi_name=...` nomeia a operacao.
- **Os modelos Pydantic sao os schemas** — request body e query params tipados aparecem no spec automaticamente.
- Exportar spec para arquivo (para o QA comparar): `curl -s http://localhost:8080/openapi.json > docs/openapi.json`.
- Desabilitar (raro): `--disable-openapi` na CLI.

## 10. Escala Multiprocesso (shared-nothing)

Referencia: https://robyn.tech/documentation/en/api_reference/scaling

Robyn escala com **modelo multiprocesso shared-nothing** — cada processo tem seu proprio interpretador e memoria (sem contencao de GIL entre processos).

```bash
# dev — hot reload
uv run robyn apps/user/src/user/bin/api.py --dev

# producao — otimizacao automatica
uv run robyn apps/user/src/user/bin/api.py --fast

# manual: N processos x M workers
uv run robyn apps/user/src/user/bin/api.py --processes 4 --workers 2
```

| Flag | Efeito |
|---|---|
| `--processes N` | N processos independentes (memoria isolada, sem GIL compartilhado) |
| `--workers M` | M worker threads por processo (bom para I/O) |
| `--fast` | Otimizacao automatica de processes/workers para a maquina |
| `--dev` | Hot reload (nunca em producao; incompativel com `--processes`) |

### Regras de escala
- **Estado nao e compartilhado entre processos** — nunca guarde estado mutavel em memoria de processo (cache, sessao, contador). Use PostgreSQL/NATS/Redis para estado compartilhado.
- No projeto (containers), o padrao e **1 replica de container = 1 stack**; escala horizontal via mais replicas atras de um LB. `--processes`/`--workers` afinam o uso de cada replica.
- I/O-bound (nosso caso: DB + NATS): `--processes` = nucleos/2, `--workers` 2-4. CPU-bound: `--processes` = nucleos, `--workers` 1.
- Conexoes (pool asyncpg, NATS) sao criadas **por processo** no startup handler — cada processo tem seu proprio pool.

## 11. Testes

Robyn **nao expoe um handle de teste in-process** — o padrao e subir a API real em **porta livre** (subprocess) e exercita-la com **`httpx`**:

```python
# tests/integration/testhelper/api.py
import socket
import subprocess
import time
import httpx


def _free_port() -> int:
    with socket.socket() as s:
        s.bind(("127.0.0.1", 0))
        return s.getsockname()[1]


def start_api(env: dict[str, str]) -> tuple[subprocess.Popen, str]:
    port = _free_port()
    proc = subprocess.Popen(
        ["uv", "run", "python", "-m", "user.bin.api"],
        env={**env, "APP_PORT": str(port), "APP_HOST": "127.0.0.1"},
    )
    base_url = f"http://127.0.0.1:{port}"

    # poll no /health ate a API responder (nunca sleep fixo)
    for _ in range(50):
        try:
            if httpx.get(f"{base_url}/health", timeout=1).status_code == 200:
                break
        except httpx.HTTPError:
            time.sleep(0.2)
    else:
        proc.terminate()
        raise RuntimeError("api did not start")

    return proc, base_url
```

```python
# tests/integration/test_user_api.py
import httpx
import pytest

pytestmark = pytest.mark.integration


def test_create_user_returns_201(api_base_url: str):
    resp = httpx.post(
        f"{api_base_url}/v1/users",
        json={"name": "Ana", "email": "ana@example.com"},
    )
    assert resp.status_code == 201
    body = resp.json()
    assert body["email"] == "ana@example.com"
```

### Regras
- **Porta livre sempre** (bind em `:0`) — nunca porta fixa (colide em CI/paralelismo).
- **`httpx`** como client — sem libs extras.
- Poll no `/health` para aguardar readiness — **nunca sleep fixo**.
- `proc.terminate()` no teardown (fixture) — processo vazado segura o pytest.
- Para E2E com banco/NATS reais, combine com testcontainers ([[testcontainers-python]]): suba os containers, injete `DATABASE_URL`/`NATS_URL` no env do subprocess.
- Teste o **contrato HTTP** (status, JSON, headers); regra de negocio fina fica no unitario.

## Checklist de Revisao (Robyn)

Ao revisar ou aprovar codigo Robyn, verifique:

- [ ] `Robyn(__file__)` com `__file__`; OpenAPI configurado quando ha API publica
- [ ] `@app.exception` registrado no bootstrap (safety net -> 500 generico, sem vazar interno)
- [ ] Todas as rotas registradas **antes** de `app.start(...)`
- [ ] Rotas especificas declaradas antes de wildcards/genericas
- [ ] Controllers montados via `SubRouter(__file__, prefix=...)` + `app.include_router(...)`
- [ ] Body validado com Pydantic via `bind_and_handle` — nunca `request.json()` cru no handler
- [ ] Query params validados com `parse_query` (Pydantic) quando numericos/enum
- [ ] Handlers lancam erros de dominio — o BaseController/ErrorMapper mapeia para status
- [ ] Respostas via `json_response(status, data)` — nunca `Response` cru espalhado
- [ ] `before_request` retorna `Request`; `after_request` retorna `Response`
- [ ] Conexoes (pool, NATS) abertas no `@app.startup_handler`, nunca no import/constructor
- [ ] Shutdown: `@app.shutdown_handler` -> NATS drain -> pool disconnect
- [ ] Docstrings + `openapi_tags` nos handlers (viram OpenAPI); spec verificavel em `/docs`
- [ ] Sem estado mutavel em memoria de processo (shared-nothing)
- [ ] Testes sobem a API em porta livre + `httpx`, com poll no `/health` e `terminate()` no teardown

## Fontes Oficiais

Toda recomendacao acima vem destas paginas — consulte-as ao bater duvida:

- https://robyn.tech (visao geral, features)
- https://robyn.tech/documentation/en/api_reference/getting_started (app, rotas, start)
- https://robyn.tech/documentation/en/api_reference/request_object (Request)
- https://robyn.tech/documentation/en/api_reference/response-objects (Response)
- https://robyn.tech/documentation/en/api_reference/middlewares (before/after request, lifecycle)
- https://robyn.tech/documentation/en/api_reference/exceptions (exception handler)
- https://robyn.tech/documentation/en/api_reference/dependency_injection (DI nativa)
- https://robyn.tech/documentation/en/api_reference/openapi (OpenAPI nativo)
- https://robyn.tech/documentation/en/api_reference/scaling (processes/workers/fast)
- https://github.com/sparckles/Robyn (repositorio, exemplos)
