# Arquitetura da Aplicacao

Este documento define a arquitetura do projeto, suas camadas, regras de dependencia, convencoes de nomenclatura e o guia passo a passo para adicionar novas features.

## Visao Geral

O projeto segue **Clean Architecture** com inversao de dependencia em uma estrutura de **monorepo** com multiplos microservicos. Camadas internas nunca importam camadas externas. Injecao de dependencia e gerenciada pelo **uber-go/fx**. O monorepo suporta **N apps independentes** (APIs e Consumers) que compartilham bibliotecas via `pkg/`.

```
apps/<app-name>/cmd/<binary>/main.go    Entry points (um por binario)
       |
       v
apps/<app-name>/internal/domain/        Entidades, interfaces (sem dependencias externas)
apps/<app-name>/internal/application/   Implementacoes de use cases (depende apenas de domain)
apps/<app-name>/internal/infrastructure/ Controllers, repos, publisher, subscriber, config
       |
       v
pkg/                                    Bibliotecas reutilizaveis compartilhadas entre apps
```

### Stack Tecnologica

| Camada | Tecnologia | Versao |
|---|---|---|
| Linguagem | Go | **1.26** |
| HTTP | Fiber v3 | `github.com/gofiber/fiber/v3` |
| Banco de Dados | PostgreSQL | **18** |
| Mensageria | NATS JetStream | **2.12** |
| DI / Lifecycle | uber-go/fx | latest |
| Logging | log/slog (JSON estruturado) | stdlib |
| Documentacao | Swagger (swaggo) | latest |
| Validacao | go-playground/validator/v10 | **v10** |
| Config | godotenv + variaveis de ambiente | - |
| Testes | testcontainers-go, testify, gomock | latest |

### Tipos de Binarios

| Tipo | Proposito | Exemplo |
|---|---|---|
| **API** | Servidor HTTP expondo endpoints REST (Fiber v3) | `apps/user/cmd/api/main.go` |
| **Consumer** | Processo em background consumindo eventos NATS JetStream | `apps/user/cmd/consumer/main.go` |

### Regra de Dependencia

```
Domain  <--  Application  <--  Infrastructure  <--  cmd/
  ^                                  |
  |                                  v
  +--- nunca depende de -------->  pkg/
```

- `domain/` importa apenas stdlib.
- `application/` importa apenas `domain/`.
- `infrastructure/` importa `domain/`, `pkg/` e bibliotecas externas.
- `pkg/` nunca importa `internal/` de nenhum app.
- `cmd/<binary>/` importa tudo para compor o grafo de dependencias FX **daquele binario especifico**.

---

## Estrutura de Diretorios

```
/
├── go.work                                          # Go workspace
├── go.work.sum
├── docker-compose.yml
├── Makefile
│
├── pkg/                                             # module: github.com/org/project/pkg
│   ├── go.mod
│   ├── postgres/
│   │   ├── conn.go                                  # NewConnection + pool config
│   │   ├── tx.go                                    # RunInTx helper
│   │   └── health.go                                # Ping health check
│   ├── nats/
│   │   ├── conn.go                                  # NewConnection + reconnect
│   │   ├── stream.go                                # EnsureStream
│   │   ├── publisher.go                             # Publisher generico
│   │   └── subscriber.go                            # Subscribe com consumer duravel
│   ├── httpserver/
│   │   └── fiber.go                                 # NewFiberApp com middlewares padrao
│   ├── controller/
│   │   ├── base.go                                  # BaseController (template method)
│   │   ├── context.go                               # RequestContext struct
│   │   ├── bind.go                                  # BindAndHandle generic
│   │   ├── validator.go                             # FiberValidator (go-playground/validator/v10)
│   │   └── errors.go                                # ErrorMapper + ErrorResponse
│   ├── logger/
│   │   └── logger.go                                # slog JSON setup com service e version
│   └── event/                                       # contratos de eventos compartilhados
│       ├── event.go                                 # Envelope base
│       ├── user.go                                  # subjects + data structs
│       └── billing.go                               # subjects + data structs
│
├── migrations/                                      # migrations SQL por app
│   ├── user/
│   │   ├── 000001_create_users.up.sql
│   │   └── 000001_create_users.down.sql
│   └── billing/
│       ├── 000001_create_accounts.up.sql
│       └── 000001_create_accounts.down.sql
│
├── apps/
│   ├── user/                                        # module: github.com/org/project/apps/user
│   │   ├── go.mod
│   │   ├── Dockerfile
│   │   ├── cmd/
│   │   │   ├── api/
│   │   │   │   └── main.go                          # API HTTP — composicao FX
│   │   │   └── consumer/
│   │   │       └── main.go                          # Consumer NATS — composicao FX
│   │   ├── internal/
│   │   │   ├── domain/
│   │   │   │   ├── entity/
│   │   │   │   │   └── user.go                      # Struct de dominio + construtor
│   │   │   │   ├── usecase/
│   │   │   │   │   └── user/                        # Subpasta por contexto de dominio
│   │   │   │   │       ├── create.go                # CreateUseCase interface
│   │   │   │   │       ├── get-by-id.go             # GetByIDUseCase interface
│   │   │   │   │       └── list.go                  # ListUseCase interface
│   │   │   │   ├── repository/
│   │   │   │   │   └── user-repository.go           # Interface de repositorio
│   │   │   │   ├── event/
│   │   │   │   │   └── user-event.go                # Interface de eventos/mensageria
│   │   │   │   └── <lib>/                           # Interfaces para libs externas
│   │   │   │       └── <name>.go                    # Interface de inversao de dependencia
│   │   │   ├── application/
│   │   │   │   └── usecase/
│   │   │   │       └── user/                        # Subpasta por contexto de dominio
│   │   │   │           ├── create-usecase.go        # Implementacao do use case
│   │   │   │           ├── create-usecase_test.go   # Teste unitario
│   │   │   │           ├── get-by-id-usecase.go
│   │   │   │           └── list-usecase.go
│   │   │   └── infrastructure/
│   │   │       ├── config/
│   │   │       │   ├── config.go                    # AppConfig + LoadConfig
│   │   │       │   └── module.go                    # fx.Module compartilhado (providers de infra)
│   │   │       ├── controller/
│   │   │       │   └── user-controller.go           # Controllers HTTP (Fiber v3) — apenas APIs
│   │   │       ├── repository/
│   │   │       │   ├── user-repository.go           # Implementacao PostgreSQL
│   │   │       │   └── user-repository_integration_test.go
│   │   │       ├── publisher/
│   │   │       │   ├── user-publisher.go            # Implementacao NATS JetStream
│   │   │       │   └── user-publisher_integration_test.go
│   │   │       ├── subscriber/
│   │   │       │   └── user-subscriber.go           # Consumer NATS — apenas Consumers
│   │   │       └── adapter/
│   │   │           └── <lib>-adapter.go             # Implementacoes de interfaces de libs externas
│   │   └── test/
│   │       └── integration/
│   │           ├── testhelper/
│   │           │   ├── postgres.go                  # Container PG compartilhado
│   │           │   └── nats.go                      # Container NATS compartilhado
│   │           └── user_integration_test.go         # Teste E2E do fluxo completo
│   │
│   └── billing/                                     # mesma estrutura interna
│       ├── go.mod
│       ├── Dockerfile
│       ├── cmd/
│       │   ├── api/
│       │   │   └── main.go
│       │   └── consumer/
│       │       └── main.go
│       ├── internal/
│       │   ├── domain/
│       │   ├── application/
│       │   └── infrastructure/
│       └── test/
│           └── integration/
```

### Go Workspace (`go.work`)

```go
go 1.26

use (
    ./pkg
    ./apps/user
    ./apps/billing
)
```

Cada app importa `pkg` no seu `go.mod`:

```go
// apps/user/go.mod
module github.com/org/project/apps/user

go 1.26

require github.com/org/project/pkg v0.0.0
```

### Convencao de Nomes para `apps/` e `cmd/`

- **Apps**: `apps/<app-name>/` -- lowercase (ex: `apps/user/`, `apps/billing/`).
- **APIs**: `apps/<app>/cmd/api/main.go`.
- **Consumers**: `apps/<app>/cmd/consumer/main.go`. Se houver multiplos consumers, usar `cmd/consumer-<dominio>/main.go`.
- Cada diretorio `cmd/<binary>/` contem **apenas** `main.go`. Nenhum outro arquivo.

---

## Camada de Dominio (`internal/domain/`)

A camada de dominio contem **entidades** e **contratos** (interfaces). Nao possui dependencias externas, apenas a stdlib do Go. **Isolada dentro de cada app.**

### entity/

Structs representando conceitos de dominio. Cada entidade possui um construtor `New<Entity>()`.

**Padrao:**

```go
// apps/user/internal/domain/entity/user.go
package entity

import "time"

type User struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}

func NewUser(name string, email string) *User {
    now := time.Now().UTC()
    return &User{
        Name:      name,
        Email:     email,
        CreatedAt: now,
        UpdatedAt: now,
    }
}
```

**Regras:**
- Arquivo: `<name>.go` em kebab-case se composto (ex: `audit-log.go`).
- Struct exportada com tags JSON em camelCase.
- Construtor `New<Entity>(...)` retorna ponteiro `*Entity`.
- Timestamps sempre em UTC: `time.Now().UTC()`.
- Sem logica de infraestrutura (sem imports de drivers, frameworks ou bibliotecas).

### usecase/

Interfaces definindo os use cases de dominio. Cada use case e representado por **uma unica interface com um unico metodo `Perform`**. Interfaces sao organizadas em **subpastas por contexto de dominio**.

**Principio:** 1 acao = 1 interface = 1 arquivo. Isso garante interfaces coesas, facilita testes (mocks granulares) e respeita o Interface Segregation Principle.

**Estrutura de diretorios:**

