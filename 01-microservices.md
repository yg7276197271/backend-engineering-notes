<a id="top"></a>
# 1. Distributed Microservices

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
- [1.1 RestTemplate](#s1-1)
- [1.2 RestClient](#s1-2)
- [1.3 FeignClient](#s1-3)
- [1.4 Service discovery](#s1-4)
- [1.5 Client-side load balancer](#s1-5)
- [1.6 Rate limiter (Resilience4j)](#s1-6)
- [1.7 Bulkhead](#s1-7)
- [1.8 Retry](#s1-8)
- [1.9 Circuit breaker](#s1-9)
- [1.10 API gateway](#s1-10)
- [1.11 Centralised configuration](#s1-11)
- [1.12 Refresh scope](#s1-12)
- [1.13 Spring Cloud Bus](#s1-13)
- [1.14 Distributed tracing](#s1-14)
- [1.15 Distributed logging](#s1-15)
- [1.16 Distributed scheduler (leader–follower)](#s1-16)

---

<a id="s1-1"></a>
### 1.1 RestTemplate

**What:** Spring's original synchronous HTTP client. Each call blocks the calling thread until the response arrives.

**How it works:**
- Template methods: `getForObject`, `getForEntity`, `postForEntity`, `exchange` (full control over method, headers and body).
- `HttpMessageConverter`s turn Java objects into JSON and back (Jackson).
- The underlying HTTP library is pluggable (JDK `HttpURLConnection` by default, or Apache HttpClient/OkHttp for connection pooling).

```java
@Bean RestTemplate restTemplate(RestTemplateBuilder b) {
    return b.setConnectTimeout(Duration.ofSeconds(2)).setReadTimeout(Duration.ofSeconds(5)).build();
}
Order o = restTemplate.getForObject("http://order-service/orders/{id}", Order.class, id);
```

**Pitfalls:**
- **No timeouts by default.** A slow downstream hangs your threads forever, so always set connect and read timeouts.
- Creating a new `RestTemplate` per request throws away connection pooling. Make it a bean.
- It's in maintenance mode. Prefer RestClient for new code.

**Interview Q&A:**
- *Why is RestTemplate a risk under load?* Each in-flight call holds a thread. Without timeouts, a slow dependency exhausts the thread pool.
- *getForObject vs exchange?* `exchange` gives headers, status and any HTTP method. `getForObject` returns only the body.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-2"></a>
### 1.2 RestClient

**What:** A fluent, synchronous HTTP client added in Spring 6.1 / Boot 3.2. It's the modern replacement for RestTemplate.

**How it works:**
- Same building blocks as RestTemplate (message converters, request factories) with WebClient's fluent API.
- `retrieve()` with `onStatus(...)` gives clean error handling.
- Can back an **HTTP interface** (`@HttpExchange`), giving a declarative client like Feign without Spring Cloud.

```java
RestClient client = RestClient.builder().baseUrl("http://order-service").build();
Order o = client.get().uri("/orders/{id}", id)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> { throw new OrderNotFound(id); })
        .body(Order.class);
```

**Pitfalls:**
- It's still blocking. It doesn't make I/O non-blocking, it just has a nicer API.
- Still set timeouts via the request factory.

**Interview Q&A:**
- *RestClient vs WebClient?* RestClient is blocking and simple. WebClient is reactive and non-blocking, the one to use with WebFlux or many concurrent calls.
- *How do you migrate from RestTemplate?* `RestClient.create(restTemplate)` reuses the existing configuration.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-3"></a>
### 1.3 FeignClient

**What:** A declarative HTTP client (Spring Cloud OpenFeign). You write an interface and Feign generates the implementation.

**How it works:**
- `@EnableFeignClients` scans for `@FeignClient` interfaces and creates proxies.
- `name` resolves through service discovery plus the load balancer, so no hard-coded host.
- Customisation: `RequestInterceptor` (add auth headers), `ErrorDecoder` (map HTTP errors to exceptions), logging level, and Resilience4j fallbacks.

```java
@FeignClient(name = "inventory-service", fallback = InventoryFallback.class)
interface InventoryClient {
    @GetMapping("/stock/{sku}") Stock stock(@PathVariable String sku);
}
```

**Pitfalls:**
- The default `ErrorDecoder` throws a generic `FeignException`, so map it to domain exceptions.
- It hides network calls behind something that looks like a local method. Remember latency and failure modes.
- OpenFeign is now feature-complete. Spring recommends HTTP interfaces (`@HttpExchange`) for new projects.

**Interview Q&A:**
- *How does Feign pick an instance?* It asks Spring Cloud LoadBalancer, which uses the discovery registry.
- *How do you pass a JWT downstream?* A `RequestInterceptor` that copies the `Authorization` header.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-4"></a>
### 1.4 Service discovery

**What:** A registry of live service instances, so callers find services by **name** instead of IP and port.

**How it works:**
- **Registration:** each instance registers on start-up (Eureka, Consul) and sends heartbeats. If heartbeats stop, it's evicted.
- **Lookup:** the client fetches and caches the registry, then resolves `order-service` to a list of instances.
- **Client-side discovery** (Eureka + client load balancer) vs **server-side discovery** (Kubernetes Service/DNS, AWS ALB).

**Pitfalls:**
- Registry caching means a dead instance can still get traffic for a few seconds, so pair discovery with retries and timeouts.
- **Eureka self-preservation:** during network trouble it stops evicting instances, on purpose. It's surprising if you don't know about it.
- In Kubernetes you usually don't need Eureka: the Service + DNS already do discovery.

**Interview Q&A:**
- *Why not just use a load balancer URL?* Discovery lets instances scale and move freely, with health-aware routing and no config change.
- *Eureka (AP) vs ZooKeeper/Consul (CP)?* Eureka favours availability, so it may serve stale entries. CP registries favour consistency but can refuse reads during a partition.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-5"></a>
### 1.5 Client-side load balancer

**What:** The **caller** chooses which instance to hit, from the list it got via discovery. Spring Cloud LoadBalancer replaced Netflix Ribbon.

**How it works:**
- `@LoadBalanced RestTemplate`/`RestClient.Builder` or Feign resolves `http://service-name/...` to a real instance.
- Strategies: round robin (default), random, or custom (zone-aware, weighted, health-based).

```java
@Bean @LoadBalanced RestClient.Builder lbBuilder() { return RestClient.builder(); }
```

**Server-side vs client-side:**
| | Client-side | Server-side (Nginx, ALB) |
|---|---|---|
| Extra network hop | No | Yes |
| Logic lives in | Every client | One place |
| Language-agnostic | No | Yes |

**Pitfalls:**
- Every client needs the library and config.
- Round robin ignores load, so a slow instance still gets its share. Use health or response-time-aware rules where it matters.

**Interview Q&A:**
- *Where does the instance list come from?* The discovery client's cached registry, refreshed periodically.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-6"></a>
### 1.6 Rate limiter (Resilience4j)

**What:** Caps how many calls are allowed per time window, either to protect **your** service from callers or to stay within a **downstream's** quota.

**How it works (Resilience4j):**
- `limitForPeriod` permits per `limitRefreshPeriod`. A caller waits up to `timeoutDuration` for a permit, then gets `RequestNotPermitted`.
- This is per instance and in memory. For a global limit across instances, use a gateway limiter backed by Redis (token bucket in a Lua script).

```yaml
resilience4j.ratelimiter.instances.payment:
  limit-for-period: 50
  limit-refresh-period: 1s
  timeout-duration: 0
```

**Pitfalls:**
- A per-instance limit of 50 across 10 pods is really 500 in total. Know which one you need.
- Return **429 Too Many Requests** with `Retry-After`, not a 500.

**Interview Q&A:**
- *Rate limiter vs bulkhead?* A rate limiter caps calls **over time**. A bulkhead caps **concurrent** calls.
- *Which algorithms exist?* Fixed window, sliding log, sliding window counter, token bucket, leaky bucket. Token bucket allows controlled bursts.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-7"></a>
### 1.7 Bulkhead

**What:** Isolates resources per dependency so one slow or failing dependency can't consume every thread, like watertight compartments in a ship.

**How it works:**
- **Semaphore bulkhead:** limits concurrent calls on the caller's thread (`maxConcurrentCalls`, `maxWaitDuration`).
- **Thread-pool bulkhead:** a separate bounded pool and queue per dependency, so calls become async.

```yaml
resilience4j.bulkhead.instances.inventory:
  max-concurrent-calls: 20
  max-wait-duration: 10ms
```

**Pitfalls:**
- A pool that's too small rejects healthy traffic. Size it from measured latency × throughput (Little's law: concurrency ≈ rate × latency).
- Thread-pool bulkheads lose `ThreadLocal` context (security, MDC) unless you propagate it.

**Interview Q&A:**
- *Give a real failure a bulkhead prevents.* A reporting endpoint calls a slow analytics service. Without a bulkhead it holds all 200 Tomcat threads and checkout goes down too. With one, only reporting degrades.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-8"></a>
### 1.8 Retry

**What:** Automatically re-attempts a failed call when the failure is likely transient (timeout, 503, connection reset).

**How it works:**
- `maxAttempts`, `waitDuration`, **exponential backoff** + **jitter**, `retryExceptions` / `ignoreExceptions`.

```yaml
resilience4j.retry.instances.inventory:
  max-attempts: 3
  wait-duration: 200ms
  enable-exponential-backoff: true
  exponential-backoff-multiplier: 2
  retry-exceptions: [java.net.SocketTimeoutException]
  ignore-exceptions: [com.shop.BusinessValidationException]
```

**Pitfalls:**
- **Only retry idempotent operations**, or make them idempotent with an idempotency key. Retrying a non-idempotent POST can double-charge someone.
- **Retry storms:** 3 retries at each of 4 layers is up to 3⁴ = 81 calls to the bottom service. Retry at one layer only.
- Never retry 4xx validation errors. They won't succeed next time.

**Interview Q&A:**
- *Why jitter?* Without it, every client retries at the same instant and creates synchronised load spikes.
- *Retry or circuit breaker?* Both: retry for blips, and the circuit breaker stops retrying a service that's actually down.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-9"></a>
### 1.9 Circuit breaker

**What:** Stops calling a dependency that is failing and fails fast instead, giving it time to recover.

**How it works (state machine):**
- **CLOSED:** calls pass through and failures are recorded in a sliding window (count- or time-based).
- When the failure rate (or slow-call rate) passes the threshold, it goes **OPEN** and calls fail immediately with `CallNotPermittedException` → fallback.
- After `waitDurationInOpenState` it goes **HALF_OPEN**: a few trial calls. Success returns it to CLOSED, failure back to OPEN.

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "stockFallback")
public Stock stock(String sku) { return client.stock(sku); }
Stock stockFallback(String sku, Throwable t) { return Stock.unknown(sku); }
```
```yaml
resilience4j.circuitbreaker.instances.inventory:
  sliding-window-size: 20
  failure-rate-threshold: 50
  slow-call-duration-threshold: 2s
  wait-duration-in-open-state: 30s
  permitted-number-of-calls-in-half-open-state: 3
```

**Resilience4j decorator order:** `Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( call )))))`. Retries go through the breaker, so repeated failures open it.

**Pitfalls:**
- A fallback that hides real errors (an empty list that looks valid) can mislead users and monitoring.
- A window that's too small means one bad second opens the circuit.
- Count **slow calls** too. A dependency at 10 s latency is effectively down.

**Interview Q&A:**
- *Why not just use timeouts?* A timeout still spends the full wait on every call. An open breaker fails in microseconds and takes load off the sick service.
- *What's HALF_OPEN for?* Probing recovery safely with a few requests instead of the full flood.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-10"></a>
### 1.10 API gateway

**What:** The single entry point in front of your microservices. It handles cross-cutting concerns so the services don't have to.

**How it works (Spring Cloud Gateway):**
- **Route** = ID + URI (`lb://order-service`) + **predicates** (path, header, method) + **filters**.
- Filters: auth/JWT validation, rate limiting (`RequestRateLimiter` with Redis), retry, circuit breaker, header and path rewrite, CORS.
- Built on reactive Netty, so it handles many concurrent connections. A servlet (MVC) variant also exists.

```yaml
spring.cloud.gateway.routes:
  - id: orders
    uri: lb://order-service
    predicates: [ Path=/api/orders/** ]
    filters: [ StripPrefix=1, name: CircuitBreaker ]
```

**Patterns:** **BFF** (backend for frontend: one gateway per client type), request aggregation, canary routing by header or weight.

**Pitfalls:**
- Business logic in the gateway turns it into a distributed monolith.
- It's a single point of failure unless you run several replicas behind a load balancer.
- Blocking code inside a reactive gateway filter stalls the event loop.

**Interview Q&A:**
- *Gateway vs load balancer?* An LB spreads traffic across instances of one service. A gateway routes across many services and applies policy (auth, limits).
- *Where do you validate JWTs?* At the gateway for authentication, and in services again for authorisation (defence in depth).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-11"></a>
### 1.11 Centralised configuration

**What:** One **Config Server** serves configuration to every service, stored in Git (versioned, reviewable) or Vault for secrets.

**How it works:**
- Server: `@EnableConfigServer` + `spring.cloud.config.server.git.uri`.
- Client: `spring.config.import=configserver:http://config:8888`. It fetches `{application}-{profile}.yml` on start-up.
- Precedence: app-specific profile > app default > shared `application.yml`.

**Pitfalls:**
- The Config Server becomes a start-up dependency. Use `optional:configserver:` or retries plus local fallbacks, and run it highly available.
- Never put plain-text secrets in Git. Use Vault or `{cipher}` encrypted values.

**Interview Q&A:**
- *Why centralise config?* One change applies to every instance, it's audited in Git, and environments differ only in config, not builds.
- *Kubernetes alternative?* ConfigMaps + Secrets, mounted or read through Spring Cloud Kubernetes.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-12"></a>
### 1.12 Refresh scope

**What:** `@RefreshScope` lets beans pick up changed configuration **without restarting** the app.

**How it works:**
- Refresh-scoped beans sit behind a proxy. `POST /actuator/refresh` re-reads config and rebinds `@ConfigurationProperties`, and refresh-scoped beans are recreated lazily on next use.

```java
@RefreshScope @RestController
class FeatureController {
    @Value("${feature.newCheckout:false}") boolean newCheckout;
}
```

**Pitfalls:**
- It only refreshes the instance you call. Use **Cloud Bus** to refresh everyone.
- Not everything can change live: data source URLs, server ports and thread pool sizes usually need a restart.
- In-flight requests may see a mix of old and new values during the refresh.

**Interview Q&A:**
- *Does @ConfigurationProperties need @RefreshScope?* No. Those beans are rebound on refresh automatically. `@Value` fields need `@RefreshScope`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-13"></a>
### 1.13 Spring Cloud Bus

**What:** Links all instances through a message broker (Kafka or RabbitMQ) so **one** refresh event reaches **all** of them.

**How it works:**
- `POST /actuator/busrefresh` on any instance (or a Git webhook to the Config Server via `spring-cloud-config-monitor`) publishes `RefreshRemoteApplicationEvent`, and every instance refreshes.
- Can target a subset: `/actuator/busrefresh/order-service:**`.

**Pitfalls:**
- Adds a broker dependency for config, so make sure it's monitored.
- A bad config value now reaches everything at once. Validate config (CI checks, `@Validated` properties) before pushing.

**Interview Q&A:**
- *How do you get zero-touch config rollout?* Commit to Git → webhook → Config Server → Bus event → all instances refresh.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-14"></a>
### 1.14 Distributed tracing

**What:** Follows one request across every service it touches, with a timing for each hop.

**How it works:**
- **Trace ID**: one per request. **Span ID**: one per operation (an HTTP call, a DB query, a Kafka publish). Spans form a tree.
- Context travels in headers: W3C `traceparent` (or B3). Instrumented clients and servers read and write them automatically.
- In Spring Boot 3: **Micrometer Tracing** with a bridge (Brave or OpenTelemetry) and an exporter (Zipkin, Tempo, Jaeger).

```yaml
management.tracing.sampling.probability: 0.1   # 10% of requests
```

**Pitfalls:**
- Context gets lost across `@Async`, executors and Kafka unless it's propagated. Use context-propagating executors and Kafka observation.
- 100% sampling is expensive in production. Sample, but always keep traces that contain errors (tail sampling).

**Interview Q&A:**
- *Tracing vs logging vs metrics?* Metrics tell you **that** something is wrong (p99 latency is up). Traces tell you **where** (which hop). Logs tell you **why** (the details).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-15"></a>
### 1.15 Distributed logging

**What:** Collecting the logs of every instance into one searchable place, correlated by trace ID.

**How it works:**
- Apps write **structured JSON** logs to stdout, including `traceId`/`spanId` from the MDC (Boot 3.4 supports structured logging natively).
- A shipper (Filebeat, Fluent Bit, Logstash) sends them to storage (Elasticsearch, Loki), and you search in Kibana or Grafana.
- **ELK** = Elasticsearch + Logstash + Kibana. **EFK** swaps in Fluentd. **PLG** = Promtail + Loki + Grafana (cheaper, label-based).

**Pitfalls:**
- Logging PII, tokens or card numbers is a compliance incident, so mask them at the source.
- DEBUG level in production fills storage and costs money. Change log levels at runtime via Actuator `/loggers` instead.
- Unstructured text logs are hard to query. Keep a consistent field schema.

**Interview Q&A:**
- *How do you debug one failed order across 6 services?* Take the trace ID from the error response or header and search all logs for it.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s1-16"></a>
### 1.16 Distributed scheduler (leader–follower)

**What:** Making sure a scheduled job runs **once** across N instances, not N times.

**Approaches:**
1. **DB lock (ShedLock):** a lock row with `lock_until`. The first instance to update it runs the job.
   ```java
   @Scheduled(cron = "0 0 * * * *")
   @SchedulerLock(name = "settlementJob", lockAtMostFor = "50m", lockAtLeastFor = "1m")
   public void settle() { ... }
   ```
2. **Leader election:** ZooKeeper/Curator `LeaderLatch`, a Kubernetes `Lease`, or Spring Integration leader election. Only the leader schedules.
3. **Dedicated scheduler service** puts jobs on a queue, and workers consume them (scales out heavy jobs).
4. **Platform scheduling:** a Kubernetes `CronJob`, which runs one pod per schedule.

**Pitfalls:**
- **Split brain:** after a GC pause or network blip, two instances may both think they're leader. Make jobs **idempotent** and use **fencing tokens** (a monotonically increasing lock version checked on write).
- If `lockAtMostFor` is shorter than the job's runtime, the lock expires mid-run and a second copy starts.

**Interview Q&A:**
- *Why not just run the scheduler on one pod?* That's a single point of failure. Leader election gives automatic failover.
- *What is a fencing token?* A number that increases with each lock grant. Storage rejects writes carrying an older token, so a stale leader can't corrupt data.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
