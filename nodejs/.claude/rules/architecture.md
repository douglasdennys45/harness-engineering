# Arquitetura da Aplicacao

Este documento define a arquitetura do projeto, suas camadas, regras de dependencia, convencoes de nomenclatura e o guia passo a passo para adicionar novas features.

## Visao Geral

O projeto segue **Clean Architecture** com inversao de dependencia em uma estrutura de **monorepo** com multiplos microservicos. Camadas internas nunca importam camadas externas. Injecao de dependencia e gerenciada pelo **awilix** (modo CLASSIC, injecao por constructor). O monorepo suporta **N apps independentes** (APIs e Consumers) que compartilham bibliotecas via `pkg/` (workspace `@org/pkg`).

```
apps/<app-name>/src/bin/<binary>.ts     Entry points (um por binario)
       |
       v
apps/<app-name>/src/domain/             Entidades, interfaces (sem dependencias externas)
apps/<app-name>/src/application/        Implementacoes de use cases (depende apenas de domain)
apps/<app-name>/src/infrastructure/     Controllers, repos, publisher, subscriber, config
       |
       v
pkg/                                    Bibliotecas reutilizaveis compartilhadas entre apps
```

### Stack Tecnologica

| Camada | Tecnologia | Versao |
|---|---|---|
| Runtime | Node.js | **22 LTS** |
| Linguagem | TypeScript (strict, ESM) | **5** |
| HTTP | Hyper-Express (uWebSockets.js) | `hyper-express` |
| Banco de Dados | PostgreSQL (via `pg`, SQL manual sem ORM) | **18** |
| Mensageria | NATS JetStream (via `nats`) | **2.12** |
| DI | awilix (modo CLASSIC, constructor injection) | latest |
| Logging | pino (JSON estruturado) | latest |
| Validacao | zod | latest |
| Documentacao | OpenAPI (swagger-jsdoc + swagger-ui) | latest |
| Config | dotenv + variaveis de ambiente | - |
| Migrations | dbmate (SQL puro) | latest |
| Testes | vitest (unit com `vi.fn`), testcontainers (integracao) | latest |
| Lint / Format | ESLint (typescript-eslint) + Prettier | latest |
| Monorepo | pnpm workspaces | latest |

### Tipos de Binarios

| Tipo | Proposito | Exemplo |
|---|---|---|
| **API** | Servidor HTTP expondo endpoints REST (Hyper-Express) | `apps/user/src/bin/api.ts` |
| **Consumer** | Processo em background consumindo eventos NATS JetStream | `apps/user/src/bin/consumer.ts` |

Em desenvolvimento os binarios rodam com **tsx** (`tsx watch src/bin/api.ts`). Em producao, compilados com **tsc** para `dist/` e executados com `node dist/bin/api.js`.

### Regra de Dependencia

```
Domain  <--  Application  <--  Infrastructure  <--  src/bin/
  ^                                  |
  |                                  v
  +--- nunca depende de -------->  pkg/
```

- `domain/` nao importa nenhuma biblioteca externa (excecao documentada: `import type` de tipos do `pg` no `Transactor` — ver secao de transactions).
- `application/` importa apenas `domain/`.
- `infrastructure/` importa `domain/`, `@org/pkg` e bibliotecas externas.
- `pkg/` nunca importa `src/` de nenhum app.
- `src/bin/<binary>.ts` importa tudo para compor o container awilix **daquele binario especifico** (composition root).

---

## Estrutura de Diretorios

```
/
├── pnpm-workspace.yaml                              # pnpm workspaces
├── package.json                                     # scripts raiz (typecheck, lint, test)
├── tsconfig.base.json                               # tsconfig strict compartilhado
├── eslint.config.js
├── .prettierrc
├── docker-compose.yml
│
├── pkg/                                             # workspace: @org/pkg
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── postgres/
│       │   ├── conn.ts                              # createPool + pool config
│       │   ├── tx.ts                                # runInTx helper
│       │   └── health.ts                            # ping health check
│       ├── nats/
│       │   ├── conn.ts                              # connectNats + reconnect
│       │   ├── stream.ts                            # ensureStream
│       │   ├── publisher.ts                         # Publisher generico
│       │   └── subscriber.ts                        # subscribe com consumer duravel
│       ├── http-server/
│       │   └── server.ts                            # createServer com middlewares padrao
│       ├── controller/
│       │   ├── base-controller.ts                   # BaseController (template method)
│       │   ├── request-context.ts                   # RequestContext interface
│       │   ├── validation.ts                        # formatZodError (zod)
│       │   ├── errors.ts                            # HttpError, ValidationError
│       │   └── error-mapper.ts                      # ErrorMapper + ErrorResponse
│       ├── logger/
│       │   └── logger.ts                            # pino JSON setup com service e version
│       └── event/                                   # contratos de eventos compartilhados
│           ├── event.ts                             # Envelope base
│           ├── user.ts                              # subjects + data types
│           └── billing.ts                           # subjects + data types
│
├── migrations/                                      # migrations SQL por app (dbmate)
│   ├── user/
│   │   └── 0001_create_users.sql                    # -- migrate:up / -- migrate:down
│   └── billing/
│       └── 0001_create_accounts.sql
│
├── apps/
│   ├── user/                                        # workspace: @org/user
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── Dockerfile
│   │   ├── src/
│   │   │   ├── bin/
│   │   │   │   ├── api.ts                           # API HTTP — composition root awilix
│   │   │   │   └── consumer.ts                      # Consumer NATS — composition root awilix
│   │   │   ├── domain/
│   │   │   │   ├── entity/
│   │   │   │   │   ├── user.ts                      # Classe de dominio + static create
│   │   │   │   │   └── errors.ts                    # Classes de erro de dominio
│   │   │   │   ├── usecase/
│   │   │   │   │   └── user/                        # Subpasta por contexto de dominio
│   │   │   │   │       ├── create.ts                # CreateUseCase interface
│   │   │   │   │       ├── get-by-id.ts             # GetByIdUseCase interface
│   │   │   │   │       └── list.ts                  # ListUseCase interface
│   │   │   │   ├── repository/
│   │   │   │   │   └── user-repository.ts           # Interface de repositorio
│   │   │   │   ├── event/
│   │   │   │   │   └── user-event.ts                # Interface de eventos/mensageria
│   │   │   │   └── <lib>/                           # Interfaces para libs externas
│   │   │   │       └── <name>.ts                    # Interface de inversao de dependencia
│   │   │   ├── application/
│   │   │   │   └── usecase/
│   │   │   │       └── user/                        # Subpasta por contexto de dominio
│   │   │   │           ├── create-usecase.ts        # Implementacao do use case
│   │   │   │           ├── create-usecase.test.ts   # Teste unitario (vitest + vi.fn)
│   │   │   │           ├── get-by-id-usecase.ts
│   │   │   │           └── list-usecase.ts
│   │   │   └── infrastructure/
│   │   │       ├── config/
│   │   │       │   ├── config.ts                    # AppConfig + loadConfig
│   │   │       │   └── container.ts                 # registerInfrastructure (providers de infra)
│   │   │       ├── controller/
│   │   │       │   └── user-controller.ts           # Controllers HTTP (Hyper-Express) — apenas APIs
│   │   │       ├── repository/
│   │   │       │   ├── user-repository.ts           # Implementacao PostgreSQL (pg.Pool)
│   │   │       │   └── user-repository.integration.test.ts
│   │   │       ├── publisher/
│   │   │       │   ├── user-publisher.ts            # Implementacao NATS JetStream
│   │   │       │   └── user-publisher.integration.test.ts
│   │   │       ├── subscriber/
│   │   │       │   └── user-subscriber.ts           # Consumer NATS — apenas Consumers
│   │   │       └── adapter/
│   │   │           └── <lib>-adapter.ts             # Implementacoes de interfaces de libs externas
│   │   └── test/
│   │       └── integration/
│   │           ├── testhelper/
│   │           │   ├── postgres.ts                  # Container PG compartilhado (testcontainers)
│   │           │   └── nats.ts                      # Container NATS compartilhado
│   │           └── user.integration.test.ts         # Teste E2E do fluxo completo
│   │
│   └── billing/                                     # mesma estrutura interna (@org/billing)
│       ├── package.json
│       ├── tsconfig.json
│       ├── Dockerfile
│       ├── src/
│       │   ├── bin/
│       │   │   ├── api.ts
│       │   │   └── consumer.ts
│       │   ├── domain/
│       │   ├── application/
│       │   └── infrastructure/
│       └── test/
│           └── integration/
```

### Workspace pnpm (`pnpm-workspace.yaml`)

```yaml
packages:
  - pkg
  - apps/*
```

**`package.json` raiz** — scripts que servem de **gates de validacao** para todo o monorepo:

```json
{
  "name": "org-project",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "pnpm -r build",
    "typecheck": "pnpm -r typecheck",
    "lint": "eslint .",
    "format": "prettier --write .",
    "test": "pnpm -r test"
  },
  "devDependencies": {
    "eslint": "latest",
    "typescript-eslint": "latest",
    "prettier": "latest",
    "typescript": "^5"
  }
}
```

| Gate | Comando | O que valida |
|---|---|---|
| Typecheck | `pnpm typecheck` | `tsc --noEmit` em todos os workspaces |
| Lint | `pnpm lint` | ESLint (typescript-eslint) + Prettier |
| Testes | `pnpm test` | `vitest run` em todos os workspaces |

**`pkg/package.json`** — biblioteca compartilhada com subpath exports por modulo:

```json
{
  "name": "@org/pkg",
  "private": true,
  "type": "module",
  "exports": {
    "./postgres": { "types": "./dist/postgres/index.d.ts", "default": "./dist/postgres/index.js" },
    "./nats": { "types": "./dist/nats/index.d.ts", "default": "./dist/nats/index.js" },
    "./http-server": { "types": "./dist/http-server/index.d.ts", "default": "./dist/http-server/index.js" },
    "./controller": { "types": "./dist/controller/index.d.ts", "default": "./dist/controller/index.js" },
    "./logger": { "types": "./dist/logger/index.d.ts", "default": "./dist/logger/index.js" },
    "./event": { "types": "./dist/event/index.d.ts", "default": "./dist/event/index.js" }
  },
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "hyper-express": "^6",
    "nats": "^2",
    "pg": "^8",
    "pino": "^9",
    "zod": "^3"
  }
}
```

Cada app importa `@org/pkg` via protocolo `workspace:`:

```json
{
  "name": "@org/user",
  "private": true,
  "type": "module",
  "scripts": {
    "dev:api": "tsx watch src/bin/api.ts",
    "dev:consumer": "tsx watch src/bin/consumer.ts",
    "build": "tsc -p tsconfig.json",
    "start:api": "node dist/bin/api.js",
    "start:consumer": "node dist/bin/consumer.js",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "@org/pkg": "workspace:*",
    "awilix": "^12",
    "dotenv": "^16",
    "hyper-express": "^6",
    "nats": "^2",
    "pg": "^8",
    "pino": "^9",
    "zod": "^3"
  },
  "devDependencies": {
    "tsx": "^4",
    "typescript": "^5",
    "vitest": "^3",
    "testcontainers": "^10",
    "@testcontainers/postgresql": "^10"
  }
}
```

**`tsconfig.base.json`** — TypeScript strict + ESM (`NodeNext`):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "declaration": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

**Regras do workspace:**
- Todos os pacotes usam `"type": "module"` (ESM). Imports relativos **sempre com extensao `.js`** (resolucao NodeNext): `import { User } from '../entity/user.js'`.
- `verbatimModuleSyntax` obriga `import type` para imports somente de tipo.
- `pnpm -r` executa scripts em ordem topologica — `pkg/` e construido antes dos apps que dependem dele.
- **Nunca minificar** o build: o awilix em modo CLASSIC resolve dependencias pelo **nome dos parametros do constructor**; minificacao quebra a resolucao.

