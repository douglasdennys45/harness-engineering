---
name: testcontainers-rust
description: Use esta skill ao escrever testes de integracao Rust com containers Docker usando o crate testcontainers 0.24 (async, Tokio) e testcontainers-modules 0.12 (features postgres, nats) — modulos pre-configurados (Postgres, Nats), GenericImage para imagens sem modulo, wait strategies (WaitFor::message_on_stdout, healthcheck, porta), ContainerAsync + .start().await, obtencao de host/porta mapeada (get_host / get_host_port_ipv4) e networking entre containers. Cobre o ciclo de vida com #[tokio::test], containers compartilhados via tokio::sync::OnceCell, aplicacao de migrations sqlx no container, padroes de cleanup/isolamento (TRUNCATE vs banco por teste) e integracao com sqlx (PgPool) e async-nats (JetStream), espelhando os helpers de apps/<app>/tests/testhelper/ (postgres.rs, nats.rs). Acione ao criar/editar testes em apps/<app>/tests/, configurar infraestrutura de teste baseada em containers ou depurar containers de teste.
keywords: [rust, testing, docker, integration-tests, testcontainers, sqlx, tokio]
license: MIT
---

# Testcontainers para Testes de Integracao em Rust

Guia para usar o crate **testcontainers** (0.24) com **testcontainers-modules** (0.12) para escrever testes de integracao confiaveis com containers Docker em projetos Rust + Tokio.

## Descricao

Testcontainers fornece instancias leves e descartaveis de bancos, message brokers ou qualquer servico Docker — controladas a partir do proprio codigo de teste, com ciclo de vida atrelado ao Rust (o container para quando o `ContainerAsync` e dropado).

**Capacidades principais:**
- Usar modulos pre-configurados (`testcontainers_modules::postgres::Postgres`, `nats::Nats`)
- `GenericImage` para imagens sem modulo, com wait strategy explicita
- Subir e gerenciar containers dentro de testes `#[tokio::test]` (API async)
- Obter host/porta mapeada (`get_host()`, `get_host_port_ipv4()`)
- Compartilhar containers entre testes via `tokio::sync::OnceCell` (helpers em `tests/testhelper/`)
- Aplicar migrations `sqlx` no container antes dos testes
- Implementar cleanup e isolamento corretos (TRUNCATE vs banco por teste)
- Integrar com `sqlx` (`PgPool`) e `async-nats` (JetStream)

## Quando Usar

- Testar repositorios `sqlx` contra um PostgreSQL 18 real, sem mocks
- Testar publishers/subscribers NATS JetStream contra um broker real
- Criar ambientes de teste reproduziveis, sem infra local pre-instalada
- E2E do fluxo completo (API Axum + banco/NATS reais por tras)

**Convencoes do projeto (ver `.claude/rules/architecture.md`):**
- Testes de integracao vivem em `apps/<app>/tests/` (crate de teste separado).
- Helpers compartilhados em `apps/<app>/tests/testhelper/` (`postgres.rs`, `nats.rs`).
- Runner: **`#[tokio::test]`** (multi-thread). Banco: **PostgreSQL 18** via `sqlx`. Mensageria: **NATS JetStream 2.12** via `async-nats`. Migrations: **sqlx-cli** (`migrations/<app>/*.up.sql` / `.down.sql`).
- Mocks unitarios usam `mockall` no proprio arquivo; **integracao usa containers reais**.

## Pre-requisitos

- **Docker** (ou Podman/Colima) instalado e rodando
- **Rust edition 2024** (MSRV 1.85) e **Tokio** (`tokio = { version = "1", features = ["full"] }`)
- Crates (versoes conforme o `Cargo.toml` da raiz):

```toml
[workspace.dependencies]
testcontainers = "0.24"
testcontainers-modules = { version = "0.12", features = ["postgres", "nats"] }
```

## 1. Instalacao & Setup

