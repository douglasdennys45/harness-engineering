---
name: testcontainers-node
description: Use esta skill ao escrever testes de integração Node.js/TypeScript com containers Docker usando a biblioteca testcontainers (testcontainers-node) — módulos pré-configurados @testcontainers/* (postgresql, nats, redis, kafka, etc.), GenericContainer, wait strategies (Wait.forLogMessage, forListeningPorts, forHealthCheck, forHttp), networking entre containers, Docker Compose, reuso de containers e helpers compartilhados em test/integration/testhelper/. Cobre ciclo de vida com vitest (beforeAll/afterAll, hookTimeout, globalSetup), aplicação de migrations dbmate no container, padrões de cleanup/isolamento (TRUNCATE vs banco por teste vs snapshot) e integração com pg e nats (JetStream). Acione ao criar/editar testes em apps/<app>/test/integration/, configurar infraestrutura de teste baseada em containers ou depurar containers de teste.
keywords: [nodejs, typescript, testing, docker, integration-tests, testcontainers, vitest, postgresql, nats]
license: MIT
---

# Testcontainers para Testes de Integração em Node.js

Guia completo para usar a biblioteca **testcontainers** (testcontainers-node) para escrever testes de integração confiáveis com containers Docker em projetos Node.js + TypeScript.

## Descrição

Esta skill ajuda a escrever testes de integração usando Testcontainers para Node.js, uma biblioteca que fornece instâncias leves e descartáveis de bancos de dados, message brokers ou qualquer serviço que rode em um container Docker — tudo controlado a partir do próprio código de teste.

**Capacidades principais:**
- Usar 40+ módulos pré-configurados (`@testcontainers/postgresql`, `@testcontainers/nats`, etc.)
- Subir e gerenciar containers Docker dentro de suites vitest
- Configurar networking, volumes, variáveis de ambiente e wait strategies
- Compartilhar containers entre testes via helpers em `test/integration/testhelper/`
- Aplicar migrations (dbmate) no container antes dos testes
- Implementar cleanup e isolamento corretos entre testes

## Quando Usar Esta Skill

Use esta skill quando precisar:
- Escrever testes de integração que exigem serviços reais (PostgreSQL, NATS, Redis, etc.)
- Testar repositórios `pg` contra um PostgreSQL 18 real, sem mocks
- Testar publishers/subscribers NATS JetStream contra um broker real
- Testar contra múltiplas versões ou configurações de dependências
- Criar ambientes de teste reprodutíveis, sem depender de infra local pré-instalada
- Montar infraestrutura de teste efêmera em CI

**Convenções do projeto:**
- Testes de integração vivem em `apps/<app>/test/integration/`
- Helpers compartilhados vivem em `apps/<app>/test/integration/testhelper/` (`postgres.ts`, `nats.ts`)
- Runner: **vitest**. Banco: **PostgreSQL 18** via `pg`. Mensageria: **NATS JetStream** via `nats`. Migrations: **dbmate**.

## Pré-requisitos

- **Docker** (ou Podman/Colima/Rancher Desktop — ver seção de Configuração) instalado e rodando
- **Node.js 22 LTS** e **pnpm** (workspace do monorepo)
- **TypeScript 5** em modo strict com ESM

## Instruções

### 1. Instalação & Setup

Adicione os pacotes como devDependencies do app (dentro do workspace pnpm):

```bash
# Núcleo da biblioteca
pnpm add -D testcontainers --filter @org/user

# Módulos pré-configurados (recomendado)
pnpm add -D @testcontainers/postgresql --filter @org/user
pnpm add -D @testcontainers/nats --filter @org/user
pnpm add -D @testcontainers/redis --filter @org/user
```

Clientes usados nos testes (já são dependências de produção do app):

```bash
pnpm add pg nats --filter @org/user
pnpm add -D @types/pg --filter @org/user
```

**Configuração do vitest para testes de integração** — containers demoram para subir, então os hooks precisam de timeout maior que o padrão (10s):

```typescript
// apps/user/vitest.integration.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["test/integration/**/*.test.ts"],
    // hooks (beforeAll/afterAll) sobem/derrubam containers — precisam de folga
    hookTimeout: 120_000,
    // testes individuais falam com serviços reais — mais lentos que unitários
    testTimeout: 30_000,
    // cada ARQUIVO de teste roda em um processo worker separado (padrão do vitest).
    // Singletons de módulo no testhelper valem por arquivo — ver seção 4.
    pool: "forks",
  },
});
```

```jsonc
// apps/user/package.json (scripts)
{
  "scripts": {
    "test": "vitest run",
    "test:integration": "vitest run --config vitest.integration.config.ts"
  }
}
```

**Verificar disponibilidade do Docker** antes de rodar a suite (útil em máquinas sem Docker):

```bash
docker info > /dev/null 2>&1 || echo "Docker indisponível — testes de integração serão pulados"
```

---

### 2. Usando Módulos Pré-Configurados (Abordagem Recomendada)

**Testcontainers para Node.js oferece 40+ módulos pré-configurados** publicados como pacotes `@testcontainers/*`. **Sempre prefira módulos a `GenericContainer`** quando existirem.

#### Por Que Usar Módulos?

- **Defaults sensatos**: portas, variáveis de ambiente e wait strategies já configurados
- **Helpers de conexão**: métodos como `getConnectionUri()`, `getConnectionOptions()`
- **Funcionalidades especializadas**: ex. snapshot/restore no PostgreSQL, `withJetStream()` no NATS
- **Credenciais automáticas**: geração e gestão segura de credenciais
- **Testados em batalha**: usados em produção por milhares de projetos

#### Principais Módulos `@testcontainers/*`

| Pacote | Classe | Uso típico |
|---|---|---|
| `@testcontainers/postgresql` | `PostgreSqlContainer` | PostgreSQL (nosso banco padrão) |
| `@testcontainers/nats` | `NatsContainer` | NATS / JetStream (nossa mensageria) |
| `@testcontainers/redis` | `RedisContainer` | Cache / filas simples |
| `@testcontainers/kafka` | `KafkaContainer` | Kafka |
| `@testcontainers/rabbitmq` | `RabbitMQContainer` | RabbitMQ |
| `@testcontainers/mysql` | `MySqlContainer` | MySQL |
| `@testcontainers/mariadb` | `MariaDbContainer` | MariaDB |
| `@testcontainers/mongodb` | `MongoDBContainer` | MongoDB |
| `@testcontainers/mssqlserver` | `MSSQLServerContainer` | SQL Server |
| `@testcontainers/cockroachdb` | `CockroachDbContainer` | CockroachDB |
| `@testcontainers/cassandra` | `CassandraContainer` | Cassandra |
| `@testcontainers/scylladb` | `ScyllaDbContainer` | ScyllaDB |
| `@testcontainers/clickhouse` | `ClickHouseContainer` | ClickHouse |
| `@testcontainers/elasticsearch` | `ElasticsearchContainer` | Elasticsearch |
| `@testcontainers/opensearch` | `OpenSearchContainer` | OpenSearch |
| `@testcontainers/neo4j` | `Neo4jContainer` | Neo4j |
| `@testcontainers/couchbase` | `CouchbaseContainer` | Couchbase |
| `@testcontainers/valkey` | `ValkeyContainer` | Valkey (fork do Redis) |
| `@testcontainers/redpanda` | `RedpandaContainer` | Redpanda (Kafka-compatible) |
| `@testcontainers/localstack` | `LocalstackContainer` | AWS local (S3, SQS, etc.) |
| `@testcontainers/azurite` | `AzuriteContainer` | Azure Storage local |
| `@testcontainers/gcloud` | emuladores GCP | Firestore, BigQuery, Pub/Sub |
| `@testcontainers/minio` | `MinioContainer` | Object storage S3-compatible |
| `@testcontainers/vault` | `VaultContainer` | HashiCorp Vault |
| `@testcontainers/etcd` | `EtcdContainer` | etcd |
| `@testcontainers/k3s` | `K3sContainer` | Kubernetes leve |
| `@testcontainers/mockserver` | `MockserverContainer` | Mock de APIs HTTP |
| `@testcontainers/toxiproxy` | `ToxiProxyContainer` | Injeção de falhas de rede |
| `@testcontainers/selenium` | `SeleniumContainer` | Browsers para E2E |
| `@testcontainers/ollama` | `OllamaContainer` | LLMs locais |
| `@testcontainers/qdrant` / `weaviate` / `chromadb` | — | Vector databases |

Lista completa e atualizada: https://testcontainers.com/modules/?language=nodejs

#### PostgreSQL — Uso Básico do Módulo

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { Pool } from "pg";

describe("UserRepository", () => {
  let container: StartedPostgreSqlContainer;
  let pool: Pool;

  beforeAll(async () => {
    // Sobe PostgreSQL 18 com credenciais e wait strategy prontos
    container = await new PostgreSqlContainer("postgres:18-alpine")
      .withDatabase("app_test")
      .withUsername("test")
      .withPassword("test")
      .start();

    pool = new Pool({ connectionString: container.getConnectionUri(), max: 5 });
  }, 120_000); // timeout explícito do hook (além do hookTimeout global)

  afterAll(async () => {
    await pool.end();       // SEMPRE fechar o client antes do container
    await container.stop();
  });

  it("conecta e executa query", async () => {
    const result = await pool.query("SELECT 1 AS ok");
    expect(result.rows[0].ok).toBe(1);
  });
});
```

**Métodos úteis do `StartedPostgreSqlContainer`:**

```typescript
container.getConnectionUri();  // "postgresql://test:test@localhost:32789/app_test"
container.getHost();           // "localhost"
container.getPort();           // porta mapeada (aleatória) da 5432
container.getDatabase();       // "app_test"
container.getUsername();       // "test"
container.getPassword();       // "test"

// Snapshot & Restore — isolamento rápido entre testes (ver seção 8)
await container.snapshot();
await container.restoreSnapshot();
```

#### NATS — Uso Básico do Módulo

```typescript
import { NatsContainer, type StartedNatsContainer } from "@testcontainers/nats";
import { connect, type NatsConnection } from "nats";

let container: StartedNatsContainer;
let nc: NatsConnection;

beforeAll(async () => {
  // withJetStream() habilita a flag -js no servidor
  container = await new NatsContainer("nats:2.12-alpine")
    .withJetStream()
    .start();

  nc = await connect(container.getConnectionOptions());
}, 120_000);

afterAll(async () => {
  await nc.drain();       // drena mensagens pendentes e fecha
  await container.stop();
});
```

> Se preferir controle total sobre o comando do servidor (ex.: `--store_dir`, limites de memória), use `GenericContainer` com `nats:2.12-alpine` e `withCommand(["-js"])` — ver seção 9.

#### `await using` — Cleanup Automático (Explicit Resource Management)

Com TypeScript 5.2+ e `"target": "es2022"` + lib `esnext.disposable`, containers suportam `await using`, que chama `stop()` automaticamente ao sair do escopo:

```typescript
it("teste isolado com container próprio", async () => {
  await using container = await new PostgreSqlContainer("postgres:18-alpine").start();
  const pool = new Pool({ connectionString: container.getConnectionUri() });
  // ... teste ...
  await pool.end();
}); // container.stop() automático aqui, mesmo em caso de erro
```

Para containers compartilhados por suite (nosso padrão), prefira `beforeAll`/`afterAll` explícitos — `await using` é ideal para testes pontuais que precisam de container exclusivo.

---

### 3. Usando GenericContainer (Fallback)

Quando não existe módulo pré-configurado — ou quando você quer controle total — use `GenericContainer`.

**IMPORTANTE: sempre defina uma wait strategy adequada.** O default é `Wait.forListeningPorts()` (aguarda as portas expostas ficarem de fato escutando), o que cobre a maioria dos casos — mas serviços que abrem a porta antes de estarem prontos (ex.: bancos que fazem restart interno no primeiro boot) exigem `Wait.forLogMessage()` ou `Wait.forHealthCheck()`. **Nunca use `setTimeout`/sleep** como espera — é anti-padrão que gera testes flaky.

```typescript
import { GenericContainer, Wait, type StartedTestContainer } from "testcontainers";

const container: StartedTestContainer = await new GenericContainer("nats:2.12-alpine")
  .withCommand(["-js"])                                  // habilita JetStream
  .withExposedPorts(4222)
  .withWaitStrategy(Wait.forLogMessage(/Server is ready/))
  .withStartupTimeout(60_000)
  .start();

const url = `nats://${container.getHost()}:${container.getMappedPort(4222)}`;

// ... testes ...

await container.stop();
```

**Opções comuns do `GenericContainer`:**

```typescript
new GenericContainer("image:tag")
  // Portas — o host recebe uma porta ALEATÓRIA mapeada para a do container
  .withExposedPorts(8080, 4222)
  .withExposedPorts({ container: 80, host: 8080 })   // binding fixo (evite!)

  // Ambiente
  .withEnvironment({ APP_ENV: "test", LOG_LEVEL: "debug" })

  // Comando e entrypoint
  .withCommand(["-js", "--store_dir", "/data"])
  .withEntrypoint(["/custom-entrypoint.sh"])

  // Arquivos e conteúdo
  .withCopyFilesToContainer([
    { source: "./testdata/nginx.conf", target: "/etc/nginx/conf.d/default.conf", mode: 0o644 },
  ])
  .withCopyContentToContainer([
    { content: "hello world", target: "/app/config.txt" },
  ])
  .withCopyDirectoriesToContainer([
    { source: "./testdata/scripts", target: "/docker-entrypoint-initdb.d" },
  ])

  // Volumes
  .withBindMounts([{ source: "/host/path", target: "/container/path", mode: "ro" }])
  .withTmpFs({ "/var/lib/postgresql/data": "rw,noexec,nosuid,size=512m" })

  // Identidade e metadados
  .withName("meu-container-de-teste")   // evite — nomes fixos colidem em paralelo
  .withLabels({ app: "user-service", env: "test" })
  .withUser("1000:1000")

  // Wait strategy + timeout
  .withWaitStrategy(Wait.forListeningPorts())
  .withStartupTimeout(60_000)

  // Recursos e plataforma
  .withResourcesQuota({ memory: 0.5, cpu: 1 })       // GB / CPUs
  .withPlatform("linux/amd64")
  .withPullPolicy(PullPolicy.alwaysPull())

  // Reuso entre execuções (ver seção 10)
  .withReuse()

  .start();
```

**Acessando host e portas do container iniciado:**

```typescript
const host = container.getHost();                 // "localhost" (ou IP do Docker host)
const port = container.getMappedPort(4222);       // porta aleatória mapeada, ex.: 32771
const portUdp = container.getMappedPort("53/udp");
const first = container.getFirstMappedPort();     // primeira porta exposta

// NUNCA use a porta interna (4222) para conectar do host —
// sempre getMappedPort(). A porta interna só vale ENTRE containers na mesma network.
```

**Executando comandos dentro do container:**

```typescript
const { output, stdout, stderr, exitCode } = await container.exec(["psql", "-U", "test", "-c", "SELECT 1"]);
expect(exitCode).toBe(0);

// Com opções
await container.exec(["ls", "-la"], {
  workingDir: "/app",
  user: "1000:1000",
  env: { PGPASSWORD: "test" },
});
```

**Lendo logs do container:**

```typescript
// Stream de logs após o start
(await container.logs())
  .on("data", (line) => console.log(line))
  .on("err", (line) => console.error(line));

// Consumer registrado ANTES do start (captura o boot inteiro)
const container = await new GenericContainer("myapp:latest")
  .withLogConsumer((stream) => {
    stream.on("data", (line) => console.log("[app]", line));
  })
  .start();
```

---

### 4. Ciclo de Vida do Container em Suites vitest

#### Padrão Central: Container Compartilhado por Suite via Testhelper

Subir um container por **teste** é lento demais; subir por **suite (arquivo)** é o equilíbrio certo. O padrão do projeto é centralizar o boot em helpers de `test/integration/testhelper/` e reutilizá-los em todas as suites:

**Como o vitest se comporta:** por padrão (`pool: "forks"`), **cada arquivo de teste roda em um processo worker separado**. Um singleton em nível de módulo no testhelper vale, portanto, **por arquivo** — dois arquivos de teste rodando em paralelo terão dois containers independentes (o que é bom: isolamento total entre arquivos). Para compartilhar **um único container em toda a execução**, use `globalSetup` (abaixo) ou `withReuse()` (seção 10).

#### `testhelper/postgres.ts` — Container PostgreSQL Compartilhado

```typescript
// apps/user/test/integration/testhelper/postgres.ts
import { execFileSync } from "node:child_process";
import path from "node:path";
import { fileURLToPath } from "node:url";

import { PostgreSqlContainer, type StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { Pool } from "pg";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const MIGRATIONS_DIR = path.resolve(__dirname, "../../../../../migrations/user");

export interface PostgresTestContext {
  container: StartedPostgreSqlContainer;
  pool: Pool;
  connectionUri: string;
}

// Singleton por processo worker (= por arquivo de teste no pool "forks")
let context: PostgresTestContext | undefined;

/**
 * Sobe (uma vez por worker) um PostgreSQL 18 com as migrations aplicadas
 * e retorna um Pool pronto para uso.
 */