### Convencao de Nomes para `apps/` e `src/bin/`

- **Apps**: `apps/<app-name>/` -- lowercase (ex: `apps/user/`, `apps/billing/`).
- **APIs**: `apps/<app>/src/bin/api.ts`.
- **Consumers**: `apps/<app>/src/bin/consumer.ts`. Se houver multiplos consumers, usar `src/bin/consumer-<dominio>.ts`.
- Cada arquivo em `src/bin/` e um **composition root completo**: carrega config, registra o container awilix, inicia o processo e trata shutdown. Nenhuma logica de negocio.

---

## Camada de Dominio (`src/domain/`)

A camada de dominio contem **entidades** e **contratos** (interfaces). Nao possui dependencias externas — apenas TypeScript puro e APIs nativas do Node (`node:crypto`, etc. quando estritamente necessario). **Isolada dentro de cada app.**

### entity/

Classes representando conceitos de dominio. Cada entidade e uma **classe** com um metodo estatico `create(...)` para novas instancias e um constructor que recebe as props completas (usado para rehidratacao a partir do banco).

**Padrao:**

```typescript
// apps/user/src/domain/entity/user.ts
export interface UserProps {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}

export class User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;

  constructor(props: UserProps) {
    this.id = props.id;
    this.name = props.name;
    this.email = props.email;
    this.createdAt = props.createdAt;
    this.updatedAt = props.updatedAt;
  }

  static create(name: string, email: string): User {
    const now = new Date();
    return new User({
      id: '', // gerado pelo banco via INSERT ... RETURNING "id"
      name,
      email,
      createdAt: now,
      updatedAt: now,
    });
  }
}
```

**Regras:**
- Arquivo: `<name>.ts` em kebab-case se composto (ex: `audit-log.ts`).
- Classe exportada em PascalCase; propriedades em camelCase (serializam direto para JSON camelCase).
- `static create(...)` para criar novas instancias com regras de negocio; `constructor(props)` para rehidratar do banco.
- Timestamps sempre em UTC: `new Date()` (o `Date` do JavaScript e um instante UTC; serializar com `toISOString()`; nunca usar bibliotecas de timezone).
- Sem logica de infraestrutura (sem imports de drivers, frameworks ou bibliotecas).
- Metodos de negocio da entidade (ex: `cancel()`) vivem na propria classe e lancam erros de dominio.

**Erros de dominio** (cada app define os seus em `entity/errors.ts`):

```typescript
// apps/user/src/domain/entity/errors.ts

/** Classe base para todos os erros de dominio do app. */
export class DomainError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
    this.name = new.target.name;
  }
}

export class InvalidInputError extends DomainError {
  constructor(message = 'invalid input') {
    super(message);
  }
}

export class UserNotFoundError extends DomainError {
  constructor(message = 'user not found') {
    super(message);
  }
}

export class UserAlreadyExistsError extends DomainError {
  constructor(message = 'email already in use') {
    super(message);
  }
}

export class InvalidRoleError extends DomainError {
  constructor(message = 'invalid role') {
    super(message);
  }
}

export class ForbiddenError extends DomainError {
  constructor(message = 'forbidden') {
    super(message);
  }
}

// Para manter rastreabilidade, wrapear a causa original com `cause`:
//   throw new UserNotFoundError(`user ${id} not found`);
//   throw new Error('create user failed', { cause: err });
// O ErrorMapper usa `instanceof` para mapear a classe -> HTTP status.
```

### usecase/

Interfaces definindo os use cases de dominio. Cada use case e representado por **uma unica interface com um unico metodo `perform`**. Interfaces sao organizadas em **subpastas por contexto de dominio**.

**Principio:** 1 acao = 1 interface = 1 arquivo. Isso garante interfaces coesas, facilita testes (mocks granulares com `vi.fn()`) e respeita o Interface Segregation Principle.

**Estrutura de diretorios:**

```
src/domain/usecase/
└── user/
    ├── create.ts             # CreateUseCase interface
    ├── get-by-id.ts          # GetByIdUseCase interface
    └── list.ts               # ListUseCase interface
```

**Padrao:**

```typescript
// apps/user/src/domain/usecase/user/create.ts
import type { User } from '../../entity/user.js';

export interface CreateInput {
  name: string;
  email: string;
}

export interface CreateOutput {
  user: User;
}

export interface CreateUseCase {
  perform(input: CreateInput): Promise<CreateOutput>;
}
```

```typescript
// apps/user/src/domain/usecase/user/get-by-id.ts
import type { User } from '../../entity/user.js';

export interface GetByIdInput {
  id: string;
}

export interface GetByIdUseCase {
  perform(input: GetByIdInput): Promise<User | null>;
}
```

```typescript
// apps/user/src/domain/usecase/user/list.ts
import type { User } from '../../entity/user.js';

export interface ListInput {
  cursor: string;
  limit: number;
}

export interface ListOutput {
  users: User[];
  nextCursor: string;
}

export interface ListUseCase {
  perform(input: ListInput): Promise<ListOutput>;
}
```

**Regras:**
- Organizacao: `domain/usecase/<contexto>/<acao>.ts` (ex: `domain/usecase/user/create.ts`).
- Cada arquivo contem **1 interface** com **1 metodo**: `perform`.
- Interface exportada: `<Action>UseCase` (ex: `CreateUseCase`, `GetByIdUseCase`).
- O metodo `perform` recebe **um unico parametro** `input` e retorna `Promise`.
- Interfaces de input (`<Action>Input`) e output (`<Action>Output`) sao definidas no mesmo arquivo da interface.
- Retorna entidades de dominio, outputs e/ou lanca erros de dominio. Ausencia e representada por `null` (nunca `undefined`).

### repository/

Interfaces definindo contratos de persistencia.

**Padrao:**

```typescript
// apps/user/src/domain/repository/user-repository.ts
import type { User } from '../entity/user.js';

export interface UserRepository {
  create(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  list(cursor: string, limit: number): Promise<{ users: User[]; nextCursor: string }>;
  update(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}
```

**Regras:**
- Arquivo: `<name>-repository.ts` (ex: `user-repository.ts`).
- Interface exportada: `<Name>Repository`.
- Todos os metodos retornam `Promise`.
- `findById` retorna `Promise<User | null>` — retorna `null` quando nao encontrado (ausencia nao e erro).

### event/

Interfaces definindo contratos de mensageria/eventos.

**Padrao:**

```typescript
// apps/user/src/domain/event/user-event.ts
import type { User } from '../entity/user.js';

export interface UserEvent {
  publishCreated(user: User): Promise<void>;
  publishUpdated(user: User): Promise<void>;
}
```

**Regras:**
- Arquivo: `<name>-event.ts` (ex: `user-event.ts`).
- Interface exportada: `<Name>Event`.
- Metodos nomeados por acao: `publish<Action>`.

### Interfaces para Bibliotecas Externas (`domain/<lib>/`)

Toda biblioteca externa utilizada no projeto **deve ser acessada via inversao de dependencia**. A interface e definida na camada de dominio em uma subpasta nomeada pelo conceito/capacidade que a lib fornece. A implementacao concreta fica em `infrastructure/adapter/`.

**Principio:** O dominio nunca conhece a lib concreta. Ele define **o que precisa** (interface), e a infraestrutura fornece **como fazer** (implementacao com a lib).

**Estrutura de diretorios:**

```
src/domain/
├── crypt/
│   └── hasher.ts              # Interface para hashing de senhas
├── token/
│   └── manager.ts             # Interface para geracao/validacao de tokens
└── mailer/
    └── sender.ts              # Interface para envio de emails
```

**Padrao:**

```typescript
// apps/user/src/domain/crypt/hasher.ts
export interface Hasher {
  hash(plain: string): Promise<string>;
  compare(hashed: string, plain: string): Promise<boolean>;
}
```

**Regras:**
- Subpasta nomeada pelo **conceito/capacidade**, nao pelo nome da lib (ex: `crypt/`, nao `bcrypt/`; `token/`, nao `jwt/`).
- Interface exportada com nome descritivo da capacidade (ex: `Hasher`, `Manager`, `Sender`).
- **Nenhum import de lib externa** -- apenas TypeScript puro.
- As implementacoes concretas ficam em `infrastructure/adapter/`.

---

## Camada de Aplicacao (`src/application/`)

Contem a **implementacao** dos use cases. Orquestra entidades, repositorios e eventos. **Isolada dentro de cada app.**

### usecase/

Implementacoes dos use cases, organizadas em **subpastas por contexto de dominio**, espelhando a estrutura de `domain/usecase/`.

**Estrutura de diretorios:**

```
src/application/usecase/
└── user/
    ├── create-usecase.ts           # CreateUseCase class + perform
    ├── create-usecase.test.ts      # Teste unitario com vi.fn()
    ├── get-by-id-usecase.ts
    └── list-usecase.ts
```

**Padrao:**

```typescript
// apps/user/src/application/usecase/user/create-usecase.ts
import { UserAlreadyExistsError } from '../../../domain/entity/errors.js';
import { User } from '../../../domain/entity/user.js';
import type { UserEvent } from '../../../domain/event/user-event.js';
import type { UserRepository } from '../../../domain/repository/user-repository.js';
import type * as usecase from '../../../domain/usecase/user/create.js';

export class CreateUseCase implements usecase.CreateUseCase {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly userEvent: UserEvent,
  ) {}

  async perform(input: usecase.CreateInput): Promise<usecase.CreateOutput> {
    const existing = await this.userRepository.findByEmail(input.email);
    if (existing !== null) {
      throw new UserAlreadyExistsError();
    }

    const user = User.create(input.name, input.email);

    await this.userRepository.create(user);

    // publica evento DEPOIS do commit no banco.
    // falha na publicacao nao desfaz a operacao de negocio.
    await this.userEvent.publishCreated(user).catch(() => undefined);

    return { user };
  }
}
```

**Regras:**
- Organizacao: `application/usecase/<contexto>/<acao>-usecase.ts`.
- Classe: `<Action>UseCase` (mesmo nome da interface, sem sufixo `Impl`) implementando a interface de dominio via `implements usecase.<Action>UseCase`.
- A interface de dominio e importada como **namespace type-only**: `import type * as usecase from '.../domain/usecase/<contexto>/<acao>.js'` — equivalente a um "pacote" que desambigua o nome da classe do nome da interface.
- Dependencias entram **pelo constructor** como `private readonly` e sao **interfaces de dominio** (nunca tipos concretos).
- Os **nomes dos parametros do constructor** devem coincidir exatamente com os nomes registrados no container awilix (modo CLASSIC): `userRepository`, `userEvent`, `hasher`, etc.
- O metodo implementado e sempre `perform`.
- Depende apenas de `src/domain/`. Nunca importa `infrastructure/` ou `@org/pkg`.
- Erros re-lancados com contexto usando `cause`: `throw new Error('create user: failed', { cause: err })` — ou simplesmente propagados (`async/await` ja propaga).
- Evento publicado **apos** persistencia no banco, nunca antes.

---

## Camada de Infraestrutura (`src/infrastructure/`)

Implementa contratos de dominio usando tecnologias concretas e gerencia a configuracao da aplicacao. Cada binario conecta apenas o que precisa.

### controller/ (apenas APIs)

Controllers HTTP usando Hyper-Express. O projeto utiliza um **BaseController** em `pkg/controller/` que implementa o template method pattern: extrai dados comuns da request (requestId, IP, headers), executa validacao e delega para o metodo concreto. Cada controller **estende** o `BaseController` e so implementa a logica especifica. **Usado apenas por binarios de API.**

