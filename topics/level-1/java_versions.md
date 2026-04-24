### Topic: Java Versions — Level 1

> **Advanced coverage:** [Level 2 — Advanced Java (17 & 21 hands-on)](../level-2/advanced_java.md)

**Why it matters (Karat angle)**
Citibank runs a mix of Java 11 and 17 in production. Being able to say "I used feature X on a real project" is much stronger than reciting features from a blog. Interviewers check whether you can frame features in terms of the problem they solve.

**Core concept**

**LTS cadence:** Oracle releases a Long-Term Support (LTS) version every 2-3 years. Non-LTS releases (every 6 months) are fine for greenfield projects but rarely used in enterprise banking. Versions you must know: **8, 11, 17, 21**.

| Version | Year | Headline features | Why it shipped |
|---------|------|-------------------|----------------|
| **8** | 2014 | Lambdas, Stream API, `Optional`, `java.time`, default methods, `CompletableFuture` | Bring functional style; replace mutable `Date`/`Calendar` |
| **11** | 2018 | `var` in lambda params, `HttpClient`, String methods (`isBlank`/`strip`/`lines`), single-file `java` | First post-8 LTS; polish standard library |
| **17** | 2021 | Sealed classes, pattern matching `instanceof`, switch expressions (stable), records (stable), text blocks | Reduce boilerplate around type dispatch and data classes |
| **21** | 2023 | Virtual threads, record patterns, sequenced collections, pattern matching in switch | Change the concurrency model; complete pattern matching |

**Feature deep-cuts by version (the ones interviewers probe):**

**Java 8 — the functional turn**
- `list.stream().filter(...).collect(toList())` — declarative collection processing.
- `Optional<T>` — explicit absence instead of null; forces consumers to handle both cases.
- `LocalDate`, `Instant`, `Duration` — immutable, thread-safe date/time.
- Default methods — interface evolution without breaking implementers.

**Java 11 — polish**
- `HttpClient` — sync + async HTTP, no external dependency.
- `"text".strip()` (Unicode-aware trim), `""isBlank()`, `"\n".lines()` (Stream<String>), `"-".repeat(n)`.
- Local variable `var` extended to lambda parameters: `(var x) -> x * 2` — mainly for adding annotations.

**Java 17 — clarity**
- **Pattern matching `instanceof`**: `if (obj instanceof String s) { s.length(); }` — no manual cast.
- **Switch expression**: `return switch (day) { case MON, TUE -> "week"; default -> "weekend"; };` — returns a value, no fall-through, exhaustive.
- **Sealed classes**: `sealed class Shape permits Circle, Square` — restrict subclassing to a known set; pairs with pattern matching.
- **Text blocks**: triple-quoted multi-line strings.
- **Records (stable)**: `record Point(int x, int y) {}`.

**Java 21 — concurrency & patterns**
- **Virtual threads**: `Thread.ofVirtual().start(r)` — millions of lightweight threads; great for blocking I/O (JDBC, HTTP).
- **Record patterns**: destructure inside switch.
- **Sequenced collections**: `List.getFirst()`, `.getLast()`, `.reversed()` on `List`, `Deque`, `LinkedHashSet`, etc.

**Mental model:**
- 8 = functional style arrived.
- 11 = standard library cleaned up.
- 17 = type system got smarter.
- 21 = concurrency model modernised.

**Real-world use cases**
- **Java 8 streams** — replacing for-loop accumulators in list processing pipelines.
- **Java 11 HttpClient** — hitting an internal `/health` or calling a downstream microservice without adding Apache HttpClient.
- **Java 17 switch expressions** — replacing nested if-else chains in dispatcher code; exhaustive check at compile time.
- **Java 17 records** — REST DTOs, event payloads.
- **Java 21 virtual threads** — high-throughput REST services where each request blocks on DB/HTTP — was thread-per-request with OS threads bottlenecked, now scales to 10k+ concurrent requests on a single JVM.

**Working code example**
```java
// File: topics/level-1/JavaVersionsDemo.java
import java.time.LocalDate;
import java.util.Optional;

public class JavaVersionsDemo {

    // ---------- Java 8: lambda + Optional + java.time ----------
    static Optional<String> findCode(String input) {
        return Optional.ofNullable(input)
                       .filter(s -> s.startsWith("TXN"));
    }

    static LocalDate tPlusTwo(LocalDate trade) {
        return trade.plusDays(2);                    // immutable; returns new instance
    }

    // ---------- Java 11: String.isBlank, strip, repeat ----------
    static String banner(String title) {
        if (title == null || title.isBlank()) title = "UNTITLED";
        return "=".repeat(40) + "\n" + title.strip() + "\n" + "=".repeat(40);
    }

    // ---------- Java 17: switch expression + pattern matching instanceof ----------
    static String classify(Object o) {
        return switch (o) {                          // Java 21 pattern matching for switch
            case Integer i when i > 0 -> "positive int " + i;
            case Integer i              -> "non-positive int " + i;
            case String s               -> "string of length " + s.length();
            case null                    -> "null";
            default                      -> "other: " + o.getClass().getSimpleName();
        };
    }

    // ---------- Java 17: records ----------
    record Money(long amount, String currency) {}

    // ---------- Java 21: virtual threads ----------
    static void virtualThreadDemo() throws InterruptedException {
        Thread vt = Thread.ofVirtual().name("worker-1").start(() -> {
            System.out.println("Running in: " + Thread.currentThread());
        });
        vt.join();
    }

    public static void main(String[] args) throws Exception {
        System.out.println(findCode("TXN42"));       // Optional[TXN42]
        System.out.println(tPlusTwo(LocalDate.of(2026, 4, 23)));

        System.out.println(banner("  Settlement Report  "));

        System.out.println(classify(7));             // positive int 7
        System.out.println(classify("hello"));       // string of length 5

        System.out.println(new Money(1500, "USD"));  // Money[amount=1500, currency=USD]

        virtualThreadDemo();
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java ships LTS versions roughly every 2-3 years — 8, 11, 17, and 21 are the key ones; each adds features that reduce boilerplate and/or extend the runtime.
2. **Why/when:** Java 8 brought functional style. Java 11 polished the standard library (HttpClient, String methods). Java 17 made type dispatch cleaner (switch expressions, pattern matching, records). Java 21 changed the concurrency model with virtual threads.
3. **Example:** Pattern matching `instanceof String s` eliminates the cast that used to follow every `instanceof` — less boilerplate, no risk of stale cast.
4. **Gotcha/tradeoff:** Virtual threads in Java 21 help **blocking I/O** workloads — thread-per-request now scales. They don't help CPU-bound tasks; those still need platform thread pools sized to core count.

**Common pitfalls**
- Thinking switch expressions are the same as switch statements — expressions return a value, must be exhaustive, no fall-through.
- Treating virtual threads as "magical scaling" — they shine for I/O-bound work; CPU-bound tasks need traditional thread pools.
- Using features available in one version on a project targeting an older one — check the `release` and `source`/`target` compiler flags.
- Forgetting that records are final and fields are final — can't extend or mutate.

**Self-check question**
Name three Java 8 features and two Java 17 features that you've used or could use in a typical Spring Boot REST service. For one of them, describe the boilerplate it replaced.
