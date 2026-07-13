---
name: nats-best-practices
description: Aplica boas praticas oficiais do NATS 2.10+ e JetStream com Python (nats-py) ao desenhar, escrever, revisar ou refatorar publishers, subscribers, streams, consumers e workers. Foco especial em padroes de distribuicao de carga onde multiplos workers leem do mesmo subject — Core NATS Queue Groups para fan-out efemero e JetStream Pull Consumers compartilhados (durable) para entrega persistente, at-least-once, com ack/nak/term, retry com backoff, redelivery, deduplicacao por Nats-Msg-Id, controle de inflight (max_ack_pending), graceful shutdown, idempotencia, observabilidade e DLQ. Cobre convencoes de subjects (kebab-case hierarquico), conexao resiliente (reconnect infinito, jitter), autenticacao (creds/NKey/token), TLS, e configuracao de Stream (retention, storage, num_replicas, max_msgs, max_bytes, max_age, duplicate_window). Use sempre que criar/editar codigo Python que use o pacote nats (nats-py), ao modelar fluxos de mensageria, ao escalar consumers horizontalmente, ao decidir entre Core NATS vs JetStream, ao definir estrategia de retry/DLQ, ou ao revisar handlers de mensagem para garantir at-least-once + idempotencia.
---

# NATS 2.10+ & JetStream Best Practices (Python / nats-py)

Skill baseada na documentacao oficial NATS (docs.nats.io) e no client oficial `nats-py` para Python. Cobre padroes de producao para mensageria com foco em escalabilidade horizontal de workers consumindo do mesmo subject.

## Quando Aplicar

- Ao criar/editar codigo Python que importe o pacote `nats` (nats-py)
- Ao modelar um novo fluxo de mensageria (event, command)
- Ao escalar consumers horizontalmente (multiplas replicas do mesmo worker)
- Ao decidir entre **Core NATS** (efemero) vs **JetStream** (persistente)
- Ao definir Stream, Consumer, Retention Policy, Storage, Replicas
- Ao implementar retry, backoff, DLQ ou redelivery
- Ao desenhar idempotencia e deduplicacao (`Nats-Msg-Id`)
- Ao revisar handlers para garantir ack/nak/term corretos
- Ao configurar graceful shutdown, max_ack_pending, fetch batch, ack_wait

## 1. Core NATS vs JetStream — Quando Usar

| Caracteristica | Core NATS | JetStream |
|---|---|---|
| Entrega | At-most-once (fire & forget) | At-least-once (com ack) ou exactly-once (com dedup) |
| Persistencia | **Nao** | **Sim** (file ou memory store) |
| Replay | Nao | Sim |
| Ack/Nak | Nao | Sim |
| Multiplos consumers | Queue Group (efemero) | Consumer duravel (1 ou N processos) |
| Use case | Notificacoes, pub/sub volatil | Eventos de dominio, jobs, integracao |

**Regra pratica:** eventos de dominio que nao podem ser perdidos -> **JetStream**. Em duvida: JetStream.

## 2. Convencao de Subjects

Subjects sao **hierarquicos**, separados por ponto. Use **kebab-case** dentro de cada token.

| Padrao | Exemplo |
|---|---|
| `events.<dominio>.<evento>` | `events.user.created` |
| `cmd.<dominio>.<acao>` | `cmd.email.send` |
| `dlq.<consumer>` | `dlq.billing-on-user-created` |

Wildcards: `*` casa **um** token; `>` casa **um ou mais** (sempre no final).

### Regras
- Subjects sao case-sensitive; padronize minusculo.
- Nao inclua dados sensiveis no subject.
- Stream nomeado em UPPER_SNAKE: `EVENTS_USER`, `EVENTS_BILLING`.
- Consumer durable nomeado por **proposito do worker**: `billing-on-user-created`.

## 3. Conexao — Setup Resiliente

