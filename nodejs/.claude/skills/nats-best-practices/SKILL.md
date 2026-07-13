---
name: nats-best-practices
description: Aplica boas práticas oficiais do NATS 2.10+ e JetStream com Node.js/TypeScript (nats.js) ao desenhar, escrever, revisar ou refatorar publishers, subscribers, streams, consumers e workers. Foco especial em padrões de distribuição de carga onde múltiplos workers leem do mesmo subject — Core NATS Queue Groups para fan-out efêmero e JetStream Pull Consumers compartilhados (durable) para entrega persistente, exactly-once, com ack/nak/term, retry com backoff, redelivery, deduplicação por Nats-Msg-Id, controle de inflight (max_ack_pending), backpressure, graceful shutdown, idempotência, observabilidade e DLQ. Cobre também convenções de nomenclatura de subjects (kebab-case hierárquico), conexão resiliente (reconnect, jitter, maxReconnectAttempts=-1), autenticação (NKey/JWT/credentials), TLS, segregação de contextos (NATS Context), pooling de conexões, fluxo request/reply, KV/Object Store, mirrors/sources e configuração de Stream (retention, storage, num_replicas, discard, max_msgs, max_bytes, max_age). Use sempre que criar/editar código TypeScript que use o pacote nats (nats.js), ao modelar fluxos de mensageria, ao escalar consumers horizontalmente, ao decidir entre Core NATS vs JetStream, ao definir estratégia de retry/DLQ, ou ao revisar handlers de mensagem para garantir at-least-once + idempotência.
---

# NATS 2.10+ & JetStream Best Practices (Node.js / TypeScript)

Skill baseada na documentação oficial NATS (docs.nats.io) e no client oficial `nats` (nats.js) para Node.js. Cobre padrões de produção para mensageria com foco em escalabilidade horizontal de workers consumindo do mesmo subject.

## Quando Aplicar

- Ao criar/editar código TypeScript que importe o pacote `nats` (nats.js)
- Ao modelar um novo fluxo de mensageria (event, command, query)
- Ao escalar consumers horizontalmente (múltiplas réplicas do mesmo worker)
- Ao decidir entre **Core NATS** (efêmero, at-most-once) vs **JetStream** (persistente, at-least-once)
- Ao definir Stream, Consumer, Retention Policy, Storage, Replicas
- Ao implementar retry, backoff, dead-letter queue (DLQ) ou redelivery
- Ao desenhar idempotência e deduplicação (`Nats-Msg-Id`)
- Ao revisar handlers para garantir ack/nak/term corretos
- Ao configurar graceful shutdown, max_ack_pending, fetch batch, ack_wait
- Ao definir convenção de subjects, autenticação (NKey/JWT/creds), TLS

## 1. Core NATS vs JetStream — Quando Usar

| Característica          | Core NATS                          | JetStream                                  |
|-------------------------|------------------------------------|--------------------------------------------|
| Entrega                 | At-most-once (fire & forget)       | At-least-once (com ack) ou exactly-once (com dedup) |
| Persistência            | **Não** (in-memory, sem replay)    | **Sim** (file ou memory store)              |
| Replay                  | Não                                | Sim (StartSeq, StartTime, DeliverAll, etc.) |
| Ack/Nak                 | Não                                | Sim                                         |
| Múltiplos consumers     | Queue Group (subscription efêmera) | Consumer durável (1 ou N processos)         |
| Replicas                | Cluster mesh                       | RAFT por Stream (R1/R3/R5)                  |
| Use case                | Notificações, pub/sub volátil, RPC | Eventos de domínio, jobs, comandos críticos |

**Regra prática:**
- Use **Core NATS** quando perder a mensagem é aceitável (telemetria, broadcast de presença, request/reply síncrono).
- Use **JetStream** quando a mensagem **não pode** ser perdida (event sourcing, jobs, integração com Postgres, billing, auditoria).
- Em dúvida: **JetStream**.

## 2. Convenção de Subjects

Subjects são **hierárquicos**, separados por ponto (`.`). Use **kebab-case** dentro de cada token.

| Padrão                                       | Exemplo                                          |
|----------------------------------------------|--------------------------------------------------|
| `<dominio>.<entidade>.<evento>`              | `billing.invoice.created`                        |
| `<dominio>.<entidade>.<id>.<evento>`         | `orders.order.4f2a.paid`                         |
| `cmd.<dominio>.<acao>`                       | `cmd.email.send`                                 |
| `q.<dominio>.<acao>`                         | `q.search.user-by-email` (request/reply)         |
| `dlq.<stream>`                               | `dlq.orders-events`                              |

Wildcards:
- `*` casa **um** token: `orders.order.*.paid` → casa `orders.order.4f2a.paid`.
- `>` casa **um ou mais** tokens (sempre no final): `orders.>` → casa `orders.order.4f2a.paid`.

### Regras

- Subjects são **case-sensitive**. Padronize tudo minúsculo.
- Não inclua dados sensíveis no subject (vão pro broker em texto, logs, monitor `$SYS.>`).
- Stream nomeado em **kebab-case maiúsculo** ou sufixo `-events`: `ORDERS_EVENTS`, `billing-events`.
- Consumer durable nomeado por **propósito do worker**: `email-sender-worker`, `invoice-projector`.
- Evite `>` em consumer de produção sem necessidade — assina o stream inteiro.

