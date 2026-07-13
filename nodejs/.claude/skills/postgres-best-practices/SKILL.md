---
name: postgres-best-practices
description: Aplica boas práticas oficiais do PostgreSQL 18 ao modelar, escrever, revisar ou refatorar schema, queries, migrations e código de acesso a dados. Cobre convenções de nomenclatura em camelCase com aspas duplas, tipos recomendados (UUID, TIMESTAMPTZ, JSONB, BIGINT em centavos), primary keys, foreign keys, constraints nomeadas, índices (B-tree, GIN, BRIN, partial, compostos), soft delete, tabelas de junção N:N, migrations zero-downtime (dbmate, SQL puro com -- migrate:up / -- migrate:down), transactions (runInTx, níveis de isolamento, eventos pós-commit), paginação por cursor, queries idiomáticas (SELECT explícito, INSERT RETURNING, UPSERT, SELECT FOR UPDATE/SKIP LOCKED), pool de conexões com pg.Pool (node-postgres) e configuração de servidor (shared_buffers, work_mem, WAL, logging). Use sempre que criar/editar migrations SQL, modelar tabelas, escrever queries, implementar repositórios TypeScript que falam com Postgres, definir índices, orquestrar transactions em use cases ou ajustar configuração do banco. Acione ao trabalhar com arquivos .sql, repositórios que usam pg (node-postgres, Pool/PoolClient), migrations em migrations/<app>/*, ou ao discutir performance/locks/isolation no PostgreSQL.
---

# PostgreSQL 18 Best Practices

Skill baseada nas práticas oficiais do PostgreSQL 18 e nos padrões internos do projeto para modelagem, queries, migrations, transactions e configuração. Todas as recomendações aqui devem ser seguidas ao criar, revisar ou refatorar qualquer artefato que envolva o banco.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.sql` (DDL ou DML)
- Ao escrever ou revisar migrations em `migrations/<app>/*`
- Ao implementar repositórios TypeScript que falam com PostgreSQL (`pg` / node-postgres)
- Ao modelar novas tabelas, índices, constraints ou tipos
- Ao orquestrar transactions em use cases (multi-repo, multi-write)
- Ao definir paginação, locking, isolamento ou estratégia de soft delete
- Ao ajustar configuração do servidor (`postgresql.conf`, `docker-compose.yml`)
- Ao discutir performance, locks, race conditions ou planos de execução (`EXPLAIN ANALYZE`)

## 1. Convenções de Nomenclatura

| Elemento     | Convenção                       | Exemplo                                      |
|--------------|---------------------------------|----------------------------------------------|
| Tabela       | camelCase, **plural**           | `users`, `orderItems`, `auditLogs`           |
| Coluna       | camelCase                       | `createdAt`, `userId`, `firstName`           |
| Primary Key  | `id`                            | `id UUID PRIMARY KEY`                        |
| Foreign Key  | `<entidade>Id`                  | `userId`, `orderId`                          |
| Constraint   | `<tabela>_<descricao>`          | `users_emailUnique`, `orders_statusCheck`    |
| Index        | `idx_<tabela>_<colunas>`        | `idx_users_email`, `idx_orders_userId_createdAt` |
| Trigger      | `trg_<tabela>_<acao>`           | `trg_users_updatedAt`                        |
| Function     | camelCase descritivo            | `setUpdatedAt()`, `validateStatus()`         |
| Enum Type    | camelCase singular              | `orderStatus`, `userRole`                    |
| Sequence     | `<tabela>_<coluna>_seq`         | `users_id_seq`                               |

### Regras inegociáveis

- PostgreSQL preserva case **apenas** quando o nome está entre aspas duplas. Como usamos camelCase, **todas** as referências a tabelas e colunas devem usar aspas duplas no SQL.
- Tabelas sempre no **plural** — representam coleções de entidades.
- Foreign key sempre nomeada como `<entidadeSingular>Id`.
- Constraints sempre nomeadas **explicitamente** — nunca deixar o PostgreSQL gerar nomes automáticos.

### Aspas duplas — padrão obrigatório

```sql
-- DDL
CREATE TABLE "orderItems" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "orderId"   UUID NOT NULL,
    "productId" UUID NOT NULL,
    "quantity"  INT  NOT NULL
);

-- DML
SELECT "id", "orderId", "quantity"
FROM "orderItems"
WHERE "orderId" = $1;
```

```typescript
// Repositório TypeScript (pg)
const query = `
    INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
    VALUES ($1, $2, $3, $4)
    RETURNING "id"`;

const result = await pool.query<{ id: string }>(query, [name, email, createdAt, updatedAt]);
```

Bônus: com colunas em camelCase entre aspas, as chaves de `result.rows` já saem em camelCase — sem mapeamento snake_case → camelCase no repositório.

## 2. Padrão de Criação de Tabelas

### Template base

```sql
CREATE TABLE "users" (
    -- identidade
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- dados de domínio
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

### Função global `setUpdatedAt`

Criar **uma única vez** na migration inicial:

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

Toda tabela com `"updatedAt"` deve registrar o trigger `"trg_<tabela>_updatedAt"`.

## 3. Tipos de Dados

### Recomendados

| Caso de Uso              | Tipo Recomendado            | Justificativa                                                                 |
|--------------------------|-----------------------------|------------------------------------------------------------------------------|
| Identificadores (PK/FK)  | `UUID`                      | Sem sequência previsível, facilita sharding, `gen_random_uuid()` no banco    |
| Texto curto              | `TEXT`                      | Sem limite artificial; validação no app                                       |
| Texto com limite rígido  | `VARCHAR(N)`                | Quando o limite faz parte da regra de negócio (ex: `VARCHAR(2)` para país)    |
| Inteiros                 | `INT` / `BIGINT`            | `INT` para contadores, `BIGINT` para IDs sequenciais ou valores grandes       |
| Dinheiro                 | `BIGINT` (centavos)         | Armazenar em centavos. Nunca `NUMERIC`/`DECIMAL` para dinheiro em APIs        |
| Booleanos                | `BOOLEAN`                   | Sempre `BOOLEAN`, nunca `INT` 0/1                                             |
| Data/hora                | `TIMESTAMPTZ`               | **Sempre** com timezone                                                       |
| Apenas data              | `DATE`                      | Para datas sem hora (ex: nascimento)                                          |
| JSON estruturado         | `JSONB`                     | Indexável; nunca `JSON`                                                       |
| Enumerações              | `TEXT` + `CHECK`            | Adicionar valor sem `ALTER TYPE` (lock exclusivo)                             |
| Arrays                   | `TEXT[]` / `UUID[]`         | Para listas pequenas; relações N:N usam tabela de junção                      |
| IP addresses             | `INET`                      | Tipo nativo                                                                   |
| Intervalos de tempo      | `INTERVAL`                  | Para durações                                                                 |

**Atenção no client `pg`:** `BIGINT` (`int8`) chega como **string** no JavaScript (para não perder precisão além de `Number.MAX_SAFE_INTEGER`). Converta explicitamente no repositório (`Number(row.totalCents)` quando o domínio garante valores seguros, ou `BigInt(row.totalCents)` para valores potencialmente grandes) — nunca deixe strings vazarem para a entidade.

### Tipos a evitar

| Tipo                  | Problema                                              | Usar em vez                          |
|-----------------------|-------------------------------------------------------|--------------------------------------|
| `SERIAL` / `BIGSERIAL`| IDs previsíveis, problemas em sharding                 | `UUID DEFAULT gen_random_uuid()`     |
| `TIMESTAMP` (sem TZ)  | Ambiguidade entre regiões                              | `TIMESTAMPTZ`                        |
| `CHAR(N)`             | Padding com espaços                                    | `TEXT` ou `VARCHAR(N)`               |
| `MONEY`               | Locale-dependent, impreciso                            | `BIGINT` (centavos)                  |
| `JSON`                | Não indexável, reprocessa a cada leitura               | `JSONB`                              |
| `ENUM` type           | `ALTER TYPE` exige lock exclusivo                      | `TEXT` + `CHECK`                     |
| `FLOAT` / `DOUBLE`    | Imprecisão de ponto flutuante                          | `BIGINT` (centavos) ou `NUMERIC`     |

## 4. Primary Keys

```sql
"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

- Sempre `UUID` como primary key.
- `gen_random_uuid()` gera UUID v4 no banco (requer `pgcrypto`; nativo no PG 13+).
- Nunca usar `SERIAL` — evita previsibilidade e facilita replicação/sharding.
- Nunca gerar UUID no app — o **banco** é a fonte de verdade.
- Usar `INSERT ... RETURNING "id"` para obter o UUID gerado.

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
- **Sempre** criar índice na coluna FK — PostgreSQL não cria automaticamente.
- `ON DELETE`:
  - `RESTRICT` (padrão): impede exclusão do pai se há filhos. Mais seguro.
  - `CASCADE`: exclui filhos. Usar apenas quando faz sentido (tenant → recursos).
  - `SET NULL`: para relações opcionais.
- `ON UPDATE CASCADE`: defensivo, raro com UUID.

## 6. Constraints

```sql
-- UNIQUE simples
CONSTRAINT "users_emailUnique" UNIQUE ("email")

-- UNIQUE composto
CONSTRAINT "users_tenantId_email_unique" UNIQUE ("tenantId", "email")

-- CHECK enumeração (preferível a ENUM type)
CONSTRAINT "orders_statusCheck"
    CHECK ("status" IN ('pending', 'processing', 'completed', 'cancelled'))

-- CHECK range
CONSTRAINT "orders_amountPositive" CHECK ("amountCents" > 0)

-- CHECK formato
CONSTRAINT "users_emailFormat"
    CHECK ("email" ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
```

- Todas as constraints **nomeadas explicitamente**.
- Preferir `TEXT` + `CHECK` em vez de `CREATE TYPE ... AS ENUM`.
- `NOT NULL` como padrão. Usar `NULL` apenas quando a ausência tem significado de negócio.

## 7. Índices

### Quando criar

| Situação                          | Índice                                              |
|-----------------------------------|-----------------------------------------------------|
| Coluna em `WHERE` frequente       | `CREATE INDEX`                                      |
| Coluna em `ORDER BY`              | `CREATE INDEX ... DESC/ASC`                         |
| Coluna em `JOIN` (FK)             | `CREATE INDEX` (**obrigatório** em toda FK)         |
| Busca por prefixo em texto        | `CREATE INDEX ... USING gin (col gin_trgm_ops)`     |
| Filtro em subconjunto             | `CREATE INDEX ... WHERE` (partial)                  |
| Coluna JSONB consultada           | `CREATE INDEX ... USING gin (col)`                  |

### Tipos

```sql
-- B-tree (padrão, 90% dos casos)
CREATE INDEX "idx_users_email" ON "users" ("email");

-- Ordenação para paginação
CREATE INDEX "idx_users_createdAt" ON "users" ("createdAt" DESC);

-- Composto
CREATE INDEX "idx_orders_userId_status" ON "orders" ("userId", "status");

-- Partial
CREATE INDEX "idx_orders_pending" ON "orders" ("createdAt")
    WHERE "status" = 'pending';

-- Unique index
CREATE UNIQUE INDEX "idx_users_email_unique" ON "users" ("email");

-- GIN para JSONB
CREATE INDEX "idx_users_metadata" ON "users" USING gin ("metadata");

-- GIN para busca textual
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX "idx_users_name_trgm" ON "users" USING gin ("name" gin_trgm_ops);

-- BRIN para tabelas grandes ordenadas por insert (logs)
CREATE INDEX "idx_auditLogs_createdAt" ON "auditLogs" USING brin ("createdAt");
```

### Índices compostos — ordem das colunas

A ordem **importa**: índice em `(a, b, c)` cobre `WHERE a` e `WHERE a AND b`, mas **não** cobre `WHERE b`.

**Regra da seletividade:** colunas mais seletivas (mais valores distintos) primeiro.

```sql
-- Query: WHERE "tenantId" = $1 AND "status" = $2 ORDER BY "createdAt" DESC
CREATE INDEX "idx_orders_tenantId_status_createdAt"
    ON "orders" ("tenantId", "status", "createdAt" DESC);
```

### Regras

- **Sempre** indexar colunas de FK.
- **Nunca** criar índice em coluna de baixa cardinalidade isolada (ex: `BOOLEAN`). Usar partial.
- **Sempre** validar com `EXPLAIN ANALYZE`.
- **Nunca** criar índices especulativos — apenas para queries existentes.
- Em tabelas grandes em produção, usar `CREATE INDEX CONCURRENTLY` para evitar lock.

## 8. Soft Delete

```sql
CREATE TABLE "users" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "name"      TEXT NOT NULL,
    "email"     TEXT NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "deletedAt" TIMESTAMPTZ  -- NULL = ativo
);

-- Unique parcial: email único apenas entre ativos
CREATE UNIQUE INDEX "idx_users_email_active"
    ON "users" ("email")
    WHERE "deletedAt" IS NULL;

-- Index parcial para queries que filtram ativos
CREATE INDEX "idx_users_active"
    ON "users" ("createdAt" DESC)
    WHERE "deletedAt" IS NULL;
```

- `"deletedAt" TIMESTAMPTZ` sem `NOT NULL` — `NULL` = ativo.
- Unique constraints devem ser **partial** com `WHERE "deletedAt" IS NULL`.
- Toda query de leitura deve incluir `WHERE "deletedAt" IS NULL`.
- Repositório expõe método `softDelete` que faz `UPDATE ... SET "deletedAt" = now()`.

## 9. Tabelas de Junção (N:N)

```sql
CREATE TABLE "userBusinessUnits" (
    "userId"         UUID NOT NULL,
    "businessUnitId" UUID NOT NULL,
    "assignedAt"     TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "userBusinessUnits_pk" PRIMARY KEY ("userId", "businessUnitId"),

    CONSTRAINT "userBusinessUnits_userId_fk"
        FOREIGN KEY ("userId") REFERENCES "users" ("id") ON DELETE CASCADE,

    CONSTRAINT "userBusinessUnits_businessUnitId_fk"
        FOREIGN KEY ("businessUnitId") REFERENCES "businessUnits" ("id") ON DELETE CASCADE
);

-- Índice reverso (PG só cria índice para o primeiro campo da PK)
CREATE INDEX "idx_userBusinessUnits_businessUnitId"
    ON "userBusinessUnits" ("businessUnitId");
```

- Nome: `<entidadeA><entidadeB>` em camelCase plural.
- PK composta com as duas FKs.
- `ON DELETE CASCADE` em ambas.
- Sempre criar **índice no segundo campo** da PK.

## 10. Migrations (dbmate)

Migrations em **SQL puro** executadas com [dbmate](https://github.com/amacneil/dbmate). Cada migration é **um único arquivo** com as seções `-- migrate:up` e `-- migrate:down`.

### Estrutura

```
migrations/
└── <app>/
    ├── 0001_setup.sql            # extensões + funções globais
    ├── 0002_create_users.sql
    ├── 0003_create_orders.sql
    └── ...
```

### Formato do arquivo

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
# Aplicar migrations pendentes
DATABASE_URL="postgres://app:secret@localhost:5432/app?sslmode=disable" \
    dbmate --migrations-dir ./migrations/<app> up

# Reverter a última migration aplicada
dbmate --migrations-dir ./migrations/<app> rollback

# Status das migrations
dbmate --migrations-dir ./migrations/<app> status
```

### Regras

- Numeração sequencial com 4 dígitos: `0001`, `0002` (padrão do projeto — não usar o timestamp do `dbmate new`; criar o arquivo manualmente com o próximo número).
- Todo arquivo tem **obrigatoriamente** as seções `-- migrate:up` e `-- migrate:down`.
- `-- migrate:down` reverte **exatamente** o que o `-- migrate:up` fez.
- Cada migration faz **uma coisa** (criar tabela, adicionar coluna, criar índice).
- **Nunca** alterar migration já aplicada em produção — criar nova.
- Testar `dbmate up` e `dbmate rollback` localmente antes de commit.
- `CREATE INDEX CONCURRENTLY` não roda dentro de transaction — nessas migrations, desabilitar o wrapping com `-- migrate:up transaction:false`.

### Migrations Zero-Downtime

| Operação                              | Segura? | Como Fazer                                                                |
|---------------------------------------|---------|---------------------------------------------------------------------------|
| Criar tabela                          | Sim     | `CREATE TABLE` normal                                                     |
| Adicionar coluna nullable             | Sim     | `ALTER TABLE ADD COLUMN ... NULL`                                         |
| Adicionar coluna NOT NULL com default | Sim     | `ALTER TABLE ADD COLUMN ... NOT NULL DEFAULT ...` (PG 11+ não reescreve)  |
| Remover coluna                        | Sim     | `ALTER TABLE DROP COLUMN` (após confirmar que nenhum código referencia)   |
| Criar índice                          | **Perigoso** | `CREATE INDEX CONCURRENTLY` (não bloqueia writes)                    |
| Renomear coluna                       | **Perigoso** | Nova coluna → copiar → dropar antiga, em migrations separadas         |
| Alterar tipo de coluna                | **Perigoso** | Criar nova, migrar dados, renomear                                    |
| Adicionar constraint                  | **Perigoso** | `ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` em migration separada |

## 11. Transactions

### Quando usar

| Cenário                                       | Transaction? | Por quê                                              |
|-----------------------------------------------|--------------|------------------------------------------------------|
| INSERT único                                  | **Não**      | Autocommit já garante atomicidade                    |
| SELECT único                                  | **Não**      | Leitura simples não precisa                          |
| INSERT + INSERT relacionados                  | **Sim**      | Tudo ou nada                                         |
| UPDATE condicional (read-then-write)          | **Sim**      | Evitar race condition                                |
| Transferência de saldo                        | **Sim**      | Débito e crédito atômicos                            |
| DELETE em cascata manual                      | **Sim**      | Consistência ao remover relacionados                 |
| Operação de negócio com múltiplas entidades   | **Sim**      | Order + estoque + log                                |
| Bulk insert                                   | **Sim**      | Sem registros órfãos                                 |
| Leitura consistente de múltiplas tabelas      | **Sim** (`READ ONLY`) | Snapshot isolation                          |

### Helper `src/shared/postgres/tx.ts`

Com `pg`, a transaction acontece em um **client dedicado** obtido de `pool.connect()`. **Nunca** use `pool.query('BEGIN')` — cada `pool.query` pode ir para uma conexão diferente. O client **sempre** volta ao pool via `client.release()` em `finally`.

```typescript
import type { Pool, PoolClient } from 'pg';

export type TxIsolationLevel = 'READ COMMITTED' | 'REPEATABLE READ' | 'SERIALIZABLE';

export interface TxOptions {
  isolationLevel?: TxIsolationLevel;
  readOnly?: boolean;
}

/**
 * runInTx executa fn dentro de uma transaction.
 * COMMIT se fn resolve, ROLLBACK se fn rejeita.
 */
