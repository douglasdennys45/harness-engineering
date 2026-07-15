# Arquitetura da Aplicacao

Este documento define a arquitetura do projeto, suas camadas, regras de dependencia, convencoes de nomenclatura e o guia passo a passo para adicionar novas features.

## Visao Geral

O projeto segue **Clean Architecture** com inversao de dependencia em uma estrutura de **monorepo** com multiplos microservicos. Camadas internas nunca importam camadas externas. Injecao de dependencia e feita por **composition root manual** (construcao explicita no `main` de cada binario, com `Arc<dyn Trait>`). O monorepo suporta **N apps independentes** (APIs e Consumers) que compartilham bibliotecas via `pkg/`.

```
apps/<app-name>/<binary>.rs     Entry points (um por binario)
       |
       v
apps/<app-name>/domain/             Entidades, traits (sem dependencias de infraestrutura)
apps/<app-name>/application/        Implementacoes de use cases (depende apenas de domain)
apps/<app-name>/infrastructure/     Controllers, repos, publisher, subscriber, config
       |
       v
pkg/                                    Crate reutilizavel compartilhado entre apps
```

### Stack Tecnologica

| Camada | Tecnologia | Versao |
|---|---|---|
| Linguagem | Rust | **edition 2024** (MSRV 1.85) |
| Runtime async | Tokio | `tokio = "1"` (multi-thread) |
| HTTP | Axum | `axum = "0.8"` |
| Banco de Dados | PostgreSQL | **18** |
| Acesso a dados | sqlx (SQL manual, sem ORM) | `sqlx = "0.8"` |
| Mensageria | NATS JetStream | **2.12** (`async-nats`) |
| DI / Lifecycle | Composition root manual + `Arc<dyn Trait>` | - |
| Logging | tracing (JSON estruturado) | `tracing` + `tracing-subscriber` |
| Documentacao | OpenAPI (utoipa + utoipa-swagger-ui) | latest |
| Validacao | validator (`#[derive(Validate)]`) | `validator = "0.20"` |
| Config | dotenvy + variaveis de ambiente | - |
| Erros | thiserror (dominio) + anyhow (propagacao) | latest |
| Testes | testcontainers, mockall, #[tokio::test] | latest |

### Tipos de Binarios

| Tipo | Proposito | Exemplo |
|---|---|---|
| **API** | Servidor HTTP expondo endpoints REST (Axum) | `apps/user/api.rs` |
| **Consumer** | Processo em background consumindo eventos NATS JetStream | `apps/user/consumer.rs` |

### Regra de Dependencia

```
Domain  <--  Application  <--  Infrastructure  <--  api.rs / consumer.rs
  ^                                  |
  |                                  v
  +--- nunca depende de -------->  pkg/
```

- `domain/` importa apenas std e crates de tipos/contratos: `serde`, `chrono`, `uuid`, `thiserror`, `anyhow`, `async-trait`. Nunca `axum`, `sqlx`, `async-nats` ou `tower`.
- `application/` importa apenas `crate::domain`.
- `infrastructure/` importa `crate::domain`, `pkg` e crates externas (axum, sqlx, async-nats).
- `pkg/` nunca importa nada dos apps.
- `<binary>.rs` importa tudo para compor o grafo de dependencias **daquele binario especifico** (composition root).

---

## Estrutura de Diretorios

```
/
├── Cargo.toml                                       # Workspace (members: pkg + apps)
├── Cargo.lock
├── docker-compose.yml
├── Makefile
│
├── pkg/                                             # crate: pkg (compartilhado)
│   ├── Cargo.toml
│   ├── lib.rs                                   # pub mod postgres; nats; httpserver; controller; logger; event;
│   ├── postgres/
│   │   ├── mod.rs
│   │   ├── conn.rs                              # new_pool + pool config
│   │   ├── tx.rs                                # helpers de transaction
│   │   └── health.rs                            # ping health check
│   ├── nats/
│   │   ├── mod.rs
│   │   ├── conn.rs                              # connect + reconnect
│   │   ├── stream.rs                            # ensure_stream
│   │   ├── publisher.rs                         # Publisher generico
│   │   └── subscriber.rs                        # subscribe com consumer duravel
│   ├── httpserver/
│   │   ├── mod.rs
│   │   └── axum.rs                              # with_default_middlewares (pilha tower-http)
│   ├── controller/
│   │   ├── mod.rs
│   │   ├── context.rs                           # RequestContext extractor
│   │   ├── validated.rs                         # ValidatedJson<T> (bind + validacao)
│   │   └── error.rs                             # ApiError + HttpDomainError + ErrorResponse
│   ├── logger/
│   │   └── mod.rs                               # setup tracing JSON com service e version
│   └── event/                                   # contratos de eventos compartilhados
│       ├── mod.rs
│       ├── envelope.rs                          # Envelope base
│       ├── user.rs                              # subjects + data structs
│       └── billing.rs                           # subjects + data structs
│
├── migrations/                                      # migrations SQL por app (sqlx-cli)
│   ├── user/
│   │   ├── 000001_create_users.up.sql
│   │   └── 000001_create_users.down.sql
│   └── billing/
│       ├── 000001_create_accounts.up.sql
│       └── 000001_create_accounts.down.sql
│
├── apps/
│   ├── user/                                        # crate: user
│   │   ├── Cargo.toml
│   │   ├── Dockerfile
│   │   ├── lib.rs                                   # pub mod domain; application; infrastructure;
│   │   ├── api.rs                                   # API HTTP — composition root
│   │   ├── consumer.rs                              # Consumer NATS — composition root
│   │   ├── domain/
│   │   │   ├── mod.rs
│   │   │   ├── entity/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── user.rs                          # Struct de dominio + construtor
│   │   │   │   └── errors.rs                        # DomainError (thiserror)
│   │   │   ├── usecase/
│   │   │   │   ├── mod.rs
│   │   │   │   └── user/                            # Submodulo por contexto de dominio
│   │   │   │       ├── mod.rs
│   │   │   │       ├── create.rs                    # trait CreateUseCase
│   │   │   │       ├── get_by_id.rs                 # trait GetByIdUseCase
│   │   │   │       └── list.rs                      # trait ListUseCase
│   │   │   ├── repository/
│   │   │   │   ├── mod.rs
│   │   │   │   └── user_repository.rs               # Trait de repositorio
│   │   │   ├── event/
│   │   │   │   ├── mod.rs
│   │   │   │   └── user_event.rs                    # Trait de eventos/mensageria
│   │   │   └── <lib>/                               # Traits para libs externas
│   │   │       └── <name>.rs                        # Trait de inversao de dependencia
│   │   ├── application/
│   │   │   ├── mod.rs
│   │   │   └── usecase/
│   │   │       ├── mod.rs
│   │   │       └── user/                            # Submodulo por contexto de dominio
│   │   │           ├── mod.rs
│   │   │           ├── create_usecase.rs            # Impl do use case (+ mod tests no proprio arquivo)
│   │   │           ├── get_by_id_usecase.rs
│   │   │           └── list_usecase.rs
│   │   ├── infrastructure/
│   │   │   ├── mod.rs
│   │   │   ├── config/
│   │   │   │   ├── mod.rs
│   │   │   │   └── config.rs                        # AppConfig + load
│   │   │   ├── controller/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── errors.rs                        # impl HttpDomainError p/ DomainError DESTE app
│   │   │   │   └── user_controller.rs               # Controllers HTTP (Axum) — apenas APIs
│   │   │   ├── repository/
│   │   │   │   ├── mod.rs
│   │   │   │   └── user_repository.rs               # Implementacao PostgreSQL (sqlx)
│   │   │   ├── publisher/
│   │   │   │   ├── mod.rs
│   │   │   │   └── user_publisher.rs                # Implementacao NATS JetStream
│   │   │   ├── subscriber/
│   │   │   │   ├── mod.rs
│   │   │   │   └── user_subscriber.rs               # Consumer NATS — apenas Consumers
│   │   │   └── adapter/
│   │   │       ├── mod.rs
│   │   │       └── <lib>_adapter.rs                 # Implementacoes de traits de libs externas
│   │   └── tests/                                   # testes de integracao (crate separado)
│   │       ├── testhelper/
│   │       │   ├── mod.rs
│   │       │   ├── postgres.rs                      # Container PG compartilhado (OnceCell)
│   │       │   └── nats.rs                          # Container NATS compartilhado (OnceCell)
│   │       └── user_integration_test.rs             # Teste E2E do fluxo completo
│   │
│   └── billing/                                     # mesma estrutura interna
│       ├── Cargo.toml
│       ├── Dockerfile
│       ├── lib.rs
│       ├── api.rs
│       ├── consumer.rs
│       ├── domain/
│       ├── application/
│       ├── infrastructure/
│       └── tests/
```

### Cargo Workspace (`Cargo.toml` na raiz)

```toml
[workspace]
resolver = "3"
members = ["pkg", "apps/user", "apps/billing"]

[workspace.dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["trace", "cors", "compression-full", "request-id", "timeout", "catch-panic", "util"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "uuid", "chrono", "migrate"] }
async-nats = "0.38"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
validator = { version = "0.20", features = ["derive"] }
utoipa = { version = "5", features = ["axum_extras", "uuid", "chrono"] }
utoipa-swagger-ui = { version = "9", features = ["axum"] }
thiserror = "2"
anyhow = "1"
async-trait = "0.1"
uuid = { version = "1", features = ["v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
dotenvy = "0.15"
futures = "0.3"
mockall = "0.13"
testcontainers = "0.24"
testcontainers-modules = { version = "0.12", features = ["postgres", "nats"] }
```

Cada app importa `pkg` no seu `Cargo.toml` e declara seus binarios:

```toml
# apps/user/Cargo.toml
[package]
name = "user"
version = "0.1.0"
edition = "2024"

# Fontes na raiz do crate (sem pasta `src/`): o `[lib]` aponta para `lib.rs`
# e cada `[[bin]]` para `<binary>.rs`, espelhando o crate `pkg`.
[lib]
path = "lib.rs"

[dependencies]
pkg = { path = "../../pkg" }
axum = { workspace = true }
tokio = { workspace = true }
# ... demais dependencias via { workspace = true }

[[bin]]
name = "api"
path = "api.rs"

[[bin]]
name = "consumer"
path = "consumer.rs"
```

### Convencao de Nomes para `apps/` e binarios