export async function setupPostgres(): Promise<PostgresTestContext> {
  if (context) {
    return context;
  }

  const container = await new PostgreSqlContainer("postgres:18-alpine")
    .withDatabase("app_test")
    .withUsername("test")
    .withPassword("test")
    // tmpfs acelera MUITO: dados ficam em memória, não em disco
    .withTmpFs({ "/var/lib/postgresql/data": "rw" })
    .start();

  const connectionUri = container.getConnectionUri();

  runMigrations(connectionUri);

  const pool = new Pool({ connectionString: connectionUri, max: 5 });

  context = { container, pool, connectionUri };
  return context;
}

/** Derruba pool e container. Chamar no afterAll da suite. */
export async function teardownPostgres(): Promise<void> {
  if (!context) return;
  await context.pool.end();       // fechar conexões ANTES de parar o container
  await context.container.stop();
  context = undefined;
}

/**
 * Aplica as migrations dbmate no banco do container.
 * dbmate roda no HOST apontando para a porta mapeada do container.
 */
function runMigrations(connectionUri: string): void {
  execFileSync("dbmate", ["--no-dump-schema", "up"], {
    env: {
      ...process.env,
      DATABASE_URL: `${connectionUri}?sslmode=disable`,
      DBMATE_MIGRATIONS_DIR: MIGRATIONS_DIR,
    },
    stdio: "inherit",
  });
}

