### Topic: Advanced Java — Level 2

> **Foundation:** [Level 1 — Java Versions](../level-1/java_versions.md) · [Level 1 — Java Fundamentals](../level-1/java_fundamentals.md)

**Why it matters (Karat angle)**
Citi is migrating services from Java 8/11 to 17+. Interviewers probe whether you've actually used the newer features — enhanced switch, text blocks, sealed classes, records, pattern matching — or just read about them. A senior answer shows how these features reduce boilerplate and prevent bugs.

**Core concept**

L1 covered feature lists. L2 is about using them fluently in real code.

**Enhanced switch (Java 14+, preview in 12)**

Old switch:
```java
// verbose, fall-through prone
String result;
switch (status) {
    case "ACTIVE":
        result = "Process";
        break;
    case "PENDING":
        result = "Queue";
        break;
    default:
        result = "Skip";
        break;
}
```

Enhanced switch — expression form:
```java
// File: topics/level-2/EnhancedSwitchDemo.java
String result = switch (status) {
    case "ACTIVE"  -> "Process";
    case "PENDING" -> "Queue";
    default        -> "Skip";
};
// No break needed. No fall-through risk. Returns a value.
```

**Switch with multiple labels + blocks:**
```java
int priority = switch (severity) {
    case "CRITICAL", "FATAL" -> 1;
    case "WARNING"           -> 2;
    case "INFO"              -> {
        log.info("Low priority event");
        yield 3;                          // 'yield' returns value from a block
    }
    default -> 4;
};
```

**Pattern matching for switch (Java 21):**
```java
// File: topics/level-2/PatternSwitchDemo.java
static String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0  -> "positive int: " + i;
        case Integer i             -> "non-positive int: " + i;
        case String s              -> "string of length " + s.length();
        case null                  -> "null";
        default                    -> "other: " + obj.getClass().getSimpleName();
    };
}
// Combines instanceof + cast + conditional in one line — no manual casting.
```

**try-with-resources (Java 7, enhanced in Java 9)**

Before:
```java
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("data.csv"));
    return br.readLine();
} finally {
    if (br != null) br.close();          // verbose, error-prone
}
```

After (Java 7):
```java
try (BufferedReader br = new BufferedReader(new FileReader("data.csv"))) {
    return br.readLine();
}
// br.close() called automatically, even on exception
```

Java 9 enhancement — effectively final variable:
```java
// File: topics/level-2/TryWithResourcesDemo.java
BufferedReader br = new BufferedReader(new FileReader("data.csv"));
try (br) {                                // no need to declare inside try()
    return br.readLine();
}
// Works if br is effectively final (assigned once, never reassigned)
```

**Sealed classes (Java 17)**
```java
// File: topics/level-2/SealedDemo.java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}
public final class Triangle implements Shape { /* ... */ }

// Compiler knows ALL subtypes → exhaustive switch (no default needed)
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.w() * r.h();
    case Triangle t  -> /* ... */ 0;
};
```

**Records (Java 16)**
```java
// File: topics/level-2/RecordDemo.java
public record Transaction(Long id, BigDecimal amount, Instant timestamp) {
    // Compact constructor for validation
    public Transaction {
        if (amount.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Amount must be positive");
    }
    // Auto-generates: constructor, getters (id(), amount(), timestamp()),
    //                 equals(), hashCode(), toString()
}
```

| Java feature | Version | Replaces |
|-------------|---------|----------|
| Enhanced switch (expressions) | 14 | Verbose switch + break |
| Text blocks (`"""`) | 15 | String concat for multi-line |
| Records | 16 | Lombok `@Value` / manual DTOs |
| Sealed classes | 17 | Open hierarchies |
| Pattern matching `instanceof` | 16 | Cast after instanceof |
| Pattern matching `switch` | 21 | Chains of if-else instanceof |
| Virtual threads | 21 | Platform thread pools for I/O |
| try-with-resources (enhanced) | 9 | Manual resource cleanup |

**Real-world use cases**
- **Records** — DTOs, API request/response objects, event payloads. Replace Lombok `@Value` with zero dependency.
- **Sealed classes** — domain models with fixed variants (payment types, account statuses). Compiler enforces exhaustive handling.
- **Enhanced switch** — mapping enums to behaviour, status-code handling, request routing.
- **Text blocks** — SQL strings, JSON templates, HTML snippets in tests.
- **try-with-resources** — every DB connection, file handle, HTTP client response. Non-negotiable for resource safety.

**What to say in the interview (4-beat answer)**
1. **Definition:** Java 17 and 21 add sealed classes (closed hierarchies), records (immutable data carriers), pattern matching (type-safe dispatch), and virtual threads (lightweight concurrency).
2. **Why/when:** Records replace boilerplate DTOs. Sealed classes + pattern matching switch give compile-time exhaustiveness — miss a case and the compiler tells you. Virtual threads replace reactive frameworks for I/O-heavy services.
3. **Example:** A `sealed interface PaymentType permits Wire, ACH, Check` with a pattern-matching switch ensures every new payment type forces a code change at every switch site — no silent fallthrough.
4. **Gotcha/tradeoff:** Records are shallowly immutable — if a field is a mutable `List`, the record's content can still change. Use `List.copyOf()` in the compact constructor.

**Common pitfalls**
- Using `yield` outside a switch expression block — it's not a general return.
- Forgetting that records generate `equals`/`hashCode` based on ALL fields — large fields (byte arrays) make equality checks expensive.
- Sealed classes require all permitted subtypes to be in the same package (or module). Cross-package hierarchies won't compile.
- Text blocks strip leading whitespace based on the closing `"""` position — misplacing it changes your string content.

**Self-check question**
You have a `sealed interface Event permits Click, Purchase, Refund`. You add a new `Cancel` event but forget to add a `case` in a switch expression. What happens at compile time vs runtime? What if the switch had a `default` clause?