```
internal/domain/usecase/
└── user/
    ├── create.go             # CreateUseCase interface
    ├── get-by-id.go          # GetByIDUseCase interface
    └── list.go               # ListUseCase interface
```

**Padrao:**

```go
// apps/user/internal/domain/usecase/user/create.go
package user

import (
    "context"

    "github.com/org/project/apps/user/internal/domain/entity"
)

type CreateInput struct {
    Name  string
    Email string
}

type CreateOutput struct {
    User *entity.User
}

type CreateUseCase interface {
    Perform(ctx context.Context, input CreateInput) (*CreateOutput, error)
}
```

```go
// apps/user/internal/domain/usecase/user/get-by-id.go
package user

import (
    "context"

    "github.com/org/project/apps/user/internal/domain/entity"
)

type GetByIDInput struct {
    ID string
}

type GetByIDUseCase interface {
    Perform(ctx context.Context, input GetByIDInput) (*entity.User, error)
}
```

```go
// apps/user/internal/domain/usecase/user/list.go
package user

import (
    "context"

    "github.com/org/project/apps/user/internal/domain/entity"
)

type ListInput struct {
    Cursor string
    Limit  int
}

type ListOutput struct {
    Users      []entity.User
    NextCursor string
}

type ListUseCase interface {
    Perform(ctx context.Context, input ListInput) (*ListOutput, error)
}
```

**Regras:**
- Organizacao: `domain/usecase/<contexto>/<acao>.go` (ex: `domain/usecase/user/create.go`).
- Cada arquivo contem **1 interface** com **1 metodo**: `Perform`.
- Nome do pacote: o contexto de dominio em **lowercase** (ex: `package user`).
- Interface exportada: `<Action>UseCase` (ex: `CreateUseCase`, `GetByIDUseCase`).
- O metodo `Perform` recebe `context.Context` como primeiro parametro.
- Structs de input (`<Action>Input`) e output (`<Action>Output`) sao definidas no mesmo arquivo da interface.
- Retorna entidades de dominio, output structs e/ou `error`.

### repository/

Interfaces definindo contratos de persistencia.

**Padrao:**

```go
// apps/user/internal/domain/repository/user-repository.go
package repository

import (
    "context"

    "github.com/org/project/apps/user/internal/domain/entity"
)

type UserRepository interface {
    Create(ctx context.Context, user *entity.User) error
    FindByID(ctx context.Context, id string) (*entity.User, error)
    FindByEmail(ctx context.Context, email string) (*entity.User, error)
    List(ctx context.Context, cursor string, limit int) ([]entity.User, string, error)
    Update(ctx context.Context, user *entity.User) error
    Delete(ctx context.Context, id string) error
}
```

**Regras:**
- Arquivo: `<name>-repository.go` (ex: `user-repository.go`).
- Interface exportada: `<Name>Repository`.
- Metodos recebem `context.Context` como primeiro parametro.
- `FindByID` retorna `(*Entity, error)` — retorna `nil, nil` quando nao encontrado.

### event/

Interfaces definindo contratos de mensageria/eventos.

**Padrao:**

```go
// apps/user/internal/domain/event/user-event.go
package event

import (
    "context"

    "github.com/org/project/apps/user/internal/domain/entity"
)

type UserEvent interface {
    PublishCreated(ctx context.Context, user *entity.User) error
    PublishUpdated(ctx context.Context, user *entity.User) error
}
```

**Regras:**
- Arquivo: `<name>-event.go` (ex: `user-event.go`).
- Interface exportada: `<Name>Event`.
- Metodos nomeados por acao: `Publish<Action>`.

### Interfaces para Bibliotecas Externas (`domain/<lib>/`)

Toda biblioteca externa utilizada no projeto **deve ser acessada via inversao de dependencia**. A interface e definida na camada de dominio em uma subpasta nomeada pelo conceito/capacidade que a lib fornece. A implementacao concreta fica em `infrastructure/adapter/`.

**Principio:** O dominio nunca conhece a lib concreta. Ele define **o que precisa** (interface), e a infraestrutura fornece **como fazer** (implementacao com a lib).

**Estrutura de diretorios:**

```
internal/domain/
├── crypt/
│   └── hasher.go              # Interface para hashing de senhas
├── token/
│   └── manager.go             # Interface para geracao/validacao de tokens
└── mailer/
    └── sender.go              # Interface para envio de emails
```

**Padrao:**

```go
// apps/user/internal/domain/crypt/hasher.go
package crypt

type Hasher interface {
    Hash(plain string) (string, error)
    Compare(hashed string, plain string) error
}
```

**Regras:**
- Subpasta nomeada pelo **conceito/capacidade**, nao pelo nome da lib (ex: `crypt/`, nao `bcrypt/`; `token/`, nao `jwt/`).
- Interface exportada com nome descritivo da capacidade (ex: `Hasher`, `Manager`, `Sender`).
- **Nenhum import de lib externa** -- apenas stdlib.
- As implementacoes concretas ficam em `infrastructure/adapter/`.

---

## Camada de Aplicacao (`internal/application/`)

Contem a **implementacao** dos use cases. Orquestra entidades, repositorios e eventos. **Isolada dentro de cada app.**

### usecase/

Implementacoes dos use cases, organizadas em **subpastas por contexto de dominio**, espelhando a estrutura de `domain/usecase/`.

**Estrutura de diretorios:**

```
internal/application/usecase/
└── user/
    ├── create-usecase.go           # CreateUseCase struct + Perform
    ├── create-usecase_test.go      # Teste unitario com gomock
    ├── get-by-id-usecase.go
    └── list-usecase.go
```

**Padrao:**

```go
// apps/user/internal/application/usecase/user/create-usecase.go
package user

import (
    "context"
    "errors"

    "github.com/org/project/apps/user/internal/domain/entity"
    "github.com/org/project/apps/user/internal/domain/event"
    "github.com/org/project/apps/user/internal/domain/repository"
    userusecase "github.com/org/project/apps/user/internal/domain/usecase/user"
)

type CreateUseCase struct {
    userRepository repository.UserRepository
    userEvent      event.UserEvent
}

func NewCreateUseCase(
    userRepository repository.UserRepository,
    userEvent event.UserEvent,
) userusecase.CreateUseCase {
    return &CreateUseCase{
        userRepository: userRepository,
        userEvent:      userEvent,
    }
}

func (u *CreateUseCase) Perform(ctx context.Context, input userusecase.CreateInput) (*userusecase.CreateOutput, error) {
    existing, err := u.userRepository.FindByEmail(ctx, input.Email)
    if err != nil {
        return nil, fmt.Errorf("find by email: %w", err)
    }
    if existing != nil {
        return nil, entity.ErrUserAlreadyExists
    }

    user := entity.NewUser(input.Name, input.Email)

    if err := u.userRepository.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("create user: %w", err)
    }

    // publica evento DEPOIS do commit no banco
    _ = u.userEvent.PublishCreated(ctx, user)

    return &userusecase.CreateOutput{User: user}, nil
}
```

**Regras:**
- Organizacao: `application/usecase/<contexto>/<acao>-usecase.go`.
- Nome do pacote: o contexto de dominio em **lowercase** (ex: `package user`).
- Struct: `<Action>UseCase` (mesmo nome da interface, sem sufixo `Impl`).
- Construtor: `New<Action>UseCase(deps...) <context>usecase.<Action>UseCase` -- retorna a **interface de dominio**.
- Campos da struct sao **interfaces de dominio** (nunca tipos concretos).
- O metodo implementado e sempre `Perform`.
- Depende apenas de `internal/domain/`. Nunca importa `infrastructure/` ou `pkg/`.
- Erros wrapeados com contexto: `fmt.Errorf("action: %w", err)`.
- Evento publicado **apos** persistencia no banco, nunca antes.

---

## Camada de Infraestrutura (`internal/infrastructure/`)

Implementa contratos de dominio usando tecnologias concretas e gerencia a configuracao da aplicacao. Cada binario conecta apenas o que precisa.

### controller/ (apenas APIs)

Controllers HTTP usando Fiber v3. O projeto utiliza um **BaseController** em `pkg/controller/` que implementa o template method pattern: extrai dados comuns da request (requestId, IP, headers), executa validacao e delega para o metodo concreto. Cada controller embute o `BaseController` e so implementa a logica especifica. **Usado apenas por binarios de API.**

#### BaseController — `pkg/controller/` (compartilhado entre todos os apps)

O `BaseController` vive no `pkg/` porque e infraestrutura generica **sem nenhuma logica de dominio**. Todos os apps reutilizam a mesma base, evitando duplicacao.

Ele centraliza:
- Extracao automatica de `requestId`, `parentId`, `IP`, `userAgent`.
- Bind + validacao do body com resposta padronizada.
- Tratamento uniforme de erros (dominio -> HTTP status).
- Logging estruturado com contexto da request.
- Suporte a transactions quando o handler precisa.

**Estrutura do `pkg/controller/`:**

```
pkg/
└── controller/
    ├── base.go                 # BaseController struct + Handle (template method)
    ├── context.go              # RequestContext struct
    ├── bind.go                 # BindAndHandle[T] generic
    └── errors.go               # DomainError, mapDomainError, ErrorResponse, RegisterError
```

**Estrutura de cada app (so o codigo especifico):**

```
apps/user/internal/infrastructure/controller/
├── errors.go                   # Registro dos erros de dominio DESTE app
├── user-controller.go          # UserController (embute pkg BaseController)
└── order-controller.go         # OrderController (embute pkg BaseController)
```

**`pkg/controller/context.go` — RequestContext:**

```go
// pkg/controller/context.go
package controller

import "log/slog"

// RequestContext contem dados extraidos automaticamente de toda request.
// Handlers recebem este struct ja preenchido, sem precisar extrair manualmente.
type RequestContext struct {
    RequestID string
    ParentID  string
    IP        string
    UserAgent string
    Logger    *slog.Logger
}
```

**`pkg/controller/base.go` — BaseController (template method):**