/**
 * Trunca todas as tabelas do schema public (exceto schema_migrations).
 * Chamar no beforeEach para isolamento entre testes.
 */
export async function truncateAllTables(pool: Pool): Promise<void> {
  const { rows } = await pool.query<{ tablename: string }>(
    `SELECT tablename FROM pg_tables
     WHERE schemaname = 'public' AND tablename <> 'schema_migrations'`,
  );
  if (rows.length === 0) return;

  const tables = rows.map((r) => `"${r.tablename}"`).join(", ");
  await pool.query(`TRUNCATE TABLE ${tables} RESTART IDENTITY CASCADE`);
}
```

#### `testhelper/nats.ts` — Container NATS JetStream Compartilhado

```typescript
// apps/user/test/integration/testhelper/nats.ts
import { GenericContainer, Wait, type StartedTestContainer } from "testcontainers";
import {
  connect,
  type NatsConnection,
  type JetStreamClient,
  type JetStreamManager,
} from "nats";

export interface NatsTestContext {
  container: StartedTestContainer;
  nc: NatsConnection;
  js: JetStreamClient;
  jsm: JetStreamManager;
  url: string;
}

// Singleton por processo worker
let context: NatsTestContext | undefined;

/**
 * Sobe (uma vez por worker) um NATS 2.12 com JetStream habilitado (-js)
 * e retorna conexão, cliente JetStream e manager prontos.
 */
export async function setupNats(): Promise<NatsTestContext> {
  if (context) {
    return context;
  }

  const container = await new GenericContainer("nats:2.12-alpine")
    .withCommand(["-js"]) // habilita JetStream
    .withExposedPorts(4222)
    .withWaitStrategy(Wait.forLogMessage(/Server is ready/))
    .start();

  const url = `nats://${container.getHost()}:${container.getMappedPort(4222)}`;

  const nc = await connect({ servers: url });
  const js = nc.jetstream();
  const jsm = await nc.jetstreamManager();

  context = { container, nc, js, jsm, url };
  return context;
}

/** Drena a conexão e derruba o container. Chamar no afterAll da suite. */
export async function teardownNats(): Promise<void> {
  if (!context) return;
  await context.nc.drain();
  await context.container.stop();
  context = undefined;
}

