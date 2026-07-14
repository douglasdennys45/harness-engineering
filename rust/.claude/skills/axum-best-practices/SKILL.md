---
name: axum-best-practices
description: Aplica boas praticas oficiais do Axum 0.8 (Tokio/Tower) ao desenhar, escrever, revisar ou refatorar controllers HTTP, rotas, extractors, middlewares e respostas em Rust. Cobre a infra generica do projeto em pkg/controller (RequestContext, ValidatedJson, ApiError, HttpDomainError, ErrorResponse), organizacao de Router por controller (routes(self) -> Router, with_state, merge/nest, versionamento /v1), sintaxe de path do Axum 0.8 ({id} e {*rest}), handlers como associated functions recebendo State<Arc<Self>> + extractors na ordem correta (body/ValidatedJson sempre por ultimo), extracao tipada (Path, Query, State), validacao declarativa com o crate validator (#[validate], nested, tipos fortes Uuid/enum serde), tratamento uniforme de erros (DomainError -> ApiError via HttpDomainError, nunca vazar 5xx), transactions no controller (pool.begin + *_with_tx, evento apos commit), pilha padrao de middlewares tower-http (SetRequestId, Trace, CatchPanic, Cors, Compression, Timeout) e a armadilha do .layer envolver apenas rotas ja adicionadas, documentacao OpenAPI com utoipa (#[utoipa::path], derive OpenApi, swagger-ui), graceful shutdown (axum::serve + with_graceful_shutdown, SIGINT/SIGTERM), observabilidade (tracing com request_id) e testes de controller. Use sempre que criar/editar controllers, rotas, extractors, middlewares ou handlers HTTP em apps/<app>/src/infrastructure/controller/ e no composition root src/bin/api.rs, ao definir contratos de request/response, mapear erros de dominio para HTTP status, ou revisar qualquer codigo que importe o crate axum.
---

# Axum 0.8 Best Practices (Rust · Tokio · Tower)

Skill baseada na documentacao oficial do Axum 0.8 (docs.rs/axum), no ecossistema Tower/tower-http e nos padroes internos do projeto (`architecture.md`). Cobre padroes de producao para a camada HTTP: controllers, rotas, extractors, validacao, erros, middlewares, OpenAPI e shutdown. Toda API do projeto usa a infra generica compartilhada de `pkg/controller/` — controllers so implementam a logica especifica.

## Quando Aplicar

- Ao criar/editar controllers em `apps/<app>/src/infrastructure/controller/`
- Ao definir rotas, extractors, middlewares ou handlers HTTP
- Ao modelar contratos de request (structs de input com `validator`) e response
- Ao mapear erros de dominio para HTTP status (`HttpDomainError`)
- Ao decidir se um endpoint precisa de transaction (`pool.begin`)
- Ao montar o Router no composition root (`src/bin/api.rs`)
- Ao documentar endpoints com OpenAPI (utoipa)
- Ao configurar middlewares (tower-http) ou graceful shutdown
- Ao revisar qualquer codigo que importe o crate `axum`

## 1. Onde Vive Cada Coisa

| Responsabilidade | Local | Observacao |
|---|---|---|
| Infra generica HTTP (sem dominio) | `pkg/controller/` + `pkg/httpserver/` | Reutilizada por todos os apps |
| `RequestContext`, `ValidatedJson`, `ApiError`, `HttpDomainError`, `ErrorResponse` | `pkg/controller/` | **Nunca** reimplementar no app |
| Pilha padrao de middlewares | `pkg/httpserver/axum.rs` | `with_default_middlewares(router)` |
| Controllers concretos | `apps/<app>/src/infrastructure/controller/<name>_controller.rs` | So logica especifica |
| Mapeamento de erros do app | `apps/<app>/src/infrastructure/controller/errors.rs` | `impl HttpDomainError for DomainError` |
| Registro de rotas + startup | `apps/<app>/src/bin/api.rs` | Composition root |

**Regra de ouro:** o `pkg/controller/` nao conhece **nenhum** erro de dominio — ele so recebe o mapeamento via trait. Cada app registra seus erros implementando `HttpDomainError`.

## 2. Router e Rotas

Cada controller expoe `pub fn routes(self) -> Router`, que embrulha o controller em `Arc` e registra as rotas com `with_state`. O composition root faz o `merge` de todos os controllers e aplica os middlewares por ultimo.

```rust
pub fn routes(self) -> Router {
    let ctrl = Arc::new(self);
    Router::new()
        .route("/v1/users", post(Self::create).get(Self::list))
        .route("/v1/users/{id}", get(Self::get_by_id))   // sintaxe {id} do Axum 0.8
        .with_state(ctrl)
}
```

### Regras de rotas

| Regra | Detalhe |
|---|---|
| **Sintaxe de path 0.8** | `{id}` para param, `{*rest}` para wildcard (o `:id`/`*rest` do 0.7 foi removido) |
| **Versionamento** | Prefixo `/v1` em toda rota publica; nova versao = novo prefixo, sem quebrar a antiga |
| **Metodos no mesmo path** | Encadear: `post(Self::create).get(Self::list)` |
| **`with_state` por controller** | Cada `routes()` fecha seu proprio state; nao vazar state entre controllers |
| **`merge` vs `nest`** | `merge` para juntar routers de mesmo nivel; `nest("/admin", r)` para prefixar um sub-router |
| **Handlers privados** | Associated functions `async fn` — nao precisam ser `pub` |
| **`.layer()` por ultimo** | Middlewares aplicados **depois** de todas as rotas (ver secao 9) |

### Composicao no `src/bin/api.rs`

```rust
let app = pkg::httpserver::with_default_middlewares(
    Router::new()
        .merge(user_controller.routes())
        .merge(order_controller.routes()),
);
```

## 3. Handlers

Handlers sao **associated functions** (`async fn`) que recebem `State<Arc<Self>>` + extractors e retornam `Result<impl IntoResponse, ApiError>`.

```rust
async fn create(
    State(ctrl): State<Arc<UserController>>,   // 1. State primeiro
    rc: RequestContext,                         // 2. extractors de partes (headers)
    ValidatedJson(input): ValidatedJson<CreateUserInput>,  // 3. body SEMPRE por ultimo
) -> Result<impl IntoResponse, ApiError> {
    let output = ctrl
        .create_use_case
        .perform(CreateInput { name: input.name, email: input.email })
        .await?;   // DomainError -> ApiError via HttpDomainError

    tracing::info!(request_id = %rc.request_id, user_id = %output.user.id, "user created");
    Ok((StatusCode::CREATED, Json(output.user)))
}
```

### Ordem dos extractors — **critico**

No Axum, o extractor que consome o **body** (`ValidatedJson`, `Json`, `String`, `Bytes`, `Form`) implementa `FromRequest` e **deve ser o ultimo argumento**. Todos os outros (`State`, `Path`, `Query`, `RequestContext`) implementam `FromRequestParts` e vem antes. Colocar o body no meio nao compila.

| Extractor | Trait | Posicao |
|---|---|---|
| `State<T>` | `FromRequestParts` | Qualquer (convencao: 1o) |
| `Path<T>` | `FromRequestParts` | Antes do body |
| `Query<T>` | `FromRequestParts` | Antes do body |
| `RequestContext` | `FromRequestParts` | Antes do body |
| `ValidatedJson<T>` / `Json<T>` | `FromRequest` | **Ultimo, sempre** |

### Regras de handlers

- Handlers **delegam** logica de negocio para use cases. **Zero** logica de negocio no controller.
- Recebem `RequestContext` ja populado — nunca extrair `request_id`, IP, `user-agent` manualmente.
- Erros de dominio propagados com `?` — a conversao `From<DomainError> for ApiError` cuida do status HTTP.
- Retorno tipado com tupla `(StatusCode, Json(...))` para status explicito (`201`, `200`).

## 4. Extractors

| Extractor | Uso | Exemplo |
|---|---|---|
| `State<Arc<Self>>` | Acesso ao controller e suas deps | `State(ctrl): State<Arc<UserController>>` |
| `Path<T>` | Path params tipados | `Path(id): Path<Uuid>` |
| `Query<T>` | Query string -> struct | `Query(filter): Query<ListUsersFilter>` |
| `ValidatedJson<T>` | Body JSON + validacao | `ValidatedJson(input): ValidatedJson<CreateUserInput>` |
| `RequestContext` | request_id/ip/user_agent | `rc: RequestContext` |

- `Path<Uuid>` valida o formato do UUID **no parse** — path invalido vira 400 automaticamente.
- `Query<T>` com `Option<...>` para params opcionais (cursor, limit).
- **Nunca** usar `serde_json::from_*` manual no handler — sempre extractors.
- **Nunca** usar `Json<T>` cru para body de escrita: usar `ValidatedJson<T>` para garantir validacao.

## 5. Validacao (crate `validator`)

O `ValidatedJson<T>` faz parse + validacao das tags `#[validate]` automaticamente; o handler so roda se o body for valido. Falha de parse ou validacao vira **400** formatado (`format_validation_errors`), sem vazar erros crus do serde/validator.

```rust
#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateOrderInput {
    pub user_id: Uuid,                       // Uuid valida no parse
    #[validate(range(min = 1))]
    pub total_cents: i64,
    pub currency: Currency,                  // enum serde fechado (equivale ao oneof)
    #[validate(length(max = 500))]
    pub notes: Option<String>,               // opcional: so valida se Some
    #[validate(length(min = 1), nested)]
    pub items: Vec<CreateItemInput>,         // valida cada item
}
```

| Atributo | Uso |
|---|---|
| `#[validate(length(min = 3, max = 100))]` | Tamanho de string |
| `#[validate(email)]` | Email valido |
| `#[validate(range(min = 1))]` | Faixa numerica |
| `#[validate(url)]` | URL valida |
| `#[validate(nested)]` | Valida cada elemento de `Vec`/struct aninhada |
| `#[validate(custom(function = "..."))]` | Validacao propria |
| `Option<T>` | So valida se preenchido |
| tipo `Uuid` / enum serde | Rejeicao no parse (formato / valor fora do conjunto) |

### Regras de validacao

- **Sempre** validar via atributos ou tipos fortes (`Uuid`, enum serde). **Nunca** validar manualmente no handler.
- Structs de input: `#[derive(Debug, Deserialize, Validate, ToSchema)]` + `#[serde(rename_all = "camelCase")]`.
- IDs como `Uuid`, enumeracoes como enum serde — nao validar no use case o que o tipo ja garante.
- Nunca expor mensagens cruas do validator/serde — o `format_validation_errors` do `pkg` padroniza.

## 6. Tratamento de Erros

A cadeia e: handler retorna `Result<_, ApiError>` → `?` converte `DomainError` em `ApiError` via `HttpDomainError` → `IntoResponse` do `ApiError` serializa `ErrorResponse` e loga (5xx como `error`, 4xx como `warn`).

```rust
// apps/user/src/infrastructure/controller/errors.rs
impl HttpDomainError for DomainError {
    fn status_code(&self) -> StatusCode {
        match self {
            DomainError::InvalidInput      => StatusCode::BAD_REQUEST,
            DomainError::UserNotFound      => StatusCode::NOT_FOUND,
            DomainError::UserAlreadyExists => StatusCode::CONFLICT,
            DomainError::InvalidRole       => StatusCode::UNPROCESSABLE_ENTITY,
            DomainError::Forbidden         => StatusCode::FORBIDDEN,
            DomainError::Unexpected(_)     => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

### Regras de erro

| Regra | Detalhe |
|---|---|
| **`match` exaustivo** | Nova variante de erro = o compilador **obriga** a mapear o status (sem 500 silencioso) |
| **Nunca vazar 5xx** | `ApiError` substitui a mensagem de `is_server_error()` por `"internal server error"` |
| **404 via `Option`** | `.perform(...).await?.ok_or(DomainError::UserNotFound)?` |
| **Regra de negocio = variante explicita** | `Err(DomainError::UserAlreadyExists)`, nunca `Unexpected` |
| **Infra propaga com contexto** | `.context("acao")?` no use case vira `DomainError::Unexpected` |
| **Log no boundary** | O `IntoResponse` do `ApiError` ja loga; nao duplicar log de erro no handler |

Detalhes de dominio (thiserror/anyhow) vivem em `rust-best-practices` e no mapeamento de status em `architecture.md`.

## 7. Respostas

- Sucesso com corpo: `(StatusCode::CREATED, Json(entidade))` / `(StatusCode::OK, Json(output))`.
- Sucesso sem corpo: `StatusCode::NO_CONTENT`.
- Entidades/outputs serializados com `#[serde(rename_all = "camelCase")]` (JSON camelCase; ver `architecture.md`).
- Erro: **sempre** `ErrorResponse { error, status }` via `ApiError` — nunca montar JSON de erro ad-hoc.
- `impl IntoResponse` como tipo de retorno do caminho feliz (deixa o Axum inferir o corpo).

## 8. Transactions no Controller

Endpoints com **multiplas escritas atomicas** orquestram a transaction no handler via `pool.begin()` + metodos `*_with_tx` dos repositorios **concretos**. O controller recebe `Arc<PostgresXRepository>` (tipo concreto) + `PgPool` apenas nesse caso.

```rust
let mut tx = ctrl.pool.begin().await.context("begin tx").map_err(DomainError::from)?;
let order = ctrl.order_repo.create_with_tx(&mut tx, &Order::new(input.user_id, input.total_cents))
    .await.map_err(DomainError::from)?;
// ... mais operacoes na MESMA tx
tx.commit().await.context("commit tx").map_err(DomainError::from)?;   // rollback automatico no drop se falhar antes

let _ = ctrl.order_event.publish_created(&order).await;   // evento SEMPRE apos o commit
```

| Cenario | Transaction? |
|---|---|
| GET / leitura | Nao |
| POST/PUT com 1 escrita | Nao (use case faz a operacao unica) |
| POST com N INSERTs relacionados | **Sim** (`pool.begin` + `*_with_tx`) |
| PUT que move saldo (debito + credito) | **Sim** |
| DELETE com cascade manual | **Sim** |
| Operacao + evento | Evento **FORA** da tx, apos `commit` |

Regras completas de transaction/sqlx: [[postgres-best-practices]]. Publicacao de eventos: [[nats-best-practices]].

## 9. Middlewares (tower-http)

**Armadilha central do Axum:** `.layer()` envolve apenas as rotas **ja adicionadas** ao `Router`. Por isso o `pkg` expoe `with_default_middlewares(router)` que aplica a pilha **sobre o Router pronto** — chamada por ultimo no composition root, depois de todos os `merge`.

```rust
// pkg/src/httpserver/axum.rs
pub fn with_default_middlewares(router: Router) -> Router {
    router.layer(
        ServiceBuilder::new()
            .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))   // request id
            .layer(PropagateRequestIdLayer::x_request_id())
            .layer(TraceLayer::new_for_http())                         // logger
            .layer(CatchPanicLayer::new())                             // recover -> 500
            .layer(CorsLayer::permissive())                            // cors (ajustar por ambiente)
            .layer(CompressionLayer::new())                            // compress
            .layer(TimeoutLayer::new(Duration::from_secs(10))),        // timeout
    )
}
```

### Regras de middleware

| Regra | Detalhe |
|---|---|
| **Aplicar por ultimo** | `with_default_middlewares` depois de todos os `merge`/`nest` |
| **Ordem importa** | `ServiceBuilder` executa top-down na request; request id antes do trace para logar o id |
| **`SetRequestId` -> `RequestContext`** | O layer popula `x-request-id`, lido pelo extractor `RequestContext` |
| **`CatchPanic`** | Converte panico em 500 sem derrubar o servidor |
| **`Cors` por ambiente** | `permissive()` so em dev; restringir origins em producao |
| **`Timeout`** | Ceil global de request; alinhar com `ack_wait`/deadlines downstream |
| **Body limit** | Manter `DefaultBodyLimit` (2 MB) ou ajustar explicitamente para uploads |

## 10. State e Injecao de Dependencia

- O state de cada controller e `Arc<Self>`, contendo use cases como `Arc<dyn Trait>` (ou repos concretos + `PgPool` quando ha transaction).
- `with_state` fecha o state no `routes()`; o composition root injeta as deps no `new(...)` (bottom-up).
- `Arc::clone` e barato (contador de referencia) — clonar deps entre controllers e ok.
- **Nunca** usar `static`/`OnceLock` para pool, client ou state — tudo passa pelo composition root (ver `architecture.md`).

## 11. OpenAPI (utoipa)

A documentacao e gerada em **compile time** (sem passo de CLI). Cada handler recebe `#[utoipa::path(...)]`; o app agrega tudo num `#[derive(OpenApi)]` e serve `/swagger-ui` + `/api-docs/openapi.json`.

```rust
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
async fn create(/* ... */) -> Result<impl IntoResponse, ApiError> { /* ... */ }
```

### Regras OpenAPI

- Todo handler publico documentado com `#[utoipa::path]` (metodo, path, tag, request_body, responses).
- Structs de request/response derivam `ToSchema`.
- Sempre documentar as respostas de erro relevantes com `body = pkg::controller::ErrorResponse`.
- Registrar o novo handler no `#[derive(OpenApi)]` do app (`paths(...)`) e conferir o `/swagger-ui`.

## 12. Servidor e Graceful Shutdown

```rust
let listener = tokio::net::TcpListener::bind(("0.0.0.0", cfg.app_port)).await.context("bind listener")?;
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())   // SIGINT + SIGTERM
    .await
    .context("server failed")?;

// cleanup na ordem inversa da criacao
drop(nats);
pool.close().await;
```

- `axum::serve` (nao mais `axum::Server` do 0.6). `into_make_service()` implicito quando o segundo arg e um `Router`.
- `with_graceful_shutdown(shutdown_signal())` cobrindo `ctrl_c` **e** `SIGTERM` (ver `shutdown_signal` em `architecture.md`).
- Cleanup no fim do `main`, sempre na ordem inversa: HTTP para → NATS drain → Postgres fecha.

## 13. Observabilidade

- Logar com `tracing` em pontos-chave usando `request_id = %rc.request_id` (correlacao ponta a ponta).
- Sucesso relevante: `tracing::info!` com IDs de entidade; erros ja sao logados pelo `IntoResponse` do `ApiError`.
- `TraceLayer::new_for_http()` gera spans por request; nao duplicar logs de entrada/saida no handler.
- Nunca logar dados sensiveis (senha, token, PII) nos campos estruturados.

## 14. Testes de Controller

- Logica de negocio e testada no use case (mockall) — ver [[rust-testing-best-practices]].
- Controllers/rotas: testar via `Router` com `tower::ServiceExt::oneshot` (unit) ou fluxo E2E com servidor real + `reqwest`/`curl` (integracao, ver [[testcontainers-rust]] e `qa-best-practices`).
- Cobrir: status HTTP correto por caminho (201/200/400/404/409), corpo `ErrorResponse` em falhas, validacao rejeitando input invalido (400), e o path param invalido (400).

```rust
use tower::ServiceExt;   // para .oneshot
// let res = app.oneshot(Request::builder().uri("/v1/users/{invalid}").body(Body::empty())?).await?;
// assert_eq!(res.status(), StatusCode::BAD_REQUEST);
```

## Checklist — Novo Controller

- [ ] Arquivo `infrastructure/controller/<name>_controller.rs`, struct `<Name>Controller`
- [ ] Deps como campos: use cases `Arc<dyn Trait>` (repos concretos + `PgPool` so se houver tx)
- [ ] Construtor `new(deps...) -> Self`
- [ ] `pub fn routes(self) -> Router` com `Arc::new(self)` + `with_state`
- [ ] Paths versionados `/v1/...` com sintaxe `{id}` do Axum 0.8
- [ ] Handlers `async fn` privados retornando `Result<impl IntoResponse, ApiError>`
- [ ] Extractors na ordem certa (body/`ValidatedJson` **por ultimo**)
- [ ] `RequestContext` para request_id/ip (nunca extrair manual)
- [ ] Input com `#[derive(Deserialize, Validate, ToSchema)]` + `camelCase` + `#[validate]`
- [ ] `?` para propagar `DomainError` (sem logica de negocio no handler)
- [ ] Novas variantes de erro mapeadas em `controller/errors.rs`
- [ ] Transaction so quando ha N escritas; evento **apos** `commit`
- [ ] `#[utoipa::path]` + `ToSchema` + registro no `#[derive(OpenApi)]`
- [ ] `merge` do `routes()` no `src/bin/api.rs`, middlewares aplicados por ultimo

## Checklist — Router / Composition Root (API)

- [ ] `Router::new().merge(<ctrl>.routes())...` para todos os controllers
- [ ] `with_default_middlewares(router)` chamado **por ultimo**
- [ ] `axum::serve(listener, app).with_graceful_shutdown(shutdown_signal())`
- [ ] `shutdown_signal()` cobre SIGINT + SIGTERM
- [ ] Cleanup na ordem inversa (HTTP → NATS → Postgres)
- [ ] `/swagger-ui` e `/api-docs/openapi.json` respondendo

## Fontes

- https://docs.rs/axum/0.8/axum/ — API reference (Router, extractors, State, IntoResponse)
- https://docs.rs/axum/0.8/axum/extract/index.html — extractors e ordem (FromRequest vs FromRequestParts)
- https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md — mudancas 0.7 -> 0.8 (path `{id}`, async fn nativo)
- https://docs.rs/tower-http/latest/tower_http/ — middlewares (trace, cors, compression, timeout, request-id, catch-panic)
- https://docs.rs/utoipa/latest/utoipa/ — OpenAPI (`#[utoipa::path]`, `ToSchema`, `OpenApi`)
- https://docs.rs/validator/latest/validator/ — atributos de validacao