#### BaseController — `pkg/controller/` (compartilhado entre todos os apps)

O `BaseController` vive no `pkg/` porque e infraestrutura generica **sem nenhuma logica de dominio**. Todos os apps reutilizam a mesma base, evitando duplicacao.

Ele centraliza:
- Extracao automatica de `requestId`, `parentId`, `IP`, `userAgent`.
- Bind + validacao do body (zod) com resposta padronizada.
- Tratamento uniforme de erros (dominio -> HTTP status).
- Logging estruturado com contexto da request (logger pino filho).
- Suporte a transactions quando o handler precisa.

**Estrutura do `pkg/controller/`:**

```
pkg/src/
└── controller/
    ├── base-controller.ts      # BaseController class + handle/bindAndHandle (template method)
    ├── request-context.ts      # RequestContext interface
    ├── validation.ts           # formatZodError
    ├── errors.ts               # HttpError, ValidationError
    └── error-mapper.ts         # ErrorMapper, ErrorResponse
```

**Estrutura de cada app (so o codigo especifico):**

```
apps/user/src/infrastructure/controller/
├── user-controller.ts          # UserController (estende pkg BaseController)
└── order-controller.ts         # OrderController (estende pkg BaseController)
```

**`pkg/controller/request-context.ts` — RequestContext:**

```typescript
// pkg/src/controller/request-context.ts
import type { Logger } from 'pino';

/**
 * RequestContext contem dados extraidos automaticamente de toda request.
 * Handlers recebem este objeto ja preenchido, sem precisar extrair manualmente.
 */
export interface RequestContext {
  requestId: string;
  parentId: string;
  ip: string;
  userAgent: string;
  logger: Logger;
}
```

**`pkg/controller/errors.ts` — Erros HTTP genericos:**

```typescript
// pkg/src/controller/errors.ts

/** Erro com status HTTP explicito (equivalente a um "erro de framework"). */
export class HttpError extends Error {
  constructor(
    readonly status: number,
    message: string,
  ) {
    super(message);
    this.name = new.target.name;
  }
}

/** Erro de validacao de input (bind do body ou query). Sempre vira 400. */
export class ValidationError extends HttpError {
  constructor(message: string) {
    super(400, message);
  }
}
```

**`pkg/controller/base-controller.ts` — BaseController (template method):**

```typescript
// pkg/src/controller/base-controller.ts
import type { MiddlewareHandler, Request, Response } from 'hyper-express';
import type { Logger } from 'pino';
import type { ZodType } from 'zod';

import { ErrorMapper } from './error-mapper.js';
import { ValidationError } from './errors.js';
import type { RequestContext } from './request-context.js';
import { formatZodError } from './validation.js';

/** HandlerFn e a assinatura que handlers concretos implementam. */
export type HandlerFn = (
  request: Request,
  response: Response,
  rc: RequestContext,
) => Promise<void>;

/** BindHandlerFn e a assinatura para handlers que recebem body tipado e validado. */
export type BindHandlerFn<T> = (
  request: Request,
  response: Response,
  rc: RequestContext,
  input: T,
) => Promise<void>;

/**
 * BaseController centraliza extracao de contexto, logging e error handling.
 * Todo controller do projeto estende esta classe.
 */
export abstract class BaseController {
  constructor(
    protected readonly errorMapper: ErrorMapper,
    protected readonly logger: Logger,
  ) {}

  /**
   * handle e o template method. Extrai o contexto comum e delega para o handler concreto.
   * Todo handler do projeto deve ser registrado via este metodo (ou bindAndHandle).
   */
  protected handle(handler: HandlerFn): MiddlewareHandler {
    return async (request: Request, response: Response): Promise<void> => {
      const rc = this.extractContext(request);

      try {
        await handler(request, response, rc);
      } catch (err) {
        this.handleError(response, rc, err);
      }
    };
  }

  /**
   * bindAndHandle faz parse (`await request.json()`) + validacao zod do body
   * automaticamente antes de chamar o handler. O handler so e executado se o
   * body for valido — recebe `input` ja tipado como a saida do schema.
   */
  protected bindAndHandle<T>(schema: ZodType<T>, handler: BindHandlerFn<T>): MiddlewareHandler {
    return this.handle(async (request, response, rc) => {
      const body: unknown = await request.json({});

      const result = schema.safeParse(body);
      if (!result.success) {
        // erro de parse OU de validacao — ambos retornam 400
        throw new ValidationError(formatZodError(result.error));
      }

      await handler(request, response, rc, result.data);
    });
  }

  /** parseQuery valida query parameters com zod. Lanca ValidationError (400) se invalido. */
  protected parseQuery<T>(schema: ZodType<T>, request: Request): T {
    const result = schema.safeParse(request.query_parameters);
    if (!result.success) {
      throw new ValidationError(formatZodError(result.error));
    }
    return result.data;
  }

  /** extractContext extrai dados comuns da request uma unica vez. */
  private extractContext(request: Request): RequestContext {
    // preenchido pelo middleware de requestId do pkg/http-server
    const requestId = String(request.locals['requestId'] ?? '');
    const parentId = request.header('x-request-id') ?? '';

    const logger = this.logger.child({
      requestId,
      parentId,
      method: request.method,
      route: request.path,
    });

    return {
      requestId,
      parentId,
      ip: request.ip,
      userAgent: request.header('user-agent') ?? '',
      logger,
    };
  }

  /** handleError mapeia erros de dominio para respostas HTTP padronizadas. */
  private handleError(response: Response, rc: RequestContext, err: unknown): void {
    const { status, message } = this.errorMapper.map(err);

    if (status >= 500) {
      rc.logger.error({ statusCode: status, err }, 'request failed');
    } else {
      rc.logger.warn({ statusCode: status, error: message }, 'request rejected');
    }

    response.status(status).json({ error: message, status });
  }
}
```

**`pkg/controller/validation.ts` — formatacao de erros zod:**

O projeto utiliza **zod** como engine de validacao. Todo body e validado com `schema.safeParse` dentro do `bindAndHandle` — o handler nunca ve input invalido.

```typescript
// pkg/src/controller/validation.ts
import type { ZodError } from 'zod';

/**
 * formatZodError converte os issues do zod em uma mensagem legivel.
 * Nunca expor o erro cru do zod para o usuario.
 */
export function formatZodError(error: ZodError): string {
  const messages = error.issues.map((issue) => {
    const field = issue.path.length > 0 ? issue.path.join('.') : 'body';
    return `${field}: ${issue.message}`;
  });
  return `validation failed: ${messages.join('; ')}`;
}
```

**Servidor HTTP padrao (`pkg/http-server/server.ts`):**

```typescript
// pkg/src/http-server/server.ts
import { randomUUID } from 'node:crypto';

import HyperExpress from 'hyper-express';

export interface ServerOptions {
  maxBodyLength?: number;
}

export function createServer(options: ServerOptions = {}): HyperExpress.Server {
  const server = new HyperExpress.Server({
    max_body_length: options.maxBodyLength ?? 4 * 1024 * 1024, // 4MB
  });

  // requestId unico em toda request (equivalente ao middleware requestid)
  server.use((request, _response, next) => {
    request.locals['requestId'] = randomUUID();
    next();
  });

  // safety net: erro nao tratado que escapou do BaseController (equivalente ao recover)
  server.set_error_handler((_request, response, _error) => {
    response.status(500).json({ error: 'internal server error', status: 500 });
  });

  server.set_not_found_handler((_request, response) => {
    response.status(404).json({ error: 'route not found', status: 404 });
  });

  return server;
}
```

Com isso, todo `bindAndHandle(schema, handler)` executa automaticamente:
1. Parse do JSON via `await request.json()`
2. Validacao via schema zod (`safeParse`)
3. Resposta 400 formatada se parse/validacao falhar

### Validacao com zod — Referencia Rapida

Validacoes mais usadas nos schemas de input (equivalencia com validadores classicos):

| Validacao | zod | Exemplo |
|---|---|---|
| Campo obrigatorio | campo sem `.optional()` | `z.string().min(1)` |
| Email valido | `.email()` | `z.string().email()` |
| Tamanho minimo (string) | `.min(n)` | `z.string().min(3)` |
| Tamanho maximo (string) | `.max(n)` | `z.string().max(100)` |
| Maior que (numeros) | `.gt(n)` / `.positive()` | `z.number().int().positive()` |
| Maior ou igual | `.gte(n)` | `z.number().gte(1)` |
| Um dos valores listados | `z.enum([...])` | `z.enum(['pending', 'active', 'inactive'])` |
| UUID valido | `.uuid()` | `z.string().uuid()` |
| URL valida | `.url()` | `z.string().url()` |
| Valida cada elemento do array | `z.array(schema)` | `z.array(itemSchema).min(1)` |
| So valida se preenchido | `.optional()` | `z.string().email().optional()` |
| Query string -> numero | `z.coerce.number()` | `z.coerce.number().int().max(100).default(20)` |

**Padrao para schemas de input** (definidos no arquivo do controller, tipos inferidos com `z.infer`):

```typescript
import { z } from 'zod';

const createOrderSchema = z.object({
  userId: z.string().uuid(),
  totalCents: z.number().int().positive(),
  currency: z.enum(['BRL', 'USD', 'EUR']),
  notes: z.string().max(500).optional(),
  items: z
    .array(
      z.object({
        productId: z.string().uuid(),
        quantity: z.number().int().positive(),
        unitPriceCents: z.number().int().positive(),
      }),
    )
    .min(1),
});

type CreateOrderInput = z.infer<typeof createOrderSchema>;
```

**Regras de validacao:**
- **Sempre** validar input com schema zod via `bindAndHandle`/`parseQuery`. Nunca validar manualmente no handler.
- **`.uuid()`** em campos que sao IDs.
- **`z.enum`** para enumeracoes (em vez de validar no use case).
- **`z.array(...).min(1)`** valida cada elemento do array individualmente (zod valida elementos por padrao).
- **`.optional()`** em campos opcionais que so devem ser validados se preenchidos.
- **`z.coerce`** para query parameters (chegam sempre como string).
- Tipos de input **sempre** inferidos do schema com `z.infer<typeof schema>` — nunca duplicar o tipo a mao.
- Mensagens de erro formatadas pelo `formatZodError` — nunca expor o erro cru do zod para o usuario.

**`pkg/controller/error-mapper.ts` — Mapeamento extensivel de erros:**

```typescript
// pkg/src/controller/error-mapper.ts
import { HttpError } from './errors.js';

/** ErrorResponse e o formato padrao de resposta de erro da API. */
export interface ErrorResponse {
  error: string;
  status: number;
}

type ErrorClass = new (...args: never[]) => Error;

interface ErrorMapping {
  target: ErrorClass;
  statusCode: number;
}

/**
 * ErrorMapper contem o mapeamento de erros de dominio -> HTTP status.
 * Cada app registra suas proprias classes de erro no mapper.
 */
export class ErrorMapper {
  private readonly mappings: ErrorMapping[] = [];

  /** register adiciona um mapeamento classe de erro -> status. */
  register(target: ErrorClass, statusCode: number): this {
    this.mappings.push({ target, statusCode });
    return this;
  }

  /**
   * map retorna { status, message } para um erro.
   * HttpError usa o proprio status. Erros nao mapeados viram 500 Internal Server Error.
   */
  map(err: unknown): { status: number; message: string } {
    if (err instanceof HttpError) {
      return { status: err.status, message: err.message };
    }

    if (err instanceof Error) {
      for (const { target, statusCode } of this.mappings) {
        if (err instanceof target) {
          return { status: statusCode, message: err.message };
        }
      }
    }

    return { status: 500, message: 'internal server error' };
  }
}
```

