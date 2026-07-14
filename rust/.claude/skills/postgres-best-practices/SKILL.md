---
name: postgres-best-practices
description: Aplica boas praticas oficiais do PostgreSQL 18 ao modelar, escrever, revisar ou refatorar schema, queries, migrations e codigo de acesso a dados em Rust com sqlx. Cobre convencoes de nomenclatura em camelCase com aspas duplas, tipos recomendados (UUID, TIMESTAMPTZ, JSONB, BIGINT em centavos), primary keys, foreign keys, constraints nomeadas, indices (B-tree, GIN, BRIN, partial, compostos), soft delete, tabelas de juncao N:N, migrations zero-downtime (sqlx-cli, arquivos .up.sql / .down.sql), transactions (pool.begin/commit com rollback automatico no drop, metodos *_with_tx, niveis de isolamento, eventos pos-commit), paginacao por cursor, queries idiomaticas (SELECT explicito, INSERT RETURNING, UPSERT, SELECT FOR UPDATE/SKIP LOCKED), row structs com sqlx::FromRow, pool de conexoes com PgPoolOptions e configuracao de servidor. Use sempre que criar/editar migrations SQL, modelar tabelas, escrever queries, implementar repositorios Rust que falam com Postgres via sqlx, definir indices, orquestrar transactions ou ajustar configuracao do banco. Acione ao trabalhar com arquivos .sql, repositorios que usam sqlx (PgPool/Transaction), migrations em migrations/<app>/*, ou ao discutir performance/locks/isolation no PostgreSQL.
---

# PostgreSQL 18 Best Practices (Rust · sqlx)

Skill baseada nas praticas oficiais do PostgreSQL 18, do crate `sqlx` 0.8 e nos padroes internos do projeto para modelagem, queries, migrations, transactions e configuracao. Todas as recomendacoes aqui devem ser seguidas ao criar, revisar ou refatorar qualquer artefato que envolva o banco.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.sql` (DDL ou DML)
- Ao escrever ou revisar migrations em `migrations/<app>/*`
- Ao implementar repositorios Rust que falam com PostgreSQL (`sqlx`)
- Ao modelar novas tabelas, indices, constraints ou tipos
- Ao orquestrar transactions (multi-repo, multi-write)
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

### camelCase no banco, snake_case no Rust

O banco usa camelCase (`"createdAt"`); a struct Rust usa snake_case (`created_at`). Como o `sqlx::FromRow` mapeia pelo **nome exato da coluna**, use `#[sqlx(rename = "...")]` na **row struct** da camada de infraestrutura (nunca na entidade de dominio, que nao deriva `FromRow`):

```rust
use chrono::{DateTime, Utc};
use uuid::Uuid;

use crate::domain::entity::User;

#[derive(sqlx::FromRow)]
struct UserRow {
    id: Uuid,
    name: String,
    email: String,
    #[sqlx(rename = "createdAt")]
    created_at: DateTime<Utc>,
    #[sqlx(rename = "updatedAt")]
    updated_at: DateTime<Utc>,
}

impl From<UserRow> for User {
    fn from(row: UserRow) -> Self {
        Self {
            id: row.id,
            name: row.name,
            email: row.email,
            created_at: row.created_at,
            updated_at: row.updated_at,
        }
    }
}
```

- `UUID` volta como `uuid::Uuid` (feature `uuid` do sqlx) — a entidade tambem usa `Uuid`, sem conversao.
- `TIMESTAMPTZ` volta como `chrono::DateTime<Utc>` (feature `chrono`) — sempre aware, nunca naive.
- `BIGINT` volta como `i64` do Rust — dinheiro em centavos sem perda de precisao.
- **Row struct sempre privada** na infraestrutura; converta para a entidade via `From<Row> for Entity`.

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
-- migrations/<app>/000001_setup.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE OR REPLACE FUNCTION "setUpdatedAt"()
RETURNS TRIGGER AS $$
BEGIN
    NEW."updatedAt" = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```sql
-- migrations/<app>/000001_setup.down.sql
DROP FUNCTION IF EXISTS "setUpdatedAt"();
```

Toda tabela com `"updatedAt"` deve registrar o trigger `"trg_<tabela>_updatedAt"`.

## 3. Tipos de Dados

### Recomendados

| Caso de Uso | Tipo | sqlx -> Rust |
|---|---|---|
| Identificadores (PK/FK) | `UUID` | `uuid::Uuid` |
| Texto | `TEXT` | `String` |
| Inteiros | `INT` / `BIGINT` | `i32` / `i64` |
| Dinheiro | `BIGINT` (centavos) | `i64` — nunca `f64`/`NUMERIC` para dinheiro em API |
| Booleanos | `BOOLEAN` | `bool` |
| Data/hora | `TIMESTAMPTZ` | `chrono::DateTime<Utc>` (aware) |
| Apenas data | `DATE` | `chrono::NaiveDate` |
| JSON estruturado | `JSONB` | `sqlx::types::Json<T>` ou `serde_json::Value` |
| Enumeracoes | `TEXT` + `CHECK` | `String` (valide com enum serde no boundary) |
| Arrays | `TEXT[]` / `UUID[]` | `Vec<String>` / `Vec<Uuid>` |
| Coluna anulavel | qualquer + `NULL` | `Option<T>` |

**JSONB no sqlx:** mapeie para `sqlx::types::Json<MyStruct>` (serializa/deserializa via serde automaticamente) ou `serde_json::Value` quando o schema for dinamico. Na row struct:

```rust
use sqlx::types::Json;

#[derive(sqlx::FromRow)]
struct OrderRow {
    id: Uuid,
    metadata: Option<Json<OrderMetadata>>,
}
```

### Tipos a evitar

| Tipo | Problema | Usar em vez |
|---|---|---|
| `SERIAL` / `BIGSERIAL` | IDs previsiveis, problemas em sharding | `UUID DEFAULT gen_random_uuid()` |
| `TIMESTAMP` (sem TZ) | Ambiguidade + volta naive no Rust | `TIMESTAMPTZ` |
| `MONEY` | Locale-dependent, impreciso | `BIGINT` (centavos) |
| `JSON` | Nao indexavel | `JSONB` |
| `ENUM` type | `ALTER TYPE` exige lock exclusivo | `TEXT` + `CHECK` |
| `FLOAT` / `DOUBLE` | Imprecisao | `BIGINT` (centavos) ou `NUMERIC` |

## 4. Primary Keys

```sql
"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

- Sempre `UUID` como primary key, gerado pelo **banco** (`gen_random_uuid()`, requer `pgcrypto`).
- Nunca gerar UUID no app — o banco e a fonte de verdade. A entidade nasce com `Uuid::nil()` e recebe o ID no `RETURNING`.
- `INSERT ... RETURNING "id"` + `fetch_one` (com destino `(Uuid,)`) para obter o UUID gerado.

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
- `NOT NULL` como padrao; `NULL` (`Option<T>` no Rust) apenas quando a ausencia tem significado de negocio.

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
CREATE INDEX "idx_orders_metadata" ON "orders" USING gin ("metadata");

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
- Em tabelas grandes em producao, `CREATE INDEX CONCURRENTLY` (fora de transaction — ver migrations).

## 8. Soft Delete

```sql
"deletedAt" TIMESTAMPTZ  -- NULL = ativo

-- Unique parcial: email unico apenas entre ativos
CREATE UNIQUE INDEX "idx_users_email_active" ON "users" ("email") WHERE "deletedAt" IS NULL;

-- Index parcial para queries que filtram ativos
CREATE INDEX "idx_users_active" ON "users" ("createdAt" DESC) WHERE "deletedAt" IS NULL;
```

- `"deletedAt" TIMESTAMPTZ` sem `NOT NULL` — `NULL` = ativo (`Option<DateTime<Utc>>` no Rust).
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

- Nome: `<entidadeA><entidadeB>` em camelCase plural.
- PK composta com as duas FKs; `ON DELETE CASCADE` em ambas.
- Sempre criar **indice no segundo campo** da PK.

## 10. Migrations (sqlx-cli)

Migrations em **SQL puro**, com pares reversiveis `.up.sql` + `.down.sql`, executadas com [sqlx-cli](https://crates.io/crates/sqlx-cli).

### Estrutura

```
migrations/
└── <app>/
    ├── 000001_setup.up.sql            # extensoes + funcoes globais
    ├── 000001_setup.down.sql
    ├── 000002_create_users.up.sql
    ├── 000002_create_users.down.sql
    └── ...
```

### Criar / rodar

```bash
# cria o par <timestamp>_create_orders.up.sql e .down.sql (-r = reversivel)
sqlx migrate add -r create_orders --source migrations/billing

export DATABASE_URL="postgres://app:secret@localhost:5432/app"
sqlx migrate run      --source migrations/billing   # aplica .up.sql pendentes
sqlx migrate revert   --source migrations/billing   # reverte o ultimo (.down.sql)
sqlx migrate info     --source migrations/billing   # status
```

### Regras

- Todo arquivo `.up.sql` tem um par `.down.sql` que reverte **exatamente** o que o up fez.
- Cada migration faz **uma coisa** (criar tabela, adicionar coluna, criar indice).
- **Nunca** alterar migration ja aplicada em producao — criar nova.
- Testar `sqlx migrate run` e `sqlx migrate revert` localmente antes do commit.
- `CREATE INDEX CONCURRENTLY` **nao** roda dentro de transaction — sqlx envolve cada migration numa transaction por padrao; use uma migration marcada como no-transaction (`-- no-transaction` no topo do arquivo) para indices concorrentes.

### Migrations Zero-Downtime

| Operacao | Segura? | Como Fazer |
|---|---|---|
| Criar tabela | Sim | `CREATE TABLE` normal |
| Adicionar coluna nullable | Sim | `ALTER TABLE ADD COLUMN ... NULL` |
| Adicionar coluna NOT NULL com default | Sim | `ADD COLUMN ... NOT NULL DEFAULT ...` (PG 11+ nao reescreve) |
| Remover coluna | Sim | `DROP COLUMN` (apos confirmar que nenhum codigo referencia) |
| Criar indice | **Perigoso** | `CREATE INDEX CONCURRENTLY` (migration `-- no-transaction`) |
| Renomear coluna | **Perigoso** | Nova coluna -> copiar -> dropar antiga, em migrations separadas |
| Alterar tipo de coluna | **Perigoso** | Criar nova, migrar dados, renomear |
| Adicionar constraint | **Perigoso** | `ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` separado |

## 11. Transactions

Nao ha interface `Transactor` no dominio: transactions sao um detalhe de infraestrutura, orquestradas onde controller e repositorio se encontram (ambos infraestrutura). O padrao canonico com sqlx usa `pool.begin()` + `tx.commit()`, com **rollback automatico no drop** se o commit nao acontecer.

### Quando usar

| Cenario | Transaction? | Por que |
|---|---|---|
| INSERT/SELECT/UPDATE unico | **Nao** | Autocommit ja garante atomicidade |
| INSERT + INSERT relacionados | **Sim** | Tudo ou nada |
| UPDATE condicional (read-then-write) | **Sim** | Evitar race condition |
| Transferencia de saldo | **Sim** | Debito e credito atomicos |
| DELETE em cascata manual | **Sim** | Consistencia ao remover relacionados |
| Operacao com multiplas entidades | **Sim** | Order + estoque + log |

### Padrao canonico

```rust
let mut tx = pool.begin().await.context("begin tx")?;   // sqlx::Transaction<'_, Postgres>
// ... operacoes com &mut *tx via metodos *_with_tx
tx.commit().await.context("commit tx")?;
// rollback e AUTOMATICO no drop se o commit nao acontecer (early return, `?`, panic)
```

A garantia central: **nenhum caminho de erro deixa transaction aberta** — o drop da `Transaction` sem commit executa rollback.

### Repositorio com metodos `*_with_tx`

Quando um repositorio participa de uma transaction externa, expoe **metodos inerentes na implementacao concreta** (fora do trait de dominio, que nao pode conhecer `sqlx::Transaction`). Controllers que orquestram tx recebem o repositorio **concreto** (`Arc<PostgresOrderRepository>`) — permitido pois controller e repositorio sao ambos infraestrutura:

```rust
use anyhow::{Context, Result};
use sqlx::{Postgres, Transaction};
use uuid::Uuid;

use crate::domain::entity::Order;

impl PostgresOrderRepository {
    pub async fn create_with_tx(
        &self,
        tx: &mut Transaction<'_, Postgres>,
        order: &Order,
    ) -> Result<Order> {
        const QUERY: &str = r#"
            INSERT INTO "orders" ("userId", "status", "totalCents", "createdAt", "updatedAt")
            VALUES ($1, $2, $3, $4, $5)
            RETURNING "id""#;

        let (id,): (Uuid,) = sqlx::query_as(QUERY)
            .bind(order.user_id)
            .bind(&order.status)
            .bind(order.total_cents)
            .bind(order.created_at)
            .bind(order.updated_at)
            .fetch_one(&mut **tx)   // &mut **tx: executor da transaction
            .await
            .context("insert order")?;

        let mut created = order.clone();
        created.id = id;
        Ok(created)
    }
}
```

### Orquestracao (controller/infra)

```rust
let mut tx = pool.begin().await.context("begin tx")?;

let order = order_repo.create_with_tx(&mut tx, &Order::new(user_id, total_cents)).await?;
for item in &items {
    stock_repo.debit_with_tx(&mut tx, item.product_id, item.quantity).await?;
}

tx.commit().await.context("commit tx")?;

// evento SEMPRE apos o commit, fora da transaction
let _ = order_event.publish_created(&order).await;
```

### Niveis de isolamento

O sqlx nao expoe um enum de isolamento; execute o `SET TRANSACTION` como primeira instrucao da transaction:

```rust
let mut tx = pool.begin().await?;
sqlx::query("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
    .execute(&mut *tx)
    .await?;
// ... operacoes
tx.commit().await?;
```

| Nivel | Quando usar |
|---|---|
| `READ COMMITTED` (padrao) | 90% dos casos |
| `REPEATABLE READ` | Leitura consistente entre tabelas |
| `SERIALIZABLE` | Operacoes criticas de saldo (retry em `40001`) |
| `READ ONLY` | Relatorios (`SET TRANSACTION READ ONLY`) |

### Regras de transactions

| Regra | Detalhe |
|---|---|
| **Transactions curtas** | Milissegundos; quanto mais longa, mais locks retidos |
| **Nenhum I/O externo dentro do tx** | Nada de HTTP/NATS dentro — sempre **apos** o commit |
| **Evento DEPOIS do commit** | `repo.create_with_tx` dentro do tx, `event.publish` fora |
| **Retry em serialization failure** | `SQLSTATE 40001` deve ser retentado pelo app |
| **Sem tx para operacoes unicas** | INSERT/SELECT/UPDATE unicos usam `&pool` direto |
| **Rollback automatico** | Nao chamar rollback manual — o drop sem commit ja reverte |
| **Commit explicito** | `tx.commit().await?` no fim do caminho feliz |

### Anti-pattern: evento dentro da transaction

```rust
// ERRADO — evento publicado antes do commit
let mut tx = pool.begin().await?;
repo.create_with_tx(&mut tx, &entity).await?;
event.publish_created(&entity).await?;   // ERRADO — dentro do tx
tx.commit().await?;

// CORRETO — evento apos o commit
repo.create(&mut entity).await?;
let _ = event.publish_created(&entity).await;   // logar, nao propagar
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

## 13. Queries — Boas Praticas (sqlx)

### Metodos do sqlx

| Metodo | Retorno | Uso |
|---|---|---|
| `sqlx::query_as(sql).bind(..).fetch_all(ex)` | `Vec<T>` | Multiplas linhas -> row struct |
| `sqlx::query_as(sql).bind(..).fetch_optional(ex)` | `Option<T>` | Uma linha (ou `None`) |
| `sqlx::query_as(sql).bind(..).fetch_one(ex)` | `T` | Exatamente uma (`RETURNING`, `COUNT(*)`) |
| `sqlx::query(sql).bind(..).execute(ex)` | `PgQueryResult` | INSERT/UPDATE/DELETE sem retorno (`rows_affected()`) |

- **Placeholders posicionais `$1, $2`** — nunca interpolar valores (formatar SQL com dados e proibido; use `.bind(...)`).
- `ex` = executor: `&self.pool` (autocommit) ou `&mut *tx` (dentro de transaction).
- `sqlx::query_as::<_, (Uuid,)>` para escalares tipados como `RETURNING "id"`.

### SELECT explicito (nunca `SELECT *`)

```rust
async fn find_by_id(&self, id: Uuid) -> Result<Option<User>> {
    const QUERY: &str = r#"
        SELECT "id", "name", "email", "createdAt", "updatedAt"
        FROM "users"
        WHERE "id" = $1"#;

    let row: Option<UserRow> = sqlx::query_as(QUERY)
        .bind(id)
        .fetch_optional(&self.pool)
        .await
        .context("find user by id")?;

    Ok(row.map(Into::into))
}
```

### INSERT com RETURNING

```rust
async fn create(&self, user: &mut User) -> Result<()> {
    const QUERY: &str = r#"
        INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
        VALUES ($1, $2, $3, $4)
        RETURNING "id""#;

    let (id,): (Uuid,) = sqlx::query_as(QUERY)
        .bind(&user.name)
        .bind(&user.email)
        .bind(user.created_at)
        .bind(user.updated_at)
        .fetch_one(&self.pool)
        .await
        .context("insert user")?;

    user.id = id;
    Ok(())
}
```

### UPSERT (idempotencia — consumers)

```sql
-- Ignorar duplicata
INSERT INTO "accounts" ("userId", "name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT ("userId") DO NOTHING;

-- Inserir ou atualizar
INSERT INTO "accounts" ("userId", "name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT ("userId") DO UPDATE SET
    "name"      = EXCLUDED."name",
    "email"     = EXCLUDED."email",
    "updatedAt" = EXCLUDED."updatedAt";
```

### Locking

```sql
SELECT "id", "balanceCents" FROM "accounts" WHERE "id" = $1 FOR UPDATE;

-- Filas de trabalho (concorrencia segura)
SELECT "id", "payload" FROM "jobs" WHERE "status" = 'pending'
ORDER BY "createdAt" ASC LIMIT 1 FOR UPDATE SKIP LOCKED;
```

### Nao-encontrado retorna `Ok(None)`

Com sqlx, "nao encontrado" **nao e erro** — use `fetch_optional`, que resolve `Ok(None)`. O repositorio retorna `Option`; nunca lance por ausencia. (`fetch_one` retorna `Err(RowNotFound)` — use apenas quando a linha deve existir, como no `RETURNING`.)

## 14. Pool de Conexoes (`PgPoolOptions`)

```rust
// pkg/src/postgres/conn.rs
use std::time::Duration;

use sqlx::postgres::{PgPool, PgPoolOptions};

pub async fn new_pool(cfg: &Config) -> anyhow::Result<PgPool> {
    let url = format!(
        "postgres://{}:{}@{}:{}/{}",
        cfg.user, cfg.password, cfg.host, cfg.port, cfg.db_name
    );

    let pool = PgPoolOptions::new()
        .max_connections(cfg.max_connections)   // 10-25 (2-5x nucleos do DB por instancia)
        .acquire_timeout(Duration::from_secs(5)) // falha rapida quando o pool esgota
        .max_lifetime(Duration::from_secs(30 * 60))  // rotaciona conexoes (DNS, PgBouncer)
        .idle_timeout(Duration::from_secs(5 * 60))    // libera ociosas
        .connect(&url)
        .await?;

    Ok(pool)
}
```

| Parametro | Recomendacao | Justificativa |
|---|---|---|
| `max_connections` | 10-25 | `max_connections` do PG / instancias do app |
| `acquire_timeout` | 5s | Falha rapida em pool esgotado |
| `max_lifetime` | 30 min | Rotacionar para respeitar DNS/PgBouncer |
| `idle_timeout` | 5 min | Liberar conns em baixo trafego |
| `pool.close().await` | no graceful shutdown | Aguarda queries em curso e fecha |

- `PgPool` e clonavel e barato de clonar (`Arc` interno) — clones compartilham o mesmo pool.
- Criar o pool no composition root (`new_pool`), nunca em `static`/`OnceLock`.
- `pool.close().await` no shutdown, na ordem inversa da criacao.

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
      - "shared_buffers=256MB"            # ~25% da RAM
      - "-c"
      - "effective_cache_size=512MB"      # 50-75% da RAM
      - "-c"
      - "work_mem=16MB"                   # por operacao de sort/hash
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
- [ ] Row struct com `#[sqlx(rename = "...")]` p/ colunas camelCase + `From<Row> for Entity`
- [ ] Repositorio: `fetch_one` + `RETURNING "id"` para o ID gerado
- [ ] `find_by*` usa `fetch_optional` -> `Ok(None)` quando nao encontrado
- [ ] Migration `.up.sql` + `.down.sql` criada e testada

## Checklist — Nova Transaction

- [ ] Multiplas escritas ou read-then-write? Se sim, `pool.begin()`
- [ ] Operacoes via metodos `*_with_tx(&mut tx, ...)` com executor `&mut *tx`
- [ ] Nenhuma chamada externa (HTTP, NATS) dentro do tx
- [ ] Evento publicado **depois** do `tx.commit()`
- [ ] Nivel de isolamento adequado (`READ COMMITTED` na maioria)
- [ ] Retry para `40001` se usando `SERIALIZABLE`
- [ ] Rollback automatico confiado ao drop (sem rollback manual)

## Checklist — Nova Migration

- [ ] Par `.up.sql` + `.down.sql` criados (`sqlx migrate add -r`)
- [ ] `.down.sql` reverte exatamente o `.up.sql`
- [ ] Uma unica responsabilidade
- [ ] `CREATE INDEX CONCURRENTLY` em migration `-- no-transaction`
- [ ] Constraints via `NOT VALID` + `VALIDATE` separados
- [ ] Testada localmente (`sqlx migrate run` e `sqlx migrate revert`)
- [ ] Nao altera migration ja aplicada em producao

## Exemplo Completo — Migration de Tabela

```sql
-- migrations/billing/000002_create_orders.up.sql

CREATE TABLE "orders" (
    "id"          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId"      UUID        NOT NULL,
    "status"      TEXT        NOT NULL DEFAULT 'pending',
    "totalCents"  BIGINT      NOT NULL,
    "currency"    VARCHAR(3)  NOT NULL DEFAULT 'BRL',
    "notes"       TEXT,
    "metadata"    JSONB,
    "createdAt"   TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt"   TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "orders_userId_fk"
        FOREIGN KEY ("userId") REFERENCES "users" ("id")
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT "orders_statusCheck"
        CHECK ("status" IN ('pending', 'processing', 'completed', 'cancelled', 'refunded')),
    CONSTRAINT "orders_totalPositive" CHECK ("totalCents" > 0),
    CONSTRAINT "orders_currencyCheck" CHECK ("currency" IN ('BRL', 'USD', 'EUR'))
);

CREATE TRIGGER "trg_orders_updatedAt"
    BEFORE UPDATE ON "orders"
    FOR EACH ROW
    EXECUTE FUNCTION "setUpdatedAt"();

CREATE INDEX "idx_orders_userId"        ON "orders" ("userId");
CREATE INDEX "idx_orders_status"        ON "orders" ("status") WHERE "status" IN ('pending', 'processing');
CREATE INDEX "idx_orders_createdAt"     ON "orders" ("createdAt" DESC);
CREATE INDEX "idx_orders_userId_status" ON "orders" ("userId", "status");
```

```sql
-- migrations/billing/000002_create_orders.down.sql

DROP TRIGGER IF EXISTS "trg_orders_updatedAt" ON "orders";
DROP TABLE   IF EXISTS "orders";
```

## Fontes

- https://www.postgresql.org/docs/18/ — documentacao oficial PG 18
- https://www.postgresql.org/docs/18/datatype.html — tipos de dados
- https://www.postgresql.org/docs/18/indexes.html — indices (B-tree, GIN, BRIN, partial)
- https://www.postgresql.org/docs/18/transaction-iso.html — niveis de isolamento
- https://www.postgresql.org/docs/18/explicit-locking.html — locks e `FOR UPDATE`/`SKIP LOCKED`
- https://docs.rs/sqlx/latest/sqlx/ — documentacao do sqlx (PgPool, query_as, FromRow, Transaction)
- https://crates.io/crates/sqlx-cli — ferramenta de migrations
