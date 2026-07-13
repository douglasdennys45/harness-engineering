---
name: node-testing-best-practices
description: Aplica melhores práticas de testes em Node.js + TypeScript cobrindo a pirâmide completa com vitest — unitários com describe/it, AAA e it.each, dublês com vi.fn()/vi.spyOn/fakes tipados injetados por constructor (awilix), fake timers, integração com testcontainers (PostgreSqlContainer, NATS) e E2E de API subindo o Hyper-Express em porta efêmera com fetch nativo. Use sempre que escrever, revisar ou refatorar arquivos *.test.ts, ao discutir estratégia de testes, mocks, fixtures, containers de teste, cobertura (v8) ou mutation testing (Stryker). Acione ao montar suites vitest, ao subir dependências reais via testcontainers ou ao configurar vitest.config.ts.
---

# Node.js Testing Best Practices (vitest · testcontainers · Stryker)

Skill baseada na documentação oficial:

- [vitest.dev](https://vitest.dev/) (`describe`/`it`, `vi`, coverage v8, workspace/projects)
- [node.testcontainers.org](https://node.testcontainers.org/) (`testcontainers`, `@testcontainers/postgresql`)
- [stryker-mutator.io](https://stryker-mutator.io/docs/stryker-js/) (mutation testing)

Toda recomendação aqui é a forma idiomática prescrita pelas fontes oficiais, adaptada à arquitetura do projeto (Clean Architecture, DI por constructor com awilix CLASSIC, 1 use case = 1 interface = método `perform`). Combine com [[nodejs-best-practices]] para convenções gerais da linguagem.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer `*.test.ts`
- Ao criar fakes/mocks/stubs ou estruturar suites
- Ao desenhar a estratégia de testes (unit, integração, E2E)
- Ao subir dependências reais (Postgres, NATS) via testcontainers
- Ao discutir cobertura, fixtures, fake timers ou mutation testing

## 0. A Pirâmide de Testes no Projeto

| Camada | Velocidade | Escopo | Ferramenta principal |
|--------|-----------|--------|----------------------|
| **Unitário** | ms | Use case/entidade isolados, sem I/O | vitest + fakes/`vi.fn()` |
| **Integração** | ~s | Repositório/publisher contra dependência **real** em container | vitest + `testcontainers` |
| **E2E de API** | s | HTTP → controller → use case → banco real, via `fetch` | vitest + Hyper-Express em porta efêmera |

Regras inegociáveis:
- Muitos unitários → poucos integração → mínimo de E2E.
- Cada camada acima é mais lenta e mais frágil. Suba o que **realmente** precisa subir.
- Toda camada usa **vitest** — não há runner separado (`node --test` não é usado). Diferencie por **convenção de arquivo + projeto no vitest.config.ts**, nunca por framework paralelo.

## 1. Convenções Gerais para `*.test.ts`

### Layout de arquivos

| Camada | Localização | Sufixo |
|--------|-------------|--------|
| Unitário | **junto do código** (`src/application/usecase/user/create-user-usecase.test.ts`) | `.test.ts` |
| Integração | `test/integration/` do app | `.integration.test.ts` |
| E2E de API | `test/integration/` do app | `.e2e.test.ts` |

- Unitário ao lado do arquivo testado — mesmo diretório, mesmo nome + `.test.ts`.
- Integração/E2E fora de `src/` — não são compilados no build (`include: ["src"]` no tsconfig de build; um `tsconfig.test.json` estende com `test/`).

### Separação por projetos no `vitest.config.ts`

```typescript
// vitest.config.ts (raiz do app)
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    projects: [
      {
        test: {
          name: 'unit',
          include: ['src/**/*.test.ts'],
          environment: 'node',
        },
      },
      {
        test: {
          name: 'integration',
          include: ['test/integration/**/*.integration.test.ts', 'test/integration/**/*.e2e.test.ts'],
          environment: 'node',
          // containers demoram para subir
          testTimeout: 60_000,
          hookTimeout: 120_000,
          // um container por suite: sem paralelismo entre arquivos
          fileParallelism: false,
        },
      },
    ],
    coverage: {
      provider: 'v8',
      include: ['src/**'],
      exclude: ['src/bin/**', 'src/**/*.test.ts'],
      reporter: ['text', 'lcov'],
    },
  },
});
```

Rode assim:

```bash
pnpm vitest --project unit                 # unitários (watch em dev)
pnpm vitest run --project unit             # unitários (CI)
pnpm vitest run --project integration      # integração + E2E (containers)
pnpm vitest run --coverage                 # cobertura v8
```

### Estrutura `describe`/`it` + AAA

- `describe` nomeia a unidade sob teste (`CreateUserUseCase`); `describe` aninhado nomeia o método/cenário.
- `it` descreve **comportamento** em linguagem de domínio: `it('lança UserAlreadyExistsError quando o email já existe')`.
- Todo teste segue **AAA** (Arrange, Act, Assert) — separe os blocos com linha em branco, sem comentários redundantes.

```typescript
import { describe, expect, it } from 'vitest';

describe('CreateUserUseCase', () => {
  describe('perform', () => {
    it('cria o usuário e publica o evento quando o email é inédito', async () => {
      // Arrange
      const repository = new FakeUserRepository();
      const publisher = new FakeUserEventPublisher();
      const useCase = new CreateUserUseCase({ userRepository: repository, userEvent: publisher });

      // Act
      const output = await useCase.perform({ name: 'Ana', email: 'ana@example.com' });

      // Assert
      expect(output.user.email).toBe('ana@example.com');
      expect(await repository.findByEmail('ana@example.com')).not.toBeNull();
      expect(publisher.published).toHaveLength(1);
    });
  });
});
```

### Asserções essenciais

| Cenário | Use |
|---------|-----|
| Igualdade estrutural | `toEqual` (deep), `toStrictEqual` (sem props `undefined` extras) |
| Igualdade referencial/primitiva | `toBe` |
| Null-safety do projeto | `toBeNull` / `not.toBeNull` — "não encontrado" é `null` |
| Erro assíncrono | `await expect(promise).rejects.toThrow(UserNotFoundError)` |
| Erro síncrono | `expect(() => fn()).toThrow(...)` |
| Coleções | `toHaveLength`, `toContainEqual`, `arrayContaining` |
| Parcial | `toMatchObject`, `objectContaining` |
| Chamadas de mock | `toHaveBeenCalledWith`, `toHaveBeenCalledTimes`, `toHaveBeenCalledExactlyOnceWith` |

Regras:
- `toEqual` para objetos, `toBe` para primitivos/referências. Nunca `expect(a === b).toBe(true)` — perde diagnóstico.
- Erro esperado **sempre** via `rejects.toThrow(ClasseDeErro)` — asserta o tipo, não só a mensagem.
- Não faça try/catch manual para assertar erro; se precisar inspecionar o erro, use `rejects.toThrowError(expect.objectContaining({ code: 'USER_NOT_FOUND' }))`.

### Table-driven tests com `it.each`

Sempre que houver ≥ 3 casos da mesma regra:

```typescript
describe('Email', () => {
  it.each([
    { input: 'ana@example.com', valid: true },
    { input: 'sem-arroba.com', valid: false },
    { input: '', valid: false },
    { input: 'a@b.co', valid: true },
  ])('valida $input como $valid', ({ input, valid }) => {
    expect(Email.isValid(input)).toBe(valid);
  });
});
```

- Prefira a forma de **objetos nomeados** (`$input` no título) à forma de arrays — auto-documenta.
- Cada linha deve ser **independente** — nenhum caso depende do estado deixado por outro.
- Para casos que esperam erro, inclua o erro esperado na tabela (`expectedError: UserNotFoundError`), não duplique o teste.

### `beforeEach`, hooks e isolamento

- Estado novo por teste: crie fakes/SUT em `beforeEach` ou em factory local (`makeSut()`), nunca em variável de módulo compartilhada mutável.
- `afterEach(() => { vi.restoreAllMocks(); vi.useRealTimers(); })` quando a suite usa spies/fake timers.
- Padrão factory (evita hooks encadeados e deixa o Arrange explícito):

```typescript
function makeSut() {
  const userRepository = new FakeUserRepository();
  const userEvent = new FakeUserEventPublisher();
  const sut = new CreateUserUseCase({ userRepository, userEvent });
  return { sut, userRepository, userEvent };
}
```

## 2. Dublês de Teste: DI por Constructor > `vi.mock`

O projeto usa **awilix em modo CLASSIC** — toda dependência entra pelo constructor como interface de domínio. Isso torna `vi.mock` (mock de módulo) **desnecessário e indesejado** na maior parte dos casos:

| Técnica | Quando usar |
|---------|-------------|
| **Fake tipado** (implementa a interface de domínio) | Padrão para use cases — comportamento realista in-memory |
| **`vi.fn()`** dentro de um objeto tipado | Verificar interação pontual (foi chamado? com quê?) |
| **`vi.spyOn`** | Observar/substituir método de um objeto real existente |
| **`vi.mock` (módulo)** | **Último recurso** — apenas para módulo sem injeção possível (ex.: top-level de lib) |

Por quê preferir DI a `vi.mock`:
- `vi.mock` é hoisted, acopla o teste ao **caminho do arquivo** e quebra silenciosamente em refactors de import.
- Fake injetado é verificado pelo compilador contra a interface de domínio — mock de módulo não.
- O teste documenta o contrato: se a interface muda, o fake não compila.

### Fake tipado — padrão do projeto

Implemente a interface de domínio com estado in-memory. Fakes vivem em `test/fakes/` (ou junto do teste se usados por um só arquivo):

```typescript
// test/fakes/fake-user-repository.ts
import type { UserRepository } from '../../src/domain/repository/user-repository.js';
import type { User } from '../../src/domain/entity/user.js';

export class FakeUserRepository implements UserRepository {
  private readonly users = new Map<string, User>();

  async create(user: User): Promise<void> {
    this.users.set(user.id, user);
  }

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return [...this.users.values()].find((u) => u.email === email) ?? null;
  }
}
```

- O fake **respeita o contrato de null-safety**: não encontrado → `null`, nunca throw.
- `implements UserRepository` é obrigatório — sem isso o compilador não protege o fake de divergir.

### Mock tipado com `vi.fn()` — verificação de interação

Quando o que importa é "chamou X com Y", monte um objeto tipado com `vi.fn()`:

```typescript
import { vi } from 'vitest';
import type { Mocked } from 'vitest';
import type { UserRepository } from '../../src/domain/repository/user-repository.js';

function makeUserRepositoryMock(): Mocked<UserRepository> {
  return {
    create: vi.fn(),
    findById: vi.fn().mockResolvedValue(null),
    findByEmail: vi.fn().mockResolvedValue(null),
  };
}

it('não cria usuário quando o email já existe', async () => {
  const repository = makeUserRepositoryMock();
  repository.findByEmail.mockResolvedValue(existingUser);
  const sut = new CreateUserUseCase({ userRepository: repository, userEvent: makeUserEventMock() });

  await expect(sut.perform({ name: 'Ana', email: 'ana@example.com' }))
    .rejects.toThrow(UserAlreadyExistsError);

  expect(repository.create).not.toHaveBeenCalled();
});
```

Regras:
- Tipar como `Mocked<Interface>` — mantém autocompletar e checagem de assinatura em `mockResolvedValue`.
- `mockResolvedValue`/`mockRejectedValue` para promessas; nunca `mockReturnValue(Promise.resolve(...))`.
- Verifique a interação **relevante** (`create` não chamado), não todas — teste comportamento, não implementação.

### `vi.spyOn` — observar sem substituir tudo

```typescript
const logger = makeLogger();
const warnSpy = vi.spyOn(logger, 'warn');

// ... act ...

expect(warnSpy).toHaveBeenCalledWith(expect.objectContaining({ code: 'USER_NOT_FOUND' }), expect.any(String));
```

Sempre restaure: `afterEach(() => vi.restoreAllMocks())`.

### O que **não** fazer com dublês

- Não use `vi.mock('pg')` ou `vi.mock('nats')` para testar repositório/publisher — isso testa a sua imaginação da lib, não o comportamento. Repositório se testa com **container real** (seção 4).
- Não asserte número de chamadas internas irrelevantes ("findByEmail chamado 1x") — asserte o resultado observável.
- Não compartilhe mocks entre testes via variável de módulo sem `beforeEach` que os recrie/resete.
- Não misture fake e mock da mesma dependência no mesmo teste.

## 3. Fake Timers e Datas

Use `vi.useFakeTimers()` para lógica com `setTimeout`/`setInterval`/`Date`:

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';

describe('RetryPolicy', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('reagenda com backoff exponencial', async () => {
    const task = vi.fn().mockRejectedValueOnce(new Error('boom')).mockResolvedValueOnce('ok');
    const promise = retryWithBackoff(task, { baseMs: 1_000 });

    await vi.advanceTimersByTimeAsync(1_000);

    await expect(promise).resolves.toBe('ok');
    expect(task).toHaveBeenCalledTimes(2);
  });

  it('usa a data congelada em createdAt', () => {
    vi.setSystemTime(new Date('2026-01-01T00:00:00Z'));

    const user = User.create('Ana', 'ana@example.com');

    expect(user.createdAt.toISOString()).toBe('2026-01-01T00:00:00.000Z');
  });
});
```

Regras:
- Em código async, use as variantes **async**: `advanceTimersByTimeAsync`, `runAllTimersAsync` — as síncronas não drenam microtasks e o teste pendura.
- **Sempre** `vi.useRealTimers()` em `afterEach` — fake timer vazando quebra testes vizinhos (inclusive os de container, que dependem de timeout real).
- Nunca `await sleep(...)` real para "esperar a lógica rodar" em unitário — é flake e lentidão; controle o relógio.
- Alternativa de arquitetura: injete um `Clock` (interface de domínio) e use `FixedClock` no teste — preferível quando a entidade precisa de data determinística.

## 4. Testes de Use Case (Unitários)

Padrão do projeto: use case recebe interfaces de domínio pelo constructor (awilix CLASSIC → objeto de dependências nomeadas), fakes/mocks entram direto — **o container awilix não participa de teste unitário**.

```typescript
// src/application/usecase/user/create-user-usecase.test.ts
import { describe, expect, it } from 'vitest';

import { CreateUserUseCase } from './create-user-usecase.js';
import { UserAlreadyExistsError } from '../../../domain/error/user-errors.js';
import { FakeUserRepository } from '../../../../test/fakes/fake-user-repository.js';
import { FakeUserEventPublisher } from '../../../../test/fakes/fake-user-event-publisher.js';

function makeSut() {
  const userRepository = new FakeUserRepository();
  const userEvent = new FakeUserEventPublisher();
  const sut = new CreateUserUseCase({ userRepository, userEvent });
  return { sut, userRepository, userEvent };
}

describe('CreateUserUseCase', () => {
  it('persiste o usuário e retorna o output com id gerado', async () => {
    const { sut, userRepository } = makeSut();

    const output = await sut.perform({ name: 'Ana', email: 'ana@example.com' });

    expect(output.user.id).toBeTruthy();
    expect(await userRepository.findById(output.user.id)).toEqual(output.user);
  });

  it('lança UserAlreadyExistsError quando o email já está em uso', async () => {
    const { sut } = makeSut();
    await sut.perform({ name: 'Ana', email: 'ana@example.com' });

    await expect(sut.perform({ name: 'Bia', email: 'ana@example.com' }))
      .rejects.toThrow(UserAlreadyExistsError);
  });

  it('publica o evento apenas após persistir', async () => {
    const { sut, userEvent } = makeSut();

    const output = await sut.perform({ name: 'Ana', email: 'ana@example.com' });

    expect(userEvent.published).toEqual([
      expect.objectContaining({ userId: output.user.id }),
    ]);
  });
});
```

Regras:
- Nomeie cenários em linguagem de domínio: `<ação> quando <condição>` / `lança <Erro> quando <condição>`.
- Um comportamento por `it`. Múltiplas asserções sobre o **mesmo** resultado são OK; múltiplos Acts não.
- Teste também os caminhos de erro de infra: injete mock com `mockRejectedValue(new Error('db down'))` e asserte que o use case propaga com `cause` ou converte no erro de domínio correto.
- Não instancie o container awilix em unitário — se o teste precisa do container, ele é de integração.

## 5. Testes de Integração com `testcontainers`

Referência oficial: https://node.testcontainers.org/

Sobem **dependências reais em containers** (PostgreSQL 18, NATS 2.x) para testar repositórios, publishers e subscribers contra o engine de verdade.

### Quando integração vale o custo

- Validar SQL real (constraints, índice único, RETURNING, cursor pagination).
- Validar migrations contra o engine real.
- Verificar publish/consume real no JetStream (ack, dedup por `Nats-Msg-Id`).

Não substitui unitários — é a camada do meio, não a base.

### Instalação

```bash
pnpm add -D testcontainers @testcontainers/postgresql
```

### Helper compartilhado por suite — padrão do projeto

Um container por **suite/arquivo** via `beforeAll`/`afterAll` (com `fileParallelism: false` no projeto integration, containers não se acumulam). Helpers vivem em `test/integration/helper/`:

```typescript
// test/integration/helper/postgres.ts
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';

import { runMigrations } from '../../../src/infrastructure/database/migrate.js';

export interface PostgresTestContext {
  container: StartedPostgreSqlContainer;
  pool: Pool;
}

export async function startPostgres(): Promise<PostgresTestContext> {
  const container = await new PostgreSqlContainer('postgres:18-alpine')
    .withDatabase('app_test')
    .withUsername('test')
    .withPassword('test')
    .start();

  const pool = new Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);

  return { container, pool };
}

export async function stopPostgres(ctx: PostgresTestContext): Promise<void> {
  await ctx.pool.end();
  await ctx.container.stop();
}
```

```typescript
// test/integration/helper/nats.ts
import { GenericContainer, Wait, type StartedTestContainer } from 'testcontainers';
import { connect, type NatsConnection } from 'nats';

export interface NatsTestContext {
  container: StartedTestContainer;
  connection: NatsConnection;
}

export async function startNats(): Promise<NatsTestContext> {
  const container = await new GenericContainer('nats:2.12-alpine')
    .withCommand(['-js']) // JetStream habilitado
    .withExposedPorts(4222)
    .withWaitStrategy(Wait.forLogMessage(/Server is ready/))
    .withStartupTimeout(30_000)
    .start();

  const connection = await connect({
    servers: `nats://${container.getHost()}:${container.getMappedPort(4222)}`,
  });

  return { container, connection };
}

export async function stopNats(ctx: NatsTestContext): Promise<void> {
  await ctx.connection.drain();
  await ctx.container.stop();
}
```

### Teste de repositório — padrão

```typescript
// test/integration/postgres-user-repository.integration.test.ts
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';

import { PostgresUserRepository } from '../../src/infrastructure/repository/postgres-user-repository.js';
import { User } from '../../src/domain/entity/user.js';
import { startPostgres, stopPostgres, type PostgresTestContext } from './helper/postgres.js';

describe('PostgresUserRepository', () => {
  let ctx: PostgresTestContext;
  let repository: PostgresUserRepository;

  beforeAll(async () => {
    ctx = await startPostgres();
    repository = new PostgresUserRepository({ pool: ctx.pool });
  });

  afterAll(async () => {
    await stopPostgres(ctx);
  });

  beforeEach(async () => {
    // isolamento entre testes: truncar, nunca recriar o container
    await ctx.pool.query('TRUNCATE TABLE "users" CASCADE');
  });

  it('persiste e recupera o usuário por id', async () => {
    const user = User.create('Ana', 'ana@example.com');
    await repository.create(user);

    const found = await repository.findById(user.id);

    expect(found).toEqual(user);
  });

  it('retorna null quando o id não existe', async () => {
    const found = await repository.findById('00000000-0000-0000-0000-000000000000');

    expect(found).toBeNull();
  });

  it('rejeita email duplicado pelo índice único', async () => {
    await repository.create(User.create('Ana', 'ana@example.com'));

    await expect(repository.create(User.create('Bia', 'ana@example.com')))
      .rejects.toThrow();
  });
});
```

### Wait strategies — escolha pela natureza da dependência

| Estratégia | Use quando |
|-----------|-----------|
| `Wait.forLogMessage(re)` | Serviço imprime linha de "ready" (NATS: `/Server is ready/`) |
| `Wait.forListeningPorts()` | Basta a porta abrir (TCP simples) |
| `Wait.forHttp('/health', port)` | Serviço expõe healthcheck HTTP |
| `Wait.forHealthCheck()` | Imagem define HEALTHCHECK no Dockerfile |
| `Wait.forAll([...])` | Combinação |

- `PostgreSqlContainer` do módulo oficial **já embute** a wait strategy correta — não a redefina.
- Em `GenericContainer`, **sempre** defina wait strategy explícita + `withStartupTimeout(...)` — sem isso o teste pendura ou flakeia em CI lento.

### Regras críticas de containers

- **Nunca** hardcode porta do host: use `container.getMappedPort(4222)` / `getConnectionUri()`.
- **Sempre** feche clientes **antes** de parar o container (`pool.end()` → `container.stop()`; `connection.drain()` → `container.stop()`) — ordem inversa vaza sockets e pendura o vitest.
- Isolamento entre testes por `TRUNCATE`/stream purge em `beforeEach`, **não** por container novo por teste (custo proibitivo).
- `hookTimeout` generoso (120s) no projeto integration — primeira execução baixa a imagem.
- Prefira o **módulo oficial** (`@testcontainers/postgresql`) a `GenericContainer` quando existir.
- Se a suite pendurar ao final, procure handle vazado: pool aberto, connection sem drain, `setInterval` sem clear.

### Teste de publisher/consumer JetStream

```typescript
// test/integration/nats-user-publisher.integration.test.ts
import { afterAll, beforeAll, describe, expect, it } from 'vitest';

import { NatsUserPublisher } from '../../src/infrastructure/publisher/nats-user-publisher.js';
import { startNats, stopNats, type NatsTestContext } from './helper/nats.js';

describe('NatsUserPublisher', () => {
  let ctx: NatsTestContext;
  let publisher: NatsUserPublisher;

  beforeAll(async () => {
    ctx = await startNats();
    const jsm = await ctx.connection.jetstreamManager();
    await jsm.streams.add({ name: 'EVENTS_USER', subjects: ['events.user.>'] });
    publisher = new NatsUserPublisher({ jetstream: ctx.connection.jetstream() });
  });

  afterAll(async () => {
    await stopNats(ctx);
  });

  it('publica o evento com Nats-Msg-Id para deduplicação', async () => {
    const user = User.create('Ana', 'ana@example.com');

    await publisher.publishCreated(user);

    const js = ctx.connection.jetstream();
    const consumer = await js.consumers.get('EVENTS_USER');
    const msg = await consumer.next();
    expect(msg?.subject).toBe('events.user.created');
    expect(msg?.headers?.get('Nats-Msg-Id')).toBe(user.id);
  });
});
```

## 6. Testes E2E de API (Hyper-Express + fetch nativo)

Sobem o servidor real em **porta efêmera**, com container de Postgres/NATS por trás, e exercitam a API por HTTP com o `fetch` nativo do Node.

### Padrão: `buildServer` + porta efêmera

O bootstrap do app expõe uma factory que monta o container awilix e o servidor **sem** dar listen (o `src/bin/api.ts` chama a factory e faz listen na porta de config):

```typescript
// test/integration/user-api.e2e.test.ts
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';

import { buildServer } from '../../src/infrastructure/http/build-server.js';
import { startPostgres, stopPostgres, type PostgresTestContext } from './helper/postgres.js';
import { startNats, stopNats, type NatsTestContext } from './helper/nats.js';

describe('POST /v1/users', () => {
  let pg: PostgresTestContext;
  let nats: NatsTestContext;
  let server: Awaited<ReturnType<typeof buildServer>>;
  let baseUrl: string;

  beforeAll(async () => {
    [pg, nats] = await Promise.all([startPostgres(), startNats()]);

    server = await buildServer({
      databaseUrl: pg.container.getConnectionUri(),
      natsUrl: `nats://${nats.container.getHost()}:${nats.container.getMappedPort(4222)}`,
    });

    await server.listen(0); // porta efêmera — nunca porta fixa
    baseUrl = `http://127.0.0.1:${server.port}`;
  });

  afterAll(async () => {
    await server.close();
    await Promise.all([stopPostgres(pg), stopNats(nats)]);
  });

  beforeEach(async () => {
    await pg.pool.query('TRUNCATE TABLE "users" CASCADE');
  });

  it('retorna 201 com o usuário criado', async () => {
    const res = await fetch(`${baseUrl}/v1/users`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name: 'Ana', email: 'ana@example.com' }),
    });

    expect(res.status).toBe(201);
    const body = await res.json();
    expect(body).toMatchObject({ name: 'Ana', email: 'ana@example.com' });
    expect(body.id).toBeTruthy();
  });

  it('retorna 409 quando o email já existe', async () => {
    await fetch(`${baseUrl}/v1/users`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name: 'Ana', email: 'ana@example.com' }),
    });

    const res = await fetch(`${baseUrl}/v1/users`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name: 'Bia', email: 'ana@example.com' }),
    });

    expect(res.status).toBe(409);
    await expect(res.json()).resolves.toMatchObject({ error: expect.any(String), status: 409 });
  });

  it('retorna 400 quando o body falha na validação zod', async () => {
    const res = await fetch(`${baseUrl}/v1/users`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name: 'A' }),
    });

    expect(res.status).toBe(400);
  });
});
```

Regras:
- **Porta efêmera sempre** (`listen(0)`) — porta fixa colide em CI e entre suites.
- `fetch` nativo do Node 22 — não adicione supertest/axios para E2E.
- E2E cobre o **contrato HTTP**: status codes, shape do JSON (camelCase), mapeamento erro de domínio → status. Regra de negócio fina fica no unitário.
- Teste os 3 tipos de resposta de cada endpoint crítico: sucesso, erro de domínio mapeado (404/409/422), erro de validação (400).
- Sempre `server.close()` antes de derrubar containers.

## 7. Cobertura e Mutation Testing

### Cobertura com v8

```bash
pnpm vitest run --coverage
```

- Provider `v8` (nativo, sem instrumentação de build) — já configurado na seção 1.
- Meça cobertura **apenas dos unitários + integração de `src/`**; exclua `src/bin/` (composição) e os próprios testes.
- Cobertura é **detector de buraco**, não meta de vaidade: 100% de linha com asserções fracas é pior que 85% com asserções fortes — por isso o projeto usa mutation testing.

### Mutation testing com Stryker

```bash
pnpm add -D @stryker-mutator/core @stryker-mutator/vitest-runner
```

```jsonc
// stryker.config.json
{
  "$schema": "https://raw.githubusercontent.com/stryker-mutator/stryker-js/master/packages/core/schema/stryker-schema.json",
  "testRunner": "vitest",
  "vitest": { "configFile": "vitest.config.ts", "related": true },
  "mutate": [
    "src/domain/**/*.ts",
    "src/application/**/*.ts",
    "!src/**/*.test.ts"
  ],
  "thresholds": { "high": 85, "low": 70, "break": 60 },
  "reporters": ["clear-text", "html", "progress"],
  "incremental": true
}
```

```bash
pnpm stryker run
```

Regras:
- Mute **domínio e aplicação** (onde vive a regra de negócio); não mute `infrastructure/` nem `bin/` — mutantes de SQL/wiring geram ruído e o custo é alto.
- Roda contra o projeto **unit** do vitest — mutation testing sobre containers é inviável.
- `incremental: true` em dev; CI noturno/semanal roda completo com `break` como gate.
- Mutante sobrevivente = teste fraco: fortaleça a **asserção** (valores exatos, erro exato), não escreva teste que apenas executa a linha.

## 8. Convenções Transversais

### Estrutura de pastas recomendada (por app)

```
apps/user/
├── src/
│   ├── domain/
│   ├── application/
│   │   └── usecase/user/
│   │       ├── create-user-usecase.ts
│   │       └── create-user-usecase.test.ts        # unitário, junto do código
│   ├── infrastructure/
│   └── bin/
├── test/
│   ├── fakes/
│   │   ├── fake-user-repository.ts
│   │   └── fake-user-event-publisher.ts
│   └── integration/
│       ├── helper/
│       │   ├── postgres.ts                        # container PG compartilhado
│       │   └── nats.ts                            # container NATS compartilhado
│       ├── postgres-user-repository.integration.test.ts
│       └── user-api.e2e.test.ts                   # E2E HTTP
├── vitest.config.ts
├── stryker.config.json
└── tsconfig.json
```

### Builders de fixture

Para entidades com muitos campos, use builder com defaults válidos e override pontual:

```typescript
// test/builders/user-builder.ts
export function makeUser(overrides: Partial<UserProps> = {}): User {
  return User.restore({
    id: randomUUID(),
    name: 'Ana',
    email: `${randomUUID()}@example.com`,
    createdAt: new Date('2026-01-01T00:00:00Z'),
    updatedAt: new Date('2026-01-01T00:00:00Z'),
    ...overrides,
  });
}
```

- Defaults sempre **válidos**; o teste sobrescreve apenas o que é relevante para o cenário.
- Email/ID aleatórios por default — evita colisão de índice único em integração.

### Não teste a implementação, teste o comportamento

- ❌ "verifica que `findByEmail` foi chamado 1x antes de `create`"
- ✅ "depois de `perform`, `findByEmail` retorna o usuário criado"

Mocks de interação são valiosos onde I/O custaria; se a regra cabe em fake in-memory, prefira — o teste sobrevive a refactors.

### Determinismo

- Nada de `Math.random()`/`Date.now()` cru no SUT sem seam (fake timers, `Clock` injetado ou builder).
- Nenhum teste depende de ordem de execução — vitest embaralha por arquivo; `beforeEach` garante estado limpo.
- Nenhum `sleep` real para sincronizar: unitário usa fake timers; integração usa `expect.poll(...)`/consumo explícito da mensagem.

```typescript
// espera por condição assíncrona em integração — sem sleep fixo
await expect.poll(() => repository.findByEmail('ana@example.com'), { timeout: 5_000 })
  .not.toBeNull();