O projeto encapsula a conexao num wrapper `NatsClient` (ver `architecture.md`), construido sincronamente e conectado no startup (no event loop). Setup interno:

```python
# pkg/src/org_pkg/nats/conn.py
import nats
from nats.aio.client import Client


async def connect_nats(cfg: NatsConfig) -> Client:
    return await nats.connect(
        servers=cfg.nats_url,                 # "nats://a:4222,nats://b:4222"
        name=cfg.service_name,                # aparece no monitor — sempre setar
        max_reconnect_attempts=-1,            # reconectar indefinidamente
        reconnect_time_wait=2,                # segundos
        ping_interval=20,
        max_outstanding_pings=3,
        error_cb=_on_error,                   # nunca silenciar erros assincronos
        disconnected_cb=_on_disconnect,
        reconnected_cb=_on_reconnect,
        closed_cb=_on_closed,
    )
```

```python
async def _on_error(err: Exception) -> None:
    logger.error("nats async error", error=str(err))

async def _on_disconnect() -> None:
    logger.warning("nats disconnected")

async def _on_reconnect() -> None:
    logger.info("nats reconnected")

async def _on_closed() -> None:
    logger.warning("nats connection closed")
```

### Regras de conexao

| Regra | Detalhe |
|---|---|
| **1 conexao por processo** | `Client` e seguro para uso concorrente. Nao criar pool |
| **Sempre setar `name`** | Essencial para debug no monitor |
| **`max_reconnect_attempts=-1`** | Reconectar sempre em producao |
| **Callbacks de status** | Logar disconnect, reconnect, erros e close |
| **Drain no shutdown** | `await nc.drain()` (nao `nc.close()`) — processa inflight e fecha |
| **TLS em producao** | creds/NKey + TLS |

## 4. JetStream — Contextos

```python
js = nc.jetstream()          # publish/consume
jsm = nc.jsm()               # administracao (streams, consumers) — via nc.jsm()
```

- Payload sempre `bytes`: `json.dumps(envelope).encode()` no publish; `json.loads(msg.data)` no consume.

## 5. Stream — Configuracao

```python
from nats.js.api import RetentionPolicy, StorageType, StreamConfig
from nats.js.errors import NotFoundError


async def ensure_stream(jsm, name: str, subjects: list[str]) -> None:
    config = StreamConfig(
        name=name,
        subjects=subjects,                       # ex: ["events.user.>"]
        retention=RetentionPolicy.LIMITS,        # default, mais flexivel
        storage=StorageType.FILE,                # File em prod; Memory so para teste
        max_age=30 * 24 * 60 * 60,               # segundos (nats-py converte para nanos internamente)
        max_bytes=20 * 1024 * 1024 * 1024,       # 20 GB
        num_replicas=3,                          # R3 em prod (HA)
        duplicate_window=2 * 60,                 # janela de dedup por Nats-Msg-Id (segundos)
    )
    try:
        await jsm.stream_info(name)
        await jsm.update_stream(config)          # ja existe -> atualiza
    except NotFoundError:
        await jsm.add_stream(config)             # nao existe -> cria
```

**Atencao:** no `nats-py`, durations em `StreamConfig`/`ConsumerConfig` sao passadas em **segundos** (o client converte para nanos). Confirme na versao — nunca escreva o valor em nanos cru.

### Retention e Storage

| Policy | Comportamento | Quando usar |
|---|---|---|
| `LIMITS` | Mantem ate bater limite | 90% dos casos |
| `WORKQUEUE` | Apaga apos ack de qualquer consumer | Job queue com 1 consumer/mensagem |
| `INTEREST` | Mantem enquanto ha interesse | Pub/sub duravel |

- `StorageType.FILE` sempre em producao; `num_replicas=3` (requer cluster >= 3 nos).

## 6. Publisher — Padrao Robusto

O `Publisher` generico do `pkg/nats/` (ver `architecture.md`) monta o envelope e publica com `Nats-Msg-Id` para dedup:

```python
# pkg/src/org_pkg/nats/publisher.py
import json
from datetime import UTC, datetime
from uuid import uuid4


class Publisher:
    def __init__(self, nats_client: NatsClient, source: str) -> None:
        self._nats_client = nats_client
        self._source = source

    async def publish(self, subject: str, msg_id: str, data: object) -> None:
        envelope = {
            "id": str(uuid4()),
            "type": subject,
            "source": self._source,
            "timestamp": datetime.now(UTC).isoformat(),
            "data": data,
        }
        # js.publish e awaitavel — resolve com o PubAck do server
        await self._nats_client.js.publish(
            subject,
            json.dumps(envelope).encode(),
            headers={"Nats-Msg-Id": msg_id},   # dedup server-side
        )
```

### Regras de publish

| Regra | Por que |
|---|---|
| **Sempre `Nats-Msg-Id` com ID estavel** | Dedup automatica na `duplicate_window` |
| **`await js.publish(...)`** | Garante que a mensagem chegou ao stream (retorna PubAck) |
| **Publicar DEPOIS do commit do DB** | Nunca dentro de tx. Ver [[postgres-best-practices]] |
| **Payload JSON** | `json.dumps(...).encode()` |
| **Headers para metadados** | `Nats-Msg-Id`, `event-type`, `schema-version`, `traceparent` |

## 7. Multiplos Workers no Mesmo Subject

### 7.1 Core NATS — Queue Subscription (efemero, at-most-once)

```python
async def message_handler(msg) -> None:
    await handle(msg.data)

# Todas as replicas no MESMO queue group => NATS distribui round-robin
await nc.subscribe("events.user.created", queue="user-created-workers", cb=message_handler)
```

Use somente quando perder mensagens e aceitavel.

### 7.2 JetStream — Pull Consumer Durável Compartilhado (recomendado)

**Um unico consumer durável** no stream; todas as replicas fazem `fetch` desse mesmo consumer (efetivamente um "queue group durável"):

```python
from nats.js.api import AckPolicy, ConsumerConfig


async def ensure_consumer(jsm, stream: str) -> None:
    await jsm.add_consumer(
        stream,
        ConsumerConfig(
            durable_name="billing-on-user-created",   # nome estavel — mesmo em todas as replicas
            filter_subject="events.user.created",
            ack_policy=AckPolicy.EXPLICIT,             # OBRIGATORIO ack manual
            ack_wait=30,                               # segundos ate redelivery se sem ack
            max_deliver=5,                             # tentativas antes de descartar/DLQ
            max_ack_pending=256,                       # inflight (todas as replicas somadas)
        ),
    )
```

### 7.3 Worker — Loop de Consumo (pull + fetch)

```python
import asyncio
import nats.errors
from nats.aio.msg import Msg


async def consume_loop(js, stream: str, durable: str, handler, stop: asyncio.Event) -> None:
    psub = await js.pull_subscribe_bind(durable=durable, stream=stream)

    while not stop.is_set():
        try:
            msgs = await psub.fetch(batch=16, timeout=5)
        except nats.errors.TimeoutError:
            continue                      # sem mensagens na janela — segue o loop
        for msg in msgs:
            await handler(msg)            # handler faz ack/nak/term e nunca lanca
```

### 7.4 Escala Horizontal

**Nao crie um consumer por replica.** Crie **UM consumer durável** e suba **N replicas** apontando para ele (mesmo `durable_name`). Para aumentar paralelismo: mais replicas ou `max_ack_pending` maior. Para dois processamentos independentes do mesmo subject: **dois consumers** com nomes diferentes.

## 8. Ack, Nak, Term — Tabela de Decisao