export async function runInTx<T>(
  pool: Pool,
  fn: (client: PoolClient) => Promise<T>,
): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    try {
      await client.query('ROLLBACK');
    } catch (rollbackErr) {
      throw new Error(`rollback failed: ${String(rollbackErr)}`, { cause: err });
    }
    throw err;
  } finally {
    client.release();
  }
}

export async function runInTxWithOptions<T>(
  pool: Pool,
  opts: TxOptions,
  fn: (client: PoolClient) => Promise<T>,
): Promise<T> {
  const client = await pool.connect();
  try {
    let begin = 'BEGIN';
    if (opts.isolationLevel) begin += ` ISOLATION LEVEL ${opts.isolationLevel}`;
    if (opts.readOnly) begin += ' READ ONLY';

    await client.query(begin);
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    try {
      await client.query('ROLLBACK');
    } catch (rollbackErr) {
      throw new Error(`rollback failed: ${String(rollbackErr)}`, { cause: err });
    }
    throw err;
  } finally {
    client.release();
  }
}
```

### Uso no repositório

```typescript
export class OrderRepository {
  constructor(private readonly pool: Pool) {}

  async createWithItems(order: Order, items: OrderItem[]): Promise<void> {
    await runInTx(this.pool, async (client) => {
      const orderQuery = `
          INSERT INTO "orders" ("userId", "status", "totalCents", "createdAt", "updatedAt")
          VALUES ($1, $2, $3, $4, $5)
          RETURNING "id"`;

      const orderResult = await client.query<{ id: string }>(orderQuery, [
        order.userId, order.status, order.totalCents, order.createdAt, order.updatedAt,
      ]);
      order.id = orderResult.rows[0]!.id;

      const itemQuery = `
          INSERT INTO "orderItems" ("orderId", "productId", "quantity", "unitPriceCents")
          VALUES ($1, $2, $3, $4)
          RETURNING "id"`;

      for (const item of items) {
        item.orderId = order.id;
        const itemResult = await client.query<{ id: string }>(itemQuery, [
          item.orderId, item.productId, item.quantity, item.unitPriceCents,
        ]);
        item.id = itemResult.rows[0]!.id;
      }
    });
  }
}
```

### Orquestração no use case com `Transactor`

Quando a transaction envolve **múltiplos repositórios**, o use case orquestra via interface `Transactor`:

```typescript
// domain/repository/transactor.ts
import type { PoolClient } from 'pg';