- **Apps**: `apps/<app-name>/` -- lowercase (ex: `apps/user/`, `apps/billing/`).
- **APIs**: `apps/<app>/api.rs`.
- **Consumers**: `apps/<app>/consumer.rs`. Se houver multiplos consumers, usar `consumer_<dominio>.rs` (nome do binario `consumer-<dominio>` no `[[bin]]`).
- Cada binario (`api.rs`, `consumer.rs`) vive na raiz do crate e contem **apenas** o composition root (`main`) daquele binario. Nenhuma logica de negocio.
- Modulos usam estilo `mod.rs`: 1 diretorio = 1 modulo, espelhando a organizacao por camada.

---

## Camada de Dominio (`domain/`)

A camada de dominio contem **entidades** e **contratos** (traits). Nao possui dependencias de infraestrutura — apenas std e crates de tipos/contratos (`serde`, `chrono`, `uuid`, `thiserror`, `anyhow`, `async-trait`). **Isolada dentro de cada app.**

### entity/

Structs representando conceitos de dominio. Cada entidade possui um construtor `new()`.

**Padrao:**

```rust
// apps/user/domain/entity/user.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct User {
    pub id: Uuid,
    pub name: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl User {
    pub fn new(name: String, email: String) -> Self {
        let now = Utc::now();
        Self {
            id: Uuid::nil(),   // preenchido pelo banco via RETURNING id
            name,
            email,
            created_at: now,
            updated_at: now,
        }
    }
}
```

**Regras:**
- Arquivo: `<name>.rs` em snake_case se composto (ex: `audit_log.rs`).
- Struct exportada com `#[serde(rename_all = "camelCase")]` (JSON em camelCase).
- Construtor `new(...)` retorna `Self`.
- Timestamps sempre em UTC: `Utc::now()` (chrono).
- `id: Uuid::nil()` ate a persistencia devolver o ID gerado (`RETURNING id`).
- Sem logica de infraestrutura (sem imports de axum, sqlx, async-nats ou tower).

### Erros de Dominio (`entity/errors.rs`)

Cada app define seu enum de erros de dominio com **thiserror**. A variante `Unexpected` faz a ponte com erros de infraestrutura propagados via **anyhow**.

```rust
// apps/user/domain/entity/errors.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("invalid input")]
    InvalidInput,
    #[error("user not found")]
    UserNotFound,
    #[error("email already in use")]
    UserAlreadyExists,
    #[error("invalid role")]
    InvalidRole,
    #[error("forbidden")]
    Forbidden,
    #[error(transparent)]
    Unexpected(#[from] anyhow::Error),
}

// Erros de infraestrutura sobem com contexto e viram Unexpected:
//   repo.create(&mut user).await.context("create user")?;
//   matches!(err, DomainError::UserAlreadyExists) == true para erros de negocio
```

### usecase/

Traits definindo os use cases de dominio. Cada use case e representado por **um unico trait com um unico metodo `perform`**. Traits sao organizados em **submodulos por contexto de dominio**.

**Principio:** 1 acao = 1 trait = 1 arquivo. Isso garante contratos coesos, facilita testes (mocks granulares com mockall) e respeita o Interface Segregation Principle.

**Estrutura de diretorios:**

```
domain/usecase/
└── user/
    ├── mod.rs                # pub mod create; get_by_id; list; + re-exports
    ├── create.rs             # trait CreateUseCase
    ├── get_by_id.rs          # trait GetByIdUseCase
    └── list.rs               # trait ListUseCase
```

**Padrao:**

```rust
// apps/user/domain/usecase/user/create.rs
use async_trait::async_trait;

use crate::domain::entity::{DomainError, User};

pub struct CreateInput {
    pub name: String,
    pub email: String,
}

pub struct CreateOutput {
    pub user: User,
}

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait CreateUseCase: Send + Sync {
    async fn perform(&self, input: CreateInput) -> Result<CreateOutput, DomainError>;
}
```

```rust
// apps/user/domain/usecase/user/get_by_id.rs
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::{DomainError, User};

pub struct GetByIdInput {
    pub id: Uuid,
}

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait GetByIdUseCase: Send + Sync {
    async fn perform(&self, input: GetByIdInput) -> Result<Option<User>, DomainError>;
}
```

```rust
// apps/user/domain/usecase/user/list.rs
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::{DomainError, User};

pub struct ListInput {
    pub cursor: Option<Uuid>,
    pub limit: i64,
}

pub struct ListOutput {
    pub users: Vec<User>,
    pub next_cursor: Option<Uuid>,
}

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait ListUseCase: Send + Sync {
    async fn perform(&self, input: ListInput) -> Result<ListOutput, DomainError>;
}
```

**Regras:**
- Organizacao: `domain/usecase/<contexto>/<acao>.rs` (ex: `domain/usecase/user/create.rs`).
- Cada arquivo contem **1 trait** com **1 metodo**: `perform`.
- Nome do modulo: o contexto de dominio em **lowercase** (ex: `mod user`).
- Trait exportado: `<Action>UseCase` (ex: `CreateUseCase`, `GetByIdUseCase`).
- Trait com bound `Send + Sync` e `#[async_trait]` (necessario para `Arc<dyn Trait>`).
- `#[cfg_attr(test, mockall::automock)]` SEMPRE antes de `#[async_trait]` (gera `Mock<Nome>` para testes).
- Structs de input (`<Action>Input`) e output (`<Action>Output`) sao definidas no mesmo arquivo do trait.
- Retorna entidades de dominio, output structs e/ou `DomainError`.

### repository/

Traits definindo contratos de persistencia. Metodos retornam `anyhow::Result` (erro de infraestrutura propagado com contexto; a decisao de negocio fica no use case).

**Padrao:**

```rust
// apps/user/domain/repository/user_repository.rs
use anyhow::Result;
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::User;

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn create(&self, user: &mut User) -> Result<()>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>>;
    async fn find_by_email(&self, email: &str) -> Result<Option<User>>;
    async fn list(&self, cursor: Option<Uuid>, limit: i64) -> Result<(Vec<User>, Option<Uuid>)>;
    async fn update(&self, user: &User) -> Result<()>;
    async fn delete(&self, id: Uuid) -> Result<()>;
}
```

**Regras:**
- Arquivo: `<name>_repository.rs` (ex: `user_repository.rs`).
- Trait exportado: `<Name>Repository`, com `Send + Sync`.
- `find_by_id` retorna `Result<Option<Entity>>` — ausencia e `Ok(None)`, nunca erro.
- `create` recebe `&mut Entity` para preencher o ID gerado pelo banco.

### event/

Traits definindo contratos de mensageria/eventos.

**Padrao:**

```rust
// apps/user/domain/event/user_event.rs
use anyhow::Result;
use async_trait::async_trait;

use crate::domain::entity::User;

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait UserEvent: Send + Sync {
    async fn publish_created(&self, user: &User) -> Result<()>;
    async fn publish_updated(&self, user: &User) -> Result<()>;
}
```

**Regras:**
- Arquivo: `<name>_event.rs` (ex: `user_event.rs`).
- Trait exportado: `<Name>Event`, com `Send + Sync`.
- Metodos nomeados por acao: `publish_<action>`.

### Traits para Bibliotecas Externas (`domain/<lib>/`)

Toda biblioteca externa utilizada no projeto **deve ser acessada via inversao de dependencia**. O trait e definido na camada de dominio em um submodulo nomeado pelo conceito/capacidade que a lib fornece. A implementacao concreta fica em `infrastructure/adapter/`.

**Principio:** O dominio nunca conhece a lib concreta. Ele define **o que precisa** (trait), e a infraestrutura fornece **como fazer** (implementacao com a lib).

**Estrutura de diretorios:**

```
domain/
├── crypt/
│   └── hasher.rs              # Trait para hashing de senhas
├── token/
│   └── manager.rs             # Trait para geracao/validacao de tokens
└── mailer/
    └── sender.rs              # Trait para envio de emails
```

**Padrao:**

```rust
// apps/user/domain/crypt/hasher.rs
use anyhow::Result;

#[cfg_attr(test, mockall::automock)]
pub trait Hasher: Send + Sync {
    fn hash(&self, plain: &str) -> Result<String>;
    fn compare(&self, hashed: &str, plain: &str) -> Result<()>;
}
```

**Regras:**
- Submodulo nomeado pelo **conceito/capacidade**, nao pelo nome da lib (ex: `crypt/`, nao `argon2/`; `token/`, nao `jwt/`).
- Trait exportado com nome descritivo da capacidade (ex: `Hasher`, `Manager`, `Sender`).
- **Nenhum import de lib externa** -- apenas std e crates de contrato.
- Metodos sincronos quando a operacao e CPU-bound (hash); `#[async_trait]` quando e I/O-bound (mailer).
- As implementacoes concretas ficam em `infrastructure/adapter/`.

---

## Camada de Aplicacao (`application/`)

Contem a **implementacao** dos use cases. Orquestra entidades, repositorios e eventos. **Isolada dentro de cada app.**

### usecase/

Implementacoes dos use cases, organizadas em **submodulos por contexto de dominio**, espelhando a estrutura de `domain/usecase/`.

**Estrutura de diretorios:**

```
application/usecase/
└── user/
    ├── mod.rs
    ├── create_usecase.rs           # struct CreateUseCase + perform (+ mod tests no mesmo arquivo)
    ├── get_by_id_usecase.rs
    └── list_usecase.rs
```

**Padrao:**

```rust
// apps/user/application/usecase/user/create_usecase.rs
use std::sync::Arc;

use anyhow::Context;
use async_trait::async_trait;

use crate::domain::entity::{DomainError, User};
use crate::domain::event::UserEvent;
use crate::domain::repository::UserRepository;
use crate::domain::usecase::user::{CreateInput, CreateOutput};

pub struct CreateUseCase {
    user_repository: Arc<dyn UserRepository>,
    user_event: Arc<dyn UserEvent>,
}

impl CreateUseCase {
    pub fn new(
        user_repository: Arc<dyn UserRepository>,
        user_event: Arc<dyn UserEvent>,
    ) -> Self {
        Self { user_repository, user_event }
    }
}

#[async_trait]
impl crate::domain::usecase::user::CreateUseCase for CreateUseCase {
    async fn perform(&self, input: CreateInput) -> Result<CreateOutput, DomainError> {
        let existing = self
            .user_repository
            .find_by_email(&input.email)
            .await
            .context("find by email")?;
        if existing.is_some() {
            return Err(DomainError::UserAlreadyExists);
        }

        let mut user = User::new(input.name, input.email);

        self.user_repository
            .create(&mut user)
            .await
            .context("create user")?;

        // publica evento DEPOIS do commit no banco
        let _ = self.user_event.publish_created(&user).await;

        Ok(CreateOutput { user })
    }
}
```

