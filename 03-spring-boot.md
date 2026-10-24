<a id="top"></a>
# 3. Spring Boot

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
**Core**
- [3.1 Introduction: Spring vs Spring Boot](#s3-1)
- [3.2 Setup and layered architecture](#s3-2)
- [3.3 Maven and its lifecycle](#s3-3)
- [3.4 Core annotations](#s3-4)
- [3.5 Bean and its lifecycle](#s3-5)
- [3.6 Dependency injection](#s3-6)
- [3.7 Bean scopes](#s3-7)
- [3.8 Dynamically initialised beans](#s3-8)
- [3.9 @ConditionalOnProperty](#s3-9)
- [3.10 Profiles](#s3-10)
- [3.11 @ConfigurationProperties](#s3-11)

**Cross-cutting**
- [3.12 AOP](#s3-12)
- [3.13 @Transactional: how it works](#s3-13)
- [3.14 @Transactional: propagation](#s3-14)
- [3.15 @Transactional: isolation and locking](#s3-15)
- [3.16 @Async (part 1): basics](#s3-16)
- [3.17 @Async (part 2): executors and pitfalls](#s3-17)

**Web layer**
- [3.18 Custom interceptors](#s3-18)
- [3.19 Filters vs interceptors](#s3-19)
- [3.20 HATEOAS](#s3-20)
- [3.21 ResponseEntity and response codes](#s3-21)
- [3.22 Exception handling](#s3-22)

**Data (JPA)**
- [3.23 JPA setup and architecture](#s3-23)
- [3.24 First-level cache](#s3-24)
- [3.25 Second-level cache](#s3-25)
- [3.26 DTO to table (entity mapping)](#s3-26)
- [3.27 One-to-one](#s3-27)
- [3.28 One-to-many and many-to-many](#s3-28)
- [3.29 Derived queries and JPQL](#s3-29)
- [3.30 Native queries and the Criteria API](#s3-30)
- [3.31 Specification API](#s3-31)

**Security & ops**
- [3.32 Common attacks](#s3-32)
- [3.33 Security setup and architecture](#s3-33)
- [3.34 Users, form login and basic auth](#s3-34)
- [3.35 JWT authentication](#s3-35)
- [3.36 OAuth 2.0 in Spring](#s3-36)
- [3.37 Method security](#s3-37)
- [3.38 Actuator](#s3-38)
- [3.39 Spring AI](#s3-39)

---

<a id="s3-1"></a>
### 3.1 Introduction: Spring vs Spring Boot

**What:** **Spring** is the framework: IoC container, DI, AOP, MVC, data and security modules. **Spring Boot** is an opinionated layer on top that removes the setup work.

**What Boot adds:**
- **Auto-configuration:** configures beans based on what's on the classpath (`spring-boot-starter-web` → Tomcat + MVC + Jackson).
- **Starters:** curated dependency bundles with compatible versions.
- **Embedded server:** run with `java -jar`, no WAR deployment.
- **Externalised config** (`application.yml`, env vars, profiles) and **Actuator** for production endpoints.

**How auto-configuration works:** `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Auto-config classes are listed in `META-INF/spring/...AutoConfiguration.imports` and guarded by `@ConditionalOnClass`/`@ConditionalOnMissingBean`, so **your own bean always wins**.

**Interview Q&A:**
- *How do you see what got auto-configured?* Run with `--debug` for the condition evaluation report, or use the Actuator `/conditions` endpoint.
- *How do you replace an auto-configured bean?* Declare your own bean of that type, since `@ConditionalOnMissingBean` backs off.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-2"></a>
### 3.2 Setup and layered architecture

**Setup:** start.spring.io → choose Java version, build tool and starters → `main` with `SpringApplication.run`.

**Layers:**
| Layer | Annotation | Responsibility |
|---|---|---|
| Controller | `@RestController` | HTTP in/out, validation, mapping DTOs. No business logic |
| Service | `@Service` | Business rules, transactions |
| Repository | `@Repository` | Persistence (Spring Data JPA). Translates DB exceptions |
| Entity / DTO | — | Entity = DB shape. DTO = API shape. Keep them separate |

**Rules:** dependencies point downward only (controller → service → repository). Never return entities straight from controllers.

**Pitfalls:**
- "Anaemic" services that just pass calls through, while the logic leaks into controllers.
- Returning JPA entities exposes internal fields and triggers lazy-loading errors during JSON serialisation.

**Interview Q&A:**
- *Why separate DTOs from entities?* A stable API contract, no accidental field exposure, and freedom to change the DB schema independently.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-3"></a>
### 3.3 Maven and its lifecycle

**What:** A build tool. `pom.xml` declares dependencies, plugins and the build. Convention over configuration (`src/main/java`, `src/test/java`).

**Default lifecycle phases (in order):** `validate → compile → test → package → verify → install → deploy`.
Running a phase runs **every phase before it**: `mvn package` compiles and tests first.

**Key concepts:**
- **Coordinates:** `groupId:artifactId:version`.
- **Scopes:** `compile` (default), `provided` (supplied by the server), `runtime`, `test`.
- **Transitive dependencies** and conflicts: the "nearest wins" rule. Inspect with `mvn dependency:tree` and pin versions in `<dependencyManagement>`.
- **Parent POM** (`spring-boot-starter-parent`) gives managed versions and plugin defaults. **BOMs** import version sets.
- Repositories: local `~/.m2` → remote (Maven Central, Nexus/Artifactory).

**Pitfalls:**
- `-DskipTests` in CI hides broken builds.
- SNAPSHOT dependencies in a release build aren't reproducible.

**Interview Q&A:**
- *install vs deploy?* `install` copies the artifact into your local `~/.m2`. `deploy` publishes it to a remote repository.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-4"></a>
### 3.4 Core annotations

| Annotation | Purpose |
|---|---|
| `@Component`, `@Service`, `@Repository`, `@Controller` | Stereotypes marking beans. `@Repository` also translates exceptions |
| `@Configuration` + `@Bean` | Java-based bean definitions (third-party classes) |
| `@Autowired`, `@Qualifier`, `@Primary` | Injection and choosing between candidates |
| `@Value("${key:default}")` | Inject a single property |
| `@RestController` = `@Controller` + `@ResponseBody` | JSON APIs |
| `@RequestMapping`, `@GetMapping`, … | Map HTTP routes |
| `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader` | Bind request parts |
| `@Valid` / `@Validated` | Trigger Bean Validation (`@NotNull`, `@Size`, …) |

**Pitfalls:**
- `@Configuration` classes are proxied so that `@Bean` methods calling each other return the **same singleton**. With `@Component` + `@Bean` ("lite mode") you'd get a new instance each time.
- Two beans of the same type without `@Primary`/`@Qualifier` cause `NoUniqueBeanDefinitionException`.

**Interview Q&A:**
- *@Component vs @Bean?* `@Component` goes on your own class (found by scanning). `@Bean` goes on a factory method, which you need for classes you can't annotate.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-5"></a>
### 3.5 Bean and its lifecycle

**What:** A bean is an object created and managed by the Spring IoC container.

**Lifecycle:**
1. Instantiate (constructor) → 2. inject dependencies → 3. `Aware` callbacks (`BeanNameAware`, `ApplicationContextAware`)
4. `BeanPostProcessor.postProcessBeforeInitialization`
5. **`@PostConstruct`** → `InitializingBean.afterPropertiesSet` → custom `initMethod`
6. `BeanPostProcessor.postProcessAfterInitialization` (**where AOP proxies are created**)
7. Bean in use
8. On shutdown: **`@PreDestroy`** → `DisposableBean.destroy` → custom `destroyMethod`

**Eager vs lazy:** singletons are created at start-up (fail fast). `@Lazy` delays creation until first use.

**Pitfalls:**
- Doing heavy work (remote calls) in a constructor or `@PostConstruct` slows start-up and can fail it. Use `ApplicationReadyEvent` for warm-up tasks.
- Prototype beans **don't** get `@PreDestroy` called. The container doesn't track them after creation.

**Interview Q&A:**
- *Where do proxies (for @Transactional) come from?* A `BeanPostProcessor` wraps the bean after initialisation, so the context holds the proxy, not the raw object.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-6"></a>
### 3.6 Dependency injection

**What:** The container supplies a class's dependencies, so the class doesn't build them itself (**Inversion of Control**).

**Types:**
| Type | Verdict |
|---|---|
| **Constructor** | ✅ Preferred: final fields, required deps explicit, easy to test, circular deps fail fast |
| Setter | Optional dependencies only |
| Field (`@Autowired` on field) | ❌ Hidden deps, can't be final, needs reflection in tests |

```java
@Service
class OrderService {
    private final PaymentClient payments;
    OrderService(PaymentClient payments) { this.payments = payments; } // no @Autowired needed (single ctor)
}
```

**Resolving ambiguity:** `@Primary` (default choice), `@Qualifier("stripe")`, or inject `List<PaymentClient>` / `Map<String, PaymentClient>` to get all of them (great for the Strategy pattern).

**Circular dependencies:** A needs B and B needs A. Boot 2.6+ **forbids** them by default, which is a sign to redesign: extract a third bean, or use events.

**Interview Q&A:**
- *Why is constructor injection better for testing?* You can `new` the class with mocks. No Spring context or reflection needed.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-7"></a>
### 3.7 Bean scopes

| Scope | Instances | Notes |
|---|---|---|
| **singleton** (default) | One per container | Must be **stateless** or thread-safe |
| **prototype** | New on every lookup | No destroy callback |
| **request** | One per HTTP request | Web only |
| **session** | One per HTTP session | Web only |
| application | One per ServletContext | |

**The classic trap:** injecting a prototype into a singleton. It's injected **once**, so you always get the same instance. Fixes:
- `ObjectProvider<MyProto>` → `provider.getObject()` each time.
- `@Lookup` method injection.
- A scoped proxy: `@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)`.

**Pitfalls:**
- Mutable fields in a singleton service cause race conditions under concurrent requests.

**Interview Q&A:**
- *How does a request-scoped bean get into a singleton controller?* Through a scoped proxy that resolves the current request's instance on every call.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-8"></a>
### 3.8 Dynamically initialised beans

**What:** Choosing **which implementation** to create at runtime, based on configuration or input.

**Techniques:**
- **`@Bean` method with logic:**
  ```java
  @Bean NotificationSender sender(@Value("${notify.channel}") String ch) {
      return ch.equals("sms") ? new SmsSender() : new EmailSender();
  }
  ```
- **Map of strategies:** inject `Map<String, PaymentProcessor>` (bean name → bean) and pick per request: `processors.get(type + "Processor")`.
- **Conditional annotations** (`@ConditionalOnProperty`, `@Profile`), decided at start-up.
- **Programmatic registration:** `BeanDefinitionRegistryPostProcessor` / `registerBean(...)` for truly dynamic sets (rarely needed).

**Pitfalls:**
- `if/else` over types scattered around the code. Centralise it with a strategy map.

**Interview Q&A:**
- *How do you pick a payment provider per request?* Inject all implementations as a map and select by key. Adding a new provider is just a new bean (open/closed principle).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-9"></a>
### 3.9 @ConditionalOnProperty

**What:** Create a bean only if a property has a given value. Used for feature toggles and swappable implementations.

```java
@Bean
@ConditionalOnProperty(name = "cache.type", havingValue = "redis", matchIfMissing = false)
CacheService redisCache() { return new RedisCacheService(); }

@Bean
@ConditionalOnProperty(name = "cache.type", havingValue = "local", matchIfMissing = true)
CacheService localCache() { return new CaffeineCacheService(); }
```

**Related conditions:** `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnBean`, `@ConditionalOnExpression` (SpEL), `@ConditionalOnWebApplication`.

**Pitfalls:**
- It's evaluated **once at start-up**. Changing the property later (even with refresh) doesn't create or remove the bean.
- Overlapping conditions can produce zero or two beans. Use `matchIfMissing` deliberately.

**Interview Q&A:**
- *Feature flag at start-up vs at runtime?* `@ConditionalOnProperty` = start-up. For runtime flags, use a flag service (Unleash, LaunchDarkly) or a refresh-scoped config check.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-10"></a>
### 3.10 Profiles

**What:** Named sets of configuration and beans per environment (`dev`, `test`, `prod`).

**How it works:**
- `application-dev.yml` overrides `application.yml` when `dev` is active.
- Activate with `spring.profiles.active=prod` (env var `SPRING_PROFILES_ACTIVE`).
- `@Profile("!prod")` on a bean: created in every profile except prod.
- **Profile groups:** `spring.profiles.group.prod=prod-db,prod-mq`.

**Property precedence (high → low):** command-line args → env vars → profile-specific files → `application.yml` → defaults.

**Pitfalls:**
- Builds that differ per environment. Build **once** and change only config.
- Secrets in `application-prod.yml` in Git. Use a vault or env vars.

**Interview Q&A:**
- *How do you use an in-memory DB for tests and Postgres in prod?* Profile-specific datasource config (or Testcontainers in tests).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-11"></a>
### 3.11 @ConfigurationProperties

**What:** Binds a group of properties to a typed, validated object, instead of many scattered `@Value` fields.

```java
@ConfigurationProperties(prefix = "payment")
@Validated
public record PaymentProps(@NotBlank String apiUrl, @DurationUnit(SECONDS) Duration timeout, int maxRetries) {}
// enable: @ConfigurationPropertiesScan on the main class
```
```yaml
payment:
  api-url: https://psp.example.com
  timeout: 5
  max-retries: 3
```

**Why it's better than @Value:** type safety, relaxed binding (`api-url` ↔ `apiUrl` ↔ `PAYMENT_APIURL`), validation at start-up, IDE metadata and autocomplete, and it's immutable with records.

**Interview Q&A:**
- *What happens if a required property is missing?* With `@Validated`, the app **fails at start-up** with a clear message, which is better than failing on first use in production.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-12"></a>
### 3.12 AOP (aspect-oriented programming)

**What:** Puts cross-cutting logic (logging, metrics, security, transactions, retries) into **aspects** instead of repeating it in every method.

**Vocabulary:**
- **Aspect:** the class holding the logic. **Advice:** when it runs (`@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`).
- **Pointcut:** where it applies (`execution(* com.shop.service.*.*(..))`, `@annotation(Audited)`). **Join point:** the matched method call.

```java
@Aspect @Component
class TimingAspect {
    @Around("@annotation(com.shop.Timed)")
    Object time(ProceedingJoinPoint pjp) throws Throwable {
        long t = System.nanoTime();
        try { return pjp.proceed(); }
        finally { log.info("{} took {} ms", pjp.getSignature(), (System.nanoTime() - t) / 1_000_000); }
    }
}
```

**How Spring does it:** **runtime proxies**. A JDK dynamic proxy if the bean implements an interface, otherwise a CGLIB subclass (Boot's default).

**Pitfalls (proxy limits):**
- **Self-invocation** (`this.method()`) skips the proxy, so no advice runs.
- `private` and `final` methods can't be advised by CGLIB.
- Only Spring beans are proxied, not objects you create with `new`.

**Interview Q&A:**
- *Why doesn't @Transactional work when I call the method from the same class?* The call never passes through the proxy. Move the method to another bean, or inject the proxy of itself.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-13"></a>
### 3.13 @Transactional: how it works

**What:** Declarative transactions. Spring starts a transaction before the method and commits or rolls back after it.

**Mechanics:**
- The proxy (from AOP) calls `PlatformTransactionManager` (JPA → `JpaTransactionManager`). The connection or EntityManager is bound to the **current thread**.
- **Commit** on normal return. **Rollback** on `RuntimeException`/`Error` by default. **Checked exceptions commit** unless you add `rollbackFor = Exception.class`.
- `readOnly = true`: hints Hibernate to skip dirty checking and may route to a read replica.
- `timeout`: rolls back if the transaction runs too long.

**Pitfalls:**
- Self-invocation and non-public methods (before Spring 6) aren't transactional.
- Catching the exception inside the method means no rollback.
- Long transactions that wrap remote HTTP calls hold DB connections and locks. Keep transactions short.
- `@Async`/new threads don't inherit the transaction (it's bound to the thread).

**Interview Q&A:**
- *A checked exception was thrown but data was committed. Why?* The default rollback rule only covers unchecked exceptions. Add `rollbackFor`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-14"></a>
### 3.14 @Transactional: propagation

| Propagation | If a transaction exists | If none exists |
|---|---|---|
| **REQUIRED** (default) | Join it | Create one |
| **REQUIRES_NEW** | **Suspend** it, run in a new independent one | Create one |
| **NESTED** | Savepoint inside it (rolls back alone) | Create one |
| MANDATORY | Join it | Throw |
| SUPPORTS | Join it | Run without |
| NOT_SUPPORTED | Suspend, run without | Run without |
| NEVER | Throw | Run without |

**Classic uses:**
- **REQUIRES_NEW** for audit or failure logs that must persist even if the main transaction rolls back.
- **NESTED** for "try this step, roll back just that step on failure". It needs JDBC savepoints and doesn't work with JPA's `JpaTransactionManager` by default.

**Pitfall: "Transaction silently rolled back because it has been marked as rollback-only".** An inner `REQUIRED` method threw and was caught by the outer one. The shared transaction is already doomed, so the commit fails.

**Interview Q&A:**
- *REQUIRES_NEW vs NESTED?* REQUIRES_NEW is a separate transaction with its own commit (it survives an outer rollback). NESTED is a savepoint whose work is lost if the outer transaction rolls back.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-15"></a>
### 3.15 @Transactional: isolation and locking

**Concurrency anomalies:**
- **Dirty read:** reading another transaction's uncommitted data.
- **Non-repeatable read:** the same row, read twice, has changed.
- **Phantom read:** the same query, run twice, returns new rows.

| Isolation | Dirty | Non-repeatable | Phantom |
|---|---|---|---|
| READ_UNCOMMITTED | ✗ possible | possible | possible |
| **READ_COMMITTED** (Postgres/Oracle default) | prevented | possible | possible |
| REPEATABLE_READ (MySQL default) | prevented | prevented | possible* |
| SERIALIZABLE | prevented | prevented | prevented |
<sub>*MySQL InnoDB and Postgres prevent most phantoms at REPEATABLE_READ.</sub>

**Locking in JPA:**
- **Optimistic:** a `@Version` column. An update fails with `OptimisticLockException` if someone changed the row. Best for low contention.
- **Pessimistic:** `@Lock(PESSIMISTIC_WRITE)` → `SELECT … FOR UPDATE`. Best for high contention on hot rows, but watch for deadlocks.

**Interview Q&A:**
- *Two users update the same order at once. How do you stop lost updates?* `@Version` optimistic locking. The second save fails, and you retry or tell the user.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-16"></a>
### 3.16 @Async (part 1): basics

**What:** Runs a method on another thread. The caller returns immediately.

```java
@EnableAsync @Configuration class AsyncConfig {}

@Service class ReportService {
    @Async
    public CompletableFuture<Report> build(long id) { return CompletableFuture.completedFuture(heavyWork(id)); }
}
```

**Return types:** `void` (fire and forget) or `CompletableFuture<T>` (get the result or error later).

**Proxy rules apply:** it must be a public method called **from another bean**. Self-invocation runs synchronously.

**Interview Q&A:**
- *@Async vs a message queue?* @Async lives in memory: if the JVM restarts, the work is lost. Use a queue for work that must survive restarts or needs retries.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-17"></a>
### 3.17 @Async (part 2): executors and pitfalls

**Always configure the executor:**
```java
@Bean(name = "reportExecutor")
ThreadPoolTaskExecutor reportExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(4); ex.setMaxPoolSize(8); ex.setQueueCapacity(100);
    ex.setThreadNamePrefix("report-");
    ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    ex.initialize(); return ex;
}
@Async("reportExecutor") public void ...
```

**ThreadPoolExecutor behaviour:** core threads first → then the **queue** → then grow to max → then the rejection policy. An unbounded queue means max size is **never reached**.

**Pitfalls:**
- Exceptions in `void` methods vanish unless you set an `AsyncUncaughtExceptionHandler`.
- `ThreadLocal` context (security, MDC/trace IDs, transactions) doesn't propagate. Use a `TaskDecorator` to copy MDC and the security context.
- One shared pool for everything means one slow task type starves the others (the bulkhead idea).

**Interview Q&A:**
- *Why didn't my pool grow past the core size?* The queue wasn't full. Threads beyond core are only created once the queue fills.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-18"></a>
### 3.18 Custom interceptors

**What:** Spring MVC hooks around controller handling. They're aware of **which handler** will run.

```java
class AuditInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("start", System.nanoTime());
        return true;                         // false = stop the chain
    }
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        log.info("{} {} -> {}", req.getMethod(), req.getRequestURI(), res.getStatus());
    }
}
@Configuration class WebConfig implements WebMvcConfigurer {
    public void addInterceptors(InterceptorRegistry r) { r.addInterceptor(new AuditInterceptor()).addPathPatterns("/api/**"); }
}
```

**Hooks:** `preHandle` (before the controller, can block), `postHandle` (after the controller, before rendering, skipped on exception), `afterCompletion` (always, good for cleanup and logging).

**Uses:** logging, tenant resolution, API-key checks, timing, reading custom annotations on handler methods.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-19"></a>
### 3.19 Filters vs interceptors

| | Servlet Filter | HandlerInterceptor |
|---|---|---|
| Level | Servlet container, **before** DispatcherServlet | Spring MVC, around the controller |
| Sees | Raw request/response, can wrap them | The handler method and its annotations |
| Applies to | All requests (static files, errors) | Only requests mapped to handlers |
| Typical use | Security (Spring Security is a filter chain), CORS, compression, request logging, trace IDs | Auth checks per handler, audit, locale/tenant |
| Can modify body | Yes (request/response wrappers) | Not easily |

**Order of execution:** `Filter → DispatcherServlet → Interceptor.preHandle → Controller → postHandle → view/JSON → afterCompletion → Filter (return)`.

**Interview Q&A:**
- *Where does Spring Security run?* As a servlet **filter chain** (`DelegatingFilterProxy` → `SecurityFilterChain`), before any controller.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-20"></a>
### 3.20 HATEOAS

**What:** *Hypermedia As The Engine Of Application State.* Responses include **links** to related resources and available actions, so clients follow links instead of hard-coding URLs. It's REST maturity level 3.

```java
EntityModel<Order> model = EntityModel.of(order,
    linkTo(methodOn(OrderController.class).get(order.id())).withSelfRel(),
    linkTo(methodOn(OrderController.class).cancel(order.id())).withRel("cancel"));
```
```json
{ "id": 42, "status": "PLACED",
  "_links": { "self": {"href": "/orders/42"}, "cancel": {"href": "/orders/42/cancel"} } }
```

**Benefit:** the server controls the workflow. For example, the `cancel` link only appears when cancelling is allowed.
**Reality:** rarely fully adopted. Most public APIs stop at level 2 (resources + HTTP verbs).

**Interview Q&A:**
- *What are the Richardson maturity levels?* 0: one endpoint (RPC). 1: resources. 2: HTTP verbs and status codes. 3: hypermedia (HATEOAS).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-21"></a>
### 3.21 ResponseEntity and response codes

**ResponseEntity** controls status, headers and body:
```java
@PostMapping("/orders")
ResponseEntity<OrderDto> create(@Valid @RequestBody CreateOrder req) {
    OrderDto o = service.create(req);
    return ResponseEntity.created(URI.create("/orders/" + o.id())).body(o);   // 201 + Location
}
```

**Codes to use correctly:**
| Code | When |
|---|---|
| 200 OK | Successful GET/PUT with a body |
| **201 Created** | POST created a resource (+ `Location` header) |
| 202 Accepted | Accepted for **async** processing |
| **204 No Content** | Success with no body (DELETE) |
| 400 Bad Request | Malformed request |
| 401 Unauthorized | Not authenticated |
| **403 Forbidden** | Authenticated but not allowed |
| 404 Not Found | Resource doesn't exist |
| **409 Conflict** | State conflict (duplicate, version clash) |
| 422 Unprocessable Entity | Valid syntax, failed business validation |
| **429 Too Many Requests** | Rate limited (+ `Retry-After`) |
| 500 / 502 / 503 / 504 | Server bug / bad upstream / unavailable / upstream timeout |

**Interview Q&A:**
- *401 vs 403?* 401 = "who are you?" (no or invalid credentials). 403 = "I know who you are, and you can't do this."

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-22"></a>
### 3.22 Exception handling

**Layers of handling:**
- `@ExceptionHandler` in one controller (local).
- **`@RestControllerAdvice`** with `@ExceptionHandler`s: **global**, consistent error bodies. The usual choice.
- `@ResponseStatus` on a custom exception: a quick status mapping.
- `ResponseStatusException`: throw with a status inline.

```java
@RestControllerAdvice
class ApiErrors {
    @ExceptionHandler(OrderNotFound.class)
    ProblemDetail notFound(OrderNotFound e) {
        ProblemDetail p = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
        p.setTitle("Order not found"); return p;
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail invalid(MethodArgumentNotValidException e) { ... }   // field errors → 400
}
```
**`ProblemDetail`** (RFC 9457, Spring 6) is the standard error format: `type`, `title`, `status`, `detail`, `instance`.

**Pitfalls:**
- Leaking stack traces or SQL errors to clients. Log them internally and return a safe message.
- Catching `Exception` everywhere and returning 200 with an error flag.

**Interview Q&A:**
- *How do you return field-level validation errors?* Handle `MethodArgumentNotValidException` and map each `FieldError` to `{field, message}`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-23"></a>
### 3.23 JPA setup and architecture

**Stack:** Spring Data JPA (repositories) → **JPA** (the specification) → **Hibernate** (the implementation) → JDBC → database.

**Key pieces:**
- **Entity:** `@Entity` class mapped to a table. `@Id` + `@GeneratedValue`.
- **EntityManager / persistence context:** tracks managed entities in a transaction.
- **Entity states:** transient (new) → **managed** (tracked) → detached (context closed) → removed.
- **Dirty checking:** changes to managed entities are flushed as UPDATEs automatically at commit. No `save()` needed.
- **Spring Data repositories:** `JpaRepository<Order, Long>` gives CRUD, paging and sorting for free.

```yaml
spring.jpa.hibernate.ddl-auto: validate   # never 'update' in prod — use Flyway/Liquibase
spring.jpa.open-in-view: false            # avoid lazy loading in the web layer
```

**Pitfalls:**
- `ddl-auto=update` in production: uncontrolled schema changes. Use migrations.
- **Open-in-view** (on by default) keeps DB connections open through view rendering and hides N+1 problems. Turn it off.

**Interview Q&A:**
- *Why is calling save() on a managed entity unnecessary?* Dirty checking flushes changes at commit.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-24"></a>
### 3.24 First-level cache

**What:** The **persistence context** itself. Within one transaction or EntityManager, each entity ID maps to **one** object instance.

**Behaviour:**
- `findById(1)` twice in the same transaction gives one SQL query and the same object (`==`).
- Always on, can't be disabled. Lives only as long as the transaction or session.
- Guarantees **repeatable reads in memory** and is the basis of dirty checking.

**Pitfalls:**
- Bulk-loading thousands of entities in one transaction keeps them all in memory. Use pagination, `clear()` in batches, or stateless sessions for batch jobs.
- JPQL bulk `UPDATE`/`DELETE` bypass the context, so in-memory entities go stale. Use `@Modifying(clearAutomatically = true)`.

**Interview Q&A:**
- *Is the first-level cache shared between requests?* No. It's per transaction. Sharing across sessions is the second-level cache.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-25"></a>
### 3.25 Second-level cache

**What:** An optional cache **shared across sessions** (per SessionFactory), backed by Ehcache, Caffeine (JCache), Hazelcast or Infinispan.

**Setup:**
```yaml
spring.jpa.properties.hibernate.cache.use_second_level_cache: true
spring.jpa.properties.hibernate.cache.region.factory_class: jcache
```
```java
@Entity @Cacheable @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
class Country { ... }
```

**Strategies:** `READ_ONLY` (never changes), `NONSTRICT_READ_WRITE`, `READ_WRITE` (soft locks), `TRANSACTIONAL`.
**Query cache** is separate (`hibernate.cache.use_query_cache`) and caches result **IDs**, so it needs the entity cache too.

**Pitfalls:**
- Data updated outside Hibernate (another service, raw SQL) leaves the cache **stale**.
- A local cache across several app instances gives inconsistent views. Use a distributed cache, or cache only reference data.

**Interview Q&A:**
- *What would you put in the second-level cache?* Read-mostly reference data: countries, currencies, product categories. Not orders or balances.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-26"></a>
### 3.26 DTO to table (entity mapping)

**Mapping annotations:**
- `@Table(name, indexes, uniqueConstraints)`, `@Column(nullable, length, unique)`, `@Enumerated(EnumType.STRING)` (**never ORDINAL**, since reordering the enum corrupts data), `@Lob`, `@Transient`.
- **IDs:** `IDENTITY` (DB auto-increment, disables JDBC batch inserts), `SEQUENCE` (best with Postgres, supports batching), `UUID`.
- **Embeddables:** `@Embeddable Address` + `@Embedded` for value objects.
- **Inheritance:** `SINGLE_TABLE` (fast, nullable columns), `JOINED` (normalised, joins), `TABLE_PER_CLASS`.
- **Auditing:** `@CreatedDate`, `@LastModifiedDate` with `@EnableJpaAuditing`.

**DTO ↔ entity:** map by hand or with **MapStruct** (compile-time generated, fast). Use DTO projections for read queries (interface or record projections fetch only the needed columns).

**Interview Q&A:**
- *Why EnumType.STRING?* ORDINAL stores 0/1/2. Adding or reordering enum values silently changes the meaning of existing rows.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-27"></a>
### 3.27 One-to-one

```java
@Entity class User {
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY) Profile profile;
}
@Entity class Profile {
    @OneToOne(fetch = FetchType.LAZY) @JoinColumn(name = "user_id") User user;   // owning side (has the FK)
}
```

**Key points:**
- The **owning side** has the foreign key (`@JoinColumn`). The inverse side uses `mappedBy`.
- `@MapsId` shares the primary key, which is the most efficient one-to-one.
- `@OneToOne` defaults to **EAGER**, so set LAZY explicitly. The inverse side often can't be lazy without `@MapsId` or bytecode enhancement.

**Cascade & orphanRemoval:** `cascade = ALL` propagates persist and remove. `orphanRemoval = true` deletes the child when it's unlinked.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-28"></a>
### 3.28 One-to-many and many-to-many

```java
@Entity class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    List<OrderItem> items = new ArrayList<>();
    void addItem(OrderItem i) { items.add(i); i.setOrder(this); }   // keep both sides in sync
}
@Entity class OrderItem { @ManyToOne(fetch = FetchType.LAZY) Order order; }   // owning side
```

**Key points:**
- `@ManyToOne` is the owning side and defaults to **EAGER**, so set LAZY. `@OneToMany` defaults to LAZY.
- A **unidirectional** `@OneToMany` without `mappedBy` creates a join table or extra UPDATEs. Prefer bidirectional or `@ManyToOne` only.
- **Many-to-many:** `@ManyToMany` with `@JoinTable`. Use a `Set`. If the link has attributes (quantity, date), model it as its **own entity**.

**The N+1 problem:** loading 100 orders, then touching `order.getItems()` in a loop, makes 1 + 100 queries.
**Fixes:** `JOIN FETCH` in JPQL, `@EntityGraph(attributePaths = "items")`, `@BatchSize`/`default_batch_fetch_size`, or DTO projections.

**Pitfalls:**
- `LazyInitializationException`: touching a lazy collection after the transaction closed. Fetch what you need inside the service.
- JOIN FETCH of two collections causes a Cartesian product or `MultipleBagFetchException`. Fetch one per query, or use Sets.

**Interview Q&A:**
- *How do you spot N+1?* Enable SQL logging or Hibernate statistics in tests, or assert query counts (e.g. with the `datasource-proxy` library).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-29"></a>
### 3.29 Derived queries and JPQL

**Derived queries:** Spring generates them from the method name.
```java
List<Order> findByStatusAndCreatedAtAfterOrderByCreatedAtDesc(Status s, Instant t);
Page<Order> findByCustomerId(Long id, Pageable p);
boolean existsByEmail(String email);
```

**JPQL:** queries against **entities and fields**, not tables.
```java
@Query("select o from Order o join fetch o.items where o.customer.id = :cid")
List<Order> withItems(@Param("cid") Long customerId);

@Modifying @Query("update Order o set o.status = :s where o.id in :ids")
int bulkUpdate(@Param("s") Status s, @Param("ids") List<Long> ids);
```

**Projections:** interface (`interface OrderSummary { Long getId(); Status getStatus(); }`) or record/class-based, to fetch only the needed columns.

**Pitfalls:**
- Very long derived method names hurt readability. Switch to `@Query` after 2–3 conditions.
- `Page` runs an extra `count` query. Use `Slice` when you don't need the total.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-30"></a>
### 3.30 Native queries and the Criteria API

**Native SQL:** for database-specific features (window functions, CTEs, JSONB, hints).
```java
@Query(value = "select * from orders where created_at > now() - interval '1 day'", nativeQuery = true)
List<Order> lastDay();
```
- You lose portability and entity-level checks. Map to DTOs or interface projections.

**Criteria API:** builds **type-safe dynamic** queries in code (no string concatenation).
```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> q = cb.createQuery(Order.class);
Root<Order> o = q.from(Order.class);
List<Predicate> ps = new ArrayList<>();
if (status != null) ps.add(cb.equal(o.get("status"), status));
if (from != null)   ps.add(cb.greaterThan(o.get("createdAt"), from));
q.where(ps.toArray(Predicate[]::new));
```

**Pitfall:** never build SQL/JPQL by concatenating user input. That's SQL injection. Use parameters.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-31"></a>
### 3.31 Specification API

**What:** Spring Data's wrapper over Criteria. Small, reusable predicates you combine for **dynamic search filters**.

```java
interface OrderRepo extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {}

class OrderSpecs {
    static Specification<Order> hasStatus(Status s) { return (r, q, cb) -> s == null ? null : cb.equal(r.get("status"), s); }
    static Specification<Order> createdAfter(Instant t) { return (r, q, cb) -> t == null ? null : cb.greaterThan(r.get("createdAt"), t); }
}
repo.findAll(where(hasStatus(status)).and(createdAfter(from)), pageable);
```

**Why:** a search screen with 10 optional filters. Without specs you'd need dozens of repository methods or string-built queries.

**Interview Q&A:**
- *How do you build a search API with optional filters?* Specifications (or Querydsl): each filter is a spec that returns null when unused, combined with `and`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-32"></a>
### 3.32 Common attacks

| Attack | How it works | Defence |
|---|---|---|
| **SQL injection** | User input concatenated into SQL | Parameterised queries/JPA params, least-privilege DB user |
| **XSS** | Injected script runs in another user's browser | Output encoding, Content-Security-Policy, sanitise HTML input |
| **CSRF** | A malicious site makes the browser send an authenticated request (cookies) | CSRF tokens, `SameSite` cookies. Not needed for stateless token APIs |
| **CORS misconfig** | `*` with credentials lets any site call your API | Explicit allowed origins |
| **Brute force / credential stuffing** | Mass login attempts | Rate limiting, lockout, MFA, BCrypt |
| **Broken access control / IDOR** | Changing `/orders/42` to `/orders/43` | Check ownership on **every** request |
| **SSRF** | Server fetches an attacker-chosen URL (cloud metadata) | Allow-list outbound hosts |
| **Session fixation** | Attacker sets the session ID beforehand | New session on login (Spring does this by default) |

**Interview Q&A:**
- *Why can a JWT REST API disable CSRF?* CSRF abuses cookies the browser sends automatically. A bearer token in a header isn't sent automatically. (If you keep JWTs in cookies, CSRF protection is needed again.)

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-33"></a>
### 3.33 Security setup and architecture

**Request flow:**
`DelegatingFilterProxy` → `FilterChainProxy` → **SecurityFilterChain** (ordered filters: CORS, CSRF, authentication, exception translation, authorization) → controller.

**Authentication pieces:**
- **AuthenticationFilter** extracts credentials → **AuthenticationManager** (`ProviderManager`) → **AuthenticationProvider** (e.g. `DaoAuthenticationProvider`) → **UserDetailsService** loads the user → **PasswordEncoder** checks the password.
- On success, the `Authentication` is stored in the **SecurityContext** (thread-local `SecurityContextHolder`).

**Modern config (Spring Security 6, no `WebSecurityConfigurerAdapter`):**
```java
@Bean SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http.csrf(c -> c.disable())
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(a -> a.requestMatchers("/public/**").permitAll()
                                     .requestMatchers("/admin/**").hasRole("ADMIN")
                                     .anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}
```

**Interview Q&A:**
- *Authentication vs authorisation?* AuthN = who you are (login). AuthZ = what you're allowed to do (roles, permissions).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-34"></a>
### 3.34 Users, form login and basic auth

**Users:**
- `InMemoryUserDetailsManager` for demos. `JdbcUserDetailsManager` or a custom `UserDetailsService` backed by your user table in real apps.
- **Passwords:** `BCryptPasswordEncoder` (adaptive, salted). `DelegatingPasswordEncoder` stores the `{bcrypt}` prefix so algorithms can be upgraded. Never plain text or SHA-256 alone.

**Form-based login:** a login page posts credentials. On success the server creates an **HTTP session** (JSESSIONID cookie). Stateful, needs CSRF protection. Good for server-rendered web apps.

**Basic authentication:** `Authorization: Basic base64(user:pass)` on **every** request. Simple but sends credentials constantly, so HTTPS only. Fine for internal tools and service accounts, not for public user-facing APIs.

**Interview Q&A:**
- *Why BCrypt instead of SHA-256?* BCrypt is deliberately slow and salted, with a tunable cost. That makes brute-forcing leaked hashes expensive. A fast hash is bad for passwords.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-35"></a>
### 3.35 JWT authentication

**What:** JSON Web Token: `header.payload.signature`, Base64URL-encoded. **Signed, not encrypted**, so anyone can read the payload.

**Flow:**
1. Login with credentials → the server issues an **access token** (short TTL, 5–15 min) and a **refresh token** (long TTL, stored securely).
2. The client sends `Authorization: Bearer <access token>` on each request.
3. The server (a filter or `oauth2ResourceServer`) **validates the signature**, `exp`, `iss`, `aud` → builds the Authentication from the claims (sub, roles).
4. When it expires, the refresh token gets a new access token. Rotate refresh tokens on use.

**Signing:** HS256 (shared secret, single service) vs **RS256/ES256** (private key signs, public key verifies, so many services can validate via JWKS).

**Pitfalls:**
- Stateless means you **can't revoke** easily. Keep access tokens short, and keep a deny-list or token version for logout and compromise.
- Secrets or PII in the payload. It's readable by anyone.
- Accepting `alg: none` or not checking `aud`.
- Storing tokens in `localStorage` exposes them to XSS. Prefer httpOnly cookies (then add CSRF protection) or in-memory storage.

**Interview Q&A:**
- *How do you log out a JWT user?* Delete the client tokens, revoke the refresh token server-side, keep the access TTL short, and optionally deny-list the `jti` until it expires.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-36"></a>
### 3.36 OAuth 2.0 in Spring

**Roles:** Resource owner (user) · Client (app) · Authorization server (Keycloak, Okta, Spring Authorization Server) · Resource server (your API).

**Grant types:**
| Grant | Use |
|---|---|
| **Authorization code + PKCE** | Users logging into web and mobile apps |
| **Client credentials** | Service-to-service (no user) |
| Refresh token | Get new access tokens |
| ~~Implicit~~, ~~Password~~ | Deprecated, don't use |

**In Spring:**
- **Resource server:** `spring.security.oauth2.resourceserver.jwt.issuer-uri=...`. It validates JWTs using the issuer's JWKS.
- **OAuth2 login (client):** `spring-boot-starter-oauth2-client` + `oauth2Login()`. "Login with Google/GitHub".
- **OIDC** adds an **ID token** (who the user is) on top of OAuth2 (what the app may access).

**Interview Q&A:**
- *Why PKCE?* It stops a stolen authorization code being exchanged by an attacker. The client proves it holds the original secret verifier.
- *OAuth vs OIDC?* OAuth = authorisation (access tokens). OIDC = authentication layer (ID token, userinfo).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-37"></a>
### 3.37 Method security

```java
@EnableMethodSecurity   // Spring Security 6
@Service class AccountService {
    @PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.name")
    Account get(String ownerId) { ... }

    @PostAuthorize("returnObject.owner == authentication.name")
    Statement statement(long id) { ... }
}
```

**Options:** `@PreAuthorize` (before the call, SpEL), `@PostAuthorize` (check the returned object), `@PreFilter`/`@PostFilter` (filter collections), `@Secured`, `@RolesAllowed`.

**Why:** URL rules alone can't express "users may only see **their own** accounts". Method security puts the rule next to the business operation.

**Pitfalls:** proxy-based, so the self-invocation rule from AOP applies. `hasRole('ADMIN')` expects the authority `ROLE_ADMIN`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-38"></a>
### 3.38 Actuator

**What:** Production-ready endpoints for health, metrics and management.

| Endpoint | Purpose |
|---|---|
| `/actuator/health` | UP/DOWN with component checks (DB, disk, Kafka). **Liveness/readiness** groups for Kubernetes probes |
| `/actuator/metrics`, `/actuator/prometheus` | Micrometer metrics (JVM, HTTP latency, pools) |
| `/actuator/info` | Build and git info |
| `/actuator/loggers` | View or **change log levels at runtime** |
| `/actuator/env`, `/configprops` | Configuration (sensitive!) |
| `/actuator/threaddump`, `/heapdump` | Diagnostics (**very sensitive**) |

```yaml
management.endpoints.web.exposure.include: health,info,prometheus
management.endpoint.health.probes.enabled: true
```

**Pitfalls:** exposing `/env` or `/heapdump` publicly leaks secrets and memory. Expose a minimal set, put management on a separate port, and secure it.

**Interview Q&A:**
- *Liveness vs readiness?* Liveness failing → Kubernetes **restarts** the pod. Readiness failing → it **stops sending traffic**. Don't put DB checks in liveness, or a DB blip restarts every pod.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s3-39"></a>
### 3.39 Spring AI

**What:** Spring abstractions for building apps on LLMs, portable across providers (Anthropic, OpenAI, Ollama, Azure, Bedrock…).

**Core pieces:**
- **`ChatClient`:** fluent API for prompts → responses. System and user messages, streaming.
- **Prompt templates** with variables. **Structured output:** map the response straight to a Java record (`.entity(MyRecord.class)`).
- **Advisors:** interceptors around calls (chat memory, logging, RAG).
- **Embeddings + `VectorStore`** (PGVector, Redis, Chroma…) for **RAG**: chunk docs → embed → store → retrieve relevant chunks → add them to the prompt.
- **Tool / function calling:** the model asks your Java method for data (`@Tool`), and Spring runs it and returns the result.

```java
String answer = chatClient.prompt()
    .system("You are a helpful banking assistant.")
    .user(question)
    .call().content();
```

**Pitfalls:** prompt injection through retrieved documents or user input, cost and latency per call, non-deterministic output (validate structured responses), and leaking PII to external providers.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