```go
// pkg/controller/base.go
package controller

import (
    "fmt"
    "log/slog"

    "github.com/gofiber/fiber/v3"
)

// HandlerFunc e a assinatura que handlers concretos implementam.
// Recebem o fiber.Ctx e o RequestContext ja populado.
type HandlerFunc func(c fiber.Ctx, rc RequestContext) error

// BaseController centraliza extracao de contexto, logging e error handling.
// Todo controller do projeto embute esta struct.
type BaseController struct {
    errorMapper *ErrorMapper
}

// NewBaseController cria um BaseController com o mapeamento de erros configurado.
func NewBaseController(mapper *ErrorMapper) BaseController {
    return BaseController{errorMapper: mapper}
}

// Handle e o template method. Extrai o contexto comum e delega para o handler concreto.
// Todo handler do projeto deve ser registrado via este metodo.
func (b *BaseController) Handle(handler HandlerFunc) fiber.Handler {
    return func(c fiber.Ctx) error {
        rc := b.extractContext(c)

        if err := handler(c, rc); err != nil {
            return b.handleError(c, rc, err)
        }

        return nil
    }
}

// extractContext extrai dados comuns da request uma unica vez.
func (b *BaseController) extractContext(c fiber.Ctx) RequestContext {
    requestID := ""
    if rid := c.Locals("requestid"); rid != nil {
        requestID = fmt.Sprint(rid)
    }

    parentID := c.Get("X-Request-Id")

    logger := slog.With(
        slog.String("requestId", requestID),
        slog.String("parentId", parentID),
        slog.String("method", c.Method()),
        slog.String("route", c.Route().Path),
    )

    return RequestContext{
        RequestID: requestID,
        ParentID:  parentID,
        IP:        c.IP(),
        UserAgent: c.Get("User-Agent"),
        Logger:    logger,
    }
}

// handleError mapeia erros de dominio para respostas HTTP padronizadas.
func (b *BaseController) handleError(c fiber.Ctx, rc RequestContext, err error) error {
    status, message := b.errorMapper.Map(err)

    if status >= 500 {
        rc.Logger.Error("request failed",
            slog.Int("statusCode", status),
            slog.String("error", err.Error()),
        )
    } else {
        rc.Logger.Warn("request rejected",
            slog.Int("statusCode", status),
            slog.String("error", err.Error()),
        )
    }

    return c.Status(status).JSON(ErrorResponse{
        Error:  message,
        Status: status,
    })
}
```

**`pkg/controller/validator.go` — StructValidator para Fiber v3 com go-playground/validator:**

O projeto utiliza **`github.com/go-playground/validator/v10`** como engine de validacao. O Fiber v3 suporta `StructValidator` nativo — ao configurar, toda chamada a `c.Bind().Body()` executa a validacao automaticamente apos o parsing.

```go
// pkg/controller/validator.go
package controller

import (
    "fmt"
    "strings"

    "github.com/go-playground/validator/v10"
)

// FiberValidator implementa a interface fiber.StructValidator.
// Configurado no fiber.New() para que c.Bind().Body() valide automaticamente.
type FiberValidator struct {
    validate *validator.Validate
}

// NewFiberValidator cria um validador com as configuracoes padrao do projeto.
func NewFiberValidator() *FiberValidator {
    v := validator.New(validator.WithRequiredStructEnabled())

    // usar nome do campo JSON nas mensagens de erro (em vez do nome da struct)
    v.RegisterTagNameFunc(func(fld reflect.StructField) string {
        name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
        if name == "-" {
            return fld.Name
        }
        return name
    })

    return &FiberValidator{validate: v}
}

// Validate implementa fiber.StructValidator.
// Chamado automaticamente pelo Fiber apos o bind do body.
func (v *FiberValidator) Validate(out any) error {
    if err := v.validate.Struct(out); err != nil {
        return formatValidationErrors(err)
    }
    return nil
}

// formatValidationErrors converte os erros do validator em uma mensagem legivel.
func formatValidationErrors(err error) error {
    var errs validator.ValidationErrors
    if !errors.As(err, &errs) {
        return err
    }

    var msgs []string
    for _, e := range errs {
        msgs = append(msgs, formatFieldError(e))
    }
    return fmt.Errorf("validation failed: %s", strings.Join(msgs, "; "))
}

func formatFieldError(e validator.FieldError) string {
    field := e.Field()
    switch e.Tag() {
    case "required":
        return fmt.Sprintf("%s is required", field)
    case "email":
        return fmt.Sprintf("%s must be a valid email", field)
    case "min":
        return fmt.Sprintf("%s must be at least %s", field, e.Param())
    case "max":
        return fmt.Sprintf("%s must be at most %s", field, e.Param())
    case "gt":
        return fmt.Sprintf("%s must be greater than %s", field, e.Param())
    case "gte":
        return fmt.Sprintf("%s must be greater than or equal to %s", field, e.Param())
    case "oneof":
        return fmt.Sprintf("%s must be one of [%s]", field, e.Param())
    default:
        return fmt.Sprintf("%s failed on %s validation", field, e.Tag())
    }
}
```

**Registro no Fiber App (`pkg/httpserver/fiber.go`):**

```go
// pkg/httpserver/fiber.go
package httpserver

import (
    "time"

    "github.com/gofiber/fiber/v3"

    pkgctrl "github.com/org/project/pkg/controller"
)

func NewFiberApp() *fiber.App {
    return fiber.New(fiber.Config{
        // validacao automatica em todo c.Bind().Body()
        StructValidator: pkgctrl.NewFiberValidator(),

        CaseSensitive: true,
        StrictRouting:  true,
        BodyLimit:      4 * 1024 * 1024, // 4MB
        ReadTimeout:    10 * time.Second,
        WriteTimeout:   10 * time.Second,
        IdleTimeout:    120 * time.Second,
    })
}
```

Com isso, todo `c.Bind().Body(&input)` no `BindAndHandle` executa automaticamente:
1. Parse do JSON
2. Validacao via tags `validate` do `go-playground/validator`
3. Retorno de erro formatado se a validacao falhar

**Estrutura do `pkg/controller/` atualizada:**

```
pkg/
└── controller/
    ├── base.go                 # BaseController struct + Handle (template method)
    ├── context.go              # RequestContext struct
    ├── bind.go                 # BindAndHandle[T] generic
    ├── validator.go            # FiberValidator (go-playground/validator/v10)
    └── errors.go               # ErrorMapper, ErrorResponse
```

**`pkg/controller/bind.go` — BindAndHandle generic:**

```go
// pkg/controller/bind.go
package controller

import "github.com/gofiber/fiber/v3"

// BindHandlerFunc e a assinatura para handlers que recebem body tipado.
type BindHandlerFunc[T any] func(c fiber.Ctx, rc RequestContext, input T) error

// BindAndHandle faz bind + validacao do body automaticamente antes de chamar o handler.
// O handler so e executado se o body for valido.
// A validacao e executada pelo FiberValidator configurado no fiber.Config.StructValidator.
func BindAndHandle[T any](b *BaseController, handler BindHandlerFunc[T]) fiber.Handler {
    return b.Handle(func(c fiber.Ctx, rc RequestContext) error {
        var input T
        if err := c.Bind().Body(&input); err != nil {
            // erro de parse OU de validacao — ambos retornam 400
            return fiber.NewError(fiber.StatusBadRequest, err.Error())
        }
        return handler(c, rc, input)
    })
}
```

### Tags de Validacao — Referencia Rapida

Tags mais usadas do `go-playground/validator/v10` nas structs de input:

| Tag | Descricao | Exemplo |
|---|---|---|
| `required` | Campo obrigatorio | `validate:"required"` |
| `email` | Email valido | `validate:"required,email"` |
| `min` | Tamanho/valor minimo | `validate:"min=3"` (string: 3 chars, int: >= 3) |
| `max` | Tamanho/valor maximo | `validate:"max=100"` |
| `gt` | Maior que (numeros) | `validate:"gt=0"` |
| `gte` | Maior ou igual | `validate:"gte=1"` |
| `oneof` | Um dos valores listados | `validate:"oneof=pending active inactive"` |
| `uuid` | UUID valido | `validate:"required,uuid"` |
| `url` | URL valida | `validate:"url"` |
| `dive` | Valida cada elemento do slice | `validate:"required,min=1,dive"` |
| `omitempty` | So valida se preenchido | `validate:"omitempty,email"` |

**Padrao para structs de input:**

```go
type CreateOrderInput struct {
    UserID     string            `json:"userId" validate:"required,uuid"`
    TotalCents int64             `json:"totalCents" validate:"required,gt=0"`
    Currency   string            `json:"currency" validate:"required,oneof=BRL USD EUR"`
    Notes      string            `json:"notes" validate:"omitempty,max=500"`
    Items      []CreateItemInput `json:"items" validate:"required,min=1,dive"`
}

type CreateItemInput struct {
    ProductID      string `json:"productId" validate:"required,uuid"`
    Quantity       int    `json:"quantity" validate:"required,gt=0"`
    UnitPriceCents int64  `json:"unitPriceCents" validate:"required,gt=0"`
}
```

**Regras de validacao:**
- **Sempre** usar tags `validate` em structs de input. Nunca validar manualmente no handler.
- **`required`** em todo campo obrigatorio.
- **`uuid`** em campos que sao IDs.
- **`oneof`** para enumeracoes (em vez de validar no use case).
- **`dive`** em slices para validar cada elemento individualmente.
- **`omitempty`** em campos opcionais que so devem ser validados se preenchidos.
- Mensagens de erro geradas automaticamente pelo `formatFieldError` — nunca expor mensagens do validator cru para o usuario.

**`pkg/controller/errors.go` — Mapeamento extensivel de erros:**

