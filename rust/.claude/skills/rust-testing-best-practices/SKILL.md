---
name: rust-testing-best-practices
description: Aplica melhores praticas de testes em Rust cobrindo a piramide completa — unitarios no proprio arquivo com `#[cfg(test)] mod tests`, `#[tokio::test]` para async, dublês com mocks gerados por mockall (`#[cfg_attr(test, mockall::automock)]`, `MockNome`) e fakes que implementam o trait de dominio, fixtures e parametrizacao com rstest (`#[case]`), asserts com `assert_eq!`/`assert!`/`matches!`, integracao e E2E com dependencias reais em containers (testcontainers + testcontainers-modules: Postgres, NATS), migrations aplicadas via sqlx, isolamento por TRUNCATE ou schema por teste, E2E de API subindo o Axum em porta livre e chamando com reqwest, cobertura com cargo-llvm-cov e mutation testing com cargo-mutants. Use sempre que escrever, revisar ou refatorar testes Rust (`#[test]`, `#[tokio::test]`, `mod tests`, `apps/<app>/tests/*`), ao criar mocks com mockall, montar fixtures com rstest, subir dependencias reais via testcontainers, definir cenarios de aceitacao ou discutir cobertura e mutantes. Data races sao prevenidas em compile time pelo ownership.
---

# Rust Testing Best Practices (mockall · rstest · testcontainers · cargo-llvm-cov · cargo-mutants)

Skill baseada na documentacao oficial:

- [doc.rust-lang.org — Writing Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) e [Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html)
- [docs.rs/mockall](https://docs.rs/mockall/) (mocks de traits com `automock`/`MockNome`)
- [docs.rs/rstest](https://docs.rs/rstest/) (fixtures + casos parametrizados)
- [docs.rs/testcontainers](https://docs.rs/testcontainers/) + [testcontainers-modules](https://docs.rs/testcontainers-modules/)
- [tokio.rs — Testing](https://tokio.rs/tokio/topics/testing) (`#[tokio::test]`)
- [github.com/taiki-e/cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) e [github.com/sourcefrog/cargo-mutants](https://github.com/sourcefrog/cargo-mutants)

Toda recomendacao aqui e a forma idiomatica prescrita pelas fontes oficiais, adaptada a arquitetura do projeto (Clean Architecture, DI por composition root manual com `Arc<dyn Trait>`, 1 use case = 1 trait = metodo `perform`, mockall gerando `Mock<Nome>`). Combine com [[postgres-best-practices]] e [[nats-best-practices]] para as camadas de banco e mensageria.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer teste Rust (`#[test]`, `#[tokio::test]`, `#[cfg(test)] mod tests`, `apps/<app>/tests/*`)
- Ao gerar mocks com mockall (`#[cfg_attr(test, mockall::automock)]`) ou escrever fakes tipados
- Ao desenhar a estrategia de testes (unit, aceitacao, integracao, E2E)
- Ao subir dependencias reais (Postgres, NATS) via testcontainers
- Ao parametrizar casos com rstest, discutir cobertura (cargo-llvm-cov) ou mutation testing (cargo-mutants)

## 0. A Piramide de Testes em Rust

| Camada | Velocidade | Escopo | Ferramenta principal |
|--------|-----------|--------|----------------------|
| **Unitario** | µs a ms | Use case/entidade isolados, sem I/O | `#[cfg(test)] mod tests` + `mockall` |
| **Aceitacao** | ~ms | Regra de negocio via composicao com **fakes** in-memory | `#[tokio::test]` + fakes |
| **Integracao** | ~100 ms | Repositorio/publisher contra dependencia **real** em container | `testcontainers` |
| **E2E de API** | s | HTTP -> controller -> use case -> banco real, via `reqwest` | `testcontainers` + Axum em porta livre |

Regras inegociaveis:
- Muitos unitarios -> poucos aceitacao -> menos integracao -> minimo de E2E.
- Cada camada acima e mais lenta e mais fragil. Suba o que **realmente** precisa subir.
- Toda camada usa `cargo test` — nao ha framework paralelo. Diferencie por **localizacao** (in-file vs `tests/`) e, quando necessario, por `feature`/`#[ignore]`, nunca por runner separado.
- **Data races nao existem aqui**: o ownership + `Send`/`Sync` do Rust as previne em compile time. Testes de concorrencia focam em **logica** (ordem, atomicidade de tx), nao em corrida de memoria.

## 1. Convencoes Gerais para Testes em Rust

### Layout de arquivos por camada

| Camada | Localizacao | Forma |
|--------|-------------|-------|
| Unitario / Aceitacao | no **proprio arquivo** da implementacao | `#[cfg(test)] mod tests` |
| Integracao / E2E | `apps/<app>/tests/<name>_integration_test.rs` | crate de teste separado (so API publica) |
| Helpers de integracao | `apps/<app>/tests/testhelper/` | `mod.rs` + `postgres.rs` + `nats.rs` (`OnceCell`) |

- **Unitarios vivem junto do codigo**: o `mod tests` no fim do `create_usecase.rs` enxerga tudo do modulo pai via `use super::*;` (inclusive itens privados). E o padrao do projeto (ver `architecture.md`).
- **Integracao vive em `tests/`**: cada arquivo em `apps/<app>/tests/` compila como um crate separado e so acessa a **API publica** do app (`user::infrastructure::repository::PostgresUserRepository`, etc). E o equivalente ao black-box.

### `#[cfg(test)]` — codigo de teste fora do binario de release

```rust
// no fim de src/application/usecase/user/create_usecase.rs
#[cfg(test)]
mod tests {
    use super::*;

    // mocks, fixtures e #[test]/#[tokio::test] aqui
}
```

`#[cfg(test)]` garante que o modulo (e os `Mock*` gerados pelo mockall via `#[cfg_attr(test, ...)]`) **so** existem em `cargo test` — zero custo no binario de producao.

### Nomenclatura

- Funcao de teste: `#[test] fn nome_descritivo()` ou `#[tokio::test] async fn ...` para async.
- Nome descreve **comportamento em linguagem de dominio**: `perform_persists_user_and_returns_id`, `perform_returns_already_exists_when_email_taken`. Evite `test1`, `test_create` (descreve metodo, nao regra).
- Helpers de teste: funcoes livres dentro do `mod tests` (`fn make_sut() -> ...`).

### Async: `#[tokio::test]`

Como todo trait do dominio e `#[async_trait]`, a maioria dos testes e async:

```rust
#[tokio::test]
async fn perform_persists_user() {
    // ...
}
```

- `#[tokio::test]` cria um runtime Tokio por teste (single-thread por padrao). Para exercitar concorrencia real: `#[tokio::test(flavor = "multi_thread", worker_threads = 2)]`.
- **Nunca** `tokio::time::sleep` real para sincronizar em unitario — controle o tempo (`tokio::time::pause()`/`advance`) ou espere por condicao.

### AAA (Arrange, Act, Assert)

Separe os tres blocos com uma linha em branco, sem comentarios redundantes:

```rust
#[tokio::test]
async fn perform_returns_output_with_id() {
    let (sut, repo) = make_sut();

    let output = sut.perform(CreateInput { name: "Ana".into(), email: "ana@example.com".into() }).await.unwrap();

    assert!(!output.user.id.is_nil());
    assert!(repo.find_by_id(output.user.id).await.unwrap().is_some());
}
```

### Falhas: `panic!` vs `Result` no teste

- Um teste falha quando **entra em panic** (`assert!` falho, `unwrap()`/`expect()` em `Err`, `panic!`). Nao ha `t.Errorf`/`t.Fatalf` — cada assert que falha aborta aquele teste.
- **Pre-condicao** (setup nao pode falhar): `.expect("mensagem de setup")` deixa claro o que quebrou.
- **Verificacao de saida**: `assert_eq!`/`assert!` com mensagem opcional.
- Teste pode retornar `Result<(), E>` e usar `?` no lugar de `unwrap` no caminho de setup:

```rust
#[tokio::test]
async fn find_by_id_returns_persisted() -> anyhow::Result<()> {
    let repo = make_repo();
    let mut user = User::new("Ana".into(), "ana@example.com".into());
    repo.create(&mut user).await?;

    let found = repo.find_by_id(user.id).await?;

    assert_eq!(found.map(|u| u.email), Some("ana@example.com".to_string()));
    Ok(())
}
```

## 2. Testes Unitarios (`#[cfg(test)] mod tests` + rstest)

Referencia: https://doc.rust-lang.org/book/ch11-01-writing-tests.html

Testam use case/entidade **isolados**, sem I/O. Dependencias entram pelo construtor como `Arc<dyn Trait>`; no teste, injete mocks (mockall) ou fakes — **nunca** container.

### Asercoes essenciais (use estas; evite reinvencoes)

| Cenario | Use |
|---------|-----|
| Igualdade | `assert_eq!(got, want)` / `assert_ne!` |
| Booleano | `assert!(cond)` — mas prefira `assert_eq!` quando comparar valores |
| Variante de erro/enum | `assert!(matches!(err, DomainError::UserAlreadyExists))` |
| `Option`/`Result` de sucesso | `assert!(x.is_some())` / `let v = r.unwrap();` |
| Erro esperado | `let err = r.unwrap_err(); assert!(matches!(err, ...))` |
| Colecoes | `assert_eq!(v.len(), 3)`, `assert!(v.contains(&x))`, `assert!(v.is_empty())` |
| Ponto flutuante | comparar com epsilon: `assert!((a - b).abs() < 1e-9)` |

Regras:
- **`matches!` para erros/enums**, nunca comparar strings de `Display`. `matches!(err, DomainError::UserNotFound)` documenta a variante e sobrevive a mudanca de mensagem.
- Nao use `assert!(a == b)` — perde o diagnostico. Use `assert_eq!(a, b)`, que imprime `left`/`right` no panic.
- Sempre asserte o **valor exato** (nao so `is_ok()`), senao o mutation testing (secao 6) acusa asercao fraca.

### Table-driven com rstest (`#[case]`)

Referencia: https://docs.rs/rstest/

Sempre que houver >= 3 casos da mesma regra, parametrize com `rstest` em vez de duplicar `#[test]`:

```rust
#[cfg(test)]
mod tests {
    use rstest::rstest;

    use super::*;

    #[rstest]
    #[case("ana@example.com", true)]
    #[case("sem-arroba.com", false)]
    #[case("", false)]
    #[case("a@b.co", true)]
    fn is_valid_email_classifies(#[case] input: &str, #[case] expected: bool) {
        assert_eq!(is_valid_email(input), expected);
    }
}
```

- Cada `#[case]` vira um subteste independente e nomeado (`..::case_1`, `case_2`, ...); filtre com `cargo test is_valid_email_classifies::case_3`.
- Cada linha e **independente** — nenhum caso depende do estado deixado por outro.
- Para casos async, combine com `#[tokio::test]`:

```rust
#[rstest]
#[case(0, false)]
#[case(1, true)]
#[tokio::test]
async fn perform_charges_only_positive_amounts(#[case] cents: i64, #[case] should_charge: bool) {
    // ...
}
```

### Fixtures com rstest (`#[fixture]`)

Para montar o SUT com defaults reaproveitaveis:

```rust
use rstest::{fixture, rstest};

#[fixture]
fn repo() -> InMemoryUserRepository {
    InMemoryUserRepository::default()
}

#[rstest]
#[tokio::test]
async fn perform_persists_user(repo: InMemoryUserRepository) {
    // `repo` injetado pela fixture
}
```

### Factory de SUT (Arrange explicito)

Alternativa a fixture quando o teste precisa **acessar** as dependencias no Assert (ex: inspecionar o fake). Retorne o SUT + as dependencias:

```rust
fn make_sut() -> (CreateUseCase, Arc<InMemoryUserRepository>, Arc<SpyUserEvent>) {
    let repo = Arc::new(InMemoryUserRepository::default());
    let event = Arc::new(SpyUserEvent::default());
    let sut = CreateUseCase::new(repo.clone(), event.clone());
    (sut, repo, event)
}
```

- Defaults sempre **validos**; o teste sobrescreve apenas o relevante.
- `email`/`id` aleatorios por default (ex: `Uuid::now_v7()`) evitam colisao de indice unico quando o fake vira container na integracao.

### Testando propagacao de erro de infraestrutura

O use case converte erro de infra (anyhow) em `DomainError::Unexpected` via `.context(...)?`. Teste esse caminho injetando um mock que falha:

```rust
#[tokio::test]
async fn perform_maps_repo_failure_to_unexpected() {
    let mut repo = MockUserRepository::new();
    repo.expect_find_by_email().returning(|_| Err(anyhow::anyhow!("db down")));
    let sut = CreateUseCase::new(Arc::new(repo), Arc::new(MockUserEvent::new()));

    let err = sut.perform(CreateInput { name: "Ana".into(), email: "ana@example.com".into() })
        .await
        .unwrap_err();

    assert!(matches!(err, DomainError::Unexpected(_)));
}
```

## 3. Test Doubles: mocks com mockall e fakes tipados

Referencia oficial: https://docs.rs/mockall/

O dominio expoe traits (`UserRepository`, `UserEvent`, `CreateUseCase`, `Hasher`...). Cada trait ja carrega, no proprio arquivo, o atributo que habilita o mock:

```rust
// apps/user/src/domain/repository/user_repository.rs
#[cfg_attr(test, mockall::automock)]   // SEMPRE antes de #[async_trait]
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn create(&self, user: &mut User) -> Result<()>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>>;
    async fn find_by_email(&self, email: &str) -> Result<Option<User>>;
}
```

`#[cfg_attr(test, mockall::automock)]` gera, **apenas em `cfg(test)`**, uma struct `MockUserRepository` com um `expect_<metodo>()` por metodo. A ordem `automock` **antes** de `#[async_trait]` e obrigatoria (mockall precisa ver a assinatura async original).

| Tecnica | Quando usar |
|---------|-------------|
| **Mock (mockall)** | Verificar interacao pontual (foi chamado? com quais args? quantas vezes?) e programar retornos |
| **Fake tipado** (implementa o trait) | Padrao para aceitacao — comportamento realista in-memory, sobrevive a refactor |

### Mock com mockall — estrutura padrao

```rust
#[cfg(test)]
mod tests {
    use std::sync::Arc;

    use mockall::predicate::*;

    use super::*;
    use crate::domain::repository::MockUserRepository;
    use crate::domain::event::MockUserEvent;

    #[tokio::test]
    async fn perform_returns_already_exists_when_email_taken() {
        let mut repo = MockUserRepository::new();
        repo.expect_find_by_email()
            .with(eq("ana@example.com"))          // matcher do argumento
            .times(1)
            .returning(|_| Ok(Some(User::new("Ana".into(), "ana@example.com".into()))));
        repo.expect_create().never();              // nunca cria se ja existe

        let event = MockUserEvent::new();          // sem expectativa => nunca chamado

        let sut = CreateUseCase::new(Arc::new(repo), Arc::new(event));

        let err = sut
            .perform(CreateInput { name: "Ana".into(), email: "ana@example.com".into() })
            .await
            .unwrap_err();

        assert!(matches!(err, DomainError::UserAlreadyExists));
    }
}
```

### Programando expectativas

| Quando o metodo deve... | Use |
|-------------------------|-----|
| Retornar valor fixo | `.returning(|_| Ok(...))` |
| Retornar uma unica vez (mover valor) | `.return_once(|_| Ok(...))` |
| Ser chamado exatamente N vezes | `.times(N)` |
| Nunca ser chamado | `.never()` (ou simplesmente nao declarar) |
| Qualquer numero de vezes | (default do mockall e "qualquer numero"; use `.times(n)` para fixar) |
| Validar arg e computar retorno | `.returning(|arg| { ...; Ok(...) })` |
| Preencher out-param (`&mut`) | `.returning(|user: &mut User| { user.id = Uuid::now_v7(); Ok(()) })` |

### Matchers (`mockall::predicate`)

| Matcher | Para |
|---------|------|
| `eq(v)` | Igualdade estrita |
| `always()` | Qualquer valor — use **escassamente**, perde sinal |
| `function(|x| ...)` | Predicado custom para inspecionar struct |
| `ge/gt/le/lt(v)` | Comparacoes numericas |
| `str::contains(...)` | Substring |

Argumento de struct complexo? Prefira `function` a um matcher gigante, ou capture e asserte no proprio `returning`:

```rust
repo.expect_create()
    .withf(|user: &User| user.email == "ana@example.com" && !user.name.is_empty())
    .returning(|user| { user.id = Uuid::now_v7(); Ok(()) });
```

(`withf` recebe referencias dos args e retorna `bool` — mais legivel que compor `function` para varios argumentos.)

### Ordenacao de chamadas (`mockall::Sequence`)

Por padrao mockall **nao** assume ordem. Para forcar ordem entre chamadas (ex: `begin` -> `insert` -> `commit`):

```rust
use mockall::Sequence;

let mut seq = Sequence::new();
repo.expect_find_by_email().times(1).in_sequence(&mut seq).returning(|_| Ok(None));
repo.expect_create().times(1).in_sequence(&mut seq).returning(|u| { u.id = Uuid::now_v7(); Ok(()) });
```

### Verificacao automatica no drop

O `MockUserRepository` verifica as expectativas (`times`, `never`) **quando e dropado** no fim do teste — se `find_by_email` era `.times(1)` e nao foi chamado, o teste entra em panic. Nao ha `Finish()` manual (diferente do gomock). Um mock **por teste**, nunca compartilhado via `static`.

### Fake tipado — implementacao alternativa do trait

Fake e uma implementacao in-memory que se comporta como a real. Para **aceitacao** prefira fake: voce se importa com o resultado, nao com "salvou exatamente uma vez". O fake vive dentro do `mod tests` (ou em `tests/testhelper/` na integracao):

```rust
#[cfg(test)]
mod tests {
    use std::sync::Mutex;

    use super::*;

    #[derive(Default)]
    struct InMemoryUserRepository {
        users: Mutex<Vec<User>>,   // Mutex: interior mutability sob &self (trait usa &self)
    }

    #[async_trait]
    impl UserRepository for InMemoryUserRepository {
        async fn create(&self, user: &mut User) -> anyhow::Result<()> {
            if user.id.is_nil() {
                user.id = Uuid::now_v7();   // simula o RETURNING id do banco
            }
            self.users.lock().unwrap().push(user.clone());
            Ok(())
        }

        async fn find_by_id(&self, id: Uuid) -> anyhow::Result<Option<User>> {
            Ok(self.users.lock().unwrap().iter().find(|u| u.id == id).cloned())
        }

        async fn find_by_email(&self, email: &str) -> anyhow::Result<Option<User>> {
            Ok(self.users.lock().unwrap().iter().find(|u| u.email == email).cloned())
        }
    }
}
```

- O fake **respeita o contrato**: nao encontrado -> `Ok(None)`, nunca `Err` (ver [[postgres-best-practices]] §13).
- Usa `Mutex` para mutar sob `&self` (os traits do dominio recebem `&self`); em teste single-thread o custo e irrelevante. Como implementa o trait, o **compilador** garante conformidade — se o trait muda, o fake nao compila.
- Um "spy" (fake que registra chamadas) e util quando o teste precisa verificar o efeito colateral: acumule os eventos publicados num `Mutex<Vec<...>>` e asserte no fim.

### O que **nao** fazer com dublês

- Nao mocke tipos concretos — mockall so gera mock de **traits**. Se precisa mockar algo concreto, extraia um trait no dominio (ver `architecture.md`, "Traits para Bibliotecas Externas").
- Nao asserte interacoes irrelevantes (`.times(1)` em tudo). Verifique a interacao **que e** o comportamento.
- Nao use fake para testar o **repositorio real** contra o banco — isso e integracao com container (secao 5).
- Nao mude assinatura do mock a mao: `automock` regenera automaticamente.

## 4. Testes de Aceitacao

Testes de aceitacao verificam **comportamento de negocio** descrito na linguagem do dominio — nao a estrutura interna. Sao o ponto onde o teste valida "essa feature faz o que o usuario pediu".

Em Rust nao ha lib canonica de BDD; `#[tokio::test]` + fakes cobrem 90% dos casos sem trazer ferramenta nova. Use Given/When/Then **na nomenclatura e nos blocos AAA**, nao em framework.

### Caracteristicas de um bom teste de aceitacao

- **Caixa preta**: usa apenas a API publica do use case/servico.
- **Linguagem do dominio**: nomes refletem regras de negocio, nao structs.
- **Independente de infra**: usa fakes/in-memory; so sobe container se a regra depende disso (entao vira integracao).
- **Um cenario por teste**: um unico Given/When/Then.

### Estrutura recomendada (composicao com fakes)

```rust
#[cfg(test)]
mod acceptance {
    use std::sync::Arc;

    use super::*;

    // Given: usuario em trial gratuito, nada cobrado ainda
    fn make_app() -> (BillingApp, Arc<InMemoryChargeRepository>) {
        let charges = Arc::new(InMemoryChargeRepository::default());
        let clock = Arc::new(FixedClock::new("2026-01-01T00:00:00Z"));
        let app = BillingApp::new(charges.clone(), clock);
        (app, charges)
    }

    #[tokio::test]
    async fn free_trial_user_not_charged_on_first_month() {
        // Given
        let (app, charges) = make_app();
        let user = app.create_user("ana@example.com", Plan::Trial).await.unwrap();

        // When
        app.run_billing_cycle().await.unwrap();

        // Then
        let issued = charges.for_user(user.id).await.unwrap();
        assert!(issued.is_empty(), "trial users must not be charged in month 1");
    }
}
```

### Nomenclatura padrao

`<ator_ou_contexto>_<acao>_<resultado_esperado>`

- OK: `free_trial_user_not_charged_on_first_month`
- OK: `paid_user_cancels_subscription_partial_refund_issued`
- Evite: `billing1`, `run_billing_cycle` (descreve metodo, nao regra)

### Fakes em vez de mocks

Mock (mockall) verifica **interacoes**; fake e implementacao alternativa que se comporta como a real. Para aceitacao prefira **fakes** — o teste sobrevive a refactors internos porque afirma o **resultado**, nao "chamou X uma vez". Para tempo deterministico, injete um `Clock` (trait de dominio) com um `FixedClock` no teste, em vez de `Utc::now()` cru.

### Quando promover para integracao/E2E

- Se o cenario depende de comportamento real do Postgres (transacao, lock, indice unico), promova para integracao com testcontainers.
- Se depende do contrato HTTP real (status, shape do JSON camelCase), promova para E2E de API.
- Senao, o fake e mais rapido e deterministico — e suficiente.

## 5. Testes de Integracao e E2E com `testcontainers`

Referencia oficial: https://docs.rs/testcontainers/ e https://docs.rs/testcontainers-modules/

Sobem **dependencias reais em containers** (PostgreSQL 18, NATS 2.12) gerenciadas pelo Docker, exercitam repositorios/publishers/subscribers e o fluxo HTTP ponta-a-ponta, e descartam tudo no fim. Vivem em `apps/<app>/tests/` (crate separado -> so API publica).

### Quando integracao/E2E vale o custo

- Validar migrations contra o engine real (sqlx aplicando `.up.sql`).
- Verificar comportamento dependente de DB (transacoes, conflitos de indice unico, `SKIP LOCKED`).
- Verificar o contrato HTTP completo (Axum + `ValidatedJson` + mapeamento `DomainError` -> status).
- Verificar consumo real de NATS JetStream (ack/nak/dedup).

Integracao/E2E **nao substitui** unitario/aceitacao. E a cereja, nao a base.

### Cargo features das dev-dependencies

Do `architecture.md` (workspace):

```toml
[workspace.dependencies]
mockall = "0.13"
testcontainers = "0.24"
testcontainers-modules = { version = "0.12", features = ["postgres", "nats"] }
rstest = "0.23"
reqwest = { version = "0.12", features = ["json"] }
```

No `Cargo.toml` do app, em `[dev-dependencies]` (so compilam para teste):

```toml
[dev-dependencies]
testcontainers = { workspace = true }
testcontainers-modules = { workspace = true }
rstest = { workspace = true }
reqwest = { workspace = true }
tokio = { workspace = true }
anyhow = { workspace = true }
sqlx = { workspace = true }
```

### Container compartilhado por sessao com `OnceCell`

O `architecture.md` prescreve helpers em `tests/testhelper/` com `OnceCell`: **um** Postgres para toda a suite (subir container por teste e caro), isolando cada teste por `TRUNCATE`.

```rust
// apps/user/tests/testhelper/postgres.rs
use sqlx::PgPool;
use testcontainers::{ContainerAsync, runners::AsyncRunner};
use testcontainers_modules::postgres::Postgres;
use tokio::sync::OnceCell;

// mantido vivo durante toda a suite; dropar => container para
static PG: OnceCell<PgContext> = OnceCell::const_new();

pub struct PgContext {
    pub pool: PgPool,
    _container: ContainerAsync<Postgres>,   // guarda a referencia viva
}

pub async fn pool() -> &'static PgPool {
    let ctx = PG
        .get_or_init(|| async {
            let container = Postgres::default()
                .with_db_name("users")
                .with_user("test")
                .with_password("test")
                .start()
                .await
                .expect("start postgres container");

            let port = container
                .get_host_port_ipv4(5432)
                .await
                .expect("map postgres port");
            let url = format!("postgres://test:test@127.0.0.1:{port}/users");

            let pool = PgPool::connect(&url).await.expect("connect pool");

            // aplica as migrations reais do app (sqlx-cli) contra o engine de verdade
            sqlx::migrate!("../../migrations/user")
                .run(&pool)
                .await
                .expect("run migrations");

            PgContext { pool, _container: container }
        })
        .await;

    &ctx.pool
}

/// Isolamento entre testes: zera as tabelas antes de cada teste.
pub async fn reset(pool: &PgPool) {
    sqlx::query(r#"TRUNCATE TABLE "users" RESTART IDENTITY CASCADE"#)
        .execute(pool)
        .await
        .expect("truncate");
}
```

- **Nunca** hardcode porta: `get_host_port_ipv4(5432)` devolve a porta efemera mapeada no host.
- O `ContainerAsync` **precisa** ficar vivo (por isso o `_container` no `PgContext` dentro do `OnceCell`): quando dropado, o testcontainers derruba o container. Guardar so o pool faria o container morrer.
- `sqlx::migrate!("../../migrations/user")` embute e roda as migrations em compile time — o path e relativo ao `Cargo.toml` do app.
- Isolamento: `TRUNCATE ... RESTART IDENTITY CASCADE` no inicio de cada teste. Alternativa de maior isolamento: **schema por teste** (`CREATE SCHEMA t_<uuid>; SET search_path`) quando testes rodam em paralelo e o TRUNCATE global causaria interferencia.

### Teste de integracao de repositorio

```rust
// apps/user/tests/user_repository_integration_test.rs
mod testhelper;

use user::domain::entity::User;
use user::domain::repository::UserRepository;
use user::infrastructure::repository::PostgresUserRepository;

use testhelper::postgres;

#[tokio::test]
async fn create_then_find_by_id_returns_persisted_user() {
    let pool = postgres::pool().await;
    postgres::reset(pool).await;
    let repo = PostgresUserRepository::new(pool.clone());

    let mut user = User::new("Ana".into(), "ana@example.com".into());
    repo.create(&mut user).await.unwrap();

    assert!(!user.id.is_nil(), "id deve ser preenchido pelo RETURNING");

    let found = repo.find_by_id(user.id).await.unwrap();
    assert_eq!(found.map(|u| u.email), Some("ana@example.com".to_string()));
}

#[tokio::test]
async fn find_by_id_returns_none_when_absent() {
    let pool = postgres::pool().await;
    postgres::reset(pool).await;
    let repo = PostgresUserRepository::new(pool.clone());

    let found = repo.find_by_id(uuid::Uuid::now_v7()).await.unwrap();

    assert!(found.is_none());   // ausencia e Ok(None), nunca erro
}
```

### NATS em container (publisher/subscriber)

```rust
// apps/user/tests/testhelper/nats.rs
use async_nats::Client;
use testcontainers::{ContainerAsync, runners::AsyncRunner};
use testcontainers_modules::nats::Nats;
use tokio::sync::OnceCell;

static NATS: OnceCell<NatsContext> = OnceCell::const_new();

pub struct NatsContext {
    pub client: Client,
    _container: ContainerAsync<Nats>,
}

pub async fn client() -> &'static Client {
    let ctx = NATS
        .get_or_init(|| async {
            let container = Nats::default().start().await.expect("start nats");
            let port = container.get_host_port_ipv4(4222).await.expect("map nats port");
            let client = async_nats::connect(format!("nats://127.0.0.1:{port}"))
                .await
                .expect("connect nats");
            NatsContext { client, _container: container }
        })
        .await;

    &ctx.client
}
```

Para JetStream, chame `pkg::nats::ensure_stream(&js, ...)` na fixture antes de publicar; asserte o consumo com um pull consumer duravel + timeout (ver [[nats-best-practices]] para ack/nak/dedup).

### E2E de API: Axum em porta livre + `reqwest`

Suba a API real ligada a `127.0.0.1:0` (porta efemera escolhida pelo SO), rode em background com `tokio::spawn`, e exercite por HTTP com `reqwest`:

```rust
// apps/user/tests/user_api_integration_test.rs
mod testhelper;

use std::sync::Arc;

use axum::Router;
use tokio::net::TcpListener;

use user::infrastructure::controller::UserController;
use user::infrastructure::repository::PostgresUserRepository;
// ... use cases + publisher fake/real

use testhelper::postgres;

async fn spawn_app() -> String {
    let pool = postgres::pool().await;
    postgres::reset(pool).await;

    // composicao minima da API (mesmo grafo do src/bin/api.rs, dependencias de teste)
    let repo = Arc::new(PostgresUserRepository::new(pool.clone()));
    let controller = UserController::new(/* use cases construidos com `repo` */);

    let app: Router = pkg::httpserver::with_default_middlewares(
        Router::new().merge(controller.routes()),
    );

    // porta 0 => SO escolhe uma porta livre; leia a porta real depois do bind
    let listener = TcpListener::bind(("127.0.0.1", 0)).await.expect("bind");
    let addr = listener.local_addr().expect("local addr");

    tokio::spawn(async move {
        axum::serve(listener, app).await.expect("serve");
    });

    format!("http://{addr}")
}

#[tokio::test]
async fn post_users_returns_201_with_created_user() {
    let base = spawn_app().await;
    let client = reqwest::Client::new();

    let resp = client
        .post(format!("{base}/v1/users"))
        .json(&serde_json::json!({ "name": "Ana", "email": "ana@example.com" }))
        .send()
        .await
        .unwrap();

    assert_eq!(resp.status(), reqwest::StatusCode::CREATED);
    let body: serde_json::Value = resp.json().await.unwrap();
    assert_eq!(body["email"], "ana@example.com");
    assert!(body["id"].is_string());
}

#[tokio::test]
async fn post_users_returns_409_when_email_already_exists() {
    let base = spawn_app().await;
    let client = reqwest::Client::new();
    let payload = serde_json::json!({ "name": "Ana", "email": "ana@example.com" });

    client.post(format!("{base}/v1/users")).json(&payload).send().await.unwrap();
    let resp = client.post(format!("{base}/v1/users")).json(&payload).send().await.unwrap();

    assert_eq!(resp.status(), reqwest::StatusCode::CONFLICT);
}

#[tokio::test]
async fn post_users_returns_400_on_invalid_body() {
    let base = spawn_app().await;
    let client = reqwest::Client::new();

    let resp = client
        .post(format!("{base}/v1/users"))
        .json(&serde_json::json!({ "name": "A" }))   // email faltando + name curto
        .send()
        .await
        .unwrap();

    assert_eq!(resp.status(), reqwest::StatusCode::BAD_REQUEST);
}
```

Regras E2E:
- **Porta livre sempre** (`("127.0.0.1", 0)`), URL montada com `listener.local_addr()`. Nunca porta hardcoded.
- A task do servidor e derrubada quando o runtime do teste termina; nao precisa `abort()` manual em teste curto.
- E2E cobre o **contrato HTTP**: status, shape do JSON (camelCase), mapeamento erro de dominio -> status. Regra de negocio fina fica no unitario.
- Teste os 3 tipos de resposta de cada endpoint critico: sucesso, erro de dominio mapeado (404/409/422) e erro de validacao (400).

### Separando unitarios de testes com container

Rust nao tem build tag. Os testes em `tests/` (integracao/E2E) compilam como crates separados; controle a execucao por escopo do `cargo test`:

```bash
cargo test --lib                       # SO unitarios/aceitacao (mod tests dentro de src/)
cargo test --test '*'                  # SO integracao/E2E (tudo em tests/) — precisa de Docker
cargo test --workspace                 # tudo
cargo test --test user_api_integration_test post_users_returns_409   # um caso especifico
```

Se algum teste exige Docker e nao pode rodar em toda maquina/CI, marque com `#[ignore]` e rode explicitamente com `cargo test -- --ignored`, ou gate atras de uma `feature` (`#[cfg(feature = "integration")]`). Mantenha o ciclo de feedback dos unitarios rapido.

## 6. Cobertura e Mutation Testing

### Cobertura com `cargo-llvm-cov`

Referencia: https://github.com/taiki-e/cargo-llvm-cov

```bash
cargo install cargo-llvm-cov

cargo llvm-cov --workspace                              # resumo no terminal
cargo llvm-cov --workspace --html                       # relatorio HTML (target/llvm-cov/html)
cargo llvm-cov --workspace --lcov --output-path lcov.info  # para CI/Codecov
cargo llvm-cov --lib                                    # so unitarios (sem containers)
```

- `cargo-llvm-cov` usa a instrumentacao **source-based** do LLVM (via rustc) — cobertura precisa de linha e regiao, superior ao antigo `grcov`/tarpaulin em muitos casos.
- Cobertura e **detector de buraco**, nao meta de vaidade: 100% de linha com asercoes fracas e pior que 85% com asercoes fortes — por isso o projeto complementa com mutation testing.
- Foque a meta nos modulos de regra de negocio (`domain/`, `application/`); `infrastructure/` (SQL, wiring) e melhor coberto por integracao real que por meta de linha.

### Mutation testing com `cargo-mutants`

Referencia: https://github.com/sourcefrog/cargo-mutants

```bash
cargo install cargo-mutants

# muta apenas a regra de negocio (domain + application) do app
cargo mutants --package user --in-place \
    -f 'apps/user/src/domain/**' -f 'apps/user/src/application/**'

cargo mutants --list                                    # lista mutantes sem rodar
```

- `cargo mutants` gera variantes do codigo (troca `==` por `!=`, `+` por `-`, remove chamadas, troca `Ok`/`Err`...) e roda a suite. Mutante **sobrevivente** = a suite passou mesmo com o codigo quebrado => teste fraco.
- Mute **domain e application** (onde vive a regra de negocio); **nao** mute `infrastructure/` nem `src/bin/` — mutantes de SQL/composition root geram ruido.
- Rode contra os **unitarios** (rapidos); mutation testing sobre containers e inviavel (cada mutante reexecutaria a suite).
- Mutante sobrevivente = fortaleca a **asercao** (valor exato, variante de erro exata com `matches!`), nao escreva teste que so executa a linha.
- **Threshold pratico: >= 80%** de mutantes mortos nos modulos da feature.

## 7. Convencoes Transversais

### Estrutura de pastas recomendada

```
apps/user/
├── src/
│   ├── application/usecase/user/
│   │   └── create_usecase.rs        # impl + #[cfg(test)] mod tests (unitario/aceitacao)
│   ├── domain/
│   │   ├── repository/user_repository.rs   # trait + #[cfg_attr(test, mockall::automock)]
│   │   └── event/user_event.rs
│   └── infrastructure/repository/user_repository.rs
└── tests/                            # crate separado (API publica)
    ├── testhelper/
    │   ├── mod.rs                    # pub mod postgres; pub mod nats;
    │   ├── postgres.rs               # container PG compartilhado (OnceCell) + reset
    │   └── nats.rs                   # container NATS compartilhado (OnceCell)
    ├── user_repository_integration_test.rs
    └── user_api_integration_test.rs
```

- `mod testhelper;` no topo de cada arquivo de teste importa o modulo (o `tests/testhelper/mod.rs` declara `pub mod postgres;` etc). Cada `*_integration_test.rs` e um binario independente.

### Determinismo

- Nada de `Utc::now()`/random cru no SUT sem seam: injete um `Clock` (trait) e use `FixedClock` no teste, ou aceite o timestamp como parametro. IDs gerados pelo banco (`RETURNING id`) — no teste unitario, o fake preenche com `Uuid::now_v7()`.
- Nenhum teste depende de ordem de execucao — `cargo test` roda em paralelo por padrao; cada teste recria estado (fake novo, `reset()` na integracao).
- Nenhum `sleep` real para sincronizar: unitario controla o tempo; integracao espera por condicao (poll com timeout).

### Nao teste a implementacao, teste o comportamento

- Evite: "verifica que `repo.create` foi chamado 1x com `updated_at` setado"
- Prefira: "depois de `perform`, `find_by_id` retorna o usuario com o novo nome"

Mocks (mockall) sao valiosos onde I/O custaria ou onde a **interacao e** o contrato; mas se a regra cabe num fake in-memory, prefira — o teste sobrevive a refactors.

### Ferramentas de qualidade no CI

```bash
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --lib                       # unitarios/aceitacao
cargo test --test '*'                  # integracao/E2E (com Docker disponivel)
cargo llvm-cov --lib --lcov --output-path lcov.info
```

O ownership do Rust ja elimina data races em compile time; `cargo test` roda em paralelo com seguranca. `clippy` com `-D warnings` trata lint como erro (ver [[qa-best-practices]]).

## 8. Checklist de Revisao

Ao escrever ou revisar testes Rust, verifique:

### Geral
- [ ] Unitario/aceitacao em `#[cfg(test)] mod tests` no proprio arquivo; integracao/E2E em `apps/<app>/tests/`
- [ ] `#[tokio::test]` para testes async; `#[test]` so para sincronos puros
- [ ] AAA com blocos separados; um comportamento por teste
- [ ] Nome em linguagem de dominio (`perform_returns_x_when_y`)
- [ ] `rstest` com `#[case]` quando houver >= 3 casos da mesma regra
- [ ] Nenhum teste depende de ordem/estado de outro (paralelo por default)

### Unitarios e asserts
- [ ] `assert_eq!` para valores; `matches!` para variantes de erro/enum
- [ ] Asserta o **valor exato**, nao apenas `is_ok()`
- [ ] Caminho de erro de infra testado (mock com `Err` -> `DomainError::Unexpected`)

### Dublês
- [ ] `#[cfg_attr(test, mockall::automock)]` **antes** de `#[async_trait]` no trait
- [ ] Mock (`MockNome`) para verificar interacao; fake (implementa o trait) para aceitacao
- [ ] `.times(n)`/`.never()` só na interacao que **e** o comportamento
- [ ] `withf`/`function` em vez de matcher gigante para structs
- [ ] `Sequence` quando a ordem importa; default e nao-ordenado
- [ ] Um mock por teste — verificado no drop, sem `Finish()` manual
- [ ] Fake respeita contrato: nao encontrado -> `Ok(None)`

### Integracao / E2E (testcontainers)
- [ ] Modulo pre-construido (`testcontainers_modules::postgres::Postgres`, `nats::Nats`)
- [ ] Container compartilhado por sessao via `OnceCell`; `ContainerAsync` mantido vivo
- [ ] `get_host_port_ipv4(...)` — nunca porta hardcoded
- [ ] Migrations aplicadas com `sqlx::migrate!` contra o engine real
- [ ] Isolamento entre testes por `TRUNCATE ... CASCADE` (ou schema por teste)
- [ ] E2E: Axum em `("127.0.0.1", 0)`, URL de `local_addr()`, chamadas com `reqwest`
- [ ] E2E cobre sucesso, erro de dominio mapeado (404/409/422) e validacao (400)

### Cobertura e mutacao
- [ ] `cargo llvm-cov` rodando os unitarios; foco em `domain/` e `application/`
- [ ] `cargo mutants` mutando apenas `domain/` e `application/` (nao `infrastructure/`/`bin/`)
- [ ] Mutante sobrevivente tratado fortalecendo a asercao (>= 80% mortos)

## 9. Fontes

- https://doc.rust-lang.org/book/ch11-00-testing.html — testes na linguagem (unit, integration, `#[cfg(test)]`)
- https://doc.rust-lang.org/book/ch11-03-test-organization.html — organizacao de testes (`tests/`, mod tests)
- https://docs.rs/mockall/ — mocks de traits (`automock`, `MockNome`, predicates, `Sequence`)
- https://docs.rs/rstest/ — fixtures e casos parametrizados (`#[case]`, `#[fixture]`)
- https://docs.rs/testcontainers/ — API de containers (`AsyncRunner`, `ContainerAsync`, `get_host_port_ipv4`)
- https://docs.rs/testcontainers-modules/ — modulos prontos (`postgres`, `nats`)
- https://tokio.rs/tokio/topics/testing — testes async com `#[tokio::test]`
- https://github.com/taiki-e/cargo-llvm-cov — cobertura source-based do LLVM
- https://github.com/sourcefrog/cargo-mutants — mutation testing

Combine sempre com [[postgres-best-practices]] (repositorios sqlx, transactions, migrations) e [[nats-best-practices]] (publish/subscribe, ack/nak, dedup) para as camadas de infraestrutura exercitadas nos testes de integracao.
