---
name: nats-best-practices
description: Aplica boas praticas oficiais do NATS 2.10+ e JetStream com Rust (async-nats) ao desenhar, escrever, revisar ou refatorar publishers, subscribers, streams, consumers e workers. Foco especial em padroes de distribuicao de carga onde multiplos workers leem do mesmo subject — Core NATS Queue Groups para fan-out efemero e JetStream Pull Consumers compartilhados (durable) para entrega persistente, at-least-once, com ack/nak/term via AckKind, retry com backoff, redelivery, deduplicacao por Nats-Msg-Id, controle de inflight (max_ack_pending), graceful shutdown, idempotencia, observabilidade e DLQ. Cobre convencoes de subjects (kebab-case hierarquico), conexao resiliente (retry_on_initial_connect, reconexao infinita, event_callback), autenticacao (creds/NKey/token), TLS, e configuracao de Stream (retention, storage, num_replicas, max_messages, max_bytes, max_age, duplicate_window). Use sempre que criar/editar codigo Rust que use o crate async-nats, ao modelar fluxos de mensageria, ao escalar consumers horizontalmente, ao decidir entre Core NATS vs JetStream, ao definir estrategia de retry/DLQ, ou ao revisar handlers de mensagem para garantir at-least-once + idempotencia.
---

# NATS 2.10+ & JetStream Best Practices (Rust · async-nats)

Skill baseada na documentacao oficial NATS (docs.nats.io) e no client oficial `async-nats` (Tokio) para Rust. Cobre padroes de producao para mensageria com foco em escalabilidade horizontal de workers consumindo do mesmo subject.

## Quando Aplicar

- Ao criar/editar codigo Rust que importe o crate `async_nats` (ou `async_nats::jetstream`)
- Ao modelar um novo fluxo de mensageria (event, command)
- Ao escalar consumers horizontalmente (multiplas replicas do mesmo worker)
- Ao decidir entre **Core NATS** (efemero, at-most-once) vs **JetStream** (persistente, at-least-once)
- Ao definir Stream, Consumer, Retention Policy, Storage, Replicas
- Ao implementar retry, backoff, DLQ ou redelivery
- Ao desenhar idempotencia e deduplicacao (`Nats-Msg-Id`)
- Ao revisar handlers para garantir ack/nak/term corretos
- Ao configurar graceful shutdown, max_ack_pending, batch, ack_wait

## 1. Core NATS vs JetStream — Quando Usar

| Caracteristica | Core NATS | JetStream |
|---|---|---|
| Entrega | At-most-once (fire & forget) | At-least-once (com ack) ou exactly-once (com dedup) |
| Persistencia | **Nao** (in-memory, sem replay) | **Sim** (file ou memory store) |
| Replay | Nao | Sim (DeliverPolicy) |
| Ack/Nak | Nao | Sim (`AckKind`) |
| Multiplos consumers | Queue Group (subscription efemera) | Consumer duravel (1 ou N processos) |
| Use case | Notificacoes, pub/sub volatil, RPC | Eventos de dominio, jobs, integracao com Postgres |

**Regra pratica:**
- Use **Core NATS** quando perder a mensagem e aceitavel (telemetria, broadcast, request/reply sincrono).
- Use **JetStream** quando a mensagem **nao pode** ser perdida (eventos de dominio, jobs, billing, auditoria).
- Em duvida: **JetStream**.

## 2. Convencao de Subjects

Subjects sao **hierarquicos**, separados por ponto (`.`). Use **kebab-case** dentro de cada token.

| Padrao | Exemplo |
|---|---|
| `events.<dominio>.<evento>` | `events.user.created` |
| `<dominio>.<entidade>.<id>.<evento>` | `orders.order.4f2a.paid` |
| `cmd.<dominio>.<acao>` | `cmd.email.send` |
| `dlq.<consumer>` | `dlq.billing-on-user-created` |

Wildcards: `*` casa **um** token (`orders.order.*.paid`); `>` casa **um ou mais** (sempre no final: `orders.>`).

### Regras

