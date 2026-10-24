<a id="top"></a>
# 4. Java — Basic to Advanced

[⌂ Main index](00-INDEX.md)

<a id="contents"></a>
## Contents
**Basics**
- [4.1 OOP concepts](#s4-1)
- [4.2 How a Java program runs (JDK, JRE, JVM)](#s4-2)
- [4.3 Why only one public class per file](#s4-3)
- [4.4 Primitive data types](#s4-4)
- [4.5 How float and double work](#s4-5)
- [4.6 Reference (non-primitive) types and String](#s4-6)
- [4.7 Methods](#s4-7)
- [4.8 Constructors](#s4-8)
- [4.9 Memory management and garbage collection](#s4-9)
- [4.10 Classes (concrete, abstract, nested, anonymous)](#s4-10)
- [4.11 Generics](#s4-11)
- [4.12 Enum and POJO](#s4-12)
- [4.13 Singleton and immutable classes](#s4-13)
- [4.14 Operators](#s4-14)
- [4.15 Control flow statements](#s4-15)

**Interfaces, reflection, annotations, exceptions**
- [4.16 Interfaces in depth](#s4-16)
- [4.17 Default, static and private interface methods](#s4-17)
- [4.18 Functional interfaces and lambdas](#s4-18)
- [4.19 Reflection](#s4-19)
- [4.20 Annotations](#s4-20)
- [4.21 Exception handling](#s4-21)

**Collections & streams**
- [4.22 Collections framework overview](#s4-22)
- [4.23 Comparable vs Comparator](#s4-23)
- [4.24 List, Queue and Deque](#s4-24)
- [4.25 HashMap internals](#s4-25)
- [4.26 LinkedHashMap, TreeMap and LRU cache](#s4-26)
- [4.27 Set implementations](#s4-27)
- [4.28 Streams](#s4-28)

**Multithreading**
- [4.29 Process vs thread](#s4-29)
- [4.30 Thread creation and lifecycle](#s4-30)
- [4.31 Thread joining, daemon threads, priority](#s4-31)
- [4.32 synchronized, volatile and the memory model](#s4-32)
- [4.33 Locks and conditions](#s4-33)
- [4.34 Lock-free concurrency (CAS)](#s4-34)
- [4.35 Thread pools and ThreadPoolExecutor](#s4-35)
- [4.36 Future, Callable and CompletableFuture](#s4-36)
- [4.37 Fork-Join pool, single, fixed and cached pools](#s4-37)
- [4.38 ScheduledThreadPoolExecutor](#s4-38)
- [4.39 Virtual threads and ThreadLocal](#s4-39)
- [4.40 Concurrent collections and synchronizers](#s4-40)

**Modern Java & I/O**
- [4.41 Project Lombok](#s4-41)
- [4.42 Records, sealed classes, pattern matching, switch, text blocks](#s4-42)
- [4.43 Sequenced collections (Java 21)](#s4-43)
- [4.44 Optional](#s4-44)
- [4.45 File handling with streams](#s4-45)
- [4.46 NIO and NIO.2](#s4-46)

---

<a id="s4-1"></a>
### 4.1 OOP concepts

| Pillar | Meaning | Java mechanism |
|---|---|---|
| **Encapsulation** | Hide state, expose behaviour | `private` fields + methods, no public setters for invariants |
| **Abstraction** | Show *what*, hide *how* | Interfaces, abstract classes |
| **Inheritance** | Reuse and specialise ("is-a") | `extends` (single class), `implements` (many interfaces) |
| **Polymorphism** | One interface, many forms | **Overloading** (compile-time) and **overriding** (runtime, dynamic dispatch) |

**Relationships:** **association** (uses), **aggregation** (has-a, independent lifetimes), **composition** (has-a, owned lifetime). *Favour composition over inheritance.*

**Interview Q&A:**
- *Overloading vs overriding?* Overloading = same name, different parameters, chosen at compile time. Overriding = same signature in a subclass, chosen at runtime by the object's actual type.
- *Why no multiple class inheritance?* The diamond problem: ambiguous state and behaviour. Interfaces allow multiple *types*, and default-method conflicts must be resolved explicitly.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-2"></a>
### 4.2 How a Java program runs (JDK, JRE, JVM)

- **JDK** = development kit (compiler `javac`, tools) ⊃ **JRE** = runtime libraries ⊃ **JVM** = executes bytecode.
- `javac Hello.java` → **bytecode** (`Hello.class`, platform-neutral) → `java Hello` → the JVM loads, verifies and executes it.
- **Class loading:** Bootstrap → Platform → Application class loaders (**parent delegation**). Then linking (verify, prepare, resolve) → initialisation (static blocks).
- **Execution:** the interpreter runs bytecode, and the **JIT compiler** (C1/C2) compiles hot methods to native code. That's why Java "warms up".
- "Write once, run anywhere": the JVM is platform-specific, the bytecode isn't.

**Interview Q&A:**
- *Why is Java both compiled and interpreted?* Compiled to bytecode ahead of time, then interpreted and JIT-compiled to native code at runtime.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-3"></a>
### 4.3 Why only one public class per file

- A `public` top-level class must live in a file with **the same name** (`OrderService.java`).
- This lets the compiler and class loader **find a class from its name** without scanning every file.
- A file may have several non-public top-level classes (package-private), and any number of nested classes.
- `main` can live in any class. The JVM starts with the class you name on the command line.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-4"></a>
### 4.4 Primitive data types

| Type | Size | Range / notes | Default (fields) |
|---|---|---|---|
| `byte` | 8 bit | −128 … 127 | 0 |
| `short` | 16 bit | −32,768 … 32,767 | 0 |
| `int` | 32 bit | ±2.1 billion | 0 |
| `long` | 64 bit | ±9.2 × 10¹⁸ (suffix `L`) | 0L |
| `float` | 32 bit | ~7 decimal digits (suffix `f`) | 0.0f |
| `double` | 64 bit | ~15–16 decimal digits | 0.0 |
| `char` | 16 bit | Unicode UTF-16 unit | '\u0000' |
| `boolean` | JVM-dependent | `true`/`false` | false |

**Key points:**
- **Local variables have no default.** They must be assigned before use (a compile error otherwise).
- **Widening** (int → long) is automatic. **Narrowing** (long → int) needs a cast and can **overflow silently**.
- **Integer overflow wraps around:** `Integer.MAX_VALUE + 1 == Integer.MIN_VALUE`. Use `Math.addExact` to detect it.
- **Wrapper classes** (`Integer`, `Long`…) with **autoboxing**. `Integer` caches −128…127, so `==` on boxed values is a trap. Use `equals`.

**Interview Q&A:**
- *Why does `Integer a = 127, b = 127; a == b` give true but 128 gives false?* The Integer cache covers −128…127. Outside that range they're different objects.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-5"></a>
### 4.5 How float and double work

- IEEE 754 binary floating point: **sign + exponent + mantissa** (float 1+8+23 bits, double 1+11+52).
- Many decimals **can't be represented exactly** in binary: `0.1 + 0.2 == 0.30000000000000004`.
- Special values: `NaN` (`NaN != NaN`), `+Infinity`, `-0.0`.
- **Money:** use `BigDecimal` (constructed from a **String**, `new BigDecimal("0.10")`, not from a double) with an explicit `RoundingMode`, or store integer minor units (paise or cents in a `long`).
- Compare doubles with a tolerance: `Math.abs(a - b) < 1e-9`.

**Interview Q&A:**
- *Why not `new BigDecimal(0.1)`?* It captures the already-imprecise binary value `0.1000000000000000055…`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-6"></a>
### 4.6 Reference (non-primitive) types and String

- A reference variable holds a **pointer** to an object on the heap. The default for fields is `null`.
- **Java is always pass-by-value.** For objects, the *reference* is copied: the method can mutate the object but can't make the caller's variable point somewhere else.
- **Arrays** are objects (`int[]`), fixed length.
- **String:** immutable, `final` class.
  - **String pool:** literals are interned, so `"a" == "a"` is true, but `new String("a")` creates a new object.
  - Concatenation in loops creates garbage. Use **StringBuilder** (not thread-safe, fast) or StringBuffer (synchronised, legacy).
  - Why immutable: safe as HashMap keys (cached hash), thread-safe, safe for the pool, security (class names, file paths).

**Interview Q&A:**
- *`==` vs `equals` for Strings?* `==` compares references. `equals` compares content. Always use `equals`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-7"></a>
### 4.7 Methods

- **Signature** = name + parameter types (not the return type). Overloading varies parameters.
- **Access modifiers:** `public` > `protected` (package + subclasses) > package-private (default) > `private`.
- **static methods** belong to the class: no `this`, can't be overridden (only **hidden**).
- **final methods** can't be overridden. **abstract methods** have no body.
- **Varargs:** `void log(String fmt, Object... args)`. Must be the last parameter.
- **Overriding rules:** same signature, return type same or **covariant**, access can't be narrowed, can't throw broader checked exceptions.
- **Recursion:** each call adds a stack frame. Too deep gives `StackOverflowError`.

**Interview Q&A:**
- *Can you override a static method?* No. Declaring the same static method in a subclass *hides* it, and the call is resolved by the reference type at compile time.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-8"></a>
### 4.8 Constructors

- Same name as the class, no return type. If you write none, the compiler adds a **default no-arg constructor**. Write any constructor and that default disappears.
- **Constructor chaining:** `this(...)` calls another constructor in the same class. `super(...)` calls the parent's. It must be the **first statement**, and the parent's constructor always runs first.
- **Initialisation order:** static blocks (once, at class load) → instance initialisers and field initialisers → constructor body.
- **Private constructor:** used for singletons, utility classes and factories.
- Copy constructors as a safer alternative to `clone()`.

**Interview Q&A:**
- *Can a constructor be final, static or abstract?* No. Constructors aren't inherited, so those modifiers don't apply.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-9"></a>
### 4.9 Memory management and garbage collection

**JVM memory areas:**
- **Heap:** all objects. Shared by threads. Generational: **Young** (Eden + 2 Survivor spaces) and **Old**.
- **Stack:** one per thread. Frames hold local primitives and references. Freed when the method returns.
- **Metaspace:** class metadata (native memory, replaced PermGen in Java 8).
- Code cache (JIT output), PC registers, native method stacks.

**Garbage collection:**
- An object becomes garbage when it's **unreachable from GC roots** (thread stacks, static fields, JNI refs). Cycles are collected too.
- **Minor GC:** Eden → Survivor → promoted to Old after surviving N cycles. **Major/Full GC:** Old generation.
- **Collectors:** Serial, Parallel (throughput), **G1** (default since 9, region-based, pause goals), **ZGC/Shenandoah** (sub-millisecond pauses on huge heaps).
- **Reference types:** strong, **soft** (cleared under memory pressure, for caches), **weak** (cleared at the next GC, as in `WeakHashMap`), phantom (cleanup).

**Memory leaks in Java:** static collections that only grow, unremoved listeners, `ThreadLocal`s in thread pools, caches without eviction, unclosed resources.

**Interview Q&A:**
- *`OutOfMemoryError: Java heap space` vs `Metaspace`?* Too many live objects vs too many loaded classes (often class-loader leaks on redeploy).
- *How do you investigate a leak?* Heap dump (`jcmd <pid> GC.heap_dump` or `-XX:+HeapDumpOnOutOfMemoryError`), then Eclipse MAT for the dominator tree.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-10"></a>
### 4.10 Classes (concrete, abstract, nested, anonymous)

| Kind | Notes |
|---|---|
| Concrete class | Can be instantiated |
| **Abstract class** | Can't be instantiated. Can have state, constructors and concrete + abstract methods |
| **Static nested class** | No reference to the outer instance. Good for builders and helpers |
| **Inner (non-static) class** | Holds an implicit reference to the outer instance (can leak memory) |
| **Local class** | Declared inside a method |
| **Anonymous class** | One-off implementation inline, largely replaced by lambdas for functional interfaces |
| `final` class | Can't be extended (`String`) |

**Abstract class vs interface:**
| | Abstract class | Interface |
|---|---|---|
| State | Instance fields allowed | Only constants |
| Constructors | Yes | No |
| Inheritance | Single | Multiple |
| Use | Shared base implementation ("is-a") | Capability or contract ("can-do") |

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-11"></a>
### 4.11 Generics

**What:** Type parameters give compile-time type safety and remove casts: `List<String>` instead of a raw `List`.

```java
class Box<T> { private T value; T get() { return value; } }
static <T extends Comparable<T>> T max(List<T> items) { ... }   // bounded type parameter
```

**Key points:**
- **Type erasure:** generic type info is removed at runtime (`List<String>` and `List<Integer>` are the same class). So no `new T()`, no `instanceof List<String>`, and no primitive type arguments.
- **Bounded types:** `<T extends Number>`.
- **Wildcards:** `List<?>` (unknown type), `<? extends T>` (read/produce), `<? super T>` (write/consume). **PECS: Producer Extends, Consumer Super.**
- Generics are **invariant**: `List<Integer>` is **not** a `List<Number>`. Arrays are covariant (which can cause `ArrayStoreException`).

**Interview Q&A:**
- *Why can't you create `new T[]`?* Erasure. At runtime the JVM doesn't know what T is.
- *Explain PECS with an example.* `Collections.copy(List<? super T> dest, List<? extends T> src)`: read from the producer, write to the consumer.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-12"></a>
### 4.12 Enum and POJO

**Enum:** a fixed set of constants. It's a full class with fields, constructors, methods, and per-constant behaviour.
```java
enum Status {
    PLACED { boolean canCancel() { return true; } },
    SHIPPED { boolean canCancel() { return false; } };
    abstract boolean canCancel();
}
```
- Type-safe, usable in `switch`, `values()`/`valueOf()`, `EnumMap`/`EnumSet` (very fast, bit-vector based).
- Enums are serialisation- and reflection-safe singletons.

**POJO** (Plain Old Java Object): a simple class with no framework requirements, just fields + getters/setters + equals/hashCode/toString.
**JavaBean:** a POJO with a no-arg constructor, private fields and getter/setter naming conventions (used by frameworks).
Today, prefer **records** for immutable data carriers.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-13"></a>
### 4.13 Singleton and immutable classes

**Singleton (one instance per JVM/class loader):**
```java
// 1) Enum — simplest, serialization- and reflection-safe
enum Config { INSTANCE; }

// 2) Holder idiom — lazy, thread-safe without locking
class Registry {
    private Registry() {}
    private static class Holder { static final Registry I = new Registry(); }
    static Registry get() { return Holder.I; }
}

// 3) Double-checked locking — needs volatile
private static volatile Cache inst;
static Cache get() { if (inst == null) { synchronized (Cache.class) { if (inst == null) inst = new Cache(); } } return inst; }
```
**Without `volatile`**, another thread may see a half-constructed object because of instruction reordering.
Ways to break a singleton: reflection, serialisation (fix with `readResolve`), cloning. The enum approach resists all three.

**Immutable class:** `final` class, `private final` fields, no setters, all fields set in the constructor, **defensive copies** of mutable inputs and outputs (`List.copyOf`). Records give you most of this.

**Interview Q&A:**
- *Why make objects immutable?* Thread-safe without locks, safe as map keys, easy to reason about and cache.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-14"></a>
### 4.14 Operators

| Group | Operators | Notes |
|---|---|---|
| Arithmetic | `+ - * / %` | Integer `/` truncates. `%` keeps the dividend's sign (`-7 % 3 == -1`) |
| Unary | `++ -- + - !` | `i++` returns the old value, `++i` the new one |
| Relational | `== != < > <= >=` | `==` on objects compares references |
| Logical | `&& || !` | **Short-circuit**: right side skipped if not needed |
| Bitwise | `& | ^ ~` | `&`/`|` on booleans **don't** short-circuit |
| Shift | `<< >> >>>` | `>>` keeps the sign. `>>>` fills with zeros |
| Assignment | `= += -= …` | Compound forms **cast implicitly**: `byte b += 1` compiles |
| Ternary | `c ? a : b` | |
| `instanceof` | Type check | Pattern matching since Java 16 |

**Precedence (high → low, simplified):** postfix → unary → `* / %` → `+ -` → shifts → relational → equality → `&` → `^` → `|` → `&&` → `||` → ternary → assignment. Use parentheses for clarity.

**Bit tricks used in DSA:** `n & (n-1)` clears the lowest set bit (a power-of-two check), `x ^ x == 0` (find the unique number), `1 << k`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-15"></a>
### 4.15 Control flow statements

- **if / else if / else.**
- **switch:** classic form with `case … break` (fall-through if you forget `break`). Works on int types, `char`, `String`, enums.
- **Switch expressions** (Java 14): `case A, B -> value;` with no fall-through, and `yield` in blocks. Must be exhaustive.
- **Loops:** `for`, enhanced `for-each` (Iterable/arrays, can't modify the collection structure → `ConcurrentModificationException`), `while`, `do-while` (runs at least once).
- **Jumps:** `break`, `continue`, **labelled break** to exit nested loops (`outer: for … break outer;`), `return`.

```java
String type = switch (day) {
    case SATURDAY, SUNDAY -> "weekend";
    default -> { log(day); yield "weekday"; }
};
```

**Interview Q&A:**
- *How do you remove items while iterating?* `iterator.remove()` or `list.removeIf(...)`. Removing inside for-each throws `ConcurrentModificationException`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-16"></a>
### 4.16 Interfaces in depth

- A contract: abstract methods (implicitly `public abstract`) and constants (implicitly `public static final`).
- A class can implement **many** interfaces, and an interface can **extend** many interfaces.
- Uses: polymorphism (`List list = new ArrayList<>()`), decoupling (code to the interface, which is DI-friendly), callbacks, **marker interfaces** (`Serializable`, `Cloneable`: no methods, just a type tag).
- **Nested interfaces** and **functional interfaces** (exactly one abstract method).

**Interview Q&A:**
- *Why "program to an interface"?* Callers depend on the abstraction, so implementations can be swapped (mocks in tests, a new provider) without changing callers.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-17"></a>
### 4.17 Default, static and private interface methods

- **default** (Java 8): a method with a body. Lets interfaces evolve (adding `List.sort`) without breaking implementers.
- **static** (Java 8): utility methods on the interface (`Comparator.comparing`). Not inherited by implementing classes.
- **private** (Java 9): shared helper code for default methods.

**Diamond rule:** if two interfaces give the same default method, the class **must override** it and can pick one: `A.super.hello()`.
**Class wins:** a method in the superclass beats an interface default.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-18"></a>
### 4.18 Functional interfaces and lambdas

**Functional interface:** exactly **one abstract method** (`@FunctionalInterface` enforces it). A lambda is a concise implementation of one.

| Interface | Signature | Example |
|---|---|---|
| `Predicate<T>` | `T → boolean` | `s -> s.isEmpty()` |
| `Function<T,R>` | `T → R` | `String::length` |
| `Consumer<T>` | `T → void` | `System.out::println` |
| `Supplier<T>` | `() → T` | `ArrayList::new` |
| `UnaryOperator<T>`, `BinaryOperator<T>` | `T → T`, `(T,T) → T` | `Integer::sum` |
| `BiFunction<T,U,R>` | `(T,U) → R` | |

**Method references:** static (`Integer::parseInt`), instance of a particular object (`list::add`), instance of an arbitrary object (`String::toUpperCase`), constructor (`Order::new`).

**Rules:** lambdas can capture only **effectively final** local variables. `this` inside a lambda refers to the enclosing instance (unlike anonymous classes).
**Composition:** `predicate.and/or/negate`, `function.andThen/compose`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-19"></a>
### 4.19 Reflection

**What:** Inspect and modify classes, methods and fields **at runtime**.
```java
Class<?> c = Class.forName("com.shop.Order");
Object o = c.getDeclaredConstructor().newInstance();
Field f = c.getDeclaredField("status"); f.setAccessible(true); f.set(o, "PAID");
Method m = c.getMethod("total"); Object result = m.invoke(o);
```

**Used by:** Spring (DI, scanning annotations), Hibernate (entity mapping), Jackson (JSON), JUnit (finding tests).

**Costs:** slower than direct calls, breaks encapsulation, no compile-time checking, restricted by modules (Java 9+: `--add-opens`).
**Alternatives:** `MethodHandle`s, code generation at compile time (annotation processors, as in Lombok and MapStruct).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-20"></a>
### 4.20 Annotations

**What:** Metadata attached to code, read by the compiler, tools or frameworks.

**Built-in:** `@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface`, `@SafeVarargs`.

**Meta-annotations** (annotations on annotations):
- `@Retention`: **SOURCE** (discarded by the compiler, e.g. Lombok), **CLASS** (in bytecode, not at runtime), **RUNTIME** (readable via reflection, e.g. Spring).
- `@Target`: where it can go (TYPE, METHOD, FIELD, PARAMETER…).
- `@Inherited`, `@Documented`, `@Repeatable`.

**Custom annotation:**
```java
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.METHOD)
public @interface Audited { String action(); }
// read: method.getAnnotation(Audited.class).action()  — or with an AOP pointcut @annotation(Audited)
```
Annotation elements may only be primitives, String, Class, enums, other annotations, or arrays of these.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-21"></a>
### 4.21 Exception handling

**Hierarchy:** `Throwable` → `Error` (OutOfMemoryError, StackOverflowError: don't catch) and `Exception` → checked exceptions (`IOException`, must be handled or declared) and **unchecked** `RuntimeException` (`NullPointerException`, `IllegalArgumentException`).

**Constructs:**
- `try / catch / finally` (`finally` always runs, except on `System.exit` or a JVM crash).
- **Multi-catch:** `catch (IOException | SQLException e)`.
- **try-with-resources:** auto-closes `AutoCloseable`s in reverse order. Exceptions from `close()` are attached as **suppressed**.
- `throw` (raise one) vs `throws` (declare in the signature).
- **Custom exceptions:** extend `RuntimeException` for business errors and keep the **cause** (`new OrderException("msg", e)`).

**Best practices:**
- Never swallow exceptions (`catch (Exception e) {}`). Log **or** rethrow, not both (that duplicates logs).
- Catch the most specific type. Don't use exceptions for normal control flow.
- `return` inside `finally` overrides the try's return or exception. Avoid it.

**Interview Q&A:**
- *Checked vs unchecked: when to use which?* Checked for recoverable conditions the caller must handle (file missing). Unchecked for programming errors and most business errors in modern frameworks.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-22"></a>
### 4.22 Collections framework overview

```
Iterable → Collection → List  (ArrayList, LinkedList, Vector/Stack)
                      → Set   (HashSet, LinkedHashSet, TreeSet)
                      → Queue (PriorityQueue, ArrayDeque, LinkedList) → Deque
Map (separate)        → HashMap, LinkedHashMap, TreeMap, Hashtable, ConcurrentHashMap, EnumMap, WeakHashMap
```

**Choosing:**
| Need | Use |
|---|---|
| Indexed list, fast random access | `ArrayList` |
| Unique items, fast lookup | `HashSet` |
| Sorted unique items / range queries | `TreeSet` / `TreeMap` |
| Key–value lookup | `HashMap` |
| Insertion order preserved | `LinkedHashMap` / `LinkedHashSet` |
| Stack or queue | `ArrayDeque` |
| Min/max-heap / top-K | `PriorityQueue` |
| Thread-safe map | `ConcurrentHashMap` |

**Utilities:** `Collections.sort`, `unmodifiableList`, `synchronizedList`. **Immutable factories:** `List.of`, `Map.of` (Java 9, null-hostile).
**Fail-fast iterators** throw `ConcurrentModificationException` on structural modification during iteration. **Fail-safe** ones (concurrent collections) iterate over a snapshot or weakly consistent view.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-23"></a>
### 4.23 Comparable vs Comparator

| | Comparable | Comparator |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T a, T b)` |
| Where | Inside the class (**natural order**) | Outside, as many orderings as you like |
| Example | `String`, `Integer` | `Comparator.comparing(Emp::dept).thenComparing(Emp::salary, reverseOrder())` |

**Contract:** negative / zero / positive. Should be **consistent with equals**, or `TreeSet`/`TreeMap` will treat "equal by comparison" items as duplicates.
**Pitfall:** `return a.x - b.x;` can **overflow** for large values, so use `Integer.compare(a.x, b.x)`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-24"></a>
### 4.24 List, Queue and Deque

| | ArrayList | LinkedList |
|---|---|---|
| Structure | Resizable array (grows ~1.5×) | Doubly linked nodes |
| get(i) | **O(1)** | O(n) |
| add at end | Amortised O(1) | O(1) |
| insert/remove in middle | O(n) shift | O(1) once positioned (finding it is O(n)) |
| Memory / cache | Compact, CPU-cache friendly | Node overhead, poor locality |
→ In practice **ArrayList almost always wins**.

**Queue/Deque:**
- `ArrayDeque`: the best general **stack and queue** (faster than `Stack` and `LinkedList`, no nulls).
- `PriorityQueue`: a binary heap. `offer`/`poll` O(log n), `peek` O(1). Iteration order is **not** sorted.
- Methods that **throw** (`add/remove/element`) vs ones that **return special values** (`offer/poll/peek`).
- `Vector` and `Stack` are legacy synchronised classes. Avoid them.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-25"></a>
### 4.25 HashMap internals

1. `hash = key.hashCode() ^ (h >>> 16)` (spreads the high bits) → `index = hash & (n - 1)`, where capacity n is a power of 2.
2. Each bucket holds a **linked list** of nodes. When a bucket reaches **8** nodes (and capacity ≥ 64) it becomes a **red-black tree**: O(log n) instead of O(n). Below 6 it goes back to a list.
3. **Load factor 0.75:** when `size > capacity × 0.75`, the table **resizes** (doubles) and entries are redistributed (each stays at index *i* or moves to *i + oldCap*).
4. One `null` key is allowed (bucket 0).

**Contract:** if `a.equals(b)` then `a.hashCode() == b.hashCode()`. Breaking it loses entries. **Mutable keys** whose hash changes after insertion become unfindable.

**Complexity:** average O(1), worst case O(log n) with treeification.

**Interview Q&A:**
- *Why is capacity a power of two?* `hash & (n-1)` is a cheap modulo, and resizing splits buckets neatly.
- *HashMap vs Hashtable vs ConcurrentHashMap?* HashMap is not thread-safe. Hashtable locks the whole table (legacy). ConcurrentHashMap uses CAS + per-bin locks (high concurrency) and allows no null keys or values.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-26"></a>
### 4.26 LinkedHashMap, TreeMap and LRU cache

**LinkedHashMap:** HashMap plus a doubly linked list through the entries, giving predictable iteration order: **insertion order** (default) or **access order** (`accessOrder = true`).

**LRU cache in a few lines:**
```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LruCache(int capacity) { super(16, 0.75f, true); this.capacity = capacity; }   // access-order
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > capacity; }
}
```
(Not thread-safe. Wrap it, or use Caffeine in production.)

**TreeMap:** red-black tree, keys **sorted**. O(log n) get/put. Navigation: `firstKey`, `floorKey`, `ceilingKey`, `headMap`, `tailMap`, `subMap` (great for range queries and time-series lookups).

**Interview Q&A:**
- *Build an LRU cache without LinkedHashMap.* HashMap<key, node> + a doubly linked list. On get, move the node to the head. On put over capacity, evict the tail. Both O(1).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-27"></a>
### 4.27 Set implementations

| Set | Backed by | Order | Ops |
|---|---|---|---|
| `HashSet` | HashMap (value = dummy object) | None | O(1) |
| `LinkedHashSet` | LinkedHashMap | Insertion | O(1) |
| `TreeSet` | TreeMap | Sorted | O(log n), plus `floor`/`ceiling`/ranges |
| `EnumSet` | Bit vector | Enum order | Very fast |
| `CopyOnWriteArraySet`, `ConcurrentSkipListSet` | — | — | Thread-safe |

Uniqueness relies on `equals`/`hashCode` (hash sets) or `compareTo`/Comparator (TreeSet).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-28"></a>
### 4.28 Streams

**Pipeline:** source → **intermediate ops** (lazy: `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `peek`) → **terminal op** (triggers execution: `collect`, `forEach`, `reduce`, `count`, `anyMatch`, `findFirst`).

```java
Map<String, Double> avgByDept = employees.stream()
    .filter(e -> e.salary() > 50_000)
    .collect(Collectors.groupingBy(Employee::dept, Collectors.averagingDouble(Employee::salary)));
```

**Key collectors:** `toList()` (Java 16, unmodifiable), `toMap` (**throws on duplicate keys** unless you pass a merge function), `groupingBy`, `partitioningBy`, `joining`, `counting`, `summingInt`.

**Rules:**
- A stream can be consumed **once**. Laziness enables short-circuiting (`findFirst` stops early).
- No side effects inside `map`/`filter`. Keep lambdas pure.
- **Primitive streams** (`IntStream`, `mapToInt`) avoid boxing costs.
- **Parallel streams** use the common ForkJoinPool. They're only worth it for big, CPU-bound, splittable work with no shared state. Never use them for blocking I/O.

**Interview Q&A:**
- *map vs flatMap?* map: one input → one output. flatMap: one input → a stream of outputs, flattened (`List<List<T>>` → `Stream<T>`).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-29"></a>
### 4.29 Process vs thread

| | Process | Thread |
|---|---|---|
| Memory | Own address space | **Shares** the process heap; own stack and PC |
| Creation / switch cost | High | Lower |
| Communication | IPC (pipes, sockets) | Shared memory, so it needs synchronisation |
| Failure isolation | Strong | A crash or OOM can take down the whole process |

**Concurrency vs parallelism:** concurrency = dealing with many tasks at once (interleaved). Parallelism = doing many at the same instant (multiple cores).
**Why threads:** responsiveness, using all cores, overlapping I/O waits.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-30"></a>
### 4.30 Thread creation and lifecycle

**Creating threads:**
1. Extend `Thread` and override `run()` (inflexible, uses up your single inheritance).
2. Implement **`Runnable`** and pass it to `new Thread(r)`, or a lambda.
3. **`Callable<V>`** returns a value or throws, used with an `ExecutorService` → `Future<V>`.
4. In real apps, use **executors**, not raw `new Thread()`.

`start()` creates a new thread that calls `run()`. Calling `run()` directly just runs it on the **current** thread.

**States:** NEW → RUNNABLE → (BLOCKED waiting for a monitor | WAITING `wait/join/park` | TIMED_WAITING `sleep/wait(t)`) → TERMINATED.

**wait/notify:** must hold the monitor (`synchronized`). Always call `wait()` in a **while loop** (spurious wakeups). `sleep()` keeps locks. `wait()` releases the monitor.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-31"></a>
### 4.31 Thread joining, daemon threads, priority

- **`join()`:** the current thread waits until the other thread finishes (`join(ms)` with a timeout).
- **Daemon threads:** background helpers (GC, timers). **The JVM exits when only daemon threads remain**, without waiting for them. Set with `setDaemon(true)` **before** `start()`. Don't use them for work that must finish (writing files): it can be cut off mid-way.
- **Priority** (`MIN_PRIORITY` 1 … `MAX_PRIORITY` 10, default 5) is only a **hint** to the OS scheduler. Behaviour is platform-dependent, so never rely on it for correctness.
- **Interruption:** `t.interrupt()` sets a flag. Blocking calls throw `InterruptedException`. Handle it by stopping or restoring the flag (`Thread.currentThread().interrupt()`), never by swallowing it.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-32"></a>
### 4.32 synchronized, volatile and the memory model

**Three problems:** **atomicity** (`count++` is read-modify-write), **visibility** (one thread's writes may be invisible to another through CPU caches), **ordering** (compiler and CPU reordering).

- **`synchronized`** (method or block): mutual exclusion on a monitor **plus** visibility (happens-before on unlock → lock). It's **re-entrant**.
- **`volatile`:** **visibility and ordering only**, not atomicity. Good for flags (`volatile boolean running`) and safe publication. `volatile int count; count++` is still a race.
- **Happens-before rules:** unlock → subsequent lock, volatile write → subsequent read, `Thread.start()`, `join()`, and the final-field semantics of constructors.

**Deadlock:** threads wait on each other's locks in a cycle. Prevent it with a **global lock ordering**, `tryLock` with a timeout, or fewer nested locks.
**Livelock:** threads keep reacting to each other without progress. **Starvation:** a thread never gets the resource.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-33"></a>
### 4.33 Locks and conditions

**`ReentrantLock`** (more than `synchronized`):
```java
private final ReentrantLock lock = new ReentrantLock();
void transfer() {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {          // timed: avoids deadlock
        try { ... } finally { lock.unlock(); }         // ALWAYS unlock in finally
    }
}
```
- `tryLock`, `lockInterruptibly`, a **fairness** option, several **Condition** objects.

**Condition** (replaces wait/notify, one per wait reason):
```java
Condition notFull = lock.newCondition(), notEmpty = lock.newCondition();
// producer: while (full) notFull.await(); add(); notEmpty.signal();
```

**`ReentrantReadWriteLock`:** many readers OR one writer. Good for read-heavy shared state.
**`StampedLock`:** adds **optimistic reads** (no locking, validate the stamp afterwards). Not re-entrant.
**Semaphores** (permits) are in [4.40](#s4-40).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-34"></a>
### 4.34 Lock-free concurrency (CAS)

**CAS (compare-and-swap):** a single CPU instruction that says "set to *new* only if the current value is still *expected*". Retry on failure. No blocking and no context switches.

```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();                         // CAS loop inside
counter.updateAndGet(x -> Math.min(x + 1, MAX));   // custom atomic update
```

- **Atomic classes:** `AtomicInteger/Long/Boolean/Reference`, `AtomicIntegerArray`.
- **`LongAdder`:** striped counters, much faster than `AtomicLong` under heavy contention (sums on read).
- **ABA problem:** the value changes A → B → A, so CAS wrongly succeeds. Fix with `AtomicStampedReference` (value + version).
- Under very high contention, CAS spin loops waste CPU, and a lock may be better.

**Interview Q&A:**
- *Why is `volatile int count; count++` unsafe but `AtomicInteger.incrementAndGet()` safe?* `++` is three steps. The atomic version uses CAS to make it one indivisible update.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-35"></a>
### 4.35 Thread pools and ThreadPoolExecutor

**Why pools:** reusing threads avoids creation cost and **bounds** concurrency and memory.

```java
new ThreadPoolExecutor(
    4,                      // corePoolSize
    16,                     // maximumPoolSize
    60, TimeUnit.SECONDS,   // keepAlive for threads above core
    new ArrayBlockingQueue<>(500),            // bounded queue
    Executors.defaultThreadFactory(),         // name your threads in real apps
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure when saturated
);
```

**Task submission order:** core threads → **queue** → extra threads up to max → **rejection**.
**Rejection policies:** `AbortPolicy` (throw, the default), `CallerRunsPolicy` (caller runs it, which slows the producer), `DiscardPolicy`, `DiscardOldestPolicy`.

**Sizing:** CPU-bound ≈ number of cores. I/O-bound ≈ cores × (1 + wait/compute). Measure, don't guess.
**Shutdown:** `shutdown()` (finish queued tasks) → `awaitTermination` → `shutdownNow()` (interrupt).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-36"></a>
### 4.36 Future, Callable and CompletableFuture

- **`Callable<V>`** returns a value and can throw checked exceptions. `executor.submit(callable)` gives a **`Future<V>`**.
- **`Future`:** `get()` **blocks**, `get(timeout)`, `cancel`, `isDone`. You can't chain or combine futures.

**`CompletableFuture`:** non-blocking composition.
```java
CompletableFuture<Price>  price = supplyAsync(() -> pricing.get(sku), ioPool);
CompletableFuture<Stock>  stock = supplyAsync(() -> inventory.get(sku), ioPool);
CompletableFuture<Offer>  offer = price.thenCombine(stock, Offer::new)
    .orTimeout(2, SECONDS)
    .exceptionally(ex -> Offer.unavailable(sku));
```
| Method | Purpose |
|---|---|
| `thenApply` | Transform the result (map) |
| `thenCompose` | Chain another async call (flatMap) |
| `thenCombine` | Join two independent futures |
| `allOf` / `anyOf` | Wait for all / the first |
| `exceptionally` / `handle` / `whenComplete` | Error handling |
| `orTimeout` / `completeOnTimeout` | Timeouts (Java 9) |

**Pitfall:** without an executor argument, async methods run on the **common ForkJoinPool**. Blocking I/O there starves the whole JVM. Pass your own pool.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-37"></a>
### 4.37 Fork-Join pool, single, fixed and cached pools

**Executors factory methods:**
| Factory | Behaviour | Risk |
|---|---|---|
| `newFixedThreadPool(n)` | n threads, **unbounded** queue | Queue grows until OOM under overload |
| `newSingleThreadExecutor()` | 1 thread, tasks run in order | Unbounded queue |
| `newCachedThreadPool()` | Unlimited threads, reused when idle | **Thread explosion** under load |
| `newScheduledThreadPool(n)` | Delayed and periodic tasks | |
| `newWorkStealingPool()` | ForkJoinPool | |
| `newVirtualThreadPerTaskExecutor()` (21) | A virtual thread per task | |
→ In production, prefer an explicitly configured `ThreadPoolExecutor` with a **bounded** queue.

**ForkJoinPool:** for **divide-and-conquer**. Split a task into subtasks (`fork`), compute, combine (`join`). **Work stealing:** idle threads take tasks from busy threads' queues.
```java
class SumTask extends RecursiveTask<Long> {
    protected Long compute() {
        if (hi - lo <= THRESHOLD) return sequentialSum();
        SumTask left = new SumTask(lo, mid), right = new SumTask(mid, hi);
        left.fork(); return right.compute() + left.join();
    }
}
```
`RecursiveTask<V>` returns a value, `RecursiveAction` doesn't. Parallel streams and `CompletableFuture` default to the **common pool**.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-38"></a>
### 4.38 ScheduledThreadPoolExecutor

```java
ScheduledExecutorService s = Executors.newScheduledThreadPool(2);
s.schedule(task, 5, SECONDS);                          // once, after a delay
s.scheduleAtFixedRate(task, 0, 10, SECONDS);           // start-to-start every 10 s
s.scheduleWithFixedDelay(task, 0, 10, SECONDS);        // 10 s after previous END
```

- **Fixed rate vs fixed delay:** with fixed rate, if a run takes longer than the period, the **next run waits until the previous one has finished** (runs never overlap, but they can start late and then run back to back). Fixed delay always leaves the full delay **after** each run ends.
- **If a periodic task throws, all its future runs are silently cancelled.** Wrap the body in try/catch.
- Better than `java.util.Timer`, which uses one thread, and one exception kills it.
- In Spring: `@Scheduled(fixedRate/fixedDelay/cron)` + `@EnableScheduling`. In a cluster, see the [distributed scheduler](01-microservices.md#s1-16).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-39"></a>
### 4.39 Virtual threads and ThreadLocal

**Virtual threads (Java 21):**
- Lightweight threads managed by the JVM and **mounted** on a few carrier (platform) threads. Millions are possible.
- When a virtual thread **blocks on I/O**, it unmounts and the carrier runs another one. Blocking becomes cheap.
- Lets you write simple **thread-per-request blocking code** that scales like async code.
```java
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
    ex.submit(() -> callRemoteService());
}
// Spring Boot 3.2+: spring.threads.virtual.enabled=true
```
- **Don't pool them** (they're cheap). Limit access to scarce resources with a **Semaphore** instead.
- Best for **I/O-bound** work. They don't speed up CPU-bound work.
- **Pinning:** on Java 21–23, blocking inside `synchronized` pins the carrier thread. Java 24 removed most pinning (JEP 491). Native calls still pin.

**ThreadLocal:**
- A per-thread variable copy (user context, transaction, `SimpleDateFormat`).
- **Leak risk in pools:** threads live forever, so **always `remove()` in a finally block**.
- With millions of virtual threads, ThreadLocals multiply memory. Prefer **ScopedValue** (immutable, bounded lifetime; preview in 21, final in Java 25).
- `InheritableThreadLocal` copies values to child threads, but not across pool reuse.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-40"></a>
### 4.40 Concurrent collections and synchronizers

**Collections:**
| Class | Use |
|---|---|
| `ConcurrentHashMap` | Thread-safe map. `compute`/`merge` are atomic per key |
| `CopyOnWriteArrayList` | Read-mostly lists (listeners): writes copy the array |
| `ConcurrentLinkedQueue` | Lock-free unbounded queue |
| `BlockingQueue` (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | **Producer–consumer**: `put` blocks when full, `take` blocks when empty |
| `PriorityBlockingQueue`, `DelayQueue` | Priority / delayed tasks |
| `ConcurrentSkipListMap` | Sorted concurrent map |

**Synchronizers:**
| Class | Purpose |
|---|---|
| `CountDownLatch` | Wait until N events happen (one-shot) |
| `CyclicBarrier` | N threads wait for each other, reusable |
| `Semaphore` | Limit concurrent access to N permits |
| `Phaser` | Flexible multi-phase barrier |
| `Exchanger` | Two threads swap objects |

**Pitfall:** `if (!map.containsKey(k)) map.put(k, v)` is a race even on ConcurrentHashMap. Use `putIfAbsent` / `computeIfAbsent`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-41"></a>
### 4.41 Project Lombok

**What:** An annotation processor that generates boilerplate **at compile time**.

| Annotation | Generates |
|---|---|
| `@Getter` / `@Setter` | Accessors |
| `@ToString`, `@EqualsAndHashCode` | Those methods |
| `@NoArgsConstructor`, `@AllArgsConstructor`, `@RequiredArgsConstructor` | Constructors (`Required` = final fields, perfect for constructor DI) |
| `@Data` | Getter + setter + toString + equals/hashCode + required-args constructor |
| `@Value` | Immutable version of `@Data` |
| `@Builder` | Builder pattern |
| `@Slf4j` | `private static final Logger log` |

**Pitfalls:**
- **`@Data` on JPA entities:** generated `equals/hashCode/toString` touch lazy relations, causing extra queries, `LazyInitializationException` and infinite recursion on bidirectional links. On entities, use `@Getter/@Setter` and write `equals` on the ID yourself.
- It hides code from reviewers and needs IDE plugin support.
- Records now cover many `@Value` use cases without a library.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-42"></a>
### 4.42 Records, sealed classes, pattern matching, switch, text blocks

| Feature | Version | What it gives you |
|---|---|---|
| **Switch expressions** | 14 | `case A, B -> value`, no fall-through, `yield` |
| **Text blocks** | 15 | `"""` multi-line strings (JSON, SQL) without escape clutter |
| **Records** | 16 | `record Point(int x, int y) {}`: immutable data class with constructor, accessors, equals/hashCode/toString. Compact constructor for validation |
| **Pattern matching for instanceof** | 16 | `if (o instanceof Order ord) ord.total();` with no cast |
| **Sealed classes** | 17 | `sealed interface Shape permits Circle, Square {}`: a closed hierarchy the compiler knows |
| **Pattern matching for switch** | 21 | Switch on types with guards: `case Circle c when c.r() > 10 ->` |
| **Record patterns** | 21 | Destructure: `case Point(int x, int y) ->` |

**Why sealed + records + pattern switch matter together:** they model **algebraic data types**. The compiler checks the switch is **exhaustive**, so adding a new subtype breaks the build wherever it isn't handled.

```java
sealed interface Payment permits Card, Upi, Wallet {}
double fee(Payment p) {
    return switch (p) {
        case Card c   -> c.amount() * 0.02;
        case Upi u    -> 0;
        case Wallet w -> w.amount() * 0.01;
    };   // no default needed — exhaustive
}
```

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-43"></a>
### 4.43 Sequenced collections (Java 21)

**Problem:** no common API for "first" and "last" across List, Deque, LinkedHashSet and LinkedHashMap.
**Solution:** new interfaces `SequencedCollection`, `SequencedSet`, `SequencedMap`:
- `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `removeLast()`, **`reversed()`** (a view, not a copy).
- Maps: `firstEntry()`, `lastEntry()`, `pollFirstEntry()`, `putFirst()`, `sequencedKeySet()`.

```java
list.getLast();                 // instead of list.get(list.size() - 1)
linkedHashMap.firstEntry();     // previously needed an iterator
```

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-44"></a>
### 4.44 Optional

**What:** A container that may or may not hold a value. It makes "no result" explicit in **return types**.

```java
Optional<User> u = repo.findByEmail(email);
String name = u.map(User::name).orElse("guest");
User user = u.orElseThrow(() -> new UserNotFound(email));
u.ifPresentOrElse(this::greet, this::signupPrompt);   // Java 9
```

**Rules:**
- ✅ Use it as a **method return type** for "maybe absent".
- ❌ Not for fields, method parameters or collections (return an empty collection instead).
- ❌ Don't call `get()` without checking. Use `orElse`/`orElseGet`/`orElseThrow`.
- `orElse(x)` **always evaluates x**. `orElseGet(() -> x)` evaluates lazily, so use it for expensive defaults.
- Java 9–11 additions: `or()`, `stream()`, `ifPresentOrElse()`, `isEmpty()`.

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-45"></a>
### 4.45 File handling with streams

**Byte streams** (`InputStream`/`OutputStream`) for binary data. **Character streams** (`Reader`/`Writer`) for text with an encoding.

```java
try (BufferedReader r = new BufferedReader(new FileReader("orders.csv", StandardCharsets.UTF_8))) {
    String line;
    while ((line = r.readLine()) != null) process(line);
}
```

- **Buffering** (`BufferedReader`/`BufferedWriter`/`BufferedInputStream`) avoids a system call per byte, which is a huge speed-up.
- **Decorator pattern:** streams wrap streams (`new BufferedReader(new InputStreamReader(socket.getInputStream(), UTF_8))`).
- Always specify the **charset**, and always use **try-with-resources**.
- `Serializable` + `ObjectOutputStream` for Java serialisation (legacy and a security risk: prefer JSON or Protobuf).

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>

---

<a id="s4-46"></a>
### 4.46 NIO and NIO.2

**NIO (Java 1.4), non-blocking I/O:**
- **Channels** (two-way, `FileChannel`, `SocketChannel`) + **Buffers** (`ByteBuffer`: position/limit/capacity, `flip()` between writing and reading).
- **Selectors:** **one thread monitors many channels** for readiness. This is the basis of Netty, Tomcat NIO and reactive servers.
- **Direct buffers** (off-heap) and **memory-mapped files** for fast large-file I/O. `transferTo` for zero-copy.

**NIO.2 (Java 7) — `java.nio.file`:**
```java
Path p = Path.of("data", "orders.csv");
List<String> lines = Files.readAllLines(p);              // small files
try (Stream<String> s = Files.lines(p)) { s.filter(...).count(); }   // large files, lazy
Files.writeString(p, text);  Files.copy(a, b, REPLACE_EXISTING);  Files.walk(dir);
```
- `WatchService` for file-change notifications, file attributes, symbolic links.

**Interview Q&A:**
- *Blocking I/O vs NIO selectors?* Blocking needs one thread per connection. A selector lets a few threads serve thousands of connections. (Virtual threads now make the blocking style scale too.)

<sub>[↑ Contents](#contents) · [⌂ Index](00-INDEX.md)</sub>
