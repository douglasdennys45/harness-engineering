---
name: rust-best-practices
description: Aplica boas praticas oficiais de Rust (doc.rust-lang.org — The Book, Rust API Guidelines, Rust Reference) e do ecossistema async (tokio.rs) ao escrever, revisar ou refatorar codigo Rust neste harness. Use ao criar/editar arquivos .rs, ao revisar codigo Rust, ao discutir convencoes de nomenclatura (snake_case/PascalCase/SCREAMING_SNAKE_CASE), organizacao de modulos, ownership/borrowing/lifetimes, tratamento de erros com Result/Option (thiserror para erros de dominio + anyhow com .context() na propagacao, ?, matches!, map_err), evitar unwrap()/panic em producao, concorrencia com Tokio (async/await, tokio::spawn, tokio::select!, cancelamento, nunca bloquear o runtime), traits e generics (Arc<dyn Trait> com Send + Sync, async-trait), iteradores, derives comuns (Debug, Clone, Serialize/Deserialize), edition 2024, clippy com -D warnings, rustfmt e evitar unsafe. Acione sempre que o trabalho envolver Rust.
---

# Rust Best Practices (edition 2024 · Tokio · thiserror + anyhow)

Skill baseada na documentacao oficial de Rust (doc.rust-lang.org — The Book, Rust API Guidelines, Rust Reference), no runtime async oficial (tokio.rs) e nos crates canonicos do ecossistema (`thiserror`, `anyhow`). Todas as recomendacoes aqui devem ser seguidas ao escrever, revisar ou refatorar codigo Rust neste harness. Para os padroes de arquitetura, camadas e composicao, a fonte de verdade e `.claude/rules/architecture.md`.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.rs`
- Ao discutir design, nomenclatura ou arquitetura de aplicacoes Rust
- Ao tratar de ownership, borrowing, lifetimes, erros, `Option` ou concorrencia async
- Ao desenhar traits, generics, injecao de dependencia (`Arc<dyn Trait>`) ou use cases
- Ao gerar exemplos de codigo Rust em respostas

## 1. Convencoes de Nomenclatura

Referencia: [Rust API Guidelines — Naming (RFC 430)](https://rust-lang.github.io/api-guidelines/naming.html)

### Casing por elemento

| Elemento | Convencao | Exemplo |
|---|---|---|
| Crates, modulos, arquivos | `snake_case` | `user_repository.rs`, `mod usecase` |
| Tipos, traits, enums, variantes | `PascalCase` | `User`, `UserRepository`, `DomainError::UserNotFound` |
| Funcoes, metodos, variaveis, campos | `snake_case` | `find_by_id`, `next_cursor`, `created_at` |
| Constantes e statics | `SCREAMING_SNAKE_CASE` | `MAX_PAGE_SIZE`, `SUBJECT_USER_CREATED` |
| Parametros de tipo (generics) | `PascalCase` curto | `T`, `E`, `K`, `V` |
| Lifetimes | `'` + minuscula curta | `'a`, `'de`, `'static` |

### Acronimos

- Acronimos e compostos contam como **uma palavra** em `PascalCase`: `HttpDomainError`, `ApiError`, `UuidV7`, **nao** `HTTPDomainError` nem `APIError`.
- Em `snake_case`, tudo minusculo: `user_id`, `request_id`, `http_client`.

### Nomenclatura de metodos (convencoes do ecossistema)

- Conversoes seguem o trio canonico: `as_` (barato, por referencia), `to_` (custoso, clona/aloca), `into_` (consome `self`). Ex.: `as_str`, `to_string`, `into_bytes`.
- Getters **sem** prefixo `get_`: o campo `owner` tem getter `owner()`, nao `get_owner()`. Setter usa `set_owner(...)`.
- Construtor padrao e `new(...) -> Self`; variacoes usam `with_*` (`with_capacity`).
- Metodos que retornam iterador: `iter` (`&T`), `iter_mut` (`&mut T`), `into_iter` (consome).
- Predicados (booleanos) com prefixo `is_`/`has_`/`can_`: `is_active`, `has_pending_orders`.

### Nomes de traits

- Traits sao substantivos/capacidades em `PascalCase`: `UserRepository`, `Hasher`, `CreateUseCase`.
- **Sem** prefixo `I` (nao existe `IUserRepository`) e **sem** sufixo `Impl` na implementacao. A implementacao concreta ganha o qualificador da tecnologia: `PostgresUserRepository`, `NatsUserPublisher`, `Argon2Adapter`.

## 2. Ownership, Borrowing e Lifetimes

Referencia: [The Book — Understanding Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)

### As regras de ownership

1. Cada valor tem **um unico dono**.
2. Quando o dono sai de escopo, o valor e liberado (`drop`) — sem GC, sem `free` manual.
3. So pode existir **ou** uma referencia mutavel (`&mut T`) **ou** N referencias imutaveis (`&T`) por vez — nunca ambos. O borrow checker garante isso em compile time (sem data races).

### Empreste em vez de mover; clone so quando precisar

```rust
// ✅ empreste (&T) para ler sem tomar posse
fn total(items: &[Order]) -> i64 {
    items.iter().map(|o| o.total_cents).sum()
}