/** Remove todos os streams — isolamento entre testes JetStream. */
export async function deleteAllStreams(jsm: JetStreamManager): Promise<void> {
  const streams = await jsm.streams.list().next();
  await Promise.all(streams.map((s) => jsm.streams.delete(s.config.name)));
}
```

#### Usando os Helpers em uma Suite

```typescript
// apps/user/test/integration/user-repository.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";

import {
  setupPostgres,
  teardownPostgres,
  truncateAllTables,
  type PostgresTestContext,
} from "./testhelper/postgres.js";
import { UserRepository } from "../../src/infrastructure/repository/user-repository.js";

describe("UserRepository (integração)", () => {
  let pg: PostgresTestContext;
  let repository: UserRepository;

  beforeAll(async () => {
    pg = await setupPostgres();               // container + migrations, 1x por worker
    repository = new UserRepository(pg.pool);
  }, 120_000);

  afterAll(async () => {
    await teardownPostgres();
  });

  beforeEach(async () => {
    await truncateAllTables(pg.pool);         // banco limpo antes de cada teste
  });

  it("cria e busca usuário por email", async () => {
    const created = await repository.create({ name: "Alice", email: "alice@example.com" });
    expect(created.id).toBeTruthy();

    const found = await repository.findByEmail("alice@example.com");
    expect(found?.name).toBe("Alice");
  });

  it("retorna null quando usuário não existe", async () => {
    const found = await repository.findById("00000000-0000-0000-0000-000000000000");
    expect(found).toBeNull();
  });
});
```

#### Alternativa: `globalSetup` — Um Container para Toda a Execução

Quando a suite de integração tem muitos arquivos e um container por arquivo fica caro, use `globalSetup` do vitest: o container sobe **uma vez** antes de todos os workers e a URI é injetada via `provide`/`inject`:

```typescript
// apps/user/test/integration/global-setup.ts
import type { TestProject } from "vitest/node";
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from "@testcontainers/postgresql";

let container: StartedPostgreSqlContainer;

export default async function setup(project: TestProject) {
  container = await new PostgreSqlContainer("postgres:18-alpine")
    .withTmpFs({ "/var/lib/postgresql/data": "rw" })
    .start();

  // migrations aqui (mesma função runMigrations do testhelper)

  project.provide("databaseUrl", container.getConnectionUri());

  // teardown: executado após todos os testes
  return async () => {
    await container.stop();
  };
}

declare module "vitest" {
  export interface ProvidedContext {
    databaseUrl: string;
  }
}
```

```typescript
// vitest.integration.config.ts
export default defineConfig({
  test: {
    include: ["test/integration/**/*.test.ts"],
    globalSetup: ["./test/integration/global-setup.ts"],
    hookTimeout: 120_000,
    testTimeout: 30_000,
  },
});
```

```typescript
// no teste
import { inject } from "vitest";
const pool = new Pool({ connectionString: inject("databaseUrl"), max: 5 });
```

**Atenção:** com um único container compartilhado entre workers paralelos, testes de arquivos diferentes **compartilham o mesmo banco**. Use um banco/schema por arquivo, desabilite paralelismo de arquivos (`fileParallelism: false`) ou mantenha o padrão de container-por-worker do testhelper. Regra prática: **testhelper singleton por worker é o default; globalSetup só quando o custo de N containers doer**.

---

### 5. Wait Strategies

**Wait strategies são críticas para testes confiáveis.** Elas garantem que o container esteja realmente pronto antes dos testes rodarem — essencial em CI, onde o timing varia.

**Boas práticas:**
- O default é `Wait.forListeningPorts()` — suficiente para a maioria dos serviços
- Use `Wait.forLogMessage()` quando o serviço abre a porta antes de estar pronto
- **Nunca use `setTimeout`/sleep** — anti-padrão que causa flakiness
- Timeout de startup default: **60 segundos**; ajuste com `.withStartupTimeout(ms)` para CI lento

#### Por Porta (default)

```typescript
import { GenericContainer, Wait } from "testcontainers";

await new GenericContainer("redis:7-alpine")
  .withExposedPorts(6379)
  .withWaitStrategy(Wait.forListeningPorts())
  .withStartupTimeout(60_000)
  .start();
```

#### Por Mensagem de Log

```typescript
// String exata
Wait.forLogMessage("Ready to accept connections");

// Regex
Wait.forLogMessage(/Server is ready/);

// N ocorrências — ex.: PostgreSQL loga "ready to accept connections" 2x
// (uma no boot temporário do initdb, outra no boot definitivo)
Wait.forLogMessage(/database system is ready to accept connections/, 2);
```

#### Por Health Check

```typescript
// Usa o HEALTHCHECK da imagem
await new GenericContainer("myapp:latest")
  .withWaitStrategy(Wait.forHealthCheck())
  .start();

// Ou define um health check próprio
await new GenericContainer("myapp:latest")
  .withHealthCheck({
    test: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
    interval: 1_000,
    timeout: 3_000,
    retries: 5,
    startPeriod: 1_000,
  })
  .withWaitStrategy(Wait.forHealthCheck())
  .start();
```

#### Por HTTP

```typescript
await new GenericContainer("myapp:latest")
  .withExposedPorts(8080)
  .withWaitStrategy(
    Wait.forHttp("/health", 8080)
      .forStatusCode(200)
      // .forStatusCodeMatching((code) => code < 500)
      // .forResponsePredicate((body) => body.includes("ok"))
      // .withMethod("POST").withHeaders({ "X-Env": "test" })
      // .usingTls().insecureTls()
      .withReadTimeout(5_000),
  )
  .start();
```

#### Por Comando Bem-Sucedido

```typescript
// Aguarda o comando retornar exit code 0 dentro do container
Wait.forSuccessfulCommand("pg_isready -U test");
```

#### Estratégias Compostas

```typescript
await new GenericContainer("myapp:latest")
  .withExposedPorts(8080)
  .withWaitStrategy(
    Wait.forAll([
      Wait.forListeningPorts(),
      Wait.forLogMessage("Application started"),
    ]).withStartupTimeout(90_000),
  )
  .start();
```

> Em Colima/Rancher Desktop há atraso no port-forwarding: combine `Wait.forListeningPorts()` com um check de log/health via `Wait.forAll` para evitar conexões prematuras.

---

### 6. Networking entre Containers

#### Rede Customizada com Aliases

Quando dois containers precisam falar **entre si** (ex.: app containerizado + banco), crie uma `Network` e use **network aliases**. Entre containers, use a **porta interna** — nunca a mapeada.

```typescript
import { GenericContainer, Network, Wait } from "testcontainers";
import { PostgreSqlContainer } from "@testcontainers/postgresql";

// 1. Criar rede
const network = await new Network().start();

// 2. Banco na rede, com alias "database"
const pgContainer = await new PostgreSqlContainer("postgres:18-alpine")
  .withNetwork(network)
  .withNetworkAliases("database")
  .start();