## 3. Conexão — Setup Resiliente

```typescript
// src/shared/nats/connection.ts
import { readFile } from 'node:fs/promises';
import { connect, credsAuthenticator, Events, type NatsConnection } from 'nats';
import { logger } from '../logger/logger.js';

export async function connectNats(cfg: NatsConfig): Promise<NatsConnection> {
  const nc = await connect({
    servers: cfg.servers,                  // ['nats://a:4222', 'nats://b:4222', 'nats://c:4222']
    name: cfg.appName,                     // aparece no monitor — sempre setar
    maxReconnectAttempts: -1,              // reconectar indefinidamente
    reconnectTimeWait: 2_000,
    reconnectJitter: 500,                  // jitter evita thundering herd
    pingInterval: 20_000,
    maxPingOut: 3,
    timeout: 5_000,                        // conexão inicial

    ...(cfg.credsFile
      ? { authenticator: credsAuthenticator(await readFile(cfg.credsFile)) } // NATS 2.x JWT
      : {}),
    ...(cfg.tlsEnabled
      ? { tls: { caFile: cfg.caFile, certFile: cfg.certFile, keyFile: cfg.keyFile } }
      : {}),
  });

  // Monitorar eventos assíncronos — nunca silenciar
  void (async () => {
    for await (const status of nc.status()) {
      switch (status.type) {
        case Events.Disconnect:
          logger.warn({ server: status.data }, 'nats disconnected');
          break;
        case Events.Reconnect:
          logger.info({ server: status.data }, 'nats reconnected');
          break;
        case Events.Error:
          logger.error({ data: status.data }, 'nats async error');
          break;
        default:
          logger.debug({ type: status.type, data: status.data }, 'nats status');
      }
    }
  })();

  // Promise que resolve quando a conexão fecha de vez (com o erro, se houver)
  void nc.closed().then((err) => {
    if (err) logger.warn({ err }, 'nats connection closed with error');
    else logger.warn('nats connection closed');
  });

  return nc;
}
```

### Regras de conexão

| Regra                                  | Detalhe                                                              |
|----------------------------------------|----------------------------------------------------------------------|
| **1 conexão por processo**             | `NatsConnection` é segura para uso concorrente. Não criar pool.      |
| **Sempre setar `name`**                | Aparece em `$SYS.REQ.SERVER.PING.CONNZ` — essencial para debug       |
| **`maxReconnectAttempts: -1`**         | Reconectar sempre; nunca desistir em produção                        |
| **`reconnectJitter`**                  | Obrigatório em cluster — evita reconexão sincronizada                |
| **Lista de `servers`**                 | Passar todos os seeds do cluster (`a,b,c`)                           |
| **Monitorar `nc.status()` e `nc.closed()`** | Logar disconnect, reconnect, erros assíncronos e o close final  |
| **Drain no shutdown**                  | `await nc.drain()` (não `nc.close()`) — processa inflight e fecha    |
| **TLS em produção**                    | NKey/JWT + mutual TLS                                                |
| **Nada de `reconnect: false`**         | Exceto em scripts CLI one-shot                                       |

## 4. JetStream — Clientes e Codecs

O acesso ao JetStream é feito por dois objetos distintos: `nc.jetstream()` para **messaging** (publish/consume) e `nc.jetstreamManager()` para **administração** (streams, consumers, purge, info).

```typescript
import type { JetStreamClient, JetStreamManager } from 'nats';

const js: JetStreamClient = nc.jetstream();          // publicar e consumir
const jsm: JetStreamManager = await nc.jetstreamManager(); // administrar streams/consumers
// com domain: nc.jetstream({ domain: 'hub' })
```

### Codecs — payload sempre `Uint8Array`

O NATS transporta bytes. Use codecs para encode/decode tipado:

```typescript
import { JSONCodec, StringCodec } from 'nats';

const jc = JSONCodec<OrderEvent>();
const payload = jc.encode(event);           // OrderEvent -> Uint8Array
const decoded = jc.decode(msg.data);        // Uint8Array -> OrderEvent

const sc = StringCodec();                   // strings puras
// Alternativa sem codec: new TextEncoder().encode(...) / msg.json<T>() em JsMsg
```

## 5. Stream — Configuração

```typescript
import { DiscardPolicy, RetentionPolicy, StorageType, nanos } from 'nats';

await jsm.streams.add({
  name: 'ORDERS_EVENTS',
  description: 'Eventos do domínio Orders',
  subjects: ['orders.>'],                       // capta todos os eventos do domínio
  storage: StorageType.File,                    // File em prod; Memory só para cache
  retention: RetentionPolicy.Limits,            // Limits é o default e o mais flexível
  discard: DiscardPolicy.Old,                   // ao bater limite, descarta antigos (FIFO)
  max_msgs: -1,                                 // sem limite por contagem
  max_bytes: 20 * 1024 * 1024 * 1024,           // 20 GB
  max_age: nanos(30 * 24 * 60 * 60 * 1000),     // 30 dias — durações em NANOSSEGUNDOS
  max_msg_size: 1 * 1024 * 1024,                // 1 MB por mensagem
  num_replicas: 3,                              // R3 em prod (mínimo para HA)
  duplicate_window: nanos(2 * 60 * 1000),       // janela de dedup por Nats-Msg-Id
});
```