// ✅ &mut T quando precisa mutar o valor do caller (padrao do repositorio: preenche o id gerado)
async fn create(&self, user: &mut User) -> Result<()> { /* ... user.id = id; */ }

// ⚠️ .clone() e explicito e as vezes correto (ex.: mover um campo para um evento),
// mas clonar so para "calar o borrow checker" costuma esconder um problema de design.
let name = user.name.clone();
```

- Prefira `&str` a `&String` e `&[T]` a `&Vec<T>` em parametros (aceita mais tipos, zero custo).
- Retorne tipos owned (`String`, `Vec<T>`) quando o valor e produzido pela funcao; empreste na entrada, possua na saida.
- `Arc<T>` para compartilhar ownership entre tarefas/threads; `Arc::clone` e barato (incrementa um contador). Use `Arc<dyn Trait>` para dependencias injetadas (ver secao 6).

### Lifetimes — o basico

- Lifetimes descrevem **por quanto tempo uma referencia e valida**; na maioria dos casos sao inferidos (lifetime elision). Anote apenas quando o compilador pedir.
- `'static` significa "vive por toda a duracao do programa" (ou e owned). Tarefas de `tokio::spawn` exigem futures `'static` — por isso passamos dados owned ou `Arc`, nunca `&` de escopo local.
- Evite structs cheias de referencias com lifetimes no dominio; prefira dados owned. Guarde referencias apenas em tipos efemeros (extractors, iteradores).

## 3. Tratamento de Erros

Referencia: [The Book — Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) · [thiserror](https://docs.rs/thiserror) · [anyhow](https://docs.rs/anyhow)

Rust nao tem exceptions. Falhas recuperaveis sao valores: `Result<T, E>`. Falhas irrecuperaveis sao `panic!` — proibidas no caminho de producao (ver secao 4).

### A divisao canonica do projeto: `thiserror` no dominio, `anyhow` na propagacao

| Camada | Tipo de erro | Por que |
|---|---|---|
| `domain` (use cases) | `enum DomainError` com `thiserror` | Erros de negocio **tipados e exaustivos**; o controller mapeia cada variante para um HTTP status |
| `application` (impl UC) | retorna `Result<_, DomainError>` | Erro de negocio explicito; erro de infra vira `Unexpected` via `?` |
| `infrastructure`/`repository`/`pkg` | `anyhow::Result<T>` | Erro opaco de infra, **enriquecido com contexto** via `.context(...)` |

### Erros de dominio com `thiserror`

Cada app define **um** enum `DomainError` em `domain/entity/errors.rs`. A variante `Unexpected` faz a ponte com erros de infraestrutura propagados via `anyhow`:

```rust
// apps/user/src/domain/entity/errors.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("invalid input")]
    InvalidInput,
    #[error("user not found")]
    UserNotFound,
    #[error("email already in use")]
    UserAlreadyExists,
    #[error("forbidden")]
    Forbidden,
    // Ponte com a infraestrutura: qualquer anyhow::Error vira Unexpected via `?`.
    #[error(transparent)]
    Unexpected(#[from] anyhow::Error),
}
```

- `#[error("...")]` gera o `impl Display` — mensagens em **minuscula, sem ponto final**, identificando a falha (`"user not found"`), como no Go.
- `#[from]` gera o `impl From<anyhow::Error> for DomainError`, permitindo `?` transparente da infra para o dominio.
- `#[error(transparent)]` delega `Display`/`source` ao erro embrulhado (nao inventa mensagem propria).
- Erros que embrulham outro erro devem expor a origem via `#[source]` (ou `#[from]`), preservando a cadeia (equivalente ao `%w` do Go).