// 3. App na mesma rede — resolve o banco pelo alias e porta INTERNA
const appContainer = await new GenericContainer("myapp:latest")
  .withNetwork(network)
  .withEnvironment({
    DB_HOST: "database", // alias, não localhost
    DB_PORT: "5432",     // porta interna, não a mapeada
    DB_USER: pgContainer.getUsername(),
    DB_PASSWORD: pgContainer.getPassword(),
    DB_NAME: pgContainer.getDatabase(),
  })
  .withExposedPorts(8080)
  .withWaitStrategy(Wait.forHttp("/health", 8080))
  .start();

// 4. Do HOST, fala-se com o app pela porta mapeada
const baseUrl = `http://${appContainer.getHost()}:${appContainer.getMappedPort(8080)}`;

// 5. Cleanup em ordem inversa; a rede por último
await appContainer.stop();
await pgContainer.stop();
await network.stop();
```

**IP do container em uma rede específica:**

```typescript
const ip = appContainer.getIpAddress(network.getName());
```

#### Expondo Portas do Host para Containers

Para um container acessar um serviço rodando **no host** (ex.: sua API em `pnpm dev`):

```typescript
import { TestContainers } from "testcontainers";

await TestContainers.exposeHostPorts(3000);

// Dentro de qualquer container, o host fica acessível em:
// http://host.testcontainers.internal:3000
```

---

### 7. Migrations dbmate no Container

Nosso padrão: **migrations vivem em `migrations/<app>/` e são aplicadas com dbmate** contra o banco do container **antes** dos testes rodarem. O dbmate roda no host, apontando `DATABASE_URL` para a porta mapeada.

```typescript
import { execFileSync } from "node:child_process";
import path from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export function runMigrations(connectionUri: string, app: string): void {
  const migrationsDir = path.resolve(__dirname, "../../../../../migrations", app);

  execFileSync("dbmate", ["--no-dump-schema", "up"], {
    env: {
      ...process.env,
      // sslmode=disable é obrigatório: o Postgres do container não tem TLS
      DATABASE_URL: `${connectionUri}?sslmode=disable`,
      DBMATE_MIGRATIONS_DIR: migrationsDir,
    },
    stdio: "inherit", // logs do dbmate visíveis no output do teste
  });
}
```

**Pontos de atenção:**
- `--no-dump-schema` evita que o dbmate reescreva `db/schema.sql` durante os testes.
- Chame `runMigrations` **uma vez por container** (dentro do `setupPostgres`), não por teste.
- A tabela `schema_migrations` do dbmate deve ser **excluída do TRUNCATE** (ver `truncateAllTables`).
- Alternativa sem dbmate no PATH do CI: copiar os `.sql` para `/docker-entrypoint-initdb.d/` com `withCopyDirectoriesToContainer` — o entrypoint do Postgres executa tudo no primeiro boot. Funciona, mas não valida o fluxo real de migration (prefira dbmate).

```typescript
// Alternativa: init scripts nativos da imagem postgres
const container = await new PostgreSqlContainer("postgres:18-alpine")
  .withCopyDirectoriesToContainer([
    { source: "./migrations/user", target: "/docker-entrypoint-initdb.d" },
  ])
  .start();
```

---

### 8. Cleanup e Isolamento entre Testes

#### Estratégias de Isolamento — Comparativo

| Estratégia | Velocidade | Isolamento | Quando usar |
|---|---|---|---|
| **TRUNCATE no `beforeEach`** (padrão do projeto) | Muito rápida (~ms) | Dados zerados, sequences resetadas | Default para repositórios |
| **Snapshot/Restore** (`PostgreSqlContainer`) | Rápida | Restaura estado com fixtures pré-carregadas | Seed caro reutilizado em N testes |
| **Banco por suite** (`CREATE DATABASE`) | Média | Schemas independentes no mesmo container | Suites com schemas conflitantes |
| **Container por teste** | Lenta (~segundos) | Total | Testes destrutivos (config do servidor, restart) |

#### TRUNCATE entre Testes (Padrão)

Já implementado no testhelper (`truncateAllTables`). Características:
- `RESTART IDENTITY` zera sequences — IDs previsíveis
- `CASCADE` resolve foreign keys sem ordenar tabelas manualmente
- Exclui `schema_migrations` para não quebrar o dbmate

```typescript
beforeEach(async () => {
  await truncateAllTables(pg.pool);
});
```

#### Snapshot & Restore (PostgreSqlContainer)

Útil quando um seed caro precisa ser restaurado entre testes:

```typescript
const container = await new PostgreSqlContainer("postgres:18-alpine").start();
// ... aplicar migrations + seed pesado ...

await container.snapshot();          // captura o estado atual

// teste 1 modifica dados...
await container.restoreSnapshot();   // volta ao estado do snapshot

// teste 2 começa limpo com o seed intacto
```

**Atenção:** o restore recria o banco — **feche/reinicie as conexões do pool** antes de restaurar, ou o `pg` manterá conexões para um banco que não existe mais.

#### Isolamento no NATS JetStream

Streams e consumers persistem entre testes no mesmo container. Duas abordagens:

```typescript
// 1. Deletar todos os streams entre testes (equivalente ao TRUNCATE)
beforeEach(async () => {
  await deleteAllStreams(nats.jsm);
});

// 2. Nomes únicos por teste (permite paralelismo dentro do arquivo)
const streamName = `EVENTS_USER_${crypto.randomUUID().slice(0, 8)}`;
```

#### Ordem de Cleanup no `afterAll`

Sempre feche os **clients antes dos containers**, e containers antes da network:

```typescript
afterAll(async () => {
  await pool.end();          // 1. clients (pg, nats)
  await nc.drain();
  await appContainer.stop(); // 2. containers dependentes
  await pgContainer.stop();  // 3. containers de infra
  await network.stop();      // 4. network
});
```

#### Cleanup Automático com Ryuk

O testcontainers-node sobe o **Ryuk** (`testcontainers/ryuk`), um garbage collector sidecar que remove containers, networks e volumes **mesmo se o processo de teste crashar** (SIGKILL, timeout do CI, etc.):

- Sobe automaticamente na primeira interação com o Docker
- Monitora a sessão de teste e limpa tudo quando ela termina
- **Não desabilite** (`TESTCONTAINERS_RYUK_DISABLED=true`) exceto em runtimes que exigem (Podman rootless)

```bash
# Diagnóstico do Ryuk
export TESTCONTAINERS_RYUK_VERBOSE=true
# Podman rootful / ambientes que exigem privilégio
export TESTCONTAINERS_RYUK_PRIVILEGED=true
```

---

### 9. NATS JetStream via GenericContainer — Exemplo Completo

Teste de integração de publisher + subscriber contra JetStream real, usando o helper da seção 4:

```typescript
// apps/user/test/integration/user-publisher.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { AckPolicy, DeliverPolicy } from "nats";

import {
  setupNats,
  teardownNats,
  deleteAllStreams,
  type NatsTestContext,
} from "./testhelper/nats.js";
import { UserPublisher } from "../../src/infrastructure/publisher/user-publisher.js";