- Subjects sao **case-sensitive**; padronize minusculo.
- Nao inclua dados sensiveis no subject (vao pro broker em texto, logs, monitor `$SYS.>`).
- Stream nomeado em UPPER_SNAKE: `EVENTS_USER`, `EVENTS_BILLING`.
- Consumer durable nomeado por **proposito do worker**: `billing-on-user-created`, `order-paid-projector`.
- Evite `>` em consumer de producao sem necessidade.

## 3. Conexao — Setup Resiliente

O projeto encapsula a conexao nos helpers do `pkg` (ver `architecture.md`). A reconexao no `async-nats` e **automatica e infinita por padrao**; use `retry_on_initial_connect()` para nao falhar caso o servidor ainda nao esteja pronto no boot.

```rust
// pkg/src/nats/conn.rs
use std::time::Duration;

pub async fn connect(url: &str, name: &str) -> anyhow::Result<async_nats::Client> {
    let client = async_nats::ConnectOptions::new()
        .name(name)                          // aparece no monitor — sempre setar
        .retry_on_initial_connect()          // nao falha se o server ainda nao subiu
        .ping_interval(Duration::from_secs(20))
        .event_callback(|event| async move { // nunca silenciar eventos assincronos
            match event {
                async_nats::Event::Disconnected => tracing::warn!("nats disconnected"),
                async_nats::Event::Connected => tracing::info!("nats connected"),
                async_nats::Event::ClientError(err) => tracing::error!(error = %err, "nats client error"),
                other => tracing::debug!(event = ?other, "nats event"),
            }
        })
        .connect(url)                        // url: "nats://a:4222,nats://b:4222"
        .await?;
    Ok(client)
}
```

Autenticacao (produção): `.credentials_file(path).await?` (JWT+NKey) ou `.require_tls(true)` + `.add_root_certificates(ca)`.

### Regras de conexao

| Regra | Detalhe |
|---|---|
| **1 conexao por processo** | `async_nats::Client` e clonavel e seguro para uso concorrente (`Arc` interno). Nao criar pool |
| **Sempre setar `name`** | Essencial para debug no monitor (`CONNZ`) |
| **Reconexao infinita** | Comportamento padrao do async-nats; nunca desabilitar em producao |
| **Lista de URLs** | Passar todos os seeds do cluster (`a,b,c`) |
| **`event_callback`** | Logar disconnect, reconnect, erros do client |
| **Drain no shutdown** | `client.drain().await` — processa inflight e fecha (nunca so `drop`) |
| **TLS em producao** | creds/NKey + TLS |

## 4. JetStream — Contexto

```rust
let js = async_nats::jetstream::new(client.clone());   // publish + administracao de streams/consumers
```

- Payload sempre `bytes` (`bytes::Bytes`): `serde_json::to_vec(&envelope)?.into()` no publish; `serde_json::from_slice(&msg.payload)?` no consume.

## 5. Stream — Configuracao

```rust
use std::time::Duration;

use async_nats::jetstream::stream::{Config as StreamConfig, DiscardPolicy, RetentionPolicy, StorageType};

async fn ensure_stream(js: &async_nats::jetstream::Context) -> anyhow::Result<()> {
    js.create_stream(StreamConfig {
        name: "EVENTS_USER".to_string(),
        description: Some("Eventos do dominio User".to_string()),
        subjects: vec!["events.user.>".to_string()],       // capta todos os eventos do dominio
        retention: RetentionPolicy::Limits,                // default, mais flexivel
        storage: StorageType::File,                         // File em prod; Memory so para teste
        discard: DiscardPolicy::Old,                        // ao bater limite, descarta antigos (FIFO)
        max_messages: -1,                                   // sem limite por contagem
        max_bytes: 20 * 1024 * 1024 * 1024,                 // 20 GB
        max_age: Duration::from_secs(30 * 24 * 60 * 60),    // 30 dias
        max_message_size: 1024 * 1024,                      // 1 MB por mensagem
        num_replicas: 3,                                    // R3 em prod (HA)
        duplicate_window: Duration::from_secs(2 * 60),      // janela de dedup por Nats-Msg-Id
        ..Default::default()
    })
    .await?;   // create_stream e idempotente (cria ou atualiza)
    Ok(())
}
```

### Retention Policies

