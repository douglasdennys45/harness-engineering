---
name: nodejs-best-practices
description: Aplica boas práticas modernas de Node.js 22 + TypeScript 5 strict em ESM ao escrever, revisar ou refatorar código TypeScript. Use ao criar/editar arquivos .ts, ao revisar código Node.js, ao discutir tsconfig strict (noUncheckedIndexedAccess, exactOptionalPropertyTypes, verbatimModuleSyntax), imports ESM com extensão .js, tratamento de erros (Error com cause, classes de erro de domínio), async/await (promises soltas, Promise.all, AbortSignal, timeouts), tipagem (unknown vs any, discriminated unions, satisfies), null-safety, imutabilidade, event loop e streams. Acione sempre que o trabalho envolver Node.js/TypeScript.
---

# Node.js + TypeScript Best Practices (Node 22 · TS 5 strict · ESM)

Skill baseada na documentação oficial do Node.js (nodejs.org/docs), TypeScript Handbook (typescriptlang.org/docs) e nos guias oficiais de ESM. Todas as recomendações aqui devem ser seguidas ao escrever, revisar ou refatorar código TypeScript neste harness.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.ts`
- Ao discutir design, nomenclatura ou arquitetura de aplicações Node.js
- Ao tratar de async/await, cancelamento, erros ou tipagem em TypeScript
- Ao gerar exemplos de código TypeScript em respostas

## 1. tsconfig Strict — Configuração Canônica

O projeto usa **TypeScript 5 em modo strict máximo** com ESM nativo. Este é o `tsconfig.json` base de todo app e pacote:

```jsonc
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2023"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "declaration": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

### Flags críticas e o que elas mudam

| Flag | Efeito | Consequência prática |
|------|--------|---------------------|
| `strict` | Liga todo o pacote strict (`strictNullChecks`, `noImplicitAny`, ...) | `null`/`undefined` nunca passam despercebidos |
| `noUncheckedIndexedAccess` | `arr[i]` e `obj[key]` retornam `T \| undefined` | Todo acesso indexado exige narrowing |
| `exactOptionalPropertyTypes` | `prop?: string` **não** aceita `undefined` explícito | Diferencia "ausente" de "presente com undefined" |
| `verbatimModuleSyntax` | Imports de tipo exigem `import type` | Elisão de import previsível, ESM seguro |
| `noImplicitOverride` | Métodos que sobrescrevem exigem `override` | Refactor de classe base quebra em compile time |
| `noPropertyAccessFromIndexSignature` | Index signature exige `obj["key"]` | Distingue propriedade declarada de acesso dinâmico |

### `noUncheckedIndexedAccess` na prática

```typescript
const users: User[] = await repository.list();

// ❌ users[0] é User | undefined — não compila sem narrowing
const first: User = users[0];

// ✅ narrowing explícito
const first = users[0];
if (first === undefined) {
  throw new UserNotFoundError('lista vazia');
}
use(first);

// ✅ ou .at() com o mesmo contrato explícito
const last = users.at(-1) ?? null;
```

### `exactOptionalPropertyTypes` na prática

```typescript
interface UpdateUserInput {
  name?: string; // ausente OU string — nunca undefined explícito
}

// ❌ não compila: undefined explícito em propriedade opcional
const input: UpdateUserInput = { name: undefined };

// ✅ omita a chave quando não há valor
const input: UpdateUserInput = {};

// Se "presente porém vazio" for um estado legítimo, declare-o:
interface Patch { name?: string | undefined }
```

## 2. ESM e Imports

O projeto é **ESM nativo**: `"type": "module"` no `package.json`, compilado com `tsc` (sem bundler), `module: "NodeNext"`.

### Regra canônica: extensão `.js` em imports relativos

Com `NodeNext`, o especificador de import é o do **arquivo compilado**. Imports relativos **sempre** levam extensão `.js` — mesmo apontando para um `.ts`:

```typescript
// ✅ src/application/usecase/user/create-user-usecase.ts
import { User } from '../../../domain/entity/user.js';
import type { UserRepository } from '../../../domain/repository/user-repository.js';

// ❌ sem extensão — quebra em runtime ESM
import { User } from '../../../domain/entity/user';

// ❌ extensão .ts — não existe após compilação
import { User } from '../../../domain/entity/user.ts';
```

Esta escolha é **consistente em todo o monorepo** — nunca misture com bundler resolution.

### `import type` obrigatório (verbatimModuleSyntax)