| Metodo | Significado | Quando usar |
|---|---|---|
| `await msg.ack()` | Sucesso — nao redeliver | Handler concluiu com sucesso |
| `await msg.nak()` | Falha — redeliver (backoff do consumer) | Erro transitorio |
| `await msg.nak(delay=N)` | Redeliver apos N segundos | Erro transitorio com backoff |
| `await msg.term()` | Erro terminal — nao redeliver | Payload invalido, regra violada |
| `await msg.in_progress()` | "Ainda processando" — reseta ack_wait | Handler longo (heartbeat) |

### Handler correto (nunca deixa excecao escapar)

```python
async def handle_user_created(self, msg: Msg) -> None:
    logger = self._logger.bind(subject=msg.subject, msg_id=(msg.headers or {}).get("Nats-Msg-Id"))

    try:
        envelope = json.loads(msg.data)
        data = UserCreatedData.model_validate(envelope["data"])
    except (json.JSONDecodeError, KeyError, ValidationError) as err:
        logger.error("invalid payload", error=str(err))
        await msg.term()                 # corrompido — nao reenviar
        return

    try:
        await self._use_case.perform(CreateAccountInput(...))
    except Exception as err:             # noqa: BLE001 — boundary de mensageria
        logger.error("processing failed", error=str(err))
        await msg.nak()                  # transitorio — redelivery
        return

    await msg.ack()                      # so apos sucesso
```

Anti-patterns: esquecer o ack; ack **antes** de processar; nak em erro permanente (estoura `max_deliver`).

## 9. Idempotencia — Obrigatorio em At-Least-Once

JetStream entrega **pelo menos uma vez**. Toda redelivery/restart pode duplicar. O handler **deve** ser idempotente.

### Padrao: tabela de processados

```sql
CREATE TABLE "processedEvents" (
    "eventId"     TEXT        PRIMARY KEY,
    "consumer"    TEXT        NOT NULL,
    "processedAt" TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```python
async def handle(self, msg: Msg) -> None:
    event_id = (msg.headers or {}).get("Nats-Msg-Id")
    if not event_id:
        await msg.term()
        return

    async def tx(conn) -> None:
        claim = await conn.fetchval(
            'INSERT INTO "processedEvents" ("eventId", "consumer") VALUES ($1, $2) '
            'ON CONFLICT ("eventId") DO NOTHING RETURNING "eventId"',
            event_id, self._consumer_name,
        )
        if claim is None:
            return                       # duplicata — apenas ack
        await self._apply(conn, msg)

    await self._transactor.run_in_tx(tx)
    await msg.ack()
```

Outras estrategias: dedup por `Nats-Msg-Id` + `duplicate_window` (janela curta), operacao naturalmente idempotente (`UPDATE SET status='paid' WHERE id=$1`), UPSERT com chave de negocio.

## 10. DLQ (Dead Letter Queue)

JetStream nao tem DLQ embutida. Implemente publicando para `dlq.<consumer>` no `term()` (so `term` **depois** que o DLQ confirmou):

```python
try:
    await self._handler(msg)
    await msg.ack()
except PermanentError as err:
    try:
        await self._publish_dlq(msg, err)   # publica em dlq.<consumer> com headers de contexto
    except Exception:
        await msg.nak(delay=30)             # nao conseguiu DLQ — tenta de novo
        return
    await msg.term()                        # so term DEPOIS do DLQ confirmado
```

Crie um stream `DLQ` capturando `dlq.>` com retencao longa para inspecao/replay.

## 11. Graceful Shutdown

```python
stop = asyncio.Event()
loop = asyncio.get_running_loop()
for sig in (signal.SIGTERM, signal.SIGINT):
    loop.add_signal_handler(sig, stop.set)

await stop.wait()

# 1. para de puxar novas mensagens (a mensagem em andamento conclui)
consumer_loop.stop()
await consumer_loop.wait()

# 2. drain da conexao NATS (flush + close, processa inflight)
await nc.drain()