| Policy | Comportamento | Quando usar |
|---|---|---|
| `RetentionPolicy::Limits` | Mantem ate bater limite (msgs, bytes, age). Default | 90% dos casos — eventos, audit |
| `RetentionPolicy::WorkQueue` | Apaga a mensagem apos ack de **qualquer** consumer | Job queue com 1 consumer por mensagem |
| `RetentionPolicy::Interest` | Mantem enquanto **algum** consumer tem interesse | Pub/sub duravel |

### Storage e Replicas

- `StorageType::File` sempre em producao; `StorageType::Memory` so para cache/teste.
- `num_replicas: 3` em producao (R3, sobrevive a 1 falha; requer cluster >= 3 nos). `5` para criticos (billing/ledger). `1` apenas dev/single-node.

## 6. Publisher — Padrao Robusto

O `Publisher` generico do `pkg/nats/` (ver `architecture.md`) monta o envelope e publica com `Nats-Msg-Id` para dedup, aguardando o `PubAck` do servidor (duplo `await`):

```rust
// pkg/src/nats/publisher.rs
use anyhow::{Context, Result};
use async_nats::jetstream;

use crate::event::Envelope;

pub struct Publisher {
    js: jetstream::Context,
    source: String,
}

impl Publisher {
    pub fn new(js: jetstream::Context, source: impl Into<String>) -> Self {
        Self { js, source: source.into() }
    }

    pub async fn publish<T: serde::Serialize>(&self, subject: &str, msg_id: &str, data: T) -> Result<()> {
        let envelope = Envelope::new(subject, &self.source, data);
        let payload = serde_json::to_vec(&envelope).context("serialize envelope")?;

        let mut headers = async_nats::HeaderMap::new();
        headers.insert("Nats-Msg-Id", msg_id);   // dedup server-side na duplicate_window

        self.js
            .publish_with_headers(subject.to_string(), headers, payload.into())
            .await
            .context("publish")?     // 1o await: enfileira no client
            .await                   // 2o await: aguarda o PubAck do JetStream
            .context("await publish ack")?;

        Ok(())
    }
}
```

### Regras de publish

| Regra | Por que |
|---|---|
| **Sempre `Nats-Msg-Id` com ID estavel** | Dedup automatica na `duplicate_window` do stream |
| **Aguardar o duplo `await` (PubAck)** | Garante que a mensagem chegou ao stream antes de seguir |
| **Publicar DEPOIS do commit do DB** | Nunca dentro de tx. Ver [[postgres-best-practices]] |
| **Payload JSON** | `serde_json::to_vec(...)?.into()` -> `bytes::Bytes` |
| **Headers para metadados** | `Nats-Msg-Id`, `event-type`, `schema-version`, `traceparent` |
| **Subject estavel** | Mudar subject = quebrar consumers. Versionar via header |

## 7. Multiplos Workers no Mesmo Subject — O Padrao Central

### 7.1 Core NATS — Queue Subscription (efemero, at-most-once)

```rust
use futures::StreamExt;

// Todas as replicas no MESMO queue group => NATS distribui round-robin
let mut sub = client
    .queue_subscribe("events.user.created", "user-created-workers".into())
    .await?;

while let Some(msg) = sub.next().await {
    handle(msg).await;   // processado por UM worker apenas — sem persistencia, sem ack
}
```

Use somente quando perder mensagens e aceitavel (telemetria, broadcast, RPC).

### 7.2 JetStream — Pull Consumer Durável Compartilhado (recomendado)

**Este e o padrao recomendado para multiplos workers** com persistencia, ack e replay. Crie **um unico consumer durável** no stream; todas as replicas consomem desse **mesmo consumer** (efetivamente um "queue group durável"):