```typescript
// ✅ tipo puro: import type
import type { UserRepository } from '../repository/user-repository.js';

// ✅ misto: separe valor de tipo
import { z } from 'zod';
import type { ZodType } from 'zod';

// ❌ importar interface sem `type` — erro com verbatimModuleSyntax
import { UserRepository } from '../repository/user-repository.js';
```

### Demais regras de módulo

- **Sem `require`/`module.exports`** — CommonJS não existe neste projeto.
- **Sem `default export`** em código de aplicação — named exports sempre (melhor refactor, melhor tree-shaking, sem renomeio silencioso).
- Módulos built-in com prefixo `node:`: `import { setTimeout } from 'node:timers/promises'`.
- `__dirname`/`__filename` não existem em ESM — use `import.meta.dirname` (Node ≥ 20.11) ou `new URL('.', import.meta.url)`.
- Imports organizados em 3 blocos separados por linha em branco: (1) `node:*`, (2) third-party, (3) projeto interno.

```typescript
import { randomUUID } from 'node:crypto';

import { z } from 'zod';

import { User } from '../../domain/entity/user.js';
import type { UserRepository } from '../../domain/repository/user-repository.js';
```

## 3. Tratamento de Erros

### Lance apenas `Error` (ou subclasses)

```typescript
// ❌ NUNCA — perde stack trace, quebra instanceof, quebra cause chain
throw 'user not found';
throw { code: 404 };

// ✅ sempre Error ou subclasse
throw new UserNotFoundError(id);
```

A regra `@typescript-eslint/only-throw-error` garante isso no lint.

### Classes de erro de domínio

Equivalente aos sentinel errors do Go. Cada app define seus erros em `src/domain/error/`:

```typescript
// src/domain/error/domain-error.ts
export abstract class DomainError extends Error {
  abstract readonly code: string;

  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
    this.name = this.constructor.name;
  }
}

// src/domain/error/user-errors.ts
import { DomainError } from './domain-error.js';

export class UserNotFoundError extends DomainError {
  readonly code = 'USER_NOT_FOUND';

  constructor(id: string) {
    super(`user ${id}: not found`);
  }
}

export class UserAlreadyExistsError extends DomainError {
  readonly code = 'USER_ALREADY_EXISTS';

  constructor(email: string) {
    super(`email ${email}: already in use`);
  }
}
```

Verificação com `instanceof` (o equivalente de `errors.Is`):

```typescript
if (err instanceof UserNotFoundError) {
  return reply(404, err.message);
}
```

### `cause` — a cadeia de erros do TypeScript (equivalente ao `%w`)

Ao adicionar contexto sem perder o erro original, use `ErrorOptions.cause`:

```typescript
try {
  await this.pool.query(sql, params);
} catch (err) {
  throw new Error('create user: insert failed', { cause: err });
}
```

Regras:
- **Sempre** propague `cause` ao re-lançar com contexto novo. Nunca `throw new Error(String(err))` — destrói stack e tipo.
- Erros de domínio embrulham erros de infra via `cause` quando a origem importa para diagnóstico.
- Mensagens começam com minúscula, sem ponto final, identificando a operação: `'create user: insert failed'`.

### `catch` recebe `unknown` — sempre estreite

Com `useUnknownInCatchVariables` (incluso em `strict`), a variável do catch é `unknown`:

```typescript
try {
  await useCase.perform(input);
} catch (err) {
  // ❌ err.message — não compila, err é unknown
  // ✅ narrowing primeiro
  if (err instanceof DomainError) {
    logger.warn({ code: err.code, err }, 'request rejected');
    return mapDomainError(err);
  }
  logger.error({ err }, 'request failed');
  throw err; // não engula o que não conhece
}
```

Helper para normalizar o desconhecido:

```typescript
export function toError(value: unknown): Error {
  return value instanceof Error ? value : new Error(String(value), { cause: value });
}
```

### Handlers globais — última linha de defesa, nunca fluxo de controle

No bootstrap (`src/bin/`), registre e **encerre** o processo — estado corrompido não se recupera:

```typescript
process.on('unhandledRejection', (reason) => {
  logger.fatal({ err: reason }, 'unhandled rejection');
  process.exit(1);
});

process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'uncaught exception');
  process.exit(1);
});
```

- `unhandledRejection` em produção é **bug**: alguma promise ficou sem `await`/`.catch`.
- Nunca use esses handlers para "continuar rodando" — eles existem para logar e morrer limpo.

## 4. Async/Await

### Nunca deixe promises soltas

Toda promise deve ser aguardada, retornada ou explicitamente tratada. A regra `@typescript-eslint/no-floating-promises` é obrigatória no lint.