describe("UserPublisher (integração)", () => {
  let nats: NatsTestContext;

  beforeAll(async () => {
    nats = await setupNats(); // GenericContainer nats:2.12-alpine com -js
  }, 120_000);

  afterAll(async () => {
    await teardownNats();
  });

  beforeEach(async () => {
    await deleteAllStreams(nats.jsm);
    // Recria o stream que o publisher usa
    await nats.jsm.streams.add({
      name: "EVENTS_USER",
      subjects: ["events.user.>"],
    });
  });

  it("publica evento user.created com deduplicação por Nats-Msg-Id", async () => {
    const publisher = new UserPublisher(nats.js, "user-api");
    const user = { id: "u-123", name: "Alice", email: "alice@example.com" };

    await publisher.publishCreated(user);
    await publisher.publishCreated(user); // duplicata — mesmo msgID

    const info = await nats.jsm.streams.info("EVENTS_USER");
    expect(info.state.messages).toBe(1); // dedup server-side funcionou
  });

  it("consumer durável recebe e faz ack do evento", async () => {
    // Arrange: consumer durável explícito
    await nats.jsm.consumers.add("EVENTS_USER", {
      durable_name: "billing-on-user-created",
      filter_subject: "events.user.created",
      ack_policy: AckPolicy.Explicit,
      deliver_policy: DeliverPolicy.All,
    });

    const publisher = new UserPublisher(nats.js, "user-api");
    await publisher.publishCreated({ id: "u-456", name: "Bob", email: "bob@example.com" });

    // Act: pull do consumer
    const consumer = await nats.js.consumers.get("EVENTS_USER", "billing-on-user-created");
    const messages = await consumer.fetch({ max_messages: 1, expires: 5_000 });

    let received = 0;
    for await (const msg of messages) {
      const event = msg.json<{ data: { user_id: string } }>();
      expect(event.data.user_id).toBe("u-456");
      msg.ack();
      received++;
    }

    expect(received).toBe(1);
  });
});
```

**Variações do comando do servidor NATS:**

```typescript
// JetStream com limites e store em tmpfs (mais rápido)
await new GenericContainer("nats:2.12-alpine")
  .withCommand(["-js", "--store_dir", "/data"])
  .withTmpFs({ "/data": "rw" })
  .withExposedPorts(4222)
  .withWaitStrategy(Wait.forLogMessage(/Server is ready/))
  .start();

// Com porta de monitoramento HTTP (útil para debug: /varz, /jsz)
await new GenericContainer("nats:2.12-alpine")
  .withCommand(["-js", "-m", "8222"])
  .withExposedPorts(4222, 8222)
  .withWaitStrategy(Wait.forLogMessage(/Server is ready/))
  .start();
```

---

### 10. Performance

#### Reuso de Containers entre Execuções

`withReuse()` mantém o container vivo após a execução e o reutiliza na próxima, eliminando o boot em runs subsequentes (ótimo para o loop local de desenvolvimento):

```typescript
const container = await new PostgreSqlContainer("postgres:18-alpine")
  .withDatabase("app_test")
  .withReuse() // container sobrevive ao fim dos testes e é reaproveitado
  .start();
```

Regras do reuso:
- O container só é reaproveitado se a **configuração for idêntica** (imagem, env, comando, etc.)
- Habilite globalmente com `TESTCONTAINERS_REUSE_ENABLE=true` (ou desabilite com `false`)
- **Não use em CI** — runners efêmeros não se beneficiam e containers órfãos acumulam
- Com reuso, o estado **persiste entre execuções**: o TRUNCATE do `beforeEach` se torna obrigatório

#### Imagens Pequenas e tmpfs

```typescript
// alpine: pull e boot mais rápidos
new PostgreSqlContainer("postgres:18-alpine");
new GenericContainer("nats:2.12-alpine");

// tmpfs: dados em memória — elimina I/O de disco (Postgres fica ~2-3x mais rápido)
.withTmpFs({ "/var/lib/postgresql/data": "rw" })
```

#### Paralelismo do vitest e Singletons

- `pool: "forks"` (default): **1 processo por arquivo** → singleton do testhelper = 1 container por arquivo. Arquivos rodam em paralelo com containers independentes. Simples e isolado; custo: N containers.
- `maxWorkers`/`minWorkers`: limite o paralelismo se a máquina não aguentar N Postgres simultâneos (`maxWorkers: 2`).
- `fileParallelism: false`: arquivos em série — combine com `globalSetup` para **1 container no run inteiro** (mais lento no total, menos recursos).
- **Nunca** compartilhe um container mutável entre arquivos paralelos sem isolamento por banco/schema/stream.

```typescript
// vitest.integration.config.ts — exemplo balanceado
export default defineConfig({
  test: {
    include: ["test/integration/**/*.test.ts"],
    hookTimeout: 120_000,
    testTimeout: 30_000,
    pool: "forks",
    maxWorkers: 2, // no máximo 2 arquivos (= 2 stacks de containers) em paralelo
  },
});
```

#### Outras Dicas

- **Pin de versão da imagem** (`postgres:18-alpine`, nunca `latest`): evita pulls surpresa e mantém reprodutibilidade
- Pré-pull das imagens no CI (`docker pull postgres:18-alpine`) em um step cacheável
- Agrupe testes que compartilham a mesma infra no mesmo arquivo — 1 boot amortizado em N testes
- `Pool` do `pg` com `max: 5` nos testes — evita esgotar `max_connections` com múltiplos workers

---

### 11. Configuração do Ambiente

#### Variáveis de Ambiente

```bash
# Docker daemon customizado
export DOCKER_HOST=tcp://docker:2375
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/certs

# Testcontainers
export TESTCONTAINERS_HOST_OVERRIDE=<host onde as portas são expostas>
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
export TESTCONTAINERS_RYUK_DISABLED=true        # desabilita Ryuk (evitar!)
export TESTCONTAINERS_RYUK_PRIVILEGED=true      # Ryuk privilegiado (Podman rootful)
export TESTCONTAINERS_RYUK_VERBOSE=true
export TESTCONTAINERS_HUB_IMAGE_NAME_PREFIX=registry.empresa.com/  # registry espelho
export TESTCONTAINERS_REUSE_ENABLE=true         # habilita withReuse globalmente

# Autenticação em registries privados (usa ~/.docker/config.json por padrão)
export DOCKER_CONFIG=/path/to/dir-with-config-json
export DOCKER_AUTH_CONFIG='{"auths": {"registry.empresa.com": {"auth": "..."}}}'
```

#### Logs de Debug da Biblioteca

```bash
DEBUG=testcontainers* pnpm test:integration       # tudo
DEBUG=testcontainers:containers pnpm test:integration  # logs dos containers
DEBUG=testcontainers:pull pnpm test:integration        # progresso de pull
DEBUG=testcontainers,testcontainers:exec pnpm test:integration  # combinações
```

#### Colima (macOS sem Docker Desktop)

```bash
export DOCKER_HOST=unix://${HOME}/.colima/default/docker.sock
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
# Colima resolve IPv6 mal — prefira IPv4:
export NODE_OPTIONS=--dns-result-order=ipv4first
```

> Colima e Rancher Desktop têm atraso no port-forwarding: use `Wait.forAll([Wait.forListeningPorts(), Wait.forLogMessage(...)])` para evitar conexões prematuras.

#### Podman

```bash
# macOS
export DOCKER_HOST=unix://$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}')
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
# rootless em macOS: desabilitar o Ryuk
export TESTCONTAINERS_RYUK_DISABLED=true