**Atenção:** todas as durações da API JetStream (`max_age`, `ack_wait`, `duplicate_window`, `backoff`) são em **nanossegundos**. Use sempre o helper `nanos(millis)` exportado pelo `nats` — nunca escreva o número cru.

Para idempotência no provisionamento (stream já existe), use o padrão ensure:

```typescript
export async function ensureStream(jsm: JetStreamManager, config: StreamConfig): Promise<void> {
  try {
    await jsm.streams.info(config.name);
    await jsm.streams.update(config.name, config); // já existe → atualiza
  } catch (err) {
    if (isStreamNotFoundError(err)) {
      await jsm.streams.add(config);               // não existe → cria
      return;
    }
    throw err;
  }
}
```

### Retention Policies

| Policy                     | Comportamento                                                            | Quando usar                                  |
|----------------------------|--------------------------------------------------------------------------|----------------------------------------------|
| `RetentionPolicy.Limits`   | Mantém até bater limite (msgs, bytes, age). Default.                     | 90% dos casos — eventos, audit                |
| `RetentionPolicy.Workqueue`| Apaga a mensagem depois que **qualquer** consumer fizer ack              | Job queue com 1 consumer por mensagem         |
| `RetentionPolicy.Interest` | Mantém enquanto **algum** consumer ainda está interessado                | Pub/sub durável sem replay para novos joiners |

### Storage

| Storage              | Durabilidade | Performance | Quando usar                          |
|----------------------|--------------|-------------|--------------------------------------|
| `StorageType.File`   | Persiste     | Alta        | **Sempre em produção**               |
| `StorageType.Memory` | Volátil      | Máxima      | Cache, dev, testes                   |

### Replicas

| Replicas | HA            | Performance         | Quando usar                          |
|----------|---------------|---------------------|--------------------------------------|
| 1        | Sem HA        | Máxima              | Dev, single-node                     |
| 3        | Sobrevive 1 falha | Bom              | Padrão produção                      |
| 5        | Sobrevive 2 falhas | Menor (mais RAFT) | Críticos (billing, ledger)         |

**Replicas só faz sentido com cluster JetStream com >= 3 servidores.**

## 6. Publisher — Padrão Robusto

```typescript
import { JSONCodec, type JetStreamClient } from 'nats';
import { logger } from '../logger/logger.js';

const jc = JSONCodec<DomainEvent>();

export class Publisher {
  constructor(private readonly js: JetStreamClient) {}

  async publishEvent(event: DomainEvent): Promise<void> {
    const subject = `orders.order.${event.aggregateId}.${event.type}`;

    // js.publish é awaitável — resolve somente com o ACK do server.
    // Para throughput em bulk, dispare várias promises e aguarde
    // Promise.all(...) ANTES de retornar do handler que chamou.
    const ack = await this.js.publish(subject, jc.encode(event), {
      msgID: event.id,                            // OBRIGATÓRIO para dedup
      expect: { streamName: 'ORDERS_EVENTS' },    // defensivo: rejeita se subject mudou de stream
    });

    logger.debug({ stream: ack.stream, seq: ack.seq, id: event.id }, 'event published');
  }
}
```

### Regras de publish

| Regra                                          | Por quê                                                              |
|------------------------------------------------|----------------------------------------------------------------------|
| **Sempre `msgID` com ID estável**              | Dedup automática na janela `duplicate_window` do stream              |
| **`await js.publish(...)` em código transacional** | Garante que a mensagem chegou ao stream antes de seguir          |
| **Publish concorrente só para bulk/throughput**| E sempre `await Promise.all(...)` antes do shutdown                  |
| **Publicar DEPOIS do commit do DB**            | Nunca dentro de tx Postgres. Vide [[postgres-best-practices]]        |
| **Payload em JSON (ou protobuf)**              | JSON para flexibilidade; protobuf para volume alto                   |
| **Headers para metadados**                     | `traceparent`, `tenant-id`, `event-type`, `schema-version`           |
| **Subject estável**                            | Mudar subject = quebrar consumers. Versionar via header              |

### Headers — convenção mínima

```typescript
import { headers } from 'nats';

const h = headers();
h.set('Nats-Msg-Id', event.id);            // dedup
h.set('event-type', event.type);           // routing/filter
h.set('schema-version', 'v1');             // evolução
h.set('traceparent', traceparent);         // W3C trace context
h.set('tenant-id', event.tenantId);

const ack = await js.publish(subject, payload, { headers: h });
```

## 7. Múltiplos Workers no Mesmo Subject — O Padrão Central

### 7.1 Core NATS — Queue Subscription (efêmero, at-most-once)

Quando vários processos do mesmo serviço se inscrevem com o **mesmo queue group**, o NATS faz **load balancing**: cada mensagem vai para **exatamente um** worker do grupo.

```typescript
// Worker 1, 2, 3... cada réplica do processo roda este código.
// Todas no mesmo queue group => NATS distribui round-robin entre eles.
const sub = nc.subscribe('orders.order.*.paid', { queue: 'order-paid-workers' });

for await (const msg of sub) {
  await handle(msg); // processado por UM worker apenas
}
```

**Características:**
- **Sem persistência** — mensagem perdida se nenhum worker estiver up.
- **Sem replay, sem ack** — fire & forget.
- **Escala horizontalmente** apenas adicionando réplicas no mesmo queue group.

