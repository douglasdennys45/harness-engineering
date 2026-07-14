---
name: task-reviewer-best-practices
description: Aplica melhores práticas de Task Review para revisão sistemática de tarefas concluídas em serviços Rust. Cobre identificação da tarefa em `.cognup/specs/[feature]/tasks/[num]_task.md`, análise de diff (`git diff`/`git log`), validação contra práticas idiomáticas de Rust (ownership, Result/Option, sem `unwrap()` em produção, thiserror/anyhow, `?`, imports organizados), Clean Architecture (`.claude/rules/architecture.md`), tratamento de erros (`DomainError` com thiserror, `anyhow::Context`, `matches!`), concorrência segura com Tokio (async/await, cancelamento, sem bloquear o runtime), Axum 0.8 (RequestContext, ValidatedJson, ApiError/HttpDomainError, `routes(self)->Router`, extractors), PostgreSQL 18 (sqlx, queries manuais, migrations sqlx-cli, `*_with_tx` + `pool.begin/commit`), NATS JetStream (async-nats, ack/nak/term via AckKind, dedup, idempotência), composition root manual + `Arc<dyn Trait>`, tracing, testes (mockall, `#[tokio::test]`, testcontainers), execução de verificações (`cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo build`, `cargo test`), classificação de problemas (Crítico/Major/Minor/Positivo) e geração ou re-escrita do artefato em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md` em Português (BR) com status em inglês (APPROVED/APPROVED WITH OBSERVATIONS/CHANGES REQUESTED). Use sempre que uma tarefa Rust foi concluída via workflow `run-task.md` e precisa ser revisada antes do merge, ao gerar artefato de revisão, ao reexecutar review após correções, ou ao avaliar aderência de uma implementação Rust aos padrões do projeto.
---

# Task Reviewer Best Practices — Revisão Sistemática de Tarefas Rust

Skill que aplica o processo metódico de **Task Review** sobre tarefas concluídas usando o workflow `run-task.md` em serviços Rust. Combina identificação da tarefa, análise de diff, validação contra padrões idiomáticos, verificações automatizadas, classificação de problemas e geração do artefato `[num]_task_review.md` rastreável.

Combina com [[rust-best-practices]] (padrões de código idiomático Rust) e [[rust-testing-best-practices]] (estratégias de teste), e — conforme o escopo do diff — com as skills de domínio [[axum-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-rust]] (testes de integração). A arquitetura de referência é a definida em `.claude/rules/architecture.md` (Clean Architecture em monorepo com Axum 0.8, PostgreSQL 18, NATS JetStream e composition root manual com `Arc<dyn Trait>`).

## Princípio Fundamental

> **Revisão completa, justa e rastreável.** O revisor identifica a tarefa, lê integralmente os arquivos modificados (não apenas os diffs), valida contra padrões idiomáticos do Rust, executa verificações automatizadas e gera o artefato `[num]_task_review.md` com problemas classificados por severidade. Cada problema referencia arquivo, linha e correção sugerida; cada boa prática observada é reconhecida.

A revisão produz exatamente **um** dos três status (em inglês — é o que o workflow `run-task.md` e os agentes de correção verificam):

- **APPROVED** — pronto para merge
- **APPROVED WITH OBSERVATIONS** — segue mas requer nova revisão após melhorias
- **CHANGES REQUESTED** — requer correções obrigatórias e nova execução do reviewer

Apenas **APPROVED** finaliza a revisão. Os outros dois disparam o ciclo de correção → re-revisão até obter APPROVED.

## Localização dos Artefatos

| Artefato | Caminho |
|---|---|
| PRD | `.cognup/specs/[nome-da-funcionalidade]/prd.md` |
| Tech Spec | `.cognup/specs/[nome-da-funcionalidade]/techspec.md` |
| Lista de tarefas | `.cognup/specs/[nome-da-funcionalidade]/tasks.md` |
| Tarefa individual | `.cognup/specs/[nome-da-funcionalidade]/tasks/[num]_task.md` |
| **Review (SAÍDA)** | `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md` |
| Regras do projeto | `.claude/rules/architecture.md` |

**REGRA CRÍTICA:** o artefato de revisão é gravado SEMPRE em `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md` — nunca ao lado do arquivo da task, nunca na raiz da feature. Se a pasta não existir, crie antes: `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`.

## Quando Aplicar

- Quando uma tarefa Rust foi finalizada via workflow `run-task.md` e o usuário pediu revisão
- Quando o usuário cita um número de tarefa específico (`task 5`, `task 12`) para revisar
- Proativamente após uma implementação significativa em Rust ser concluída
- Em re-revisão: quando `tasks/reviews/[num]_task_review.md` já existe com status diferente de APPROVED
- Antes de abrir Pull Request de uma tarefa concluída
- Ao validar aderência de código novo às convenções do projeto (Clean Architecture, composition root, Axum 0.8, PostgreSQL, NATS)

## Missão do Revisor

Você é um revisor de código de elite com domínio profundo em **Rust, sistemas distribuídos, REST APIs, Clean Architecture e engenharia de software**. Seu olhar é meticuloso para código idiomático Rust, ownership, qualidade, manutenibilidade e aderência aos padrões do projeto.

Em toda revisão, seu trabalho é:

1. Identificar qual tarefa foi concluída encontrando o `[num]_task.md` correspondente
2. Compreender o que foi solicitado na tarefa
3. Revisar **TODAS** as alterações de código relacionadas
4. Executar verificações automatizadas
5. Classificar problemas (Crítico/Major/Minor/Positivo)
6. Gerar/sobrescrever `tasks/reviews/[num]_task_review.md` com avaliação final em Português (BR)

## Pipeline de Revisão — 7 Etapas

### Etapa 1: Identificar a Tarefa e o Contexto

Localize o arquivo da tarefa concluída:

- Identifique a feature em `.cognup/specs/[nome-da-funcionalidade]/` (se não informada, use a feature com atividade mais recente)
- Localize a task em `tasks/[num]_task.md` (fallback: procure `[num]_task.md` dentro da pasta da feature)
- Se nenhum número for fornecido, encontre em `tasks.md` a última task marcada como concluída
- **Ler integralmente** a task, o `techspec.md` e o trecho relevante do `prd.md` antes de qualquer julgamento

**Detecção de Re-revisão:** Se `tasks/reviews/[num]_task_review.md` já existir com **Status** diferente de **APPROVED** (ex.: APPROVED WITH OBSERVATIONS, CHANGES REQUESTED):

- Trata-se de **re-revisão** após correções
- Execute a revisão completa no código **atual**
- **Sobrescreva** o arquivo com a nova avaliação
- Marque `**Re-revisão**: Sim (após correções)` no cabeçalho

```bash
# Localizar a task
ls .cognup/specs/*/tasks/*_task.md

# Verificar se já existe review (para detecção de re-revisão)
ls -la .cognup/specs/[feature]/tasks/reviews/*_task_review.md 2>/dev/null
```

### Etapa 2: Identificar Arquivos Alterados

Use git para identificar o escopo real da tarefa:

```bash
# Diff vs branch base (o projeto usa Gitflow: base é develop)
git diff develop...HEAD --stat

# Commits da branch atual
git log develop...HEAD --oneline

# Arquivos modificados não commitados (caso a tarefa ainda não tenha sido commitada)
git status --short
git diff HEAD --stat
```

**Regras críticas:**

- Listar **todos** os arquivos alterados (não apenas os "mais relevantes")
- Ler o **contexto completo** de cada arquivo modificado, não apenas o trecho do diff — bugs frequentemente vivem nas linhas adjacentes não modificadas
- Para arquivos novos, ler 100% do conteúdo
- Para arquivos grandes, focar nas funções/métodos alterados mas verificar imports, constantes, tipos relacionados

### Etapa 3: Preparação Técnica

Antes de iniciar a revisão detalhada, carregue o contexto técnico do projeto:

1. **Consultar `.claude/rules/architecture.md`** — camadas (domain/application/infrastructure), regra de dependência, nomenclatura snake_case, convenções de construtores (`new() -> Self`), padrão de controller (RequestContext/ValidatedJson/ApiError), transactions com sqlx (`pool.begin()` + `*_with_tx`), composition root manual com `Arc<dyn Trait>`, guia "Adicionando uma Nova Feature"
2. **Consultar `CLAUDE.md`** (se existir) — convenções específicas e comandos de build/test
3. **Confirmar a stack** (padrão do projeto): **Rust edition 2024 (MSRV 1.85), Tokio, Axum 0.8, PostgreSQL 18 (sqlx 0.8), NATS JetStream (async-nats), composition root manual + `Arc<dyn Trait>`, tracing, validator 0.20, utoipa, thiserror + anyhow, mockall/testcontainers** — valide desvios pelos `Cargo.toml` do workspace
4. **Carregar [[rust-best-practices]]** como referência idiomática (sempre)
5. **Carregar as skills de domínio aplicáveis ao diff**: [[axum-best-practices]] se tocou controllers/rotas/middlewares; [[postgres-best-practices]] se tocou repositórios/queries/migrations; [[nats-best-practices]] se tocou publishers/subscribers/streams; [[rust-testing-best-practices]] e [[testcontainers-rust]] se tocou testes
6. **Não carregar skills fora do escopo** — ex.: não carregue NATS para uma task que só cria entidade de domínio. Registre no artefato quais foram usadas e quais foram N/A

A revisão precisa ser fiel **ao projeto que você está revisando** — não imponha padrões que o projeto não adota. Mas valide rigorosamente os padrões que o projeto **declarou** adotar.

### Etapa 4: Conduzir a Revisão

Revise o código contra os critérios abaixo, agrupados por área. Para cada violação, registre **arquivo + linha + descrição + correção sugerida**.

#### 4.1 Padrões Gerais de Código

- **Idioma do código**: tudo em inglês (variáveis, funções, structs, comentários)
- **Nomenclatura clara**: identificadores descritivos; nomes curtos apenas em escopos pequenos (Rust permite `i`, `id`, `err` localmente)
- **Constantes**: sem números mágicos — usar `const` (`const QUERY: &str`, `const MAX_RETRIES: u32`)
- **Funções**: uma única ação clara, verbos para mutações (`create_order`, `update_user`)
- **Parâmetros**: > 3-4 parâmetros indica necessidade de struct de input/config
- **Condicionais**: sem aninhamento profundo, **early returns**, `?` e `let ... else` para guard clauses
- **Tamanho de método**: lógica complexa < 50 linhas; se ultrapassar, extrair helper
- **Tamanho de arquivo**: responsabilidade única; dividir quando exceder ~500 linhas
- **Comentários**: itens públicos (`pub`) **DEVEM** ter doc comments (`///`); evitar comentários que repetem o código

#### 4.2 Rust Idioms

> Referências: [The Rust Book](https://doc.rust-lang.org/book/), [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/). Use [[rust-best-practices]] para detalhes.

| Critério | Verificar |
|---|---|
| **Nomenclatura** | `snake_case` para funções/módulos/variáveis; `UpperCamelCase` para tipos/traits; `SCREAMING_SNAKE_CASE` para consts — nunca camelCase em Rust |
| **Ownership & borrowing** | preferir `&str`/`&[T]` em parâmetros; `clone()` apenas quando necessário; sem `Rc`/`Arc` desnecessário |
| **`Option`/`Result`** | ausência via `Option<T>`; falha via `Result<T, E>`; sem sentinelas ou `-1`/`""` mágicos |
| **Sem `unwrap()`/`expect()` em produção** | apenas em testes ou bootstrap do `main`; caminhos de request usam `?` |
| **Propagação com `?`** | preferir `?` a `match` verboso quando só propaga o erro |
| **Traits pequenas** | focadas, poucos métodos (ISP); compor via bounds |
| **`impl Trait` / generics** | evitar `Box<dyn>` quando `impl Trait` ou generics servem; `Arc<dyn Trait>` é o padrão para injeção |
| **Enums fechados** | modelar estados/variações com enums em vez de strings mágicas |
| **Sem stuttering** | `user::User`, não `user::UserStruct`; módulo não repete o nome do tipo |
| **Conversões idiomáticas** | `From`/`Into`/`TryFrom`; `Into::into` em `.map()`; sem `as` que trunca silenciosamente |
| **Iteradores** | `iter()`/`map()`/`filter()`/`collect()` sobre loops manuais quando mais claro |
| **Derives** | `#[derive(Debug, Clone, Serialize, Deserialize)]` conforme necessário; `#[serde(rename_all = "camelCase")]` nas entidades |

#### 4.3 Tratamento de Erros

| Regra | Como Verificar |
|---|---|
| Sem `unwrap()`/`expect()`/`panic!` em produção | apenas em testes ou bootstrap do `main` |
| Erros de domínio tipados | enum `DomainError` com `thiserror` (`#[derive(Debug, Error)]`) por app; variantes de negócio explícitas |
| Ponte com infra via `Unexpected` | `#[error(transparent)] Unexpected(#[from] anyhow::Error)` |
| Contexto na propagação | `.context("ação")?` (anyhow) em repositórios/adapters/publishers |
| Decisão por variante | `matches!(err, DomainError::UserAlreadyExists)` / `match` — nunca comparar `str(err)` ou mensagens mágicas |
| Sem `Result` descartado | sem `let _ = fallible()` sem justificativa (exceto publish de evento pós-commit, se for o padrão do projeto) |
| Mensagens de erro | minúsculas, sem pontuação final, descrevem o que falhou (`#[error("user not found")]`) |
| Não logar e propagar | escolha **logar na boundary** (`IntoResponse` do `ApiError`) OU **propagar com `?`**, nunca ambos |

#### 4.4 Concorrência (Tokio)

| Regra | Como Verificar |
|---|---|
| Lifecycle de tasks | toda `tokio::spawn` tem caminho de encerramento (shutdown signal, `JoinHandle`, `abort()` no shutdown) |
| Sem task leak | tasks terminam no shutdown (`tokio::select!` com sinal, drain) |
| Sincronização correta | `Arc<Mutex<T>>`/`RwLock` para estado compartilhado; `tokio::sync::Mutex` quando o guard atravessa `.await` |
| Sem bloquear o runtime | sem I/O síncrono/`std::thread::sleep`/CPU-bound longo no contexto async; usar `tokio::task::spawn_blocking` |
| Data races | prevenidas em compile time pelo ownership + `Send`/`Sync`; sinalizar qualquer `unsafe` injustificado |
| `#[async_trait]` + `Send + Sync` | traits de domínio com bound `Send + Sync` (necessário para `Arc<dyn Trait>`) |
| Cancelamento gracioso | `with_graceful_shutdown(shutdown_signal())` na API; `worker.abort()` + drain no consumer |

#### 4.5 Estrutura do Projeto (Clean Architecture — `.claude/rules/architecture.md`)

| Camada | Pode importar |
|---|---|
| `src/domain/` | apenas std e crates de tipos/contratos (`serde`, `chrono`, `uuid`, `thiserror`, `anyhow`, `async-trait`) — **nunca** `axum`, `sqlx`, `async-nats`, `tower` |
| `src/application/` | apenas `crate::domain` |
| `src/infrastructure/` | `crate::domain`, `pkg`, crates externas (axum, sqlx, async-nats) |
| `pkg/` | **nunca** importa nada dos apps |
| `apps/<app>/src/bin/<binary>.rs` | tudo (composition root manual daquele binário) |

Validar também:

- **Use cases**: 1 ação = 1 trait = 1 arquivo, método único `perform(&self, input)`; trait em `domain/usecase/<contexto>/<ação>.rs`, impl em `application/usecase/<contexto>/<ação>_usecase.rs` (mesmo nome, sem sufixo `Impl`)
- **Construtores**: todas as camadas expõem `new(...) -> Self`; o **composition root** embrulha em `Arc<dyn Trait>`
- **Campos de struct** são **traits de domínio** via `Arc<dyn Trait>` (nunca tipos concretos), exceto repositórios concretos + `PgPool` quando o controller orquestra transaction
- **Libs externas via inversão de dependência**: trait em `domain/<capacidade>/` (ex.: `crypt/hasher.rs`, `token/manager.rs`), implementação em `infrastructure/adapter/<lib>_adapter.rs`
- **Eventos publicados DEPOIS do commit** no banco, nunca dentro da transaction
- **Row structs** (`#[derive(sqlx::FromRow)]`) vivem na infra + `From<Row> for Entity`; o domínio **nunca** deriva `FromRow`
- Convenções de nomenclatura de arquivos: snake_case (`user_repository.rs`), módulos estilo `mod.rs`
- Sem imports circulares; direção limpa de dependências; sem estado global (`static`/`lazy_static`/`OnceLock` para infra)

#### 4.6 REST/HTTP (Axum 0.8, quando aplicável)

> Referência: [[axum-best-practices]] e o padrão de controller de `architecture.md`.

| Critério | Verificar |
|---|---|
| Versão | `axum = "0.8"` |
| Handler signature | associated functions `async fn` recebendo `State<Arc<Self>>` + extractors, retornando `Result<impl IntoResponse, ApiError>` |
| `routes(self)` | controller implementa `pub fn routes(self) -> Router`, embrulha `self` em `Arc` e registra rotas com `.with_state(ctrl)` |
| RequestContext | handlers declaram o extractor `RequestContext` já populado (request_id, parent_id, ip, user_agent) — nunca extrair headers manualmente |
| Bind + validação | body via `ValidatedJson<T>` (parse + `#[validate]` do crate `validator`); `Query<T>` para query params; `Path<T>` para path params — nunca `serde_json::from_*` manual |
| Validação declarativa | structs de input com `#[derive(Deserialize, Validate, ToSchema)]` + `#[serde(rename_all = "camelCase")]` e atributos `#[validate(...)]`; tipos fortes (`Uuid`, enums serde) — nunca validação manual no handler |
| Erros de domínio | propagados com `?`; `From<DomainError> for ApiError` mapeia via `HttpDomainError::status_code()`; cada app implementa `HttpDomainError` em `infrastructure/controller/errors.rs` (match exaustivo) |
| Sintaxe de rotas | path params Axum 0.8: `{id}` (e `{*rest}` para wildcard) — não `:id` |
| Recursos | inglês, plural, kebab-case (`/v1/users`, `/v1/orders`); versionados |
| Status codes | `StatusCode::*` (não números mágicos) |
| Transactions no handler | somente quando N operações atômicas: `pool.begin()` + métodos `*_with_tx` + `tx.commit()`; rollback automático no drop; repositório recebido como tipo **concreto** (`Arc<PostgresXRepository>`) |
| Evento pós-commit | `event.publish(...)` **depois** de `tx.commit()`, nunca dentro da tx |
| Middlewares | pilha padrão via `pkg::httpserver::with_default_middlewares(router)` aplicada **por último** (request-id, trace, catch-panic, cors, compression, timeout) |
| Graceful shutdown | `axum::serve(...).with_graceful_shutdown(shutdown_signal())` |
| OpenAPI | handlers documentados com `#[utoipa::path(...)]`; DTOs com `#[derive(ToSchema)]` |

#### 4.7 Logging (tracing)

- Logging estruturado em JSON via `tracing` (setup por `pkg::logger::setup`)
- Níveis: `debug!` (dev), `info!` (eventos notáveis), `warn!` (recuperável), `error!` (falha)
- Nunca arquivos — usar stdout/stderr (subscriber JSON)
- Nunca logar dados sensíveis (senhas, tokens, PII, CPF, cartão)
- Mensagens claras + campos estruturados (`tracing::info!(user_id = %id, "user created")`)
- Nunca silenciar erros (`let _ = err`)
- Incluir `request_id`/`parent_id` para rastreabilidade em handlers e subscribers
- 5xx logado como `error!` e 4xx como `warn!` — feito no `IntoResponse` do `ApiError` (não duplicar no handler)

#### 4.8 Banco de Dados (PostgreSQL 18 via sqlx, quando aplicável)

> Referência: [[postgres-best-practices]] e os padrões de repositório de `architecture.md`.

- SQL manual com placeholders posicionais (`$1`, `$2`) — **nunca** interpolação/`format!` no SQL; sem ORM
- Sem `SELECT *` — listar colunas explicitamente; queries como `const QUERY: &str` dentro do método
- Ausência → `fetch_optional` retorna `Ok(None)` (ausência não é erro)
- `INSERT ... RETURNING id` (`fetch_one` + `query_as`) para obter ID gerado; `create` recebe `&mut Entity` para preencher o ID
- Paginação por **cursor**, nunca `OFFSET` em tabelas grandes
- Migrations `.up.sql`/`.down.sql` em `migrations/<app>/`, criadas com `sqlx migrate add -r <nome> --source migrations/<app>`
- Operações multi-step atômicas via `pool.begin()` + métodos `*_with_tx` (impl concreta, fora do trait de domínio) + `tx.commit()`; rollback automático no drop
- Struct `Postgres<Name>Repository` recebe `PgPool`; row structs privadas com `#[derive(sqlx::FromRow)]` + `From<Row> for Entity`
- Repositórios implementam **traits de domínio**; métodos de trait retornam `anyhow::Result` com `.context("ação")?`

#### 4.9 Mensageria (NATS JetStream via async-nats, quando aplicável)

> Referência: [[nats-best-practices]].

- Publishers (`Nats<Name>Publisher`) implementam traits de evento do domínio (`UserEvent`), via `Publisher` genérico do `pkg`
- `Nats-Msg-Id` = ID da entidade para deduplicação server-side
- Subscribers (`<Name>Subscriber`) delegam para use cases — **sem lógica de negócio no consumer**; handlers `handle_<event>`
- Ack/Nak/Term adequados: `msg.ack().await` em sucesso; `msg.ack_with(AckKind::Nak(None)).await` para erro recuperável (redelivery com backoff); `msg.ack_with(AckKind::Term).await` para mensagem corrompida/inválida
- Handler nunca deixa exceção/erro escapar sem ack/nak/term correspondente
- Consumers duráveis (pull) com `max_deliver`/`ack_wait`; idempotência no consumo (`INSERT ... ON CONFLICT DO NOTHING`)
- Subjects hierárquicos `events.<domínio>.<ação>`; um stream por domínio (`EVENTS_USER`)
- Lifecycle no composition root do consumer (`ensure_stream` + loop `messages().next()`); `worker.abort()` + drain no shutdown
- Publicar **depois** do commit no banco, nunca antes ou dentro da transaction

#### 4.10 Injeção de Dependência (Composition Root Manual)

- Não há framework de DI: cada binário compõe seu grafo **explicitamente** no `main`, bottom-up (infra → repos/publishers/adapters → use cases → controllers/subscribers)
- **Infraestrutura compartilhada primeiro**: `AppConfig::load()`, `pkg::postgres::new_pool`, `pkg::nats::connect`
- Cada binário compõe **apenas o que precisa** — uma API compõe controllers; um consumer compõe subscribers; nenhum compõe as dependências do outro
- Toda dependência injetada embrulhada em `Arc<dyn Trait>`; `Arc::clone` é barato (contador de referência)
- Construtores seguem convenção `new(deps...) -> Self`; o compilador garante em compile time que nenhuma dependência falta
- Graceful shutdown (SIGINT + SIGTERM) e ordem de cleanup inversa da criação (HTTP → NATS drain → Postgres `pool.close()`)
- Rotas/startup do servidor no `src/bin/api.rs`; `ensure_stream`/loop do subscriber no `src/bin/consumer.rs` — nunca em módulo compartilhado

#### 4.11 Testes

> Referência completa: [[rust-testing-best-practices]] e [[testcontainers-rust]].

| Critério | Verificar |
|---|---|
| Framework | `cargo test` nativo; `#[tokio::test]` para async |
| Unitários junto da impl | `#[cfg(test)] mod tests` no mesmo arquivo da implementação (ex.: no `create_usecase.rs`) |
| Independência/paralelismo | `cargo test` roda em paralelo por padrão — sem dependência de ordem nem estado compartilhado |
| Asserções | `assert!`/`assert_eq!`/`matches!` nativos |
| Mocks | `mockall` (`MockUserRepository`, gerado via `#[cfg_attr(test, mockall::automock)]`) apenas para dependências externas — sem over-mocking |
| Integração | testcontainers em crate de teste separado (`tests/`): PostgreSQL para repositories, NATS para publishers/subscribers; cleanup automático no drop dos containers |
| HTTP | subir a API/handlers e testar via cliente HTTP quando aplicável |
| Edge cases | input vazio, `None`, error paths, valores no limite; não apenas happy path |
| Data races | prevenidas em compile time (ownership); sinalizar `unsafe` injustificado |
| Cobertura | `cargo llvm-cov`; código novo coberto (threshold do projeto: 80%) |

### Etapa 5: Verificação Automatizada

Execute todas as ferramentas e registre saídas para anexar ao artefato:

```bash
# 1. Compilação — bloqueante
cargo build --workspace

# 2. Clippy — construções suspeitas, bugs potenciais, lints (warning = erro)
cargo clippy --workspace --all-targets -- -D warnings

# 3. Testes (unitários #[cfg(test)] + integração tests/)
cargo test --workspace

# 4. Cobertura (cargo-llvm-cov)
cargo llvm-cov --workspace --summary-only

# 5. Formatação
cargo fmt --all -- --check

# 6. Lockfile sincronizado
cargo build --workspace --locked

# 7. Auditoria de dependências (se configurado no projeto)
cargo audit
```

**Regras:**

- Qualquer falha em `cargo build` → **Crítico** automático
- Qualquer warning em `cargo clippy -- -D warnings` → **Major** (ou Crítico se apontar bug real/`unsafe`)
- Qualquer teste falhando → **Crítico**
- Bloco `unsafe` injustificado no código da feature → **Crítico** (justifique ou remova)
- Arquivos fora de padrão em `cargo fmt --all -- --check` → **Major**
- Cobertura abaixo do threshold do projeto (típico 80%) → **Major**
- `Cargo.lock` fora de sincronia (`--locked` falha) → **Major**

### Etapa 6: Classificar Problemas

Cada problema encontrado recebe **uma** classificação:

#### **CRÍTICO** (bloqueante para merge)

- Bugs funcionais
- Problemas de segurança (SQL injection via `format!` no SQL, exposição de credenciais, JWT mal validado)
- Funcionalidade quebrada (não compila, testes falham)
- **`unwrap()`/`expect()`/`panic!` em código de produção** (fora de testes/bootstrap)
- **Task leak** (`tokio::spawn` sem caminho de encerramento)
- **Bloqueio do runtime async** (I/O síncrono/CPU-bound no contexto async)
- Bloco `unsafe` injustificado
- `Result` descartado silenciosamente em caminho crítico (`let _ = ...`)
- Violações da regra de dependência da Clean Architecture (`domain/` importando `sqlx`/`axum`, `application/` importando `infrastructure/`, `pkg/` importando apps)
- Evento publicado dentro de transaction
- Logging de dados sensíveis (PII, senhas, tokens)
- Guard de `Mutex` (std) mantido através de `.await`

#### **MAJOR** (deve ser corrigido)

- Violações de idiomas Rust (camelCase, stuttering, `as` truncando, `Box<dyn>` onde cabe `impl Trait`)
- Padrões do projeto desrespeitados (construtor fora do padrão, arquivo em pasta errada, use case sem trait/`perform`, controller com lógica de negócio)
- Validação manual no handler em vez de `ValidatedJson`/`#[validate]`
- Erro de domínio não mapeado em `HttpDomainError` (match não exaustivo forçaria erro de compilação — mas mapeamento incorreto de status é Major)
- Testes ausentes para código novo
- Nomenclatura inadequada (nomes genéricos, abreviações ruins)
- Contexto de erro ausente na infra (sem `.context(...)`)
- Itens públicos (`pub`) sem doc comment
- Composition root incorreto (dependência construída fora do `main`, ordem errada, binário compondo o que não precisa)
- Cobertura significativamente abaixo do threshold
- `cargo fmt` não aplicado; warning de `cargo clippy`
- Mensagens de erro com Maiúscula inicial ou pontuação final
- `SELECT *` ou paginação por OFFSET em tabela grande
- `sqlx::FromRow` derivado no domínio (deveria ser row struct na infra)

#### **MINOR** (sugestão)

- Sugestões de estilo
- Otimizações opcionais (ex.: `clone()` evitável)
- Padrões não-idiomáticos que ainda funcionam
- Comentários redundantes
- Variáveis com nomes pouco descritivos em escopo amplo
- Funções um pouco longas mas ainda coesas

#### **POSITIVO** (reconhecer)

- Rust idiomático bem aplicado (ownership limpo, `?`, iteradores, enums fechados)
- Boa cobertura de testes incluindo edge cases
- Tratamento de erros consistente com `DomainError`/thiserror, `.context(...)` e `matches!`
- Uso correto de async/await e Tokio, sem bloquear o runtime
- Traits pequenas e focadas
- Estrutura aderente à Clean Architecture e ao composition root
- Doc comments completos
- Uso correto de `#[tokio::test]`, `mockall`, testcontainers
- Graceful shutdown bem implementado

### Etapa 7: Gerar/Atualizar o Artefato de Revisão

1. `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`
2. Crie ou **sobrescreva** `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md`

#### Formato Obrigatório

```markdown
# Revisão: Task [num] - [Título da Task]

**Revisor**: AI Rust Code Reviewer
**Data**: [AAAA-MM-DD]
**Arquivo da task**: .cognup/specs/[nome-da-funcionalidade]/tasks/[num]_task.md
**Status**: [APPROVED | APPROVED WITH OBSERVATIONS | CHANGES REQUESTED]
**Re-revisão**: [Não | Sim (após correções)]

## Resumo

[Resumo breve do que foi implementado e avaliação geral de qualidade]

## Skills Técnicas Utilizadas

| Skill | Utilizada | Observações |
|-------|-----------|-------------|
| rust-best-practices | Sim | [Breve nota] |
| rust-testing-best-practices | [Sim/N/A] | [Breve nota] |
| axum-best-practices | [Sim/N/A] | [Breve nota] |
| postgres-best-practices | [Sim/N/A] | [Breve nota] |
| nats-best-practices | [Sim/N/A] | [Breve nota] |
| testcontainers-rust | [Sim/N/A] | [Breve nota] |

## Arquivos Revisados

| Arquivo | Status | Problemas |
|---------|--------|-----------|
| [caminho do arquivo] | [OK / Problemas / Crítico] | [contagem] |

## Verificações Automatizadas

| Verificação | Comando | Resultado | Detalhes |
|---|---|---|---|
| Build | `cargo build --workspace` | OK / FALHOU | ... |
| Clippy | `cargo clippy --workspace --all-targets -- -D warnings` | OK / FALHOU | ... |
| Testes | `cargo test --workspace` | OK / FALHOU | ... |
| Cobertura | `cargo llvm-cov --workspace --summary-only` | [X]% | threshold: [Y]% |
| Formatação | `cargo fmt --all -- --check` | OK / FALHOU | ... |
| Lockfile | `cargo build --workspace --locked` | OK / FALHOU | ... |

## Problemas Encontrados

### Problemas Críticos

[Liste cada problema crítico com: arquivo, linha, descrição e correção sugerida com exemplo de código]
[Se nenhum: "Nenhum problema crítico encontrado."]

### Problemas Major

[Mesmo formato]

### Problemas Minor

[Mesmo formato]

## Destaques Positivos

[Liste o que foi bem feito — Rust idiomático, ownership limpo, cobertura, design limpo, etc.]

## Conformidade com Padrões

| Padrão | Status |
|--------|--------|
| Rust Idioms (rust-best-practices) | [OK / Atenção / Crítico] |
| Tratamento de Erros (thiserror/anyhow) | [OK / Atenção / Crítico] |
| Concorrência (Tokio) | [OK / Atenção / Crítico / N/A] |
| Clean Architecture (architecture.md) | [OK / Atenção / Crítico] |
| HTTP/Axum 0.8 | [OK / Atenção / Crítico / N/A] |
| PostgreSQL (sqlx, migrations, tx) | [OK / Atenção / Crítico / N/A] |
| NATS JetStream | [OK / Atenção / Crítico / N/A] |
| Logging (tracing) | [OK / Atenção / Crítico / N/A] |
| Composition Root (Arc<dyn Trait>) | [OK / Atenção / Crítico] |
| Testes | [OK / Atenção / Crítico] |

## Recomendações

[Lista numerada de recomendações priorizadas para melhoria]

## Veredicto

[Avaliação final com próximos passos claros. Se status ≠ APPROVED, deixe explícito que correções devem ser aplicadas e o task-reviewer executado novamente para atualizar este arquivo até obter APPROVED.]
```

## Critérios de Status

| Status | Quando Usar |
|---|---|
| **APPROVED** | Sem problemas críticos ou major. Código pronto para merge. **Único status que finaliza a revisão.** |
| **APPROVED WITH OBSERVATIONS** | Sem problemas críticos; problemas minor ou poucos majors não-bloqueantes. Requer correções e nova execução do reviewer até obter APPROVED. |
| **CHANGES REQUESTED** | Problemas críticos OU múltiplos majors bloqueantes. Requer correções e nova execução do reviewer. |

Os status são escritos **em inglês** — é o valor que o workflow `run-task.md` e o agente `executor-bug` verificam para decidir o próximo passo do ciclo.

## Re-revisão Após Correções

Quando o workflow `run-task.md` aplica correções após **APPROVED WITH OBSERVATIONS** ou **CHANGES REQUESTED**, este reviewer será invocado novamente:

1. O arquivo `tasks/reviews/[num]_task_review.md` já existirá — **trate como re-revisão**
2. Execute a revisão completa sobre o código **atual** (com as correções aplicadas)
3. **Sobrescreva** o arquivo com a nova avaliação — não deixe o status antigo no lugar
4. No cabeçalho, marque `**Re-revisão**: Sim (após correções)`
5. Defina **APPROVED** apenas quando todos os problemas bloqueantes estiverem resolvidos; caso contrário use APPROVED WITH OBSERVATIONS ou CHANGES REQUESTED para que o ciclo continue

Fluxo esperado quando status ≠ APPROVED:

1. Desenvolvedor (ou agente de correção) aplica fixes
2. Reviewer é executado novamente
3. `tasks/reviews/[num]_task_review.md` é **sobrescrito** com novo resultado
4. Repetir até **APPROVED**

## Diretrizes Operacionais

1. **Caminho de saída fixo** — o review vai SEMPRE em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md`
2. **Ler antes de julgar** — leia integralmente os arquivos modificados, não apenas o diff
3. **Seja específico** — sempre referencie arquivo e número da linha
4. **Forneça solução** — não apenas aponte; sugira correção com exemplo de código
5. **Verifique testes** — código novo sem teste correspondente é **Major** no mínimo
6. **Execute as verificações** — `cargo build`, `cargo clippy -- -D warnings`, `cargo test`, `cargo fmt --check`
7. **Cheque contra requisitos** — o implementado corresponde ao solicitado em `[num]_task.md` e no `techspec.md`?
8. **Aplique [[rust-best-practices]]** para validar idiomas Rust; **[[rust-testing-best-practices]]** para testes; e as skills de domínio aplicáveis ao diff
9. **Aplique as regras do projeto** (`.claude/rules/architecture.md`, `CLAUDE.md`)
10. **Reconheça o bom** — destaques positivos são parte do artefato
11. **Re-revisão sobrescreve** — não acumule reviews; o arquivo sempre reflete o estado mais recente
12. **Idioma** — artefato em Português (BR); exemplos de código e valores de Status em inglês
13. **Seja justo** — revisão é construtiva; padrões existem para ajudar, não para humilhar
14. **Atualize memória** — registre padrões recorrentes do projeto para revisões futuras

## Anti-Patterns de Task Review (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "LGTM 👍" sem revisar | Toda revisão produz artefato `[num]_task_review.md` |
| Gravar o review fora de `tasks/reviews/` | Caminho fixo: `.cognup/specs/[feature]/tasks/reviews/` |
| Status em português (APROVADO) | Status em inglês (APPROVED) — é o que o workflow verifica |
| Revisar apenas o diff (sem ler arquivos completos) | Ler contexto completo dos arquivos modificados |
| "Tudo certo" sem rodar testes | Sempre executar `cargo build`, `cargo clippy`, `cargo test` |
| Apontar problema sem solução | Sempre sugerir correção com exemplo de código |
| Ignorar testes ausentes | Código novo sem teste = Major |
| Não classificar problemas | Toda observação tem severidade explícita (Crítico/Major/Minor/Positivo) |
| Aprovar com testes falhando | Falha de teste = Crítico, CHANGES REQUESTED |
| Aprovar com `cargo build` quebrado | Build falha = Crítico imediato |
| Tolerar warnings do clippy | `-D warnings` — warning é falha |
| Aceitar `unwrap()`/`expect()`/`panic!` em produção | Apenas em testes ou bootstrap do `main` — caso contrário Crítico |
| Aceitar `unsafe` injustificado | Justifique ou remova — Crítico |
| Manter review antigo após correções | Re-revisão **sobrescreve** o arquivo |
| Reportar status sem rastreabilidade | Cada decisão tem referência a arquivo:linha + comando executado |
| Importar padrões de outro projeto | Validar contra os padrões **deste** projeto (`.claude/rules/`, skills) |
| Carregar todas as skills sempre | Carregar apenas as aplicáveis ao escopo do diff |
| Só apontar erros, ignorar acertos | Sempre incluir seção "Destaques Positivos" |
| Aceitar `let _ =` engolindo `Result` | Propagar com `?` ou tratar — descartar silenciosamente é Crítico em caminho crítico |
| Aceitar evento publicado dentro da tx | Publicar **depois** do `tx.commit()` |

## Cheat Sheet — Comandos do Reviewer

```bash
# Identificar tarefa
ls .cognup/specs/*/tasks/*_task.md
ls -la .cognup/specs/[feature]/tasks/reviews/*_task_review.md 2>/dev/null   # detectar re-revisão

# Identificar arquivos alterados (Gitflow: base develop)
git diff develop...HEAD --stat
git log develop...HEAD --oneline
git status --short
git diff HEAD --name-only

# Ler diff completo de um arquivo
git diff develop...HEAD -- apps/user/src/application/usecase/user/create_usecase.rs

# Verificações automatizadas
cargo build --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo llvm-cov --workspace --summary-only          # cobertura total
cargo fmt --all -- --check
cargo build --workspace --locked
cargo audit

# Testes específicos do diff
cargo test -p user create_usecase
cargo test -p user --test user_integration_test

# Identificar handlers HTTP modificados
git diff develop...HEAD --name-only -- 'apps/*/src/infrastructure/controller/*'

# Identificar migrations modificadas
git diff develop...HEAD --name-only -- 'migrations/*'

# Identificar testes (unitários no arquivo da impl + integração em tests/)
git diff develop...HEAD --name-only -- 'apps/*/tests/*'

# Criar a pasta do review e gravar o artefato
mkdir -p .cognup/specs/[feature]/tasks/reviews

# Verificar OpenAPI (gerado em compile time pelo utoipa) — se API
curl -s http://localhost:8080/api-docs/openapi.json | jq '.paths | keys'
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/swagger-ui/
```

## Idioma

Artefato de revisão em **Português (Brasil)**. Exemplos de código e valores de **Status** permanecem em **inglês**.

## Manutenção de Memória do Revisor

Ao longo das revisões, registre padrões e violações recorrentes para construir conhecimento institucional sobre o codebase:

- Violações recorrentes entre tarefas (ex.: time se esquece de usar `.context(...)` nos repositórios)
- Padrões arquiteturais consolidados no projeto (camadas, organização por contexto de domínio, composition root)
- Abordagens comuns de teste e lacunas frequentes (mockall, testcontainers)
- Convenções de nomenclatura efetivamente em uso (vs. as documentadas)
- Dependências e crates adotados
- Padrões de tratamento de erros consolidados (hierarquia `DomainError`, `Unexpected(#[from] anyhow::Error)`)
- Padrões de concorrência (tasks Tokio, cancelamento, graceful shutdown)
- Padrões de composição manual com `Arc<dyn Trait>` e violações recorrentes
- Endpoints/recursos mais propensos a regressões

Essas notas tornam revisões futuras mais rápidas e consistentes.