```rust
use std::time::Duration;

use async_nats::jetstream::consumer::{pull::Config as PullConfig, AckPolicy};

async fn ensure_consumer(
    js: &async_nats::jetstream::Context,
) -> anyhow::Result<async_nats::jetstream::consumer::Consumer<PullConfig>> {
    let stream = js.get_stream("EVENTS_USER").await?;

    let consumer = stream
        .create_consumer(PullConfig {
            durable_name: Some("billing-on-user-created".to_string()), // nome estavel — igual em todas as replicas
            filter_subject: "events.user.created".to_string(),         // filtra do stream
            ack_policy: AckPolicy::Explicit,                           // OBRIGATORIO ack manual
            ack_wait: Duration::from_secs(30),                         // ate redelivery se sem ack
            max_deliver: 5,                                            // tentativas antes de descartar/DLQ
            max_ack_pending: 256,                                      // inflight (todas as replicas somadas)
            backoff: vec![                                             // backoff entre redeliveries
                Duration::from_secs(1),
                Duration::from_secs(5),
                Duration::from_secs(15),
                Duration::from_secs(60),
            ],
            ..Default::default()
        })
        .await?;

    Ok(consumer)
}
```

### 7.3 Worker — Loop de Consumo

```rust
use futures::StreamExt;

let mut messages = consumer.messages().await?;
while let Some(msg) = messages.next().await {
    let Ok(msg) = msg else { continue };   // erro de stream — segue o loop
    subscriber.handle_user_created(msg).await;   // handler faz ack/nak/term e NUNCA propaga panico
}
```

Rode o loop numa `tokio::task` e finalize com `worker.abort()` no shutdown (ver `architecture.md`).

### 7.4 Escala Horizontal

**Nao crie um consumer por replica.** Crie **UM consumer durável** e suba **N replicas** do mesmo binario apontando para ele (mesmo `durable_name`).

| Decisao | Padrao Recomendado |
|---|---|
| Quantos consumers? | **1 por proposito de processamento**, nao por replica |
| `durable_name` igual em todas as replicas? | **Sim** — e o que faz N replicas dividirem trabalho |
| Como aumentar paralelismo? | Subir mais replicas ou aumentar `max_ack_pending` |
| Dois processamentos independentes do mesmo subject? | **Dois consumers** com `durable_name` distintos, mesmo `filter_subject` |

### 7.5 Workers Independentes (fan-out de domínio)

Quando dois dominios reagem ao mesmo evento (ex: `events.user.created` cria conta **E** dispara email), crie **dois consumers durables distintos** apontando para o mesmo subject. Cada um mantem seu proprio offset, redelivery e DLQ — falha no email **nao** afeta a criacao de conta.

## 8. Ack, Nak, Term — Tabela de Decisao

No `async-nats`, o ack e feito via `msg.ack().await` ou `msg.ack_with(AckKind::...).await`:

| Chamada | Significado | Quando usar |
|---|---|---|
| `msg.ack().await` | Sucesso — nao redeliver | Handler concluiu com sucesso |
| `msg.ack_with(AckKind::Nak(None)).await` | Falha — redeliver (backoff do consumer) | Erro transitorio |
| `msg.ack_with(AckKind::Nak(Some(dur))).await` | Redeliver apos `dur` | Erro transitorio com backoff especifico |
| `msg.ack_with(AckKind::Term).await` | Erro terminal — **nao** redeliver | Payload invalido, regra de negocio violada |
| `msg.ack_with(AckKind::Progress).await` | "Ainda processando" — reseta ack_wait | Handler longo (heartbeat) |

### Handler correto (nunca deixa panico/erro escapar)

```rust
use async_nats::jetstream::{AckKind, Message};

pub async fn handle_user_created(&self, msg: Message) {
    let msg_id = msg.headers.as_ref()
        .and_then(|h| h.get("Nats-Msg-Id"))
        .map(|v| v.to_string())
        .unwrap_or_default();

    // 1. parse/validacao -> Term em payload corrompido (nunca reenviar)
    let envelope: Envelope = match serde_json::from_slice(&msg.payload) {
        Ok(env) => env,
        Err(err) => {
            tracing::error!(msg_id, error = %err, "invalid payload");
            let _ = msg.ack_with(AckKind::Term).await;
            return;
        }
    };

    // 2. processamento -> Nak em erro transitorio (redelivery com backoff)
    match self.use_case.perform(/* ... */).await {
        Ok(_) => {
            let _ = msg.ack().await;   // so apos sucesso
        }
        Err(err) => {
            tracing::error!(msg_id, error = %err, "processing failed");
            let _ = msg.ack_with(AckKind::Nak(None)).await;
        }
    }
}
```

### Anti-patterns