No `Cargo.toml` do app, adicione os crates de teste em `[dev-dependencies]`:

```toml
# apps/user/Cargo.toml
[dev-dependencies]
testcontainers = { workspace = true }
testcontainers-modules = { workspace = true }
tokio = { workspace = true }
sqlx = { workspace = true }
async-nats = { workspace = true }
anyhow = { workspace = true }
```

**Verificar Docker** antes de rodar a suite:

```bash
docker info > /dev/null 2>&1 || echo "Docker indisponivel — testes de integracao serao pulados"
cargo test -p user --test '*'   # roda apenas os testes de integracao (tests/)
```

O `testcontainers` sobe automaticamente o **Ryuk** (garbage collector) que remove containers mesmo se o processo de teste crashar. Nao desabilite (`TESTCONTAINERS_RYUK_DISABLED`) exceto em Podman rootless.

## 2. PostgreSQL — Modulo Pre-Configurado

O modulo `Postgres` traz defaults sensatos (db/user/password `postgres`), a wait strategy correta embutida (`ready_conditions`) e integra com o `AsyncRunner`. O container e mantido vivo dentro de um `tokio::sync::OnceCell` **estatico** para ser compartilhado por toda a suite — o `ContainerAsync` **nao pode** ser dropado, senao o container para.

```rust
// apps/user/tests/testhelper/postgres.rs
use std::time::Duration;

use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;
use testcontainers::ContainerAsync;
use testcontainers_modules::postgres::Postgres;
use testcontainers_modules::testcontainers::runners::AsyncRunner;
use tokio::sync::OnceCell;

/// Container + pool compartilhados por toda a suite (1 boot amortizado em N testes).
pub struct SharedPg {
    _container: ContainerAsync<Postgres>, // mantem o container vivo; drop => container para
    pub pool: PgPool,
}

static PG: OnceCell<SharedPg> = OnceCell::const_new();

/// Retorna o pool compartilhado, ja com migrations aplicadas.
/// Chame `truncate_all` no inicio de cada teste para isolamento.
pub async fn pool() -> &'static PgPool {
    let shared = PG
        .get_or_init(|| async {
            let container = Postgres::default()
                .with_db_name("app_test")
                .with_user("app")
                .with_password("secret")
                .start()
                .await
                .expect("start postgres container");

            // host = "localhost" (ou IP do docker host); porta = mapeada aleatoria
            let host = container.get_host().await.expect("get host");
            let port = container
                .get_host_port_ipv4(5432)
                .await
                .expect("get mapped port");

            let url = format!("postgres://app:secret@{host}:{port}/app_test");

            let pool = PgPoolOptions::new()
                .max_connections(5) // baixo nos testes — evita esgotar max_connections
                .acquire_timeout(Duration::from_secs(5))
                .connect(&url)
                .await
                .expect("connect pool");

            run_migrations(&pool).await;

            SharedPg { _container: container, pool }
        })
        .await;

    &shared.pool
}

/// Aplica as migrations sqlx uma unica vez por container (embutidas em compile time).
/// O path e relativo ao Cargo.toml do app.
async fn run_migrations(pool: &PgPool) {
    sqlx::migrate!("../../migrations/user")
        .run(pool)
        .await
        .expect("run migrations");
}

/// Isolamento entre testes: limpa todas as tabelas, exceto _sqlx_migrations.
pub async fn truncate_all(pool: &PgPool) {
    let rows: Vec<(String,)> = sqlx::query_as(
        "SELECT tablename FROM pg_tables \
         WHERE schemaname = 'public' AND tablename <> '_sqlx_migrations'",
    )
    .fetch_all(pool)
    .await
    .expect("list tables");

    if rows.is_empty() {
        return;
    }

    let tables = rows
        .iter()
        .map(|(t,)| format!("\"{t}\""))
        .collect::<Vec<_>>()
        .join(", ");

    sqlx::query(&format!("TRUNCATE TABLE {tables} RESTART IDENTITY CASCADE"))
        .execute(pool)
        .await
        .expect("truncate tables");
}
```

