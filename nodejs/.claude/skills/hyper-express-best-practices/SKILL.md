---
name: hyper-express-best-practices
description: Aplica melhores práticas oficiais do Hyper-Express (github.com/kartikk221/hyper-express) ao escrever, revisar ou refatorar APIs HTTP em Node.js + TypeScript. Cobre inicialização do `HyperExpress.Server` e opções do construtor (`max_body_length`, `max_body_buffer`, `trust_proxy`, `fast_buffers`, `fast_abort`, `auto_close`, SSL via `key_file_name`/`cert_file_name`), roteamento (métodos HTTP, path params `:param`, wildcards `*`, `Router` modular, montagem via `server.use('/v1/users', router)`, `route()` chainable, route options com `middlewares` e `max_body_length` por rota, ordem de registro antes do `listen()`), Request (`json()`, `text()`, `buffer()`, `urlencoded()`, `multipart()`, `path_parameters`/`params`, `query_parameters`/`query`, `headers`, `cookies`, `ip`, `proxy_ip`, streaming de body, gotcha de body assíncrono e sem chunked transfer), Response (`status()`, `header()`, `type()`, `json()`, `send()`, `stream()`, `redirect()`, `cookie()`, `atomic()`, guard de `completed` para não responder duas vezes, eventos `abort`/`close`), middlewares globais e por rota (assinatura `(request, response, next)` e variante async por Promise, `next(error)`), tratamento de erros centralizado (`server.set_error_handler`, `server.set_not_found_handler`, integração com o padrão ErrorMapper do projeto — erros de domínio → HTTP status), validação de input no boundary com zod (`schema.safeParse` + resposta 400 padronizada via `bindAndHandle`), graceful shutdown (`server.shutdown()` vs `server.close()`, drenagem, SIGTERM/SIGINT), diferenças e gotchas vs Express (uWebSockets.js, sem `request.body` síncrono, incompatibilidade com middlewares que dependem de `http.IncomingMessage`, sem supertest, sem HTTP/2, sem live reload de rotas), e testes subindo o server em porta efêmera com `fetch` nativo. Use sempre que criar/editar arquivos `.ts` que importam `hyper-express`, ao desenhar rotas, controllers, middlewares ou error handlers HTTP, ao validar input com zod no boundary HTTP, ao configurar graceful shutdown do servidor, ou ao revisar PRs que tocam a camada HTTP de um serviço Node.js.
---

# Hyper-Express Best Practices (Documentação Oficial)

Skill baseada exclusivamente na documentação oficial em https://github.com/kartikk221/hyper-express — README e `docs/` (Server, Router, Request, Response, Middlewares, Websocket). Todas as recomendações aqui devem ser seguidas ao escrever, revisar ou refatorar código que use o pacote `hyper-express`.

Hyper-Express é um servidor HTTP/WebSocket de alta performance construído sobre **uWebSockets.js** (binding C++). Ele oferece uma API *parcialmente* compatível com Express — mas **não é Express**: o transporte não é o `http` nativo do Node.js, e isso muda gotchas, middlewares compatíveis e estratégia de testes (ver §9).

Esta skill **complementa** [[typescript-best-practices]] (strict mode, ESM, tipos) e [[nodejs-testing-best-practices]] (vitest). Sempre carregue-as em conjunto ao trabalhar com a camada HTTP.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.ts` que importe `hyper-express`
- Ao desenhar rotas, controllers, middlewares ou error handlers
- Ao implementar parsing/validação de request com zod, ou tratamento centralizado de erros
- Ao configurar startup/shutdown do servidor HTTP
- Ao revisar PRs que tocam a camada HTTP de um serviço Node.js

## Pré-requisitos

- **Node.js 22 LTS** (Hyper-Express suporta as três últimas LTS)
- Pacote: `hyper-express` (traz `uWebSockets.js` como dependência)
- TypeScript 5 em modo `strict`, projeto ESM (`"type": "module"`)

```typescript
import HyperExpress from 'hyper-express';
```

## 1. Inicialização do `Server`

Referência: [docs/Server.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Server.md)

### Padrão idiomático

```typescript
// pkg/httpserver/server.ts
import HyperExpress from 'hyper-express';