```typescript
// ❌ floating promise — rejeição vira unhandledRejection
repository.save(user);

// ✅ aguarde
await repository.save(user);

// ✅ fire-and-forget INTENCIONAL: marque e trate a rejeição
void publisher.publishCreated(user).catch((err: unknown) => {
  logger.error({ err }, 'publish user created failed');
});
```

### `Promise.all` para paralelismo, `allSettled` quando falha parcial é aceitável

```typescript
// ❌ sequencial sem necessidade — 3x mais lento
const user = await userRepository.findById(id);
const orders = await orderRepository.listByUser(id);
const balance = await balanceRepository.getByUser(id);

// ✅ operações independentes em paralelo
const [user, orders, balance] = await Promise.all([
  userRepository.findById(id),
  orderRepository.listByUser(id),
  balanceRepository.getByUser(id),
]);
```

- `Promise.all` **rejeita rápido** na primeira falha — as demais continuam rodando ao fundo; garanta que não deixam efeito colateral órfão.
- `Promise.allSettled` quando você precisa do resultado de todas mesmo com falhas.
- `Promise.race` apenas para timeout/primeiro-vence — prefira `AbortSignal.timeout` (abaixo).

### `AbortSignal` — o `context.Context` do Node.js

Cancelamento e deadline propagam via `AbortSignal`, **último parâmetro opcional** das funções assíncronas de I/O:

```typescript
// Interface de domínio aceita signal opcional
export interface UserRepository {
  findById(id: string, signal?: AbortSignal): Promise<User | null>;
}

// Timeout pontual
const res = await fetch(url, { signal: AbortSignal.timeout(5_000) });

// Combinação de sinais (timeout + shutdown)
const signal = AbortSignal.any([requestSignal, AbortSignal.timeout(5_000)]);

// Cancelamento manual
const controller = new AbortController();
process.on('SIGTERM', () => controller.abort(new Error('shutdown')));
```

Regras:
- Funções que fazem I/O de longa duração **aceitam** `signal?: AbortSignal` e o repassam para baixo (fetch, pg, streams, `setTimeout` de `node:timers/promises`).
- Verifique `signal.throwIfAborted()` antes de trabalho caro em loops.
- Nunca engula `AbortError` silenciosamente — logue como cancelamento, não como falha.

### Timeouts com `node:timers/promises`

```typescript
import { setTimeout as sleep } from 'node:timers/promises';

await sleep(1_000, undefined, { signal }); // cancelável

// ❌ new Promise((r) => setTimeout(r, 1000)) — não cancelável, vaza timer
```

### `async` até o fim

- Não misture `.then()` com `await` — padronize `await`.
- Não use `async` em função que não tem `await` (regra `require-await`).
- `for...of` + `await` para processamento **sequencial** intencional; `Promise.all(items.map(...))` para paralelo. Nunca `forEach(async ...)` — as promises se perdem.

## 5. Tipagem

### `unknown` vs `any`

- **`any` é proibido** em código de aplicação (`@typescript-eslint/no-explicit-any`). `any` desliga o compilador de forma viral.
- **`unknown`** para dados de fronteira (body HTTP, mensagem NATS, JSON parseado) — força narrowing antes do uso. Combine com zod:

```typescript
const CreateUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
});
type CreateUserInput = z.infer<typeof CreateUserSchema>;

// fronteira: unknown entra, tipo validado sai
const input: CreateUserInput = CreateUserSchema.parse(body); // body: unknown
```

### `type` vs `interface`

| Use | Quando |
|-----|--------|
| `interface` | Contratos implementáveis: repositórios, use cases, eventos, adapters |
| `type` | Unions, tuples, mapped types, aliases de função, composições |

```typescript
// Contrato de domínio: interface (implementável por classe)
export interface CreateUserUseCase {
  perform(input: CreateUserInput): Promise<CreateUserOutput>;
}

// Union/estado: type
export type PaymentStatus = 'pending' | 'paid' | 'refused';
```

### Discriminated unions + narrowing exaustivo

Modele estados mutuamente exclusivos como union discriminada, nunca como campos opcionais coexistentes:

```typescript
type PaymentEvent =
  | { kind: 'authorized'; authorizationId: string }
  | { kind: 'refused'; reason: string }
  | { kind: 'expired' };

function handle(event: PaymentEvent): string {
  switch (event.kind) {
    case 'authorized':
      return event.authorizationId; // narrowed
    case 'refused':
      return event.reason;
    case 'expired':
      return 'expired';
    default: {
      // exaustividade verificada em compile time
      const _exhaustive: never = event;
      throw new Error(`unhandled event: ${JSON.stringify(_exhaustive)}`);
    }
  }
}
```