**Metodos uteis do `Postgres` (builder) e do `ContainerAsync`:**

```rust
Postgres::default()
    .with_db_name("app_test")   // POSTGRES_DB
    .with_user("app")           // POSTGRES_USER
    .with_password("secret")    // POSTGRES_PASSWORD
    .with_init_sql(             // SQL rodado no boot (alternativa a migrations)
        "CREATE EXTENSION IF NOT EXISTS pgcrypto;".to_string().into_bytes(),
    );

let host = container.get_host().await?;              // Host (localhost / IP)
let port = container.get_host_port_ipv4(5432).await?; // u16 — porta mapeada aleatoria
```

> A porta **interna** e sempre `5432`; do host, use SEMPRE `get_host_port_ipv4(5432)` (mapeada). Usar a porta interna direto do host causa `Connection refused`.

## 3. NATS — Modulo Pre-Configurado

O modulo `Nats` sobe o broker; para JetStream, passe `NatsServerCmd::default().with_jetstream()` (equivale ao `-js`). Mesmo padrao de `OnceCell` compartilhado.

```rust
// apps/user/tests/testhelper/nats.rs
use async_nats::jetstream::{self, Context};
use testcontainers::ContainerAsync;
use testcontainers_modules::nats::{Nats, NatsServerCmd};
use testcontainers_modules::testcontainers::runners::AsyncRunner;
use tokio::sync::OnceCell;

pub struct SharedNats {
    _container: ContainerAsync<Nats>, // mantem o container vivo
    pub client: async_nats::Client,
}

static NATS: OnceCell<SharedNats> = OnceCell::const_new();

/// Retorna um client NATS conectado ao container compartilhado (JetStream habilitado).
pub async fn client() -> &'static async_nats::Client {
    let shared = NATS
        .get_or_init(|| async {
            let cmd = NatsServerCmd::default().with_jetstream();
            let container = Nats::default()
                .with_cmd(&cmd)
                .start()
                .await
                .expect("start nats container");

            let host = container.get_host().await.expect("get host");
            let port = container
                .get_host_port_ipv4(4222)
                .await
                .expect("get mapped port");

            let client = async_nats::connect(format!("nats://{host}:{port}"))
                .await
                .expect("connect nats");

            SharedNats { _container: container, client }
        })
        .await;

    &shared.client
}

/// Context de JetStream a partir do client compartilhado.
pub async fn jetstream() -> Context {
    jetstream::new(client().await.clone())
}

/// Isolamento entre testes: remove todos os streams antes de cada teste.
pub async fn delete_all_streams(js: &Context) {
    use futures::StreamExt;

    let mut names = js.stream_names();
    while let Some(name) = names.next().await {
        if let Ok(name) = name {
            let _ = js.delete_stream(name).await;
        }
    }
}
```

Registre os helpers no `apps/user/tests/testhelper/mod.rs`:

```rust
// apps/user/tests/testhelper/mod.rs
pub mod nats;
pub mod postgres;
```

## 4. GenericImage — Imagens Sem Modulo

Quando nao existe modulo pre-configurado, use `GenericImage` + `ImageExt`. **SEMPRE** adicione uma wait strategy explicita ao expor portas — nunca confie em `sleep`.

```rust
use std::time::Duration;

use testcontainers::core::{IntoContainerPort, WaitFor};
use testcontainers::runners::AsyncRunner;
use testcontainers::{GenericImage, ImageExt};

// Ex.: NATS "na mao" (equivalente ao modulo, sem testcontainers-modules)
let container = GenericImage::new("nats", "2.12-alpine")
    .with_exposed_port(4222.tcp())                              // 4222/tcp
    .with_wait_for(WaitFor::message_on_stderr("Server is ready")) // wait por log
    .with_cmd(["-js"])                                          // habilita JetStream
    .with_startup_timeout(Duration::from_secs(30))
    .start()
    .await?;

let host = container.get_host().await?;
let port = container.get_host_port_ipv4(4222).await?;
let url = format!("nats://{host}:{port}");
```