export interface Transactor {
  runInTx<T>(fn: (client: PoolClient) => Promise<T>): Promise<T>;
}
```

```typescript
// infrastructure/repository/postgres-transactor.ts
import type { Pool, PoolClient } from 'pg';
import { runInTx } from '../../shared/postgres/tx.js';
import type { Transactor } from '../../domain/repository/transactor.js';

export class PostgresTransactor implements Transactor {
  constructor(private readonly pool: Pool) {}

  runInTx<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    return runInTx(this.pool, fn);
  }
}
```

```typescript
// application/usecase/billing/transfer-usecase.ts
export class TransferUseCase {
  constructor(
    private readonly transactor: Transactor,
    private readonly accountRepository: AccountRepository,
  ) {}

  async perform(input: TransferInput): Promise<void> {
    await this.transactor.runInTx(async (client) => {
      await this.accountRepository.debitWithTx(client, input.fromAccountId, input.amountCents);
      await this.accountRepository.creditWithTx(client, input.toAccountId, input.amountCents);
    });
  }
}
```

Repositórios que participam de tx externa expõem variantes `withTx` que recebem o `PoolClient`:

```typescript
export class AccountRepository {
  async debitWithTx(client: PoolClient, id: string, amountCents: number): Promise<void> {
    const query = `
        UPDATE "accounts"
        SET "balanceCents" = "balanceCents" - $1, "updatedAt" = now()
        WHERE "id" = $2 AND "balanceCents" >= $1`;

    const result = await client.query(query, [amountCents, id]);
    if (result.rowCount === 0) {
      throw new InsufficientBalanceError(id);
    }
  }
}
```

### Níveis de isolamento

| Nível                       | Quando usar                              | Config                                                        |
|-----------------------------|------------------------------------------|---------------------------------------------------------------|
| `READ COMMITTED` (padrão)   | 90% dos casos                            | `runInTx(pool, fn)`                                           |
| `REPEATABLE READ`           | Leitura consistente entre tabelas        | `runInTxWithOptions(pool, { isolationLevel: 'REPEATABLE READ' }, fn)` |
| `SERIALIZABLE`              | Operações críticas de saldo              | `runInTxWithOptions(pool, { isolationLevel: 'SERIALIZABLE' }, fn)` |
| `READ ONLY`                 | Relatórios                               | `runInTxWithOptions(pool, { readOnly: true }, fn)`            |

### Regras de transactions

| Regra                                    | Detalhe                                                                          |
|------------------------------------------|----------------------------------------------------------------------------------|
| **Transactions curtas**                  | Milissegundos. Quanto mais longa, mais locks retidos                             |
| **Nenhum I/O externo dentro do tx**      | Nada de HTTP, NATS, RabbitMQ dentro do tx — sempre **após** o commit             |
| **Evento DEPOIS do commit**              | `repository.create` dentro do tx, `event.publish` fora                           |
| **Retry em serialization failure**       | `SQLSTATE 40001` (`err.code === '40001'`) deve ser retentado pelo app            |
| **Sem tx para operações únicas**         | INSERT, SELECT, UPDATE únicos não precisam                                       |
| **Sem tx aninhados**                     | PostgreSQL não suporta — usar `SAVEPOINT` se necessário                          |
| **Timeout na operação**                  | Sempre limitar duração (`statement_timeout` na sessão ou `AbortSignal` no app)   |
| **Commit explícito**                     | Nunca confiar em close automático                                                |
| **`client.release()` sempre em `finally`** | Client não devolvido = conexão vazada até esgotar o pool                       |

### Anti-pattern: evento dentro da transaction

```typescript
// ERRADO — evento publicado antes do commit
await runInTx(this.pool, async (client) => {
  await this.repository.createWithTx(client, entity);
  await this.event.publishCreated(entity); // ERRADO
});