export function createServer(): HyperExpress.Server {
  return new HyperExpress.Server({
    max_body_length: 4 * 1024 * 1024, // 4 MiB — default é ~250kb
    max_body_buffer: 16 * 1024,       // 16kb em memória antes de bufferizar
    trust_proxy: true,                // atrás de LB/CDN: habilita proxy_ip
    fast_buffers: false,              // true = Buffer.allocUnsafe (perf > segurança)
    fast_abort: false,                // true = fecha conexões ruins imediatamente
    auto_close: true,                 // fecha o server no exit do processo
  });
}
```

### Opções do construtor

| Opção | Tipo | Default | Uso |
|---|---|---|---|
| `max_body_length` | Number | `256000` (~250kb) | Tamanho máximo do body. Ajuste explicitamente para uploads |
| `max_body_buffer` | Number | `16384` (16kb) | Limite de buffering em memória antes de acumular |
| `trust_proxy` | Boolean | `false` | Confia em headers de proxy (`X-Forwarded-For`) — habilita `request.proxy_ip` |
| `fast_buffers` | Boolean | `false` | Usa `Buffer.allocUnsafe` para bodies. Só habilite se o body **nunca** vaza cru para fora do handler |
| `fast_abort` | Boolean | `false` | Encerra conexões malformadas imediatamente, sem resposta de erro |
| `auto_close` | Boolean | `true` | Fecha o server automaticamente no exit do processo |
| `key_file_name` / `cert_file_name` | String | — | Caminhos de chave/certificado para servidor SSL |
| `passphrase` / `dh_params_file_name` / `ssl_prefer_low_memory_usage` | — | — | Opções adicionais de SSL |
| `streaming` | Object | — | `{ readable: ReadableOptions, writable: WritableOptions }` para streams de request/response |

### Regras absolutas

- **Sempre configure `max_body_length`** explicitamente — o default (~250kb) rejeita uploads e payloads grandes silenciosamente com erro.
- **`trust_proxy: true`** quando atrás de load balancer; caso contrário `request.ip` retorna o IP do proxy.
- **Sempre configure `set_error_handler` e `set_not_found_handler`** (ver §6) — os defaults respondem `text/plain` genérico.
- **Não habilite `fast_buffers`** por padrão: `Buffer.allocUnsafe` pode expor memória antiga se um body incompleto vazar para logs/respostas.
- Em produção, **TLS termina no proxy/ingress** — só use as opções SSL do construtor quando o Node atende TLS diretamente.

### `listen()`

```typescript
const server = createServer();

// listen(port, host?) retorna Promise<uws_listen_socket>
await server.listen(8080, '0.0.0.0');
console.log(`listening on :${server.port}`);