```go
// pkg/controller/errors.go
package controller

import (
    "errors"

    "github.com/gofiber/fiber/v3"
)

// ErrorResponse e a struct padrao de resposta de erro da API.
type ErrorResponse struct {
    Error  string `json:"error"`
    Status int    `json:"status"`
}

// ErrorMapping associa um erro de dominio a um status HTTP.
type ErrorMapping struct {
    Target     error
    StatusCode int
}

// ErrorMapper contem o mapeamento de erros de dominio -> HTTP status.
// Cada app registra seus proprios erros no mapper.
type ErrorMapper struct {
    mappings []ErrorMapping
}

// NewErrorMapper cria um mapper vazio.
func NewErrorMapper() *ErrorMapper {
    return &ErrorMapper{}
}

// Register adiciona um mapeamento erro -> status.
func (m *ErrorMapper) Register(target error, statusCode int) *ErrorMapper {
    m.mappings = append(m.mappings, ErrorMapping{Target: target, StatusCode: statusCode})
    return m
}

// Map retorna o (statusCode, mensagem) para um erro.
// Erros nao mapeados viram 500 Internal Server Error.
func (m *ErrorMapper) Map(err error) (int, string) {
    for _, mapping := range m.mappings {
        if errors.Is(err, mapping.Target) {
            return mapping.StatusCode, err.Error()
        }
    }
    return fiber.StatusInternalServerError, "internal server error"
}
```

**Erros de dominio base (cada app define os seus):**

```go
// apps/user/internal/domain/entity/errors.go
package entity

import "errors"

var (
    ErrInvalidInput      = errors.New("invalid input")
    ErrUserNotFound      = errors.New("user not found")
    ErrUserAlreadyExists = errors.New("email already in use")
    ErrInvalidRole       = errors.New("invalid role")
    ErrForbidden         = errors.New("forbidden")
)

// Erros especificos wrapeiam os erros base para manter rastreabilidade:
//   fmt.Errorf("user %s: %w", id, ErrUserNotFound)
//   errors.Is(err, ErrUserNotFound) == true
```

**Registro dos erros no app (wiring no `main.go`):**

```go
// apps/user/cmd/api/main.go

import (
    "github.com/gofiber/fiber/v3"

    pkgctrl "github.com/org/project/pkg/controller"
    "github.com/org/project/apps/user/internal/domain/entity"
)

// ProvideErrorMapper configura o mapeamento de erros DESTE app.
func ProvideErrorMapper() *pkgctrl.ErrorMapper {
    return pkgctrl.NewErrorMapper().
        Register(entity.ErrInvalidInput, fiber.StatusBadRequest).
        Register(entity.ErrUserNotFound, fiber.StatusNotFound).
        Register(entity.ErrUserAlreadyExists, fiber.StatusConflict).
        Register(entity.ErrInvalidRole, fiber.StatusUnprocessableEntity).
        Register(entity.ErrForbidden, fiber.StatusForbidden)
}

// ProvideBaseController cria o BaseController com o mapper do app.
func ProvideBaseController(mapper *pkgctrl.ErrorMapper) pkgctrl.BaseController {
    return pkgctrl.NewBaseController(mapper)
}

func main() {
    fx.New(
        // ...
        fx.Provide(
            ProvideErrorMapper,
            ProvideBaseController,
            // ...
        ),
        // ...
    ).Run()
}
```

#### Controller Concreto — Exemplo Simples (sem transaction)

```go
// apps/user/internal/infrastructure/controller/user-controller.go
package controller

import (
    "log/slog"

    "github.com/gofiber/fiber/v3"

    pkgctrl "github.com/org/project/pkg/controller"
    "github.com/org/project/apps/user/internal/domain/entity"
    userusecase "github.com/org/project/apps/user/internal/domain/usecase/user"
)

type UserController struct {
    pkgctrl.BaseController                            // embute o BaseController do pkg
    createUseCase  userusecase.CreateUseCase
    getByIDUseCase userusecase.GetByIDUseCase
    listUseCase    userusecase.ListUseCase
}

func NewUserController(
    base pkgctrl.BaseController,                      // injetado via FX
    createUseCase userusecase.CreateUseCase,
    getByIDUseCase userusecase.GetByIDUseCase,
    listUseCase userusecase.ListUseCase,
) *UserController {
    return &UserController{
        BaseController: base,
        createUseCase:  createUseCase,
        getByIDUseCase: getByIDUseCase,
        listUseCase:    listUseCase,
    }
}

func (ctrl *UserController) RegisterRoutes(app *fiber.App) {
    group := app.Group("/v1/users")
    group.Post("/", pkgctrl.BindAndHandle(&ctrl.BaseController, ctrl.Create))
    group.Get("/:id", ctrl.Handle(ctrl.GetByID))
    group.Get("/", ctrl.Handle(ctrl.List))
}

// Create godoc
// @Summary      Cria um usuario
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        input body CreateUserInput true "Dados do usuario"
// @Success      201  {object}  entity.User
// @Failure      400  {object}  pkgctrl.ErrorResponse
// @Failure      409  {object}  pkgctrl.ErrorResponse
// @Router       /v1/users [post]
func (ctrl *UserController) Create(c fiber.Ctx, rc pkgctrl.RequestContext, input CreateUserInput) error {
    // body ja foi parseado e validado pelo BindAndHandle
    // requestId, parentId, IP, logger ja estao no rc — zero boilerplate
    output, err := ctrl.createUseCase.Perform(c.Context(), userusecase.CreateInput{
        Name:  input.Name,
        Email: input.Email,
    })
    if err != nil {
        return err // BaseController.handleError mapeia para HTTP status via ErrorMapper
    }

    rc.Logger.Info("user created", slog.String("userId", output.User.ID))
    return c.Status(fiber.StatusCreated).JSON(output.User)
}

func (ctrl *UserController) GetByID(c fiber.Ctx, rc pkgctrl.RequestContext) error {
    id := c.Params("id")

    user, err := ctrl.getByIDUseCase.Perform(c.Context(), userusecase.GetByIDInput{ID: id})
    if err != nil {
        return err
    }
    if user == nil {
        return entity.ErrUserNotFound
    }

    return c.Status(fiber.StatusOK).JSON(user)
}

func (ctrl *UserController) List(c fiber.Ctx, rc pkgctrl.RequestContext) error {
    var filter ListUsersFilter
    if err := c.Bind().Query(&filter); err != nil {
        return fiber.NewError(fiber.StatusBadRequest, err.Error())
    }

    output, err := ctrl.listUseCase.Perform(c.Context(), userusecase.ListInput{
        Cursor: filter.Cursor,
        Limit:  filter.Limit,
    })
    if err != nil {
        return err
    }

    return c.Status(fiber.StatusOK).JSON(output)
}

type CreateUserInput struct {
    Name  string `json:"name" validate:"required,min=2,max=100"`
    Email string `json:"email" validate:"required,email"`
}

type ListUsersFilter struct {
    Cursor string `query:"cursor"`
    Limit  int    `query:"limit,default:20"`
}
```

#### Controller com Transaction — Exemplo Completo

Quando um endpoint precisa executar **multiplas operacoes atomicas** (ex: criar order + items + atualizar estoque), o controller orquestra a transaction via `Transactor`:

```go
// apps/billing/internal/infrastructure/controller/order-controller.go
package controller

import (
    "database/sql"
    "log/slog"

    "github.com/gofiber/fiber/v3"

    pkgctrl "github.com/org/project/pkg/controller"
    "github.com/org/project/apps/billing/internal/domain/entity"
    "github.com/org/project/apps/billing/internal/domain/event"
    "github.com/org/project/apps/billing/internal/domain/repository"
)

type OrderController struct {
    pkgctrl.BaseController                             // embute o BaseController do pkg
    transactor  repository.Transactor
    orderRepo   repository.OrderRepository
    stockRepo   repository.StockRepository
    orderEvent  event.OrderEvent
}

func NewOrderController(
    base pkgctrl.BaseController,                       // injetado via FX
    transactor repository.Transactor,
    orderRepo repository.OrderRepository,
    stockRepo repository.StockRepository,
    orderEvent event.OrderEvent,
) *OrderController {
    return &OrderController{
        BaseController: base,
        transactor:     transactor,
        orderRepo:      orderRepo,
        stockRepo:      stockRepo,
        orderEvent:     orderEvent,
    }
}

func (ctrl *OrderController) RegisterRoutes(app *fiber.App) {
    group := app.Group("/v1/orders")
    group.Post("/", pkgctrl.BindAndHandle(&ctrl.BaseController, ctrl.Create))
    group.Post("/:id/cancel", ctrl.Handle(ctrl.Cancel))
}

// Create — operacao que PRECISA de transaction
// Cria order + items + debita estoque atomicamente.
func (ctrl *OrderController) Create(c fiber.Ctx, rc pkgctrl.RequestContext, input CreateOrderInput) error {
    ctx := c.Context()

    var order *entity.Order

    // === TRANSACTION: tudo dentro e atomico ===
    err := ctrl.transactor.RunInTx(ctx, func(tx *sql.Tx) error {
        // 1. criar order
        var err error
        order, err = ctrl.orderRepo.CreateWithTx(ctx, tx, entity.NewOrder(
            input.UserID,
            input.TotalCents,
        ))
        if err != nil {
            return err
        }

        // 2. criar items (mesmo tx)
        for _, item := range input.Items {
            _, err := ctrl.orderRepo.CreateItemWithTx(ctx, tx, entity.NewOrderItem(
                order.ID,
                item.ProductID,
                item.Quantity,
                item.UnitPriceCents,
            ))
            if err != nil {
                return err
            }
        }

        // 3. debitar estoque (mesmo tx)
        for _, item := range input.Items {
            if err := ctrl.stockRepo.DebitWithTx(ctx, tx, item.ProductID, item.Quantity); err != nil {
                return err
            }
        }

        return nil // COMMIT — order, items e estoque persistidos atomicamente
    })

    if err != nil {
        return err // BaseController mapeia para HTTP status via ErrorMapper
    }

    // === DEPOIS DO COMMIT: publicar evento ===
    // Nunca publicar evento dentro da transaction
    _ = ctrl.orderEvent.PublishCreated(ctx, order)

    rc.Logger.Info("order created", slog.String("orderId", order.ID))
    return c.Status(fiber.StatusCreated).JSON(order)
}

// Cancel — operacao SIMPLES, sem transaction
// Apenas atualiza status. Uma unica operacao nao precisa de tx.
func (ctrl *OrderController) Cancel(c fiber.Ctx, rc pkgctrl.RequestContext) error {
    id := c.Params("id")
    ctx := c.Context()

    order, err := ctrl.orderRepo.FindByID(ctx, id)
    if err != nil {
        return err
    }
    if order == nil {
        return entity.ErrOrderNotFound
    }

    if err := order.Cancel(); err != nil {
        return err // regra de negocio (ex: order ja cancelada)
    }

    if err := ctrl.orderRepo.Update(ctx, order); err != nil {
        return err
    }

    _ = ctrl.orderEvent.PublishCancelled(ctx, order)

    rc.Logger.Info("order cancelled", slog.String("orderId", order.ID))
    return c.Status(fiber.StatusOK).JSON(order)
}

type CreateOrderInput struct {
    UserID     string             `json:"userId" validate:"required"`
    TotalCents int64              `json:"totalCents" validate:"required,gt=0"`
    Items      []CreateItemInput  `json:"items" validate:"required,min=1,dive"`
}

type CreateItemInput struct {
    ProductID      string `json:"productId" validate:"required"`
    Quantity       int    `json:"quantity" validate:"required,gt=0"`
    UnitPriceCents int64  `json:"unitPriceCents" validate:"required,gt=0"`
}
```