**Ordem importa:** chame primeiro os metodos do `GenericImage` (`with_exposed_port`, `with_wait_for`), depois os de `ImageExt` (`with_cmd`, `with_env_var`, `with_network`, ...).

**Opcoes comuns do `ImageExt`:**

```rust
GenericImage::new("myimage", "1.0")
    .with_exposed_port(8080.tcp())
    .with_wait_for(WaitFor::http(/* HttpWaitStrategy */))
    .with_env_var("APP_ENV", "test")
    .with_network("app-net")            // rede compartilhada entre containers
    .with_mapped_port(0, 8080.tcp())    // 0 = porta de host aleatoria (padrao)
    .start()
    .await?;
```

## 5. Ciclo de Vida com `#[tokio::test]`

- **Container**: 1 por suite via `OnceCell` estatico (amortiza o boot). O `ContainerAsync` fica preso dentro do `OnceCell` durante todo o processo de teste; o Ryuk faz a limpeza ao final.
- **Pool/limpeza**: reaproveita o pool compartilhado, mas roda `truncate_all` / `delete_all_streams` no **inicio de cada teste** para isolamento.
- **Nunca** dropar o `ContainerAsync` no meio da suite — isso para o container.

```rust
// apps/user/tests/user_repository_integration_test.rs
mod testhelper;

use testhelper::postgres;
use user::infrastructure::repository::PostgresUserRepository;
use user::domain::entity::User;
use user::domain::repository::UserRepository;

#[tokio::test]
async fn create_and_find_user() {
    let pool = postgres::pool().await;      // container compartilhado (boot 1x)
    postgres::truncate_all(pool).await;     // isolamento antes do teste

    let repo = PostgresUserRepository::new(pool.clone());

    let mut user = User::new("Ana".into(), "ana@example.com".into());
    repo.create(&mut user).await.expect("create");
    assert_ne!(user.id, uuid::Uuid::nil());

    let found = repo.find_by_id(user.id).await.expect("find").expect("some");
    assert_eq!(found.email, "ana@example.com");
}
```

## 6. Wait Strategies

Wait strategies garantem que o container esteja **pronto** antes do teste rodar — critico em CI, onde o timing varia.

| Estrategia | API | Quando usar |
|---|---|---|
| Log (stdout) | `WaitFor::message_on_stdout("...")` | Servicos que logam prontidao no stdout |
| Log (stderr) | `WaitFor::message_on_stderr("Server is ready")` | NATS e afins (logam no stderr) |
| Healthcheck | `WaitFor::healthcheck()` | Imagens com `HEALTHCHECK` no Dockerfile |
| HTTP | `WaitFor::http(HttpWaitStrategy::new("/health"))` | APIs com endpoint de health |
| Duracao (evitar) | `WaitFor::seconds(n)` | Ultimo recurso; **prefira log/porta** |

```rust
use testcontainers::core::WaitFor;

// Postgres/Nats via modulo ja embutem a wait strategy correta — nao redefina.
// Para GenericImage, escolha a estrategia adequada ao servico:
let img = GenericImage::new("elasticsearch", "8.7.0")
    .with_exposed_port(9200.tcp())
    .with_wait_for(WaitFor::message_on_stdout("started"));
```

- **Nunca** use `std::thread::sleep` / `tokio::time::sleep` como wait — e anti-pattern que gera testes flaky.
- Ajuste `with_startup_timeout(Duration::from_secs(60))` em CI lento.

## 7. Migrations sqlx no Container

As migrations vivem em `migrations/<app>/` (pares `.up.sql` / `.down.sql`, sqlx-cli). Aplique-as no container **uma vez por boot**, dentro do `OnceCell`.