```rust
// ERRADO — esquecer o ack: ack_wait expira -> redelivery ate max_deliver
// ERRADO — ack antes de processar: perda silenciosa se o processamento falhar
// ERRADO — Nak em payload corrompido: estoura max_deliver antes de "desistir" (use Term)
```

## 9. Idempotencia — Obrigatorio em At-Least-Once

JetStream entrega **pelo menos uma vez**. Toda redelivery, failover ou restart pode duplicar. O handler **deve** ser idempotente.

### Estrategias

| Estrategia | Quando usar |
|---|---|
| **Dedup via `Nats-Msg-Id` + `duplicate_window`** | Janela curta (minutos). Resolve duplicatas do publisher |
| **Tabela de processados** (`processedEvents`) | Janela longa. UPSERT por `eventId` |
| **Operacao naturalmente idempotente** | `UPDATE SET status='paid' WHERE id=$1` |
| **UPSERT com chave de negocio** | `INSERT ... ON CONFLICT DO NOTHING` |

### Padrao: tabela de processados (dentro da transaction)

```sql
CREATE TABLE "processedEvents" (
    "eventId"     TEXT        PRIMARY KEY,
    "consumer"    TEXT        NOT NULL,
    "processedAt" TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```rust
let event_id = msg.headers.as_ref()
    .and_then(|h| h.get("Nats-Msg-Id"))
    .map(|v| v.to_string());
let Some(event_id) = event_id else {
    let _ = msg.ack_with(AckKind::Term).await;   // sem id -> nao ha como deduplicar
    return;
};

let mut tx = pool.begin().await?;
let claimed: Option<(String,)> = sqlx::query_as(
    r#"INSERT INTO "processedEvents" ("eventId", "consumer") VALUES ($1, $2)
       ON CONFLICT ("eventId") DO NOTHING RETURNING "eventId""#,
)
.bind(&event_id)
.bind(&consumer_name)
.fetch_optional(&mut *tx)
.await?;

if claimed.is_none() {
    // duplicata — apenas ack, sem reprocessar
} else {
    // aplica a logica de negocio na MESMA transaction
}
tx.commit().await?;
let _ = msg.ack().await;
```

## 10. DLQ (Dead Letter Queue)

JetStream nao tem DLQ embutida. Implemente publicando para `dlq.<consumer>` no caminho terminal — e so faca `Term` **depois** que o DLQ confirmou:

```rust
match self.handler(msg.clone()).await {
    Ok(_) => { let _ = msg.ack().await; }
    Err(err) if err.is_permanent() => {
        match self.publish_dlq(&msg, &err).await {   // dlq.<consumer> com headers de contexto
            Ok(_) => { let _ = msg.ack_with(AckKind::Term).await; }   // so Term apos DLQ confirmado
            Err(_) => { let _ = msg.ack_with(AckKind::Nak(Some(Duration::from_secs(30)))).await; }
        }
    }
    Err(_) => { let _ = msg.ack_with(AckKind::Nak(None)).await; }   // transitorio
}
```

Crie um stream `DLQ` capturando `dlq.>` com retencao longa (30-90 dias) para inspecao e replay manual.

## 11. Graceful Shutdown

Alinhado ao consumer do `architecture.md` (SIGINT + SIGTERM):

```rust
tracing::info!("consumer started");
shutdown_signal().await;   // ctrl_c ou SIGTERM