**Use somente quando** perder mensagens é aceitável (telemetria, broadcast, RPC).

### 7.2 JetStream — Pull Consumer Durável Compartilhado (recomendado)

**Este é o padrão recomendado para múltiplos workers** com persistência, ack e replay.

> **Como funciona:** crie **um único consumer durável** no stream. Todas as réplicas do worker fazem `consume()` desse **mesmo consumer** (obtido via `js.consumers.get(stream, durable)`). O servidor distribui mensagens entre as réplicas, respeitando `max_ack_pending`, `ack_wait` e redelivery. É efetivamente um "queue group durável".

```typescript
import { AckPolicy, DeliverPolicy, ReplayPolicy, nanos } from 'nats';

// Cria/garante um consumer durável compartilhado por TODAS as réplicas do worker.
await jsm.consumers.add('ORDERS_EVENTS', {
  durable_name: 'order-paid-projector',        // nome estável — mesmo em todas as réplicas
  description: 'Projeta orders.*.paid no read model',
  filter_subject: 'orders.order.*.paid',       // filtra do stream
  ack_policy: AckPolicy.Explicit,              // OBRIGATÓRIO ack manual
  ack_wait: nanos(30_000),                     // tempo até redelivery se sem ack
  max_deliver: 5,                              // tentativas antes de descartar/DLQ
  backoff: [                                   // backoff entre redeliveries (nanossegundos)
    nanos(1_000),
    nanos(5_000),
    nanos(15_000),
    nanos(60_000),
  ],
  max_ack_pending: 256,                        // inflight por consumer (TODAS as réplicas somadas)
  deliver_policy: DeliverPolicy.All,           // novo consumer pega tudo desde o início
  replay_policy: ReplayPolicy.Instant,
  num_replicas: 3,                             // HA do estado do consumer
});
```

### 7.3 Worker — Loop de Consumo

```typescript
import type { Consumer, ConsumerMessages, JetStreamClient, JsMsg } from 'nats';
import { logger } from '../logger/logger.js';

export class Worker {
  private messages?: ConsumerMessages;

  constructor(
    private readonly js: JetStreamClient,
    private readonly handler: Handler,
  ) {}

  async run(): Promise<void> {
    // Todas as réplicas pegam o MESMO consumer durável.
    const consumer: Consumer = await this.js.consumers.get('ORDERS_EVENTS', 'order-paid-projector');

    // consume() entrega mensagens continuamente, com controle de inflight/buffer.
    this.messages = await consumer.consume({ max_messages: 64 }); // batch interno do client

    for await (const msg of this.messages) {
      await this.handle(msg);
      // Para paralelismo interno controlado, dispare sem await usando p-limit —
      // nunca paralelismo ilimitado (estoura RAM e ack_wait).
    }
    // O loop termina quando stop() é chamado — a mensagem em andamento é concluída.
  }

  stop(): void {
    this.messages?.stop(); // para de puxar novas mensagens; inflight termina
  }

  private async handle(msg: JsMsg): Promise<void> {
    const info = msg.info;
    const log = logger.child({
      stream: info.stream,
      consumer: info.consumer,
      seq: info.streamSequence,
      delivered: info.redeliveryCount,
      msgId: msg.headers?.get('Nats-Msg-Id'),
    });

    try {
      // Timeout por mensagem — sempre menor que ack_wait
      await this.handler.handle(msg, AbortSignal.timeout(20_000));
      msg.ack();
    } catch (err) {
      if (err instanceof PermanentError) {
        // erro terminal — não retentar; envia pro DLQ via term()
        log.warn({ err }, 'permanent error, terminating');
        msg.term();
      } else {
        // erro transitório — nak com delay calculado (millis)
        const delay = backoffFor(info.redeliveryCount);
        log.warn({ err, delay }, 'transient error, naking');
        msg.nak(delay);
      }
    }
  }
}
```

Alternativa com callback (sem for-await):

```typescript
this.messages = await consumer.consume({
  max_messages: 64,
  callback: (msg) => { void this.handle(msg); },
});
```

Para processamento em lotes controlados, use `fetch` em loop:

```typescript
while (!stopped) {
  const batch = await consumer.fetch({ max_messages: 32, expires: 30_000 }); // long-poll
  for await (const msg of batch) {
    await this.handle(msg);
  }
}
```

### 7.4 Escala Horizontal — Como Adicionar Mais Workers

**Não crie um consumer por réplica.** Crie **UM consumer durável** e suba **N réplicas** do mesmo processo apontando para esse consumer.

```
                        ┌─────────────────────────────┐
                        │  Stream: ORDERS_EVENTS      │
                        │  Subjects: orders.>         │
                        └─────────────┬───────────────┘
                                      │
                            ┌─────────▼──────────┐
                            │ Consumer Durável:  │
                            │ order-paid-projector│
                            │ max_ack_pending: 256│
                            └─┬────────┬───────┬─┘
                              │        │       │
                  ┌───────────▼┐  ┌────▼───┐ ┌─▼──────────┐
                  │ Worker #1  │  │Worker#2│ │ Worker #3  │
                  └────────────┘  └────────┘ └────────────┘
```