// CORRETO — evento após o commit
await this.repository.create(entity);
await this.event.publishCreated(entity).catch(() => { /* logar, não propagar */ });
```

## 12. Paginação

### Por cursor (recomendada)

```sql
-- Primeira página
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "deletedAt" IS NULL
ORDER BY "id" ASC
LIMIT $1;

-- Próximas (cursor = último ID)
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "deletedAt" IS NULL AND "id" > $1
ORDER BY "id" ASC
LIMIT $2;
```

- Performance constante (usa índice).
- Sem duplicados/pulos quando dados mudam.
- Não usar quando o usuário precisa pular para uma página específica.

### Por offset (apenas quando necessário)

```sql
SELECT "id", "name", "email"
FROM "users"
WHERE "deletedAt" IS NULL
ORDER BY "createdAt" DESC
LIMIT $1 OFFSET $2;
```

- Performance degrada com offsets grandes.
- Resultados inconsistentes se dados mudam.
- Usar **apenas** em dashboards administrativos com poucas páginas.

## 13. Queries — Boas Práticas

### SELECT explícito (nunca `SELECT *`)

```sql
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "id" = $1;
```

### INSERT com RETURNING

```sql
INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4)
RETURNING "id", "createdAt";
```

### UPSERT (idempotência)

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

### Bulk insert

```sql
INSERT INTO "orderItems" ("orderId", "productId", "quantity", "unitPriceCents")
VALUES
    ($1, $2, $3, $4),
    ($5, $6, $7, $8),
    ($9, $10, $11, $12)