**Registro dos erros no app (wiring no `bin/api.ts`):**

```typescript
// apps/user/src/bin/api.ts
import { ErrorMapper } from '@org/pkg/controller';

import {
  ForbiddenError,
  InvalidInputError,
  InvalidRoleError,
  UserAlreadyExistsError,
  UserNotFoundError,
} from '../domain/entity/errors.js';

/** provideErrorMapper configura o mapeamento de erros DESTE app. */
function provideErrorMapper(): ErrorMapper {
  return new ErrorMapper()
    .register(InvalidInputError, 400)
    .register(UserNotFoundError, 404)
    .register(UserAlreadyExistsError, 409)
    .register(InvalidRoleError, 422)
    .register(ForbiddenError, 403);
}

// registrado no container awilix:
//   errorMapper: asFunction(provideErrorMapper).singleton()
```

#### Controller Concreto — Exemplo Simples (sem transaction)

```typescript
// apps/user/src/infrastructure/controller/user-controller.ts
import { BaseController, ErrorMapper, type RequestContext } from '@org/pkg/controller';
import HyperExpress, { type Request, type Response } from 'hyper-express';
import type { Logger } from 'pino';
import { z } from 'zod';

import { UserNotFoundError } from '../../domain/entity/errors.js';
import type { CreateUseCase } from '../../domain/usecase/user/create.js';
import type { GetByIdUseCase } from '../../domain/usecase/user/get-by-id.js';
import type { ListUseCase } from '../../domain/usecase/user/list.js';

const createUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
});

type CreateUserInput = z.infer<typeof createUserSchema>;

const listUsersQuerySchema = z.object({
  cursor: z.string().default(''),
  limit: z.coerce.number().int().positive().max(100).default(20),
});

export class UserController extends BaseController {
  constructor(
    errorMapper: ErrorMapper,                       // injetado via awilix (CLASSIC)
    logger: Logger,
    private readonly createUserUseCase: CreateUseCase,
    private readonly getUserByIdUseCase: GetByIdUseCase,
    private readonly listUsersUseCase: ListUseCase,
  ) {
    super(errorMapper, logger);
  }

  registerRoutes(server: HyperExpress.Server): void {
    const router = new HyperExpress.Router();
    router.post('/', this.bindAndHandle(createUserSchema, this.create));
    router.get('/:id', this.handle(this.getById));
    router.get('/', this.handle(this.list));
    server.use('/v1/users', router);
  }

  /**
   * @openapi
   * /v1/users:
   *   post:
   *     summary: Cria um usuario
   *     tags: [users]
   *     requestBody:
   *       required: true
   *       content:
   *         application/json:
   *           schema:
   *             $ref: '#/components/schemas/CreateUserInput'
   *     responses:
   *       201: { description: Usuario criado }
   *       400: { description: Input invalido }
   *       409: { description: Email ja em uso }
   */
  private readonly create = async (
    _request: Request,
    response: Response,
    rc: RequestContext,
    input: CreateUserInput,
  ): Promise<void> => {
    // body ja foi parseado e validado pelo bindAndHandle
    // requestId, parentId, IP, logger ja estao no rc — zero boilerplate
    const output = await this.createUserUseCase.perform({
      name: input.name,
      email: input.email,
    });
    // erros lancados sobem para BaseController.handleError -> ErrorMapper -> HTTP status

    rc.logger.info({ userId: output.user.id }, 'user created');
    response.status(201).json(output.user);
  };

  private readonly getById = async (
    request: Request,
    response: Response,
    _rc: RequestContext,
  ): Promise<void> => {
    const { id } = request.params;

    const user = await this.getUserByIdUseCase.perform({ id: id ?? '' });
    if (user === null) {
      throw new UserNotFoundError();
    }

    response.status(200).json(user);
  };

  private readonly list = async (
    request: Request,
    response: Response,
    _rc: RequestContext,
  ): Promise<void> => {
    const filter = this.parseQuery(listUsersQuerySchema, request);

    const output = await this.listUsersUseCase.perform({
      cursor: filter.cursor,
      limit: filter.limit,
    });

    response.status(200).json(output);
  };
}
```

**Nota sobre `this`:** handlers sao **arrow functions como propriedades de classe** (`private readonly create = async (...) => {...}`) para preservar o `this` ao serem passados como referencia para `handle`/`bindAndHandle`. Nunca usar metodos comuns sem `.bind(this)`.

#### Controller com Transaction — Exemplo Completo

Quando um endpoint precisa executar **multiplas operacoes atomicas** (ex: criar order + items + atualizar estoque), o controller orquestra a transaction via `Transactor`:

```typescript
// apps/billing/src/infrastructure/controller/order-controller.ts
import { BaseController, ErrorMapper, type RequestContext } from '@org/pkg/controller';
import HyperExpress, { type Request, type Response } from 'hyper-express';
import type { Logger } from 'pino';
import { z } from 'zod';

import { OrderNotFoundError } from '../../domain/entity/errors.js';
import { Order, OrderItem } from '../../domain/entity/order.js';
import type { OrderEvent } from '../../domain/event/order-event.js';
import type { OrderRepository } from '../../domain/repository/order-repository.js';
import type { StockRepository } from '../../domain/repository/stock-repository.js';
import type { Transactor } from '../../domain/repository/transactor.js';

const createOrderSchema = z.object({
  userId: z.string().min(1),
  totalCents: z.number().int().positive(),
  items: z
    .array(
      z.object({
        productId: z.string().min(1),
        quantity: z.number().int().positive(),
        unitPriceCents: z.number().int().positive(),
      }),
    )
    .min(1),
});

type CreateOrderInput = z.infer<typeof createOrderSchema>;

export class OrderController extends BaseController {
  constructor(
    errorMapper: ErrorMapper,                        // injetado via awilix (CLASSIC)
    logger: Logger,
    private readonly transactor: Transactor,
    private readonly orderRepository: OrderRepository,
    private readonly stockRepository: StockRepository,
    private readonly orderEvent: OrderEvent,
  ) {
    super(errorMapper, logger);
  }

  registerRoutes(server: HyperExpress.Server): void {
    const router = new HyperExpress.Router();
    router.post('/', this.bindAndHandle(createOrderSchema, this.create));
    router.post('/:id/cancel', this.handle(this.cancel));
    server.use('/v1/orders', router);
  }

  /**
   * create — operacao que PRECISA de transaction.
   * Cria order + items + debita estoque atomicamente.
   */
  private readonly create = async (
    _request: Request,
    response: Response,
    rc: RequestContext,
    input: CreateOrderInput,
  ): Promise<void> => {
    // === TRANSACTION: tudo dentro e atomico ===
    const order = await this.transactor.runInTx(async (client) => {
      // 1. criar order
      const created = await this.orderRepository.createWithTx(
        client,
        Order.create(input.userId, input.totalCents),
      );

      // 2. criar items (mesmo client/tx)
      for (const item of input.items) {
        await this.orderRepository.createItemWithTx(
          client,
          OrderItem.create(created.id, item.productId, item.quantity, item.unitPriceCents),
        );
      }

      // 3. debitar estoque (mesmo client/tx)
      for (const item of input.items) {
        await this.stockRepository.debitWithTx(client, item.productId, item.quantity);
      }

      return created; // COMMIT — order, items e estoque persistidos atomicamente
    });
    // erro dentro do runInTx => ROLLBACK e o erro sobe para o ErrorMapper

    // === DEPOIS DO COMMIT: publicar evento ===
    // Nunca publicar evento dentro da transaction
    await this.orderEvent.publishCreated(order).catch(() => undefined);

    rc.logger.info({ orderId: order.id }, 'order created');
    response.status(201).json(order);
  };

  /**
   * cancel — operacao SIMPLES, sem transaction.
   * Apenas atualiza status. Uma unica operacao nao precisa de tx.
   */
  private readonly cancel = async (
    request: Request,
    response: Response,
    rc: RequestContext,
  ): Promise<void> => {
    const { id } = request.params;

    const order = await this.orderRepository.findById(id ?? '');
    if (order === null) {
      throw new OrderNotFoundError();
    }

    order.cancel(); // regra de negocio (ex: lanca erro se ja cancelada)

    await this.orderRepository.update(order);

    await this.orderEvent.publishCancelled(order).catch(() => undefined);

    rc.logger.info({ orderId: order.id }, 'order cancelled');
    response.status(200).json(order);
  };
}
```

#### Transactor — Interface de Dominio

O `Transactor` e a **unica excecao documentada** de import externo no dominio: importa **apenas o tipo** `PoolClient` do `pg` (type-only, sem codigo em runtime), cumprindo o papel que um handle de transaction da stdlib cumpriria.

```typescript
// apps/billing/src/domain/repository/transactor.ts
import type { PoolClient } from 'pg';

export interface Transactor {
  runInTx<T>(fn: (client: PoolClient) => Promise<T>): Promise<T>;
}
```

```typescript
// apps/billing/src/infrastructure/repository/transactor.ts
import { runInTx } from '@org/pkg/postgres';
import type { Pool, PoolClient } from 'pg';

import type { Transactor } from '../../domain/repository/transactor.js';

export class PostgresTransactor implements Transactor {
  constructor(private readonly pool: Pool) {}

  runInTx<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    return runInTx(this.pool, fn);
  }
}
```

**Helper compartilhado (`pkg/postgres/tx.ts`):**

```typescript
// pkg/src/postgres/tx.ts
import type { Pool, PoolClient } from 'pg';

/**
 * runInTx abre uma connection dedicada, executa fn dentro de BEGIN/COMMIT
 * e faz ROLLBACK em caso de erro. A connection sempre volta para o pool.
 */
export async function runInTx<T>(pool: Pool, fn: (client: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

#### Repositorio com Metodos `withTx`

Quando um repositorio participa de uma transaction externa, expoe metodos que recebem `PoolClient`:

```typescript
// apps/billing/src/domain/repository/order-repository.ts
import type { PoolClient } from 'pg';

import type { Order, OrderItem } from '../entity/order.js';

export interface OrderRepository {
  // Metodos normais (sem transaction, autocommit via pool)
  findById(id: string): Promise<Order | null>;
  update(order: Order): Promise<void>;

  // Metodos withTx (participam de transaction externa)
  createWithTx(client: PoolClient, order: Order): Promise<Order>;
  createItemWithTx(client: PoolClient, item: OrderItem): Promise<OrderItem>;
}
```

```typescript
// apps/billing/src/infrastructure/repository/order-repository.ts

export class PostgresOrderRepository implements OrderRepository {
  constructor(private readonly pool: Pool) {}

  async createWithTx(client: PoolClient, order: Order): Promise<Order> {
    const query = `
      INSERT INTO "orders" ("userId", "status", "totalCents", "createdAt", "updatedAt")
      VALUES ($1, $2, $3, $4, $5)
      RETURNING "id"`;

    const result = await client.query<{ id: string }>(query, [
      order.userId,
      order.status,
      order.totalCents,
      order.createdAt,
      order.updatedAt,
    ]);

    order.id = result.rows[0]!.id;
    return order;
  }