#### Transactor — Interface de Dominio

```go
// apps/billing/internal/domain/repository/transactor.go
package repository

import (
    "context"
    "database/sql"
)

type Transactor interface {
    RunInTx(ctx context.Context, fn func(tx *sql.Tx) error) error
}
```

```go
// apps/billing/internal/infrastructure/repository/transactor.go
package repository

import (
    "context"
    "database/sql"

    "github.com/org/project/pkg/postgres"
    domainrepo "github.com/org/project/apps/billing/internal/domain/repository"
)

type PostgresTransactor struct {
    db *sql.DB
}

func NewPostgresTransactor(db *sql.DB) domainrepo.Transactor {
    return &PostgresTransactor{db: db}
}

func (t *PostgresTransactor) RunInTx(ctx context.Context, fn func(tx *sql.Tx) error) error {
    return postgres.RunInTx(ctx, t.db, fn)
}
```

#### Repositorio com Metodos `WithTx`

Quando um repositorio participa de uma transaction externa, expoe metodos que recebem `*sql.Tx`:

```go
// apps/billing/internal/domain/repository/order-repository.go
package repository

import (
    "context"
    "database/sql"

    "github.com/org/project/apps/billing/internal/domain/entity"
)

type OrderRepository interface {
    // Metodos normais (sem transaction, autocommit)
    FindByID(ctx context.Context, id string) (*entity.Order, error)
    Update(ctx context.Context, order *entity.Order) error

    // Metodos WithTx (participam de transaction externa)
    CreateWithTx(ctx context.Context, tx *sql.Tx, order *entity.Order) (*entity.Order, error)
    CreateItemWithTx(ctx context.Context, tx *sql.Tx, item *entity.OrderItem) (*entity.OrderItem, error)
}
```

```go
// apps/billing/internal/infrastructure/repository/order-repository.go
package repository

func (r *OrderRepository) CreateWithTx(ctx context.Context, tx *sql.Tx, order *entity.Order) (*entity.Order, error) {
    const query = `
        INSERT INTO "orders" ("userId", "status", "totalCents", "createdAt", "updatedAt")
        VALUES ($1, $2, $3, $4, $5)
        RETURNING "id"`

    err := tx.QueryRowContext(ctx, query,
        order.UserID, order.Status, order.TotalCents, order.CreatedAt, order.UpdatedAt,
    ).Scan(&order.ID)
    if err != nil {
        return nil, fmt.Errorf("insert order: %w", err)
    }
    return order, nil
}

func (r *OrderRepository) FindByID(ctx context.Context, id string) (*entity.Order, error) {
    const query = `
        SELECT "id", "userId", "status", "totalCents", "createdAt", "updatedAt"
        FROM "orders"
        WHERE "id" = $1`

    var o entity.Order
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &o.ID, &o.UserID, &o.Status, &o.TotalCents, &o.CreatedAt, &o.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("find order by id: %w", err)
    }
    return &o, nil
}
```

#### Quando Usar Transaction no Controller

| Cenario | Transaction? | Pattern |
|---|---|---|
| GET (leitura simples) | **Nao** | `ctrl.Handle(ctrl.GetByID)` |
| POST com 1 INSERT | **Nao** | `BindAndHandle(&ctrl.BaseController, ctrl.Create)` — use case faz INSERT unico |
| POST com N INSERTs relacionados | **Sim** | `transactor.RunInTx` no handler |
| PUT/PATCH simples | **Nao** | Use case faz UPDATE unico |
| PUT que move saldo (debito + credito) | **Sim** | `transactor.RunInTx` no handler |
| DELETE com cascade manual | **Sim** | `transactor.RunInTx` no handler |
| Operacao + publicacao de evento | **Evento FORA do tx** | `RunInTx(...)` depois `event.Publish(...)` |

#### Regras do Controller

- `BaseController`, `RequestContext`, `BindAndHandle`, `ErrorMapper` e `ErrorResponse` vivem em **`pkg/controller/`**. Nenhum app reimplementa essa logica.
- Arquivo: `<name>-controller.go` (ex: `user-controller.go`) em `internal/infrastructure/controller/`.
- Struct: `<Name>Controller` que embute `pkgctrl.BaseController`.
- Construtor: `New<Name>Controller(base pkgctrl.BaseController, deps...) *<Name>Controller` -- recebe o `BaseController` via FX.
- Dependencias sao **interfaces de dominio de use case**, **repositorios** (quando precisa de tx) e **Transactor**.
- Deve implementar `RegisterRoutes(app *fiber.App)`.
- Rotas registradas via `ctrl.Handle(ctrl.Method)` ou `pkgctrl.BindAndHandle(&ctrl.BaseController, ctrl.Method)`.
- Handlers recebem `pkgctrl.RequestContext` ja populado — nunca extrair requestId, IP, etc. manualmente.
- Handlers delegam logica de negocio para use cases. Sem logica de negocio no controller.
- Usar `c.Context()` ao passar contexto para camadas internas. Nunca `fiber.Ctx`.
- Handlers HTTP usam anotacoes Swagger (swaggo).
- Structs de input com tags `json` e `validate`.
- Usar `c.Bind().Body()` para parsing, nunca `json.Unmarshal`.
- Evento publicado **DEPOIS** do commit da transaction, nunca dentro.
- Erros de dominio retornados como `return err` — o `BaseController.handleError` mapeia para HTTP status via `ErrorMapper`.
- Cada app registra seus erros de dominio no `ErrorMapper` via `ProvideErrorMapper()` no `main.go`. O `pkg/controller` nao conhece nenhum erro de dominio — ele so recebe o mapeamento.

### subscriber/ (apenas Consumers)

Handlers de consumo NATS JetStream. Cada subscriber consome de um subject especifico e delega o processamento para use cases. **Usado apenas por binarios de Consumer.**

**Padrao:**

```go
// apps/billing/internal/infrastructure/subscriber/user-subscriber.go
package subscriber

import (
    "context"
    "encoding/json"
    "log/slog"

    "github.com/google/uuid"
    "github.com/nats-io/nats.go/jetstream"

    pkgevent "github.com/org/project/pkg/event"
    billingusecase "github.com/org/project/apps/billing/internal/domain/usecase/billing"
)

type UserSubscriber struct {
    createAccount billingusecase.CreateAccountUseCase
}

func NewUserSubscriber(createAccount billingusecase.CreateAccountUseCase) *UserSubscriber {
    return &UserSubscriber{createAccount: createAccount}
}

func (s *UserSubscriber) HandleUserCreated(ctx context.Context, msg jetstream.Msg) error {
    requestID := uuid.New().String()
    parentID := msg.Headers().Get("Nats-Msg-Id")

    logger := slog.With(
        slog.String("requestId", requestID),
        slog.String("parentId", parentID),
        slog.String("subject", msg.Subject()),
    )

    var env pkgevent.Envelope
    if err := json.Unmarshal(msg.Data(), &env); err != nil {
        logger.Error("unmarshal envelope failed", slog.String("error", err.Error()))
        msg.Term() // mensagem corrompida — nao reenviar
        return err
    }

    dataBytes, _ := json.Marshal(env.Data)
    var data pkgevent.UserCreatedData
    if err := json.Unmarshal(dataBytes, &data); err != nil {
        logger.Error("unmarshal data failed", slog.String("error", err.Error()))
        msg.Term()
        return err
    }

    logger.Info("processing user created event")

    _, err := s.createAccount.Perform(ctx, billingusecase.CreateAccountInput{
        UserID: data.UserID,
        Name:   data.Name,
        Email:  data.Email,
    })
    if err != nil {
        logger.Error("create account failed", slog.String("error", err.Error()))
        msg.Nak() // redelivery com backoff
        return err
    }

    logger.Info("account created successfully")
    msg.Ack()
    return nil
}
```

**Regras:**
- Arquivo: `<name>-subscriber.go` (ex: `user-subscriber.go`).
- Struct: `<Name>Subscriber`.
- Construtor: `New<Name>Subscriber(deps...) *<Name>Subscriber` -- retorna **tipo concreto** (ponteiro).
- Dependencias sao **interfaces de dominio de use case**.
- Handlers nomeados: `Handle<EventName>` (ex: `HandleUserCreated`).
- `msg.Ack()` apenas apos processamento com sucesso.
- `msg.Nak()` para erros recuperaveis (JetStream faz redelivery com backoff).
- `msg.Term()` para mensagens corrompidas/invalidas (nao reenviar).
- Logging estruturado com `requestId` e `parentId` em todo handler.