**Regras:**
- Organizacao: `application/usecase/<contexto>/<acao>_usecase.rs`.
- Nome do modulo: o contexto de dominio em **lowercase** (ex: `mod user`).
- Struct: `<Action>UseCase` (mesmo nome do trait de dominio — modulos distintos, sem sufixo `Impl`).
- Construtor: `new(deps...) -> Self`; o **composition root** embrulha em `Arc<dyn Trait>`.
- Campos da struct sao **traits de dominio** via `Arc<dyn Trait>` (nunca tipos concretos).
- O metodo implementado e sempre `perform`, via `#[async_trait] impl crate::domain::usecase::<contexto>::<Action>UseCase for <Action>UseCase`.
- Depende apenas de `crate::domain`. Nunca importa `infrastructure/` ou `pkg`.
- Erros de infraestrutura propagados com contexto: `.context("acao")?` (vira `DomainError::Unexpected`).
- Erros de negocio retornados como variantes explicitas: `Err(DomainError::UserAlreadyExists)`.
- Evento publicado **apos** persistencia no banco, nunca antes.
- Teste unitario no proprio arquivo, em `#[cfg(test)] mod tests`, usando os mocks gerados pelo mockall.

---

## Camada de Infraestrutura (`infrastructure/`)

Implementa contratos de dominio usando tecnologias concretas e gerencia a configuracao da aplicacao. Cada binario conecta apenas o que precisa.

### controller/ (apenas APIs)

Controllers HTTP usando Axum. O projeto utiliza a infraestrutura generica de `pkg/controller/`, que substitui o "template method" classico por **extractors** e **conversao de erros** do Axum: extracao de dados comuns da request (`RequestContext`), bind + validacao do body (`ValidatedJson<T>`) e tratamento uniforme de erros (`ApiError` + `HttpDomainError`). Cada controller so implementa a logica especifica. **Usado apenas por binarios de API.**

#### Infra generica — `pkg/controller/` (compartilhada entre todos os apps)

O `pkg/controller/` vive no `pkg/` porque e infraestrutura generica **sem nenhuma logica de dominio**. Todos os apps reutilizam a mesma base, evitando duplicacao.

Ele centraliza:
- Extracao automatica de `request_id`, `parent_id`, `ip`, `user_agent` (extractor `RequestContext`).
- Bind + validacao do body com resposta padronizada (extractor `ValidatedJson<T>`).
- Tratamento uniforme de erros (dominio -> HTTP status via trait `HttpDomainError`).
- Logging estruturado de erros (5xx como `error`, 4xx como `warn`) no `IntoResponse` do `ApiError`.
- Resposta de erro padronizada (`ErrorResponse`).

**Estrutura do `pkg/controller/`:**

```
pkg/
└── controller/
    ├── mod.rs                  # re-exports (ApiError, RequestContext, ValidatedJson, ...)
    ├── context.rs              # RequestContext extractor (FromRequestParts)
    ├── validated.rs            # ValidatedJson<T> (FromRequest: parse + validacao)
    └── error.rs                # ApiError, HttpDomainError, ErrorResponse
```

**Estrutura de cada app (so o codigo especifico):**

```
apps/user/infrastructure/controller/
├── mod.rs
├── errors.rs                   # impl HttpDomainError p/ o DomainError DESTE app
├── user_controller.rs          # UserController (usa a infra do pkg)
└── order_controller.rs         # OrderController (usa a infra do pkg)
```

**`pkg/controller/context.rs` — RequestContext:**

```rust
// pkg/controller/context.rs
use axum::{extract::FromRequestParts, http::request::Parts};

/// RequestContext contem dados extraidos automaticamente de toda request.
/// Handlers declaram este extractor e recebem o struct ja preenchido,
/// sem precisar extrair manualmente.
#[derive(Clone, Debug)]
pub struct RequestContext {
    pub request_id: String,
    pub parent_id: String,
    pub ip: String,
    pub user_agent: String,
}

impl<S: Send + Sync> FromRequestParts<S> for RequestContext {
    type Rejection = std::convert::Infallible;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let header = |name: &str| {
            parts
                .headers
                .get(name)
                .and_then(|v| v.to_str().ok())
                .unwrap_or_default()
                .to_string()
        };

        Ok(Self {
            request_id: header("x-request-id"),   // populado pelo SetRequestIdLayer
            parent_id: header("x-parent-id"),
            ip: header("x-forwarded-for"),
            user_agent: header("user-agent"),
        })
    }
}
```

(No Axum 0.8, `FromRequestParts`/`FromRequest` usam async fn nativo — sem `#[async_trait]`.)

**`pkg/controller/error.rs` — ApiError + HttpDomainError (mapeamento de erros):**

```rust
// pkg/controller/error.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;
use utoipa::ToSchema;

/// ErrorResponse e a struct padrao de resposta de erro da API.
#[derive(Debug, Serialize, ToSchema)]
pub struct ErrorResponse {
    pub error: String,
    pub status: u16,
}

/// Trait que cada app implementa no seu enum de erros de dominio,
/// mapeando cada variante para um StatusCode.
/// O pkg/controller nao conhece nenhum erro de dominio — ele so recebe o mapeamento.
pub trait HttpDomainError: std::error::Error {
    fn status_code(&self) -> StatusCode;
}

/// Erro HTTP final. Handlers retornam Result<_, ApiError>;
/// o `?` converte DomainError -> ApiError automaticamente via HttpDomainError.
#[derive(Debug)]
pub struct ApiError {
    pub status: StatusCode,
    pub message: String,
}

impl ApiError {
    pub fn bad_request(message: impl Into<String>) -> Self {
        Self { status: StatusCode::BAD_REQUEST, message: message.into() }
    }
}

impl<E: HttpDomainError> From<E> for ApiError {
    fn from(err: E) -> Self {
        let status = err.status_code();
        let message = if status.is_server_error() {
            "internal server error".to_string()   // nunca vazar detalhes de 5xx
        } else {
            err.to_string()
        };
        Self { status, message }
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        if self.status.is_server_error() {
            tracing::error!(status_code = self.status.as_u16(), error = %self.message, "request failed");
        } else {
            tracing::warn!(status_code = self.status.as_u16(), error = %self.message, "request rejected");
        }

        (
            self.status,
            Json(ErrorResponse { error: self.message, status: self.status.as_u16() }),
        )
            .into_response()
    }
}
```

**`pkg/controller/validated.rs` — ValidatedJson (bind + validacao automatica):**

O projeto utiliza o crate **`validator`** como engine de validacao. O extractor `ValidatedJson<T>` executa parse do JSON e validacao das tags `#[validate]` automaticamente — o handler so e executado se o body for valido.

```rust
// pkg/controller/validated.rs
use axum::{
    extract::{FromRequest, Request},
    Json,
};
use validator::{Validate, ValidationErrors};

use super::error::ApiError;

/// ValidatedJson faz bind + validacao do body automaticamente antes do handler.
/// Uso: async fn create(ValidatedJson(input): ValidatedJson<CreateUserInput>) -> ...
pub struct ValidatedJson<T>(pub T);

impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + Validate,
{
    type Rejection = ApiError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|err| ApiError::bad_request(err.body_text()))?;   // erro de parse -> 400

        value
            .validate()
            .map_err(|err| ApiError::bad_request(format_validation_errors(&err)))?;   // validacao -> 400

        Ok(ValidatedJson(value))
    }
}

/// Converte os erros do validator em uma mensagem legivel
/// ("validation failed: <campo> <regra>; ...") usando o nome do campo JSON.
fn format_validation_errors(errors: &ValidationErrors) -> String {
    let mut msgs: Vec<String> = Vec::new();
    for (field, field_errors) in errors.field_errors() {
        for e in field_errors {
            let msg = match e.code.as_ref() {
                "required" => format!("{field} is required"),
                "email" => format!("{field} must be a valid email"),
                "length" => format!("{field} has invalid length"),
                "range" => format!("{field} is out of range"),
                "url" => format!("{field} must be a valid url"),
                _ => format!("{field} failed on {} validation", e.code),
            };
            msgs.push(msg);
        }
    }
    format!("validation failed: {}", msgs.join("; "))
}
```

**Pilha padrao de middlewares (`pkg/httpserver/axum.rs`):**

IMPORTANTE (armadilha do Axum): `.layer()` so envolve rotas **ja adicionadas** ao `Router`. Por isso o `pkg` expoe uma funcao que **aplica** os middlewares em cima do Router pronto — nunca um Router vazio pre-configurado:

```rust
// pkg/httpserver/axum.rs
use std::time::Duration;

use axum::Router;
use tower::ServiceBuilder;
use tower_http::{
    catch_panic::CatchPanicLayer,
    compression::CompressionLayer,
    cors::CorsLayer,
    request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer},
    timeout::TimeoutLayer,
    trace::TraceLayer,
};

/// Aplica a pilha padrao de middlewares sobre as rotas ja montadas.
/// Chamar por ultimo no composition root, depois de todos os merges.
pub fn with_default_middlewares(router: Router) -> Router {
    router.layer(
        ServiceBuilder::new()
            .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))   // requestid
            .layer(PropagateRequestIdLayer::x_request_id())
            .layer(TraceLayer::new_for_http())                         // logger
            .layer(CatchPanicLayer::new())                             // recover
            .layer(CorsLayer::permissive())                            // cors (ajustar por ambiente)
            .layer(CompressionLayer::new())                            // compress
            .layer(TimeoutLayer::new(Duration::from_secs(10))),        // timeout
    )
}
```

Com isso, todo handler que declara `ValidatedJson<T>` executa automaticamente:
1. Parse do JSON
2. Validacao via atributos `#[validate]` do crate `validator`
3. Retorno de erro 400 formatado se parse ou validacao falharem

### Atributos de Validacao — Referencia Rapida

Atributos mais usados do crate `validator` nas structs de input:

| Atributo | Descricao | Exemplo |
|---|---|---|
| `length` | Tamanho minimo/maximo de string | `#[validate(length(min = 3, max = 100))]` |
| `email` | Email valido | `#[validate(email)]` |
| `range` | Valor numerico min/max | `#[validate(range(min = 1))]` |
| `url` | URL valida | `#[validate(url)]` |
| `nested` | Valida cada elemento de Vec/struct aninhada | `#[validate(nested)]` (equivale ao `dive`) |
| `custom` | Funcao de validacao propria | `#[validate(custom(function = "validate_currency"))]` |
| campo `Option<T>` | So valida se preenchido | `Option<String>` + `#[validate(email)]` (equivale ao `omitempty`) |
| enum serde | Enumeracoes fechadas | `enum Currency { BRL, USD, EUR }` — deserializacao rejeita valores fora do enum (equivale ao `oneof`) |
| tipo `Uuid` | UUID valido | `pub user_id: Uuid` — o proprio tipo rejeita UUID invalido no parse |

