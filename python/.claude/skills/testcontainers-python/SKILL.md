---
name: testcontainers-python
description: Use esta skill ao escrever testes de integracao Python com containers Docker usando a biblioteca testcontainers-python — modulos pre-configurados (PostgresContainer, NatsContainer/DockerContainer), wait strategies (wait_for_logs, wait_container_is_ready), networking entre containers e helpers compartilhados em tests/integration/testhelper/. Cobre ciclo de vida com pytest (fixtures scope session/module/function, event_loop scope), aplicacao de migrations dbmate no container, padroes de cleanup/isolamento (TRUNCATE vs banco por teste) e integracao com asyncpg e nats-py (JetStream). Acione ao criar/editar testes em apps/<app>/tests/integration/, configurar infraestrutura de teste baseada em containers ou depurar containers de teste.
---

# Testcontainers para Testes de Integracao em Python

Guia para usar a biblioteca **testcontainers-python** para escrever testes de integracao confiaveis com containers Docker em projetos Python + pytest.

## Descricao

Testcontainers fornece instancias leves e descartaveis de bancos, message brokers ou qualquer servico Docker — controladas a partir do proprio codigo de teste.

**Capacidades principais:**
- Usar modulos pre-configurados (`testcontainers.postgres.PostgresContainer`, etc.)
- Subir e gerenciar containers Docker dentro de suites pytest
- Configurar env, wait strategies e networking
- Compartilhar containers entre testes via fixtures em `tests/integration/testhelper/`
- Aplicar migrations (dbmate) no container antes dos testes
- Implementar cleanup e isolamento corretos entre testes

## Quando Usar

- Testar repositorios `asyncpg` contra um PostgreSQL 18 real, sem mocks
- Testar publishers/subscribers NATS JetStream contra um broker real
- Criar ambientes de teste reproduziveis, sem infra local pre-instalada
- E2E da API Robyn com banco/NATS reais por tras

**Convencoes do projeto:**
- Testes de integracao vivem em `apps/<app>/tests/integration/`
- Helpers compartilhados em `apps/<app>/tests/integration/testhelper/` (`postgres.py`, `nats.py`, `api.py`)
- Runner: **pytest**. Banco: **PostgreSQL 18** via `asyncpg`. Mensageria: **NATS JetStream** via `nats-py`. Migrations: **dbmate**.
- Marker: todo teste de integracao tem `pytestmark = pytest.mark.integration` (nao roda por padrao).

## Pre-requisitos

- **Docker** (ou Podman/Colima) instalado e rodando
- **Python 3.13** e **uv** (workspace do monorepo)
- Pacote: `testcontainers[postgres]` (devDependency)

## 1. Instalacao & Setup

```bash
uv add --dev "testcontainers[postgres]"   # nucleo + modulo postgres
uv add --dev httpx                          # cliente HTTP para E2E
```

Config pytest (ver [[python-testing-best-practices]] §1): `asyncio_mode = "auto"`, marker `integration`, `addopts = "-m 'not integration'"`.

**Verificar Docker** antes de rodar:

```bash
docker info > /dev/null 2>&1 || echo "Docker indisponivel — testes de integracao serao pulados"
```

## 2. PostgreSQL — Modulo Pre-Configurado

O modulo `PostgresContainer` traz defaults sensatos, credenciais e um `get_connection_url()`.

```python
# tests/integration/testhelper/postgres.py
import subprocess
from collections.abc import AsyncIterator
from pathlib import Path

import asyncpg
import pytest_asyncio
from testcontainers.postgres import PostgresContainer

MIGRATIONS_DIR = Path(__file__).resolve().parents[4] / "migrations" / "user"


@pytest_asyncio.fixture(scope="session")
async def pg_container() -> AsyncIterator[PostgresContainer]:
    with PostgresContainer("postgres:18-alpine", driver=None) as container:
        _run_migrations(container.get_connection_url())
        yield container


@pytest_asyncio.fixture
async def pg_pool(pg_container: PostgresContainer) -> AsyncIterator[asyncpg.Pool]:
    # driver=None => URL no formato postgresql://... (asyncpg aceita)
    dsn = pg_container.get_connection_url().replace("postgresql+psycopg2", "postgresql")
    pool = await asyncpg.create_pool(dsn=dsn, min_size=1, max_size=5)
    try:
        await _truncate_all(pool)          # isolamento: banco limpo antes de cada teste
        yield pool
    finally:
        await pool.close()                 # SEMPRE fechar o pool antes do container parar


def _run_migrations(connection_url: str) -> None:
    dsn = connection_url.replace("postgresql+psycopg2", "postgresql")
    subprocess.run(
        ["dbmate", "--no-dump-schema", "--migrations-dir", str(MIGRATIONS_DIR), "up"],
        env={"DATABASE_URL": f"{dsn}?sslmode=disable"},
        check=True,
    )


async def _truncate_all(pool: asyncpg.Pool) -> None:
    rows = await pool.fetch(
        "SELECT tablename FROM pg_tables WHERE schemaname = 'public' AND tablename <> 'schema_migrations'"
    )
    if not rows:
        return
    tables = ", ".join(f'"{r["tablename"]}"' for r in rows)
    await pool.execute(f"TRUNCATE TABLE {tables} RESTART IDENTITY CASCADE")
```