// também aceita UNIX socket path: server.listen('/tmp/app.sock')
```

- `listen` retorna **Promise** — sempre `await` e trate rejeição (porta ocupada) no bootstrap.
- **Registre todas as rotas ANTES de `listen()`.** O uWebSockets.js compila as rotas na inicialização — rotas adicionadas depois não são garantidas (ver §9).
- Porta efêmera para testes: `listen(0)` e leia `server.port` (ver §10).

## 2. Roteamento

Referência: [docs/Router.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Router.md)

### Métodos HTTP

`Server` e `Router` expõem os mesmos métodos de registro:

```typescript
server.get('/users/:id', handler);
server.post('/users', handler);
server.put('/users/:id', handler);
server.patch('/users/:id', handler);
server.delete('/users/:id', handler);
server.head('/health', handler);
server.options('/users', handler);
server.any('/catch', handler);   // qualquer método (alias: all)
server.ws('/live', options, handler); // WebSocket (ver §9)
```

Assinaturas suportadas (overloads):

```typescript
router.get(pattern, handler);
router.get(pattern, options, handler);                    // options: { max_body_length, middlewares, streaming }
router.get(pattern, middleware1, middleware2, handler);   // middlewares por rota, variádico
router.get(pattern, options, middleware, handler);
```

> **Gotcha:** route options `middlewares` **não são suportados** em rotas `any()` — use a forma variádica ou middlewares globais.

### Path params e wildcards

| Sintaxe | Significado | Acesso |
|---|---|---|
| `:name` | Parâmetro nomeado | `request.path_parameters.name` (alias: `request.params`) |
| `*` | Wildcard (prefixo/catch-all) | rota `/files/*` casa `/files/a/b/c` |

```typescript
server.get('/users/:id/orders/:orderId', async (request, response) => {
  const { id, orderId } = request.path_parameters;
  // ...
});

server.get('/static/*', serveStatic);
```

### Ordem de registro importa

Declare **rotas específicas antes** de wildcards e padrões genéricos, e registre **tudo antes do `listen()`**:

```typescript
server.get('/users/me', currentUser);   // específica primeiro
server.get('/users/:id', userById);     // genérica depois
server.any('/*', notFound);             // catch-all por último (ou use set_not_found_handler)
```

### `Router`: composição modular

Cada contexto de domínio expõe seu próprio `Router`; o `main` monta tudo no `Server`:

```typescript
// apps/user/src/infrastructure/http/user-router.ts
import HyperExpress from 'hyper-express';

export function createUserRouter(controller: UserController): HyperExpress.Router {
  const router = new HyperExpress.Router();

  router.post('/', controller.create);
  router.get('/:id', controller.getById);
  router.get('/', controller.list);

  return router;
}
```

```typescript
// apps/user/src/bin/api.ts — composição
const v1 = new HyperExpress.Router();
v1.use('/users', createUserRouter(userController));
v1.use('/orders', createOrderRouter(orderController));

server.use('/v1', v1); // → POST /v1/users, GET /v1/users/:id, ...
```

- `use(router)` monta em `/`; `use('/prefix', router)` monta com prefixo.
- Routers podem ser aninhados (`Router` dentro de `Router`) — os prefixos se acumulam.
- **O pattern do `use()` é wildcard implícito** e **não aceita** `:param` nem `*` no próprio prefixo.
- Middlewares registrados via `use()` num router aplicam-se a todas as rotas montadas sob ele.

### `route()` chainable

Para empilhar múltiplos métodos no mesmo path sem repetir o pattern:

```typescript
router.route('/users/:id')
  .get(getById)
  .put(update)
  .delete(remove);
```

### Route options por rota

```typescript
// upload aceita body maior que o limite global
router.post('/avatar', { max_body_length: 20 * 1024 * 1024 }, uploadAvatar);

// middlewares específicos da rota
router.post('/admin/reset', { middlewares: [requireAdmin] }, resetHandler);
```

## 3. Request

Referência: [docs/Request.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Request.md)

### Propriedades

| Propriedade | Tipo | Descrição |
|---|---|---|
| `request.method` | String | Método HTTP em **uppercase** |
| `request.url` | String | Path + query string |
| `request.path` | String | Path sem a query |
| `request.path_query` | String | Query string sem o `?` |
| `request.headers` | Object | Headers (chaves em lowercase) |
| `request.cookies` | Object | Cookies parseados |
| `request.path_parameters` | Object | Path params (`:name`) — alias Express: `request.params` |
| `request.query_parameters` | Object | Query params parseados — alias Express: `request.query` |
| `request.ip` | String | IP da conexão remota |
| `request.proxy_ip` | String | IP informado pelo proxy (requer `trust_proxy: true`) |
| `request.app` | Server | Instância do `Server` que originou a request |
| `request.raw` | uWS.HttpRequest | Objeto cru do uWS — **unsafe**, não use |

### Body: sempre assíncrono

**Não existe `request.body` preenchido por body-parser.** O body é lido do socket sob demanda, via métodos que retornam `Promise`:

```typescript
const buf  = await request.buffer();        // Buffer
const txt  = await request.text();          // string
const body = await request.json();          // objeto — default {} se parse falhar
const strict = await request.json(null);    // null em vez de {} quando inválido
const form = await request.urlencoded();    // objeto de form url-encoded
```

- **`request.json()` nunca lança em JSON inválido** — resolve para o `default_value` (default `{}`). Combine com zod para distinguir "body vazio" de "body inválido" (ver §7). Passe `null` como default quando precisar detectar parse failure.
- **Handlers com body devem ser `async`** e aguardar o parsing antes de responder.
- **Leia o body uma única vez por request**: os métodos consomem o stream do socket. Não misture `await request.json()` com streaming manual (`request.pipe(...)`) na mesma request.
- O tamanho aceito respeita `max_body_length` global — ou o override por rota (§2).
- **Chunked transfer encoding em requests NÃO é suportado** pelo Hyper-Express. Clientes devem enviar `Content-Length`.

### Multipart (upload de arquivos)

```typescript
router.post('/upload', { max_body_length: 50 * 1024 * 1024 }, async (request, response) => {
  await request.multipart(async (field) => {
    if (field.file) {
      await field.write(`/tmp/uploads/${field.file.name}`);
    }
  });
  response.status(201).json({ ok: true });
});
```

### Streaming de body

`Request` é um `Readable` do Node — para payloads grandes, faça pipe direto sem bufferizar:

```typescript
import { pipeline } from 'node:stream/promises';

router.put('/blobs/:id', { max_body_length: 1024 * 1024 * 1024 }, async (request, response) => {
  await pipeline(request, createWriteStream(`/data/${request.path_parameters.id}`));
  response.status(204).send();
});
```

### Regras

- **Nunca acesse `request.raw`** (uWS cru) — a doc oficial o marca como *unsafe*.
- Headers chegam com **chave lowercase**: `request.headers['x-request-id']`.
- `sign(value, secret)` / `unsign(signed, secret)` estão disponíveis para valores assinados (cookies de sessão).

## 4. Response

Referência: [docs/Response.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Response.md)

### Métodos principais

```typescript
response.status(201);                          // status code (chainable)
response.header('X-Request-Id', requestId);    // header (sobrescreve por default)
response.type('json');                         // content-type por extensão → application/json
response.json({ id: user.id });                // seta JSON + envia
response.send('plain body');                   // envia String | Buffer | ArrayBuffer
response.send();                               // envia sem body (ex: 204)
response.redirect('https://example.com');      // 302 + Location
response.cookie('session', token, 86_400_000, { httpOnly: true, secure: true, sameSite: 'strict' });
response.stream(readable, totalSize);          // pipe de um Readable como body
```

- `status()`, `header()`, `type()` são **chainable**: `response.status(201).json(user)`.
- **`status()` deve vir ANTES de `json()`/`send()`** — depois que a resposta inicia (`initiated === true`), status e headers não podem mais ser alterados.
- `json()` e `send()` **finalizam** a resposta — nada deve ser escrito depois.
- `stream(readable, total_size?)`: passe `total_size` quando souber o tamanho (define `Content-Length`); sem ele o Hyper-Express usa transferência em chunks na resposta.
- `cookie(name, null)` deleta o cookie. Expiry é em **milissegundos**.

### Gotcha crítico: nunca responder duas vezes

Responder após a resposta completar (ou após o cliente abortar) lança erro no runtime uWS. Use a propriedade **`response.completed`** como guard em qualquer caminho assíncrono:

```typescript
router.get('/slow', async (request, response) => {
  const data = await slowOperation(); // cliente pode ter desistido nesse meio tempo

  if (response.completed) return;     // aborted/completed — não toque mais na response
  response.json(data);
});
```

| Propriedade | Significado |
|---|---|
| `response.initiated` | Headers já foram enviados (status/headers travados) |
| `response.completed` | Resposta finalizada **ou** conexão abortada (alias de `aborted`) |
| `response.aborted` | Idem `completed` |

### Eventos de ciclo de vida

```typescript
response.on('abort', () => controller.abort());  // cliente fechou a conexão — cancele trabalho
response.on('finish', () => { /* resposta enviada (sem garantia de recepção) */ });
response.on('close', () => { /* conexão encerrada — fim absoluto da request */ });
```

Ligue operações longas a `AbortController` disparado no evento `abort` — evita desperdiçar DB/CPU em clientes que já desistiram.

### `atomic()` para múltiplas escritas

Wrapper do `cork` do uWS — agrupa escritas na mesma syscall:

```typescript
response.atomic(() => {
  response.status(200);
  response.header('X-Total-Count', String(total));
  response.json(items);
});
```

Use quando escrever status + vários headers + body em sequência em rotas quentes.

### Regras

- **Sempre finalize a resposta** em todos os caminhos do handler (incluindo `catch`) — ou lance/propague o erro para o error handler central (§6). Request sem resposta = conexão pendurada até timeout do cliente.
- **`return` após responder** dentro de branches — evita cair em um segundo `send()`.
- Não guarde `response` fora do ciclo da request (timers, caches, closures de longa duração).

## 5. Middlewares

Referência: [docs/Middlewares.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Middlewares.md)

### Assinaturas suportadas

```typescript
// 1. estilo callback (Express-like)
const mw: HyperExpress.MiddlewareHandler = (request, response, next) => {
  // ...
  next();          // continua a cadeia
  // next(error);  // interrompe e envia ao error handler central
};