### `satisfies` — validar sem alargar nem perder literal

```typescript
const config = {
  port: 3000,
  logLevel: 'info',
} satisfies AppConfig;

config.logLevel; // tipo 'info' (literal preservado), e AppConfig validado
```

Use `satisfies` em objetos de configuração, mapas de rotas e registros do container — valida a forma mantendo inferência precisa. Nunca use `as AppConfig` para "fazer compilar": assertion esconde erro, `satisfies` revela.

### Type guards nomeados

```typescript
export function isDomainError(value: unknown): value is DomainError {
  return value instanceof DomainError;
}
```

## 6. Null-Safety: `null` vs `undefined`

Padronização do projeto (espelha o `nil, nil` do harness Go):

| Situação | Valor |
|----------|-------|
| Consulta que não encontrou (`findById`, `findByEmail`) | **retorna `null`** |
| Propriedade opcional ausente | `undefined` (omissão de chave) |
| Campo de banco nullable | `null` no tipo da entidade |

```typescript
export interface UserRepository {
  // "não encontrado" NÃO é erro — é null
  findById(id: string): Promise<User | null>;
}

// Caller decide se ausência é erro no SEU contexto:
const user = await this.userRepository.findById(input.id);
if (user === null) {
  throw new UserNotFoundError(input.id);
}
```

Regras:
- **Nunca** lance exceção para "não encontrado" dentro do repositório — ausência é dado, não falha.
- Use `??` (nunca `||`) para defaults — `||` engole `0`, `''` e `false`.
- Optional chaining com intenção: `user?.address?.city ?? null` só quando cada elo é legitimamente opcional.
- Comparação estrita: `=== null` / `=== undefined`. Nunca `== null` (mascarar a distinção que padronizamos).

## 7. Imutabilidade

```typescript
// readonly em propriedades de entidade que não mudam após criação
export class User {
  constructor(
    readonly id: string,
    readonly email: string,
    public name: string,
    readonly createdAt: Date,
  ) {}
}

// ReadonlyArray / readonly em contratos que não devem mutar
export interface ListUsersOutput {
  readonly users: readonly User[];
  readonly nextCursor: string | null;
}

// as const para tabelas fixas — literais preservados
export const SUBJECTS = {
  userCreated: 'events.user.created',
  userUpdated: 'events.user.updated',
} as const;
export type Subject = (typeof SUBJECTS)[keyof typeof SUBJECTS];
```

Regras:
- Prefira criar novo objeto (`{ ...user, name }`) a mutar parâmetro recebido.
- `Object.freeze` apenas em singletons de configuração — `readonly` já cobre o resto em compile time.
- Arrays expostos por interfaces de domínio são `readonly T[]`.

## 8. Organização de Módulos e Naming

### Arquivos e diretórios

- Arquivos e pastas em **kebab-case**: `create-user-usecase.ts`, `user-repository.ts`.
- 1 use case = 1 interface = 1 arquivo, método `perform(input)`.
- Camadas: `src/domain/` → `src/application/` → `src/infrastructure/` → `src/bin/`. Camada interna nunca importa externa.
- `pkg/` (workspace pnpm) nunca importa `apps/`.

### Identificadores

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Classes, interfaces, types, enums | `PascalCase` | `CreateUserUseCase`, `UserRepository` |
| Funções, métodos, variáveis | `camelCase` | `findById`, `nextCursor` |
| Constantes verdadeiras (módulo) | `SCREAMING_SNAKE_CASE` | `MAX_PAGE_SIZE` |
| Acrônimos | Como palavra | `userId`, `parseUrl`, `HttpServer` (não `parseURL`) |
| Booleanos | Prefixo `is`/`has`/`can` | `isActive`, `hasPendingOrders` |

- Sem prefixo `I` em interfaces (`UserRepository`, não `IUserRepository`). A implementação concreta ganha o qualificador: `PostgresUserRepository`.
- Sem sufixo `Impl`.
- Getters sem prefixo `get` quando são propriedades (`user.fullName`, accessor `get fullName()`).

## 9. Event Loop, CPU-Bound e Streams

### Não bloqueie o event loop

Node atende todas as conexões em uma thread. Qualquer trabalho síncrono longo trava **todas** as requests:

- **Proibido** em código de request: `fs.readFileSync`, `crypto.pbkdf2Sync`, `zlib.gzipSync`, `JSON.parse` de payloads gigantes, loops O(n²) sobre coleções grandes.
- Use as variantes assíncronas (`node:fs/promises`, `crypto.pbkdf2` callback→promisify, `zlib` streams).
- CPU-bound real (hash de senha em lote, compressão pesada, parsing de arquivos grandes) vai para **`worker_threads`** — de preferência com um pool (ex.: `piscina`):

```typescript
import { Worker } from 'node:worker_threads';
// dispatcher no processo principal; o trabalho pesado roda em workers/hash-worker.js
```

- Sinal de alerta em produção: monitore event loop delay com `perf_hooks.monitorEventLoopDelay()`.

### Streams e backpressure

Para payloads grandes (export CSV, proxy de arquivos, ingestão), **nunca** carregue tudo em memória:

```typescript
import { pipeline } from 'node:stream/promises';

// ✅ pipeline gerencia backpressure, erros e cleanup de TODOS os stages
await pipeline(
  createReadStream(inputPath),
  createGzip(),
  createWriteStream(outputPath),
  { signal },
);

// ❌ .pipe() manual — não propaga erro nem destrói streams upstream
```

- `pipeline` de `node:stream/promises` sempre — aceita `AbortSignal`, propaga erro, fecha tudo.
- Ao escrever manualmente, respeite o retorno de `write()`: `false` → aguarde `'drain'`.
- Web Streams (`ReadableStream`) interoperam via `stream.Readable.fromWeb/toWeb` — use nas fronteiras com `fetch`.

## 10. Ferramentas

| Ferramenta | Papel | Regra |
|-----------|-------|-------|
| `tsc --noEmit` | Typecheck no CI | Zero erros, sem `@ts-ignore` (use `@ts-expect-error` com justificativa quando inevitável) |
| ESLint (`typescript-eslint`, preset `strict-type-checked`) | Lint com type info | `no-floating-promises`, `no-explicit-any`, `only-throw-error`, `no-misused-promises` obrigatórias |
| Prettier | Formatação | Sem debate de estilo; roda no pre-commit e CI |
| `tsx` | Executar TS em dev (`tsx watch src/bin/api.ts`) | Apenas dev — produção roda `node dist/bin/api.js` compilado |
| vitest | Testes | Ver [[node-testing-best-practices]] — `node --test` não é usado neste projeto (vitest cobre unit+integração com um só runner) |
| pino | Logging JSON estruturado | Nunca `console.log` em código de aplicação |
| pnpm | Workspaces do monorepo | `pnpm -r build`, `pnpm --filter <app> test` |

## Checklist de Revisão

Ao escrever ou revisar código TypeScript, verifique:

- [ ] Imports relativos com extensão `.js`
- [ ] `import type` para tipos puros (verbatimModuleSyntax)
- [ ] Named exports — sem `export default`
- [ ] Built-ins com prefixo `node:`
- [ ] Nenhum `any`; fronteiras usam `unknown` + zod
- [ ] Acesso indexado com narrowing (`noUncheckedIndexedAccess`)
- [ ] `throw` apenas de `Error`/subclasse; erros de domínio como classes com `code`
- [ ] Re-throw com contexto usa `{ cause: err }`
- [ ] `catch (err)` estreita `unknown` antes de usar
- [ ] Nenhuma promise solta; fire-and-forget com `void` + `.catch`
- [ ] Operações independentes com `Promise.all`
- [ ] I/O de longa duração aceita `signal?: AbortSignal`
- [ ] Timeouts via `AbortSignal.timeout`, sleeps via `node:timers/promises`
- [ ] "Não encontrado" retorna `null`, nunca exceção do repositório
- [ ] Defaults com `??`, nunca `||`
- [ ] `readonly`/`as const` em dados que não mutam
- [ ] Discriminated unions com `default` exaustivo (`never`)
- [ ] `satisfies` em objetos de configuração, nunca `as` para calar o compilador
- [ ] Arquivos em kebab-case; interfaces sem prefixo `I`
- [ ] Nenhum `*Sync` em caminho de request; CPU-bound em worker_threads
- [ ] Streams via `pipeline` de `node:stream/promises`
- [ ] `console.log` ausente — pino em todo logging

## Fontes Oficiais

Toda recomendação acima vem destas fontes — consulte-as quando houver dúvida:

- https://nodejs.org/docs/latest-v22.x/api/
- https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop
- https://nodejs.org/api/esm.html
- https://www.typescriptlang.org/docs/handbook/
- https://www.typescriptlang.org/tsconfig/
- https://typescript-eslint.io/rules/