### repository/

Implementacoes de repositorio usando PostgreSQL. Queries SQL escritas manualmente, sem ORM.

**Padrao:**

```go
// apps/user/internal/infrastructure/repository/user-repository.go
package repository

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    "github.com/org/project/apps/user/internal/domain/entity"
    "github.com/org/project/apps/user/internal/domain/repository"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) repository.UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(ctx context.Context, user *entity.User) error {
    const query = `
        INSERT INTO users (name, email, created_at, updated_at)
        VALUES ($1, $2, $3, $4)
        RETURNING id`

    return r.db.QueryRowContext(ctx, query,
        user.Name, user.Email, user.CreatedAt, user.UpdatedAt,
    ).Scan(&user.ID)
}

func (r *UserRepository) FindByID(ctx context.Context, id string) (*entity.User, error) {
    const query = `
        SELECT id, name, email, created_at, updated_at
        FROM users
        WHERE id = $1`

    var u entity.User
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &u.ID, &u.Name, &u.Email, &u.CreatedAt, &u.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("find user by id: %w", err)
    }
    return &u, nil
}

func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*entity.User, error) {
    const query = `
        SELECT id, name, email, created_at, updated_at
        FROM users
        WHERE email = $1`

    var u entity.User
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &u.ID, &u.Name, &u.Email, &u.CreatedAt, &u.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("find user by email: %w", err)
    }
    return &u, nil
}

// List usa paginacao por cursor (sem OFFSET)
func (r *UserRepository) List(ctx context.Context, cursor string, limit int) ([]entity.User, string, error) {
    var rows *sql.Rows
    var err error

    if cursor == "" {
        const query = `
            SELECT id, name, email, created_at, updated_at
            FROM users
            ORDER BY id ASC
            LIMIT $1`
        rows, err = r.db.QueryContext(ctx, query, limit+1)
    } else {
        const query = `
            SELECT id, name, email, created_at, updated_at
            FROM users
            WHERE id > $1
            ORDER BY id ASC
            LIMIT $2`
        rows, err = r.db.QueryContext(ctx, query, cursor, limit+1)
    }
    if err != nil {
        return nil, "", fmt.Errorf("list users: %w", err)
    }
    defer rows.Close()

    var users []entity.User
    for rows.Next() {
        var u entity.User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt, &u.UpdatedAt); err != nil {
            return nil, "", fmt.Errorf("scan user: %w", err)
        }
        users = append(users, u)
    }
    if err := rows.Err(); err != nil {
        return nil, "", fmt.Errorf("rows iteration: %w", err)
    }

    var nextCursor string
    if len(users) > limit {
        nextCursor = users[limit].ID
        users = users[:limit]
    }

    return users, nextCursor, nil
}
```

**Regras:**
- Arquivo: `<name>-repository.go` (ex: `user-repository.go`).
- Struct: `<Name>Repository` (mesmo nome da interface).
- Construtor: `New<Name>Repository(deps...) repository.<Name>Repository` -- retorna a **interface de dominio**.
- Recebe `*sql.DB` como dependencia (injetado via FX).
- SQL escrito manualmente com placeholders posicionais (`$1`, `$2`). Sem ORM.
- Sem `SELECT *`. Listar colunas explicitamente.
- `INSERT ... RETURNING id` para obter ID gerado.
- `sql.ErrNoRows` retorna `nil, nil` (nao e erro, e ausencia).
- Paginacao por cursor, nunca `OFFSET` (para tabelas grandes).
- Queries como `const` dentro do metodo.
- `rows.Close()` sempre com `defer`.
- Sempre verificar `rows.Err()` apos iteracao.
- Erros wrapeados com contexto: `fmt.Errorf("action: %w", err)`.

### publisher/

Implementacoes de eventos usando NATS JetStream.

**Padrao:**

```go
// apps/user/internal/infrastructure/publisher/user-publisher.go
package publisher

import (
    "context"

    "github.com/org/project/apps/user/internal/domain/entity"
    "github.com/org/project/apps/user/internal/domain/event"
    pkgevent "github.com/org/project/pkg/event"
    pkgnats "github.com/org/project/pkg/nats"
)

type UserPublisher struct {
    publisher *pkgnats.Publisher
}

func NewUserPublisher(publisher *pkgnats.Publisher) event.UserEvent {
    return &UserPublisher{publisher: publisher}
}

func (p *UserPublisher) PublishCreated(ctx context.Context, user *entity.User) error {
    return p.publisher.Publish(ctx, pkgevent.SubjectUserCreated, user.ID, pkgevent.UserCreatedData{
        UserID: user.ID,
        Name:   user.Name,
        Email:  user.Email,
    })
}
```

**Regras:**
- Arquivo: `<name>-publisher.go` (ex: `user-publisher.go`).
- Struct: `<Name>Publisher`.
- Construtor: `New<Name>Publisher(deps...) event.<Name>Event` -- retorna a **interface de dominio**.
- `msgID` = ID da entidade para deduplicacao no JetStream.
- Publicar DEPOIS do commit no banco, nunca antes.

### adapter/

Implementacoes concretas das interfaces de bibliotecas externas definidas em `domain/<lib>/`.

**Padrao:**

```go
// apps/user/internal/infrastructure/adapter/bcrypt-adapter.go
package adapter

import (
    "github.com/org/project/apps/user/internal/domain/crypt"
    "golang.org/x/crypto/bcrypt"
)

type BcryptAdapter struct {
    cost int
}

func NewBcryptAdapter() crypt.Hasher {
    return &BcryptAdapter{cost: bcrypt.DefaultCost}
}

func (a *BcryptAdapter) Hash(plain string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(plain), a.cost)
    return string(bytes), err
}

func (a *BcryptAdapter) Compare(hashed string, plain string) error {
    return bcrypt.CompareHashAndPassword([]byte(hashed), []byte(plain))
}
```

**Regras:**
- Arquivo: `<lib-name>-adapter.go` (ex: `bcrypt-adapter.go`, `jwt-adapter.go`).
- Struct: `<LibName>Adapter`.
- Construtor: `New<LibName>Adapter(deps...) <domain-package>.<Interface>` -- retorna a **interface de dominio**.
- O nome do arquivo/struct reflete a **lib concreta**, pois esta na camada de infraestrutura.

### config/

Configuracao da aplicacao e modulo FX compartilhado de infraestrutura.

**`config.go`** -- Struct de configuracao e carregamento de variaveis de ambiente:

```go
// apps/user/internal/infrastructure/config/config.go
package config

import "os"

type AppConfig struct {
    AppPort        string
    DBHost         string
    DBPort         int
    DBUser         string
    DBPassword     string
    DBName         string
    DBSSLMode      string
    DBMaxOpenConns int
    DBMaxIdleConns int
    NatsURL        string
    ServiceName    string
    ServiceVersion string
    LogLevel       string
}

func LoadConfig() *AppConfig {
    return &AppConfig{
        AppPort:        env("APP_PORT", "8080"),
        DBHost:         env("DB_HOST", "localhost"),
        DBPort:         envInt("DB_PORT", 5432),
        DBUser:         env("DB_USER", "app"),
        DBPassword:     env("DB_PASSWORD", ""),
        DBName:         env("DB_NAME", "users"),
        DBSSLMode:      env("DB_SSLMODE", "disable"),
        DBMaxOpenConns: envInt("DB_MAX_OPEN_CONNS", 10),
        DBMaxIdleConns: envInt("DB_MAX_IDLE_CONNS", 10),
        NatsURL:        env("NATS_URL", "nats://localhost:4222"),
        ServiceName:    env("SERVICE_NAME", "user-api"),
        ServiceVersion: env("SERVICE_VERSION", "dev"),
        LogLevel:       env("LOG_LEVEL", "INFO"),
    }
}

func env(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

**`module.go`** -- Modulo FX compartilhado com providers de infraestrutura:

```go
// apps/user/internal/infrastructure/config/module.go
package config

import (
    "context"
    "database/sql"
    "time"

    "go.uber.org/fx"

    pkgnats "github.com/org/project/pkg/nats"
    "github.com/org/project/pkg/postgres"
)

var InfraModule = fx.Module("infra",
    fx.Provide(
        LoadConfig,
        ProvidePostgres,
        ProvideNATS,
        ProvideNATSPublisher,
    ),
)

func ProvidePostgres(lc fx.Lifecycle, cfg *AppConfig) (*sql.DB, error) {
    db, err := postgres.NewConnection(context.Background(), postgres.Config{
        Host:            cfg.DBHost,
        Port:            cfg.DBPort,
        User:            cfg.DBUser,
        Password:        cfg.DBPassword,
        DBName:          cfg.DBName,
        SSLMode:         cfg.DBSSLMode,
        MaxOpenConns:    cfg.DBMaxOpenConns,
        MaxIdleConns:    cfg.DBMaxIdleConns,
        ConnMaxLifetime: 30 * time.Minute,
        ConnMaxIdleTime: 5 * time.Minute,
    })
    if err != nil {
        return nil, err
    }

    lc.Append(fx.Hook{
        OnStop: func(ctx context.Context) error {
            return db.Close()
        },
    })

    return db, nil
}

func ProvideNATS(lc fx.Lifecycle, cfg *AppConfig) (*pkgnats.Conn, error) {
    nc, err := pkgnats.NewConnection(pkgnats.Config{
        URL:           cfg.NatsURL,
        Name:          cfg.ServiceName,
        MaxReconnects: -1,
        ReconnectWait: 2 * time.Second,
    })
    if err != nil {
        return nil, err
    }

    lc.Append(fx.Hook{
        OnStop: func(ctx context.Context) error {
            nc.Close()
            return nil
        },
    })

    return nc, nil
}