**Opcao A — macro `sqlx::migrate!` (embute em compile time):**

```rust
// path relativo ao Cargo.toml do app (apps/user -> ../../migrations/user)
sqlx::migrate!("../../migrations/user")
    .run(pool)
    .await
    .expect("run migrations");
```

**Opcao B — `Migrator` em runtime (path resolvido em tempo de execucao):**

```rust
use std::path::Path;
use sqlx::migrate::Migrator;

let migrator = Migrator::new(Path::new("../../migrations/user"))
    .await
    .expect("load migrations");
migrator.run(pool).await.expect("run migrations");
```

- Rode **uma vez por container** (dentro do `get_or_init`), nunca por teste.
- O sqlx cria a tabela de controle **`_sqlx_migrations`** — **exclua-a do TRUNCATE**.
- Prefira a macro (`sqlx::migrate!`) — valida o path em compile time e embute os arquivos no binario de teste.

## 8. Cleanup e Isolamento

| Estrategia | Velocidade | Quando usar |
|---|---|---|
| **TRUNCATE por teste** (padrao) | ~ms | Default para repositorios |
| **Banco por modulo/arquivo** (`CREATE DATABASE`) | Media | Suites com schemas conflitantes |
| **Container por teste** | Lenta | Testes destrutivos (config do servidor) |

TRUNCATE (padrao): `TRUNCATE TABLE ... RESTART IDENTITY CASCADE`, excluindo `_sqlx_migrations` (ver `truncate_all` na §2). Chame no **inicio** de cada teste.

NATS: `delete_all_streams` entre testes, ou use nomes de stream unicos por teste (ver §3).

**Container por teste** (isolamento maximo — cada teste sobe o seu, sem `OnceCell`):

```rust
#[tokio::test]
async fn destructive_test() {
    let container = Postgres::default().start().await.expect("start");
    // ... container vive ate o fim do teste; drop no fim => para o container
    let port = container.get_host_port_ipv4(5432).await.expect("port");
    // ...
}
```

### Ryuk (garbage collector)

O `testcontainers` sobe o **Ryuk** como sidecar — remove containers mesmo se o processo de teste crashar ou for morto. Confie nele para a limpeza final; nao desabilite (`TESTCONTAINERS_RYUK_DISABLED=true`) exceto em Podman rootless.

## 9. Networking entre Containers

Para que containers conversem entre si (ex.: app -> banco), coloque-os na **mesma rede** e use o **alias de rede + porta interna** (nao a porta mapeada do host):

```rust
use testcontainers::runners::AsyncRunner;
use testcontainers::{GenericImage, ImageExt};
use testcontainers_modules::postgres::Postgres;

const NET: &str = "it-net";

// banco na rede, com nome de container previsivel
let pg = Postgres::default()
    .with_network(NET)
    .with_container_name("it-postgres")
    .start()
    .await?;

// app na mesma rede — alcanca o banco por "it-postgres:5432" (porta INTERNA)
let app = GenericImage::new("myapp", "latest")
    .with_exposed_port(8080.tcp())
    .with_wait_for(WaitFor::http(/* /health */))
    .with_network(NET)
    .with_env_var("DB_HOST", "it-postgres")  // alias de rede, nao localhost
    .with_env_var("DB_PORT", "5432")         // porta interna, nao a mapeada
    .start()
    .await?;

// do HOST, para bater na API, use a porta MAPEADA:
let host = app.get_host().await?;
let app_port = app.get_host_port_ipv4(8080).await?;
```

Regra de ouro: **entre containers** use alias + porta interna; **do host** use `get_host()` + `get_host_port_ipv4()`.

## 10. Exemplo Completo — Repositorio Postgres + Pool + Migrations