// 2. estilo async — retorna Promise, SEM chamar next
const asyncMw = async (request: HyperExpress.Request, response: HyperExpress.Response) => {
  request.locals.user = await authenticate(request.headers.authorization);
  // resolve = continua; reject/throw = error handler central
};
```

- Middleware que **retorna Promise não deve chamar `next`** — a resolução da Promise avança a cadeia; um `throw`/reject vai para o `set_error_handler`.
- No estilo callback, `next(error)` encaminha o erro ao error handler central.
- Um middleware pode **encerrar a request** respondendo (`response.status(401).json(...)`) — a cadeia para ali.

### Global vs por rota

```typescript
// global (todo o server)
server.use(requestIdMiddleware);
server.use(loggingMiddleware);

// escopado por prefixo
server.use('/v1/admin', requireAdmin);

// escopado por router (aplica a tudo montado nele)
router.use(authMiddleware);

// por rota (variádico ou via options)
router.get('/reports', authMiddleware, rateLimit, listReports);
router.post('/jobs', { middlewares: [authMiddleware] }, createJob);
```

### Ordem recomendada no projeto

```typescript
server.use(requestContext);   // 1. requestId + logger filho (pino) em request.locals
server.use(accessLog);        // 2. log de acesso já com requestId
server.use(corsMiddleware);   // 3. CORS antes de auth (preflight OK)
server.use('/v1', authMiddleware); // 4. auth apenas nas rotas de negócio
```

### Exemplo canônico: request context com pino

```typescript
// pkg/http/middleware/request-context.ts
import { randomUUID } from 'node:crypto';
import type { MiddlewareHandler } from 'hyper-express';
import { logger } from '../../logger/logger.js';