### Propagacao de infra com `anyhow` + `.context()`

Na infraestrutura, retorne `anyhow::Result<T>` e **sempre** adicione contexto na borda de cada operacao de I/O:

```rust
use anyhow::{Context, Result};

async fn find_by_email(&self, email: &str) -> Result<Option<User>> {
    let row: Option<UserRow> = sqlx::query_as(QUERY)
        .bind(email)
        .fetch_optional(&self.pool)
        .await
        .context("find user by email")?;   // contexto de diagnostico, sem vazar detalhes ao cliente
    Ok(row.map(Into::into))
}
```

- `.context("acao")` anexa uma mensagem legivel; `.with_context(|| format!(...))` quando a mensagem precisa de dados (avalia o closure so no erro).
- **Nunca** descarte o erro original com `.map_err(|_| ...)` sem motivo — perde a cadeia de diagnostico. Prefira `.context(...)`.
- O `?` propaga o erro convertendo o tipo automaticamente via `From` (por isso `anyhow::Error -> DomainError::Unexpected` "simplesmente funciona").

### O operador `?` e o caminho feliz

```rust
// ✅ `?` propaga o erro; o caminho feliz flui verticalmente, sem aninhamento
async fn perform(&self, input: CreateInput) -> Result<CreateOutput, DomainError> {
    let existing = self.user_repository
        .find_by_email(&input.email)
        .await
        .context("find by email")?;          // anyhow -> DomainError::Unexpected
    if existing.is_some() {
        return Err(DomainError::UserAlreadyExists);   // erro de negocio explicito
    }

    let mut user = User::new(input.name, input.email);
    self.user_repository.create(&mut user).await.context("create user")?;
    let _ = self.user_event.publish_created(&user).await;   // evento apos persistir; falha nao desfaz negocio
    Ok(CreateOutput { user })
}
```

- Erro de **negocio** -> variante explicita (`Err(DomainError::UserAlreadyExists)`).
- Erro de **infra** -> `.context("...")?` (vira `Unexpected`).
- Como no Go, use early return com `?` e deixe o caminho feliz alinhado a esquerda — nada de `if let Ok(...) { ... } else { ... }` aninhado.

### Inspecionando erros: `matches!`, `map_err`

```rust
// Testar variante especifica (equivalente ao errors.Is do Go)
if matches!(err, DomainError::UserAlreadyExists) {
    // ...
}

// Extrair/transformar antes de propagar (equivalente pontual ao %v vs %w)
let order = order_repo
    .find_by_id(id)
    .await
    .map_err(DomainError::from)?     // anyhow -> DomainError quando o From nao e inferido
    .ok_or(DomainError::UserNotFound)?;   // Option -> Result: ausencia vira erro NO CALLER
```

- `matches!(valor, Padrao)` para checagem booleana de variante — mais legivel que `if let` quando so precisa do `bool`.
- `map_err(...)` para converter o tipo do erro; `ok_or(...)`/`ok_or_else(...)` para transformar `Option::None` em `Err`.
- Em `match` sobre `DomainError`, seja **exaustivo**: ao adicionar uma variante, o compilador obriga a tratar o novo caso (o mapeamento de HTTP status em `infrastructure/controller/errors.rs` depende disso).

### Nunca vaze detalhes de 5xx

O `ApiError` do `pkg` ja converte `DomainError` -> HTTP status e **oculta** a mensagem de erros 5xx (`"internal server error"`), logando o detalhe internamente. Nao reimplemente isso no app — apenas propague com `?` e mapeie as variantes em `HttpDomainError`.

## 4. `Option` e a Ausencia de `null`