RETURNING "id";
```

### Locking

```sql
-- Lock para read-then-write
SELECT "id", "balanceCents"
FROM "accounts"
WHERE "id" = $1
FOR UPDATE;

-- Filas de trabalho (concorrência segura)
SELECT "id", "payload"
FROM "jobs"
WHERE "status" = 'pending'
ORDER BY "createdAt" ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

### Não-encontrado retorna `null`

Com `pg`, "não encontrado" **não é erro** — a query resolve com zero rows. O repositório retorna `null`:

```typescript
async findById(id: string): Promise<User | null> {
  const query = `
      SELECT "id", "name", "email", "createdAt", "updatedAt"
      FROM "users"
      WHERE "id" = $1`;

  const result = await this.pool.query<UserRow>(query, [id]);
  const row = result.rows[0];
  return row ? this.toEntity(row) : null;
}
```

- `findBy*` retorna `Promise<Entity | null>` — nunca lança erro por ausência.
- Sempre usar **placeholders posicionais** (`$1`, `$2`) — nunca interpolar valores na string (SQL injection).
- `pool.query(text, values)` para operações únicas; `PoolClient` de `pool.connect()` apenas para transactions.

## 14. Pool de Conexões (`pg.Pool`)

```typescript
// src/shared/postgres/pool.ts
import { Pool } from 'pg';

export function createPool(cfg: PostgresConfig): Pool {
  const pool = new Pool({
    host: cfg.host,
    port: cfg.port,
    user: cfg.user,
    password: cfg.password,
    database: cfg.database,

    max: cfg.maxConnections,          // 10-25 (2-5× núcleos do DB)
    idleTimeoutMillis: 300_000,       // 5 min — libera conexões ociosas
    connectionTimeoutMillis: 5_000,   // falha rápido se o pool está esgotado
    maxUses: 7_500,                   // rotaciona a conexão (DNS, PgBouncer)
    allowExitOnIdle: false,           // servidores de longa duração
  });

  // OBRIGATÓRIO: sem este handler, um erro em conexão ociosa derruba o processo
  pool.on('error', (err) => {
    logger.error({ err }, 'postgres idle client error');
  });

  return pool;
}
```