**Padrao para structs de input:**

```rust
#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateOrderInput {
    pub user_id: Uuid,                       // Uuid ja valida formato no parse
    #[validate(range(min = 1))]
    pub total_cents: i64,
    pub currency: Currency,                  // enum fechado (equivale ao oneof)
    #[validate(length(max = 500))]
    pub notes: Option<String>,               // opcional: so valida se Some
    #[validate(length(min = 1), nested)]
    pub items: Vec<CreateItemInput>,
}

#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateItemInput {
    pub product_id: Uuid,
    #[validate(range(min = 1))]
    pub quantity: i32,
    #[validate(range(min = 1))]
    pub unit_price_cents: i64,
}

#[derive(Debug, Deserialize, Serialize, ToSchema)]
pub enum Currency { BRL, USD, EUR }
```

**Regras de validacao:**
- **Sempre** usar atributos `#[validate]` (ou tipos que validam no parse: `Uuid`, enums serde) em structs de input. Nunca validar manualmente no handler.
- **Tipos fortes** para campos que sao IDs (`Uuid`) e enumeracoes (enum serde em vez de validar no use case).
- **`nested`** em `Vec<_>` para validar cada elemento individualmente.
- **`Option<T>`** em campos opcionais que so devem ser validados se preenchidos.
- Mensagens de erro geradas pelo `format_validation_errors` — nunca expor erros crus do validator/serde para o usuario.

**Registro dos erros no app (impl do trait, papel do ProvideErrorMapper):**

```rust
// apps/user/infrastructure/controller/errors.rs
use axum::http::StatusCode;
use pkg::controller::HttpDomainError;

use crate::domain::entity::DomainError;

/// Configura o mapeamento de erros DESTE app: variante -> HTTP status.
impl HttpDomainError for DomainError {
    fn status_code(&self) -> StatusCode {
        match self {
            DomainError::InvalidInput => StatusCode::BAD_REQUEST,
            DomainError::UserNotFound => StatusCode::NOT_FOUND,
            DomainError::UserAlreadyExists => StatusCode::CONFLICT,
            DomainError::InvalidRole => StatusCode::UNPROCESSABLE_ENTITY,
            DomainError::Forbidden => StatusCode::FORBIDDEN,
            DomainError::Unexpected(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

O `match` e exaustivo: ao adicionar uma nova variante de erro, o compilador **obriga** a mapear o status HTTP correspondente (no Go, um erro nao registrado virava 500 silenciosamente).

#### Controller Concreto — Exemplo Simples (sem transaction)

```rust
// apps/user/infrastructure/controller/user_controller.rs
use std::sync::Arc;

use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use serde::Deserialize;
use utoipa::ToSchema;
use uuid::Uuid;
use validator::Validate;

use pkg::controller::{ApiError, RequestContext, ValidatedJson};

use crate::domain::entity::DomainError;
use crate::domain::usecase::user::{
    CreateInput, CreateUseCase, GetByIdInput, GetByIdUseCase, ListInput, ListUseCase,
};

pub struct UserController {
    create_use_case: Arc<dyn CreateUseCase>,
    get_by_id_use_case: Arc<dyn GetByIdUseCase>,
    list_use_case: Arc<dyn ListUseCase>,
}

impl UserController {
    pub fn new(
        create_use_case: Arc<dyn CreateUseCase>,
        get_by_id_use_case: Arc<dyn GetByIdUseCase>,
        list_use_case: Arc<dyn ListUseCase>,
    ) -> Self {
        Self { create_use_case, get_by_id_use_case, list_use_case }
    }

    pub fn routes(self) -> Router {
        let ctrl = Arc::new(self);
        Router::new()
            .route("/v1/users", post(Self::create).get(Self::list))
            .route("/v1/users/{id}", get(Self::get_by_id))   // sintaxe {id} do Axum 0.8
            .with_state(ctrl)
    }

    /// Cria um usuario
    #[utoipa::path(
        post,
        path = "/v1/users",
        tag = "users",
        request_body = CreateUserInput,
        responses(
            (status = 201, description = "Usuario criado", body = crate::domain::entity::User),
            (status = 400, description = "Input invalido", body = pkg::controller::ErrorResponse),
            (status = 409, description = "Email ja em uso", body = pkg::controller::ErrorResponse),
        )
    )]
    async fn create(
        State(ctrl): State<Arc<UserController>>,
        rc: RequestContext,
        ValidatedJson(input): ValidatedJson<CreateUserInput>,
    ) -> Result<impl IntoResponse, ApiError> {
        // body ja foi parseado e validado pelo ValidatedJson
        // request_id, parent_id, ip ja estao no rc — zero boilerplate
        let output = ctrl
            .create_use_case
            .perform(CreateInput { name: input.name, email: input.email })
            .await?;   // DomainError -> ApiError via HttpDomainError

        tracing::info!(request_id = %rc.request_id, user_id = %output.user.id, "user created");
        Ok((StatusCode::CREATED, Json(output.user)))
    }

    async fn get_by_id(
        State(ctrl): State<Arc<UserController>>,
        Path(id): Path<Uuid>,
    ) -> Result<impl IntoResponse, ApiError> {
        let user = ctrl
            .get_by_id_use_case
            .perform(GetByIdInput { id })
            .await?
            .ok_or(DomainError::UserNotFound)?;

        Ok((StatusCode::OK, Json(user)))
    }

    async fn list(
        State(ctrl): State<Arc<UserController>>,
        Query(filter): Query<ListUsersFilter>,
    ) -> Result<impl IntoResponse, ApiError> {
        let output = ctrl
            .list_use_case
            .perform(ListInput {
                cursor: filter.cursor,
                limit: filter.limit.unwrap_or(20),
            })
            .await?;

        Ok((StatusCode::OK, Json(output)))
    }
}

#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateUserInput {
    #[validate(length(min = 2, max = 100))]
    pub name: String,
    #[validate(email)]
    pub email: String,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ListUsersFilter {
    pub cursor: Option<Uuid>,
    pub limit: Option<i64>,
}
```

#### Controller com Transaction — Exemplo Completo

Quando um endpoint precisa executar **multiplas operacoes atomicas** (ex: criar order + items + atualizar estoque), o controller orquestra a transaction via `pool.begin()` e metodos `*_with_tx` dos repositorios concretos:

```rust
// apps/billing/infrastructure/controller/order_controller.rs
use std::sync::Arc;

use anyhow::Context;
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    routing::post,
    Json, Router,
};
use serde::Deserialize;
use sqlx::PgPool;
use utoipa::ToSchema;
use uuid::Uuid;
use validator::Validate;

use pkg::controller::{ApiError, RequestContext, ValidatedJson};

use crate::domain::entity::{DomainError, Order, OrderItem};
use crate::domain::event::OrderEvent;
use crate::infrastructure::repository::{PostgresOrderRepository, PostgresStockRepository};

pub struct OrderController {
    pool: PgPool,
    order_repo: Arc<PostgresOrderRepository>,   // tipo CONCRETO: necessario p/ metodos *_with_tx
    stock_repo: Arc<PostgresStockRepository>,
    order_event: Arc<dyn OrderEvent>,
}

impl OrderController {
    pub fn new(
        pool: PgPool,
        order_repo: Arc<PostgresOrderRepository>,
        stock_repo: Arc<PostgresStockRepository>,
        order_event: Arc<dyn OrderEvent>,
    ) -> Self {
        Self { pool, order_repo, stock_repo, order_event }
    }

    pub fn routes(self) -> Router {
        let ctrl = Arc::new(self);
        Router::new()
            .route("/v1/orders", post(Self::create))
            .route("/v1/orders/{id}/cancel", post(Self::cancel))
            .with_state(ctrl)
    }

    /// Create — operacao que PRECISA de transaction
    /// Cria order + items + debita estoque atomicamente.
    async fn create(
        State(ctrl): State<Arc<OrderController>>,
        rc: RequestContext,
        ValidatedJson(input): ValidatedJson<CreateOrderInput>,
    ) -> Result<impl IntoResponse, ApiError> {
        // === TRANSACTION: tudo dentro e atomico ===
        let mut tx = ctrl.pool.begin().await.context("begin tx").map_err(DomainError::from)?;

        // 1. criar order
        let order = ctrl
            .order_repo
            .create_with_tx(&mut tx, &Order::new(input.user_id, input.total_cents))
            .await
            .map_err(DomainError::from)?;

        // 2. criar items (mesma tx)
        for item in &input.items {
            ctrl.order_repo
                .create_item_with_tx(
                    &mut tx,
                    &OrderItem::new(order.id, item.product_id, item.quantity, item.unit_price_cents),
                )
                .await
                .map_err(DomainError::from)?;
        }

        // 3. debitar estoque (mesma tx)
        for item in &input.items {
            ctrl.stock_repo
                .debit_with_tx(&mut tx, item.product_id, item.quantity)
                .await
                .map_err(DomainError::from)?;
        }

        // COMMIT — order, items e estoque persistidos atomicamente.
        // Se qualquer `?` acima retornar antes, o drop da tx faz ROLLBACK automatico.
        tx.commit().await.context("commit tx").map_err(DomainError::from)?;

        // === DEPOIS DO COMMIT: publicar evento ===
        // Nunca publicar evento dentro da transaction
        let _ = ctrl.order_event.publish_created(&order).await;

        tracing::info!(request_id = %rc.request_id, order_id = %order.id, "order created");
        Ok((StatusCode::CREATED, Json(order)))
    }

    /// Cancel — operacao SIMPLES, sem transaction
    /// Apenas atualiza status. Uma unica operacao nao precisa de tx.
    async fn cancel(
        State(ctrl): State<Arc<OrderController>>,
        rc: RequestContext,
        Path(id): Path<Uuid>,
    ) -> Result<impl IntoResponse, ApiError> {
        let mut order = ctrl
            .order_repo
            .find_by_id(id)
            .await
            .map_err(DomainError::from)?
            .ok_or(DomainError::OrderNotFound)?;

        order.cancel()?;   // regra de negocio (ex: order ja cancelada) — DomainError

        ctrl.order_repo.update(&order).await.map_err(DomainError::from)?;

        let _ = ctrl.order_event.publish_cancelled(&order).await;

        tracing::info!(request_id = %rc.request_id, order_id = %order.id, "order cancelled");
        Ok((StatusCode::OK, Json(order)))
    }
}

#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateOrderInput {
    pub user_id: Uuid,
    #[validate(range(min = 1))]
    pub total_cents: i64,
    #[validate(length(min = 1), nested)]
    pub items: Vec<CreateItemInput>,
}