**Metodos uteis do `PostgresContainer`:**

```python
container.get_connection_url()   # "postgresql+psycopg2://test:test@localhost:32789/test"
container.get_exposed_port(5432) # porta mapeada (aleatoria)
```

> `driver=None` (ou remover o `+psycopg2` da URL) e importante: o asyncpg quer o esquema `postgresql://`, nao `postgresql+psycopg2://`.

## 3. NATS — GenericContainer (DockerContainer)

Nao ha modulo NATS oficial estavel — use `DockerContainer` com wait por log:

```python
# tests/integration/testhelper/nats.py
from collections.abc import AsyncIterator

import nats
import pytest_asyncio
from nats.aio.client import Client
from testcontainers.core.container import DockerContainer
from testcontainers.core.waiting_utils import wait_for_logs


@pytest_asyncio.fixture(scope="session")
async def nats_url() -> AsyncIterator[str]:
    container = (
        DockerContainer("nats:2.12-alpine")
        .with_command("-js")               # habilita JetStream
        .with_exposed_ports(4222)
    )
    container.start()
    wait_for_logs(container, "Server is ready", timeout=30)
    host = container.get_container_host_ip()
    port = container.get_exposed_port(4222)
    try:
        yield f"nats://{host}:{port}"
    finally:
        container.stop()


@pytest_asyncio.fixture
async def nats_client(nats_url: str) -> AsyncIterator[Client]:
    nc = await nats.connect(nats_url)
    try:
        await _delete_all_streams(nc)      # isolamento entre testes
        yield nc
    finally:
        await nc.drain()


async def _delete_all_streams(nc: Client) -> None:
    jsm = nc.jsm()
    async for stream in jsm.streams_info():
        await jsm.delete_stream(stream.config.name)
```

## 4. Ciclo de Vida em pytest

### Escopo de fixtures

- **Container**: `scope="session"` (ou `"module"`) — sobe uma vez, amortiza o boot.
- **Pool/conexao + limpeza**: `scope="function"` (default) — TRUNCATE/delete streams antes de cada teste.
- Feche **clients antes do container** (pool.close, nc.drain no `finally`).

### `event_loop` e asyncio

Com `pytest-asyncio` em `asyncio_mode = "auto"`, fixtures async funcionam direto. **Cuidado com scope**: um pool asyncpg criado numa fixture `session` fica preso ao event loop daquela sessao. O padrao acima usa **container `session` + pool `function`** — o pool nasce e morre no loop de cada teste, evitando "attached to a different loop".

Se precisar de pool `session`, garanta `event_loop` tambem `session` (fixture custom) — mas o padrao container-session + pool-function e mais simples e seguro.

## 5. Wait Strategies

- **Postgres (modulo)**: o `PostgresContainer` ja embute a wait strategy correta — nao redefina.
- **NATS/DockerContainer**: `wait_for_logs(container, "Server is ready", timeout=30)` — nunca `time.sleep` fixo.
- Outras: `wait_container_is_ready()` (retry ate o comando de health passar).

```python
from testcontainers.core.waiting_utils import wait_for_logs
wait_for_logs(container, "Server is ready", timeout=30)
```

## 6. Migrations dbmate no Container

Migrations vivem em `migrations/<app>/` e sao aplicadas com **dbmate** contra o container **antes** dos testes:

```python
subprocess.run(
    ["dbmate", "--no-dump-schema", "--migrations-dir", str(MIGRATIONS_DIR), "up"],
    env={"DATABASE_URL": f"{dsn}?sslmode=disable"},   # sslmode=disable: container sem TLS
    check=True,
)
```

- `--no-dump-schema` evita reescrever `schema.sql` durante os testes.
- Rode **uma vez por container** (fixture `session`), nao por teste.
- `schema_migrations` **excluida do TRUNCATE**.