| Parâmetro                  | Recomendação        | Justificativa                                            |
|----------------------------|---------------------|----------------------------------------------------------|
| `max`                      | 10-25               | `max_connections` do PG ÷ instâncias do app              |
| `idleTimeoutMillis`        | 300000 (5 min)      | Liberar conns em baixo tráfego                           |
| `connectionTimeoutMillis`  | 5000                | Erro claro em pool esgotado em vez de espera infinita    |
| `maxUses`                  | ~7500               | Rotacionar conexões para respeitar DNS/PgBouncer         |
| `pool.on('error')`         | **sempre registrar**| Erro em client ocioso sem handler = crash do processo    |
| `pool.end()`               | no graceful shutdown| Aguarda clients em uso e fecha todas as conexões         |

## 15. Configuração do Servidor (PostgreSQL 18)

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
    volumes:
      - pgdata:/var/lib/postgresql/data
    command:
      - "postgres"
      # Memória
      - "-c"
      - "shared_buffers=256MB"            # ~25% da RAM
      - "-c"
      - "effective_cache_size=512MB"      # 50-75% da RAM
      - "-c"
      - "work_mem=16MB"                   # por operação de sort/hash
      - "-c"
      - "maintenance_work_mem=128MB"      # VACUUM, CREATE INDEX
      # WAL
      - "-c"
      - "wal_buffers=16MB"
      - "-c"
      - "min_wal_size=512MB"
      - "-c"
      - "max_wal_size=2GB"
      # Planner
      - "-c"
      - "random_page_cost=1.1"            # SSD
      - "-c"
      - "effective_io_concurrency=200"    # SSD
      # Logging
      - "-c"
      - "log_min_duration_statement=100"  # loga queries > 100ms
      - "-c"
      - "log_statement=none"
      - "-c"
      - "log_lock_waits=on"
      # Conexões
      - "-c"
      - "max_connections=100"
      # Checkpoints
      - "-c"
      - "checkpoint_completion_target=0.9"