#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateItemInput {
    pub product_id: Uuid,
    #[validate(range(min = 1))]
    pub quantity: i32,
    #[validate(range(min = 1))]
    pub unit_price_cents: i64,
}
```

#### Transactions com sqlx

Nao ha interface `Transactor` no dominio: transactions sao um detalhe de infraestrutura, orquestradas onde controller e repositorio se encontram (ambos infraestrutura). O padrao canonico:

```rust
let mut tx = pool.begin().await.context("begin tx")?;   // sqlx::Transaction<'_, Postgres>
// ... operacoes com &mut tx via metodos *_with_tx
tx.commit().await.context("commit tx")?;
// rollback e AUTOMATICO no drop se o commit nao acontecer (early return, `?`, panic)
```

Helpers opcionais vivem em `pkg/postgres/tx.rs`. A garantia central: **nenhum caminho de erro deixa transaction aberta** — o drop da `Transaction` sem commit executa rollback.

#### Repositorio com Metodos `*_with_tx`

Quando um repositorio participa de uma transaction externa, expoe **metodos inerentes na implementacao concreta** (fora do trait de dominio, que nao pode conhecer `sqlx::Transaction`). Controllers que orquestram tx recebem o repositorio **concreto** (`Arc<PostgresOrderRepository>`) — permitido pois controller e repositorio sao ambos infraestrutura:

```rust
// apps/billing/domain/repository/order_repository.rs
use anyhow::Result;
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::Order;

/// Trait de dominio: apenas metodos SEM transaction (autocommit).
#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Order>>;
    async fn update(&self, order: &Order) -> Result<()>;
}
```

```rust
// apps/billing/infrastructure/repository/order_repository.rs
use anyhow::{Context, Result};
use sqlx::{Postgres, Transaction};