// ordem inversa da criacao
worker.abort();            // 1. para de consumir novas mensagens
client.drain().await?;     // 2. drain do NATS (processa inflight, flush, fecha)
pool.close().await;        // 3. fecha o pool Postgres
tracing::info!("shutdown complete");
```

- Use `client.drain().await`, nunca apenas `drop`, no shutdown gracioso.
- Pare o loop de consumo **antes** do drain.
- `ack_wait` deve ser maior que o tempo de drain esperado.

## 12. Observabilidade

- Logs estruturados (`tracing`) com `stream`, `consumer`, `seq`, `num_delivered`, `msg_id`, `subject`, `error`. Metadados de entrega via `msg.info()` (`num_delivered`, `stream_sequence`).
- Metricas por worker: recebidas, acked, naked, termed, duracao, pending (via `consumer.info().await`).
- Tracing W3C: propague `traceparent` via header; cada handler abre um span filho.

## 13. Autenticacao e Seguranca

| Mecanismo | Quando | Config (`ConnectOptions`) |
|---|---|---|
| **creds (JWT+NKey)** | Padrao producao | `.credentials_file("/path/svc.creds").await?` |
| **NKey** | Account interno | `.nkey(seed)` |
| **User/Password** | Dev local | `.user_and_password(user, pass)` |
| **Token** | Scripts | `.token(token)` |
| **TLS** | Sempre em prod | `.require_tls(true)` + `.add_root_certificates(ca)` |

- Nunca commitar `.creds`/seed/senha (usar secret manager). TLS obrigatorio fora do localhost. Credencial por servico.

## 14. Configuracao de Servidor (Docker)

```yaml
# docker-compose.yml — NATS com JetStream (dev single-node)
services:
  nats:
    image: nats:2.12-alpine
    command: ["-js", "-sd", "/data", "-m", "8222"]
    ports: ["4222:4222", "8222:8222"]
    volumes: ["nats-data:/data"]

volumes:
  nats-data: {}
```

Para producao, cluster R3 (3 nos com `--cluster`/`--routes` e `--server_name` distintos).

## Checklist — Novo Stream

- [ ] Nome em UPPER_SNAKE (`EVENTS_USER`)
- [ ] `subjects` cobre apenas o dominio (sem `>` raiz)
- [ ] `storage: StorageType::File` em producao
- [ ] `num_replicas: 3` em producao
- [ ] `retention: RetentionPolicy::Limits` na maioria
- [ ] `max_age`, `max_bytes`, `max_message_size` definidos
- [ ] `duplicate_window` configurado (1-5 min) para dedup
- [ ] `create_stream` idempotente no `ensure_stream`

## Checklist — Novo Consumer Durável

- [ ] `durable_name` em kebab-case, descreve o proposito do worker
- [ ] `filter_subject` especifico (nao `>`)
- [ ] `ack_policy: AckPolicy::Explicit`
- [ ] `ack_wait` >= maior timeout de handler esperado
- [ ] `max_deliver` finito (3-10), com DLQ
- [ ] `backoff` definido — sem reentregar em loop apertado
- [ ] `max_ack_pending` calibrado ao throughput (total entre replicas)
- [ ] `num_replicas: 3` em producao

## Checklist — Novo Worker

- [ ] 1 conexao NATS por processo (`Client` clonado), com `client.drain()` no shutdown
- [ ] Handler assincrono que **nunca propaga panico** — ack/nak/term em todos os caminhos
- [ ] `AckKind::Term` para erros permanentes/payload invalido; `AckKind::Nak` para transitorios
- [ ] Idempotencia garantida (dedup, UPSERT, ou tabela de processados)
- [ ] Logs com `stream`, `consumer`, `seq`, `num_delivered`, `msg_id`
- [ ] DLQ implementado (subject `dlq.<consumer>`)
- [ ] Graceful shutdown: `worker.abort()` -> `client.drain()` -> `pool.close()`
- [ ] Testes de integracao com NATS real (testcontainers)

## Checklist — Multiplos Workers no Mesmo Subject

- [ ] **Um unico** consumer durável (nao um por replica)
- [ ] Todas as replicas usam o **mesmo** `durable_name`
- [ ] `max_ack_pending` dimensionado para o **total** entre replicas
- [ ] Idempotencia **obrigatoria** — qualquer replica pode pegar a redelivery
- [ ] Dois processamentos paralelos -> **dois consumers** com `durable_name` distintos
- [ ] Se for Core NATS efemero: `queue_subscribe` com queue group estavel

## Fontes

- https://docs.nats.io/nats-concepts/jetstream — conceitos JetStream
- https://docs.nats.io/nats-concepts/jetstream/consumers — Consumer config
- https://docs.nats.io/nats-concepts/jetstream/streams — Stream config
- https://github.com/nats-io/nats.rs — cliente oficial Rust (async-nats)
- https://docs.rs/async-nats/latest/async_nats/ — API reference (Client, jetstream, AckKind)
- https://docs.nats.io/running-a-nats-service/configuration/securing_nats — auth/TLS