export const requestContext: MiddlewareHandler = (request, response, next) => {
  const requestId = request.headers['x-request-id'] ?? randomUUID();

  request.locals.requestId = requestId;
  request.locals.logger = logger.child({
    requestId,
    method: request.method,
    path: request.path,
  });

  response.header('X-Request-Id', requestId);
  next();
};
```

### Regras

- **Hyper-Express NÃO captura exceptions síncronas fora da cadeia gerenciada em todos os casos** — mantenha handlers/middlewares `async` (Promise rejeitada é sempre capturada pelo error handler central).
- **Não use middlewares de Express que tocam `req`/`res` nativos** (`http.IncomingMessage`/`ServerResponse`) — ver §9. `hyper-express` é apenas *parcialmente* compatível com o ecossistema Express.
- Dados por request vão em **`request.locals`** (equivalente a `res.locals` do Express) — nunca em variáveis de módulo.

## 6. Tratamento de Erros Centralizado

Referência: [docs/Server.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Server.md)

### Filosofia: **lance o erro, não responda**

Handlers e middlewares devem **lançar** (ou rejeitar) erros de domínio. O handler global decide status, formato e logging — exatamente como o `ErrorMapper` + `BaseController` do harness:

```typescript
router.get('/:id', async (request, response) => {
  const user = await getByIdUseCase.perform({ id: request.path_parameters.id });
  if (!user) throw new UserNotFoundError(request.path_parameters.id);
  response.status(200).json(user);
});
```

### `set_error_handler` + ErrorMapper

O `ErrorMapper` (em `pkg/http/`) mapeia classes de erro de domínio → HTTP status. Cada app registra os seus erros no bootstrap:

```typescript
// pkg/http/error-mapper.ts
export interface ErrorResponseBody {
  error: string;
  status: number;
}

type ErrorClass = new (...args: never[]) => Error;

export class ErrorMapper {
  private readonly mappings = new Map<ErrorClass, number>();

  register(errorClass: ErrorClass, statusCode: number): this {
    this.mappings.set(errorClass, statusCode);
    return this;
  }

  map(error: unknown): { status: number; message: string } {
    if (error instanceof Error) {
      for (const [errorClass, status] of this.mappings) {
        if (error instanceof errorClass) {
          return { status, message: error.message };
        }
      }
    }
    return { status: 500, message: 'internal server error' };
  }
}
```

```typescript
// apps/user/src/bin/api.ts — wiring
import { DomainValidationError, UserNotFoundError, UserAlreadyExistsError } from '../domain/errors.js';

const errorMapper = new ErrorMapper()
  .register(DomainValidationError, 400)
  .register(UserNotFoundError, 404)
  .register(UserAlreadyExistsError, 409);

server.set_error_handler((request, response, error) => {
  const { status, message } = errorMapper.map(error);
  const log = request.locals.logger ?? logger;

  if (status >= 500) {
    log.error({ err: error, status }, 'request failed');
  } else {
    log.warn({ status, message }, 'request rejected');
  }

  if (response.completed) return; // conexão pode ter abortado durante o erro
  response.status(status).json({ error: status >= 500 ? 'internal server error' : message, status });
});
```

- O `set_error_handler` captura erros **síncronos e assíncronos** de handlers e middlewares (assinatura: `(request, response, error) => void`).
- **Nunca vaze mensagem de erro interno em 5xx** — logue o erro completo, responda mensagem genérica.
- Sempre proteja com `response.completed` antes de responder no handler de erro.

### `set_not_found_handler`

```typescript
server.set_not_found_handler((request, response) => {
  response.status(404).json({ error: 'route not found', status: 404 });
});
```

- Registre **uma única vez**, no bootstrap, **depois** de montar todos os routers — internamente vira uma rota catch-all.
- Sem ele, rotas desconhecidas recebem o default `text/plain` do Hyper-Express.

## 7. Validação com zod no Boundary

Todo input HTTP é validado com **zod** no boundary, antes de chegar ao use case. O padrão do harness é o helper genérico `bindAndHandle`:

```typescript
// pkg/http/bind-and-handle.ts
import type { Request, Response } from 'hyper-express';
import type { ZodType } from 'zod';

export class RequestValidationError extends Error {
  constructor(readonly details: Record<string, string[]>) {
    super('validation failed');
    this.name = 'RequestValidationError';
  }
}

type BoundHandler<T> = (request: Request, response: Response, input: T) => Promise<void>;

export function bindAndHandle<T>(schema: ZodType<T>, handler: BoundHandler<T>) {
  return async (request: Request, response: Response): Promise<void> => {
    const raw = await request.json(null); // null = distingue JSON inválido de {}
    const result = schema.safeParse(raw);

    if (!result.success) {
      throw new RequestValidationError(result.error.flatten().fieldErrors as Record<string, string[]>);
    }

    await handler(request, response, result.data);
  };
}
```

Registro no `ErrorMapper` com resposta 400 detalhada:

```typescript
server.set_error_handler((request, response, error) => {
  if (error instanceof RequestValidationError) {
    if (response.completed) return;
    response.status(400).json({ error: 'validation failed', status: 400, details: error.details });
    return;
  }
  // ... ErrorMapper para os demais (§6)
});
```

Uso no controller:

```typescript
// apps/user/src/infrastructure/http/user-controller.ts
import { z } from 'zod';

const createUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
});

router.post('/', bindAndHandle(createUserSchema, async (request, response, input) => {
  const output = await createUserUseCase.perform(input);
  response.status(201).json(output.user);
}));
```

### Validando params e query

```typescript
const listQuerySchema = z.object({
  cursor: z.string().optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

router.get('/', async (request, response) => {
  const query = listQuerySchema.safeParse(request.query_parameters);
  if (!query.success) {
    throw new RequestValidationError(query.error.flatten().fieldErrors as Record<string, string[]>);
  }
  const output = await listUseCase.perform(query.data);
  response.status(200).json(output);
});
```

### Regras

- **Sempre `safeParse`** no boundary — nunca `parse` (que lança `ZodError` cru e vaza estrutura interna).
- **`z.coerce`** para query/path params — chegam sempre como string.
- Schemas vivem junto do controller (infrastructure), **nunca no domain** — o domain não conhece zod nem HTTP.
- O tipo do input do use case vem do domain; use `schema satisfies ZodType<CreateUserInput>` (ou `z.infer`) para manter alinhamento em compile time.

## 8. Graceful Shutdown

Referência: [docs/Server.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Server.md)

| Método | Comportamento |
|---|---|
| `server.shutdown()` | **Graceful**: para de aceitar conexões e aguarda requests pendentes. Retorna `Promise<boolean>` |
| `server.close()` | **Imediato**: derruba todas as conexões na hora. Retorna `boolean` |

### Padrão do harness

```typescript
// apps/user/src/bin/api.ts
const server = createServer();
// ... montar rotas, error handlers ...
await server.listen(config.appPort);
logger.info({ port: server.port }, 'server started');

let shuttingDown = false;

async function shutdown(signal: string): Promise<void> {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info({ signal }, 'shutting down');

  const timeout = setTimeout(() => {
    logger.error('graceful shutdown timed out, forcing close');
    server.close();
    process.exit(1);
  }, 30_000);

  try {
    await server.shutdown();       // 1. drena requests HTTP em andamento
    await natsConnection.drain();  // 2. drena NATS
    await pgPool.end();            // 3. fecha pool do Postgres
    clearTimeout(timeout);
    process.exit(0);
  } catch (error) {
    logger.error({ err: error }, 'shutdown failed');
    process.exit(1);
  }
}

process.on('SIGTERM', () => void shutdown('SIGTERM'));
process.on('SIGINT', () => void shutdown('SIGINT'));
```

### Regras

- **Sempre `shutdown()` em produção**, nunca `close()` direto — `close()` corta requests no meio.
- Ordem de drenagem: **HTTP primeiro** (para de receber trabalho novo), depois mensageria, depois banco.
- **Timeout de segurança** no shutdown (SIGKILL do orquestrador costuma vir em 30s) — force `close()` + `exit(1)` se a drenagem travar.
- Proteja contra sinal duplicado (flag `shuttingDown`).
- Em Kubernetes, combine com readiness probe: marque não-ready antes de drenar.

## 9. Diferenças e Gotchas vs Express

Referência: [README](https://github.com/kartikk221/hyper-express) — *"HyperExpress is mostly compatible with Express but not 100%"*.

| Aspecto | Express | Hyper-Express |
|---|---|---|
| Transporte | `node:http` (`IncomingMessage`/`ServerResponse`) | **uWebSockets.js** (C++), objetos próprios |
| Body | `req.body` via body-parser middleware | `await request.json()/text()/buffer()` — sempre async |
| Query/params | `req.query` / `req.params` | `request.query_parameters` / `request.path_parameters` (aliases `query`/`params` existem) |
| Erro assíncrono | precisa de `next(err)` manual (Express 4) | Promise rejeitada cai direto no `set_error_handler` |
| 404 | último middleware da cadeia | `set_not_found_handler` |
| Shutdown | `httpServer.close()` | `server.shutdown()` (graceful) / `server.close()` (imediato) |
| Testes | `supertest(app)` via handle interno | **porta real efêmera + fetch** (ver §10) |
| Performance | baseline | throughput significativamente maior (benchmarks oficiais do uWS) |

### Gotchas que quebram silenciosamente

1. **Middlewares do ecossistema Express que dependem de `http.IncomingMessage`/`ServerResponse` NÃO funcionam** (ex: `express-session`, `passport`, `multer`, `helmet` em partes). A compatibilidade é de *assinatura* `(req, res, next)`, não de objetos. Prefira implementações próprias finas (CORS, request-id, auth) ou os recursos nativos (multipart embutido, `SessionEngine` do próprio Hyper-Express).
2. **Rotas devem ser registradas antes do `listen()`** — o uWS compila o roteamento no startup. Não há live reload/registro dinâmico de rotas em runtime.
3. **Sem chunked transfer encoding em requests** — clientes precisam mandar `Content-Length`.
4. **Sem HTTP/2** — uWebSockets.js atende HTTP/1.1 (e WebSocket). Termine HTTP/2/3 no proxy.
5. **Responder duas vezes lança erro fatal no uWS** — sempre guard com `response.completed` após qualquer `await` (§4).
6. **`request.json()` não lança em JSON inválido** — resolve `{}` por default. Sempre valide com zod depois (§7).
7. **Cluster/multi-processo**: uWS não compartilha o socket via módulo `cluster` do Node da forma tradicional — para escalar horizontalmente, rode N processos atrás de um LB (padrão em containers de qualquer forma).
8. **Header `uWebSockets`**: o header de versão do uWS vem desabilitado; não reative (`KEEP_UWS_HEADER`) em produção — evita fingerprinting.
9. **`request.raw` / `response.raw`** expõem os objetos uWS crus e são marcados *unsafe* na doc — nunca use em código de aplicação.

### WebSocket e SSE (visão rápida)

Referência: [docs/Websocket.md](https://github.com/kartikk221/hyper-express/blob/master/docs/Websocket.md)

```typescript
server.ws('/live/:room', {
  idle_timeout: 60,            // segundos (divisível por 4)
  max_payload_length: 32 * 1024,
}, (ws) => {
  ws.on('message', (msg) => ws.send(msg));
  ws.on('close', () => { /* cleanup */ });
});

// pub/sub por tópico (sintaxe MQTT)
server.publish('room/42', JSON.stringify(event));
```

- Autenticação de WS: rota `upgrade()` separada valida credenciais e chama `response.upgrade(context)`.
- SSE: `response.sse` fica disponível quando o cliente aceita `text/event-stream`.

## 10. Testes

Hyper-Express **não expõe um handle compatível com supertest** (não é `http.Server`). O padrão é subir o servidor real em **porta efêmera** e testar com **fetch nativo** do Node:

```typescript
// apps/user/test/http/user-routes.test.ts
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { buildServer } from '../helpers/build-server.js'; // monta server + rotas + handlers com deps fake

describe('POST /v1/users', () => {
  let server: HyperExpress.Server;
  let baseUrl: string;

  beforeAll(async () => {
    server = buildServer({ userRepository: new InMemoryUserRepository() });
    await server.listen(0);                  // 0 = porta efêmera do SO
    baseUrl = `http://127.0.0.1:${server.port}`;
  });

  afterAll(async () => {
    await server.shutdown();
  });

  it('cria usuário e retorna 201', async () => {
    const response = await fetch(`${baseUrl}/v1/users`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name: 'Ada', email: 'ada@example.com' }),
    });

    expect(response.status).toBe(201);
    const body = await response.json();
    expect(body).toMatchObject({ name: 'Ada', email: 'ada@example.com' });
  });

  it('retorna 400 com detalhes de validação para body inválido', async () => {
    const response = await fetch(`${baseUrl}/v1/users`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name: 'A' }),
    });

    expect(response.status).toBe(400);
    const body = await response.json();
    expect(body.error).toBe('validation failed');
    expect(body.details).toHaveProperty('email');
  });
});
```

### Regras

- **`listen(0)`** + `server.port` — nunca porta fixa (colide em CI/paralelismo do vitest).
- **`shutdown()` no `afterAll`** — server vazado mantém o processo do vitest vivo.
- Uma factory `buildServer(deps)` que recebe as dependências (awilix scope de teste ou fakes) e devolve o `Server` completo — mesma composição da produção, dependências trocadas.
- Teste o **contrato HTTP** (status, body, headers), não a implementação interna — use cases têm seus próprios testes unitários.
- Para integração real (Postgres/NATS), combine com testcontainers e a mesma factory.

## 11. Referência Rápida das APIs

### Server

| API | Assinatura | Nota |
|---|---|---|
| `new HyperExpress.Server(options?)` | ver §1 | `max_body_length`, `trust_proxy`, etc. |
| `listen` | `(port, host?) => Promise<socket>` | também aceita unix path; `listen(0)` = efêmera |
| `shutdown` | `() => Promise<boolean>` | graceful — aguarda requests pendentes |
| `close` | `() => boolean` | imediato — derruba conexões |
| `set_error_handler` | `((req, res, error) => void)` | erros sync + async, global |
| `set_not_found_handler` | `((req, res) => void)` | registrar uma vez, após as rotas |
| `use` | `(pattern?, ...mw \| Router)` | middlewares globais e montagem de routers |
| `get/post/put/patch/delete/head/options/any` | `(pattern, options?, ...mw, handler)` | registro de rotas |
| `ws` | `(pattern, options, handler)` | rota WebSocket |
| `publish` | `(topic, message, is_binary?, compress?)` | pub/sub WS por tópico MQTT |
| `port` / `locals` / `routes` | propriedades | porta ativa, storage global, rotas registradas |

### Router

| API | Assinatura | Nota |
|---|---|---|
| `new HyperExpress.Router()` | — | routers modulares, aninháveis |
| `use` | `(pattern?, ...mw \| Router)` | pattern é wildcard implícito; sem `:param`/`*` no prefixo |
| `route` | `(pattern) => chainable` | `.get().post()...` no mesmo path |
| métodos HTTP | idem Server | com route options `{ max_body_length, middlewares, streaming }` |

### Request

| API | Retorno | Nota |
|---|---|---|
| `path_parameters` / `params` | `Object` | path params `:name` |
| `query_parameters` / `query` | `Object` | valores sempre string — use `z.coerce` |
| `headers` / `cookies` / `ip` / `proxy_ip` / `method` / `path` / `url` | — | headers em lowercase; `proxy_ip` requer `trust_proxy` |
| `json(default?)` | `Promise<object>` | **não lança** — resolve default (`{}`) em JSON inválido |
| `text()` / `buffer()` / `urlencoded()` | `Promise` | body lido uma vez do socket |
| `multipart(opts?, handler)` | `Promise` | uploads via Busboy embutido |
| (Readable) | stream | pipe direto para payloads grandes |

### Response

| API | Assinatura | Nota |
|---|---|---|
| `status` | `(code, message?) => this` | antes de enviar body |
| `header` | `(name, value, overwrite?) => this` | sobrescreve por default |
| `type` | `(mime) => this` | `'json'` → `application/json` |
| `json` / `send` | `(body?) => boolean` | finalizam a resposta |
| `stream` | `(readable, total_size?)` | body via stream |
| `redirect` | `(url)` | 302 |
| `cookie` | `(name, value, expiryMs?, options?, sign?)` | `null` deleta |
| `atomic` | `(fn)` | cork do uWS — agrupa escritas |
| `completed` / `initiated` / `aborted` | propriedades | guard obrigatório após `await` |
| eventos | `abort`, `prepare`, `finish`, `close` | `abort` → cancele trabalho pendente |

## Checklist de Revisão (Hyper-Express)

Ao revisar ou aprovar código Hyper-Express, verifique:

- [ ] `Server` criado com `max_body_length` e `trust_proxy` explícitos
- [ ] `set_error_handler` e `set_not_found_handler` configurados no bootstrap
- [ ] Todas as rotas registradas **antes** do `await server.listen(...)`
- [ ] Rotas específicas declaradas antes de wildcards/genéricas
- [ ] Routers modulares por contexto, montados via `server.use('/prefix', router)`
- [ ] Body lido com `await request.json()/text()/buffer()` — nunca `request.body`
- [ ] `request.json(null)` + zod `safeParse` no boundary; erros viram 400 padronizado via `RequestValidationError`
- [ ] Query/path params validados com `z.coerce` quando numéricos
- [ ] Handlers lançam erros de domínio — **nunca** respondem 4xx/5xx manualmente fora do error handler
- [ ] `ErrorMapper` do app registra todas as classes de erro de domínio → status
- [ ] `response.completed` verificado após qualquer `await` antes de responder
- [ ] Nenhum caminho do handler termina sem resposta nem erro lançado
- [ ] Nenhum uso de `request.raw` / `response.raw`
- [ ] Nenhum middleware do ecossistema Express que dependa de `http.IncomingMessage`
- [ ] Dados por request em `request.locals` (requestId, logger filho do pino)
- [ ] Shutdown: SIGTERM/SIGINT → `server.shutdown()` → drenagem NATS → `pgPool.end()`, com timeout + `close()` de fallback
- [ ] Testes sobem o server com `listen(0)` + `fetch` nativo e fazem `shutdown()` no teardown — sem supertest
- [ ] Uploads usam `multipart()` nativo com `max_body_length` por rota

## Fontes Oficiais

Toda recomendação acima vem destas páginas — consulte-as ao bater dúvida:

- https://github.com/kartikk221/hyper-express (README, features, compatibilidade com Express)
- https://github.com/kartikk221/hyper-express/blob/master/docs/Server.md (construtor, listen, shutdown, error handlers)
- https://github.com/kartikk221/hyper-express/blob/master/docs/Router.md (rotas, use, route options, ws)
- https://github.com/kartikk221/hyper-express/blob/master/docs/Request.md (propriedades, body parsing, multipart)
- https://github.com/kartikk221/hyper-express/blob/master/docs/Response.md (métodos, completed, eventos, atomic)
- https://github.com/kartikk221/hyper-express/blob/master/docs/Websocket.md (rotas WS, pub/sub)
- https://github.com/kartikk221/hyper-express/blob/master/docs/Middlewares.md (assinaturas, execução)