impl PostgresOrderRepository {
    /// Metodos *_with_tx (participam de transaction externa) — impl concreta, fora do trait.
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

#### Quando Usar Transaction no Controller

| Cenario | Transaction? | Pattern |
|---|---|---|
| GET (leitura simples) | **Nao** | handler com `Path`/`Query` + use case |
| POST com 1 INSERT | **Nao** | `ValidatedJson` + use case faz INSERT unico |
| POST com N INSERTs relacionados | **Sim** | `pool.begin()` + `*_with_tx` no handler |
| PUT/PATCH simples | **Nao** | Use case faz UPDATE unico |
| PUT que move saldo (debito + credito) | **Sim** | `pool.begin()` + `*_with_tx` no handler |
| DELETE com cascade manual | **Sim** | `pool.begin()` + `*_with_tx` no handler |
| Operacao + publicacao de evento | **Evento FORA da tx** | `tx.commit()` depois `event.publish(...)` |

#### Regras do Controller

- `RequestContext`, `ValidatedJson`, `ApiError`, `HttpDomainError` e `ErrorResponse` vivem em **`pkg/controller/`**. Nenhum app reimplementa essa logica.
- Arquivo: `<name>_controller.rs` (ex: `user_controller.rs`) em `infrastructure/controller/`.
- Struct: `<Name>Controller` com os use cases como campos `Arc<dyn Trait>`.
- Construtor: `new(deps...) -> Self`.
- Dependencias sao **traits de dominio de use case**; repositorios **concretos** + `PgPool` apenas quando o handler orquestra transaction.
- Deve implementar `pub fn routes(self) -> Router`, que embrulha o controller em `Arc` e registra as rotas com `with_state`.
- Handlers sao associated functions `async fn` recebendo `State<Arc<Self>>` + extractors.
- Handlers recebem `RequestContext` ja populado — nunca extrair request_id, IP, etc. manualmente.
- Handlers delegam logica de negocio para use cases. Sem logica de negocio no controller.
- Sintaxe de path params do Axum 0.8: `{id}` (e `{*rest}` para wildcard).
- Handlers documentados com `#[utoipa::path]` (OpenAPI).
- Structs de input com `#[derive(Deserialize, Validate, ToSchema)]` + `#[serde(rename_all = "camelCase")]`.
- Usar `ValidatedJson` para body, `Query` para query params, `Path` para path params. Nunca `serde_json::from_*` manual.
- Evento publicado **DEPOIS** do commit da transaction, nunca dentro.
- Erros de dominio propagados com `?` — o `From<DomainError> for ApiError` mapeia para HTTP status via `HttpDomainError`.
- Cada app registra seus erros implementando `HttpDomainError` em `infrastructure/controller/errors.rs`. O `pkg/controller` nao conhece nenhum erro de dominio.

### subscriber/ (apenas Consumers)

Handlers de consumo NATS JetStream. Cada subscriber consome de um subject especifico e delega o processamento para use cases. **Usado apenas por binarios de Consumer.**

**Padrao:**

```rust
// apps/billing/infrastructure/subscriber/user_subscriber.rs
use std::sync::Arc;

use async_nats::jetstream::{AckKind, Message};
use uuid::Uuid;

use pkg::event::{Envelope, UserCreatedData};

use crate::domain::usecase::billing::{CreateAccountInput, CreateAccountUseCase};

pub struct UserSubscriber {
    create_account: Arc<dyn CreateAccountUseCase>,
}

impl UserSubscriber {
    pub fn new(create_account: Arc<dyn CreateAccountUseCase>) -> Self {
        Self { create_account }
    }

    pub async fn handle_user_created(&self, msg: Message) {
        let request_id = Uuid::now_v7().to_string();
        let parent_id = msg
            .headers
            .as_ref()
            .and_then(|h| h.get("Nats-Msg-Id"))
            .map(|v| v.to_string())
            .unwrap_or_default();
        let subject = msg.subject.to_string();

        let envelope: Envelope = match serde_json::from_slice(&msg.payload) {
            Ok(env) => env,
            Err(err) => {
                tracing::error!(request_id, parent_id, subject, error = %err, "unmarshal envelope failed");
                let _ = msg.ack_with(AckKind::Term).await;   // mensagem corrompida — nao reenviar
                return;
            }
        };

        let data: UserCreatedData = match serde_json::from_value(envelope.data) {
            Ok(data) => data,
            Err(err) => {
                tracing::error!(request_id, parent_id, subject, error = %err, "unmarshal data failed");
                let _ = msg.ack_with(AckKind::Term).await;
                return;
            }
        };

        tracing::info!(request_id, parent_id, subject, "processing user created event");

        let result = self
            .create_account
            .perform(CreateAccountInput {
                user_id: data.user_id,
                name: data.name,
                email: data.email,
            })
            .await;

        match result {
            Ok(_) => {
                tracing::info!(request_id, parent_id, "account created successfully");
                let _ = msg.ack().await;
            }
            Err(err) => {
                tracing::error!(request_id, parent_id, error = %err, "create account failed");
                let _ = msg.ack_with(AckKind::Nak(None)).await;   // redelivery com backoff
            }
        }
    }
}
```

**Regras:**
- Arquivo: `<name>_subscriber.rs` (ex: `user_subscriber.rs`).
- Struct: `<Name>Subscriber`.
- Construtor: `new(deps...) -> Self`.
- Dependencias sao **traits de dominio de use case** (`Arc<dyn Trait>`).
- Handlers nomeados: `handle_<event_name>` (ex: `handle_user_created`).
- `msg.ack().await` apenas apos processamento com sucesso.
- `msg.ack_with(AckKind::Nak(None)).await` para erros recuperaveis (JetStream faz redelivery com backoff).
- `msg.ack_with(AckKind::Term).await` para mensagens corrompidas/invalidas (nao reenviar).
- Logging estruturado com `request_id` e `parent_id` em todo handler.

### repository/

Implementacoes de repositorio usando PostgreSQL via sqlx. Queries SQL escritas manualmente, sem ORM. A infra define **row structs** (`#[derive(sqlx::FromRow)]`) e converte para entidades — o dominio nunca deriva `FromRow`.

**Padrao:**

```rust
// apps/user/infrastructure/repository/user_repository.rs
use anyhow::{Context, Result};
use async_trait::async_trait;
use chrono::{DateTime, Utc};
use sqlx::PgPool;
use uuid::Uuid;

use crate::domain::entity::User;
use crate::domain::repository::UserRepository;

#[derive(sqlx::FromRow)]
struct UserRow {
    id: Uuid,
    name: String,
    email: String,
    created_at: DateTime<Utc>,
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

pub struct PostgresUserRepository {
    pool: PgPool,
}

impl PostgresUserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn create(&self, user: &mut User) -> Result<()> {
        const QUERY: &str = r#"
            INSERT INTO users (name, email, created_at, updated_at)
            VALUES ($1, $2, $3, $4)
            RETURNING id"#;

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

    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>> {
        const QUERY: &str = r#"
            SELECT id, name, email, created_at, updated_at
            FROM users
            WHERE id = $1"#;

        let row: Option<UserRow> = sqlx::query_as(QUERY)
            .bind(id)
            .fetch_optional(&self.pool)
            .await
            .context("find user by id")?;

        Ok(row.map(Into::into))
    }

    async fn find_by_email(&self, email: &str) -> Result<Option<User>> {
        const QUERY: &str = r#"
            SELECT id, name, email, created_at, updated_at
            FROM users
            WHERE email = $1"#;

        let row: Option<UserRow> = sqlx::query_as(QUERY)
            .bind(email)
            .fetch_optional(&self.pool)
            .await
            .context("find user by email")?;

        Ok(row.map(Into::into))
    }

    // list usa paginacao por cursor (sem OFFSET)
    async fn list(&self, cursor: Option<Uuid>, limit: i64) -> Result<(Vec<User>, Option<Uuid>)> {
        let rows: Vec<UserRow> = match cursor {
            None => {
                const QUERY: &str = r#"
                    SELECT id, name, email, created_at, updated_at
                    FROM users
                    ORDER BY id ASC
                    LIMIT $1"#;
                sqlx::query_as(QUERY)
                    .bind(limit + 1)
                    .fetch_all(&self.pool)
                    .await
                    .context("list users")?
            }
            Some(cursor) => {
                const QUERY: &str = r#"
                    SELECT id, name, email, created_at, updated_at
                    FROM users
                    WHERE id > $1
                    ORDER BY id ASC
                    LIMIT $2"#;
                sqlx::query_as(QUERY)
                    .bind(cursor)
                    .bind(limit + 1)
                    .fetch_all(&self.pool)
                    .await
                    .context("list users")?
            }
        };

        let mut users: Vec<User> = rows.into_iter().map(Into::into).collect();

        let next_cursor = if users.len() as i64 > limit {
            users.truncate(limit as usize);
            users.last().map(|u| u.id)
        } else {
            None
        };

        Ok((users, next_cursor))
    }
}
```

**Regras:**
- Arquivo: `<name>_repository.rs` (ex: `user_repository.rs`).
- Struct: `Postgres<Name>Repository` (prefixo da tecnologia; evita alias de import com o trait).
- Construtor: `new(pool: PgPool) -> Self`; o composition root embrulha em `Arc<dyn Trait>`.
- Recebe `PgPool` como dependencia (clonavel; clones compartilham o mesmo pool).
- SQL escrito manualmente com placeholders posicionais (`$1`, `$2`). Sem ORM.
- Sem `SELECT *`. Listar colunas explicitamente.
- `INSERT ... RETURNING id` para obter ID gerado.
- `fetch_optional` retorna `Ok(None)` quando nao encontrado (nao e erro, e ausencia).
- Paginacao por cursor, nunca `OFFSET` (para tabelas grandes).
- Queries como `const QUERY: &str` dentro do metodo.
- Row structs privadas (`UserRow`) com `#[derive(sqlx::FromRow)]` + `From<Row> for Entity`.
- Erros com contexto: `.context("acao")?`.

### publisher/

Implementacoes de eventos usando NATS JetStream, via `Publisher` generico do `pkg`.

**Padrao:**

```rust
// apps/user/infrastructure/publisher/user_publisher.rs
use anyhow::Result;
use async_trait::async_trait;

use pkg::event::{self, UserCreatedData, UserUpdatedData};
use pkg::nats::Publisher;

use crate::domain::entity::User;
use crate::domain::event::UserEvent;

pub struct NatsUserPublisher {
    publisher: Publisher,
}

impl NatsUserPublisher {
    pub fn new(publisher: Publisher) -> Self {
        Self { publisher }
    }
}

#[async_trait]
impl UserEvent for NatsUserPublisher {
    async fn publish_created(&self, user: &User) -> Result<()> {
        self.publisher
            .publish(
                event::SUBJECT_USER_CREATED,
                &user.id.to_string(),   // msg_id = ID da entidade (deduplicacao)
                UserCreatedData {
                    user_id: user.id,
                    name: user.name.clone(),
                    email: user.email.clone(),
                },
            )
            .await
    }

    async fn publish_updated(&self, user: &User) -> Result<()> {
        self.publisher
            .publish(
                event::SUBJECT_USER_UPDATED,
                &user.id.to_string(),
                UserUpdatedData { user_id: user.id, name: user.name.clone(), email: user.email.clone() },
            )
            .await
    }
}
```

**Regras:**
- Arquivo: `<name>_publisher.rs` (ex: `user_publisher.rs`).
- Struct: `Nats<Name>Publisher`.
- Construtor: `new(publisher: Publisher) -> Self`; o composition root embrulha em `Arc<dyn <Name>Event>`.
- `msg_id` = ID da entidade para deduplicacao no JetStream (`Nats-Msg-Id`).
- Publicar DEPOIS do commit no banco, nunca antes.

### adapter/

Implementacoes concretas dos traits de bibliotecas externas definidos em `domain/<lib>/`.

**Padrao:**

```rust
// apps/user/infrastructure/adapter/argon2_adapter.rs
use anyhow::{anyhow, Result};
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

use crate::domain::crypt::Hasher;

pub struct Argon2Adapter;

impl Argon2Adapter {
    pub fn new() -> Self {
        Self
    }
}

impl Hasher for Argon2Adapter {
    fn hash(&self, plain: &str) -> Result<String> {
        let salt = SaltString::generate(&mut OsRng);
        let hash = Argon2::default()
            .hash_password(plain.as_bytes(), &salt)
            .map_err(|err| anyhow!("hash password: {err}"))?;
        Ok(hash.to_string())
    }

    fn compare(&self, hashed: &str, plain: &str) -> Result<()> {
        let parsed = PasswordHash::new(hashed).map_err(|err| anyhow!("parse hash: {err}"))?;
        Argon2::default()
            .verify_password(plain.as_bytes(), &parsed)
            .map_err(|_| anyhow!("password mismatch"))
    }
}
```

**Regras:**
- Arquivo: `<lib_name>_adapter.rs` (ex: `argon2_adapter.rs`, `jwt_adapter.rs`).
- Struct: `<LibName>Adapter`.
- Construtor: `new(deps...) -> Self`; o composition root embrulha em `Arc<dyn <Trait>>`.
- O nome do arquivo/struct reflete a **lib concreta**, pois esta na camada de infraestrutura.

### config/

Configuracao da aplicacao carregada de variaveis de ambiente.

**`config.rs`** -- Struct de configuracao e carregamento:

```rust
// apps/user/infrastructure/config/config.rs
use std::env;

pub struct AppConfig {
    pub app_port: u16,
    pub db_host: String,
    pub db_port: u16,
    pub db_user: String,
    pub db_password: String,
    pub db_name: String,
    pub db_max_connections: u32,
    pub nats_url: String,
    pub service_name: String,
    pub service_version: String,
    pub log_level: String,
}

impl AppConfig {
    pub fn load() -> Self {
        Self {
            app_port: env_parse("APP_PORT", 8080),
            db_host: env_or("DB_HOST", "localhost"),
            db_port: env_parse("DB_PORT", 5432),
            db_user: env_or("DB_USER", "app"),
            db_password: env_or("DB_PASSWORD", ""),
            db_name: env_or("DB_NAME", "users"),
            db_max_connections: env_parse("DB_MAX_CONNECTIONS", 10),
            nats_url: env_or("NATS_URL", "nats://localhost:4222"),
            service_name: env_or("SERVICE_NAME", "user-api"),
            service_version: env_or("SERVICE_VERSION", "dev"),
            log_level: env_or("LOG_LEVEL", "info"),
        }
    }

    pub fn database(&self) -> pkg::postgres::Config {
        pkg::postgres::Config {
            host: self.db_host.clone(),
            port: self.db_port,
            user: self.db_user.clone(),
            password: self.db_password.clone(),
            db_name: self.db_name.clone(),
            max_connections: self.db_max_connections,
        }
    }
}

fn env_or(key: &str, fallback: &str) -> String {
    env::var(key).unwrap_or_else(|_| fallback.to_string())
}

fn env_parse<T: std::str::FromStr>(key: &str, fallback: T) -> T {
    env::var(key).ok().and_then(|v| v.parse().ok()).unwrap_or(fallback)
}
```

**Regras:**
- `config/` contem **apenas** o `AppConfig` e o carregamento de env (`dotenvy::dotenv().ok()` e chamado no inicio do `main`).
- O papel do "modulo de infraestrutura compartilhada" (conexoes Postgres/NATS) e exercido pelos construtores do `pkg` (`pkg::postgres::new_pool`, `pkg::nats::connect`) chamados no **composition root** de cada binario.
- Registro de rotas e startup do servidor ficam no **`api.rs`**, NAO na config.
- Logica de startup do subscriber fica no **`consumer.rs`**.
- Cleanup de conexoes acontece no fim do `main`, na ordem inversa da criacao.

---

## Camada de Pacotes (`pkg/`)

Crate reutilizavel compartilhado entre todos os apps. Nunca importa nada dos apps.

| Modulo | Responsabilidade |
|---|---|
| `postgres` | `new_pool`, pool config, helpers de transaction, health check |
| `nats` | `connect`, `ensure_stream`, `Publisher`, `subscribe` |
| `httpserver` | `with_default_middlewares` (pilha padrao tower-http) |
| `controller` | `RequestContext`, `ValidatedJson`, `ApiError`, `HttpDomainError`, `ErrorResponse` |
| `logger` | `setup()` com tracing JSON + `service` e `version` |
| `event` | Envelope base e contratos de eventos compartilhados |

**Padroes principais:**

```rust
// pkg/postgres/conn.rs
use std::time::Duration;

use sqlx::postgres::{PgPool, PgPoolOptions};

pub struct Config {
    pub host: String,
    pub port: u16,
    pub user: String,
    pub password: String,
    pub db_name: String,
    pub max_connections: u32,
}

pub async fn new_pool(cfg: &Config) -> anyhow::Result<PgPool> {
    let url = format!(
        "postgres://{}:{}@{}:{}/{}",
        cfg.user, cfg.password, cfg.host, cfg.port, cfg.db_name
    );

    let pool = PgPoolOptions::new()
        .max_connections(cfg.max_connections)
        .acquire_timeout(Duration::from_secs(5))
        .max_lifetime(Duration::from_secs(30 * 60))
        .idle_timeout(Duration::from_secs(5 * 60))
        .connect(&url)
        .await?;

    Ok(pool)
}
```

```rust
// pkg/nats/conn.rs
pub async fn connect(url: &str, name: &str) -> anyhow::Result<async_nats::Client> {
    let client = async_nats::ConnectOptions::new()
        .name(name)
        .retry_on_initial_connect()   // reconexao automatica e ilimitada por padrao
        .connect(url)
        .await?;
    Ok(client)
}
```

```rust
// pkg/nats/publisher.rs
use anyhow::{Context, Result};
use async_nats::jetstream;

use crate::event::Envelope;

pub struct Publisher {
    js: jetstream::Context,
    source: String,
}

impl Publisher {
    pub fn new(js: jetstream::Context, source: impl Into<String>) -> Self {
        Self { js, source: source.into() }
    }

    pub async fn publish<T: serde::Serialize>(&self, subject: &str, msg_id: &str, data: T) -> Result<()> {
        let envelope = Envelope::new(subject, &self.source, data);
        let payload = serde_json::to_vec(&envelope).context("serialize envelope")?;

        let mut headers = async_nats::HeaderMap::new();
        headers.insert("Nats-Msg-Id", msg_id);   // deduplicacao server-side

        self.js
            .publish_with_headers(subject.to_string(), headers, payload.into())
            .await
            .context("publish")?
            .await                                // aguarda o ack do JetStream
            .context("await publish ack")?;

        Ok(())
    }
}
```

```rust
// pkg/logger/mod.rs
use tracing_subscriber::EnvFilter;

pub fn setup(service: &str, version: &str, level: &str) {
    tracing_subscriber::fmt()
        .json()
        .with_env_filter(EnvFilter::try_new(level).unwrap_or_else(|_| EnvFilter::new("info")))
        .with_current_span(true)
        .flatten_event(true)
        .init();

    tracing::info!(service, version, "logger initialized");
}
```

**Regras:**
- Sem estado global (sem `static` para conexoes; sem `lazy_static`/`OnceLock` para infra).
- Funcoes puras que recebem e retornam dependencias explicitamente.
- Nunca importa nada de nenhum app.
- Pode ser usado por outros projetos.

---

## Composition Root (Injecao de Dependencia manual)

Nao ha framework de DI: cada binario compoe seu proprio grafo de dependencias **explicitamente** no `main`, de baixo para cima (infra -> repos/publishers/adapters -> use cases -> controllers/subscribers), embrulhando implementacoes em `Arc<dyn Trait>`. O compilador garante em compile time que nenhuma dependencia falta — o equivalente do grafo do FX, sem runtime.

### Composicao no `api.rs` da API

```rust
// apps/user/api.rs
use std::sync::Arc;

use anyhow::Context;
use axum::Router;

use user::application::usecase::user as app_user;
use user::domain::event::UserEvent;
use user::domain::repository::UserRepository;
use user::domain::usecase::user::{CreateUseCase, GetByIdUseCase, ListUseCase};
use user::infrastructure::config::AppConfig;
use user::infrastructure::controller::UserController;
use user::infrastructure::publisher::NatsUserPublisher;
use user::infrastructure::repository::PostgresUserRepository;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    dotenvy::dotenv().ok();
    let cfg = AppConfig::load();

    // === SetupLogger ===
    pkg::logger::setup(&cfg.service_name, &cfg.service_version, &cfg.log_level);

    // === Infraestrutura compartilhada (config, postgres, nats) ===
    let pool = pkg::postgres::new_pool(&cfg.database()).await.context("connect postgres")?;
    let nats = pkg::nats::connect(&cfg.nats_url, &cfg.service_name).await.context("connect nats")?;
    let js = async_nats::jetstream::new(nats.clone());
    let publisher = pkg::nats::Publisher::new(js, cfg.service_name.clone());

    // === Providers especificos da API (construcao bottom-up) ===

    // repositories
    let user_repository: Arc<dyn UserRepository> =
        Arc::new(PostgresUserRepository::new(pool.clone()));

    // publishers
    let user_event: Arc<dyn UserEvent> = Arc::new(NatsUserPublisher::new(publisher));

    // use cases
    let create_use_case: Arc<dyn CreateUseCase> =
        Arc::new(app_user::CreateUseCase::new(user_repository.clone(), user_event.clone()));
    let get_by_id_use_case: Arc<dyn GetByIdUseCase> =
        Arc::new(app_user::GetByIdUseCase::new(user_repository.clone()));
    let list_use_case: Arc<dyn ListUseCase> =
        Arc::new(app_user::ListUseCase::new(user_repository.clone()));

    // controllers
    let user_controller = UserController::new(create_use_case, get_by_id_use_case, list_use_case);

    // === RegisterRoutes ===
    let app = pkg::httpserver::with_default_middlewares(
        Router::new().merge(user_controller.routes()),
    );

    // === StartServer + graceful shutdown ===
    let listener = tokio::net::TcpListener::bind(("0.0.0.0", cfg.app_port))
        .await
        .context("bind listener")?;
    tracing::info!(port = cfg.app_port, "server started");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .context("server failed")?;

    // ordem inversa: HTTP ja parou -> NATS drain -> Postgres fecha
    drop(nats);
    pool.close().await;
    tracing::info!("shutdown complete");
    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("install ctrl_c handler");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("install sigterm handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

### Composicao no `consumer.rs` do Consumer

Cada binario de consumer compoe seu proprio grafo. **Sem Axum, sem controllers, sem rotas.**

```rust
// apps/billing/consumer.rs
use std::sync::Arc;
use std::time::Duration;

use anyhow::Context;
use futures::StreamExt;

use billing::application::usecase::billing as app_billing;
use billing::domain::repository::AccountRepository;
use billing::domain::usecase::billing::CreateAccountUseCase;
use billing::infrastructure::config::AppConfig;
use billing::infrastructure::repository::PostgresAccountRepository;
use billing::infrastructure::subscriber::UserSubscriber;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    dotenvy::dotenv().ok();
    let cfg = AppConfig::load();