# 3. fecha pool PG
await database.disconnect()
```

- `await nc.drain()`, nunca `nc.close()`, no shutdown gracioso.
- Pare o loop de consumo **antes** do drain.
- `ack_wait` deve ser maior que o tempo de drain.

## 12. Observabilidade

- Logs estruturados (structlog) com `stream`, `consumer`, `seq`, `num_delivered`, `msg_id`, `subject`, `error`.
- Metricas por worker: recebidas, acked, naked, termed, duracao, pending (`consumer_info().num_pending`).
- Tracing W3C: propague `traceparent` via header; cada handler abre um span filho.

## 13. Autenticacao e Seguranca

| Mecanismo | Quando | Config (`nats.connect`) |
|---|---|---|
| **creds (JWT+NKey)** | Padrao producao | `user_credentials="/path/svc.creds"` |
| **NKey** | Account interno | `nkeys_seed="/path/seed.nk"` |
| **User/Password** | Dev local | `user="...", password="..."` |
| **Token** | Scripts | `token="..."` |
| **TLS** | Sempre em prod | `tls=ssl_context` |

- Nunca commitar `.creds`/seed/senha. TLS obrigatorio fora do localhost. Credencial por servico.

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

Para producao, cluster R3 (3 nos com `--cluster`/`--routes`).

## Checklist — Novo Stream

- [ ] Nome em UPPER_SNAKE (`EVENTS_USER`)
- [ ] `subjects` cobre apenas o dominio (sem `>` raiz)
- [ ] `storage=StorageType.FILE` em producao
- [ ] `num_replicas=3` em producao
- [ ] `retention=RetentionPolicy.LIMITS` na maioria
- [ ] `max_age`, `max_bytes` definidos
- [ ] `duplicate_window` configurado para dedup
- [ ] Padrao `ensure_stream` (stream_info -> update, exceto NotFoundError -> add)

## Checklist — Novo Consumer Durável

- [ ] `durable_name` em kebab-case, descreve o proposito
- [ ] `filter_subject` especifico (nao `>`)
- [ ] `ack_policy=AckPolicy.EXPLICIT`
- [ ] `ack_wait` >= maior timeout de handler esperado
- [ ] `max_deliver` finito (3-10), com DLQ
- [ ] `max_ack_pending` calibrado ao throughput
- [ ] `num_replicas=3` em producao

## Checklist — Novo Worker

- [ ] 1 conexao NATS por processo, com `nc.drain()` no shutdown
- [ ] Handlers com timeout proprio (`asyncio.timeout`) menor que `ack_wait`
- [ ] Ack/Nak/Term explicitos em todos os caminhos; handler nunca lanca
- [ ] Erros permanentes -> `term`/DLQ
- [ ] Idempotencia garantida (dedup, UPSERT, ou tabela de processados)
- [ ] Logs com `stream`, `consumer`, `seq`, `num_delivered`, `msg_id`
- [ ] Graceful shutdown: `stop` -> `nc.drain()` -> `database.disconnect()`

## Checklist — Multiplos Workers no Mesmo Subject

- [ ] **Um unico** consumer durável (nao um por replica)
- [ ] Todas as replicas usam o **mesmo** `durable_name` no `pull_subscribe_bind`
- [ ] `max_ack_pending` dimensionado para o **total** entre replicas
- [ ] Idempotencia **obrigatoria** — qualquer replica pode pegar a redelivery
- [ ] Dois processamentos paralelos -> **dois consumers** com nomes distintos

## Fontes

- https://docs.nats.io/nats-concepts/jetstream — conceitos JetStream
- https://docs.nats.io/nats-concepts/jetstream/consumers — Consumer config
- https://docs.nats.io/nats-concepts/jetstream/streams — Stream config
- https://github.com/nats-io/nats.py — cliente oficial Python (nats-py)
- https://nats-io.github.io/nats.py/ — API reference do nats-py (JetStream, KV, Object Store)
- https://docs.nats.io/running-a-nats-service/configuration/securing_nats — auth/TLS