| Decisão                                | Padrão Recomendado                                             |
|----------------------------------------|----------------------------------------------------------------|
| Quantos consumers?                     | **1 por propósito de processamento**, não por réplica          |
| `durable_name` igual em todas as réplicas? | **Sim** — é o que faz N réplicas dividirem trabalho        |
| Como aumentar paralelismo?             | (a) Subir mais réplicas; (b) aumentar `max_ack_pending`        |
| Como reduzir lag?                      | Aumentar réplicas + `max_messages` + paralelismo interno       |
| Como ter dois processamentos paralelos? | **Dois consumers** com nomes diferentes, mesmo filter_subject |

### 7.5 Workers Independentes para o Mesmo Subject (fan-out de domínio)

Quando dois domínios precisam reagir ao mesmo evento (ex: `orders.order.*.paid` precisa atualizar projeção **E** enviar email), crie **dois consumers durables distintos** apontando para o mesmo subject. Cada um é uma "trilha" independente, com seu próprio ack/retry.

```typescript
await jsm.consumers.add('ORDERS_EVENTS', {
  durable_name: 'order-paid-projector',
  filter_subject: 'orders.order.*.paid',
  ack_policy: AckPolicy.Explicit,
});

await jsm.consumers.add('ORDERS_EVENTS', {
  durable_name: 'order-paid-emailer',
  filter_subject: 'orders.order.*.paid',
  ack_policy: AckPolicy.Explicit,
});
```

Cada consumer mantém seu próprio offset, redelivery e DLQ. Falha no email **não** afeta a projeção.

## 8. Ack, Nak, Term, Working — Tabela de Decisão

| Método                         | Significado                                            | Quando usar                                    |
|--------------------------------|--------------------------------------------------------|------------------------------------------------|
| `msg.ack()`                    | Sucesso — não redeliver                                | Handler concluiu com sucesso                   |
| `msg.nak()`                    | Falha — redeliver **imediatamente** (sujeito a backoff) | Erro transitório sem delay específico         |
| `msg.nak(delayMillis)`         | Falha — redeliver após `delayMillis`                   | Erro transitório com backoff calculado         |
| `msg.term()`                   | Erro terminal — **não** redeliver, **não** conta para max_deliver | Payload inválido, regra de negócio violada |
| `msg.working()`                | "Ainda processando" — reseta ack_wait                  | Handler longo (heartbeat para evitar redelivery) |
| `await msg.ackAck()`           | Ack com confirmação do server                          | Quando perder o ack é catastrófico             |

### Anti-patterns

```typescript
// ERRADO — esquecer o ack
async function handle(msg: JsMsg): Promise<void> {
  await process(msg.data);
  // sem ack → ack_wait expira → redelivery infinita até max_deliver
}

// ERRADO — ack antes de processar
async function handle(msg: JsMsg): Promise<void> {
  msg.ack();          // ack já saiu...
  await process(msg.data); // ...mas se falhar aqui, mensagem perdida sem redelivery
}

// ERRADO — nak sem distinguir erro permanente
async function handle(msg: JsMsg): Promise<void> {
  try {
    await process(msg.data);
    msg.ack();
  } catch {
    msg.nak();        // payload corrompido vai redeliver 5x antes de desistir
  }
}

// CORRETO
async function handle(msg: JsMsg): Promise<void> {
  try {
    await process(msg.data);
    msg.ack();
  } catch (err) {
    if (err instanceof PermanentError) {
      msg.term();     // direto pro DLQ via max_deliver=1 ou max_deliver excedido
    } else {
      msg.nak(backoff);
    }
  }
}
```

## 9. Idempotência — Obrigatório em At-Least-Once

JetStream entrega **pelo menos uma vez**. Toda redelivery, todo failover, todo restart pode causar duplicata. O handler **deve** ser idempotente.

### Estratégias

| Estratégia                              | Quando usar                                          |
|-----------------------------------------|------------------------------------------------------|
| **Dedup via `Nats-Msg-Id` + `duplicate_window`** | Janela curta (minutos). Resolve duplicatas do publisher |
| **Tabela de processados** (`processedEvents`) | Janela longa. UPSERT por `eventId`                  |
| **Operação naturalmente idempotente**   | `UPDATE SET status='paid' WHERE id=$1`               |
| **UPSERT com chave de negócio**         | `INSERT ... ON CONFLICT DO NOTHING`                  |
| **Versionamento otimista**              | `WHERE version = $expected_version`                  |

### Padrão: tabela de processados

```sql
CREATE TABLE "processedEvents" (
    "eventId"     TEXT        PRIMARY KEY,
    "consumer"    TEXT        NOT NULL,
    "processedAt" TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export class EventHandler {
  constructor(
    private readonly transactor: Transactor,
    private readonly consumerName: string,
  ) {}

  async handle(msg: JsMsg): Promise<void> {
    const eventId = msg.headers?.get('Nats-Msg-Id');
    if (!eventId) {
      throw new PermanentError('missing Nats-Msg-Id');
    }

    await this.transactor.runInTx(async (client) => {
      // Tenta marcar como processado; se já existe, é duplicata → ack sem reprocessar.
      const claim = `
          INSERT INTO "processedEvents" ("eventId", "consumer")
          VALUES ($1, $2)
          ON CONFLICT ("eventId") DO NOTHING
          RETURNING "eventId"`;

      const result = await client.query(claim, [eventId, this.consumerName]);
      if (result.rowCount === 0) {
        return; // duplicata — apenas ack
      }
      await this.applyBusinessLogic(client, msg);
    });
  }
}
```

