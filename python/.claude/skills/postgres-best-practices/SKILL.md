---
name: postgres-best-practices
description: Aplica boas praticas oficiais do PostgreSQL 18 ao modelar, escrever, revisar ou refatorar schema, queries, migrations e codigo de acesso a dados em Python com asyncpg. Cobre convencoes de nomenclatura em camelCase com aspas duplas, tipos recomendados (UUID, TIMESTAMPTZ, JSONB, BIGINT em centavos), primary keys, foreign keys, constraints nomeadas, indices (B-tree, GIN, BRIN, partial, compostos), soft delete, tabelas de juncao N:N, migrations zero-downtime (dbmate, SQL puro com -- migrate:up / -- migrate:down), transactions (run_in_tx com asyncpg, niveis de isolamento, eventos pos-commit), paginacao por cursor, queries idiomaticas (SELECT explicito, INSERT RETURNING, UPSERT, SELECT FOR UPDATE/SKIP LOCKED), pool de conexoes com asyncpg.create_pool e configuracao de servidor. Use sempre que criar/editar migrations SQL, modelar tabelas, escrever queries, implementar repositorios Python que falam com Postgres via asyncpg, definir indices, orquestrar transactions em use cases ou ajustar configuracao do banco. Acione ao trabalhar com arquivos .sql, repositorios que usam asyncpg (Pool/Connection), migrations em migrations/<app>/*, ou ao discutir performance/locks/isolation no PostgreSQL.
---

# PostgreSQL 18 Best Practices (Python · asyncpg)

Skill baseada nas praticas oficiais do PostgreSQL 18, do driver `asyncpg` e nos padroes internos do projeto para modelagem, queries, migrations, transactions e configuracao. Todas as recomendacoes aqui devem ser seguidas ao criar, revisar ou refatorar qualquer artefato que envolva o banco.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.sql` (DDL ou DML)
- Ao escrever ou revisar migrations em `migrations/<app>/*`
- Ao implementar repositorios Python que falam com PostgreSQL (`asyncpg`)
- Ao modelar novas tabelas, indices, constraints ou tipos
- Ao orquestrar transactions em use cases (multi-repo, multi-write)
- Ao definir paginacao, locking, isolamento ou estrategia de soft delete
- Ao ajustar configuracao do servidor (`postgresql.conf`, `docker-compose.yml`)
- Ao discutir performance, locks, race conditions ou planos de execucao (`EXPLAIN ANALYZE`)

## 1. Convencoes de Nomenclatura

| Elemento | Convencao | Exemplo |
|---|---|---|
| Tabela | camelCase, **plural** | `users`, `orderItems`, `auditLogs` |
| Coluna | camelCase | `createdAt`, `userId`, `firstName` |
| Primary Key | `id` | `id UUID PRIMARY KEY` |
| Foreign Key | `<entidade>Id` | `userId`, `orderId` |
| Constraint | `<tabela>_<descricao>` | `users_emailUnique`, `orders_statusCheck` |
| Index | `idx_<tabela>_<colunas>` | `idx_users_email`, `idx_orders_userId_createdAt` |
| Trigger | `trg_<tabela>_<acao>` | `trg_users_updatedAt` |
| Function | camelCase descritivo | `setUpdatedAt()` |
| Sequence | `<tabela>_<coluna>_seq` | `users_id_seq` |

### Regras inegociaveis

- PostgreSQL preserva case **apenas** quando o nome esta entre aspas duplas. Como usamos camelCase, **todas** as referencias a tabelas e colunas devem usar aspas duplas no SQL.
- Tabelas sempre no **plural**.
- Foreign key sempre nomeada como `<entidadeSingular>Id`.
- Constraints sempre nomeadas **explicitamente** — nunca deixar o PostgreSQL gerar nomes automaticos.

### camelCase no banco, snake_case no Python

O banco usa camelCase (`"createdAt"`); a entidade Python usa snake_case (`created_at`). A conversao acontece **no repositorio**, no metodo `_to_entity`:

```python
def _to_entity(self, row: asyncpg.Record) -> User:
    return User(
        id=str(row["id"]),
        name=row["name"],
        email=row["email"],
        created_at=row["createdAt"],   # coluna camelCase -> atributo snake_case
        updated_at=row["updatedAt"],
    )
```

- `asyncpg.Record` acessa colunas pelo **nome exato da coluna** (`row["createdAt"]`).
- `UUID` volta como `uuid.UUID` — converta com `str(row["id"])` para o dominio (que usa `str`).
- `TIMESTAMPTZ` volta como `datetime` **aware** (com tzinfo) — nunca naive.
- `BIGINT` volta como `int` do Python (sem limite de precisao) — diferente de outras stacks, nao ha conversao de string.

## 2. Padrao de Criacao de Tabelas

### Template base

```sql
CREATE TABLE "users" (
    -- identidade
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- dados de dominio
    "name"      TEXT NOT NULL,
    "email"     TEXT NOT NULL,
    "role"      TEXT NOT NULL DEFAULT 'viewer',

    -- timestamps
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- constraints nomeadas
    CONSTRAINT "users_emailUnique" UNIQUE ("email"),
    CONSTRAINT "users_roleCheck"   CHECK ("role" IN ('admin', 'editor', 'viewer'))
);

CREATE TRIGGER "trg_users_updatedAt"
    BEFORE UPDATE ON "users"
    FOR EACH ROW
    EXECUTE FUNCTION "setUpdatedAt"();

CREATE INDEX "idx_users_email"     ON "users" ("email");
CREATE INDEX "idx_users_createdAt" ON "users" ("createdAt" DESC);
```

### Funcao global `setUpdatedAt`

Criar **uma unica vez** na migration inicial:

```sql
-- migrations/<app>/0001_setup.sql

-- migrate:up
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE OR REPLACE FUNCTION "setUpdatedAt"()
RETURNS TRIGGER AS $$
BEGIN
    NEW."updatedAt" = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- migrate:down
DROP FUNCTION IF EXISTS "setUpdatedAt"();
```

## 3. Tipos de Dados

### Recomendados

| Caso de Uso | Tipo | asyncpg -> Python |
|---|---|---|
| Identificadores (PK/FK) | `UUID` | `uuid.UUID` (converta com `str(...)` no dominio) |
| Texto | `TEXT` | `str` |
| Inteiros | `INT` / `BIGINT` | `int` (sem perda de precisao) |
| Dinheiro | `BIGINT` (centavos) | `int` — nunca `float`/`NUMERIC` para dinheiro em API |
| Booleanos | `BOOLEAN` | `bool` |
| Data/hora | `TIMESTAMPTZ` | `datetime` **aware** (com tzinfo) |
| Apenas data | `DATE` | `date` |
| JSON estruturado | `JSONB` | `str` por padrao — **registre codec** para dict (ver abaixo) |
| Enumeracoes | `TEXT` + `CHECK` | `str` (valide com `Literal` no Pydantic) |
| Arrays | `TEXT[]` / `UUID[]` | `list[...]` |

**JSONB no asyncpg:** por padrao volta como `str`. Para trabalhar com `dict`, registre o codec no setup do pool:

```python
import json

async def _init_conn(conn: asyncpg.Connection) -> None:
    await conn.set_type_codec(
        "jsonb", encoder=json.dumps, decoder=json.loads, schema="pg_catalog"
    )

pool = await asyncpg.create_pool(dsn=..., init=_init_conn)
```

### Tipos a evitar

| Tipo | Problema | Usar em vez |
|---|---|---|
| `SERIAL` / `BIGSERIAL` | IDs previsiveis, problemas em sharding | `UUID DEFAULT gen_random_uuid()` |
| `TIMESTAMP` (sem TZ) | Ambiguidade + volta naive no Python | `TIMESTAMPTZ` |
| `MONEY` | Locale-dependent, impreciso | `BIGINT` (centavos) |
| `JSON` | Nao indexavel | `JSONB` |
| `ENUM` type | `ALTER TYPE` exige lock exclusivo | `TEXT` + `CHECK` |
| `FLOAT` / `DOUBLE` | Imprecisao | `BIGINT` (centavos) ou `NUMERIC` |

## 4. Primary Keys

```sql
"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

- Sempre `UUID` como primary key, gerado pelo **banco** (`gen_random_uuid()`, requer `pgcrypto`).
- Nunca gerar UUID no app — o banco e a fonte de verdade.
- `INSERT ... RETURNING "id"` + `fetchval` para obter o UUID gerado.

## 5. Foreign Keys

```sql
CREATE TABLE "orders" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId"    UUID NOT NULL,
    "status"    TEXT NOT NULL DEFAULT 'pending',
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "orders_userId_fk"
        FOREIGN KEY ("userId")
        REFERENCES "users" ("id")
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);

CREATE INDEX "idx_orders_userId" ON "orders" ("userId");
```

- Toda FK **nomeada**: `CONSTRAINT "<tabela>_<coluna>_fk"`.
- **Sempre** criar indice na coluna FK — PostgreSQL nao cria automaticamente.
- `ON DELETE`: `RESTRICT` (default, mais seguro), `CASCADE` (tenant -> recursos), `SET NULL` (relacoes opcionais).

## 6. Constraints

```sql
CONSTRAINT "users_emailUnique" UNIQUE ("email")
CONSTRAINT "users_tenantId_email_unique" UNIQUE ("tenantId", "email")
CONSTRAINT "orders_statusCheck" CHECK ("status" IN ('pending', 'processing', 'completed', 'cancelled'))
CONSTRAINT "orders_amountPositive" CHECK ("amountCents" > 0)
```

- Todas nomeadas explicitamente.
- Preferir `TEXT` + `CHECK` a `CREATE TYPE ... AS ENUM`.
- `NOT NULL` como padrao; `NULL` apenas quando a ausencia tem significado de negocio.

## 7. Indices

### Tipos

```sql
-- B-tree (padrao, 90% dos casos)
CREATE INDEX "idx_users_email" ON "users" ("email");

-- Ordenacao para paginacao
CREATE INDEX "idx_users_createdAt" ON "users" ("createdAt" DESC);

-- Composto (colunas mais seletivas primeiro)
CREATE INDEX "idx_orders_userId_status" ON "orders" ("userId", "status");

-- Partial
CREATE INDEX "idx_orders_pending" ON "orders" ("createdAt") WHERE "status" = 'pending';

-- GIN para JSONB
CREATE INDEX "idx_users_metadata" ON "users" USING gin ("metadata");

-- GIN para busca textual
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX "idx_users_name_trgm" ON "users" USING gin ("name" gin_trgm_ops);

-- BRIN para tabelas grandes ordenadas por insert (logs)
CREATE INDEX "idx_auditLogs_createdAt" ON "auditLogs" USING brin ("createdAt");
```

### Regras

- **Sempre** indexar colunas de FK.
- Indice composto `(a, b, c)` cobre `WHERE a` e `WHERE a AND b`, mas **nao** `WHERE b`. Colunas mais seletivas primeiro.
- **Nunca** criar indice em coluna de baixa cardinalidade isolada (ex: `BOOLEAN`) — usar partial.
- **Sempre** validar com `EXPLAIN ANALYZE`; nunca criar indices especulativos.
- Em tabelas grandes em producao, `CREATE INDEX CONCURRENTLY` (com `transaction:false` no dbmate).

## 8. Soft Delete

```sql
"deletedAt" TIMESTAMPTZ  -- NULL = ativo

-- Unique parcial: email unico apenas entre ativos
CREATE UNIQUE INDEX "idx_users_email_active" ON "users" ("email") WHERE "deletedAt" IS NULL;
```

- `"deletedAt" TIMESTAMPTZ` sem `NOT NULL` — `NULL` = ativo.
- Unique constraints devem ser **partial** com `WHERE "deletedAt" IS NULL`.
- Toda query de leitura inclui `WHERE "deletedAt" IS NULL`.
- Repositorio expoe `soft_delete` que faz `UPDATE ... SET "deletedAt" = now()`.

## 9. Tabelas de Juncao (N:N)

```sql
CREATE TABLE "userBusinessUnits" (
    "userId"         UUID NOT NULL,
    "businessUnitId" UUID NOT NULL,
    "assignedAt"     TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "userBusinessUnits_pk" PRIMARY KEY ("userId", "businessUnitId"),
    CONSTRAINT "userBusinessUnits_userId_fk" FOREIGN KEY ("userId") REFERENCES "users" ("id") ON DELETE CASCADE,
    CONSTRAINT "userBusinessUnits_businessUnitId_fk" FOREIGN KEY ("businessUnitId") REFERENCES "businessUnits" ("id") ON DELETE CASCADE
);

-- Indice reverso (PG so cria indice para o primeiro campo da PK)
CREATE INDEX "idx_userBusinessUnits_businessUnitId" ON "userBusinessUnits" ("businessUnitId");
```

## 10. Migrations (dbmate)

Migrations em **SQL puro** executadas com [dbmate](https://github.com/amacneil/dbmate). Cada migration e **um unico arquivo** com as secoes `-- migrate:up` e `-- migrate:down`. (dbmate e uma ferramenta standalone, independente da linguagem — identica em todos os harnesses.)

### Estrutura

```
migrations/
└── <app>/
    ├── 0001_setup.sql            # extensoes + funcoes globais
    ├── 0002_create_users.sql
    └── ...
```

### Formato

```sql
-- migrations/<app>/0002_create_users.sql

-- migrate:up
CREATE TABLE "users" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "name"      TEXT NOT NULL,
    "email"     TEXT NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "users_emailUnique" UNIQUE ("email")
);

-- migrate:down
DROP TABLE IF EXISTS "users";
```

### Comandos

```bash
DATABASE_URL="postgres://app:secret@localhost:5432/app?sslmode=disable" \
    dbmate --migrations-dir ./migrations/<app> up
dbmate --migrations-dir ./migrations/<app> rollback
dbmate --migrations-dir ./migrations/<app> status
```

### Regras

- Numeracao sequencial com 4 digitos: `0001`, `0002` (criar manualmente com o proximo numero).
- Todo arquivo tem **obrigatoriamente** `-- migrate:up` e `-- migrate:down`.
- `-- migrate:down` reverte **exatamente** o que o `-- migrate:up` fez.
- Cada migration faz **uma coisa**.
- **Nunca** alterar migration ja aplicada em producao — criar nova.
- Testar `dbmate up` e `dbmate rollback` localmente antes do commit.
- `CREATE INDEX CONCURRENTLY` nao roda em transaction — usar `-- migrate:up transaction:false`.

### Migrations Zero-Downtime

| Operacao | Segura? | Como Fazer |
|---|---|---|
| Criar tabela | Sim | `CREATE TABLE` normal |
| Adicionar coluna nullable | Sim | `ALTER TABLE ADD COLUMN ... NULL` |
| Adicionar coluna NOT NULL com default | Sim | `ADD COLUMN ... NOT NULL DEFAULT ...` (PG 11+ nao reescreve) |
| Criar indice | **Perigoso** | `CREATE INDEX CONCURRENTLY` |
| Renomear coluna | **Perigoso** | Nova coluna -> copiar -> dropar antiga, em migrations separadas |
| Adicionar constraint | **Perigoso** | `ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` separado |

## 11. Transactions

### Quando usar

| Cenario | Transaction? |
|---|---|
| INSERT/SELECT/UPDATE unico | **Nao** (autocommit) |
| INSERT + INSERT relacionados | **Sim** |
| UPDATE condicional (read-then-write) | **Sim** |
| Transferencia de saldo | **Sim** |
| Operacao de negocio com multiplas entidades | **Sim** |

### Helper `pkg/postgres/tx.py`

Com `asyncpg`, a transaction acontece em uma **connection dedicada** adquirida do pool, dentro de `async with conn.transaction()`. A connection sempre volta ao pool ao sair do `async with pool.acquire()`.

```python
# pkg/src/org_pkg/postgres/tx.py
from collections.abc import Awaitable, Callable

from asyncpg import Connection, Pool


async def run_in_tx[T](pool: Pool, fn: Callable[[Connection], Awaitable[T]]) -> T:
    """Adquire uma connection dedicada e executa fn dentro de uma transaction.

    COMMIT ao retornar; ROLLBACK se fn lanca. A connection volta para o pool.
    """
    async with pool.acquire() as conn:
        async with conn.transaction():
            return await fn(conn)
```

### Uso no repositorio (metodos `with_tx`)

```python
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
```

### Orquestracao no use case / controller com `Transactor`

```python
async def tx(conn: Connection) -> Order:
    created = await self._order_repository.create_with_tx(conn, Order.create(...))
    for item in body.items:
        await self._stock_repository.debit_with_tx(conn, item.product_id, item.quantity)
    return created

order = await self._transactor.run_in_tx(tx)
# evento SEMPRE apos o commit, fora do run_in_tx
```

### Niveis de isolamento

`asyncpg` aceita o nivel no `conn.transaction(isolation=...)`:

```python
async with conn.transaction(isolation="serializable"):
    ...
```

| Nivel | Quando usar |
|---|---|
| `read_committed` (padrao) | 90% dos casos |
| `repeatable_read` | Leitura consistente entre tabelas |
| `serializable` | Operacoes criticas de saldo (retry em `SerializationError`) |

### Regras de transactions

| Regra | Detalhe |
|---|---|
| **Transactions curtas** | Milissegundos; quanto mais longa, mais locks |
| **Nenhum I/O externo dentro do tx** | Nada de HTTP/NATS dentro — sempre **apos** o commit |
| **Evento DEPOIS do commit** | `repository.create` dentro do tx, `event.publish` fora |
| **Retry em serialization failure** | `asyncpg.SerializationError` deve ser retentado pelo app |
| **Sem tx para operacoes unicas** | INSERT/SELECT/UPDATE unicos nao precisam |
| **Connection volta ao pool** | Garantido pelo `async with pool.acquire()` |

### Anti-pattern: evento dentro da transaction

```python
# ERRADO — evento publicado antes do commit
async def tx(conn):
    await self._repository.create_with_tx(conn, entity)
    await self._event.publish_created(entity)  # ERRADO — dentro do tx

# CORRETO — evento apos o commit
await self._repository.create(entity)
with contextlib.suppress(Exception):
    await self._event.publish_created(entity)  # logar, nao propagar
```

## 12. Paginacao

### Por cursor (recomendada)

```sql
-- Primeira pagina
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "deletedAt" IS NULL
ORDER BY "id" ASC
LIMIT $1;

-- Proximas (cursor = ultimo id)
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "deletedAt" IS NULL AND "id" > $1
ORDER BY "id" ASC
LIMIT $2;
```

- Performance constante (usa indice); sem duplicados/pulos. Buscar `limit + 1` para descobrir o `next_cursor`.

### Por offset (apenas dashboards administrativos)

```sql
SELECT "id", "name" FROM "users" WHERE "deletedAt" IS NULL ORDER BY "createdAt" DESC LIMIT $1 OFFSET $2;
```

- Degrada com offsets grandes; usar so quando o usuario precisa pular para pagina especifica.

## 13. Queries — Boas Praticas (asyncpg)

### Metodos do asyncpg

| Metodo | Retorno | Uso |
|---|---|---|
| `await conn.fetch(sql, *args)` | `list[Record]` | Multiplas linhas |
| `await conn.fetchrow(sql, *args)` | `Record | None` | Uma linha (ou `None`) |
| `await conn.fetchval(sql, *args)` | valor escalar | `RETURNING "id"`, `COUNT(*)` |
| `await conn.execute(sql, *args)` | `str` (status) | INSERT/UPDATE/DELETE sem retorno |
| `await conn.executemany(sql, args_list)` | — | Bulk |

- **Placeholders posicionais `$1, $2`** — asyncpg **nao** aceita `%s`. Nunca interpolar valores (f-string em SQL e proibido).
- No projeto, o repositorio usa `self._database.pool.fetchrow(...)` para operacoes simples (o pool adquire/devolve a connection automaticamente).

### SELECT explicito (nunca `SELECT *`)

```python
query = """
    SELECT "id", "name", "email", "createdAt"
    FROM "users"
    WHERE "id" = $1
"""
row = await self._database.pool.fetchrow(query, id)
return None if row is None else self._to_entity(row)
```

### INSERT com RETURNING

```python
user_id = await self._database.pool.fetchval(
    'INSERT INTO "users" ("name", "email", "createdAt", "updatedAt") VALUES ($1, $2, $3, $4) RETURNING "id"',
    user.name, user.email, user.created_at, user.updated_at,
)
user.id = str(user_id)
```

### UPSERT (idempotencia — consumers)

```sql
INSERT INTO "accounts" ("userId", "name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT ("userId") DO NOTHING;
```

### Locking

```sql
SELECT "id", "balanceCents" FROM "accounts" WHERE "id" = $1 FOR UPDATE;

-- Filas de trabalho (concorrencia segura)
SELECT "id", "payload" FROM "jobs" WHERE "status" = 'pending'
ORDER BY "createdAt" ASC LIMIT 1 FOR UPDATE SKIP LOCKED;
```

### Nao-encontrado retorna `None`

Com `asyncpg`, "nao encontrado" **nao e erro** — `fetchrow` resolve `None`. O repositorio retorna `None`; nunca lance por ausencia.

## 14. Pool de Conexoes (`asyncpg.create_pool`)

O projeto encapsula o pool num wrapper `Database` (ver `architecture.md`) com ciclo de vida explicito (`connect`/`disconnect`), pois o pool asyncpg **deve ser criado no event loop** do processo.

```python
# pkg/src/org_pkg/postgres/conn.py
import asyncpg


async def create_pool(cfg: PostgresConfig) -> asyncpg.Pool:
    return await asyncpg.create_pool(
        host=cfg.db_host,
        port=cfg.db_port,
        user=cfg.db_user,
        password=cfg.db_password,
        database=cfg.db_name,
        min_size=1,
        max_size=cfg.db_pool_max,        # 10-25 (2-5x nucleos do DB por instancia)
        max_inactive_connection_lifetime=300.0,  # 5 min — recicla conexoes ociosas
        command_timeout=10,              # timeout por comando
    )
```

| Parametro | Recomendacao | Justificativa |
|---|---|---|
| `max_size` | 10-25 | `max_connections` do PG / instancias do app |
| `min_size` | 1 | Nao segurar conexoes a toa |
| `max_inactive_connection_lifetime` | 300 | Liberar conns ociosas |
| `command_timeout` | 5-10 | Falha rapida em query travada |
| `pool.close()` | no graceful shutdown | Aguarda queries em curso e fecha |

- **Criar o pool no startup handler**, no event loop do servidor — nunca no import/constructor.
- `await pool.close()` no shutdown (via `Database.disconnect()`).

## 15. Configuracao do Servidor (PostgreSQL 18)

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]
    command:
      - "postgres"
      - "-c"
      - "shared_buffers=256MB"
      - "-c"
      - "effective_cache_size=512MB"
      - "-c"
      - "work_mem=16MB"
      - "-c"
      - "random_page_cost=1.1"            # SSD
      - "-c"
      - "log_min_duration_statement=100"  # loga queries > 100ms
      - "-c"
      - "max_connections=100"
```

## Checklist — Nova Tabela

- [ ] Nome em camelCase plural com aspas duplas (`"orderItems"`)
- [ ] `"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [ ] Colunas em camelCase com aspas duplas
- [ ] `TIMESTAMPTZ` para datas (nunca `TIMESTAMP`)
- [ ] `"createdAt"`/`"updatedAt"` com `DEFAULT now()`
- [ ] Trigger `"trg_<tabela>_updatedAt"` registrado
- [ ] Constraints nomeadas
- [ ] FKs com indice na coluna FK e `ON DELETE` explicito
- [ ] `CHECK` para enumeracoes (em vez de `ENUM` type)
- [ ] Indices para colunas em `WHERE`/`ORDER BY`
- [ ] Repositorio: `fetchval` + `RETURNING "id"`; `_to_entity` converte UUID->str
- [ ] `find_by*` retorna `None` quando nao encontrado
- [ ] Migration com `-- migrate:up`/`-- migrate:down` criada e testada

## Checklist — Nova Transaction

- [ ] Multiplas escritas ou read-then-write? Se sim, `run_in_tx`
- [ ] Connection adquirida via `pool.acquire()` dentro do helper
- [ ] Nenhuma chamada externa (HTTP, NATS) dentro do tx
- [ ] Evento publicado **depois** do commit
- [ ] Nivel de isolamento adequado (`read_committed` na maioria)
- [ ] Retry para `SerializationError` se usando `serializable`
- [ ] Repositorios participantes expoem `<metodo>_with_tx(conn, ...)`

## Checklist — Nova Migration

- [ ] Numeracao sequencial com 4 digitos
- [ ] Secoes `-- migrate:up` e `-- migrate:down` presentes
- [ ] `-- migrate:down` reverte exatamente o up
- [ ] Uma unica responsabilidade
- [ ] `CREATE INDEX CONCURRENTLY` com `transaction:false` para tabelas grandes
- [ ] Testada localmente (`dbmate up` e `dbmate rollback`)

## Fontes

- https://www.postgresql.org/docs/18/ — documentacao oficial PG 18
- https://www.postgresql.org/docs/18/datatype.html — tipos
- https://www.postgresql.org/docs/18/indexes.html — indices
- https://www.postgresql.org/docs/18/transaction-iso.html — isolamento
- https://magicstack.github.io/asyncpg/current/ — documentacao do asyncpg (Pool, Connection, type codecs)
- https://github.com/amacneil/dbmate — ferramenta de migrations
