<a id="top"></a>
# 6. Low-Level Design

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
**SOLID**
- [6.1 Single responsibility](#s6-1) · [6.2 Open/closed](#s6-2) · [6.3 Liskov substitution](#s6-3) · [6.4 Interface segregation](#s6-4) · [6.5 Dependency inversion](#s6-5)

**Creational patterns**
- [6.6 Singleton](#s6-6) · [6.7 Factory method](#s6-7) · [6.8 Abstract factory](#s6-8) · [6.9 Builder](#s6-9) · [6.10 Prototype](#s6-10) · [6.11 Object pool](#s6-11)

**Structural patterns**
- [6.12 Adapter](#s6-12) · [6.13 Decorator](#s6-13) · [6.14 Proxy](#s6-14) · [6.15 Composite](#s6-15) · [6.16 Facade](#s6-16) · [6.17 Bridge](#s6-17) · [6.18 Flyweight](#s6-18)

**Behavioral patterns**
- [6.19 Strategy](#s6-19) · [6.20 Observer](#s6-20) · [6.21 State](#s6-21) · [6.22 Chain of responsibility](#s6-22) · [6.23 Command](#s6-23) · [6.24 Template method](#s6-24) · [6.25 Iterator](#s6-25) · [6.26 Mediator](#s6-26) · [6.27 Memento](#s6-27) · [6.28 Visitor](#s6-28) · [6.29 Interpreter](#s6-29) · [6.30 Null object](#s6-30) · [6.31 MVC](#s6-31)

**LLD interview problems**
- [6.32 How to approach an LLD round](#s6-32)
- [6.33 Parking lot](#s6-33) · [6.34 Tic-tac-toe](#s6-34) · [6.35 Elevator system](#s6-35) · [6.36 Car rental](#s6-36) · [6.37 Snake and ladder](#s6-37)
- [6.38 HashMap implementation](#s6-38) · [6.39 BookMyShow](#s6-39) · [6.40 Vending machine](#s6-40) · [6.41 ATM](#s6-41) · [6.42 Chess](#s6-42)
- [6.43 File system](#s6-43) · [6.44 Splitwise](#s6-44) · [6.45 Cricbuzz](#s6-45) · [6.46 Inventory / order management](#s6-46) · [6.47 Word processor](#s6-47)
- [6.48 Undo/redo](#s6-48) · [6.49 Apply coupons to a cart](#s6-49) · [6.50 Payment gateway](#s6-50) · [6.51 Meeting scheduler](#s6-51)

---

<a id="s6-1"></a>
### 6.1 Single responsibility principle (SRP)

**A class should have one reason to change**, meaning one actor or stakeholder it serves.
- ❌ `Invoice` that calculates totals **and** prints a PDF **and** saves to the DB.
- ✅ `Invoice` (data + totals), `InvoicePrinter`, `InvoiceRepository`.
- **Smell:** class names like `Manager`/`Helper`/`Util`, 1,000-line classes, methods that touch unrelated fields.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-2"></a>
### 6.2 Open/closed principle (OCP)

**Open for extension, closed for modification.** Add behaviour by adding code, not by editing tested code.
- ❌ `if (type == UPI) … else if (type == CARD) …` growing with every new payment type.
- ✅ A `PaymentMethod` interface plus one class per method. A new method is a new class (Strategy, polymorphism).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-3"></a>
### 6.3 Liskov substitution principle (LSP)

**Subtypes must be usable anywhere the base type is**, without breaking expectations.
- Classic violation: `Square extends Rectangle`. Setting the width also changes the height, which breaks code that relies on `Rectangle` behaviour.
- Another: `Ostrich extends Bird` where `Bird.fly()` throws. Fix by splitting the abstraction (`FlyingBird`).
- **Rule of thumb:** a subclass may not strengthen preconditions, weaken postconditions, or throw new unexpected exceptions.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-4"></a>
### 6.4 Interface segregation principle (ISP)

**Clients shouldn't depend on methods they don't use.** Prefer small, focused interfaces.
- ❌ `Worker { work(); eat(); }`, so a `Robot` has to implement `eat()`.
- ✅ `Workable`, `Eatable`. Spring Data's `CrudRepository` vs `PagingAndSortingRepository` is a real example.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-5"></a>
### 6.5 Dependency inversion principle (DIP)

**High-level modules depend on abstractions, not on low-level details.**
- ❌ `OrderService` creates `new MySqlOrderRepository()`.
- ✅ `OrderService` depends on an `OrderRepository` interface, and the implementation is injected (DI is the mechanism, DIP is the principle).
- Benefit: swap the DB, mock in tests, and keep the business logic free of infrastructure.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-6"></a>
### 6.6 Singleton

**Intent:** exactly one instance with a global access point.
**Use:** configuration, registries, connection pools (in Spring, beans are singletons by default, so you rarely hand-roll this).
**Implementations and pitfalls:** see [Java 4.13](04-java.md#s4-13) (enum, holder idiom, double-checked locking with `volatile`).
**Criticism:** hidden global state, hard to test. Prefer DI-managed singletons.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-7"></a>
### 6.7 Factory method

**Intent:** let a method decide **which concrete class** to create, so callers depend only on the interface.
```java
interface Notification { void send(String msg); }
class NotificationFactory {
    static Notification of(Channel c) {
        return switch (c) { case EMAIL -> new EmailNotification(); case SMS -> new SmsNotification(); case PUSH -> new PushNotification(); };
    }
}
```
- **Simple factory** (one static method, above) vs the **GoF factory method** (an abstract `createX()` overridden by subclasses).
- **Use:** creation varies by type or config, or creation logic is complex.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-8"></a>
### 6.8 Abstract factory

**Intent:** create **families of related objects** that must be used together, without naming concrete classes.
```java
interface UiFactory { Button button(); Checkbox checkbox(); }
class DarkFactory implements UiFactory { ... }   // DarkButton + DarkCheckbox
class LightFactory implements UiFactory { ... }
```
- **vs factory method:** a factory method creates **one** product. An abstract factory creates a **matching set** (luxury car + luxury tyres, AWS client + AWS storage).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-9"></a>
### 6.9 Builder

**Intent:** build a complex object **step by step**, avoiding "telescoping" constructors with many optional parameters.
```java
Pizza p = Pizza.builder().size(LARGE).cheese(true).topping("olive").build();
```
- Produces **immutable** objects. `build()` can **validate** the combination.
- In real code: Lombok `@Builder`, `HttpRequest.newBuilder()`, `StringBuilder`, Spring's `RestClient.builder()`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-10"></a>
### 6.10 Prototype

**Intent:** create new objects by **copying** an existing prototype, when construction is expensive or configuration-heavy.
- Implement a copy method or copy constructor. Decide **deep vs shallow** copy (shallow shares mutable children, which is a bug source).
- Java's `Cloneable`/`clone()` is widely considered broken. Prefer copy constructors or static `copyOf`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-11"></a>
### 6.11 Object pool

**Intent:** reuse a fixed set of **expensive objects** instead of creating and destroying them each time.
- Examples: **DB connection pools** (HikariCP), thread pools, socket and HTTP connection pools.
- Mechanics: `acquire()` (wait or fail if empty, with a timeout) → use → **`release()` in finally**. Validate an object before reuse. Size limits.
- Pitfall: leaked objects (never released) exhaust the pool. Pools need leak detection and timeouts.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-12"></a>
### 6.12 Adapter

**Intent:** make an **incompatible interface** fit the one your code expects (a wrapper that translates calls).
```java
interface PaymentGateway { Receipt pay(Money m); }
class LegacyBankAdapter implements PaymentGateway {
    private final LegacyBankApi api;
    public Receipt pay(Money m) { return toReceipt(api.doTransfer(m.paise(), "INR")); }
}
```
- Use when integrating third-party or legacy APIs without changing your domain. Examples: `Arrays.asList`, `InputStreamReader` (bytes → chars).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-13"></a>
### 6.13 Decorator

**Intent:** add behaviour to an object **dynamically** by wrapping it in objects with the **same interface**. This avoids a subclass explosion.
```java
Coffee c = new MilkDecorator(new SugarDecorator(new BasicCoffee()));
c.cost();   // each wrapper adds its price then delegates
```
- Examples: Java I/O (`new BufferedInputStream(new FileInputStream(f))`), Spring caching, retry and transaction wrappers.
- **vs proxy:** a decorator **adds features** (often stacked). A proxy **controls access** to the same feature.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-14"></a>
### 6.14 Proxy

**Intent:** a stand-in object that **controls access** to the real one.
- **Virtual proxy:** lazy loading (Hibernate lazy entities). **Protection proxy:** access checks. **Remote proxy:** a local object hiding network calls (Feign). **Caching proxy.**
- Spring AOP (`@Transactional`, `@Cacheable`, `@Async`) is proxies all the way down. That's why self-invocation skips them.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-15"></a>
### 6.15 Composite

**Intent:** treat **individual objects and groups the same way** through a tree structure.
```java
interface FileSystemNode { long size(); }
class File implements FileSystemNode { long size() { return bytes; } }
class Directory implements FileSystemNode {
    List<FileSystemNode> children;
    long size() { return children.stream().mapToLong(FileSystemNode::size).sum(); }
}
```
- Examples: file systems, UI component trees, org charts, menus, expression trees.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-16"></a>
### 6.16 Facade

**Intent:** a **simple front door** over a complex subsystem.
- `CheckoutFacade.placeOrder()` internally calls inventory, pricing, payment, shipping and notification.
- Reduces coupling: clients depend on one simple API. Spring's `JdbcTemplate` is a facade over raw JDBC.
- Pitfall: letting the facade become a God object. Keep it a coordinator only.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-17"></a>
### 6.17 Bridge

**Intent:** separate an **abstraction** from its **implementation** so the two can vary independently. Composition instead of an N×M class explosion.
- Example: `Shape` (Circle, Square) × `Renderer` (Vector, Raster). Without a bridge you'd need 4 classes. With one, `Shape` holds a `Renderer`.
- Example: `Notification` (Alert, Reminder) × `Sender` (Email, SMS). JDBC: the `DriverManager` API bridged to vendor drivers.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-18"></a>
### 6.18 Flyweight

**Intent:** share common (**intrinsic**) state across many similar objects to save memory. The unique (**extrinsic**) state is passed in.
- **Word processor:** millions of characters. Font and glyph objects are shared (intrinsic), and position is extrinsic.
- Games: tree or bullet types shared across thousands of instances.
- Java examples: the `Integer.valueOf` cache, the String pool.
- Flyweights must be **immutable** to be shared safely.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-19"></a>
### 6.19 Strategy

**Intent:** a family of interchangeable **algorithms** behind one interface, chosen at runtime.
```java
interface PricingStrategy { Money price(Cart c); }
class FestivePricing implements PricingStrategy { ... }
class RegularPricing implements PricingStrategy { ... }
checkout.setPricing(isFestive ? festive : regular);
```
- Removes if/else chains (OCP). In Spring: inject `Map<String, Strategy>` and pick by key.
- Examples: payment methods, sorting (`Comparator`), parking-spot allocation, shipping cost, elevator scheduling.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-20"></a>
### 6.20 Observer

**Intent:** when a **subject** changes, all registered **observers** are notified automatically (one-to-many).
```java
interface ScoreObserver { void update(Score s); }
class Match { List<ScoreObserver> observers; void ballBowled(...) { observers.forEach(o -> o.update(score)); } }
```
- Examples: live score updates, stock tickers, UI events, Spring `ApplicationEventPublisher` + `@EventListener`, and Kafka at system scale.
- Pitfalls: memory leaks from observers that are never removed, and a slow observer blocking the subject (notify asynchronously).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-21"></a>
### 6.21 State

**Intent:** an object **changes its behaviour when its internal state changes**. Each state is a class, instead of big switch statements.
```java
interface VendingState { void insertCoin(Machine m, int amt); void selectItem(Machine m, String code); void dispense(Machine m); }
class IdleState implements VendingState { ... }       // insertCoin → HasMoneyState
class HasMoneyState implements VendingState { ... }   // selectItem → DispensingState
```
- Examples: vending machine, ATM, order lifecycle, traffic light, document workflow.
- **vs strategy:** strategy is chosen **by the client**. State transitions happen **inside** the object as events occur.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-22"></a>
### 6.22 Chain of responsibility

**Intent:** pass a request along a **chain of handlers**. Each one handles it, passes it on, or both.
```java
abstract class CashHandler {
    CashHandler next; int note;
    void dispense(int amount) {
        int count = amount / note, rest = amount % note;
        give(count, note);
        if (rest > 0 && next != null) next.dispense(rest);
    }
}
// 2000 → 500 → 100 handlers chained
```
- Examples: ATM cash dispensing, logging levels, servlet filters and the Spring Security filter chain, approval workflows (manager → director → CFO).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-23"></a>
### 6.23 Command

**Intent:** wrap a request as an **object** (action + parameters) so it can be queued, logged, retried and **undone**.
```java
interface Command { void execute(); void undo(); }
class TypeTextCommand implements Command { ... }
Deque<Command> history; // execute → push; undo → pop().undo()
```
- Examples: undo/redo in editors, task queues, macro recording, remote controls, transactional scripts.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-24"></a>
### 6.24 Template method

**Intent:** a base class defines the **skeleton of an algorithm**, and subclasses fill in specific steps.
```java
abstract class ReportGenerator {
    final void generate() { fetch(); format(); export(); }   // fixed order
    abstract void fetch(); abstract void format();
    void export() { ... }                                     // default hook
}
```
- Examples: `JdbcTemplate`/`RestTemplate` (fixed flow, you supply the callback), `AbstractList`, data-import pipelines.
- Downside: inheritance-based. The strategy pattern is the composition alternative.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-25"></a>
### 6.25 Iterator

**Intent:** traverse a collection **without exposing its internal structure**.
- Java's `Iterator` (`hasNext`, `next`, `remove`) and `Iterable` (for-each).
- Custom iterators: walking a tree (in-order), paging through API results lazily, iterating a playlist in shuffle order.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-26"></a>
### 6.26 Mediator

**Intent:** objects talk **through a central mediator** instead of to each other. Many-to-many becomes many-to-one.
- Examples: a chat room (users send to the room, which routes), an air-traffic control tower, UI form components coordinating via a dialog controller, an auction house.
- Risk: the mediator becomes a God object.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-27"></a>
### 6.27 Memento

**Intent:** capture and restore an object's **state snapshot** without breaking encapsulation.
- **Originator** (creates and restores mementos), **Memento** (opaque snapshot), **Caretaker** (stores the history).
- Examples: editor undo (snapshot-based), game save points, rolling back a form.
- **vs command-based undo:** memento stores state (simple, memory-heavy). Command stores the inverse operation (compact, more complex).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-28"></a>
### 6.28 Visitor

**Intent:** add **new operations** to a stable class hierarchy without modifying those classes (**double dispatch**).
```java
interface ShapeVisitor<R> { R visit(Circle c); R visit(Square s); }
interface Shape { <R> R accept(ShapeVisitor<R> v); }
// AreaVisitor, ExportToXmlVisitor — new operations, no change to shapes
```
- Good when the **types are stable but operations change often** (compilers/AST, tax calculation over item types).
- Modern Java alternative: **sealed interfaces + pattern-matching switch** ([4.42](04-java.md#s4-42)).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-29"></a>
### 6.29 Interpreter

**Intent:** represent a simple **grammar** as classes and evaluate sentences in it.
- Each rule is a class (`NumberExpr`, `AddExpr`, `AndExpr`) with `interpret(context)`, forming an expression tree (it's a Composite).
- Examples: rule engines, simple query or filter languages, Spring Expression Language (SpEL), regex engines.
- Only practical for **small** grammars. Use a parser generator (ANTLR) for real languages.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-30"></a>
### 6.30 Null object

**Intent:** instead of returning `null`, return an object with **do-nothing default behaviour**, so callers skip null checks.
```java
class NoDiscount implements Discount { public Money apply(Money m) { return m; } }
Discount d = coupons.find(code).orElse(new NoDiscount());
```
- Examples: a `NoOpLogger`, a `GuestUser`, empty collections instead of null.
- Caution: don't hide real errors. Absence must genuinely mean "do nothing".

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-31"></a>
### 6.31 MVC

**Model–View–Controller:** separates **data and business logic** (model), **presentation** (view) and **input handling** (controller).
- Flow: user action → controller → updates or queries the model → selects a view → view renders the model.
- Spring MVC: `DispatcherServlet` (front controller) → `@Controller` → `Model` → view (Thymeleaf), or JSON for REST.
- Variants: MVP, MVVM (data binding, used in front-end frameworks).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-32"></a>
### 6.32 How to approach an LLD round

1. **Clarify requirements** (5 min): core features, actors, scale, what's out of scope. Write them down.
2. **Identify entities and relationships:** nouns → classes, verbs → methods. Draw the class diagram.
3. **Define interfaces first** where variation exists (payment, pricing, scheduling) and pick patterns for a **reason**.
4. **Write core code:** models, services, the main flow. Enums for states and types.
5. **Handle concurrency** where it's real (seat booking, parking-spot allocation).
6. **Walk through a scenario**, then discuss extensions (new vehicle type, new payment method) and show the design absorbs them.

**Interviewers look for:** SOLID, sensible patterns (not pattern-stuffing), clean naming, extensibility, edge cases, and thread safety where shared state exists.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-33"></a>
### 6.33 Parking lot

- **Requirements:** multiple floors, spot types (bike/car/truck), entry/exit gates, tickets, fee by duration and vehicle type, availability display.
- **Entities:** `ParkingLot` → `ParkingFloor` → `ParkingSpot` (type, isFree), `Vehicle` (type), `Ticket` (spot, entryTime), `EntryGate`/`ExitGate`, `Payment`.
- **Patterns:** **Strategy** for spot allocation (nearest to gate, lowest floor) and for **pricing** (hourly, flat, weekend). **Factory** for vehicles and spots. Singleton for the lot (or a DI singleton).
- **Concurrency:** two cars must never get the same spot. Allocate spots atomically (synchronised allocation per floor, or `ConcurrentLinkedQueue` of free spots per type).
- **Extensions:** EV charging spots, reservations, multiple lots.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-34"></a>
### 6.34 Tic-tac-toe

- **Entities:** `Game` (board, players, turn, status), `Board` (n×n cells), `Player` (symbol), `Move`.
- **Win check in O(1) per move:** keep row, column, diagonal and anti-diagonal **counters** (+1 for X, −1 for O). A counter reaching ±n is a win. No full board scan needed.
- **Extensions:** an n×n board, more than 2 players, an AI player (Strategy: random vs minimax), undo (Command).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-35"></a>
### 6.35 Elevator system

- **Entities:** `ElevatorController`/`Dispatcher`, `Elevator` (currentFloor, direction, state, request queues), `Request` (external: floor + direction; internal: destination), `Door`, `Display`.
- **States:** IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN, MAINTENANCE (**State pattern**).
- **Scheduling (Strategy):**
  - **SCAN/LOOK** (like a disk head): serve all requests in the current direction, then reverse. Use two **min/max heaps** (up stops, down stops).
  - **Dispatch:** pick the elevator by cost: same direction and on the way > idle and nearest > otherwise.
- **Extensions:** peak-hour modes, VIP or express elevators, capacity limits, emergency stop.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-36"></a>
### 6.36 Car rental

- **Entities:** `Store/Location` → `VehicleInventory` → `Vehicle` (type, status), `User`, `Reservation` (vehicle, from/to, pickup/drop location, status), `Bill`, `Payment`.
- **Flows:** search available vehicles for a date range → reserve → pick up → return → bill (duration + extras + late fees).
- **Key logic:** availability means **no overlapping reservations** for the vehicle (interval overlap: `start < otherEnd && otherStart < end`).
- **Patterns:** Strategy (pricing: hourly/daily/weekly), State (reservation lifecycle), Factory (vehicle types).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-37"></a>
### 6.37 Snake and ladder

- **Entities:** `Board` (size, map of jumps start → end: snakes go down, ladders go up), `Dice` (count, roll), `Player` (position), `Game` (turn queue, winner).
- **Flow:** a `Deque<Player>` for turns → roll → move → apply jump if it lands on a snake or ladder → check the exact finish (≥ 100 or must-be-exact rule) → put the player back in the queue.
- Store snakes and ladders as **one `Map<Integer,Integer>`** of jumps. Validate there are no cycles and no jump on cell 100.
- **Extensions:** multiple dice, crooked dice (Strategy), bonus turn on a 6.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-38"></a>
### 6.38 HashMap implementation

```java
class MyHashMap<K, V> {
    static class Node<K, V> { final K key; V value; Node<K, V> next; }
    Node<K, V>[] table = new Node[16]; int size;

    V get(K key) {
        for (Node<K, V> n = table[index(key)]; n != null; n = n.next) if (Objects.equals(n.key, key)) return n.value;
        return null;
    }
    void put(K key, V value) {
        int i = index(key);
        for (Node<K, V> n = table[i]; n != null; n = n.next) if (Objects.equals(n.key, key)) { n.value = value; return; }
        table[i] = new Node<>(key, value, table[i]);     // prepend
        if (++size > table.length * 0.75) resize();       // double + rehash
    }
    int index(K key) { int h = key == null ? 0 : key.hashCode(); return (h ^ (h >>> 16)) & (table.length - 1); }
}
```
**Talking points:** separate chaining vs open addressing, load factor, resize cost (amortised O(1)), treeification in the JDK, and the equals/hashCode contract ([4.25](04-java.md#s4-25)).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-39"></a>
### 6.39 BookMyShow

- **Entities:** `City` → `Theatre` → `Screen` → `Seat` (type, price), `Movie`, `Show` (movie + screen + time), `ShowSeat` (seat status per show), `Booking`, `Payment`.
- **Flows:** search shows by city, movie and date → view the seat map → **select and lock seats** → pay → confirm, or release on timeout.
- **Concurrency (the key discussion):**
  - In a single JVM: `synchronized` per show, or `ConcurrentHashMap` with putIfAbsent for seat locks.
  - In a real distributed system: a DB row lock or optimistic version on `ShowSeat`, or a **Redis lock with a TTL** (seat hold for 10 minutes). See [HLD ticket booking](07-hld.md#s7-24).
- **Patterns:** Strategy (pricing: weekday/weekend, seat type), Observer (notify waitlist), State (booking lifecycle).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-40"></a>
### 6.40 Vending machine

- **States (State pattern):** Idle → HasMoney → Selection → Dispensing → (ReturnChange) → Idle. Each state class allows only valid operations (inserting a coin while dispensing is rejected).
- **Entities:** `VendingMachine` (current state, inventory, balance), `Inventory` (code → `ItemShelf` with item + quantity), `Coin`/`Note` enums.
- **Change-making:** greedy over coin denominations (works for canonical coin systems). Refund if exact change isn't possible.
- **Edge cases:** sold-out item, insufficient funds, cancel and refund, admin restock mode.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-41"></a>
### 6.41 ATM

- **States:** Idle → HasCard → PinVerified → (select operation) → Withdraw/Balance/Deposit → eject card.
- **Cash dispensing:** **Chain of Responsibility** over note handlers (2000 → 500 → 200 → 100). Each takes as many notes as it can and passes the remainder. Check feasibility **before** dispensing.
- **Entities:** `ATM` (state, cash inventory), `Card`, `Account`, `BankService` (interface: verify PIN, debit), `Transaction`.
- **Concerns:** limit wrong-PIN attempts, debit and dispense atomically (debit first, reverse on dispense failure), daily limits.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-42"></a>
### 6.42 Chess

- **Entities:** `Game` (board, players, turn, status, move history), `Board` (8×8 `Cell`s), abstract `Piece` (color, `canMove(board, from, to)`) with King, Queen, Rook, Bishop, Knight, Pawn subclasses, `Move`, `Player`.
- **Move validation:** each piece's movement rule (**polymorphism**), path not blocked (sliding pieces), not leaving your own king in check.
- **Special rules:** castling, en passant, promotion, check/checkmate/stalemate detection.
- **Patterns:** polymorphism or Strategy per piece movement, Command for moves (undo), Factory for the initial setup.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-43"></a>
### 6.43 File system

- **Composite pattern:** `FileSystemEntry` (name, created, `size()`, `path()`) → `File` (content) and `Directory` (`Map<String, FileSystemEntry>` children).
- **Operations:** `mkdir -p`, create/read/write file, `ls`, delete (recursive), move, size (recursive sum), search by name (DFS/BFS).
- Path resolution: split on `/` and walk from the root. Maps keep children lookup O(1).
- **Extensions:** permissions (Proxy), symbolic links, versioning (Memento).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-44"></a>
### 6.44 Splitwise

- **Entities:** `User`, `Group`, `Expense` (paidBy, amount, `List<Split>`, type), `Split` (user, amount), `BalanceSheet`.
- **Split strategies:** EQUAL, EXACT, PERCENT (**Strategy** + validation: exact splits sum to the total, percents sum to 100).
- **Balances:** `Map<User, Map<User, Amount>>`, or a **net balance per user** (positive = gets money back).
- **Simplify debts:** match the biggest creditor with the biggest debtor greedily using two heaps. This minimises the number of transactions in practice. (Exact minimisation is NP-hard.)
- Use integer paise or `BigDecimal` and handle rounding remainders (give the extra paisa to one person).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-45"></a>
### 6.45 Cricbuzz

- **Entities:** `Match` (teams, format, innings, status), `Team` → `Player`, `Innings` → `Over` → `Ball` (runs, extras, wicket type, bowler, batsman), `Scorecard`.
- **Live updates:** **Observer**: each ball event notifies subscribers (scoreboard, commentary, notifications, stats).
- **Derived stats:** strike rate, economy, run rate and required run rate computed from ball events. Store events and derive stats (event-sourcing-like).
- **Patterns:** Strategy for the match format (T20/ODI/Test rules), State for the match status.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-46"></a>
### 6.46 Inventory / order management

- **Entities:** `Product` (SKU), `Warehouse` → `InventoryItem` (sku, quantity, reserved), `Cart`, `Order` (items, status), `Address`, `Payment`, `Invoice`.
- **Flow:** add to cart → checkout → **reserve stock** → pay → confirm (deduct reserved) or release on failure or timeout.
- **Warehouse selection (Strategy):** nearest to the customer, or the one with full stock.
- **Concurrency:** decrement stock atomically (`UPDATE … SET qty = qty - ? WHERE qty >= ?`, optimistic version, or a lock per SKU).
- **States:** CREATED → RESERVED → PAID → SHIPPED → DELIVERED / CANCELLED.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-47"></a>
### 6.47 Word processor

- **Flyweight:** characters share `CharacterStyle`/glyph objects (font, size, colour) from a **factory cache**. Each character stores only its value plus a reference to the shared style. Millions of characters, a handful of style objects.
- **Composite:** document → pages → paragraphs → lines → characters.
- **Command / Memento:** undo/redo of edits.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-48"></a>
### 6.48 Undo/redo

- **Command approach:** each edit is a `Command` with `execute()` and `undo()`. There are two stacks: `undoStack` and `redoStack`.
  - New command: execute → push to undo → **clear redo**.
  - Undo: pop undo → `undo()` → push to redo. Redo: pop redo → `execute()` → push to undo.
- **Memento approach:** snapshot the state before each change. Simpler, but uses more memory.
- Limit history size with a bounded deque, and merge small commands (typing characters into one "type word" command).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-49"></a>
### 6.49 Apply coupons to a cart

- **Entities:** `Cart` → `CartItem` (product, qty, price), `Coupon` (code, rules, validity, usage limits).
- **Coupon types (Strategy or Decorator):** percent off, flat off, buy-X-get-Y, category-specific, free shipping. **Decorator** lets discounts stack: `new PercentOff(new FlatOff(baseCart))`.
- **Rules (Chain of Responsibility):** validity date → minimum cart value → user eligibility → category match → usage limit.
- **Policies:** stacking rules (one coupon vs many), apply order (percent before flat?), cap the discount, never below zero.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-50"></a>
### 6.50 Payment gateway

- **Entities:** `PaymentRequest` (merchant, amount, currency, method, **idempotencyKey**), `Transaction` (status), `PaymentMethod` interface (Card, UPI, NetBanking, Wallet), `PaymentProcessor`/`BankAdapter`.
- **Patterns:** **Strategy** per payment method, **Adapter** per bank/PSP API, **Factory** to pick the processor, **State** for the transaction (INITIATED → PENDING → SUCCESS/FAILED/REFUNDED), **Observer** for webhooks or notifications.
- **Reliability:** idempotency key on create (a retry returns the same transaction), **retry with backoff** for transient PSP errors, timeouts → PENDING → reconciliation, never retry blind on an unknown outcome.
- See the HLD view: [7.21 Payment system](07-hld.md#s7-21).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s6-51"></a>
### 6.51 Meeting scheduler

- **Entities:** `Room` (capacity, `Calendar`), `Meeting` (interval, attendees, room), `Interval` (start, end), `Scheduler`.
- **Overlap check:** `a.start < b.end && b.start < a.end`. Keep each room's bookings in a **`TreeMap<start, Meeting>`** and check only `floorEntry(start)` and `ceilingEntry(start)`. That's O(log n).
- **Room selection (Strategy):** smallest room that fits, or nearest.
- **Extensions:** recurring meetings, attendee conflicts, notifications (Observer), "minimum rooms needed" (classic heap/sweep problem).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