func ProvideNATSPublisher(nc *pkgnats.Conn, cfg *AppConfig) *pkgnats.Publisher {
    return pkgnats.NewPublisher(nc.JS, cfg.ServiceName)
}
```

**Regras:**
- `InfraModule` fornece **apenas** infraestrutura compartilhada (`*AppConfig`, `*sql.DB`, `*pkgnats.Conn`).
- Registro de rotas e startup do servidor ficam no **`main.go` da API**, NAO no `InfraModule`.
- Logica de startup do subscriber fica no **`main.go` do consumer**.
- Lifecycle hooks `OnStop` para cleanup de conexoes.

---

## Camada de Pacotes (`pkg/`)

Bibliotecas reutilizaveis compartilhadas entre todos os apps. Nunca importam `internal/` de nenhum app.

| Pacote | Responsabilidade |
|---|---|
| `postgres` | `NewConnection`, pool config, `RunInTx`, health check |
| `nats` | `NewConnection`, `EnsureStream`, `Publisher`, `Subscribe` |
| `httpserver` | `NewFiberApp()` com middlewares padrao |
| `controller` | `BaseController`, `RequestContext`, `BindAndHandle`, `mapDomainError` |
| `logger` | `Setup()` com slog JSON + `service` e `version` globais |
| `event` | Envelope base e contratos de eventos compartilhados |

**Regras:**
- Sem estado global (sem `var` em nivel de pacote para conexoes).
- Funcoes puras que recebem e retornam dependencias explicitamente.
- Nunca importa nada de `internal/` de nenhum app.
- Pode ser usada por outros projetos.

---

## Injecao de Dependencia com uber-go/fx

### Composicao no `main.go` da API

Cada binario de API compoe seu proprio grafo FX, reutilizando `config.InfraModule` para infraestrutura compartilhada.

```go
// apps/user/cmd/api/main.go
package main

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/compress"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/helmet"
    "github.com/gofiber/fiber/v3/middleware/recover"
    "github.com/gofiber/fiber/v3/middleware/requestid"
    "go.uber.org/fx"

    "github.com/org/project/pkg/logger"
    pkghttp "github.com/org/project/pkg/httpserver"

    "github.com/org/project/apps/user/internal/infrastructure/config"
    "github.com/org/project/apps/user/internal/infrastructure/controller"
    infrarepo "github.com/org/project/apps/user/internal/infrastructure/repository"
    "github.com/org/project/apps/user/internal/infrastructure/publisher"
    "github.com/org/project/apps/user/internal/infrastructure/adapter"
    appuser "github.com/org/project/apps/user/internal/application/usecase/user"
)

func main() {
    fx.New(
        fx.StopTimeout(30*time.Second),

        // Infraestrutura compartilhada (config, postgres, nats)
        config.InfraModule,

        // Providers especificos da API
        fx.Provide(
            pkghttp.NewFiberApp,

            // repositories
            infrarepo.NewUserRepository,

            // publishers
            publisher.NewUserPublisher,

            // adapters
            adapter.NewBcryptAdapter,

            // use cases
            appuser.NewCreateUseCase,
            appuser.NewGetByIDUseCase,
            appuser.NewListUseCase,

            // controllers
            controller.NewUserController,
        ),

        // Invocacoes especificas da API
        fx.Invoke(
            SetupLogger,
            RegisterRoutes,
            StartServer,
        ),
    ).Run()
}

func SetupLogger(cfg *config.AppConfig) {
    logger.Setup(logger.Config{
        Service: cfg.ServiceName,
        Version: cfg.ServiceVersion,
        Level:   cfg.LogLevel,
    })
}

func RegisterRoutes(
    app *fiber.App,
    userCtrl *controller.UserController,
) {
    userCtrl.RegisterRoutes(app)
}

func StartServer(lc fx.Lifecycle, shutdowner fx.Shutdowner, app *fiber.App, cfg *config.AppConfig) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go func() {
                if err := app.Listen(":" + cfg.AppPort); err != nil {
                    slog.Error("server failed", "error", err)
                    shutdowner.Shutdown()
                }
            }()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            return app.Shutdown()
        },
    })
}
```

### Composicao no `main.go` do Consumer

Cada binario de consumer compoe seu proprio grafo FX. **Sem Fiber, sem controllers, sem rotas.**

```go
// apps/billing/cmd/consumer/main.go
package main

import (
    "context"
    "time"

    "go.uber.org/fx"

    pkgnats "github.com/org/project/pkg/nats"
    "github.com/org/project/pkg/logger"

    "github.com/org/project/apps/billing/internal/infrastructure/config"
    infrarepo "github.com/org/project/apps/billing/internal/infrastructure/repository"
    "github.com/org/project/apps/billing/internal/infrastructure/subscriber"
    appbilling "github.com/org/project/apps/billing/internal/application/usecase/billing"
)

func main() {
    fx.New(
        fx.StopTimeout(30*time.Second),

        config.InfraModule,

        // Sem controller, sem HTTP — apenas o necessario para consumir
        fx.Provide(
            infrarepo.NewAccountRepository,
            appbilling.NewCreateAccountUseCase,
            subscriber.NewUserSubscriber,
        ),

        fx.Invoke(
            SetupLogger,
            EnsureStreams,
            StartSubscriber,
        ),
    ).Run()
}

func SetupLogger(cfg *config.AppConfig) {
    logger.Setup(logger.Config{
        Service: cfg.ServiceName,
        Version: cfg.ServiceVersion,
        Level:   cfg.LogLevel,
    })
}

func EnsureStreams(nc *pkgnats.Conn) error {
    ctx := context.Background()
    _, err := pkgnats.EnsureStream(ctx, nc.JS, pkgnats.StreamConfig{
        Name:     "EVENTS_USER",
        Subjects: []string{"events.user.>"},
        MaxAge:   7 * 24 * time.Hour,
        MaxBytes: 1 * 1024 * 1024 * 1024,
        Replicas: 1,
    })
    return err
}

func StartSubscriber(lc fx.Lifecycle, shutdowner fx.Shutdowner, nc *pkgnats.Conn, sub *subscriber.UserSubscriber) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            err := pkgnats.Subscribe(ctx, nc.JS, pkgnats.ConsumerConfig{
                Stream:     "EVENTS_USER",
                Consumer:   "billing-on-user-created",
                FilterSubj: "events.user.created",
                MaxDeliver: 5,
                AckWait:    30 * time.Second,
            }, sub.HandleUserCreated)
            if err != nil {
                return err
            }
            return nil
        },
    })
}
```

### Regras de Composicao

1. **`config.InfraModule`** e sempre incluido primeiro em todo binario. Fornece `*AppConfig`, `*sql.DB` e `*pkgnats.Conn`.
2. **Cada binario fornece apenas o que precisa.** Uma API fornece controllers; um consumer fornece subscribers. Nenhum fornece as dependencias do outro.
3. **`RegisterRoutes` e `StartServer`** sao definidos no `main.go` da API, nao em um modulo compartilhado.
4. **`EnsureStreams` e `StartSubscriber`** sao definidos no `main.go` do consumer, nao em um modulo compartilhado.
5. **`fx.Provide`** registra construtores. FX resolve parametros automaticamente por tipo.
6. **`fx.Invoke`** registra funcoes chamadas apos todos os providers estarem prontos. Usado para side-effects.
7. **`fx.StopTimeout`** define o timeout maximo para shutdown graceful (padrao 30s).
8. **Lifecycle hooks** (`OnStart`/`OnStop`) gerenciam startup e shutdown graceful.
9. **`fx.Shutdowner`** pode ser injetado para acionar shutdown programatico.

### Ordem de Shutdown

FX executa hooks `OnStop` na ordem **inversa** de registro:

**API:**
1. Servidor HTTP para de aceitar conexoes e aguarda requests em andamento.
2. Conexao NATS faz drain (aguarda mensagens pendentes).
3. PostgreSQL desconecta.

**Consumer:**
1. Subscriber para de consumir mensagens.
2. Conexao NATS faz drain.
3. PostgreSQL desconecta.

---

## Convencoes de Nomenclatura

### Arquivos

| Localizacao | Padrao | Exemplo |
|---|---|---|
| `cmd/<binary>/` | `main.go` | `cmd/api/main.go` |
| `domain/entity/` | `<name>.go` | `user.go` |
| `domain/usecase/<context>/` | `<action>.go` | `user/create.go` |
| `domain/repository/` | `<name>-repository.go` | `user-repository.go` |
| `domain/event/` | `<name>-event.go` | `user-event.go` |
| `domain/<lib>/` | `<name>.go` | `crypt/hasher.go` |
| `application/usecase/<context>/` | `<action>-usecase.go` | `user/create-usecase.go` |
| `infrastructure/controller/` | `<name>-controller.go` | `user-controller.go` |
| `infrastructure/repository/` | `<name>-repository.go` | `user-repository.go` |
| `infrastructure/publisher/` | `<name>-publisher.go` | `user-publisher.go` |
| `infrastructure/subscriber/` | `<name>-subscriber.go` | `user-subscriber.go` |
| `infrastructure/adapter/` | `<lib>-adapter.go` | `bcrypt-adapter.go` |

- Nomes de arquivo sempre em **kebab-case**.
- Testes: `<file>_test.go` (unitarios) ou `<file>_integration_test.go` (integracao).

### Pacotes

Sempre **lowercase** e **singular**: `entity`, `usecase`, `repository`, `event`, `crypt`, `token`, `controller`, `publisher`, `subscriber`, `adapter`, `config`.

### Tipos e Interfaces

| Elemento | Padrao | Exemplo |
|---|---|---|
| Entidade | `<Name>` | `User` |
| Interface UC | `<Action>UseCase` | `CreateUseCase` |
| Interface Repo | `<Name>Repository` | `UserRepository` |
| Interface Event | `<Name>Event` | `UserEvent` |
| Interface Lib | `<Capability>` | `Hasher`, `Manager` |
| Struct Impl UC | `<Action>UseCase` | `CreateUseCase` (sem Impl) |
| Controller | `<Name>Controller` | `UserController` |
| Publisher | `<Name>Publisher` | `UserPublisher` |
| Subscriber | `<Name>Subscriber` | `UserSubscriber` |
| Adapter | `<LibName>Adapter` | `BcryptAdapter` |

### Construtores

| Camada | Assinatura | Tipo de Retorno |
|---|---|---|
| Entidade | `New<Name>(params...) *<Name>` | Ponteiro concreto |
| Application UC | `New<Action>UseCase(interfaces...) <context>usecase.<Action>UseCase` | Interface de dominio |
| Infra Repo | `New<Name>Repository(deps...) repository.<Name>Repository` | Interface de dominio |
| Infra Publisher | `New<Name>Publisher(deps...) event.<Name>Event` | Interface de dominio |
| Controller | `New<Name>Controller(interfaces...) *<Name>Controller` | Ponteiro concreto |
| Subscriber | `New<Name>Subscriber(interfaces...) *<Name>Subscriber` | Ponteiro concreto |
| Adapter | `New<LibName>Adapter(deps...) <domain-pkg>.<Interface>` | Interface de dominio |

### Imports

Organizados em 3 blocos separados por linhas em branco:

```go
import (
    // 1. stdlib
    "context"
    "fmt"

    // 2. third-party
    "github.com/gofiber/fiber/v3"
    "go.uber.org/fx"

    // 3. projeto interno
    "github.com/org/project/apps/user/internal/domain/usecase/user"
    "github.com/org/project/pkg/postgres"
)
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