  async findById(id: string): Promise<Order | null> {
    const query = `
      SELECT "id", "userId", "status", "totalCents", "createdAt", "updatedAt"
      FROM "orders"
      WHERE "id" = $1`;

    const result = await this.pool.query<OrderRow>(query, [id]);
    const row = result.rows[0];
    if (row === undefined) {
      return null;
    }
    return this.toEntity(row);
  }
}
```

#### Quando Usar Transaction no Controller

| Cenario | Transaction? | Pattern |
|---|---|---|
| GET (leitura simples) | **Nao** | `this.handle(this.getById)` |
| POST com 1 INSERT | **Nao** | `this.bindAndHandle(schema, this.create)` — use case faz INSERT unico |
| POST com N INSERTs relacionados | **Sim** | `transactor.runInTx` no handler |
| PUT/PATCH simples | **Nao** | Use case faz UPDATE unico |
| PUT que move saldo (debito + credito) | **Sim** | `transactor.runInTx` no handler |
| DELETE com cascade manual | **Sim** | `transactor.runInTx` no handler |
| Operacao + publicacao de evento | **Evento FORA do tx** | `runInTx(...)` depois `event.publish(...)` |

#### Regras do Controller

- `BaseController`, `RequestContext`, `bindAndHandle`, `ErrorMapper` e `ErrorResponse` vivem em **`pkg/controller/`**. Nenhum app reimplementa essa logica.
- Arquivo: `<name>-controller.ts` (ex: `user-controller.ts`) em `src/infrastructure/controller/`.
- Classe: `<Name>Controller` que **estende** `BaseController`.
- Constructor: `(errorMapper, logger, ...useCases)` chamando `super(errorMapper, logger)` — resolvido via awilix (nomes dos parametros = nomes de registro no container).
- Dependencias sao **interfaces de dominio de use case**, **repositorios** (quando precisa de tx) e **Transactor**.
- Deve implementar `registerRoutes(server: HyperExpress.Server): void` criando um `HyperExpress.Router` e montando via `server.use('/v1/<recurso>', router)`.
- Rotas registradas via `this.handle(this.method)` ou `this.bindAndHandle(schema, this.method)`.
- Handlers sao **arrow functions como propriedades de classe** para preservar o `this`.
- Handlers recebem `RequestContext` ja populado — nunca extrair requestId, IP, etc. manualmente.
- Handlers delegam logica de negocio para use cases. Sem logica de negocio no controller.
- Handlers HTTP usam anotacoes JSDoc `@openapi` (swagger-jsdoc).
- Todo body e validado com **schema zod** via `bindAndHandle`; query parameters via `parseQuery`. Nunca `JSON.parse` manual nem acesso direto a body sem validacao.
- Path parameters lidos de `request.params`; query parameters de `request.query_parameters` (sempre via `parseQuery`).
- Evento publicado **DEPOIS** do commit da transaction, nunca dentro.
- Erros de dominio propagados com `throw` — o `BaseController.handleError` mapeia para HTTP status via `ErrorMapper`.
- Cada app registra suas classes de erro de dominio no `ErrorMapper` via `provideErrorMapper()` no `bin/api.ts`. O `pkg/controller` nao conhece nenhum erro de dominio — ele so recebe o mapeamento.

### subscriber/ (apenas Consumers)

Handlers de consumo NATS JetStream. Cada subscriber consome de um subject especifico e delega o processamento para use cases. **Usado apenas por binarios de Consumer.**

**Padrao:**

```typescript
// apps/billing/src/infrastructure/subscriber/user-subscriber.ts
import { randomUUID } from 'node:crypto';

import type { Envelope, UserCreatedData } from '@org/pkg/event';
import type { JsMsg } from 'nats';
import type { Logger } from 'pino';
import { z } from 'zod';

import type { CreateAccountUseCase } from '../../domain/usecase/billing/create-account.js';

const userCreatedDataSchema = z.object({
  userId: z.string().min(1),
  name: z.string().min(1),
  email: z.string().email(),
});

export class UserSubscriber {
  constructor(
    private readonly createAccountUseCase: CreateAccountUseCase,
    private readonly logger: Logger,
  ) {}

  async handleUserCreated(msg: JsMsg): Promise<void> {
    const requestId = randomUUID();
    const parentId = msg.headers?.get('Nats-Msg-Id') ?? '';

    const logger = this.logger.child({
      requestId,
      parentId,
      subject: msg.subject,
    });

    let envelope: Envelope<UserCreatedData>;
    try {
      envelope = msg.json<Envelope<UserCreatedData>>();
    } catch (err) {
      logger.error({ err }, 'unmarshal envelope failed');
      msg.term(); // mensagem corrompida — nao reenviar
      return;
    }

    const parsed = userCreatedDataSchema.safeParse(envelope.data);
    if (!parsed.success) {
      logger.error({ issues: parsed.error.issues }, 'invalid event data');
      msg.term();
      return;
    }

    logger.info('processing user created event');

    try {
      await this.createAccountUseCase.perform({
        userId: parsed.data.userId,
        name: parsed.data.name,
        email: parsed.data.email,
      });
    } catch (err) {
      logger.error({ err }, 'create account failed');
      msg.nak(); // redelivery com backoff
      return;
    }

    logger.info('account created successfully');
    msg.ack();
  }
}
```

**Regras:**
- Arquivo: `<name>-subscriber.ts` (ex: `user-subscriber.ts`).
- Classe: `<Name>Subscriber`.
- Dependencias sao **interfaces de dominio de use case** (+ `logger`), injetadas pelo constructor via awilix.
- Handlers nomeados: `handle<EventName>` (ex: `handleUserCreated`), com assinatura `(msg: JsMsg) => Promise<void>`.
- Payload decodificado com `msg.json<T>()` e **validado com zod** antes de processar.
- `msg.ack()` apenas apos processamento com sucesso.
- `msg.nak()` para erros recuperaveis (JetStream faz redelivery com backoff).
- `msg.term()` para mensagens corrompidas/invalidas (nao reenviar).
- Handler **nunca deixa excecao escapar**: todo caminho termina em `ack`, `nak` ou `term`.
- Logging estruturado com `requestId` e `parentId` em todo handler.

### repository/

Implementacoes de repositorio usando PostgreSQL via `pg` (node-postgres). Queries SQL escritas manualmente, sem ORM.

**Padrao:**

```typescript
// apps/user/src/infrastructure/repository/user-repository.ts
import type { Pool } from 'pg';

import { User } from '../../domain/entity/user.js';
import type { UserRepository } from '../../domain/repository/user-repository.js';

interface UserRow {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}

export class PostgresUserRepository implements UserRepository {
  constructor(private readonly pool: Pool) {}

  async create(user: User): Promise<void> {
    const query = `
      INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
      VALUES ($1, $2, $3, $4)
      RETURNING "id"`;

    const result = await this.pool.query<{ id: string }>(query, [
      user.name,
      user.email,
      user.createdAt,
      user.updatedAt,
    ]);

    user.id = result.rows[0]!.id;
  }

  async findById(id: string): Promise<User | null> {
    const query = `
      SELECT "id", "name", "email", "createdAt", "updatedAt"
      FROM "users"
      WHERE "id" = $1`;

    const result = await this.pool.query<UserRow>(query, [id]);
    const row = result.rows[0];
    if (row === undefined) {
      return null; // ausencia nao e erro
    }
    return this.toEntity(row);
  }

  async findByEmail(email: string): Promise<User | null> {
    const query = `
      SELECT "id", "name", "email", "createdAt", "updatedAt"
      FROM "users"
      WHERE "email" = $1`;

    const result = await this.pool.query<UserRow>(query, [email]);
    const row = result.rows[0];
    if (row === undefined) {
      return null;
    }
    return this.toEntity(row);
  }

  // list usa paginacao por cursor (sem OFFSET)
  async list(cursor: string, limit: number): Promise<{ users: User[]; nextCursor: string }> {
    let result;
    if (cursor === '') {
      const query = `
        SELECT "id", "name", "email", "createdAt", "updatedAt"
        FROM "users"
        ORDER BY "id" ASC
        LIMIT $1`;
      result = await this.pool.query<UserRow>(query, [limit + 1]);
    } else {
      const query = `
        SELECT "id", "name", "email", "createdAt", "updatedAt"
        FROM "users"
        WHERE "id" > $1
        ORDER BY "id" ASC
        LIMIT $2`;
      result = await this.pool.query<UserRow>(query, [cursor, limit + 1]);
    }

    const users = result.rows.map((row) => this.toEntity(row));

    let nextCursor = '';
    if (users.length > limit) {
      nextCursor = users[limit]!.id;
      users.length = limit;
    }

    return { users, nextCursor };
  }

  private toEntity(row: UserRow): User {
    return new User({
      id: row.id,
      name: row.name,
      email: row.email,
      createdAt: row.createdAt,
      updatedAt: row.updatedAt,
    });
  }
}
```

**Regras:**
- Arquivo: `<name>-repository.ts` (ex: `user-repository.ts`).
- Classe: `Postgres<Name>Repository` implementando a interface de dominio `<Name>Repository`.
- Recebe `Pool` (do `pg`) pelo constructor (registrado no container como `pool`).
- SQL escrito manualmente com placeholders posicionais (`$1`, `$2`). Sem ORM. **Nunca** interpolar valores na string da query.
- Sem `SELECT *`. Listar colunas explicitamente, entre aspas duplas (`"createdAt"`).
- `INSERT ... RETURNING "id"` para obter ID gerado pelo banco.
- Zero linhas retorna `null` (nao e erro, e ausencia).
- Paginacao por cursor, nunca `OFFSET` (para tabelas grandes). Buscar `limit + 1` linhas para descobrir o `nextCursor`.
- Queries como `const` dentro do metodo.
- Tipar as linhas com uma interface `<Name>Row` privada do arquivo e converter para entidade em um metodo `toEntity`.
- Erros do driver propagam com `throw`; quando precisar de contexto, re-lancar com `new Error('find user by id: failed', { cause: err })`.

### publisher/

Implementacoes de eventos usando NATS JetStream.

**Padrao:**

```typescript
// apps/user/src/infrastructure/publisher/user-publisher.ts
import { SUBJECT_USER_CREATED, SUBJECT_USER_UPDATED } from '@org/pkg/event';
import type { Publisher } from '@org/pkg/nats';

import type { User } from '../../domain/entity/user.js';
import type { UserEvent } from '../../domain/event/user-event.js';

export class UserPublisher implements UserEvent {
  constructor(private readonly publisher: Publisher) {}

  async publishCreated(user: User): Promise<void> {
    await this.publisher.publish(SUBJECT_USER_CREATED, user.id, {
      userId: user.id,
      name: user.name,
      email: user.email,
    });
  }

  async publishUpdated(user: User): Promise<void> {
    await this.publisher.publish(SUBJECT_USER_UPDATED, user.id, {
      userId: user.id,
      name: user.name,
      email: user.email,
    });
  }
}
```

**Publisher generico compartilhado (`pkg/nats/publisher.ts`):**

```typescript
// pkg/src/nats/publisher.ts
import { randomUUID } from 'node:crypto';

import type { JetStreamClient } from 'nats';

import type { Envelope } from '../event/event.js';

export class Publisher {
  constructor(
    private readonly js: JetStreamClient,
    private readonly source: string,
  ) {}

  /** publish envia um Envelope para o subject com msgID para deduplicacao server-side. */
  async publish(subject: string, msgId: string, data: unknown): Promise<void> {
    const envelope: Envelope = {
      id: randomUUID(),
      type: subject,
      source: this.source,
      timestamp: new Date().toISOString(),
      data,
    };

    // msgID vira o header Nats-Msg-Id — JetStream descarta duplicatas na janela de dedup
    await this.js.publish(subject, JSON.stringify(envelope), { msgID: msgId });
  }
}
```

**Regras:**
- Arquivo: `<name>-publisher.ts` (ex: `user-publisher.ts`).
- Classe: `<Name>Publisher` implementando a interface de dominio `<Name>Event`.
- Registrado no container com o **nome da interface de dominio**: `userEvent: asClass(UserPublisher).singleton()` — assim os use cases recebem `userEvent` sem conhecer a implementacao.
- `msgId` = ID da entidade para deduplicacao no JetStream (`Nats-Msg-Id`).
- Publicar DEPOIS do commit no banco, nunca antes.

### adapter/

Implementacoes concretas das interfaces de bibliotecas externas definidas em `domain/<lib>/`.

**Padrao:**

```typescript
// apps/user/src/infrastructure/adapter/bcrypt-adapter.ts
import bcrypt from 'bcrypt';