    // === SetupLogger ===
    pkg::logger::setup(&cfg.service_name, &cfg.service_version, &cfg.log_level);

    // === Infraestrutura compartilhada ===
    let pool = pkg::postgres::new_pool(&cfg.database()).await.context("connect postgres")?;
    let nats = pkg::nats::connect(&cfg.nats_url, &cfg.service_name).await.context("connect nats")?;
    let js = async_nats::jetstream::new(nats.clone());

    // === Providers: apenas o necessario para consumir ===
    let account_repository: Arc<dyn AccountRepository> =
        Arc::new(PostgresAccountRepository::new(pool.clone()));
    let create_account: Arc<dyn CreateAccountUseCase> =
        Arc::new(app_billing::CreateAccountUseCase::new(account_repository.clone()));
    let subscriber = Arc::new(UserSubscriber::new(create_account));

    // === EnsureStreams ===
    pkg::nats::ensure_stream(&js, pkg::nats::StreamConfig {
        name: "EVENTS_USER".to_string(),
        subjects: vec!["events.user.>".to_string()],
        max_age: Duration::from_secs(7 * 24 * 60 * 60),
        max_bytes: 1024 * 1024 * 1024,
        replicas: 1,
    })
    .await
    .context("ensure stream")?;

    // === StartSubscriber ===
    let consumer = pkg::nats::pull_consumer(&js, pkg::nats::ConsumerConfig {
        stream: "EVENTS_USER".to_string(),
        consumer: "billing-on-user-created".to_string(),
        filter_subject: "events.user.created".to_string(),
        max_deliver: 5,
        ack_wait: Duration::from_secs(30),
    })
    .await
    .context("create consumer")?;

    let worker = tokio::spawn({
        let subscriber = subscriber.clone();
        async move {
            let mut messages = match consumer.messages().await {
                Ok(m) => m,
                Err(err) => {
                    tracing::error!(error = %err, "subscribe failed");
                    return;
                }
            };
            while let Some(msg) = messages.next().await {
                let Ok(msg) = msg else { continue };
                subscriber.handle_user_created(msg).await;
            }
        }
    });

    tracing::info!("consumer started");
    shutdown_signal().await;