## 10. DLQ (Dead Letter Queue)

JetStream não tem DLQ embutida, mas você implementa via:

### Opção A — Stream avisa quando max_deliver é excedido

Configure o stream para emitir advisory em `$JS.EVENT.ADVISORY.CONSUMER.MAX_DELIVERIES.<STREAM>.<CONSUMER>`. Um worker dedicado pode capturar e republicar em `dlq.<consumer>`.

### Opção B — Publicar para DLQ explicitamente no `term()`

```typescript
export class Worker {
  private async handle(msg: JsMsg): Promise<void> {
    const info = msg.info;

    try {
      await this.handler.handle(msg);
      msg.ack();
      return;
    } catch (err) {
      // Se for permanente OU excedeu o limite → DLQ
      if (err instanceof PermanentError || info.redeliveryCount >= this.maxDeliver) {
        try {
          await this.publishDLQ(msg, err);
        } catch {
          // não consegue mandar pro DLQ — nak para tentar de novo
          msg.nak(30_000);
          return;
        }
        msg.term(); // só term DEPOIS que DLQ confirmou
        return;
      }
      msg.nak(backoffFor(info.redeliveryCount));
    }
  }

  private async publishDLQ(original: JsMsg, cause: unknown): Promise<void> {
    const h = headers();
    for (const [key, values] of original.headers ?? []) {
      for (const value of values) h.append(key, value);
    }
    h.set('dlq-reason', cause instanceof Error ? cause.message : String(cause));
    h.set('dlq-original-subject', original.subject);
    h.set('dlq-consumer', this.consumerName);
    h.set('dlq-at', new Date().toISOString());

    await this.js.publish(`dlq.${this.consumerName}`, original.data, {
      headers: h,
      msgID: original.headers?.get('Nats-Msg-Id'),
    });
  }
}
```

Crie um stream separado `DLQ` capturando `dlq.>` com retenção longa (30-90 dias) para inspeção e replay manual.

## 11. Graceful Shutdown

```typescript
// src/apps/orders/consumer/main.ts
async function main(): Promise<void> {
  const nc = await connectNats(cfg);
  const js = nc.jetstream();
  const worker = buildWorker(js);

  const runPromise = worker.run();

  const shutdown = async (signal: string): Promise<void> => {
    logger.info({ signal }, 'shutdown requested');

    // 1) parar de puxar novas mensagens, mas processar inflight
    worker.stop();                    // messages.stop() — o for-await termina
    await withTimeout(runPromise, 30_000);

    // 2) drain da conexão NATS (flush + close)
    try {
      await nc.drain();
    } catch (err) {
      logger.error({ err }, 'nats drain failed');
    }
  };

  process.once('SIGINT', () => void shutdown('SIGINT'));
  process.once('SIGTERM', () => void shutdown('SIGTERM'));

  const err = await nc.closed();      // bloqueia até a conexão fechar de vez
  if (err) {
    logger.error({ err }, 'nats closed with error');
    process.exitCode = 1;
  }
}

void main();
```

**Regras:**
- Use `await nc.drain()`, nunca `nc.close()`, em shutdown gracioso.
- Chame `messages.stop()` **antes** do drain — o loop de consumo termina após concluir a mensagem em andamento.
- `ack_wait` deve ser maior que o tempo de drain configurado.
- Capture `SIGINT`/`SIGTERM` com `process.once` — o segundo sinal deve matar o processo.

## 12. Observabilidade

### Métricas mínimas por worker

| Métrica                          | Tipo       | Labels                                |
|----------------------------------|------------|---------------------------------------|
| `nats_messages_received_total`   | Counter    | `consumer`, `subject`                 |
| `nats_messages_acked_total`      | Counter    | `consumer`                            |
| `nats_messages_naked_total`      | Counter    | `consumer`, `reason`                  |
| `nats_messages_termed_total`     | Counter    | `consumer`, `reason`                  |
| `nats_handler_duration_seconds`  | Histogram  | `consumer`, `outcome`                 |
| `nats_consumer_pending`          | Gauge      | `stream`, `consumer` (via `consumer.info()`) |
| `nats_consumer_ack_pending`      | Gauge      | `stream`, `consumer`                  |
| `nats_consumer_num_redelivered`  | Gauge      | `stream`, `consumer`                  |

### Tracing (W3C)

Propague `traceparent` via header. Cada handler abre um span filho:

```typescript
import { context, propagation, trace } from '@opentelemetry/api';

const carrier: Record<string, string> = {};
for (const [key, values] of msg.headers ?? []) {
  carrier[key.toLowerCase()] = values[0] ?? '';
}

const parentCtx = propagation.extract(context.active(), carrier);
const span = tracer.startSpan(`nats.consume ${msg.subject}`, undefined, parentCtx);
try {
  // ... handler ...
} finally {
  span.end();
}
```

### Logs estruturados

Sempre logar com `pino` incluindo: `stream`, `consumer`, `seq`, `delivered`, `msgId`, `subject`, `err`.

## 13. Autenticação e Segurança

