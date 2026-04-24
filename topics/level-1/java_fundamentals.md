### Topic: Java Fundamentals — Level 1

> **Advanced coverage:** [Level 2 — Advanced Java](../level-2/advanced_java.md) for Java 17/21 features.

**Why it matters (Karat angle)**
Version awareness is a quick signal of whether you've kept pace with the ecosystem. Banks like Citi run Java 11 or 17 in production — being able to say "I used Java 11's HttpClient on a real project" lands better than reciting features from a tutorial.

**Core concept**

"Java Fundamentals" in the cheatsheet means three things: **Java 8 and 11 language features**, **Collections**, and **Exception handling**. You need to name features confidently and write them without hesitation.

**Java 8 (2014) — the functional turn**

| Feature | Purpose | Typical use |
|---------|---------|-------------|
| Lambda | Inline `(x) -> x * 2` | Stream operations, comparators, callbacks |
| Stream API | Declarative collection processing | `filter`, `map`, `collect`, `reduce` |
| `Optional<T>` | Explicit absence over `null` | Method returns; avoid NPE chains |
| `java.time` | Immutable, thread-safe date/time | `LocalDate`, `Instant`, `Duration` |
| Default/static methods in interfaces | Add methods without breaking implementers | Evolve APIs without breaking callers |
| `CompletableFuture` | Async composition | Chained async I/O |

**Java 11 (2018) — the first post-8 LTS**

| Feature | Purpose | Typical use |
|---------|---------|-------------|
| `var` in lambda params | Apply annotations to inferred types | `(var x) -> ...` with `@Nonnull` |
| `HttpClient` | Built-in async HTTP | Replaces Apache HttpClient for simple cases |
| `String.isBlank/strip/lines/repeat` | Unicode-aware String helpers | Input sanitisation, multi-line processing |
| Single-file run | `java Script.java` | Quick scripts without `javac` |
| `Files.readString/writeString` | One-line file I/O | Config loading, test fixtures |

**Mental model:** Java 8 changed *how* you process data (functional pipelines). Java 11 polished the standard library (String, HTTP, Files).

**Real-world use cases**
- **REST service filtering:** `list.stream().filter(...).map(...).collect(toList())` replaces a for-loop accumulator.
- **Null-safe config:** `Optional.ofNullable(config.get("key")).orElseThrow(...)` makes missing config explicit.
- **Parsing audit logs:** `Files.lines(path).filter(...).count()` streams a large file without loading it.
- **Consuming an internal API:** Java 11 `HttpClient.newHttpClient().send(...)` is enough — no Maven dep needed for simple GET/POST.

**Working code example**
```java
// File: topics/level-1/JavaFundamentalsDemo.java
import java.net.URI;
import java.net.http.*;
import java.time.LocalDate;
import java.util.*;
import java.util.stream.*;

public class JavaFundamentalsDemo {

    // ---------- Java 8: Stream + lambda + Optional ----------
    static int sumOfEvens(List<Integer> nums) {
        return nums.stream()
                   .filter(n -> n % 2 == 0)          // lambda predicate
                   .mapToInt(Integer::intValue)       // method reference
                   .sum();                            // terminal op
    }

    static String firstMatchOrDefault(List<String> codes, String prefix) {
        return codes.stream()
                    .filter(c -> c.startsWith(prefix))
                    .findFirst()                     // returns Optional<String>
                    .orElse("NO_MATCH");             // unwrap with default
    }

    // ---------- Java 8: java.time ----------
    static LocalDate settlementDate(LocalDate tradeDate, int businessDays) {
        LocalDate d = tradeDate;
        int added = 0;
        while (added < businessDays) {
            d = d.plusDays(1);
            if (d.getDayOfWeek().getValue() < 6) added++;  // skip weekends
        }
        return d;
    }

    // ---------- Java 11: HttpClient ----------
    static String fetch(String url) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest req = HttpRequest.newBuilder(URI.create(url)).build();
        HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
        return resp.body();
    }

    // ---------- Java 11: String helpers ----------
    static boolean isMeaningfulInput(String s) {
        // isBlank = null-safe-ish (handles "   " as blank); strip is Unicode-aware trim
        return s != null && !s.isBlank() && s.strip().length() > 2;
    }

    public static void main(String[] args) {
        System.out.println(sumOfEvens(List.of(1,2,3,4,5,6)));            // 12
        System.out.println(firstMatchOrDefault(List.of("TXN1","ABC"), "TXN")); // TXN1
        System.out.println(settlementDate(LocalDate.of(2026, 4, 23), 2));      // T+2
        System.out.println(isMeaningfulInput("  hi  "));                 // true
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java Fundamentals covers the core language — types, collections, exceptions — plus Java 8's functional features (lambda, stream, Optional, time API) and Java 11's API polish (HttpClient, String methods).
2. **Why/when:** Streams replace verbose loops for collection processing; `Optional` forces null handling at the API boundary; `java.time` replaces the mutable, non-thread-safe `Date`/`Calendar`.
3. **Example:** `list.stream().filter(...).mapToInt(...).sum()` is three readable lines versus a five-line loop with an accumulator variable.
4. **Gotcha/tradeoff:** Streams are lazy and single-use — calling a terminal operation twice on the same stream throws `IllegalStateException`. Always build a new stream.

**Common pitfalls**
- Using `new Date()` or `Calendar` in new code — mutable, not thread-safe, confusing API. Always `java.time`.
- `Optional.get()` without `isPresent()` — throws `NoSuchElementException`; use `orElse`, `orElseThrow`, `ifPresent`.
- Treating `Optional` as a parameter type — it was designed for return types; using it in parameters or fields is an anti-pattern.
- Calling `stream()` twice on the same source expecting different results — streams are not reusable.

**Self-check question**
What's the difference between `String.trim()` and `String.strip()` (Java 11), and why does it matter for internationalised user input?