## 7. Cleanup e Isolamento

| Estrategia | Velocidade | Quando usar |
|---|---|---|
| **TRUNCATE em fixture function** (padrao) | ~ms | Default para repositorios |
| **Banco por modulo** (`CREATE DATABASE`) | Media | Suites com schemas conflitantes |
| **Container por teste** | Lenta | Testes destrutivos (config do servidor) |

TRUNCATE (padrao): `TRUNCATE TABLE ... RESTART IDENTITY CASCADE`, excluindo `schema_migrations`.

NATS: delete todos os streams entre testes, ou use nomes de stream unicos por teste.

### Ryuk (garbage collector)

O testcontainers sobe o **Ryuk** — remove containers mesmo se o pytest crashar. Nao desabilite (`TESTCONTAINERS_RYUK_DISABLED`) exceto em Podman rootless.

## 8. E2E de API Robyn com Container

Combine os containers com o subprocess da API (ver [[robyn-best-practices]] §11):

```python
# tests/integration/test_user_flow.py
import httpx
import pytest

pytestmark = pytest.mark.integration


def test_create_user_persists_and_returns_201(api_base_url: str) -> None:
    # a fixture api_base_url sobe: PostgresContainer + NATS + subprocess da API,
    # injetando DATABASE_URL/NATS_URL no env do processo Robyn
    resp = httpx.post(f"{api_base_url}/v1/users", json={"name": "Ana", "email": "ana@example.com"})
    assert resp.status_code == 201
    assert resp.json()["email"] == "ana@example.com"
```

A fixture `api_base_url` orquestra: sobe containers -> injeta `DATABASE_URL`/`NATS_URL` -> `subprocess.Popen(["uv","run","python","-m","user.bin.api"], env=...)` -> poll no `/health` -> `yield base_url` -> `proc.terminate()`.

## 9. Performance

- **Imagens alpine** (`postgres:18-alpine`, `nats:2.12-alpine`) — boot rapido; nunca `latest`.
- **Container por sessao**, nao por teste — 1 boot amortizado em N testes.
- Pool `asyncpg` com `max_size=5` nos testes — evita esgotar `max_connections`.
- `maxWorkers`/`-n` (pytest-xdist): limite o paralelismo se cada worker sobe sua stack.
- Pre-pull das imagens no CI (`docker pull postgres:18-alpine`) num step cacheavel.

## 10. Troubleshooting

- **`RuntimeError: attached to a different loop`**: pool criado em fixture com scope diferente do event loop. Use container `session` + pool `function`.
- **`ConnectionRefusedError`**: usou porta interna em vez de `get_exposed_port()`. Do host, sempre `get_container_host_ip()` + `get_exposed_port()`.
- **URL do asyncpg**: troque `postgresql+psycopg2://` por `postgresql://` (asyncpg nao entende o sufixo do driver).
- **pytest pendura no fim**: pool nao fechado (`await pool.close()`) ou conexao NATS sem `drain()`. Feche tudo no `finally` da fixture.
- **Timeout no boot**: `wait_for_logs(..., timeout=60)` — CI e mais lento; pre-pull da imagem ajuda.
- **Debug**: `DEBUG=testcontainers*` / logs do container via `container.get_logs()`.

## Boas Praticas

1. **Modulo pre-configurado quando existir** (`PostgresContainer`) — defaults e helpers de conexao
2. **Centralize o boot em `tests/integration/testhelper/`** (`postgres.py`, `nats.py`, `api.py`)
3. **Container por sessao, pool/limpeza por function** — isolamento sem re-boot
4. **Wait strategy correta** — `PostgresContainer` ja tem; NATS via `wait_for_logs`; nunca sleep
5. **`get_container_host_ip()` + `get_exposed_port()` do host** — elimina `ConnectionRefused`
6. **Migrations dbmate uma vez por container**, dentro da fixture session
7. **TRUNCATE em fixture function** (excluindo `schema_migrations`)
8. **Feche clients antes de containers** (pool.close, nc.drain no `finally`)
9. **Pin de versao de imagem** — nunca `latest`
10. **URL asyncpg sem `+psycopg2`**

## Recursos

- https://testcontainers-python.readthedocs.io/ — documentacao oficial
- https://testcontainers-python.readthedocs.io/en/latest/modules/postgres/ — modulo Postgres
- https://github.com/testcontainers/testcontainers-python — repositorio
- https://magicstack.github.io/asyncpg/current/ — asyncpg
- https://github.com/amacneil/dbmate — dbmate
