<a id="top"></a>
# 5. Testing — JUnit 5 & Mockito

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
**JUnit 5**
- [5.1 Unit testing introduction](#s5-1)
- [5.2 JUnit 5 architecture](#s5-2)
- [5.3 Lifecycle, parallel execution and AAA](#s5-3)
- [5.4 Assertions](#s5-4)
- [5.5 Extensions](#s5-5)
- [5.6 Repeated tests](#s5-6)
- [5.7 Parameterised tests](#s5-7)
- [5.8 Assumptions](#s5-8)
- [5.9 Tagging and filtering](#s5-9)
- [5.10 Test suites](#s5-10)
- [5.11 Dynamic tests (@TestFactory)](#s5-11)

**Mockito**
- [5.12 Mockito introduction](#s5-12)
- [5.13 Mockito architecture](#s5-13)
- [5.14 @InjectMocks](#s5-14)
- [5.15 @Mock in constructors and method parameters](#s5-15)
- [5.16 Stubbing](#s5-16)
- [5.17 Verifying](#s5-17)

**Beyond unit tests**
- [5.18 Spring Boot test slices and Testcontainers](#s5-18)

---

<a id="s5-1"></a>
### 5.1 Unit testing introduction

**What:** Testing the smallest unit (a class or method) **in isolation**, fast and repeatable, with dependencies replaced by test doubles.

**Test pyramid:** many **unit** tests (milliseconds) → fewer **integration** tests (real DB or broker) → very few **end-to-end** tests (slow, brittle).

**Good unit tests are FIRST:** **F**ast, **I**ndependent (no ordering), **R**epeatable (no clock, network or random dependence), **S**elf-validating (pass/fail, no reading logs), **T**imely (written with the code).

**Test doubles:** **dummy** (unused filler), **stub** (canned answers), **mock** (verifies interactions), **spy** (real object, partially stubbed), **fake** (working lightweight implementation, e.g. an in-memory repository).

**Interview Q&A:**
- *What makes code hard to test?* Static calls, `new` inside methods, hidden time and randomness, God classes. DI and small classes fix most of it.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-2"></a>
### 5.2 JUnit 5 architecture

**JUnit 5 = Platform + Jupiter + Vintage:**
- **JUnit Platform:** the foundation that **launches** tests on the JVM. IDEs, Maven Surefire and Gradle talk to it. Provides the `TestEngine` API.
- **JUnit Jupiter:** the new programming model (`@Test`, `@BeforeEach`…) and the **extension** model, plus the Jupiter test engine.
- **JUnit Vintage:** an engine to run old JUnit 3/4 tests on the platform.

**Differences from JUnit 4:** `@Before` → `@BeforeEach`, `@BeforeClass` → `@BeforeAll`, `@Ignore` → `@Disabled`, `@Category` → `@Tag`, `@RunWith`/`@Rule` → **`@ExtendWith`**. Test classes and methods don't need to be `public`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-3"></a>
### 5.3 Lifecycle, parallel execution and AAA

**Lifecycle annotations:**
| Annotation | Runs |
|---|---|
| `@BeforeAll` / `@AfterAll` | Once per class (**static** unless `@TestInstance(PER_CLASS)`) |
| `@BeforeEach` / `@AfterEach` | Around every test |
| `@Test`, `@DisplayName`, `@Disabled`, `@Nested` | Test definition and grouping |

- **Default instance lifecycle: PER_METHOD.** A new test class instance for each test, so no shared state leaks.
- **`@Nested`** inner classes group related scenarios ("when cart is empty" → tests).

**Parallel execution:** `junit.jupiter.execution.parallel.enabled=true` + `mode.default=concurrent`. Tests must not share mutable state. Use `@ResourceLock` / `@Execution(SAME_THREAD)` for those that do.

**AAA pattern:**
```java
@Test
void appliesTenPercentDiscountForGoldCustomers() {
    // Arrange
    Cart cart = new Cart(Customer.gold()); cart.add(item(100));
    // Act
    Money total = pricing.total(cart);
    // Assert
    assertEquals(Money.of(90), total);
}
```
One behaviour per test, with a name that reads like a requirement.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-4"></a>
### 5.4 Assertions

| Assertion | Use |
|---|---|
| `assertEquals(expected, actual, "msg")` | Equality (expected value first) |
| `assertTrue / assertFalse / assertNull / assertNotNull` | Conditions |
| `assertSame / assertNotSame` | Reference identity |
| `assertArrayEquals`, `assertIterableEquals`, `assertLinesMatch` | Collections |
| **`assertThrows(Ex.class, () -> …)`** | Returns the exception so you can check its message |
| `assertDoesNotThrow` | |
| **`assertAll(...)`** | Groups assertions and reports **all** failures, not just the first |
| `assertTimeout` / `assertTimeoutPreemptively` | Time limits |

```java
var ex = assertThrows(InsufficientFunds.class, () -> account.withdraw(500));
assertEquals("balance 100 < 500", ex.getMessage());
```
Use a `Supplier<String>` message (`() -> "..."`) to build failure messages lazily. Many teams use **AssertJ** (`assertThat(list).hasSize(3).contains(x)`) for fluent, readable assertions.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-5"></a>
### 5.5 Extensions

**What:** JUnit 5's single plug-in model, which replaces JUnit 4 runners and rules. Register with `@ExtendWith(...)` or `@RegisterExtension` (field, configurable).

**Extension points:** lifecycle callbacks (`BeforeEachCallback`, `AfterAllCallback`…), `ParameterResolver` (inject test-method parameters), `TestExecutionExceptionHandler`, `ExecutionCondition` (enable/disable), `TestInstancePostProcessor`.

**Common ones:** `MockitoExtension`, `SpringExtension` (included in `@SpringBootTest`), Testcontainers, `TempDirectory` (`@TempDir`).

```java
class TimingExtension implements BeforeTestExecutionCallback, AfterTestExecutionCallback { ... }
@ExtendWith(TimingExtension.class) class SlowServiceTest { ... }
```

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-6"></a>
### 5.6 Repeated tests

```java
@RepeatedTest(value = 5, name = "{displayName} {currentRepetition}/{totalRepetitions}")
void generatesUniqueIds(RepetitionInfo info) { ... }
```
- Uses: catching **flaky** behaviour, randomness, concurrency bugs that appear intermittently.
- Each repetition gets the full `@BeforeEach`/`@AfterEach` lifecycle.
- `failureThreshold` (5.10+) stops after N failures.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-7"></a>
### 5.7 Parameterised tests

```java
@ParameterizedTest(name = "{0} is a valid email")
@ValueSource(strings = {"a@b.com", "x.y@z.io"})
void validEmails(String email) { assertTrue(validator.isValid(email)); }

@ParameterizedTest
@CsvSource({ "100, GOLD, 90", "100, SILVER, 95", "100, NONE, 100" })
void discount(int price, Tier tier, int expected) { ... }

@ParameterizedTest
@MethodSource("edgeCases")
void edges(Order o, Status expected) { ... }
static Stream<Arguments> edgeCases() { return Stream.of(Arguments.of(emptyOrder(), REJECTED)); }
```

**Sources:** `@ValueSource`, `@EnumSource`, `@CsvSource`, `@CsvFileSource`, `@MethodSource`, `@ArgumentsSource`, `@NullAndEmptySource`.
**Why:** one test method covers a whole table of cases (boundaries, equivalence classes) with clear reporting per case.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-8"></a>
### 5.8 Assumptions

**What:** Conditions that **abort** (skip) a test instead of failing it when they don't hold.

```java
@Test void onlyOnCi() {
    assumeTrue("CI".equals(System.getenv("ENV")));
    ...
}
assumingThat(isLinux(), () -> assertEquals("/", File.separator));   // partial
```
**Declarative alternatives:** `@EnabledOnOs`, `@EnabledOnJre`, `@EnabledIfEnvironmentVariable`, `@EnabledIfSystemProperty`, `@DisabledIf("customCondition")`.

**Assertion vs assumption:** an assertion failure = the code is broken. An assumption failure = this test doesn't apply in this environment.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-9"></a>
### 5.9 Tagging and filtering

```java
@Tag("slow") @Tag("integration") class PaymentGatewayIT { ... }
```
- Run subsets by tag in the build:
  ```xml
  <!-- maven-surefire-plugin -->
  <configuration><groups>fast</groups><excludedGroups>slow</excludedGroups></configuration>
  ```
  Gradle: `useJUnitPlatform { includeTags 'fast' }`.
- Tag expressions: `fast & !flaky`, `integration | e2e`.
- Typical split: unit tests on every commit, `integration`-tagged tests in a later CI stage.
- Custom composed annotation: `@Tag("slow") @Test @interface SlowTest {}`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-10"></a>
### 5.10 Test suites

```java
@Suite
@SuiteDisplayName("Checkout regression")
@SelectPackages("com.shop.checkout")
@IncludeTags("regression")
class CheckoutSuite {}
```
- Needs `junit-platform-suite`. Selectors: `@SelectClasses`, `@SelectPackages`; filters: `@IncludeTags`, `@ExcludeClassNamePatterns`.
- Useful to run a named group from the IDE. In CI, tag filtering in the build tool is usually simpler.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-11"></a>
### 5.11 Dynamic tests (@TestFactory)

**What:** Tests generated **at runtime** from data. The method returns a stream or collection of `DynamicTest`.

```java
@TestFactory
Stream<DynamicTest> everyCurrencyRoundsCorrectly() {
    return Currency.getAvailableCurrencies().stream()
        .map(c -> dynamicTest("rounds " + c, () -> assertValidRounding(c)));
}
```
- **vs @ParameterizedTest:** parameterised = a fixed method with argument sources. Dynamic = arbitrary generated tests (from files, an API spec, directory scans), and they can be nested with `dynamicContainer`.
- Caveat: `@BeforeEach`/`@AfterEach` run around the **factory**, not around each dynamic test.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-12"></a>
### 5.12 Mockito introduction

**What:** A library to create **mocks** of dependencies so you can test a class in isolation, controlling what dependencies return and checking how they were called.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock PaymentClient payments;
    @Mock OrderRepository repo;
    @InjectMocks OrderService service;

    @Test void chargesAndSaves() {
        when(payments.charge(any())).thenReturn(Receipt.ok("r1"));
        service.place(order());
        verify(repo).save(argThat(o -> o.status() == PAID));
    }
}
```
**What not to mock:** value objects, the class under test, types you don't own deeply (wrap them instead), and things like `List`.
**Strict stubs** (the default with `MockitoExtension`) fail on unused stubbing, which keeps tests clean.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-13"></a>
### 5.13 Mockito architecture

- A mock is a **runtime-generated subclass/proxy** (built with **ByteBuddy**) that intercepts every method call.
- Each call is recorded as an **invocation**. Stubbing registers **answers** matched by method + argument matchers. Verification queries the recorded invocations.
- **Default answers:** `null` for objects, `0`/`false` for primitives, **empty collections**, empty `Optional`. `RETURNS_DEEP_STUBS` chains mocks (a code smell).
- **inline mock maker** (the default since Mockito 5) uses instrumentation, so it can mock **final classes and methods**, and **static methods** via `mockStatic` (use sparingly).
- **Spies** wrap a real object: unstubbed calls hit real code.

**Interview Q&A:**
- *Mock vs spy?* A mock is fully fake (defaults everywhere). A spy is the real object with selected methods stubbed. Use `doReturn().when(spy).method()` to avoid calling the real method while stubbing.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-14"></a>
### 5.14 @InjectMocks

**What:** Creates the class under test and injects the `@Mock`/`@Spy` fields into it.

**Injection order (first one that works wins):**
1. **Constructor injection:** the biggest constructor. Mocks matched by type.
2. **Setter/property injection:** a no-arg constructor, then setters.
3. **Field injection:** sets fields by type, then by name.

**Pitfalls:**
- It **fails silently**: if it can't inject something, that field stays `null` and you get NPEs later.
- With constructor injection, if a constructor argument has no matching mock, Mockito passes `null`.
- Clearer alternative: build the object yourself, `new OrderService(payments, repo)`, in `@BeforeEach`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-15"></a>
### 5.15 @Mock in constructors and method parameters

```java
@ExtendWith(MockitoExtension.class)
class TransferTest {
    private final AccountRepo repo;
    TransferTest(@Mock AccountRepo repo) { this.repo = repo; }   // constructor parameter

    @Test void audits(@Mock AuditLog audit) {                    // method parameter: scoped to this test
        new TransferService(repo, audit).transfer(a, b, 100);
        verify(audit).record(any());
    }
}
```
- Works because `MockitoExtension` is a JUnit 5 **ParameterResolver**.
- **Method-parameter mocks** keep a mock local to the one test that needs it, so the class doesn't fill up with fields used by a single test.
- Programmatic alternative: `mock(AccountRepo.class)`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-16"></a>
### 5.16 Stubbing

```java
when(repo.findById(1L)).thenReturn(Optional.of(order));
when(client.call()).thenThrow(new TimeoutException())     // consecutive calls
                   .thenReturn(response);
when(calc.apply(anyInt())).thenAnswer(inv -> inv.getArgument(0, Integer.class) * 2);

doThrow(new IOException()).when(storage).delete(any());   // void methods
doReturn(42).when(spy).expensive();                        // spies: don't call the real method
```

**Argument matchers:** `any()`, `anyString()`, `eq(x)`, `argThat(pred)`, `isNull()`. **Rule:** if one argument uses a matcher, **all** must (`eq(5), any()`).
**Pitfalls:** over-stubbing (testing the mock, not the code), and strict-stubs failures for stubs a test doesn't use (lenient only when it's justified).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-17"></a>
### 5.17 Verifying

```java
verify(repo).save(order);                       // exactly once
verify(mailer, times(2)).send(any());
verify(mailer, never()).send(argThat(m -> m.isSpam()));
verify(audit, atLeastOnce()).record(any());
verifyNoMoreInteractions(repo);                 // use sparingly — brittle

InOrder inOrder = inOrder(payments, repo);
inOrder.verify(payments).charge(any());
inOrder.verify(repo).save(any());

ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(repo).save(captor.capture());
assertEquals(PAID, captor.getValue().status());
```

**Stub or verify?** Stub **queries** (methods that return data). Verify **commands** (side effects like save or send). Don't verify calls you already stubbed and asserted through the result.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s5-18"></a>
### 5.18 Spring Boot test slices and Testcontainers

| Annotation | Loads | Use |
|---|---|---|
| `@SpringBootTest` | The full context | End-to-end within the app (slowest) |
| `@WebMvcTest(Controller.class)` | MVC layer only + `MockMvc` | Controller mapping, validation, JSON |
| `@DataJpaTest` | JPA + repositories (transaction rolled back after each test) | Queries and mappings |
| `@JsonTest`, `@RestClientTest` | Narrow slices | Serialisation, HTTP clients |

- **`@MockitoBean`** (Boot 3.4+, previously `@MockBean`) replaces a bean in the context with a mock.
- **Testcontainers:** real Postgres, Kafka or Redis in Docker for integration tests. `@ServiceConnection` (Boot 3.1+) wires the connection properties automatically.
```java
@SpringBootTest @Testcontainers
class OrderRepoIT {
    @Container @ServiceConnection static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");
}
```
**Why:** H2 behaves differently from Postgres (SQL dialect, locking). Test against the real engine.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