import type { Hasher } from '../../domain/crypt/hasher.js';

export class BcryptAdapter implements Hasher {
  private readonly cost = 12;

  async hash(plain: string): Promise<string> {
    return bcrypt.hash(plain, this.cost);
  }

  async compare(hashed: string, plain: string): Promise<boolean> {
    return bcrypt.compare(plain, hashed);
  }
}
```

**Regras:**
- Arquivo: `<lib-name>-adapter.ts` (ex: `bcrypt-adapter.ts`, `jwt-adapter.ts`).
- Classe: `<LibName>Adapter` implementando a interface de dominio.
- Registrado no container com o **nome da capacidade de dominio**: `hasher: asClass(BcryptAdapter).singleton()`.
- O nome do arquivo/classe reflete a **lib concreta**, pois esta na camada de infraestrutura.

### config/

Configuracao da aplicacao e registro compartilhado de infraestrutura no container.

**`config.ts`** -- Interface de configuracao e carregamento de variaveis de ambiente:

```typescript
// apps/user/src/infrastructure/config/config.ts

export interface AppConfig {
  appPort: number;
  dbHost: string;
  dbPort: number;
  dbUser: string;
  dbPassword: string;
  dbName: string;
  dbSsl: boolean;
  dbPoolMax: number;
  natsUrl: string;
  serviceName: string;
  serviceVersion: string;
  logLevel: string;
}

export function loadConfig(): AppConfig {
  return {
    appPort: envInt('APP_PORT', 8080),
    dbHost: env('DB_HOST', 'localhost'),
    dbPort: envInt('DB_PORT', 5432),
    dbUser: env('DB_USER', 'app'),
    dbPassword: env('DB_PASSWORD', ''),
    dbName: env('DB_NAME', 'users'),
    dbSsl: env('DB_SSL', 'false') === 'true',
    dbPoolMax: envInt('DB_POOL_MAX', 10),
    natsUrl: env('NATS_URL', 'nats://localhost:4222'),
    serviceName: env('SERVICE_NAME', 'user-api'),
    serviceVersion: env('SERVICE_VERSION', 'dev'),
    logLevel: env('LOG_LEVEL', 'info'),
  };
}

function env(key: string, fallback: string): string {
  const value = process.env[key];
  return value !== undefined && value !== '' ? value : fallback;
}

function envInt(key: string, fallback: number): number {
  const value = Number(process.env[key]);
  return Number.isInteger(value) ? value : fallback;
}
```

O `.env` e carregado no **topo de cada binario** com `import 'dotenv/config'` — nunca dentro de `loadConfig`.

**`container.ts`** -- Registro compartilhado de infraestrutura (equivalente ao modulo de infra):

```typescript
// apps/user/src/infrastructure/config/container.ts
import { createPool } from '@org/pkg/postgres';
import { connectNats, Publisher } from '@org/pkg/nats';
import { createLogger } from '@org/pkg/logger';
import { asFunction, asValue, type AwilixContainer } from 'awilix';
import type { NatsConnection } from 'nats';
import type { Pool } from 'pg';

import { loadConfig, type AppConfig } from './config.js';

/**
 * registerInfrastructure conecta e registra a infraestrutura compartilhada
 * (config, logger, pool PG, conexao NATS, publisher) no container.
 * Recursos assincronos sao criados ANTES do registro — awilix nao aguarda factories async.
 */
export async function registerInfrastructure(container: AwilixContainer): Promise<void> {
  const config = loadConfig();

  const logger = createLogger({
    service: config.serviceName,
    version: config.serviceVersion,
    level: config.logLevel,
  });

  const pool = createPool({
    host: config.dbHost,
    port: config.dbPort,
    user: config.dbUser,
    password: config.dbPassword,
    database: config.dbName,
    ssl: config.dbSsl,
    max: config.dbPoolMax,
  });

  const natsConnection = await connectNats({
    servers: config.natsUrl,
    name: config.serviceName,
  });

  container.register({
    config: asValue(config),
    logger: asValue(logger),

    // disposers rodam no container.dispose() durante o shutdown graceful
    pool: asFunction(() => pool)
      .singleton()
      .disposer((p: Pool) => p.end()),

    natsConnection: asFunction(() => natsConnection)
      .singleton()
      .disposer((nc: NatsConnection) => nc.drain()),

    publisher: asFunction(
      (natsConnection: NatsConnection, config: AppConfig) =>
        new Publisher(natsConnection.jetstream(), config.serviceName),
    ).singleton(),
  });
}
```

**Regras:**
- `registerInfrastructure` fornece **apenas** infraestrutura compartilhada (`config`, `logger`, `pool`, `natsConnection`, `publisher`).
- Registro de rotas e startup do servidor ficam no **`bin/api.ts`**, NAO no `container.ts`.
- Logica de startup do subscriber fica no **`bin/consumer.ts`**.
- Recursos assincronos (`connectNats`) sao aguardados **antes** de registrar — factories do awilix sao sincronas.
- Disposers (`.disposer(...)`) registrados para cleanup de conexoes no `container.dispose()`.

---

## Camada de Pacotes (`pkg/` — workspace `@org/pkg`)

Bibliotecas reutilizaveis compartilhadas entre todos os apps. Nunca importam `src/` de nenhum app.

| Modulo | Import | Responsabilidade |
|---|---|---|
| `postgres` | `@org/pkg/postgres` | `createPool`, pool config, `runInTx`, health check |
| `nats` | `@org/pkg/nats` | `connectNats`, `ensureStream`, `Publisher`, `subscribe` |
| `http-server` | `@org/pkg/http-server` | `createServer()` com middlewares padrao (requestId, error handler) |
| `controller` | `@org/pkg/controller` | `BaseController`, `RequestContext`, `bindAndHandle`, `ErrorMapper` |
| `logger` | `@org/pkg/logger` | `createLogger()` com pino JSON + `service` e `version` na base |
| `event` | `@org/pkg/event` | `Envelope` base e contratos de eventos compartilhados |

**`pkg/logger/logger.ts`:**

```typescript
// pkg/src/logger/logger.ts
import { pino, type Logger } from 'pino';

export interface LoggerConfig {
  service: string;
  version: string;
  level: string;
}

export function createLogger(config: LoggerConfig): Logger {
  return pino({
    level: config.level.toLowerCase(),
    base: { service: config.service, version: config.version },
    timestamp: pino.stdTimeFunctions.isoTime,
  });
}
```

**`pkg/postgres/conn.ts`:**

```typescript
// pkg/src/postgres/conn.ts
import pg from 'pg';

export interface PostgresConfig {
  host: string;
  port: number;
  user: string;
  password: string;
  database: string;
  ssl: boolean;
  max: number;
}

export function createPool(config: PostgresConfig): pg.Pool {
  return new pg.Pool({
    host: config.host,
    port: config.port,
    user: config.user,
    password: config.password,
    database: config.database,
    ssl: config.ssl,
    max: config.max,
    idleTimeoutMillis: 5 * 60 * 1000,
    connectionTimeoutMillis: 10 * 1000,
  });
}
```

**`pkg/nats/conn.ts` e `pkg/nats/stream.ts`:**

```typescript
// pkg/src/nats/conn.ts
import { connect, type NatsConnection } from 'nats';

export interface NatsConfig {
  servers: string;
  name: string;
  reconnectTimeWaitMs?: number;
}

export async function connectNats(config: NatsConfig): Promise<NatsConnection> {
  return connect({
    servers: config.servers,
    name: config.name,
    maxReconnectAttempts: -1, // reconectar para sempre
    reconnectTimeWait: config.reconnectTimeWaitMs ?? 2000,
    waitOnFirstConnect: true,
  });
}
```

```typescript
// pkg/src/nats/stream.ts
import { RetentionPolicy, StorageType, type NatsConnection, type StreamConfig } from 'nats';

export interface StreamOptions {
  name: string;
  subjects: string[];
  maxAgeDays: number;
  maxBytes: number;
  replicas: number;
}

/** ensureStream cria o stream se nao existir, ou atualiza a configuracao. */
export async function ensureStream(nc: NatsConnection, opts: StreamOptions): Promise<void> {
  const jsm = await nc.jetstreamManager();

  const config: Partial<StreamConfig> = {
    name: opts.name,
    subjects: opts.subjects,
    retention: RetentionPolicy.Limits,
    storage: StorageType.File,
    max_age: opts.maxAgeDays * 24 * 60 * 60 * 1_000_000_000, // nanos
    max_bytes: opts.maxBytes,
    num_replicas: opts.replicas,
  };

  try {
    await jsm.streams.info(opts.name);
    await jsm.streams.update(opts.name, config);
  } catch {
    await jsm.streams.add(config);
  }
}
```

**`pkg/nats/subscriber.ts`:**

```typescript
// pkg/src/nats/subscriber.ts
import { AckPolicy, type ConsumerMessages, type JsMsg, type NatsConnection } from 'nats';

export interface ConsumerOptions {
  stream: string;
  consumer: string;       // nome do consumer duravel
  filterSubject: string;
  maxDeliver: number;
  ackWaitSeconds: number;
}

/**
 * subscribe garante o consumer duravel e inicia o loop de consumo.
 * Retorna ConsumerMessages — chamar `.stop()` no shutdown graceful.
 */