volumes:
  pgdata:
```

## Checklist — Nova Tabela

- [ ] Nome em camelCase plural com aspas duplas (`"orderItems"`)
- [ ] `"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [ ] Colunas em camelCase com aspas duplas
- [ ] `TIMESTAMPTZ` para datas (nunca `TIMESTAMP`)
- [ ] `"createdAt" TIMESTAMPTZ NOT NULL DEFAULT now()`
- [ ] `"updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now()`
- [ ] Trigger `"trg_<tabela>_updatedAt"` registrado
- [ ] Constraints nomeadas (`CONSTRAINT "<tabela>_<descricao>"`)
- [ ] FKs com índice na coluna FK
- [ ] FKs com `ON DELETE` explícito
- [ ] `NOT NULL` em todas as colunas obrigatórias
- [ ] `CHECK` para enumerações (em vez de `ENUM` type)
- [ ] Índices para colunas em `WHERE` e `ORDER BY`
- [ ] `INSERT ... RETURNING "id"` no repositório
- [ ] `findBy*` retorna `null` quando não encontrado (zero rows não é erro)
- [ ] `BIGINT` convertido de string para número no repositório
- [ ] Migration com `-- migrate:up` e `-- migrate:down` criada e testada

## Checklist — Nova Transaction

- [ ] Múltiplas escritas ou read-then-write? Se sim, usar transaction
- [ ] `runInTx` ou `runInTxWithOptions` do helper compartilhado
- [ ] Client obtido de `pool.connect()` — nunca `pool.query('BEGIN')`
- [ ] `client.release()` garantido em `finally`
- [ ] Nenhuma chamada externa (HTTP, NATS, RabbitMQ) dentro do tx
- [ ] Evento publicado **depois** do commit
- [ ] Duração limitada (`statement_timeout` ou timeout no app)
- [ ] Nível de isolamento adequado (`READ COMMITTED` na maioria)
- [ ] Retry para `serialization_failure` (`err.code === '40001'`) se usando `SERIALIZABLE`
- [ ] Repositórios participantes expõem variantes `withTx(client, ...)`