```rust
// apps/user/tests/user_repository_integration_test.rs
mod testhelper;

use testhelper::postgres;
use user::domain::entity::User;
use user::domain::repository::UserRepository;
use user::infrastructure::repository::PostgresUserRepository;

#[tokio::test]
async fn create_persists_and_generates_id() {
    let pool = postgres::pool().await;
    postgres::truncate_all(pool).await;

    let repo = PostgresUserRepository::new(pool.clone());

    let mut user = User::new("Ana".into(), "ana@example.com".into());
    repo.create(&mut user).await.expect("create user");

    assert_ne!(user.id, uuid::Uuid::nil(), "id preenchido via RETURNING");
}

#[tokio::test]
async fn find_by_email_returns_none_when_absent() {
    let pool = postgres::pool().await;
    postgres::truncate_all(pool).await;

    let repo = PostgresUserRepository::new(pool.clone());

    let found = repo.find_by_email("missing@example.com").await.expect("query");
    assert!(found.is_none(), "ausencia = Ok(None), nunca erro");
}

#[tokio::test]
async fn list_paginates_by_cursor() {
    let pool = postgres::pool().await;
    postgres::truncate_all(pool).await;

    let repo = PostgresUserRepository::new(pool.clone());
    for i in 0..3 {
        let mut u = User::new(format!("u{i}"), format!("u{i}@example.com"));
        repo.create(&mut u).await.expect("create");
    }

    let (page, next) = repo.list(None, 2).await.expect("list");
    assert_eq!(page.len(), 2);
    assert!(next.is_some(), "ha mais paginas");
}
```

## 11. Exemplo Completo — NATS + JetStream (Publish/Consume)

```rust
// apps/user/tests/user_publisher_integration_test.rs
mod testhelper;

use std::time::Duration;

use async_nats::jetstream::consumer::PullConsumer;
use futures::StreamExt;
use testhelper::nats;

#[tokio::test]
async fn publish_and_consume_event() {
    let js = nats::jetstream().await;
    nats::delete_all_streams(&js).await; // isolamento

    // 1. cria stream para o subject
    js.create_stream(async_nats::jetstream::stream::Config {
        name: "EVENTS_USER".to_string(),
        subjects: vec!["events.user.>".to_string()],
        ..Default::default()
    })
    .await
    .expect("create stream");

    // 2. publica um evento (aguarda o ack do JetStream)
    js.publish("events.user.created", "{\"userId\":\"1\"}".into())
        .await
        .expect("publish")
        .await
        .expect("await ack");

    // 3. consumidor pull duravel
    let consumer: PullConsumer = js
        .get_stream("EVENTS_USER")
        .await
        .expect("get stream")
        .create_consumer(async_nats::jetstream::consumer::pull::Config {
            durable_name: Some("test-consumer".to_string()),
            filter_subject: "events.user.created".to_string(),
            ..Default::default()
        })
        .await
        .expect("create consumer");

    // 4. le e faz ack
    let mut messages = consumer
        .fetch()
        .max_messages(1)
        .messages()
        .await
        .expect("fetch");

    let msg = tokio::time::timeout(Duration::from_secs(5), messages.next())
        .await
        .expect("timeout")
        .expect("stream ended")
        .expect("message");

    assert_eq!(msg.subject.as_str(), "events.user.created");
    msg.ack().await.expect("ack");
}
```

## 12. Performance

- **Imagens com tag fixa e alpine quando possivel** (`postgres:18-alpine`, `nats:2.12-alpine`) — boot rapido; nunca `latest` (o modulo ja fixa uma tag conhecida).
- **Container por suite** (`OnceCell` estatico), nao por teste — 1 boot amortizado em N testes.
- Pool `sqlx` com `max_connections(5)` nos testes — evita esgotar `max_connections` do Postgres.
- `cargo test` roda testes do mesmo binario em **threads paralelas**: containers compartilhados via `OnceCell` sao seguros (o `OnceCell` serializa a init), mas o **isolamento entre testes paralelos** exige cuidado — prefira dados/streams com nomes unicos por teste, ou rode com `--test-threads=1` quando o TRUNCATE global colidir.
- Pre-pull das imagens no CI (`docker pull postgres:18-alpine`) num step cacheavel.

