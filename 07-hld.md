<a id="top"></a>
# 7. High-Level Design

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
**Networking & traffic**
- [7.1 Network protocols](#s7-1) · [7.2 DNS](#s7-2) · [7.3 Proxy vs reverse proxy](#s7-3) · [7.4 Load balancers and algorithms](#s7-4)

**Estimation & scaling**
- [7.5 Back-of-the-envelope estimation](#s7-5) · [7.6 Scale from zero to millions](#s7-6)

**Data**
- [7.7 CAP theorem (and PACELC)](#s7-7) · [7.8 SQL vs NoSQL](#s7-8) · [7.9 Database indexing](#s7-9) · [7.10 Consistent hashing](#s7-10) · [7.11 Design a key-value store](#s7-11)

**Caching**
- [7.12 Distributed cache and caching strategies](#s7-12) · [7.13 Thundering herd](#s7-13)

**Consistency & transactions**
- [7.14 Distributed transactions: 2PC and saga](#s7-14) · [7.15 Dual-write problem and transactional outbox](#s7-15) · [7.16 Concurrency control in distributed systems](#s7-16) · [7.17 Two-phase locking (2PL)](#s7-17) · [7.18 Idempotent POST API](#s7-18)

**Messaging & limits**
- [7.19 Distributed messaging queue](#s7-19) · [7.20 Rate limiter](#s7-20)

**Classic designs**
- [7.21 Payment system](#s7-21) · [7.22 URL shortener](#s7-22) · [7.23 WhatsApp / chat](#s7-23) · [7.24 Ticket booking](#s7-24) · [7.25 Video calling app](#s7-25)

**Architecture**
- [7.26 High availability](#s7-26) · [7.27 Microservice patterns](#s7-27) · [7.28 How many microservices?](#s7-28) · [7.29 API gateway (system view)](#s7-29) · [7.30 Service mesh](#s7-30)

**Security**
- [7.31 Symmetric and asymmetric encryption](#s7-31) · [7.32 OAuth 2.0 flows](#s7-32) · [7.33 JWT explained](#s7-33)

---

<a id="s7-1"></a>
### 7.1 Network protocols

| Protocol | Layer | Key traits | Use |
|---|---|---|---|
| **TCP** | Transport | Connection (3-way handshake), reliable, ordered, flow and congestion control | HTTP, DBs, most APIs |
| **UDP** | Transport | Connectionless, no delivery or order guarantee, low overhead | Video/voice, DNS, gaming |
| **HTTP/1.1** | App | Text, keep-alive, one request at a time per connection (head-of-line blocking) | Web |
| **HTTP/2** | App | Binary, **multiplexed** streams on one connection, header compression | gRPC, modern web |
| **HTTP/3** | App | Over **QUIC** (UDP): faster handshakes, no TCP head-of-line blocking | Mobile, lossy networks |
| **WebSocket** | App | Full-duplex, long-lived, starts as an HTTP upgrade | Chat, live updates |
| **SSE** | App | Server → client push over HTTP | Live feeds, notifications |
| **gRPC** | App | HTTP/2 + Protobuf, streaming, typed contracts | Service-to-service |

**Real-time options compared:** short polling (wasteful) → long polling (the server holds the request) → **SSE** (one-way push, simple) → **WebSocket** (two-way).
**TLS:** encrypts TCP. The handshake uses asymmetric crypto to agree a symmetric session key ([7.31](#s7-31)).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-2"></a>
### 7.2 DNS

**Resolution:** browser cache → OS cache → **recursive resolver** (ISP, 8.8.8.8) → **root** server → **TLD** server (.com) → **authoritative** server → IP. Each answer is cached for its **TTL**.

**Record types:** `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias), `MX` (mail), `TXT` (verification, SPF), `NS` (name servers).

**DNS in system design:**
- **GeoDNS / latency-based routing:** send users to the nearest region.
- **DNS failover:** health-checked records (Route 53). A low TTL makes failover faster, but means more queries.
- Weakness: caching means changes propagate slowly, and some clients ignore TTLs.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-3"></a>
### 7.3 Proxy vs reverse proxy

| | Forward proxy | Reverse proxy |
|---|---|---|
| Sits in front of | **Clients** | **Servers** |
| Hides | The client's identity | The server topology |
| Uses | Corporate filtering, caching, anonymity | **TLS termination**, load balancing, caching, compression, WAF, routing |
| Examples | Squid, corporate proxy | **Nginx**, HAProxy, Envoy, Cloudflare |

An API gateway is a specialised reverse proxy with API-aware features (auth, rate limits).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-4"></a>
### 7.4 Load balancers and algorithms

**L4 vs L7:**
- **L4** (transport): routes by IP and port, very fast, can't see HTTP.
- **L7** (application): routes by URL, headers or cookies, can terminate TLS, do content-based routing, add sticky sessions.

**Algorithms:**
| Algorithm | Behaviour | Good for |
|---|---|---|
| Round robin | Rotate through servers | Uniform servers and requests |
| Weighted round robin | More traffic to bigger servers | Mixed capacity |
| **Least connections** | Server with the fewest active connections | Long or variable requests |
| Least response time | Fastest + least loaded | Latency-sensitive |
| **IP hash** | Same client → same server | Simple stickiness |
| **Consistent hashing** | Key → server on a ring | Caches, stateful sharding ([7.10](#s7-10)) |

**Health checks** take dead servers out of rotation. **High availability for the LB itself:** active–passive pair with a floating IP, or managed cloud LBs.
**Sticky sessions** are an anti-pattern for scaling. Keep app servers **stateless** (sessions in Redis).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-5"></a>
### 7.5 Back-of-the-envelope estimation

**Handy numbers:**
- 1 day ≈ **86,400 s ≈ 10⁵ s**. 1 million requests/day ≈ **12 QPS**.
- Peak ≈ **2–3× average** (up to 10× for spiky events).
- Latencies: memory ~100 ns, SSD read ~100 µs, same-datacentre round trip ~0.5 ms, disk seek ~10 ms, cross-continent ~150 ms.
- Sizes: char ≈ 1 byte (ASCII), UUID 16 bytes, typical row 0.5–1 KB, image ~200 KB–2 MB.

**Template:**
1. **Users:** DAU, actions per user per day.
2. **QPS:** `DAU × actions / 10⁵` → peak ×3. Split reads and writes (often 10:1 or 100:1).
3. **Storage:** `writes/day × size × retention (days) × replication (3)`.
4. **Bandwidth:** `QPS × payload size`.
5. **Cache:** about 20% of daily reads cover 80% of traffic (Pareto).

**Example:** 100M DAU × 10 reads = 10⁹/day → ~10K QPS average, ~30K peak. 1M writes/day × 1 KB × 365 × 5 years ≈ 1.8 TB (×3 replicas ≈ 5.5 TB).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-6"></a>
### 7.6 Scale from zero to millions

1. **Single server:** app + DB on one box.
2. **Separate DB server:** scale them independently.
3. **Load balancer + multiple stateless app servers:** horizontal scaling. Sessions move to Redis.
4. **Cache** (Redis) for hot reads → much less DB load.
5. **CDN** for static assets and media near users.
6. **DB replication:** primary for writes, **read replicas** for reads (watch replication lag).
7. **Message queues:** move slow work (email, image processing) out of the request path.
8. **Sharding / partitioning:** split data across DBs by key once one primary can't handle the writes.
9. **Multiple data centres / regions:** GeoDNS, data replication, failover.
10. **Observability and automation:** metrics, logs, tracing, autoscaling, CI/CD.

**Principle:** scale **only the bottleneck you've measured**. Each step adds operational cost.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-7"></a>
### 7.7 CAP theorem (and PACELC)

**CAP:** in a distributed data store, when a **network partition (P)** happens, you must choose **consistency (C)** (every read sees the latest write, or errors) or **availability (A)** (every request gets a non-error response, possibly stale).
- P isn't optional over real networks, so the real choice is **CP vs AP during partitions**.
- **CP examples:** ZooKeeper, etcd, HBase, banking ledgers. **AP examples:** Cassandra, DynamoDB (default), DNS, shopping carts.

**PACELC:** **if P** → choose A or C; **Else** (normal operation) → choose **L**atency or **C**onsistency. Example: DynamoDB is PA/EL, Spanner is PC/EC.

**Consistency models:** strong (linearizable) → read-your-writes → monotonic reads → causal → **eventual**.

**Interview angle:** choose per feature. Payments and inventory reservation need CP behaviour. Product catalogue, likes and recommendations can be AP.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-8"></a>
### 7.8 SQL vs NoSQL

| | SQL (Postgres, MySQL) | NoSQL |
|---|---|---|
| Model | Tables, relations, fixed schema | Key-value (Redis, DynamoDB), document (MongoDB), wide-column (Cassandra), graph (Neo4j) |
| Transactions | **ACID**, multi-row/table | Often single-item or limited. Many give **BASE** (eventual) |
| Queries | Rich SQL, joins, ad hoc | Designed around **access patterns**. Few or no joins |
| Scaling | Vertical + read replicas. Sharding is hard | **Horizontal** by design (partitioning) |
| Schema change | Migrations | Flexible |

**Choose SQL when:** relational data, strong consistency, complex queries, money.
**Choose NoSQL when:** huge scale with simple access patterns (by key), flexible or semi-structured data, very high write throughput (time series, events), graph traversals.
Many systems use both (**polyglot persistence**).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-9"></a>
### 7.9 Database indexing

**What:** an auxiliary structure that finds rows without a full table scan. **Faster reads, slower writes, more storage.**

**Types:**
- **B+ tree** (default): sorted, O(log n), supports **equality and range** queries and ORDER BY.
- **Hash index:** equality only.
- **Composite index** `(a, b, c)`: usable for `a`, `a,b`, `a,b,c` (the **leftmost-prefix rule**), but not `b` alone.
- **Covering index:** includes every column the query needs, so there's no table lookup ("index-only scan").
- **Clustered** (rows stored in index order; one per table, e.g. the InnoDB primary key) vs **non-clustered** (separate structure pointing to rows).
- Others: full-text, GIN (JSONB/arrays), geospatial (R-tree), bitmap.

**Pitfalls:** functions on indexed columns (`WHERE lower(email) = ?`) skip the index unless you have an expression index. Low-cardinality columns rarely benefit. Too many indexes slow down writes.
**Tooling:** `EXPLAIN (ANALYZE)` to check the plan actually uses the index.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-10"></a>
### 7.10 Consistent hashing

**Problem:** `hash(key) % N` remaps **almost every key** when N changes, so adding a cache node wipes the cache.

**Solution:** put servers and keys on a **hash ring** (0 … 2³²). A key belongs to the **first server clockwise**.
- Adding or removing a node moves only **~1/N of the keys** (its neighbour's range).
- **Virtual nodes:** each physical server gets 100–200 points on the ring, which evens out the load and spreads a failed node's keys across many servers.
- **Replication:** store each key on the next R distinct servers clockwise.

**Used by:** Cassandra, DynamoDB, Memcached clients, CDNs, load balancers for stateful routing. (Redis Cluster uses a variant: 16,384 fixed hash slots.)

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-11"></a>
### 7.11 Design a key-value store

**Requirements:** `put(k, v)`, `get(k)`. Huge scale, high availability, tunable consistency, low latency.

**Design (Dynamo-style):**
- **Partitioning:** consistent hashing with virtual nodes ([7.10](#s7-10)).
- **Replication:** N replicas on the next nodes of the ring, across availability zones.
- **Quorum:** write to W replicas, read from R. **R + W > N** gives read-your-latest-write (e.g. N=3, W=2, R=2).
- **Conflict resolution:** last-write-wins (timestamps) or **vector clocks** + client merge.
- **Failure handling:** **hinted handoff** (another node holds writes for a down node), **read repair**, background **anti-entropy with Merkle trees**.
- **Membership and failure detection:** a **gossip** protocol.
- **Storage engine:** write-ahead log → **memtable** → flushed to **SSTables**, compaction, and **Bloom filters** to skip SSTables that can't contain a key (LSM tree).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-12"></a>
### 7.12 Distributed cache and caching strategies

**Where caches live:** client/browser → CDN → API gateway → app (in-process, e.g. Caffeine) → **distributed** (Redis, Memcached) → DB buffer pool.

**Read strategies:**
- **Cache-aside (lazy loading):** the app checks the cache → on a miss reads the DB → populates the cache. Most common. Only caches what's requested.
- **Read-through:** the cache library loads from the DB itself on a miss.

**Write strategies:**
| Strategy | How | Trade-off |
|---|---|---|
| **Write-through** | Write cache + DB synchronously | Consistent, slower writes |
| **Write-behind (write-back)** | Write cache, flush to DB asynchronously | Fast, **risk of loss** |
| **Write-around** | Write DB only, cache on the next read | Avoids caching cold writes |
| Invalidate on write | Write DB, **delete** the cache key | Simple, common with cache-aside |

**Eviction:** LRU, LFU, FIFO, plus **TTL**. **Two-level caching:** L1 in-process + L2 Redis.
**Hard parts:** invalidation across services, stale reads after writes, hot keys, cache penetration (repeated misses for non-existent keys → cache the "null" or use a Bloom filter), stampedes ([7.13](#s7-13)).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-13"></a>
### 7.13 Thundering herd

**What:** many requests hit a backend **at the same moment**. Typical triggers: a hot cache key expires (**cache stampede**), many clients retry together, or everything reconnects after an outage.

**Fixes:**
- **Request coalescing / single-flight:** only one request rebuilds a key, and the others wait for its result.
- **Mutex per key:** `SET lock:key NX PX 5000`. The lock winner refreshes, and the rest serve stale data or wait briefly.
- **Stale-while-revalidate:** serve the expired value while one background refresh runs.
- **Jittered TTLs:** don't let thousands of keys expire at the same second.
- **Probabilistic early refresh:** refresh shortly before expiry with a rising probability.
- **Retries with exponential backoff + jitter**, plus load shedding and rate limiting at the edge.
- **Cache warming** before traffic spikes (a sale launch).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-14"></a>
### 7.14 Distributed transactions: 2PC and saga

**Two-phase commit (2PC):**
1. **Prepare:** the coordinator asks every participant "can you commit?" They lock resources and vote yes or no.
2. **Commit/abort:** if all vote yes, commit everywhere. Otherwise abort.
- ✅ Atomic. ❌ **Blocking** (participants hold locks while waiting), the coordinator is a single point of failure, it doesn't scale, and it's rarely supported across microservices or NoSQL. (3PC reduces blocking but is rarely used.)

**Saga:** a sequence of **local transactions**, each with a **compensating action** that undoes it.
- **Choreography:** services react to each other's events (OrderCreated → PaymentDone → StockReserved). Loose coupling, but the flow is hard to see.
- **Orchestration:** a central orchestrator (Temporal, Camunda, a state machine) calls each step and triggers compensations. Clear flow and easier to monitor.
- Consistency is **eventual**. Compensations must be **idempotent**. Design for "semantic" rollback (refund, not delete).

**Interview angle:** order → payment → inventory → shipping. If inventory fails: refund the payment, cancel the order.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-15"></a>
### 7.15 Dual-write problem and transactional outbox

**Problem:** a service must **update its DB and publish an event**. These are two systems with no shared transaction:
- DB commit succeeds, the publish fails (crash) → the event is **lost**.
- Publish succeeds, the DB commit fails → a **phantom** event for data that doesn't exist.

**Transactional outbox:**
1. In the **same DB transaction** as the business change, insert the event into an `outbox` table.
2. A relay publishes outbox rows to Kafka:
   - **Polling publisher:** periodically reads unsent rows, publishes, marks them sent, or
   - **CDC (Debezium):** tails the DB transaction log and streams outbox inserts to Kafka (lower latency, no polling load).
3. Delivery is **at-least-once**, so consumers must be **idempotent** ([2.10](02-kafka-eda.md#s2-10)).

**Alternatives:** **listen to yourself** (publish only to Kafka, and the service consumes its own event to update the DB), or event sourcing.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-16"></a>
### 7.16 Concurrency control in distributed systems

**Pessimistic:** lock before access.
- DB row locks (`SELECT … FOR UPDATE`), **distributed locks** (Redis `SET NX PX`, ZooKeeper/etcd leases).
- Safe under high contention, but causes blocking, deadlocks, and a lock service to keep available.

**Optimistic (OCC):** no lock. Check for conflicts on write.
- A **version column**: `UPDATE … SET …, version = version + 1 WHERE id = ? AND version = ?`. Zero rows updated means a conflict, so retry or report it.
- Best for low contention (most CRUD).

**MVCC** (Postgres, InnoDB): readers see a snapshot, so readers don't block writers.

**Distributed lock safety:** use a lease **TTL** (the lock frees itself if the holder dies) plus **fencing tokens** (a monotonically increasing number checked by storage) to stop a paused, expired holder from writing ([1.16](01-microservices.md#s1-16)).

**Choosing:** a short critical section on one row → OCC or a row lock. A long hold (a seat held for 10 minutes) → a lock with a TTL ([7.24](#s7-24)).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-17"></a>
### 7.17 Two-phase locking (2PL)

**Rule:** a transaction has a **growing phase** (acquires locks, releases none) and then a **shrinking phase** (releases locks, acquires none). This guarantees **serialisability**.
- **Shared (S) locks** for reads, **exclusive (X) locks** for writes. S is compatible with S, but X conflicts with everything.
- **Strict 2PL:** hold X locks until commit or abort. This prevents cascading aborts and is what databases actually implement.
- **Downsides:** reduced concurrency and **deadlocks**. Databases detect them (wait-for graph cycles) and abort one transaction, or prevent them with timeouts or ordering.
- Not to be confused with **2PC** (an atomic commit protocol across nodes, [7.14](#s7-14)).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-18"></a>
### 7.18 Idempotent POST API

**Problem:** the client sends `POST /payments`, the response times out, the client retries, and now there's a **double charge**.

**Design:**
1. The client generates an **`Idempotency-Key`** (a UUID) per logical operation and sends it as a header.
2. The server stores `key → (request hash, status, response)` in a table or Redis with a TTL (e.g. 24 h), with a **unique constraint** on the key.
3. On request:
   - **New key:** insert as IN_PROGRESS (the unique insert acts as the lock) → process → store the response.
   - **Same key, completed:** return the **stored response**. Don't process again.
   - **Same key, in progress:** return 409 or 425, or wait.
   - **Same key, different payload:** return 422 (key reuse error).
4. Downstream calls pass the key along (PSPs like Stripe support it).

**Note:** GET, PUT and DELETE are idempotent by HTTP semantics. POST and PATCH need this mechanism.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-19"></a>
### 7.19 Distributed messaging queue

**Why queues:** decouple producers from consumers, absorb spikes (buffering), async processing, retries, fan-out.

**Two models:**
- **Queue / point-to-point:** each message goes to **one** consumer (work distribution). Example: SQS, RabbitMQ queue.
- **Pub/sub:** each subscriber gets **every** message (fan-out). Example: SNS, Kafka consumer groups.

**Designing one (Kafka-like):**
- **Topics → partitions** (append-only log segments on disk, sequential I/O, page cache, zero-copy sends).
- **Replication** with leader/followers and ISR. `acks` for durability.
- **Consumer offsets** stored by the broker. Consumers pull at their own pace (back-pressure for free).
- **Retention** by time or size, compaction for keyed state.
- **Delivery semantics:** at-most-once / at-least-once / exactly-once ([2.5](02-kafka-eda.md#s2-5), [2.10](02-kafka-eda.md#s2-10)).
- **Extras:** dead-letter queues, delayed messages, ordering per key, schema registry.

**Kafka vs RabbitMQ vs SQS:** a replayable high-throughput log vs a smart router with per-message acks vs a managed simple queue.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-20"></a>
### 7.20 Rate limiter

**Where:** API gateway or middleware, keyed by user ID, API key or IP. Return **429** with `Retry-After` and `X-RateLimit-*` headers.

**Algorithms:**
| Algorithm | Idea | Trade-off |
|---|---|---|
| Fixed window counter | Count per minute bucket | Boundary bursts (2× at the window edge) |
| Sliding window log | Store each timestamp | Exact, memory-heavy |
| **Sliding window counter** | Weighted previous + current window | Close approximation, 2 counters |
| **Token bucket** | Tokens refill at rate r, bucket size b allows bursts | Most popular |
| Leaky bucket | Requests drain at a fixed rate | Smooths output, no bursts |

**Distributed:** state in **Redis**, updated **atomically with a Lua script** (read → refill → decrement in one step). Shard by client key (Redis Cluster). Choose **fail-open vs fail-closed** if Redis is down. Use local pre-checks to cut Redis round trips.
Implementation view: [Resilience4j rate limiter 1.6](01-microservices.md#s1-6).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-21"></a>
### 7.21 Payment system

**Core flow:** create a **payment intent** → collect card details **directly** via the PSP's SDK or iframe (they never touch the merchant's servers, which reduces PCI scope) → charge through the card network → update the intent and transaction status **atomically** → notify the merchant via a webhook.

**Key design decisions:**
- **Idempotency keys** on every create and charge ([7.18](#s7-18)).
- **Record the attempt before calling the network.** Status `PENDING`. On a timeout, **don't guess**: a reconciliation service queries the network and processes **settlement files**.
- **Ledger:** double-entry, append-only (debits = credits). Never update balances in place without an entry.
- **Durability and audit:** CDC → Kafka → immutable audit store.
- **Security:** tokenise cards, keys in an HSM, signed webhooks, request signing with nonce + timestamp (anti-replay).
- **Scale:** shard by merchant ID. Async webhooks with retries and backoff.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-22"></a>
### 7.22 URL shortener

**Requirements:** shorten a long URL, redirect fast, optional custom alias and expiry, analytics. Heavily read-dominant (100:1).

**ID generation:**
- **Counter + base62** (a-z A-Z 0-9): 7 characters ≈ 3.5 trillion IDs. Use a distributed counter (ranges handed out per server) or **Snowflake** IDs, so there are no collisions to check.
- Hash (MD5/SHA) + truncate → possible collisions, so you need a check-and-retry.

**Redirect:** **301** (permanent, browsers cache it, less load, worse analytics) vs **302** (temporary, every hit reaches you, better analytics).

**Architecture:** write API → KV store (short → long). Read path: **cache** (hot links) → DB. CDN or edge for the hottest links. Analytics via async events (Kafka) → OLAP store.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-23"></a>
### 7.23 WhatsApp / chat

**Requirements:** 1:1 and group messages, online delivery in real time, offline delivery later, sent/delivered/read receipts, media, multiple devices.

**Design:**
- Clients keep a **WebSocket** to a **chat server** (gateway).
- **Routing between servers:** a registry of user → server (Redis/ZooKeeper), or **Redis Pub/Sub** (each server subscribes to channels for its connected users).
- **Storage:** messages in a write-scalable store (Cassandra/DynamoDB, partitioned by chat ID, sorted by time). An **inbox table** per recipient device for undelivered messages. Delete from the inbox on client **ACK**.
- **Offline users:** on reconnect, drain the inbox. Push notification (APNs/FCM) when offline.
- **Media:** the client uploads straight to object storage (S3) via a **pre-signed URL**. Messages carry only the URL.
- **Groups:** fan-out on write to member inboxes (cap group size), or fan-out on read for very large groups.
- **Ordering:** per-chat sequence numbers. **End-to-end encryption** (Signal protocol) means the server only sees ciphertext.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-24"></a>
### 7.24 Ticket booking

**Requirements:** browse events, view the seat map, reserve seats for a few minutes, pay, confirm. **Never double-book.** Survive huge on-sale spikes.

**Design:**
- **Booking:** a two-step **reserve → confirm**. Postgres for tickets and seats (strong consistency).
- **Seat hold:** a **Redis lock with a TTL** (10 min) per seat. It expires by itself if the user abandons. (A `reserved` status + cron cleanup is the weaker alternative, because it misses edge cases.)
- **Confirm:** inside a DB transaction, check the lock is still held → insert the ticket → mark the seat booked (with a unique constraint on show + seat as the final guard).
- **Read scale:** cache event and venue data heavily. **Elasticsearch** (fed by CDC) for search by name, date and location.
- **Huge events:** a **virtual waiting room**: users queue (WebSocket) and are admitted in batches so the booking path isn't overwhelmed.
- **Live seat map:** SSE or WebSocket updates, or simply re-check on submit.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-25"></a>
### 7.25 Video calling app

**Requirements:** 1:1 and group video/audio, low latency (< 150 ms), screen share, works behind NAT, scales to many concurrent calls.

**Design:**
- **Signalling server** (WebSocket): exchanges SDP offers/answers and ICE candidates to set up the call. It doesn't carry media.
- **Media transport:** **WebRTC** over UDP (SRTP, encrypted).
- **NAT traversal:** **STUN** (discover the public address) → direct peer-to-peer if possible. Otherwise relay through **TURN** servers (costly bandwidth).
- **Topologies:**
  - **Mesh (P2P):** fine for 2–4 people. Upload grows with every participant.
  - **SFU (Selective Forwarding Unit):** each client uploads once, and the server forwards streams to the others. The standard for groups. **Simulcast** sends multiple qualities.
  - **MCU:** the server mixes streams into one. Cheap for clients, CPU-heavy for the server.
- **Adaptivity:** bandwidth estimation, dynamic resolution and bitrate, jitter buffers. Recording via a server-side SFU tap.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-26"></a>
### 7.26 High availability

**Measured in "nines":** 99.9% ≈ 8.8 h downtime/year, **99.99% ≈ 53 min/year**, 99.999% ≈ 5 min/year. A chain of dependencies **multiplies** availability (0.999 × 0.999 ≈ 0.998).

**Techniques:**
- **Redundancy at every tier:** multiple app instances, LB pairs, DB primary + replicas, **multi-AZ**, and multi-region for the highest tiers.
- **Failover:** active–passive (standby takes over) vs **active–active** (all serve traffic). Automated health checks + failover (DB failover, DNS failover).
- **Eliminate single points of failure** (one DB, one cache node, one region, one config server).
- **Graceful degradation:** circuit breakers, fallbacks, feature shedding. Serve stale data rather than nothing.
- **Deployment safety:** rolling, blue-green and canary deploys, with fast rollback.
- **RTO/RPO:** recovery time objective (how fast you're back) and recovery point objective (how much data you can lose). They drive the backup and replication strategy.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-27"></a>
### 7.27 Microservice patterns

| Category | Patterns |
|---|---|
| **Decomposition** | By business capability, by subdomain (DDD bounded contexts), **strangler fig** (migrate a monolith piece by piece) |
| **Data** | **Database per service**, shared DB (anti-pattern), **saga**, **CQRS** (separate read and write models), **event sourcing** (state = log of events), API composition |
| **Communication** | Sync (REST/gRPC), async messaging, **API gateway**, **BFF** |
| **Reliability** | Circuit breaker, retry, bulkhead, timeout, rate limiting ([section 1](01-microservices.md)) |
| **Discovery & deployment** | Service registry, client/server-side discovery, **sidecar**, service mesh |
| **Observability** | Log aggregation, distributed tracing, health check API, metrics |
| **Integration** | **Transactional outbox**, anti-corruption layer (ACL) to legacy systems |

**CQRS in one line:** writes go to a normalised model, and events update denormalised **read models** tuned for queries (often in another store, like Elasticsearch).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-28"></a>
### 7.28 How many microservices?

- **Split by business capability and team ownership** (Conway's law), not by entity noun (`CustomerService`, `AddressService` gives you a chatty, distributed monolith).
- **Signals to split:** a different scaling profile, a different release cadence, a separate team, a separate data owner, a clear bounded context.
- **Signals you split too far:** every feature changes 5 services, synchronous call chains 6 deep, distributed transactions everywhere, one team owning 20 services.
- **Start with a modular monolith** with clean module boundaries, and extract services when there's a concrete need.
- Rough guide: a service should be ownable by **one team** (the "two-pizza team").

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-29"></a>
### 7.29 API gateway (system view)

- **Responsibilities:** routing, authentication, rate limiting, TLS termination, request/response transformation, aggregation, caching, canary routing, observability at the edge.
- **Variants:** one gateway for everything, or **BFF** (one per client type: web, mobile, partners).
- **HA:** stateless replicas behind an LB. Rate-limit state in Redis.
- **Don't:** put business logic or long synchronous orchestration in it.
- Implementation details (Spring Cloud Gateway): [1.10](01-microservices.md#s1-10).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-30"></a>
### 7.30 Service mesh

**What:** an infrastructure layer that handles **service-to-service communication** through a **sidecar proxy** (Envoy) next to every service instance. Examples: Istio, Linkerd.

- **Data plane:** sidecars intercept all traffic → **mTLS** encryption and identity, retries, timeouts, circuit breaking, load balancing, telemetry.
- **Control plane:** pushes config and policy to the sidecars (traffic splitting for canaries, authorisation rules).
- **Benefit:** consistent resilience and security **without library code** in each language.
- **Cost:** extra latency per hop, resource overhead, operational complexity. (Sidecar-less "ambient" modes reduce this.)
- **vs API gateway:** the gateway handles **north–south** traffic (clients → services). The mesh handles **east–west** traffic (service ↔ service).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-31"></a>
### 7.31 Symmetric and asymmetric encryption

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | **One shared secret** | **Public + private** key pair |
| Speed | Fast | Slow (~1000× slower) |
| Algorithms | **AES-256-GCM**, ChaCha20 | RSA, **ECC** (ECDSA, ECDH, Ed25519) |
| Problem | Distributing the key safely | Performance |
| Use | Bulk data, data at rest, session traffic | Key exchange, **digital signatures**, certificates |

**How TLS combines them:** the server proves its identity with a **certificate** (public key signed by a CA) → client and server run **ECDHE** key exchange → derive a **symmetric session key** → all traffic uses AES.
**Signatures:** sign with the **private** key, verify with the **public** key (JWT RS256, code signing). **Encryption:** encrypt with the public key, decrypt with the private one.
**Hashing is not encryption:** SHA-256 is one-way (integrity). Passwords need slow hashes (BCrypt, Argon2).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-32"></a>
### 7.32 OAuth 2.0 flows

**Authorization code + PKCE** (users):
1. The app redirects the user to the **authorization server** with `client_id`, `redirect_uri`, `scope`, `state` and a **code_challenge** (hash of a random verifier).
2. The user logs in and consents → redirected back with a one-time **authorization code**.
3. The app exchanges code + **code_verifier** for an **access token** (+ refresh token, + ID token with OIDC).
4. The app calls the API with `Authorization: Bearer <access token>`. The resource server validates it.

**Client credentials** (machine-to-machine): the service sends client ID + secret (or a signed JWT/mTLS) → gets an access token. There's no user.
**Refresh token:** gets new access tokens without logging in again. Rotate it and detect reuse.
**Device code:** TVs and CLIs (like the GitHub CLI login).
**Deprecated:** implicit and password grants.

**Scopes** = what the token may do. **Audience** = which API it's for. **OIDC** adds identity on top.
Spring implementation: [3.36](03-spring-boot.md#s3-36).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s7-33"></a>
### 7.33 JWT explained

- **Structure:** `base64url(header).base64url(payload).signature`. The header has `alg`/`kid`. The payload has **claims**: `sub`, `iss`, `aud`, `exp`, `iat`, `jti`, plus custom ones (roles).
- **Signed (JWS), not encrypted:** anyone can decode the payload. Encrypted JWTs (JWE) exist but are rarer.
- **Validation:** check the signature (with the right key from **JWKS** by `kid`) → `exp`/`nbf` → `iss` → `aud` → only then trust the claims. **Reject `alg: none`** and algorithm confusion attacks.
- **Pros:** stateless verification, so any service can validate without a DB call.
- **Cons:** hard to revoke before expiry, so keep TTLs short, rotate refresh tokens, and use a deny-list for emergencies. Token size grows with the claims.
- Spring usage and storage advice: [3.35](03-spring-boot.md#s3-35).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