| Mecanismo            | Quando usar                                | Configuração no client                          |
|----------------------|--------------------------------------------|-------------------------------------------------|
| **JWT + NKey (creds)** | **Padrão produção** (multi-tenant)        | `authenticator: credsAuthenticator(credsBytes)` |
| **NKey puro**        | Account interno sem JWT                    | `authenticator: nkeyAuthenticator(seed)`        |
| **User/Password**    | Dev local                                  | `user: '...', pass: '...'`                      |
| **Token**            | Scripts simples                            | `token: '...'`                                  |
| **TLS**              | **Sempre em produção** (mTLS preferível)   | `tls: { caFile, certFile, keyFile }`            |

**Regras de segurança:**
- **Nunca** commitar `.creds`, NKey ou senha. Usar secret manager.
- **TLS obrigatório** em qualquer ambiente fora do localhost.
- Cada serviço com **sua própria credencial** — facilita revogação e auditoria.
- Use **accounts** NATS para isolar tenants/ambientes.
- Permissões granulares por subject (`publish: ["orders.>"]`, `subscribe: ["cmd.email.>"]`).

## 14. Request/Reply (Core NATS)

```typescript
// Servidor (responder)
const sub = nc.subscribe('q.search.user-by-email', { queue: 'search-workers' });
void (async () => {
  for await (const msg of sub) {
    try {
      const user = await repository.findByEmail(msg.headers?.get('email') ?? '');
      msg.respond(jc.encode(user));
    } catch (err) {
      // padrão de erro: header "error"
      respondError(msg, err);
    }
  }
})();

// Cliente (requester)
const reply = await nc.request('q.search.user-by-email', payload, { timeout: 2_000 });
const user = jc.decode(reply.data);
```

**Regras:**
- Sempre passar `{ timeout }` no `nc.request` (timeout obrigatório).
- Queue group no respondente para load balance.
- Para erros, use headers na resposta (`error`, `error-code`) — nunca payload ambíguo.
- Para fluxos críticos com replay/audit, **prefira JetStream** com subject `cmd.<dominio>.<acao>` + resposta via outro subject.

## 15. KV Store e Object Store

JetStream oferece duas abstrações úteis:

### KV — chave/valor versionado

```typescript
const kv = await js.views.kv('feature-flags', {
  history: 5,
  ttl: 0, // sem TTL
});

await kv.put('billing.new-checkout', 'true');
const entry = await kv.get('billing.new-checkout');
console.log(entry?.string()); // 'true'
```

Use para: feature flags, configuração dinâmica, leader election leve, cache distribuído.

### Object Store — blobs grandes

```typescript
const os = await js.views.os('invoices-pdf');
await os.putBlob({ name: 'inv-123.pdf' }, pdfBytes);
```

Use para: PDFs, imagens, exports — qualquer payload > 1 MB que não cabe bem em mensagem.

## 16. Anti-patterns a Evitar

| Anti-pattern                                                         | Por que evitar                                                      |
|----------------------------------------------------------------------|---------------------------------------------------------------------|
| Criar conexão NATS por requisição                                    | Conexão é cara e segura para concorrência — use 1 por processo      |
| Criar consumer por réplica do worker                                 | Quebra distribuição. Use 1 consumer durable compartilhado           |
| `max_ack_pending` muito alto (>10k)                                  | Consome RAM, atrasa redelivery, mascara lentidão                    |
| `max_ack_pending: 1` sem necessidade                                 | Serializa o consumer todo — péssimo throughput                      |
| Esquecer `msgID` no publish                                          | Sem dedup nativa — handler precisa fazer tudo                       |
| Handler sem timeout                                                  | Trava o worker; ack_wait expira; redelivery em loop                 |
| Ack antes de processar                                               | Perda silenciosa de mensagem em falha                               |
| Nak em erro permanente                                               | Estoura max_deliver — atrasa entrega do DLQ                         |
| Publicar dentro de transaction Postgres                              | Vide [[postgres-best-practices]] — eventos sempre **após** commit  |
| `term()` sem mandar para DLQ                                         | Mensagem some sem rastro                                            |
| Subject com dado sensível (CPF, email)                               | Aparece em logs, monitor, advisory                                  |
| `nc.close()` em shutdown gracioso                                    | Use `await nc.drain()` para processar inflight                      |
| `maxReconnectAttempts` finito em produção                            | Worker morre numa janela de rede                                    |
| Wildcard `>` em consumer crítico                                     | Captura mensagens inesperadas; subject novo entra silenciosamente   |
| Promise de `js.publish` sem `await` (fire & forget)                  | Erro engolido; mensagem pode nunca ter chegado ao stream            |
| Duração em milissegundos onde a API espera nanossegundos             | `ack_wait: 30000` = 30 µs. Sempre `nanos(30_000)`                   |
| `StorageType.Memory` em produção                                     | Perde tudo em restart do server                                     |
| `num_replicas: 1` em stream crítico                                  | Sem HA — perda total se o servidor cair                             |
| Stream com `RetentionPolicy.Workqueue` + múltiplos consumers diferentes | Inválido — Workqueue exige 1 consumer por mensagem               |

## 17. Configuração de Servidor (Docker)