## 13. Troubleshooting

- **`Connection refused` do host**: usou a porta **interna** (5432/4222) em vez da mapeada. Do host, sempre `get_host()` + `get_host_port_ipv4(porta_interna)`.
- **Container some no meio da suite**: o `ContainerAsync` foi dropado. Guarde-o no `OnceCell` estatico (campo `_container`), nao em variavel local de escopo curto.
- **`sqlx::migrate!` nao encontra o path**: o path e relativo ao **Cargo.toml do app**, nao ao arquivo de teste. De `apps/user`, use `../../migrations/user`.
- **Testes paralelos interferindo**: TRUNCATE global limpa dados de outro teste em execucao. Use nomes unicos por teste ou `cargo test -- --test-threads=1` na suite de integracao.
- **Timeout no boot**: aumente `with_startup_timeout(Duration::from_secs(60))`; CI e mais lento. Pre-pull da imagem ajuda.
- **Docker indisponivel**: o `.start().await` retorna erro de conexao com o daemon. Cheque `docker info` e o socket (`DOCKER_HOST`).
- **Debug**: `RUST_LOG=testcontainers=debug cargo test` mostra o ciclo de vida; adicione `.with_log_consumer(...)` para inspecionar logs do container.

## Boas Praticas

1. **Modulo pre-configurado quando existir** (`Postgres`, `Nats`) — defaults e wait strategy corretos; `GenericImage` so como fallback.
2. **Centralize o boot em `tests/testhelper/`** (`postgres.rs`, `nats.rs`) com `tokio::sync::OnceCell`.
3. **Container por suite, TRUNCATE/limpeza por teste** — isolamento sem re-boot.
4. **Guarde o `ContainerAsync` vivo** no `OnceCell` (campo `_container`) — o drop para o container.
5. **Wait strategy explicita** em `GenericImage` (`WaitFor::message_on_*`, `healthcheck`, `http`); nunca `sleep`.
6. **`get_host()` + `get_host_port_ipv4()` do host**; alias de rede + porta interna entre containers.
7. **Migrations sqlx uma vez por container** (`sqlx::migrate!`), dentro do `get_or_init`.
8. **TRUNCATE excluindo `_sqlx_migrations`**; NATS: `delete_all_streams` entre testes.
9. **Pin de tag de imagem** — nunca `latest`; prefira alpine.
10. **Confie no Ryuk** para a limpeza final; nao desabilite salvo Podman rootless.

## Fontes

- https://docs.rs/testcontainers/0.24.0/testcontainers/ — crate principal (`GenericImage`, `ImageExt`, `ContainerAsync`, `runners::AsyncRunner`)
- https://docs.rs/testcontainers/0.24.0/testcontainers/core/ — `WaitFor`, `IntoContainerPort`, wait strategies
- https://docs.rs/testcontainers-modules/0.12.1/testcontainers_modules/ — modulos (`postgres::Postgres`, `nats::Nats`, `nats::NatsServerCmd`)
- https://docs.rs/testcontainers-modules/0.12.1/testcontainers_modules/postgres/ — modulo Postgres
- https://docs.rs/testcontainers-modules/0.12.1/testcontainers_modules/nats/ — modulo NATS (JetStream via `with_jetstream`)
- https://testcontainers.com/ — visao geral, catalogo de modulos e conceitos
- https://github.com/testcontainers/testcontainers-rs — repositorio do testcontainers-rs
- https://github.com/testcontainers/testcontainers-rs-modules-community — repositorio dos modulos
- https://docs.rs/sqlx/latest/sqlx/macro.migrate.html — macro `sqlx::migrate!` e `Migrator`
- https://docs.rs/async-nats/latest/async_nats/jetstream/ — JetStream (streams, consumers, ack)