## Checklist — Nova Migration

- [ ] Numeração sequencial com 4 dígitos (`0001`, `0002`)
- [ ] Seções `-- migrate:up` e `-- migrate:down` presentes
- [ ] `-- migrate:down` reverte exatamente o `-- migrate:up`
- [ ] Uma única responsabilidade por migration
- [ ] `CREATE INDEX CONCURRENTLY` para tabelas grandes (com `transaction:false`)
- [ ] Constraints adicionadas via `NOT VALID` + `VALIDATE` separados
- [ ] Testada localmente (`dbmate up` e `dbmate rollback`) antes do commit
- [ ] Não altera migration já aplicada em produção

## Exemplo Completo — Migration de Tabela

```sql
-- migrations/billing/0002_create_orders.sql

-- migrate:up
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
        FOREIGN KEY ("userId")
        REFERENCES "users" ("id")
        ON DELETE RESTRICT
        ON UPDATE CASCADE,

    CONSTRAINT "orders_statusCheck"
        CHECK ("status" IN ('pending', 'processing', 'completed', 'cancelled', 'refunded')),

    CONSTRAINT "orders_totalPositive"
        CHECK ("totalCents" > 0),

    CONSTRAINT "orders_currencyCheck"
        CHECK ("currency" IN ('BRL', 'USD', 'EUR'))
);

CREATE TRIGGER "trg_orders_updatedAt"
    BEFORE UPDATE ON "orders"
    FOR EACH ROW
    EXECUTE FUNCTION "setUpdatedAt"();

CREATE INDEX "idx_orders_userId"          ON "orders" ("userId");
CREATE INDEX "idx_orders_status"          ON "orders" ("status") WHERE "status" IN ('pending', 'processing');
CREATE INDEX "idx_orders_createdAt"       ON "orders" ("createdAt" DESC);
CREATE INDEX "idx_orders_userId_status"   ON "orders" ("userId", "status");

COMMENT ON TABLE  "orders"               IS 'Pedidos de compra dos usuários';
COMMENT ON COLUMN "orders"."totalCents"  IS 'Valor total em centavos';
COMMENT ON COLUMN "orders"."metadata"    IS 'Dados adicionais semi-estruturados do pedido';

-- migrate:down
DROP TRIGGER IF EXISTS "trg_orders_updatedAt" ON "orders";
DROP TABLE   IF EXISTS "orders";
```

## Fontes

- https://www.postgresql.org/docs/18/ — documentação oficial PG 18
- https://www.postgresql.org/docs/18/ddl.html — DDL, constraints, herança
- https://www.postgresql.org/docs/18/datatype.html — tipos de dados
- https://www.postgresql.org/docs/18/indexes.html — índices (B-tree, GIN, BRIN, partial)
- https://www.postgresql.org/docs/18/transaction-iso.html — níveis de isolamento
- https://www.postgresql.org/docs/18/explicit-locking.html — locks e `FOR UPDATE`/`SKIP LOCKED`
- https://www.postgresql.org/docs/18/runtime-config.html — configuração do servidor
- https://node-postgres.com/ — documentação do `pg` (Pool, transactions, tipos)
- https://github.com/amacneil/dbmate — ferramenta de migrations