Referencia: [`Option` — std](https://doc.rust-lang.org/std/option/enum.Option.html)

Rust nao tem `null`. Ausencia e `Option<T>` (`Some(x)` / `None`) — verificada em compile time.

| Situacao | Tipo/valor |
|---|---|
| Consulta que nao encontrou (`find_by_id`) | retorna `Ok(None)` — **ausencia nao e erro** |
| Parametro opcional | `Option<T>` (default `None`) |
| Coluna anulavel no banco | `Option<T>` na entidade |

```rust
// Repositorio: ausencia e Ok(None), nunca Err
async fn find_by_id(&self, id: Uuid) -> Result<Option<User>> {
    let row: Option<UserRow> = sqlx::query_as(QUERY)
        .bind(id)
        .fetch_optional(&self.pool)   // fetch_optional -> Ok(None) quando nao ha linha
        .await
        .context("find user by id")?;
    Ok(row.map(Into::into))
}

// Caller decide se ausencia e erro NO SEU contexto:
let user = self.user_repository.find_by_id(id).await?
    .ok_or(DomainError::UserNotFound)?;
```

### `unwrap()`/`expect()`/`panic!` — proibidos em producao

- **Nunca** use `.unwrap()` ou `.expect()` em `Result`/`Option` no caminho de producao — um `None`/`Err` inesperado derruba a tarefa (ou o processo). Propague com `?` ou trate explicitamente.
- Excecoes toleradas: **testes** (`#[cfg(test)]`), invariantes verdadeiramente impossiveis (documente com `expect("motivo da invariante")`), e o `main` (`-> anyhow::Result<()>` deixa o `?` reportar e sair). No `main`, `install ctrl_c handler`/`install sigterm handler` usam `.expect(...)` por serem falhas de setup irrecuperaveis.
- Combinadores em vez de `unwrap`: `unwrap_or(default)`, `unwrap_or_else(|| ...)`, `unwrap_or_default()`, `map(...)`, `and_then(...)`, `?`.
- **Nunca** indexe slices/vetores sem certeza (`v[i]` pode dar panic) — use `.get(i) -> Option`.
- Aritmetica que pode estourar: prefira `checked_*`/`saturating_*` quando o overflow e possivel e relevante.

## 5. Concorrencia com Tokio (async/await)

Referencia: [Tokio Tutorial](https://tokio.rs/tokio/tutorial) · [Async Book](https://rust-lang.github.io/async-book/)

O runtime e **Tokio multi-thread** (`tokio = { version = "1", features = ["full"] }`). `async fn` retorna uma `Future` **preguicosa** — nada executa ate ser `.await`ada ou agendada.

### `.await` toda future; nunca deixe uma future "solta"

```rust
// ❌ future criada e ignorada — nada acontece (o compilador emite `unused must_use`)
self.repository.create(&mut user);

// ✅ aguarde
self.repository.create(&mut user).await?;
```

- Uma `Future` nao aguardada nem agendada e um bug (equivalente a goroutine que nunca roda). O lint `unused_must_use` avisa — trate-o como erro.

### `tokio::spawn` — toda tarefa precisa de caminho de termino

```rust
// Tarefa de background: recebe dados OWNED ou Arc (future deve ser 'static + Send)
let worker = tokio::spawn({
    let subscriber = subscriber.clone();   // Arc clone — barato
    async move {
        while let Some(msg) = messages.next().await {
            let Ok(msg) = msg else { continue };
            subscriber.handle_user_created(msg).await;
        }
    }
});

// No shutdown, encerre explicitamente (equivalente a evitar "goroutine leak")
worker.abort();
```

- A future de `spawn` deve ser `'static` e `Send`: capture dados owned (`move`) ou `Arc`, nunca `&` de escopo local.
- **Toda tarefa deve ter um caminho claro de termino** (fim natural do loop, `abort()`, cancelamento via `select!`) — tarefa orfa e leak.
- `JoinHandle::abort()` no shutdown; para varias tarefas, prefira `JoinSet` para aguardar/cancelar em grupo.

### `tokio::select!` — corrida entre futures e cancelamento

```rust
// Espera por multiplos sinais; a primeira a completar vence e as outras sao dropadas (canceladas)
tokio::select! {
    _ = ctrl_c => {},
    _ = terminate => {},
}
```

- `select!` implementa cancelamento: ao completar um ramo, os demais futures sao dropados. Garanta que suas futures sejam **cancel-safe** (nao deixem estado corrompido ao serem dropadas no meio).
- Padrao canonico de graceful shutdown do projeto: `select!` entre `ctrl_c()` e `SIGTERM`, depois cleanup na **ordem inversa** da criacao (HTTP para -> NATS drain -> `pool.close().await`).

### Paralelismo estruturado

```rust
// ✅ operacoes independentes em paralelo (equivale ao errgroup/gather)
let (user, orders) = tokio::try_join!(
    user_repository.find_by_id(id),
    order_repository.list_by_user(id),
)?;

// futures::future::join_all para um Vec de futures homogeneas
let results = futures::future::join_all(ids.into_iter().map(|id| repo.find_by_id(id))).await;
```

- `tokio::join!` aguarda todas (sem short-circuit); `tokio::try_join!` aborta na primeira `Err`.
- Timeouts em toda I/O externa: `tokio::time::timeout(Duration::from_secs(5), fut).await`.

### Nunca bloqueie o runtime

O executor async atende muitas tarefas por thread; uma chamada **bloqueante** trava o worker inteiro.

- **Proibido** em codigo async: `std::thread::sleep` (use `tokio::time::sleep`), I/O de arquivo sincrono, chamadas de rede sincronas, CPU-bound longo, `hash` de senha direto no handler.
- CPU-bound ou lib sincrona inevitavel -> `tokio::task::spawn_blocking(move || ...)` (roda numa thread pool separada).
- Use sempre drivers **async**: `sqlx` (Postgres), `async-nats`, `reqwest`/`hyper` (HTTP) — nunca um driver sincrono no caminho async.

### Estado compartilhado

- Comunicacao entre tarefas: canais `tokio::sync::mpsc`/`oneshot`/`broadcast` ("share memory by communicating").
- Estado mutavel compartilhado: `Arc<Mutex<T>>`/`Arc<RwLock<T>>` do **`tokio::sync`** (nao os da std) quando o lock e mantido atravessando um `.await`. Mantenha secoes criticas curtas; nunca segure um lock sincrono da std atravessando `.await`.
- O borrow checker + `Send`/`Sync` eliminam data races em compile time — mas deadlocks logicos ainda sao possiveis; mantenha ordem de aquisicao de locks consistente.

## 6. Traits e Generics

Referencia: [The Book — Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) · [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)

Traits sao a abstracao central — o equivalente das interfaces do Go. O projeto usa traits para **inversao de dependencia**: o dominio define o contrato, a infraestrutura implementa.

### Traits de dominio: pequenos, focados, com `Send + Sync`

```rust
// apps/user/src/domain/repository/user_repository.rs
use anyhow::Result;
use async_trait::async_trait;
use uuid::Uuid;

use crate::domain::entity::User;

#[cfg_attr(test, mockall::automock)]   // gera MockUserRepository nos testes
#[async_trait]                          // permite `async fn` em trait para uso com dyn
pub trait UserRepository: Send + Sync {
    async fn create(&self, user: &mut User) -> Result<()>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>>;
    async fn find_by_email(&self, email: &str) -> Result<Option<User>>;
}
```

- **`Send + Sync`** e obrigatorio em todo trait injetado via `Arc<dyn Trait>` — permite compartilhar a dependencia entre tarefas Tokio (multi-thread).
- **`#[async_trait]`** e necessario para metodos `async` em traits usados como `dyn` (object-safe). Em extractors do Axum 0.8 (`FromRequest`/`FromRequestParts`) use `async fn` **nativo**, sem `#[async_trait]`.
- **`#[cfg_attr(test, mockall::automock)]`** SEMPRE antes de `#[async_trait]` — gera o mock (`Mock<Nome>`) apenas em build de teste.
- Prefira traits pequenos e coesos (Interface Segregation): 1 use case = 1 trait com 1 metodo `perform`. Facilita mocks granulares e mantem contratos enxutos.

### Injecao de dependencia: `Arc<dyn Trait>`

```rust
pub struct CreateUseCase {
    user_repository: Arc<dyn UserRepository>,   // depende do TRAIT, nunca do tipo concreto
    user_event: Arc<dyn UserEvent>,
}

impl CreateUseCase {
    pub fn new(user_repository: Arc<dyn UserRepository>, user_event: Arc<dyn UserEvent>) -> Self {
        Self { user_repository, user_event }
    }
}
```

- Campos de use case/controller sao **traits de dominio** via `Arc<dyn Trait>` — nunca tipos concretos (a excecao sao repositorios concretos quando o handler orquestra transaction, ver `architecture.md`).
- O **composition root** (`src/bin/*.rs`) constroi bottom-up e embrulha cada implementacao em `Arc::new(...)`. O compilador garante em compile time que nenhuma dependencia falta.

### Static dispatch (generics) vs dynamic dispatch (`dyn`)

- **Generics / `impl Trait`** (static dispatch): monomorfizado, sem custo em runtime. Prefira em codigo quente e APIs internas: `fn total(items: &[impl Priced]) -> i64`.
- **`dyn Trait`** (dynamic dispatch): resolucao via vtable, permite heterogeneidade e reduz codigo gerado. Use para **injecao de dependencia** (`Arc<dyn Trait>`) e coleções de tipos diferentes.
- Regra pratica: "aceite `impl Trait`/generic na entrada quando possivel; use `dyn` quando precisa guardar/injetar a abstracao".

### Trait objects precisam ser object-safe

- Um trait so vira `dyn Trait` se for object-safe (sem metodos genericos, sem `Self` por valor no retorno, etc.). `#[async_trait]` contorna a limitacao de `async fn` em `dyn`.

## 7. Structs, Enums e Derives

Referencia: [Rust API Guidelines — Interoperability](https://rust-lang.github.io/api-guidelines/interoperability.html)

### Derives comuns

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]   // JSON em camelCase; struct em snake_case
pub struct User {
    pub id: Uuid,
    pub name: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

- **`Debug`** em praticamente todo tipo publico (Rust API Guidelines C-DEBUG) — indispensavel para logs e testes.
- **`Clone`** quando o valor precisa ser duplicado; nao derive por reflexo (clone deve ser intencional).
- **`Serialize`/`Deserialize`** (serde) em entidades e DTOs que cruzam a fronteira (HTTP, eventos NATS) — com `#[serde(rename_all = "camelCase")]`.
- **`ToSchema`** (utoipa) em tipos expostos no OpenAPI.
- `PartialEq`/`Eq`/`Hash` quando o tipo e comparado ou usado como chave; `Default` quando ha um valor "vazio" natural.
- **Enums fechados** (serde) para enumeracoes de dominio (`enum Currency { BRL, USD, EUR }`) — a deserializacao rejeita valores fora do enum (equivale ao `oneof`).

### Construtores e entidades

- Toda entidade tem `pub fn new(...) -> Self`. Timestamps em UTC (`Utc::now()`); `id: Uuid::nil()` ate o banco devolver o ID gerado (`RETURNING id`).
- Row structs (`#[derive(sqlx::FromRow)]`) sao **privadas** na infraestrutura, com `From<Row> for Entity` — o dominio nunca deriva `FromRow` (ver `postgres-best-practices`).

## 8. Iteradores e Closures

Referencia: [The Book — Iterators](https://doc.rust-lang.org/book/ch13-02-iterators.html)

Iteradores sao preguicosos, zero-cost e mais idiomaticos que loops manuais com indice.

```rust
// ✅ transformacao declarativa (row -> entity), sem indice manual
let users: Vec<User> = rows.into_iter().map(Into::into).collect();

// ✅ filtro + soma
let pending_total: i64 = orders.iter()
    .filter(|o| o.status == "pending")
    .map(|o| o.total_cents)
    .sum();

// ✅ paginacao por cursor: usa o ultimo id como proximo cursor
let next_cursor = users.last().map(|u| u.id);
```

- Prefira combinadores (`map`, `filter`, `filter_map`, `fold`, `any`, `all`, `find`, `collect`) a loops `for` com `push` manual.
- `into_iter()` consome (dono), `iter()` empresta `&T`, `iter_mut()` empresta `&mut T` — escolha conforme a ownership desejada.
- Colete em `Result` para curto-circuitar em erro: `items.iter().map(parse).collect::<Result<Vec<_>, _>>()?`.
- Nao force iteradores onde um `for` simples e mais claro; legibilidade primeiro.

## 9. `unsafe` — evitar

Referencia: [The Rustonomicon](https://doc.rust-lang.org/nomicon/)

- **Nao use `unsafe`** em codigo de aplicacao neste projeto. As garantias de seguranca de memoria e ausencia de data races do Rust dependem disso.
- Praticamente todo caso de uso legitimo (FFI, otimizacao de baixo nivel) ja esta encapsulado em crates auditados do ecossistema — dependa deles, nao reimplemente.
- Se, em caso extremo, `unsafe` for inevitavel: isole no menor bloco possivel, documente a invariante de seguranca com `// SAFETY: ...`, e cubra com testes. Isso deve ser raro e revisado com cuidado.

## 10. Ferramentas

| Ferramenta | Papel | Regra |
|---|---|---|
| `cargo fmt` | Formatacao | `cargo fmt --check` no CI; sem debate de estilo. Preserva os 3 blocos de `use` (std -> third-party -> crate/pkg) |
| `cargo clippy` | Lint | `cargo clippy --all-targets --all-features -- -D warnings` — **zero warnings**; lints so silenciados com `#[allow(...)]` justificado |
| `cargo build` / `cargo check` | Compilacao | Deve compilar limpo; warnings do compilador tratados como erro no CI |
| `cargo test` | Testes | Unit `#[cfg(test)] mod tests` (mockall) + integracao em `tests/` (testcontainers, `#[tokio::test]`) |
| `cargo audit` | Vulnerabilidades | Auditoria de dependencias no CI |
| `tracing` | Logging JSON estruturado | Nunca `println!`/`dbg!` em codigo de aplicacao; use `tracing::{info,warn,error}` com campos |

- `cargo clippy -D warnings` e gate de CI. Corrija a causa; `#[allow(clippy::...)]` apenas pontual e com comentario.
- Organizacao de `use` em 3 blocos separados por linha em branco: **std**, **third-party**, **projeto (`pkg` + `crate`)** — o rustfmt preserva a ordem.
- `edition = "2024"` (MSRV 1.85) em todos os crates do workspace.

## Checklist de Revisao

Ao escrever ou revisar codigo Rust, verifique:

- [ ] `snake_case` para arquivos/funcoes/variaveis, `PascalCase` para tipos/traits, `SCREAMING_SNAKE_CASE` para constantes
- [ ] Acronimos como uma palavra em PascalCase (`ApiError`, `HttpDomainError`)
- [ ] Traits **sem** prefixo `I`; impl **sem** sufixo `Impl` (impl concreta usa a tecnologia: `PostgresUserRepository`)
- [ ] Emprestimo (`&T`/`&str`/`&[T]`) na entrada; `.clone()` apenas quando intencional
- [ ] `Arc<dyn Trait>` para dependencias injetadas; `Arc::clone` no lugar de reconstruir
- [ ] Erro de dominio como `enum DomainError` (`thiserror`); infra propaga `anyhow::Result` com `.context("acao")?`
- [ ] `Unexpected(#[from] anyhow::Error)` presente; `?` converte infra -> dominio
- [ ] Erro de negocio = variante explicita; erro de infra = `.context(...)?`
- [ ] Caminho feliz alinhado a esquerda com `?`; sem `if let Ok`/`else` aninhado
- [ ] `matches!` para checar variante; `ok_or`/`map_err` nas conversoes
- [ ] **Nenhum `unwrap()`/`expect()`/`panic!`/indexacao** no caminho de producao (so testes/invariantes documentadas)
- [ ] Ausencia = `Ok(None)` no repositorio; caller decide se vira erro (`.ok_or(...)`)
- [ ] Toda `Future` e `.await`ada ou agendada; nenhum `unused_must_use`
- [ ] `tokio::spawn` com dados owned/`Arc` e caminho de termino (`abort`/`select!`) — sem leak de tarefa
- [ ] Nenhuma chamada bloqueante no runtime; CPU-bound em `spawn_blocking`; drivers async
- [ ] Timeout em toda I/O externa; graceful shutdown via `select!` + cleanup em ordem inversa
- [ ] Traits de dominio com `Send + Sync`, `#[async_trait]` e `#[cfg_attr(test, mockall::automock)]`
- [ ] Derives adequados (`Debug` em todo tipo publico; `Serialize/Deserialize` + `#[serde(rename_all = "camelCase")]` na fronteira)
- [ ] Iteradores/combinadores no lugar de loops manuais quando mais claro
- [ ] Sem `unsafe` em codigo de aplicacao
- [ ] `cargo fmt --check`, `cargo clippy -- -D warnings` e `cargo test` passam limpos

## Fontes

Toda recomendacao acima vem destas fontes — consulte-as quando houver duvida:

- https://doc.rust-lang.org/book/ — The Rust Programming Language (The Book)
- https://doc.rust-lang.org/reference/ — The Rust Reference
- https://rust-lang.github.io/api-guidelines/ — Rust API Guidelines (naming, interoperability)
- https://doc.rust-lang.org/nomicon/ — The Rustonomicon (unsafe)
- https://rust-lang.github.io/async-book/ — Asynchronous Programming in Rust
- https://tokio.rs/tokio/tutorial — Tokio (async runtime, spawn, select, shutdown)
- https://docs.rs/thiserror — thiserror (erros de dominio)
- https://docs.rs/anyhow — anyhow (propagacao com contexto)
- https://doc.rust-lang.org/clippy/ — Clippy (lints)
