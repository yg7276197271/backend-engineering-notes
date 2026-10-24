<a id="top"></a>
# 2. Event-Driven Architecture & Kafka

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
- [2.1 What is event-driven architecture](#s2-1)
- [2.2 Kafka architecture](#s2-2)
- [2.3 Cluster setup](#s2-3)
- [2.4 Producer internals and setup](#s2-4)
- [2.5 Producer idempotency](#s2-5)
- [2.6 Consumer setup](#s2-6)
- [2.7 Consumer exception handling](#s2-7)
- [2.8 Consumer advanced configuration](#s2-8)
- [2.9 Producer transactions](#s2-9)
- [2.10 Consumer transactions & exactly-once](#s2-10)

---

<a id="s2-1"></a>
### 2.1 What is event-driven architecture

**What:** Services communicate by **publishing events** (facts that already happened: `OrderPlaced`) instead of calling each other directly. Consumers react on their own schedule.

**Key ideas:**
- **Event vs command:** an event says "this happened" (past tense, many listeners). A command says "do this" (one handler).
- **Event notification** (thin event, consumers call back for details) vs **event-carried state transfer** (the event carries the data, so consumers need no callback).
- **Choreography:** services react to each other's events. **Orchestration:** a coordinator tells each service what to do.

**Benefits:** loose coupling, independent scaling, natural audit trail, easy to add new consumers.
**Costs:** eventual consistency, harder debugging (needs tracing), duplicate and out-of-order messages, schema evolution.

**Pitfalls:**
- Using events for request/response flows that need an immediate answer.
- Breaking consumers with schema changes. Use a schema registry (Avro/Protobuf) with compatibility rules.

**Interview Q&A:**
- *When would you not use EDA?* When the caller needs an immediate, consistent answer (like a balance check before a debit), or for simple CRUD.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-2"></a>
### 2.2 Kafka architecture

**What:** A distributed, partitioned, replicated **append-only commit log**.

**Building blocks:**
- **Topic** → split into **partitions** (the unit of parallelism and ordering). Each record has an **offset** within its partition.
- **Key → partition:** `hash(key) % partitions`. The same key always lands in the same partition, so it keeps its order.
- **Broker:** a server that stores partitions. Each partition has one **leader** and N **followers** (replication factor).
- **ISR (in-sync replicas):** followers caught up with the leader. Only ISR members can become leader.
- **Retention:** by time or size, or **compaction** (keep the latest value per key, used for state topics).
- **Consumer groups:** each partition is consumed by exactly one consumer in a group. Several groups each get every message.

**Pitfalls:**
- Ordering is only **per partition**. Global ordering needs one partition, which caps throughput.
- Adding partitions later changes `hash % n`, so existing keys move and per-key ordering breaks across the change.
- Hot keys (one huge customer) overload a single partition.

**Interview Q&A:**
- *How many consumers can work in parallel?* At most the number of partitions per group. Extra consumers sit idle.
- *Kafka vs RabbitMQ?* Kafka is a durable log you can replay, with very high throughput. RabbitMQ is a broker with flexible routing where messages are deleted once acknowledged.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-3"></a>
### 2.3 Cluster setup

**What:** Running several brokers together for fault tolerance and scale.

**How it works:**
- **Controller:** manages partition leadership and metadata. Older Kafka used **ZooKeeper**. Kafka 3.3+ uses **KRaft** (built-in Raft quorum), and ZooKeeper was removed entirely in Kafka 4.0.
- Typical production setup: 3+ brokers, `replication.factor=3`, `min.insync.replicas=2`, brokers spread across availability zones (`broker.rack` for rack-aware placement).
- Local dev: Docker Compose with one KRaft broker.

```bash
kafka-topics.sh --create --topic orders --partitions 6 --replication-factor 3 --bootstrap-server localhost:9092
```

**Pitfalls:**
- `replication.factor=3` with `min.insync.replicas=3` means one broker down blocks all `acks=all` writes.
- Under-replicated partitions are an early warning sign. Alert on them.

**Interview Q&A:**
- *What does KRaft change?* No separate ZooKeeper cluster to run, faster controller failover, and support for far more partitions.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-4"></a>
### 2.4 Producer internals and setup

**What happens on `send()`:**
1. Serialise key and value → pick a partition (the key's hash, or the **sticky partitioner** for null keys).
2. Append to a per-partition **batch** in the record accumulator (memory pool `buffer.memory`).
3. A background **sender thread** ships batches when they're full (`batch.size`) or after `linger.ms`.
4. The broker acknowledges according to `acks`. The callback or `Future` completes.

**Key settings:**
| Setting | Meaning |
|---|---|
| `acks=0/1/all` | No ack / leader only / all ISR. `all` = durable |
| `linger.ms`, `batch.size` | Larger batches mean better throughput and slightly more latency |
| `compression.type` | lz4/zstd: less network and disk |
| `retries`, `delivery.timeout.ms` | How long to keep retrying |
| `max.in.flight.requests.per.connection` | ≤ 5 keeps ordering with idempotence on |

```java
kafkaTemplate.send("orders", order.id(), event)
    .whenComplete((res, ex) -> { if (ex != null) log.error("publish failed", ex); });
```

**Pitfalls:**
- Fire-and-forget without checking the result means silent data loss.
- `acks=1` loses data if the leader dies before followers copy the record.

**Interview Q&A:**
- *How do you get high throughput?* Batching (`linger.ms` 5–20), compression, async sends, enough partitions.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-5"></a>
### 2.5 Producer idempotency

**Problem:** the producer sends, the broker writes, the ack is lost, the producer retries, and now there's a **duplicate**.

**How idempotence fixes it:**
- The broker gives each producer a **Producer ID (PID)**. Each record carries a per-partition **sequence number**.
- The broker rejects a sequence it has already written and acks it as a duplicate.
- `enable.idempotence=true` (the **default since Kafka 3.0**) needs `acks=all`, `retries > 0`, `max.in.flight ≤ 5`.

**What it guarantees and what it doesn't:**
- ✅ No duplicates from **producer retries**, per partition, within one producer session.
- ❌ Not across a producer **restart** (new PID). That needs a `transactional.id`.
- ❌ Not duplicates from your **application** calling `send()` twice, and not duplicate **consumer** processing.

**Interview Q&A:**
- *Is idempotent producer = exactly-once?* No. It's exactly-once **writes** per partition. End-to-end exactly-once also needs transactions or idempotent consumers.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-6"></a>
### 2.6 Consumer setup

**How a consumer works:**
- Joins a **consumer group** (`group.id`). The group coordinator assigns partitions.
- `poll()` loop: fetch records → process → **commit offsets** (where to resume after a restart).
- **Rebalance** when consumers join or leave or partitions change. Partitions get reassigned.

**Spring Kafka:**
```java
@KafkaListener(topics = "orders", groupId = "billing", concurrency = "3")
public void onOrder(OrderEvent e, Acknowledgment ack) {
    billing.process(e);
    ack.acknowledge();        // manual commit after success
}
```

**Commit strategies:**
| Strategy | Delivery | Risk |
|---|---|---|
| Auto-commit (`enable.auto.commit`) | Can be either | Commit before processing = loss; crash after processing = duplicate |
| Commit **after** processing | **At-least-once** | Duplicates on crash, so make processing idempotent |
| Commit **before** processing | At-most-once | Loss on crash |

**Pitfalls:**
- `concurrency` above the partition count leaves threads idle.
- Slow processing past `max.poll.interval.ms` makes the consumer get kicked out, which triggers a rebalance and duplicates.

**Interview Q&A:**
- *What is `auto.offset.reset`?* Where a **new** group starts: `earliest` (from the beginning) or `latest` (new messages only).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-7"></a>
### 2.7 Consumer exception handling

**Problem:** one bad message (a "poison pill") fails forever and **blocks its partition**.

**Tools (Spring Kafka):**
- **`DefaultErrorHandler`** with a `BackOff`: retry in place a few times, then hand off to a recoverer.
- **`DeadLetterPublishingRecoverer`:** publish the failed record to `orders.DLT` with exception headers, then move on.
- **Non-blocking retries** (`@RetryableTopic`): failed records go to `orders-retry-1`, `orders-retry-2` … with growing delays, so the main partition keeps flowing.
- **Deserialisation errors:** wrap the deserializer in `ErrorHandlingDeserializer` so a corrupt payload doesn't crash the loop.

```java
@RetryableTopic(attempts = "4", backoff = @Backoff(delay = 1000, multiplier = 2),
                exclude = ValidationException.class)
@KafkaListener(topics = "orders")
public void handle(OrderEvent e) { ... }
```

**Pitfalls:**
- Retrying non-transient errors (validation, bad data) just wastes time. Exclude them and send them straight to the DLT.
- Non-blocking retries **break per-key ordering**: a retried message can be overtaken by later ones.
- A DLT nobody monitors is just slower data loss. Alert on it and build a replay tool.

**Interview Q&A:**
- *Blocking vs non-blocking retries?* Blocking keeps order but stalls the partition. Non-blocking keeps throughput but loses ordering for retried messages.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-8"></a>
### 2.8 Consumer advanced configuration

| Setting | Use |
|---|---|
| `max.poll.records` | Records per poll. Lower it if processing is slow |
| `max.poll.interval.ms` | Max time between polls before the consumer is evicted |
| `session.timeout.ms` / `heartbeat.interval.ms` | Liveness detection |
| `fetch.min.bytes` / `fetch.max.wait.ms` | Trade latency for fewer, bigger fetches |
| `partition.assignment.strategy` | **CooperativeSticky** avoids stop-the-world rebalances |
| `group.instance.id` | **Static membership**: restarts within the timeout don't trigger a rebalance |
| `isolation.level` | `read_committed` hides aborted transactional writes |

**Patterns:**
- **Batch listeners** for bulk DB inserts. Handle partial failures with `BatchListenerFailedException`.
- **Pause/resume** partitions when a downstream is overloaded (back-pressure).
- Watch **consumer lag** (end offset − committed offset) as the key health metric.

**Interview Q&A:**
- *How do you reduce rebalance pain during deployments?* Cooperative sticky assignment plus static membership, and processing that fits well inside `max.poll.interval.ms`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-9"></a>
### 2.9 Producer transactions

**What:** Atomically write to **several partitions/topics**, optionally together with **consumer offsets**: all or nothing.

**How it works:**
- Set `transactional.id` (stable per producer instance). This enables idempotence and **fences zombies**: a restarted producer with the same ID bumps an epoch and the old one gets rejected.
- `initTransactions()` → `beginTransaction()` → sends → `sendOffsetsToTransaction()` → `commitTransaction()` / `abortTransaction()`.
- A **transaction coordinator** writes commit/abort markers. `read_committed` consumers only see committed data.

```java
kafkaTemplate.executeInTransaction(ops -> {
    ops.send("payments", id, payment);
    ops.send("ledger", id, entry);
    return true;
});
```

**Pitfalls:**
- It covers **Kafka only**. A DB write in the same method is **not** part of the Kafka transaction (that's the dual-write problem, see [7.15](07-hld.md#s7-15)).
- It adds latency (commit markers), so don't use it everywhere by default.

**Interview Q&A:**
- *What is zombie fencing?* After a network partition, the old instance of a producer is blocked from committing because its epoch is stale.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s2-10"></a>
### 2.10 Consumer transactions & exactly-once

**Read-process-write inside Kafka (consume → transform → produce):**
- Consume, produce the results **and** commit the input offsets in **one producer transaction**. Downstream consumers use `read_committed`.
- Kafka Streams does all of this with `processing.guarantee=exactly_once_v2`.

**Kafka → database (the common real case):**
- Kafka transactions can't include your database. Use **idempotent consumers** instead:
  - Store the processed `eventId` in a `processed_events` table in the **same DB transaction** as the business write. Skip the event if it's already there.
  - Or make the write naturally idempotent (upsert by business key).

```java
@Transactional
public void on(PaymentEvent e) {
    if (processedRepo.existsById(e.eventId())) return;   // duplicate
    ledger.post(e);
    processedRepo.save(new Processed(e.eventId()));
}
```

**Interview Q&A:**
- *How do you get exactly-once into Postgres?* At-least-once delivery + idempotent consumer (dedupe table or upsert) in one DB transaction, then commit the offset after it.
- *Why not exactly-once everywhere?* It costs latency and complexity, and only covers Kafka-to-Kafka. Idempotency is simpler and works for any sink.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
