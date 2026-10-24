<a id="top"></a>
# 8. Platform — Redis, AWS, Git

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
**Redis**
- [8.1 Redis data structures](#s8-1) · [8.2 Redis use cases](#s8-2) · [8.3 Persistence, replication and clustering](#s8-3) · [8.4 Distributed locks with Redis](#s8-4)

**AWS**
- [8.5 Compute](#s8-5) · [8.6 Storage](#s8-6) · [8.7 Databases](#s8-7) · [8.8 Messaging and integration](#s8-8) · [8.9 Networking](#s8-9) · [8.10 Security and IAM](#s8-10) · [8.11 Observability and well-architected](#s8-11)

**Git**
- [8.12 Git model and everyday commands](#s8-12) · [8.13 Branching strategies](#s8-13) · [8.14 Merge vs rebase](#s8-14) · [8.15 Undoing things](#s8-15)

---

<a id="s8-1"></a>
### 8.1 Redis data structures

| Type | Commands | Typical use |
|---|---|---|
| **String** | `GET/SET`, `INCR`, `SETNX`, `EX/PX` | Cache values, counters, locks |
| **Hash** | `HSET/HGET/HINCRBY` | Objects (user profile), token buckets |
| **List** | `LPUSH/RPOP`, `BLPOP` | Simple queues, latest-N feeds |
| **Set** | `SADD/SISMEMBER/SINTER` | Unique visitors, tags, friends-in-common |
| **Sorted set (ZSET)** | `ZADD/ZRANGE/ZRANK` | **Leaderboards**, priority queues, sliding-window rate limits |
| **Stream** | `XADD/XREADGROUP/XACK` | Durable event log with consumer groups |
| **Geo** | `GEOADD/GEOSEARCH` | Nearby drivers or stores |
| **HyperLogLog** | `PFADD/PFCOUNT` | Approximate unique counts with tiny memory |
| **Bitmap** | `SETBIT/BITCOUNT` | Daily active flags, feature flags per user |
| Pub/Sub | `PUBLISH/SUBSCRIBE` | Fire-and-forget fan-out (not durable) |

Single-threaded command execution makes each command **atomic**. **Lua scripts** and `MULTI/EXEC` make multi-step operations atomic.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-2"></a>
### 8.2 Redis use cases

- **Cache** (cache-aside, TTLs, LRU/LFU eviction via `maxmemory-policy`).
- **Session store** for stateless app servers.
- **Rate limiting:** token bucket in a hash, updated by a Lua script ([7.20](07-hld.md#s7-20)).
- **Distributed locks** and seat holds with a TTL ([8.4](#s8-4)).
- **Leaderboards** (ZSET), **counters** (INCR), **unique counts** (HyperLogLog).
- **Queues and streams:** lightweight job queues (Lists, or Streams with consumer groups).
- **Real-time messaging:** Pub/Sub to route chat messages between gateway servers ([7.23](07-hld.md#s7-23)).
- **Geo search:** nearby drivers or stores with `GEOSEARCH`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-3"></a>
### 8.3 Persistence, replication and clustering

**Persistence:**
- **RDB:** point-in-time snapshots. Compact, fast restarts, but you lose the writes since the last snapshot.
- **AOF:** append-only log of writes (`appendfsync everysec` loses at most ~1 s). Larger files, rewritten periodically.
- Common choice: both. Or none, for a pure cache.

**High availability:**
- **Replication:** primary → replicas (asynchronous, so the most recent writes can be lost on failover).
- **Sentinel:** monitors and fails over automatically for a non-clustered setup.
- **Redis Cluster:** shards data across **16,384 hash slots** spread over primaries, each with replicas. Multi-key operations need keys in the same slot: use **hash tags** `{user:42}:cart`.

**Pitfalls:** big keys (MBs) and hot keys, `KEYS *` in production (use `SCAN`), no `maxmemory` set (OOM), using Redis as the only copy of critical data.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-4"></a>
### 8.4 Distributed locks with Redis

```
SET lock:seat:A12 <unique-token> NX PX 600000      # acquire: only if absent, 10 min TTL
-- release only if we still own it (Lua, atomic):
if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end
```
- **Unique token per holder**, so you never delete someone else's lock after yours expired.
- **TTL** frees the lock if the holder crashes. Pick it longer than the work takes, or extend it with a "watchdog" (Redisson does this).
- **Redlock** (majority across 5 independent nodes) is debated. For correctness-critical work, add **fencing tokens** checked by the data store, or use a consensus system (etcd/ZooKeeper).
- In Java: **Redisson** `RLock` (re-entrant, watchdog, fair locks).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-5"></a>
### 8.5 AWS compute

| Service | What | When |
|---|---|---|
| **EC2** | Virtual machines. Instance families (compute/memory-optimised), Auto Scaling Groups | Full control, long-running services |
| **Lambda** | Functions triggered by events (API Gateway, S3, SQS, EventBridge). Pay per invocation | Spiky, event-driven, glue code. Watch **cold starts** (SnapStart helps Java) and the 15-min limit |
| **ECS / Fargate** | Run containers (Fargate = serverless, no EC2 to manage) | Containerised microservices with less ops |
| **EKS** | Managed Kubernetes | Teams already on Kubernetes, portability |
| Elastic Beanstalk | PaaS wrapper | Quick simple deployments |

**Pricing levers:** on-demand, reserved/savings plans (steady load), **spot** (up to ~90% cheaper, interruptible, for batch jobs).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-6"></a>
### 8.6 AWS storage

- **S3:** object storage, 11 nines durability. Storage classes (Standard → Infrequent Access → Glacier), lifecycle rules, versioning, **pre-signed URLs** for direct client upload/download, **event notifications** → Lambda/SQS (e.g. an invoice-processing pipeline).
- **EBS:** block disks attached to one EC2 instance (gp3, io2). Snapshots to S3.
- **EFS:** shared NFS file system across instances.
- **CloudFront:** CDN in front of S3 or APIs.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-7"></a>
### 8.7 AWS databases

| Service | Type | Notes |
|---|---|---|
| **RDS** | Managed Postgres/MySQL/SQL Server/Oracle | Multi-AZ failover, read replicas, automated backups |
| **Aurora** | AWS-built MySQL/Postgres-compatible engine | Storage replicated 6× across 3 AZs, fast failover, Serverless v2 |
| **DynamoDB** | Key-value / document NoSQL | Single-digit ms at any scale. **Partition key design is everything**. GSIs/LSIs, on-demand or provisioned capacity, Streams for CDC, TTL |
| **ElastiCache** | Managed Redis/Memcached | Caching, sessions |
| Redshift / Athena | Analytics warehouse / SQL over S3 | OLAP, reporting |

**DynamoDB tip:** design tables from the **access patterns** first (single-table design). Avoid hot partitions with high-cardinality keys.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-8"></a>
### 8.8 AWS messaging and integration

| Service | Model | Use |
|---|---|---|
| **SQS** | Queue (standard: at-least-once, best-effort order; **FIFO**: ordered, exactly-once processing within limits) | Decouple workers, buffer load. **DLQ** for poison messages. Visibility timeout |
| **SNS** | Pub/sub fan-out | One event → many subscribers (SQS queues, Lambda, email) |
| **EventBridge** | Event bus with content-based routing rules, schedules | Event-driven integration across services and SaaS |
| **Kinesis** / **MSK** | Streaming log / managed Kafka | High-throughput ordered streams, replay |
| Step Functions | Workflow orchestration (state machines) | Sagas, multi-step processes with retries |

**Common pattern:** SNS topic → several SQS queues (fan-out + buffering + independent retries per consumer).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-9"></a>
### 8.9 AWS networking

- **VPC:** your private network. **Public subnets** (route to an Internet Gateway) for load balancers. **Private subnets** for apps and DBs (outbound via a **NAT Gateway**).
- **Security groups:** stateful, instance-level allow rules. **NACLs:** stateless, subnet-level.
- **Load balancers:** **ALB** (L7: HTTP routing, path/host rules) and **NLB** (L4: TCP/UDP, static IPs, extreme performance).
- **Route 53:** DNS with health checks and latency/geo/weighted/failover routing.
- **API Gateway:** managed REST/HTTP/WebSocket APIs in front of Lambda or services. Throttling and auth built in.
- VPC endpoints / PrivateLink reach AWS services without the public internet.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-10"></a>
### 8.10 AWS security and IAM

- **IAM:** users, groups, **roles** (temporary credentials, which services should always use), **policies** (JSON allow/deny on actions + resources). **Least privilege.** Explicit deny beats allow.
- **Never put access keys in code.** EC2, Lambda and ECS assume roles automatically.
- **Secrets Manager** (rotation) / **Parameter Store** for config and secrets. **KMS** for encryption keys (envelope encryption).
- **Cognito** for user sign-up and sign-in (OIDC). **WAF** + **Shield** for web attacks and DDoS.
- Audit: **CloudTrail** (who did what), AWS Config (resource compliance), GuardDuty (threat detection).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-11"></a>
### 8.11 AWS observability and the Well-Architected Framework

- **CloudWatch:** metrics, logs (Logs Insights queries), alarms, dashboards. **X-Ray** (or OpenTelemetry) for tracing.
- **Well-Architected pillars:** operational excellence, **security**, **reliability**, **performance efficiency**, **cost optimisation**, sustainability.
- **Reliability basics:** multi-AZ by default, Auto Scaling, backups with tested restores, infrastructure as code (CloudFormation/CDK/Terraform).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-12"></a>
### 8.12 Git model and everyday commands

**Model:** a **commit** is a snapshot plus parent pointer(s), identified by a SHA. A **branch** is a movable pointer to a commit. **HEAD** is where you are now. Three areas: working directory → **staging area (index)** → repository.

```bash
git status / git diff / git diff --staged
git add -p                      # stage hunks interactively-ish (review what you commit)
git commit -m "feat: ..."
git log --oneline --graph --all
git switch -c feature/x         # create + switch branch
git fetch && git pull --rebase  # update without merge bubbles
git push -u origin feature/x
git stash / git stash pop       # park unfinished work
git cherry-pick <sha>           # copy one commit onto current branch
git blame / git bisect          # who changed it / binary-search the bad commit
```

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-13"></a>
### 8.13 Branching strategies

| Strategy | Shape | Fits |
|---|---|---|
| **Trunk-based** | Short-lived branches (hours to days) merged to `main` often. Feature flags hide unfinished work | CI/CD, frequent releases |
| **GitHub flow** | `main` + feature branches + PRs, deploy from main | Web services |
| **Git flow** | `main`, `develop`, feature/release/hotfix branches | Versioned releases, slower cadence (heavier) |

**Good practice:** small PRs, required reviews and CI checks, protected `main`, conventional commit messages, semantic version **tags** for releases.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-14"></a>
### 8.14 Merge vs rebase

- **Merge:** joins histories with a **merge commit**. It keeps the true history and is non-destructive. Can look noisy.
- **Rebase:** **replays** your commits on top of another branch, giving a **linear** history. It rewrites commit SHAs.
- **Golden rule:** **never rebase commits others have already pulled** (shared or public branches). Rebase your local feature branch onto `main`, then merge.
- **Squash merge:** one clean commit per PR on `main`.
- **Conflicts:** edit the files → `git add` → `git rebase --continue` (or commit, for a merge). `--abort` to back out.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s8-15"></a>
### 8.15 Undoing things

| Situation | Command | Effect |
|---|---|---|
| Unstage a file | `git restore --staged f` | Keeps your changes |
| Discard local edits to a file | `git restore f` | ⚠️ Edits lost |
| Fix the last commit (not pushed) | `git commit --amend` | Rewrites the last commit |
| Undo a **pushed** commit safely | **`git revert <sha>`** | New commit that inverts it. History preserved |
| Move a branch back (local only) | `git reset --soft/--mixed/--hard <sha>` | soft: keep staged · mixed: keep in working dir · **hard: discard** ⚠️ |
| Recover "lost" commits | `git reflog` | Shows where HEAD has been, so you can reset or cherry-pick back |

**Rule:** `revert` for shared history, `reset` only for local, unpushed work.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