export async function subscribe(
  nc: NatsConnection,
  opts: ConsumerOptions,
  handler: (msg: JsMsg) => Promise<void>,
): Promise<ConsumerMessages> {
  const jsm = await nc.jetstreamManager();

  await jsm.consumers.add(opts.stream, {
    durable_name: opts.consumer,
    ack_policy: AckPolicy.Explicit,
    filter_subject: opts.filterSubject,
    max_deliver: opts.maxDeliver,
    ack_wait: opts.ackWaitSeconds * 1_000_000_000, // nanos
  });

  const consumer = await nc.jetstream().consumers.get(opts.stream, opts.consumer);
  const messages = await consumer.consume();

  void (async () => {
    for await (const msg of messages) {
      // o handler e responsavel por ack/nak/term e nunca lanca
      await handler(msg);
    }
  })();

  return messages;
}
```

**Regras:**
- Sem estado global (sem `let`/`const` de modulo para conexoes — tudo criado por funcao e injetado).
- Funcoes puras que recebem e retornam dependencias explicitamente.
- Nunca importa nada de `apps/*/src/`.
- Pode ser usado por outros projetos.

---

## Injecao de Dependencia com awilix

O container awilix e criado em **modo CLASSIC**: as dependencias sao resolvidas pelo **nome dos parametros do constructor**. Cada binario (`bin/api.ts`, `bin/consumer.ts`) e um **composition root** que registra apenas o que precisa.

| Registro | Uso |
|---|---|
| `asValue(x)` | Valores prontos (config, logger) |
| `asFunction(fn).singleton()` | Factories (pool, conexao NATS, errorMapper) — com `.disposer(fn)` para cleanup |
| `asClass(C).singleton()` | Classes com constructor injection (repos, use cases, controllers, subscribers) |

### Composicao no `bin/api.ts` da API

Cada binario de API compoe seu proprio container, reutilizando `registerInfrastructure` para infraestrutura compartilhada.

```typescript
// apps/user/src/bin/api.ts
import 'dotenv/config';

import { ErrorMapper } from '@org/pkg/controller';
import { createServer } from '@org/pkg/http-server';
import { asClass, asFunction, createContainer, InjectionMode } from 'awilix';
import type { Logger } from 'pino';

import { CreateUseCase } from '../application/usecase/user/create-usecase.js';
import { GetByIdUseCase } from '../application/usecase/user/get-by-id-usecase.js';
import { ListUseCase } from '../application/usecase/user/list-usecase.js';
import {
  ForbiddenError,
  InvalidInputError,
  InvalidRoleError,
  UserAlreadyExistsError,
  UserNotFoundError,
} from '../domain/entity/errors.js';
import { BcryptAdapter } from '../infrastructure/adapter/bcrypt-adapter.js';
import { registerInfrastructure } from '../infrastructure/config/container.js';
import type { AppConfig } from '../infrastructure/config/config.js';
import { UserController } from '../infrastructure/controller/user-controller.js';
import { UserPublisher } from '../infrastructure/publisher/user-publisher.js';
import { PostgresUserRepository } from '../infrastructure/repository/user-repository.js';

function provideErrorMapper(): ErrorMapper {
  return new ErrorMapper()
    .register(InvalidInputError, 400)
    .register(UserNotFoundError, 404)
    .register(UserAlreadyExistsError, 409)
    .register(InvalidRoleError, 422)
    .register(ForbiddenError, 403);
}

async function main(): Promise<void> {
  const container = createContainer({ injectionMode: InjectionMode.CLASSIC });

  // Infraestrutura compartilhada (config, logger, pool, nats, publisher)
  await registerInfrastructure(container);

  // Providers especificos da API
  container.register({
    errorMapper: asFunction(provideErrorMapper).singleton(),

    // repositories
    userRepository: asClass(PostgresUserRepository).singleton(),

    // publishers (registrados pelo nome da interface de dominio)
    userEvent: asClass(UserPublisher).singleton(),

    // adapters (registrados pelo nome da capacidade de dominio)
    hasher: asClass(BcryptAdapter).singleton(),

    // use cases (nome de registro: <acao><Contexto>UseCase)
    createUserUseCase: asClass(CreateUseCase).singleton(),
    getUserByIdUseCase: asClass(GetByIdUseCase).singleton(),
    listUsersUseCase: asClass(ListUseCase).singleton(),

    // controllers
    userController: asClass(UserController).singleton(),
  });

  const config = container.resolve<AppConfig>('config');
  const logger = container.resolve<Logger>('logger');

  // Servidor HTTP + rotas
  const server = createServer();
  container.resolve<UserController>('userController').registerRoutes(server);

  await server.listen(config.appPort);
  logger.info({ port: config.appPort }, 'api started');

  // === Shutdown graceful ===
  const shutdown = async (signal: string): Promise<void> => {
    logger.info({ signal }, 'shutting down');

    // 1. HTTP para de aceitar novas conexoes
    server.close();

    // 2/3. disposers do container: NATS drain, depois pool PG end
    await container.dispose();

    logger.info('shutdown complete');
    process.exit(0);
  };

  process.on('SIGTERM', () => void shutdown('SIGTERM'));
  process.on('SIGINT', () => void shutdown('SIGINT'));
}

main().catch((err: unknown) => {
  console.error('fatal:', err);
  process.exit(1);
});
```

### Composicao no `bin/consumer.ts` do Consumer

Cada binario de consumer compoe seu proprio container. **Sem Hyper-Express, sem controllers, sem rotas.**

```typescript
// apps/billing/src/bin/consumer.ts
import 'dotenv/config';

import { ensureStream, subscribe } from '@org/pkg/nats';
import { asClass, createContainer, InjectionMode } from 'awilix';
import type { NatsConnection } from 'nats';
import type { Logger } from 'pino';

import { CreateAccountUseCase } from '../application/usecase/billing/create-account-usecase.js';
import { registerInfrastructure } from '../infrastructure/config/container.js';
import { PostgresAccountRepository } from '../infrastructure/repository/account-repository.js';
import { UserSubscriber } from '../infrastructure/subscriber/user-subscriber.js';

async function main(): Promise<void> {
  const container = createContainer({ injectionMode: InjectionMode.CLASSIC });

  await registerInfrastructure(container);

  // Sem controller, sem HTTP — apenas o necessario para consumir
  container.register({
    accountRepository: asClass(PostgresAccountRepository).singleton(),
    createAccountUseCase: asClass(CreateAccountUseCase).singleton(),
    userSubscriber: asClass(UserSubscriber).singleton(),
  });

  const logger = container.resolve<Logger>('logger');
  const natsConnection = container.resolve<NatsConnection>('natsConnection');

  // Garante o stream antes de consumir
  await ensureStream(natsConnection, {
    name: 'EVENTS_USER',
    subjects: ['events.user.>'],
    maxAgeDays: 7,
    maxBytes: 1 * 1024 * 1024 * 1024,
    replicas: 1,
  });

  // Inicia o consumer duravel
  const userSubscriber = container.resolve<UserSubscriber>('userSubscriber');
  const messages = await subscribe(
    natsConnection,
    {
      stream: 'EVENTS_USER',
      consumer: 'billing-on-user-created',
      filterSubject: 'events.user.created',
      maxDeliver: 5,
      ackWaitSeconds: 30,
    },
    (msg) => userSubscriber.handleUserCreated(msg),
  );

  logger.info('consumer started');

  // === Shutdown graceful ===
  const shutdown = async (signal: string): Promise<void> => {
    logger.info({ signal }, 'shutting down');

    // 1. para de consumir novas mensagens
    messages.stop();

    // 2/3. disposers do container: NATS drain (aguarda acks pendentes), depois pool PG end
    await container.dispose();

    logger.info('shutdown complete');
    process.exit(0);
  };

  process.on('SIGTERM', () => void shutdown('SIGTERM'));
  process.on('SIGINT', () => void shutdown('SIGINT'));
}

main().catch((err: unknown) => {
  console.error('fatal:', err);
  process.exit(1);
});
```

### Regras de Composicao

1. **`registerInfrastructure(container)`** e sempre chamado primeiro em todo binario. Fornece `config`, `logger`, `pool`, `natsConnection` e `publisher`.
2. **Cada binario registra apenas o que precisa.** Uma API registra controllers; um consumer registra subscribers. Nenhum registra as dependencias do outro.
3. **Startup do servidor HTTP** (`createServer` + `registerRoutes` + `listen`) fica no `bin/api.ts`, nao em um modulo compartilhado.
4. **`ensureStream` e `subscribe`** ficam no `bin/consumer.ts`, nao em um modulo compartilhado.
5. **Modo CLASSIC**: awilix resolve os parametros do constructor **pelo nome**. O nome do parametro deve ser identico ao nome de registro (`userRepository`, `userEvent`, `createUserUseCase`, `errorMapper`, `logger`, `pool`, ...).
6. **Nomes de registro** sao camelCase e unicos no container: use cases usam `<acao><Contexto>UseCase` (ex: `createUserUseCase`), implementacoes de contratos de dominio usam o **nome do contrato** (`userRepository`, `userEvent`, `hasher`, `transactor`).
7. **Tudo `.singleton()`**: repos, use cases, controllers e subscribers nao guardam estado por request.
8. **`.disposer(fn)`** registra cleanup executado pelo `container.dispose()` no shutdown.
9. **Nunca usar o container como service locator** dentro das camadas: `container.resolve` so aparece no composition root (`bin/`).

### Ordem de Shutdown

Os handlers de `SIGTERM`/`SIGINT` executam o shutdown graceful na ordem:

**API:**
1. `server.close()` — Hyper-Express para de aceitar novas conexoes.
2. `container.dispose()` — disposer do NATS faz `drain()` (aguarda mensagens pendentes).
3. `container.dispose()` — disposer do PostgreSQL faz `pool.end()`.

**Consumer:**
1. `messages.stop()` — subscriber para de consumir mensagens.
2. `container.dispose()` — NATS `drain()`.
3. `container.dispose()` — PostgreSQL `pool.end()`.

### Documentacao OpenAPI (apenas APIs)

- Handlers anotados com JSDoc `@openapi` (formato OpenAPI 3) diretamente no controller.
- O spec e gerado com **swagger-jsdoc** varrendo `src/infrastructure/controller/**/*.ts` e servido com **swagger-ui** em `/docs` (rota registrada no `bin/api.ts`).
- Script por app: `"openapi": "tsx scripts/generate-openapi.ts"` — regenerar sempre que rotas/schemas mudarem.

---

## Convencoes de Nomenclatura

### Arquivos

| Localizacao | Padrao | Exemplo |
|---|---|---|
| `src/bin/` | `<binary>.ts` | `api.ts`, `consumer.ts` |
| `domain/entity/` | `<name>.ts` | `user.ts` |
| `domain/usecase/<context>/` | `<action>.ts` | `user/create.ts` |
| `domain/repository/` | `<name>-repository.ts` | `user-repository.ts` |
| `domain/event/` | `<name>-event.ts` | `user-event.ts` |
| `domain/<lib>/` | `<name>.ts` | `crypt/hasher.ts` |
| `application/usecase/<context>/` | `<action>-usecase.ts` | `user/create-usecase.ts` |
| `infrastructure/controller/` | `<name>-controller.ts` | `user-controller.ts` |
| `infrastructure/repository/` | `<name>-repository.ts` | `user-repository.ts` |
| `infrastructure/publisher/` | `<name>-publisher.ts` | `user-publisher.ts` |
| `infrastructure/subscriber/` | `<name>-subscriber.ts` | `user-subscriber.ts` |
| `infrastructure/adapter/` | `<lib>-adapter.ts` | `bcrypt-adapter.ts` |

- Nomes de arquivo sempre em **kebab-case**.
- Testes: `<file>.test.ts` (unitarios, vitest) ou `<file>.integration.test.ts` (integracao, testcontainers).

### Diretorios

Sempre **lowercase** e **singular**: `entity`, `usecase`, `repository`, `event`, `crypt`, `token`, `controller`, `publisher`, `subscriber`, `adapter`, `config`, `bin`.

### Tipos, Interfaces e Classes

| Elemento | Padrao | Exemplo |
|---|---|---|
| Entidade | `<Name>` (classe) | `User` |
| Interface UC | `<Action>UseCase` | `CreateUseCase` |
| Interface Repo | `<Name>Repository` | `UserRepository` |
| Interface Event | `<Name>Event` | `UserEvent` |
| Interface Lib | `<Capability>` | `Hasher`, `Manager` |
| Classe Impl UC | `<Action>UseCase` (sem `Impl`) | `CreateUseCase` |
| Classe Impl Repo | `Postgres<Name>Repository` | `PostgresUserRepository` |
| Controller | `<Name>Controller` | `UserController` |
| Publisher | `<Name>Publisher` | `UserPublisher` |
| Subscriber | `<Name>Subscriber` | `UserSubscriber` |
| Adapter | `<LibName>Adapter` | `BcryptAdapter` |

- Classes e interfaces em **PascalCase**. Interfaces **sem** prefixo `I`.
- Quando a classe implementa uma interface homonima, o arquivo da implementacao importa a interface como namespace type-only: `import type * as usecase from '.../create.js'` e usa `implements usecase.CreateUseCase`.

### Nomes de Registro no Container (awilix CLASSIC)

| Elemento | Nome de registro | Exemplo |
|---|---|---|
| Config | `config` | `asValue(config)` |
| Logger | `logger` | `asValue(logger)` |
| Pool PG | `pool` | `asFunction(() => pool).singleton().disposer(...)` |
| Conexao NATS | `natsConnection` | idem |
| Publisher generico | `publisher` | `asFunction(...).singleton()` |
| ErrorMapper | `errorMapper` | `asFunction(provideErrorMapper).singleton()` |
| Repositorio | `<name>Repository` | `userRepository: asClass(PostgresUserRepository)` |
| Evento (publisher) | `<name>Event` | `userEvent: asClass(UserPublisher)` |
| Lib externa (adapter) | `<capability>` | `hasher: asClass(BcryptAdapter)` |
| Transactor | `transactor` | `transactor: asClass(PostgresTransactor)` |
| Use case | `<action><Context>UseCase` | `createUserUseCase: asClass(CreateUseCase)` |
| Controller | `<name>Controller` | `userController: asClass(UserController)` |
| Subscriber | `<name>Subscriber` | `userSubscriber: asClass(UserSubscriber)` |

> Os **parametros do constructor** de qualquer classe registrada devem usar exatamente esses nomes — e assim que o awilix (CLASSIC) resolve o grafo.

### Imports

Organizados em 4 blocos separados por linhas em branco (ESLint `import/order`):

```typescript
// 1. builtins do Node (sempre com prefixo node:)
import { randomUUID } from 'node:crypto';