# Linux (rootless)
systemctl --user enable --now podman.socket
export DOCKER_HOST=unix://$(podman info --format '{{.Host.RemoteSocket.Path}}')
```

#### Testcontainers Cloud

Alternativa gerenciada ao Docker local: os containers rodam em workers na nuvem, sem Docker instalado na máquina/CI. Basta instalar o agente (`testcontainers-cloud`) e autenticar — a biblioteca detecta o endpoint automaticamente via arquivo `~/.testcontainers.properties`. Nenhuma mudança de código é necessária. Detalhes: https://testcontainers.com/cloud/

---

### 12. Troubleshooting

#### Container não fica pronto (timeout de startup)

```typescript
// 1. Aumente o timeout — CI costuma ser mais lento que a máquina local
.withStartupTimeout(120_000)

// 2. Veja o que o container está logando durante o boot
.withLogConsumer((stream) => {
  stream.on("data", (line) => console.log("[container]", line));
})

// 3. Confirme que a wait strategy bate com a realidade do serviço
//    (a mensagem de log mudou entre versões da imagem? a porta é outra?)
```

#### `ECONNREFUSED` ao conectar no serviço

- Você usou a porta **interna** em vez de `getMappedPort()`? Do host, sempre `getHost()` + `getMappedPort()`.
- Entre containers, o oposto: alias da network + porta **interna**.
- Em Colima/Rancher: port-forwarding atrasado — use wait strategy composta (seção 5).
- IPv6: `export NODE_OPTIONS=--dns-result-order=ipv4first`.

#### Hook `beforeAll` estoura timeout

- Configure `hookTimeout: 120_000` no vitest config **ou** passe o timeout como segundo argumento: `beforeAll(async () => {...}, 120_000)`.
- Primeira execução inclui o `docker pull` da imagem — pré-pull no CI resolve.

#### Containers órfãos acumulando

```bash
# O Ryuk deve estar rodando durante os testes:
docker ps | grep ryuk

# Se você usou withReuse(), os containers ficam vivos POR DESIGN. Limpeza manual:
docker ps -a --filter "label=org.testcontainers" -q | xargs docker rm -f
```

- Verifique se `TESTCONTAINERS_RYUK_DISABLED` não está setado sem necessidade.
- Garanta `await container.stop()` no `afterAll` mesmo com Ryuk (limpeza determinística > garbage collection).

#### Falha de pull de imagem

```bash
# Valide manualmente
docker pull postgres:18-alpine

# Registry privado: login antes (testcontainers usa ~/.docker/config.json)
docker login registry.empresa.com

# Espelho corporativo do Docker Hub
export TESTCONTAINERS_HUB_IMAGE_NAME_PREFIX=registry.empresa.com/
```

#### Pool do `pg` trava no fim da suite (`vitest` não termina)

- `await pool.end()` no `afterAll` — conexões abertas seguram o event loop.
- No NATS, `await nc.drain()` (ou `nc.close()`); subscriptions ativas também seguram o processo.
- Diagnóstico: `vitest run --reporter=verbose` + flag `--no-file-parallelism` para reproduzir isolado.

#### Erros de tipo/ESM ao importar

- `testcontainers` e módulos são TypeScript-first e funcionam com `"module": "nodenext"`.
- `pg` é CommonJS: com `esModuleInterop: true`, `import { Pool } from "pg"` funciona; caso contrário, `import pg from "pg"; const { Pool } = pg;`.
- Imports relativos em ESM exigem extensão `.js` (mesmo em arquivos `.ts`): `./testhelper/postgres.js`.

---

## Exemplos Completos

### Exemplo 1: Repositório PostgreSQL com `pg` (fluxo completo)

```typescript
// apps/user/test/integration/user-repository.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";

import {
  setupPostgres,
  teardownPostgres,
  truncateAllTables,
  type PostgresTestContext,
} from "./testhelper/postgres.js";
import { UserRepository } from "../../src/infrastructure/repository/user-repository.js";
import { newUser } from "../../src/domain/entity/user.js";

describe("UserRepository (integração)", () => {
  let pg: PostgresTestContext;
  let repository: UserRepository;

  beforeAll(async () => {
    pg = await setupPostgres();
    repository = new UserRepository(pg.pool);
  }, 120_000);

  afterAll(async () => {
    await teardownPostgres();
  });

  beforeEach(async () => {
    await truncateAllTables(pg.pool);
  });

  it("create persiste e retorna id gerado", async () => {
    const user = newUser("Alice", "alice@example.com");

    const created = await repository.create(user);

    expect(created.id).toMatch(/^[0-9a-f-]{36}$/);
    expect(created.createdAt).toBeInstanceOf(Date);
  });

  it("findByEmail retorna null para email inexistente", async () => {
    const found = await repository.findByEmail("ninguem@example.com");
    expect(found).toBeNull();
  });

  it("list pagina por cursor", async () => {
    for (let i = 0; i < 5; i++) {
      await repository.create(newUser(`User ${i}`, `user${i}@example.com`));
    }

    const page1 = await repository.list({ limit: 2 });
    expect(page1.users).toHaveLength(2);
    expect(page1.nextCursor).toBeTruthy();

    const page2 = await repository.list({ cursor: page1.nextCursor, limit: 2 });
    expect(page2.users).toHaveLength(2);
    expect(page2.users[0].id).not.toBe(page1.users[0].id);
  });

  it("constraint de email único gera erro de conflito", async () => {
    await repository.create(newUser("Alice", "alice@example.com"));

    await expect(
      repository.create(newUser("Alice 2", "alice@example.com")),
    ).rejects.toThrow(/duplicate key|already exists/);
  });
});
```

### Exemplo 2: Fluxo E2E — Publisher grava no banco e publica no JetStream

```typescript
// apps/user/test/integration/create-user-flow.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { AckPolicy } from "nats";

import { setupPostgres, teardownPostgres, truncateAllTables } from "./testhelper/postgres.js";
import { setupNats, teardownNats, deleteAllStreams } from "./testhelper/nats.js";
import { UserRepository } from "../../src/infrastructure/repository/user-repository.js";
import { UserPublisher } from "../../src/infrastructure/publisher/user-publisher.js";
import { CreateUserUseCase } from "../../src/application/usecase/user/create-usecase.js";