```yaml
# docker-compose.yml — cluster JetStream R3 para dev/staging
services:
  nats-1:
    image: nats:2.12-alpine
    command:
      - "-js"                                    # habilita JetStream
      - "-sd"
      - "/data"                                  # storage dir
      - "-m"
      - "8222"                                   # monitor HTTP
      - "--cluster_name"
      - "app-cluster"
      - "--cluster"
      - "nats://0.0.0.0:6222"
      - "--routes"
      - "nats://nats-2:6222,nats://nats-3:6222"
      - "--server_name"
      - "nats-1"
    ports: ["4222:4222", "8222:8222"]
    volumes: ["nats1-data:/data"]

  nats-2:
    image: nats:2.12-alpine
    command:
      - "-js"
      - "-sd"
      - "/data"
      - "--cluster_name"
      - "app-cluster"
      - "--cluster"
      - "nats://0.0.0.0:6222"
      - "--routes"
      - "nats://nats-1:6222,nats://nats-3:6222"
      - "--server_name"
      - "nats-2"
    volumes: ["nats2-data:/data"]

  nats-3:
    image: nats:2.12-alpine
    command:
      - "-js"
      - "-sd"
      - "/data"
      - "--cluster_name"
      - "app-cluster"
      - "--cluster"
      - "nats://0.0.0.0:6222"
      - "--routes"
      - "nats://nats-1:6222,nats://nats-2:6222"
      - "--server_name"
      - "nats-3"
    volumes: ["nats3-data:/data"]

volumes:
  nats1-data: {}
  nats2-data: {}
  nats3-data: {}
```

## Checklist — Novo Stream

- [ ] Nome em UPPER_SNAKE ou kebab-case com sufixo `-events`
- [ ] `subjects` cobre apenas o domínio (sem `>` raiz)
- [ ] `storage: StorageType.File` em produção
- [ ] `num_replicas: 3` em produção
- [ ] `retention` adequada (`RetentionPolicy.Limits` na maioria)
- [ ] `max_age`, `max_bytes` definidos — nunca deixar default infinito
- [ ] `duplicate_window` configurado (1-5 min) para dedup
- [ ] `max_msg_size` definido para evitar payloads gigantes
- [ ] Durações passadas com `nanos(millis)` — nunca número cru
- [ ] Documentado no `description` com finalidade do stream

## Checklist — Novo Consumer Durável

- [ ] `durable_name` em kebab-case, descreve o propósito do worker
- [ ] `filter_subject` específico (não `>`)
- [ ] `ack_policy: AckPolicy.Explicit`
- [ ] `ack_wait` >= maior `handler timeout` esperado
- [ ] `max_deliver` finito (3-10), nunca infinito sem DLQ
- [ ] `backoff` definido — sem reentregar em loop apertado
- [ ] `max_ack_pending` calibrado ao throughput e à RAM
- [ ] `num_replicas: 3` em produção (HA do estado do consumer)
- [ ] `description` explica qual worker consome

## Checklist — Novo Worker

- [ ] 1 conexão NATS por processo, com `nc.drain()` no shutdown
- [ ] Handlers com timeout próprio (`AbortSignal.timeout` / helper) menor que `ack_wait`
- [ ] Ack/Nak/Term explícitos em todos os caminhos
- [ ] Erros permanentes detectados (`err instanceof PermanentError`) → `term` ou DLQ
- [ ] Idempotência garantida (dedup, UPSERT, ou tabela de processados)
- [ ] Métricas expostas: received, acked, naked, termed, duration, pending
- [ ] Tracing W3C propagado via `traceparent` header
- [ ] Logs com `stream`, `consumer`, `seq`, `delivered`, `msgId`
- [ ] DLQ implementado (subject `dlq.<consumer>`)
- [ ] Graceful shutdown com `messages.stop()` + `nc.drain()`
- [ ] Testes de integração com `nats-server` real ou testcontainers

## Checklist — Múltiplos Workers no Mesmo Subject

- [ ] **Um único** consumer durável criado (não um por réplica)
- [ ] Todas as réplicas usam o **mesmo** `durable_name` no `js.consumers.get(stream, durable)`
- [ ] `max_ack_pending` dimensionado para o **total** entre todas as réplicas
- [ ] Idempotência **obrigatória** — qualquer réplica pode pegar a redelivery
- [ ] Métrica de lag (`num_pending`) monitorada para decidir escalar réplicas
- [ ] Se precisar de processamentos paralelos: **dois consumers** com nomes distintos
- [ ] Se for Core NATS efêmero: `nc.subscribe(subject, { queue })` com queue group estável

## Fontes

- https://docs.nats.io/nats-concepts/jetstream — conceitos JetStream
- https://docs.nats.io/nats-concepts/jetstream/consumers — Consumer config
- https://docs.nats.io/nats-concepts/jetstream/streams — Stream config
- https://docs.nats.io/using-nats/developer/connecting — conexão e reconexão
- https://docs.nats.io/using-nats/developer/sending — publish patterns
- https://docs.nats.io/using-nats/developer/receiving/queues — queue groups (Core)
- https://github.com/nats-io/nats.js — cliente oficial JavaScript/TypeScript
- https://nats-io.github.io/nats.js/ — API reference do nats.js (JetStream, KV, Object Store)
- https://docs.nats.io/running-a-nats-service/configuration/securing_nats — auth/TLS
- https://docs.nats.io/running-a-nats-service/configuration/clustering/jetstream_clustering — JetStream clustering