```go
// pkg/event/event.go
package event

import "time"

type Envelope struct {
    ID        string    `json:"id"`
    Type      string    `json:"type"`
    Source    string    `json:"source"`
    Timestamp time.Time `json:"timestamp"`
    Data      any       `json:"data"`
}
```

```go
// pkg/event/user.go
package event

const (
    SubjectUserCreated = "events.user.created"
    SubjectUserUpdated = "events.user.updated"
)

type UserCreatedData struct {
    UserID string `json:"user_id"`
    Name   string `json:"name"`
    Email  string `json:"email"`
}
```

**Regras:**
- Subjects hierarquicos: `events.<dominio>.<acao>` (ex: `events.user.created`).
- Um stream por dominio: `EVENTS_USER`, `EVENTS_BILLING`.
- Publisher usa `Nats-Msg-Id` = ID da entidade para deduplicacao server-side.
- Consumer duravel com ack explicito, backoff exponencial e MaxDeliver.
- Idempotencia no consumer: `INSERT ... ON CONFLICT DO NOTHING`.

---

## Guia Passo a Passo: Adicionando uma Nova Feature

Exemplo: adicionando o dominio `Order` no app `billing`.

### Passo 1 -- Entidade de Dominio

Crie `apps/billing/internal/domain/entity/order.go`:

```go
package entity

import "time"

type Order struct {
    ID        string    `json:"id"`
    UserID    string    `json:"userId"`
    Amount    int64     `json:"amount"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}

func NewOrder(userID string, amount int64) *Order {
    now := time.Now().UTC()
    return &Order{
        UserID:    userID,
        Amount:    amount,
        Status:    "pending",
        CreatedAt: now,
        UpdatedAt: now,
    }
}
```

### Passo 2 -- Interfaces de Dominio (Use Cases)

Crie uma interface por use case, organizadas por contexto:

Crie `apps/billing/internal/domain/usecase/order/create.go`:

```go
package order

import (
    "context"

    "github.com/org/project/apps/billing/internal/domain/entity"
)

type CreateInput struct {
    UserID string
    Amount int64
}

type CreateOutput struct {
    Order *entity.Order
}

type CreateUseCase interface {
    Perform(ctx context.Context, input CreateInput) (*CreateOutput, error)
}
```

### Passo 2b -- Interfaces de Dominio (Repository e Event)

Crie `apps/billing/internal/domain/repository/order-repository.go`:

```go
package repository

import (
    "context"

    "github.com/org/project/apps/billing/internal/domain/entity"
)

type OrderRepository interface {
    Create(ctx context.Context, order *entity.Order) error
    FindByID(ctx context.Context, id string) (*entity.Order, error)
}
```

Crie `apps/billing/internal/domain/event/order-event.go` (se aplicavel):

```go
package event

import (
    "context"

    "github.com/org/project/apps/billing/internal/domain/entity"
)

type OrderEvent interface {
    PublishCreated(ctx context.Context, order *entity.Order) error
}
```

### Passo 3 -- Use Case de Aplicacao

Crie `apps/billing/internal/application/usecase/order/create-usecase.go`:

```go
package order

import (
    "context"
    "fmt"

    "github.com/org/project/apps/billing/internal/domain/entity"
    "github.com/org/project/apps/billing/internal/domain/repository"
    orderusecase "github.com/org/project/apps/billing/internal/domain/usecase/order"
)

type CreateUseCase struct {
    orderRepository repository.OrderRepository
}

func NewCreateUseCase(orderRepository repository.OrderRepository) orderusecase.CreateUseCase {
    return &CreateUseCase{orderRepository: orderRepository}
}

func (u *CreateUseCase) Perform(ctx context.Context, input orderusecase.CreateInput) (*orderusecase.CreateOutput, error) {
    order := entity.NewOrder(input.UserID, input.Amount)
    if err := u.orderRepository.Create(ctx, order); err != nil {
        return nil, fmt.Errorf("create order: %w", err)
    }
    return &orderusecase.CreateOutput{Order: order}, nil
}
```

### Passo 4 -- Repositorio de Infraestrutura

Crie `apps/billing/internal/infrastructure/repository/order-repository.go`.

### Passo 5 -- Publisher de Infraestrutura (se aplicavel)

Crie `apps/billing/internal/infrastructure/publisher/order-publisher.go`.

### Passo 6a -- Controller (se a API precisar)

Crie `apps/billing/internal/infrastructure/controller/order-controller.go`.

### Passo 6b -- Subscriber (se o Consumer precisar)

Crie `apps/billing/internal/infrastructure/subscriber/order-subscriber.go`.

### Passo 7 -- Migration SQL

Crie `migrations/billing/000002_create_orders.up.sql` seguindo as boas praticas de PostgreSQL 18.

### Passo 8 -- Composicao FX

**Para a API** -- atualize `apps/billing/cmd/api/main.go`:

```go
fx.Provide(
    // ... existentes ...
    infrarepo.NewOrderRepository,
    publisher.NewOrderPublisher,
    apporder.NewCreateUseCase,
    controller.NewOrderController,
),
```

Atualize `RegisterRoutes` no mesmo arquivo:

```go
func RegisterRoutes(app *fiber.App, orderCtrl *controller.OrderController) {
    orderCtrl.RegisterRoutes(app)
}
```

### Passo 9 -- Testes

- Unitario: `apps/billing/internal/application/usecase/order/create-usecase_test.go`
- Integracao: `apps/billing/internal/infrastructure/repository/order-repository_integration_test.go`

### Passo 10 -- Swagger (apenas APIs)

Execute `swag init -g apps/billing/cmd/api/main.go` para regenerar a documentacao.

### Checklist -- Nova Feature

- [ ] `internal/domain/entity/<name>.go` -- Entidade com construtor
- [ ] `internal/domain/usecase/<context>/<action>.go` -- 1 interface por use case (metodo `Perform`)
- [ ] `internal/domain/repository/<name>-repository.go` -- Interface de repositorio
- [ ] `internal/domain/event/<name>-event.go` -- Interface de evento (se aplicavel)
- [ ] `internal/domain/<lib>/<name>.go` -- Interface para lib externa (se aplicavel)
- [ ] `internal/application/usecase/<context>/<action>-usecase.go` -- Implementacao do use case
- [ ] `internal/application/usecase/<context>/<action>-usecase_test.go` -- Teste unitario
- [ ] `internal/infrastructure/repository/<name>-repository.go` -- Implementacao PostgreSQL
- [ ] `internal/infrastructure/repository/<name>-repository_integration_test.go` -- Teste de integracao
- [ ] `internal/infrastructure/publisher/<name>-publisher.go` -- Implementacao NATS (se aplicavel)
- [ ] `internal/infrastructure/adapter/<lib>-adapter.go` -- Implementacao de lib externa (se aplicavel)
- [ ] `internal/infrastructure/controller/<name>-controller.go` -- Controller Fiber v3 (se API)
- [ ] `internal/infrastructure/subscriber/<name>-subscriber.go` -- Subscriber NATS (se Consumer)
- [ ] `migrations/<app>/NNNNNN_<descricao>.up.sql` -- Migration SQL
- [ ] `apps/<app>/cmd/api/main.go` -- Registrar providers e atualizar `RegisterRoutes` (se API)
- [ ] `apps/<app>/cmd/consumer/main.go` -- Criar ou atualizar binario do consumer (se Consumer)
- [ ] Executar `swag init -g apps/<app>/cmd/api/main.go` (se API)

### Checklist -- Novo App

- [ ] Criar `apps/<app-name>/go.mod`
- [ ] Adicionar ao `go.work`
- [ ] Criar `apps/<app-name>/cmd/api/main.go` e/ou `cmd/consumer/main.go`
- [ ] Incluir `config.InfraModule`
- [ ] Fornecer apenas repositorios, publishers, use cases e handlers necessarios
- [ ] Definir funcao de startup (`RegisterRoutes`+`StartServer` para API, `EnsureStreams`+`StartSubscriber` para Consumer)
- [ ] Invocar `SetupLogger` e funcao de startup
- [ ] Criar `migrations/<app-name>/` com migrations iniciais
- [ ] Criar `Dockerfile`
- [ ] Adicionar ao `docker-compose.yml`