    // ordem inversa: para de consumir -> NATS drain -> Postgres fecha
    worker.abort();
    drop(nats);
    pool.close().await;
    tracing::info!("shutdown complete");
    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("install ctrl_c handler");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("install sigterm handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

### Regras de Composicao

1. **Infraestrutura compartilhada primeiro** em todo binario: `AppConfig::load()`, `pkg::postgres::new_pool`, `pkg::nats::connect`.
2. **Cada binario compoe apenas o que precisa.** Uma API compoe controllers; um consumer compoe subscribers. Nenhum compoe as dependencias do outro.
3. **Rotas e startup do servidor** sao definidos no `api.rs`, nao em modulo compartilhado.
4. **`ensure_stream` e o loop do subscriber** sao definidos no `consumer.rs`, nao em modulo compartilhado.
5. **Construcao bottom-up explicita**: infra -> repos/publishers/adapters -> use cases -> controllers/subscribers. Cada dependencia e passada por parametro; o compilador acusa qualquer falta em compile time.
6. **`Arc<dyn Trait>`** para toda dependencia injetada; `Arc::clone` e barato (contador de referencia).
7. **Graceful shutdown** via `with_graceful_shutdown(shutdown_signal())` na API e `shutdown_signal().await` no consumer (SIGINT + SIGTERM).
8. **Ordem de cleanup** no fim do `main` e sempre a inversa da criacao.

### Ordem de Shutdown

**API:**
1. Servidor HTTP para de aceitar conexoes e aguarda requests em andamento (`with_graceful_shutdown`).
2. Conexao NATS e derrubada (drop) apos o drain de mensagens pendentes.
3. PostgreSQL desconecta (`pool.close().await`).

**Consumer:**
1. Subscriber para de consumir mensagens (`worker.abort()`).
2. Conexao NATS e derrubada (drop).
3. PostgreSQL desconecta (`pool.close().await`).

---

## Convencoes de Nomenclatura

### Arquivos

| Localizacao | Padrao | Exemplo |
|---|---|---|
| `apps/<app>/` (raiz do crate) | `<binary>.rs` | `api.rs`, `consumer.rs` |
| `domain/entity/` | `<name>.rs` | `user.rs` |
| `domain/usecase/<context>/` | `<action>.rs` | `user/create.rs` |
| `domain/repository/` | `<name>_repository.rs` | `user_repository.rs` |
| `domain/event/` | `<name>_event.rs` | `user_event.rs` |
| `domain/<lib>/` | `<name>.rs` | `crypt/hasher.rs` |
| `application/usecase/<context>/` | `<action>_usecase.rs` | `user/create_usecase.rs` |
| `infrastructure/controller/` | `<name>_controller.rs` | `user_controller.rs` |
| `infrastructure/repository/` | `<name>_repository.rs` | `user_repository.rs` |
| `infrastructure/publisher/` | `<name>_publisher.rs` | `user_publisher.rs` |
| `infrastructure/subscriber/` | `<name>_subscriber.rs` | `user_subscriber.rs` |
| `infrastructure/adapter/` | `<lib>_adapter.rs` | `argon2_adapter.rs` |

- Nomes de arquivo sempre em **snake_case** (exigencia da linguagem).
- Testes unitarios: `#[cfg(test)] mod tests` no proprio arquivo da implementacao.
- Testes de integracao: `apps/<app>/tests/<name>_integration_test.rs`.

### Modulos

Sempre **lowercase** e **singular**: `entity`, `usecase`, `repository`, `event`, `crypt`, `token`, `controller`, `publisher`, `subscriber`, `adapter`, `config`. Estilo `mod.rs` (1 diretorio = 1 modulo).

### Tipos e Traits

| Elemento | Padrao | Exemplo |
|---|---|---|
| Entidade | `<Name>` | `User` |
| Trait UC | `<Action>UseCase` | `CreateUseCase` |
| Trait Repo | `<Name>Repository` | `UserRepository` |
| Trait Event | `<Name>Event` | `UserEvent` |
| Trait Lib | `<Capability>` | `Hasher`, `Manager` |
| Struct Impl UC | `<Action>UseCase` | `CreateUseCase` (mesmo nome, modulo `application`) |
| Controller | `<Name>Controller` | `UserController` |
| Impl Repo | `Postgres<Name>Repository` | `PostgresUserRepository` |
| Impl Publisher | `Nats<Name>Publisher` | `NatsUserPublisher` |
| Subscriber | `<Name>Subscriber` | `UserSubscriber` |
| Adapter | `<LibName>Adapter` | `Argon2Adapter` |
| Erros de dominio | `DomainError` (enum thiserror) | `DomainError::UserNotFound` |

### Construtores

| Camada | Assinatura | Embrulho no composition root |
|---|---|---|
| Entidade | `new(params...) -> Self` | - |
| Application UC | `new(deps: Arc<dyn Trait>...) -> Self` | `Arc<dyn <Action>UseCase>` |
| Infra Repo | `new(pool: PgPool) -> Self` | `Arc<dyn <Name>Repository>` (ou `Arc<Postgres<Name>Repository>` p/ tx) |
| Infra Publisher | `new(publisher: Publisher) -> Self` | `Arc<dyn <Name>Event>` |
| Controller | `new(use_cases...) -> Self` | consumido por `routes(self)` |
| Subscriber | `new(use_cases...) -> Self` | `Arc<<Name>Subscriber>` |
| Adapter | `new(deps...) -> Self` | `Arc<dyn <Capability>>` |

### Imports (`use`)

Organizados em 3 blocos separados por linhas em branco (ordem que o rustfmt preserva):

```rust
// 1. std
use std::sync::Arc;
use std::time::Duration;

// 2. third-party
use axum::{Json, Router};
use sqlx::PgPool;

// 3. projeto interno (pkg + crate)
use pkg::controller::{ApiError, ValidatedJson};

use crate::domain::usecase::user::CreateUseCase;
```

---

## Comunicacao entre Servicos

Cada app tem seu proprio banco (database-per-service). Comunicacao entre apps e feita via NATS JetStream.

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

```rust
// pkg/event/envelope.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize)]
pub struct Envelope {
    pub id: String,
    pub r#type: String,
    pub source: String,
    pub timestamp: DateTime<Utc>,
    pub data: serde_json::Value,
}

impl Envelope {
    pub fn new<T: Serialize>(event_type: &str, source: &str, data: T) -> Self {
        Self {
            id: Uuid::now_v7().to_string(),
            r#type: event_type.to_string(),
            source: source.to_string(),
            timestamp: Utc::now(),
            data: serde_json::to_value(data).unwrap_or(serde_json::Value::Null),
        }
    }
}
```

```rust
// pkg/event/user.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

pub const SUBJECT_USER_CREATED: &str = "events.user.created";
pub const SUBJECT_USER_UPDATED: &str = "events.user.updated";

#[derive(Debug, Serialize, Deserialize)]
pub struct UserCreatedData {
    pub user_id: Uuid,
    pub name: String,
    pub email: String,
}
```

**Regras:**
- Subjects hierarquicos: `events.<dominio>.<acao>` (ex: `events.user.created`).
- Um stream por dominio: `EVENTS_USER`, `EVENTS_BILLING`.
- Publisher usa `Nats-Msg-Id` = ID da entidade para deduplicacao server-side.
- Consumer duravel (pull) com ack explicito, backoff exponencial e `max_deliver`.
- Idempotencia no consumer: `INSERT ... ON CONFLICT DO NOTHING`.

---

## Guia Passo a Passo: Adicionando uma Nova Feature

Exemplo: adicionando o dominio `Order` no app `billing`.

### Passo 1 -- Entidade de Dominio

Crie `apps/billing/domain/entity/order.rs`:

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct Order {
    pub id: Uuid,
    pub user_id: Uuid,
    pub amount: i64,
    pub status: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl Order {
    pub fn new(user_id: Uuid, amount: i64) -> Self {
        let now = Utc::now();
        Self {
            id: Uuid::nil(),
            user_id,
            amount,
            status: "pending".to_string(),
            created_at: now,
            updated_at: now,
        }
    }
}
```

Registre o modulo em `apps/billing/domain/entity/mod.rs`.

### Passo 2 -- Traits de Dominio (Use Cases)

Crie um trait por use case, organizados por contexto:

Crie `apps/billing/domain/usecase/order/create.rs`:

```rust
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::{DomainError, Order};

pub struct CreateInput {
    pub user_id: Uuid,
    pub amount: i64,
}

pub struct CreateOutput {
    pub order: Order,
}

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait CreateUseCase: Send + Sync {
    async fn perform(&self, input: CreateInput) -> Result<CreateOutput, DomainError>;
}
```

### Passo 2b -- Traits de Dominio (Repository e Event)

Crie `apps/billing/domain/repository/order_repository.rs`:

```rust
use anyhow::Result;
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::Order;

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn create(&self, order: &mut Order) -> Result<()>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Order>>;
}
```

Crie `apps/billing/domain/event/order_event.rs` (se aplicavel):

```rust
use anyhow::Result;
use async_trait::async_trait;

use crate::domain::entity::Order;

#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait OrderEvent: Send + Sync {
    async fn publish_created(&self, order: &Order) -> Result<()>;
}
```

### Passo 3 -- Use Case de Aplicacao

Crie `apps/billing/application/usecase/order/create_usecase.rs`:

```rust
use std::sync::Arc;

use anyhow::Context;
use async_trait::async_trait;

use crate::domain::entity::{DomainError, Order};
use crate::domain::repository::OrderRepository;
use crate::domain::usecase::order::{CreateInput, CreateOutput};

pub struct CreateUseCase {
    order_repository: Arc<dyn OrderRepository>,
}

impl CreateUseCase {
    pub fn new(order_repository: Arc<dyn OrderRepository>) -> Self {
        Self { order_repository }
    }
}

#[async_trait]
impl crate::domain::usecase::order::CreateUseCase for CreateUseCase {
    async fn perform(&self, input: CreateInput) -> Result<CreateOutput, DomainError> {
        let mut order = Order::new(input.user_id, input.amount);
        self.order_repository
            .create(&mut order)
            .await
            .context("create order")?;
        Ok(CreateOutput { order })
    }
}
```

### Passo 4 -- Repositorio de Infraestrutura

Crie `apps/billing/infrastructure/repository/order_repository.rs` (struct `PostgresOrderRepository` + row struct + impl do trait).

### Passo 5 -- Publisher de Infraestrutura (se aplicavel)

Crie `apps/billing/infrastructure/publisher/order_publisher.rs` (struct `NatsOrderPublisher`).

### Passo 6a -- Controller (se a API precisar)

Crie `apps/billing/infrastructure/controller/order_controller.rs` e mapeie novos erros em `infrastructure/controller/errors.rs`.

### Passo 6b -- Subscriber (se o Consumer precisar)

Crie `apps/billing/infrastructure/subscriber/order_subscriber.rs`.

### Passo 7 -- Migration SQL

Crie a migration com sqlx-cli seguindo as boas praticas de PostgreSQL 18:

```bash
sqlx migrate add -r create_orders --source migrations/billing
# gera migrations/billing/<timestamp>_create_orders.up.sql e .down.sql
```

### Passo 8 -- Composition Root

**Para a API** -- atualize `apps/billing/api.rs`:

```rust
// providers novos (bottom-up)
let order_repository: Arc<dyn OrderRepository> =
    Arc::new(PostgresOrderRepository::new(pool.clone()));
let order_event: Arc<dyn OrderEvent> = Arc::new(NatsOrderPublisher::new(publisher.clone()));
let create_order: Arc<dyn order_usecase::CreateUseCase> =
    Arc::new(app_order::CreateUseCase::new(order_repository.clone()));
let order_controller = OrderController::new(create_order);

// registrar rotas
let app = pkg::httpserver::with_default_middlewares(
    Router::new()
        .merge(user_controller.routes())
        .merge(order_controller.routes()),
);
```

### Passo 9 -- Testes

- Unitario: `#[cfg(test)] mod tests` em `apps/billing/application/usecase/order/create_usecase.rs` (mockall).
- Integracao: `apps/billing/tests/order_repository_integration_test.rs` (testcontainers).

### Passo 10 -- OpenAPI (apenas APIs)

Adicione o handler ao `#[derive(OpenApi)]` do app (utoipa) e verifique `/swagger-ui` e `/api-docs/openapi.json` — a documentacao e gerada em compile time, sem passo de CLI.

### Checklist -- Nova Feature

- [ ] `domain/entity/<name>.rs` -- Entidade com construtor
- [ ] `domain/usecase/<context>/<action>.rs` -- 1 trait por use case (metodo `perform`)
- [ ] `domain/repository/<name>_repository.rs` -- Trait de repositorio
- [ ] `domain/event/<name>_event.rs` -- Trait de evento (se aplicavel)
- [ ] `domain/<lib>/<name>.rs` -- Trait para lib externa (se aplicavel)
- [ ] `application/usecase/<context>/<action>_usecase.rs` -- Implementacao do use case
- [ ] `#[cfg(test)] mod tests` no arquivo do use case -- Teste unitario (mockall)
- [ ] `infrastructure/repository/<name>_repository.rs` -- Implementacao PostgreSQL (sqlx)
- [ ] `tests/<name>_repository_integration_test.rs` -- Teste de integracao (testcontainers)
- [ ] `infrastructure/publisher/<name>_publisher.rs` -- Implementacao NATS (se aplicavel)
- [ ] `infrastructure/adapter/<lib>_adapter.rs` -- Implementacao de lib externa (se aplicavel)
- [ ] `infrastructure/controller/<name>_controller.rs` -- Controller Axum (se API)
- [ ] `infrastructure/controller/errors.rs` -- Mapear novas variantes de erro (se API)
- [ ] `infrastructure/subscriber/<name>_subscriber.rs` -- Subscriber NATS (se Consumer)
- [ ] `migrations/<app>/<timestamp>_<descricao>.up.sql` -- Migration SQL (sqlx-cli)
- [ ] `apps/<app>/api.rs` -- Registrar providers e rotas no composition root (se API)
- [ ] `apps/<app>/consumer.rs` -- Criar ou atualizar binario do consumer (se Consumer)
- [ ] Registrar `mod`s novos nos `mod.rs` de cada camada
- [ ] Atualizar o `#[derive(OpenApi)]` do app (se API)

### Checklist -- Novo App

- [ ] Criar `apps/<app-name>/Cargo.toml` (deps via `{ workspace = true }`, `[[bin]]` api/consumer)
- [ ] Adicionar ao `members` do `Cargo.toml` da raiz
- [ ] Criar `apps/<app-name>/lib.rs` com `pub mod domain; application; infrastructure;`
- [ ] Criar `apps/<app-name>/api.rs` e/ou `consumer.rs`
- [ ] Carregar `AppConfig` + `pkg::postgres::new_pool` + `pkg::nats::connect` no composition root
- [ ] Compor apenas repositorios, publishers, use cases e handlers necessarios
- [ ] Definir startup (`routes`+`axum::serve` para API, `ensure_stream`+loop do subscriber para Consumer)
- [ ] Chamar `pkg::logger::setup` e a funcao de startup
- [ ] Criar `migrations/<app-name>/` com migrations iniciais (sqlx-cli)
- [ ] Criar `Dockerfile` (multi-stage: `rust:1.85` builder + runtime slim)
- [ ] Adicionar ao `docker-compose.yml`