// 2. third-party
import HyperExpress from 'hyper-express';
import { z } from 'zod';

// 3. workspace compartilhado
import { BaseController } from '@org/pkg/controller';

// 4. relativos do app (sempre com extensao .js — resolucao ESM/NodeNext)
import { User } from '../../domain/entity/user.js';
import type { UserRepository } from '../../domain/repository/user-repository.js';
```

- `import type` obrigatorio para imports somente de tipo (`verbatimModuleSyntax`).
- Imports relativos **sempre** com extensao `.js` (mesmo em arquivos `.ts`).

---

## Comunicacao entre Servicos

Cada app tem seu proprio banco (database-per-service). Comunicacao entre apps e feita via NATS JetStream — **nunca** por chamadas HTTP diretas nem por acesso ao banco do outro app.

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

```typescript
// pkg/src/event/event.ts

/** Envelope base de todo evento publicado no JetStream. */
export interface Envelope<T = unknown> {
  id: string;        // uuid do evento
  type: string;      // subject (ex: events.user.created)
  source: string;    // nome do servico que publicou
  timestamp: string; // ISO-8601 UTC (new Date().toISOString())
  data: T;
}
```

```typescript
// pkg/src/event/user.ts

export const SUBJECT_USER_CREATED = 'events.user.created';
export const SUBJECT_USER_UPDATED = 'events.user.updated';

export interface UserCreatedData {
  userId: string;
  name: string;
  email: string;
}
```

**Regras:**
- Subjects hierarquicos: `events.<dominio>.<acao>` (ex: `events.user.created`).
- Um stream por dominio: `EVENTS_USER`, `EVENTS_BILLING`.
- Publisher usa `msgID` (`Nats-Msg-Id`) = ID da entidade para deduplicacao server-side.
- Consumer duravel com ack explicito, backoff e `max_deliver`.
- Idempotencia no consumer: `INSERT ... ON CONFLICT DO NOTHING`.

---

## Guia Passo a Passo: Adicionando uma Nova Feature

Exemplo: adicionando o dominio `Order` no app `billing`.

### Passo 1 -- Entidade de Dominio

Crie `apps/billing/src/domain/entity/order.ts`:

```typescript
export interface OrderProps {
  id: string;
  userId: string;
  amount: number; // centavos (inteiro)
  status: string;
  createdAt: Date;
  updatedAt: Date;
}

export class Order {
  id: string;
  userId: string;
  amount: number;
  status: string;
  createdAt: Date;
  updatedAt: Date;

  constructor(props: OrderProps) {
    this.id = props.id;
    this.userId = props.userId;
    this.amount = props.amount;
    this.status = props.status;
    this.createdAt = props.createdAt;
    this.updatedAt = props.updatedAt;
  }

  static create(userId: string, amount: number): Order {
    const now = new Date();
    return new Order({
      id: '',
      userId,
      amount,
      status: 'pending',
      createdAt: now,
      updatedAt: now,
    });
  }
}
```

### Passo 2 -- Interfaces de Dominio (Use Cases)

Crie uma interface por use case, organizadas por contexto:

Crie `apps/billing/src/domain/usecase/order/create.ts`:

```typescript
import type { Order } from '../../entity/order.js';

export interface CreateInput {
  userId: string;
  amount: number;
}

export interface CreateOutput {
  order: Order;
}

export interface CreateUseCase {
  perform(input: CreateInput): Promise<CreateOutput>;
}
```

### Passo 2b -- Interfaces de Dominio (Repository e Event)

Crie `apps/billing/src/domain/repository/order-repository.ts`:

```typescript
import type { Order } from '../entity/order.js';

export interface OrderRepository {
  create(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}
```

Crie `apps/billing/src/domain/event/order-event.ts` (se aplicavel):

```typescript
import type { Order } from '../entity/order.js';

export interface OrderEvent {
  publishCreated(order: Order): Promise<void>;
}
```

### Passo 3 -- Use Case de Aplicacao

Crie `apps/billing/src/application/usecase/order/create-usecase.ts`:

```typescript
import { Order } from '../../../domain/entity/order.js';
import type { OrderRepository } from '../../../domain/repository/order-repository.js';
import type * as usecase from '../../../domain/usecase/order/create.js';

export class CreateUseCase implements usecase.CreateUseCase {
  constructor(private readonly orderRepository: OrderRepository) {}

  async perform(input: usecase.CreateInput): Promise<usecase.CreateOutput> {
    const order = Order.create(input.userId, input.amount);
    await this.orderRepository.create(order);
    return { order };
  }
}
```

### Passo 4 -- Repositorio de Infraestrutura

Crie `apps/billing/src/infrastructure/repository/order-repository.ts` (`PostgresOrderRepository`, queries manuais com `$1`, `RETURNING "id"`, `null` quando nao encontrado).

### Passo 5 -- Publisher de Infraestrutura (se aplicavel)

Crie `apps/billing/src/infrastructure/publisher/order-publisher.ts` (`OrderPublisher implements OrderEvent`, subject e data em `pkg/event/billing.ts`).

### Passo 6a -- Controller (se a API precisar)

Crie `apps/billing/src/infrastructure/controller/order-controller.ts` (estende `BaseController`, schemas zod no proprio arquivo, `registerRoutes`).

### Passo 6b -- Subscriber (se o Consumer precisar)

Crie `apps/billing/src/infrastructure/subscriber/order-subscriber.ts` (`handle<EventName>` com `ack`/`nak`/`term`).

### Passo 7 -- Migration SQL

Crie `migrations/billing/0002_create_orders.sql` (dbmate) seguindo as boas praticas de PostgreSQL 18:

```sql
-- migrate:up
CREATE TABLE "orders" (
    "id"         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId"     UUID        NOT NULL,
    "status"     TEXT        NOT NULL DEFAULT 'pending',
    "amount"     BIGINT      NOT NULL CHECK ("amount" > 0),
    "createdAt"  TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt"  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX "idx_orders_userId" ON "orders" ("userId");

-- migrate:down
DROP TABLE "orders";
```

Aplicar com `dbmate --migrations-dir migrations/billing up` (a `DATABASE_URL` de cada app aponta para seu proprio banco).

### Passo 8 -- Composicao awilix

**Para a API** -- atualize `apps/billing/src/bin/api.ts`:

```typescript
container.register({
  // ... existentes ...
  orderRepository: asClass(PostgresOrderRepository).singleton(),
  orderEvent: asClass(OrderPublisher).singleton(),
  createOrderUseCase: asClass(CreateUseCase).singleton(),
  orderController: asClass(OrderController).singleton(),
});
```

Registre as rotas no mesmo arquivo:

```typescript
container.resolve<OrderController>('orderController').registerRoutes(server);
```

**Para o Consumer** -- atualize `apps/billing/src/bin/consumer.ts` registrando o subscriber e o `subscribe(...)` do novo consumer duravel.

### Passo 9 -- Testes

- Unitario: `apps/billing/src/application/usecase/order/create-usecase.test.ts` (vitest, dependencias mockadas com `vi.fn()`).
- Integracao: `apps/billing/src/infrastructure/repository/order-repository.integration.test.ts` (testcontainers com PostgreSQL real).

### Passo 10 -- OpenAPI (apenas APIs)

Anote os handlers com JSDoc `@openapi` e execute `pnpm --filter @org/billing openapi` para regenerar a documentacao.

### Checklist -- Nova Feature

- [ ] `src/domain/entity/<name>.ts` -- Entidade (classe com `static create`)
- [ ] `src/domain/usecase/<context>/<action>.ts` -- 1 interface por use case (metodo `perform`)
- [ ] `src/domain/repository/<name>-repository.ts` -- Interface de repositorio
- [ ] `src/domain/event/<name>-event.ts` -- Interface de evento (se aplicavel)
- [ ] `src/domain/<lib>/<name>.ts` -- Interface para lib externa (se aplicavel)
- [ ] `src/application/usecase/<context>/<action>-usecase.ts` -- Implementacao do use case
- [ ] `src/application/usecase/<context>/<action>-usecase.test.ts` -- Teste unitario (vitest + `vi.fn`)
- [ ] `src/infrastructure/repository/<name>-repository.ts` -- Implementacao PostgreSQL (`pg.Pool`)
- [ ] `src/infrastructure/repository/<name>-repository.integration.test.ts` -- Teste de integracao (testcontainers)
- [ ] `src/infrastructure/publisher/<name>-publisher.ts` -- Implementacao NATS (se aplicavel)
- [ ] `src/infrastructure/adapter/<lib>-adapter.ts` -- Implementacao de lib externa (se aplicavel)
- [ ] `src/infrastructure/controller/<name>-controller.ts` -- Controller Hyper-Express + schemas zod (se API)
- [ ] `src/infrastructure/subscriber/<name>-subscriber.ts` -- Subscriber NATS (se Consumer)
- [ ] `migrations/<app>/NNNN_<descricao>.sql` -- Migration dbmate (`-- migrate:up` / `-- migrate:down`)
- [ ] `apps/<app>/src/bin/api.ts` -- Registrar no container e chamar `registerRoutes` (se API)
- [ ] `apps/<app>/src/bin/consumer.ts` -- Registrar subscriber e `subscribe(...)` (se Consumer)
- [ ] Anotar handlers com `@openapi` e regenerar o spec (se API)
- [ ] Rodar os gates: `pnpm typecheck`, `pnpm lint`, `pnpm test`

### Checklist -- Novo App

- [ ] Criar `apps/<app-name>/package.json` (`@org/<app-name>`, `"type": "module"`, `@org/pkg: workspace:*`)
- [ ] Criar `apps/<app-name>/tsconfig.json` estendendo `tsconfig.base.json`
- [ ] Confirmar que `pnpm-workspace.yaml` cobre o novo app (`apps/*`)
- [ ] Criar `apps/<app-name>/src/bin/api.ts` e/ou `src/bin/consumer.ts`
- [ ] Chamar `registerInfrastructure(container)` (config, logger, pool, natsConnection, publisher)
- [ ] Registrar apenas repositorios, publishers, adapters, use cases e handlers necessarios
- [ ] Definir startup (`createServer` + `registerRoutes` + `listen` para API; `ensureStream` + `subscribe` para Consumer)
- [ ] Registrar handlers de `SIGTERM`/`SIGINT` com shutdown graceful (`server.close()`/`messages.stop()` -> `container.dispose()`)
- [ ] Criar `migrations/<app-name>/` com migrations iniciais (dbmate)
- [ ] Criar `Dockerfile` (build com `tsc`, runtime `node dist/bin/<binary>.js`)
- [ ] Adicionar ao `docker-compose.yml`