describe("Fluxo de criação de usuário (integração)", () => {
  let pg: Awaited<ReturnType<typeof setupPostgres>>;
  let nats: Awaited<ReturnType<typeof setupNats>>;
  let useCase: CreateUserUseCase;

  beforeAll(async () => {
    // Containers sobem em paralelo — economiza tempo de boot
    [pg, nats] = await Promise.all([setupPostgres(), setupNats()]);

    useCase = new CreateUserUseCase(
      new UserRepository(pg.pool),
      new UserPublisher(nats.js, "user-api"),
    );
  }, 120_000);

  afterAll(async () => {
    await Promise.all([teardownPostgres(), teardownNats()]);
  });

  beforeEach(async () => {
    await truncateAllTables(pg.pool);
    await deleteAllStreams(nats.jsm);
    await nats.jsm.streams.add({ name: "EVENTS_USER", subjects: ["events.user.>"] });
  });

  it("persiste no Postgres e publica events.user.created no JetStream", async () => {
    // Act
    const output = await useCase.perform({ name: "Alice", email: "alice@example.com" });

    // Assert 1: persistido no banco real
    const { rows } = await pg.pool.query('SELECT "email" FROM "users" WHERE "id" = $1', [
      output.user.id,
    ]);
    expect(rows[0].email).toBe("alice@example.com");

    // Assert 2: evento no stream real
    await nats.jsm.consumers.add("EVENTS_USER", {
      durable_name: "assert-consumer",
      ack_policy: AckPolicy.Explicit,
    });
    const consumer = await nats.js.consumers.get("EVENTS_USER", "assert-consumer");
    const batch = await consumer.fetch({ max_messages: 1, expires: 5_000 });

    for await (const msg of batch) {
      expect(msg.subject).toBe("events.user.created");
      const event = msg.json<{ data: { user_id: string; email: string } }>();
      expect(event.data.user_id).toBe(output.user.id);
      msg.ack();
    }
  });
});
```

### Exemplo 3: App Containerizado + Banco na Mesma Network

```typescript
// apps/user/test/integration/api-e2e.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { GenericContainer, Network, Wait, type StartedTestContainer, type StartedNetwork } from "testcontainers";
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from "@testcontainers/postgresql";

describe("API E2E (imagem Docker do app)", () => {
  let network: StartedNetwork;
  let pgContainer: StartedPostgreSqlContainer;
  let appContainer: StartedTestContainer;
  let baseUrl: string;

  beforeAll(async () => {
    network = await new Network().start();

    pgContainer = await new PostgreSqlContainer("postgres:18-alpine")
      .withNetwork(network)
      .withNetworkAliases("database")
      .start();

    appContainer = await new GenericContainer("user-api:test")
      .withNetwork(network)
      .withEnvironment({
        DB_HOST: "database",
        DB_PORT: "5432",
        DB_USER: pgContainer.getUsername(),
        DB_PASSWORD: pgContainer.getPassword(),
        DB_NAME: pgContainer.getDatabase(),
      })
      .withExposedPorts(8080)
      .withWaitStrategy(Wait.forHttp("/health", 8080).forStatusCode(200))
      .start();

    baseUrl = `http://${appContainer.getHost()}:${appContainer.getMappedPort(8080)}`;
  }, 180_000);

  afterAll(async () => {
    await appContainer.stop();
    await pgContainer.stop();
    await network.stop();
  });

  it("POST /v1/users cria usuário via HTTP real", async () => {
    const response = await fetch(`${baseUrl}/v1/users`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name: "Alice", email: "alice@example.com" }),
    });

    expect(response.status).toBe(201);
    const body = await response.json();
    expect(body.id).toBeTruthy();
  });
});
```

### Exemplo 4: Docker Compose

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import {
  DockerComposeEnvironment,
  Wait,
  type StartedDockerComposeEnvironment,
} from "testcontainers";

describe("Stack via docker-compose", () => {
  let environment: StartedDockerComposeEnvironment;

  beforeAll(async () => {
    environment = await new DockerComposeEnvironment(".", "docker-compose.test.yml")
      // nome do container segue o padrão <service>-1
      .withWaitStrategy("postgres-1", Wait.forLogMessage(/ready to accept connections/, 2))
      .withWaitStrategy("nats-1", Wait.forLogMessage(/Server is ready/))
      .withEnvironment({ TAG: "test" })
      // .withBuild()                    // builda imagens do compose
      // .up(["postgres", "nats"])       // sobe só serviços específicos
      .up();
  }, 180_000);

  afterAll(async () => {
    await environment.down({ timeout: 10_000 });
  });

  it("acessa serviços do compose", async () => {
    const pgContainer = environment.getContainer("postgres-1");
    const host = pgContainer.getHost();
    const port = pgContainer.getMappedPort(5432);

    expect(host).toBeTruthy();
    expect(port).toBeGreaterThan(0);
  });
});
```

> Prefira containers programáticos (módulos/GenericContainer) a compose nos testes — configuração fica no código, tipada e por suite. Use `DockerComposeEnvironment` quando o compose file já é a fonte de verdade do ambiente (ex.: validar o próprio `docker-compose.yml`).

---

## Boas Práticas

1. **Sempre use módulos pré-configurados quando existirem** (`@testcontainers/postgresql`, `@testcontainers/nats`) — defaults sensatos e helpers de conexão
2. **Centralize o boot em `test/integration/testhelper/`** (`postgres.ts`, `nats.ts`) — 1 container por suite/worker via singleton de módulo, nunca boot inline repetido
3. **Configure `hookTimeout` alto** (120s) no vitest config de integração — boot de container + pull de imagem não cabem nos 10s default
4. **Wait strategy correta sempre** — `forListeningPorts()` como base; `forLogMessage()`/`forHealthCheck()` quando o serviço engana; **nunca sleep**
5. **`getHost()` + `getMappedPort()` do host; alias + porta interna entre containers** — decorar essa regra elimina 90% dos `ECONNREFUSED`
6. **Migrations dbmate uma vez por container**, dentro do `setupPostgres` — teste valida o fluxo real de schema
7. **TRUNCATE no `beforeEach`** (excluindo `schema_migrations`) como isolamento padrão; snapshot/restore para seeds caros
8. **Feche clients antes de containers** no `afterAll` (`pool.end()`, `nc.drain()`) — evita processos pendurados
9. **Pin de versão de imagem** (`postgres:18-alpine`, `nats:2.12-alpine`) — nunca `latest`
10. **`withTmpFs` para dados de banco** — I/O em memória acelera a suite significativamente
11. **`withReuse()` apenas no loop local** — nunca em CI; com reuso, o cleanup entre testes vira obrigação
12. **Limite `maxWorkers`** quando cada arquivo sobe sua própria stack de containers
13. **Debug com `DEBUG=testcontainers*` e `withLogConsumer`** — antes de chutar, leia o log do container

---

## Recursos Adicionais

- **Documentação oficial**: https://node.testcontainers.org/
- **Lista de módulos**: https://testcontainers.com/modules/?language=nodejs
- **Repositório GitHub**: https://github.com/testcontainers/testcontainers-node (exemplos em `packages/modules/*/src/*.test.ts`)
- **Módulo PostgreSQL**: https://node.testcontainers.org/modules/postgresql/
- **Módulo NATS**: https://node.testcontainers.org/modules/nats/
- **Wait strategies**: https://node.testcontainers.org/features/wait-strategies/
- **Slack da comunidade**: https://testcontainers.slack.com