```

## 9. Checklist de Revisão

Ao escrever ou revisar testes, verifique:

### Geral
- [ ] Unitário `.test.ts` junto do código; integração/E2E em `test/integration/` com `.integration.test.ts`/`.e2e.test.ts`
- [ ] `vitest.config.ts` com projetos `unit` e `integration` separados
- [ ] AAA com blocos separados; um comportamento por `it`
- [ ] Nomes de teste em linguagem de domínio (`lança X quando Y`)
- [ ] `it.each` com objetos nomeados quando houver ≥ 3 casos
- [ ] Nenhum teste depende de ordem/estado de outro

### Dublês
- [ ] Fake/mock **injetado por constructor** — `vi.mock` só como último recurso
- [ ] Fakes com `implements <Interface>` e contrato de `null` para não encontrado
- [ ] Mocks tipados como `Mocked<Interface>`; `mockResolvedValue`/`mockRejectedValue`
- [ ] `vi.restoreAllMocks()` em `afterEach` quando há spies
- [ ] Asserção de interação apenas quando a interação **é** o comportamento

### Timers
- [ ] `vi.useFakeTimers()` + `vi.useRealTimers()` pareados
- [ ] Variantes async (`advanceTimersByTimeAsync`) em código async
- [ ] `vi.setSystemTime` ou `Clock` injetado para datas determinísticas
- [ ] Nenhum sleep real em unitário

### Integração (testcontainers)
- [ ] Container compartilhado por suite (`beforeAll`/`afterAll`), isolamento por `TRUNCATE` em `beforeEach`
- [ ] Módulo oficial (`@testcontainers/postgresql`) quando existe
- [ ] `getMappedPort`/`getConnectionUri` — nunca porta hardcoded
- [ ] Wait strategy explícita + `withStartupTimeout` em `GenericContainer`
- [ ] Clientes fechados antes do `container.stop()` (pool.end, connection.drain)
- [ ] `fileParallelism: false` e timeouts generosos no projeto integration

### E2E
- [ ] Server em porta efêmera (`listen(0)`), URL montada com a porta real
- [ ] `fetch` nativo — sem lib HTTP extra
- [ ] Cobre sucesso, erro de domínio mapeado e erro de validação por endpoint crítico
- [ ] `server.close()` antes de derrubar containers

### Cobertura e mutação
- [ ] Coverage v8 excluindo `src/bin/` e testes
- [ ] Stryker mutando apenas `domain/` e `application/`, rodando o projeto unit
- [ ] Mutante sobrevivente tratado fortalecendo asserção, não inflando cobertura

## 10. Fontes Oficiais

- https://vitest.dev/guide/
- https://vitest.dev/api/vi.html
- https://vitest.dev/guide/mocking.html
- https://vitest.dev/guide/coverage.html
- https://node.testcontainers.org/
- https://node.testcontainers.org/modules/postgresql/
- https://stryker-mutator.io/docs/stryker-js/introduction/
- https://github.com/nats-io/nats.js

Combine sempre com [[nodejs-best-practices]] para convenções gerais da linguagem (imports ESM, erros com cause, async/await, null-safety).
