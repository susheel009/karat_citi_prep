### Topic: Immutable Classes — Level 1

> **Advanced coverage:** [Level 3 — Immutable Classes and Objects](../level-3/immutable_classes_and_objects.md)

**Why it matters (Karat angle)**
Immutability is the cheapest form of concurrency safety and the simplest form of correctness — interviewers ask this to see if you know *why* `String`, `LocalDate`, and `BigDecimal` are immutable, and whether you can build one yourself.

**Core concept**

An **immutable class** has two guarantees:
1. The observable state of an instance never changes after construction.
2. Every operation that "modifies" the object returns a new instance instead.

**The five rules for building one correctly:**

| Rule | Why |
|------|-----|
| Mark the class `final` | Prevents subclasses from adding mutable fields |
| All fields `private final` | No reassignment after construction |
| No setters — ever | Only constructor can set state |
| Defensively copy mutable inputs in the constructor | Caller can't mutate the internal state through their reference |
| Defensively copy mutable fields in getters | Caller can't mutate them through the returned reference |

If all fields are themselves immutable (`String`, primitive wrappers, `LocalDate`), you can skip defensive copies.

**What immutability buys you**

| Benefit | Why |
|---------|-----|
| **Thread safety for free** | No mutation → no races → no locks needed |
| **Safe as `HashMap` keys / `HashSet` members** | Hash code can't change after insertion |
| **Safe sharing** | Give out references freely; no one can corrupt state |
| **Failure atomicity** | Constructor either builds a valid object or throws — no half-initialised state |
| **Caching-friendly** | Value-based equality → can memoise results |

**Immutable types built into Java**
- All primitive wrappers: `Integer`, `Long`, `Boolean`, `Double`, ...
- `String`
- All `java.time` types: `LocalDate`, `LocalDateTime`, `Instant`, `Duration`, `Period`
- `BigInteger`, `BigDecimal`
- Records (Java 14+) — if fields are immutable types, the record is immutable
- `Map.of(...)`, `List.of(...)`, `Set.of(...)` — Java 9+ unmodifiable factories

**Records as the modern immutable** — Java 14+:
```java
public record Money(long amount, String currency) {}
// Auto-generated: constructor, accessors, equals, hashCode, toString
// Fields are implicitly private final; the record itself is final
```
Records are the shortest path to immutability for data carriers.

**Mental model:** an immutable object is a **photograph** — you can make a copy with different content (new photograph), but you can never edit the original.

**Real-world use cases**
- **Value objects:** `Money`, `EmailAddress`, `AccountId` — domain primitives where equality by content is what you want.
- **DTOs for REST APIs:** immutable response payloads prevent accidental mutation after serialisation.
- **Configuration:** loaded once at startup; used read-only everywhere.
- **Event payloads** in event-driven systems: immutable to survive the queue and multiple consumers.
- **Cache keys:** hash code can't drift; invalidation is safe.

**Working code example**
```java
// File: topics/level-1/ImmutableMoney.java
import java.util.*;

public final class ImmutableMoney {                   // rule 1: class is final

    private final long amount;                         // rule 2: private final
    private final String currency;

    public ImmutableMoney(long amount, String currency) {
        if (currency == null || currency.isBlank()) {
            throw new IllegalArgumentException("currency required");
        }
        this.amount = amount;
        this.currency = currency;                     // String is already immutable — no copy needed
    }

    public long getAmount()    { return amount; }
    public String getCurrency(){ return currency; }

    // rule 3: no setters — provide "mutator" methods that return a NEW instance
    public ImmutableMoney plus(long delta) {
        return new ImmutableMoney(this.amount + delta, this.currency);
    }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ImmutableMoney m)) return false;
        return amount == m.amount && currency.equals(m.currency);
    }
    @Override public int hashCode() { return Objects.hash(amount, currency); }
    @Override public String toString(){ return amount + " " + currency; }
}
```

**Edge case: mutable fields require defensive copying**
```java
// File: topics/level-1/ImmutableAccount.java
import java.util.*;

public final class ImmutableAccount {

    private final String id;
    private final List<String> tags;                  // List is mutable — must copy

    public ImmutableAccount(String id, List<String> tags) {
        this.id = id;
        // rule 4: defensive copy of mutable input
        this.tags = List.copyOf(tags);                // Java 10+: returns unmodifiable snapshot
    }

    public String getId() { return id; }

    public List<String> getTags() {
        // rule 5: already unmodifiable — safe to return directly
        return tags;
    }
}

// Contrast — broken version
class BrokenAccount {
    private final List<String> tags;
    public BrokenAccount(List<String> tags) { this.tags = tags; }   // ❌ shares reference
    public List<String> getTags() { return tags; }                  // ❌ exposes internal state
}
// Caller can do: account.getTags().clear();  — BrokenAccount state is now corrupted.
```

**Edge case: record with a List field**
```java
// Records don't auto-defensive-copy. Do it in the compact constructor.
public record ImmutableOrder(String id, List<String> items) {
    public ImmutableOrder {
        items = List.copyOf(items);           // snapshot in constructor
    }
    // accessor items() returns the unmodifiable list — safe
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** An immutable class has state that cannot change after construction; any "modification" returns a new instance. Enforced with `final` class, `private final` fields, no setters, and defensive copies for mutable inputs and outputs.
2. **Why/when:** Immutability buys thread safety without locks, makes objects safe to share freely, safe as map keys, and failure-atomic. Use for value objects, DTOs, cache keys, and event payloads.
3. **Example:** `ImmutableMoney.plus(delta)` doesn't mutate — it returns a new `ImmutableMoney` with the updated amount, same as `String.concat` or `LocalDate.plusDays`.
4. **Gotcha/tradeoff:** Immutable objects allocate on every modification — for deeply nested structures, this can produce garbage. Records in Java 14+ remove the boilerplate; if all fields are immutable types, a record is trivially immutable.

**Common pitfalls**
- Forgetting `final` on the class — a subclass can add mutable state.
- Storing a reference to a mutable input without copying — caller can still mutate through their reference.
- Returning the mutable internal field directly from a getter — caller can mutate through the returned reference.
- Making `equals`/`hashCode` use a mutable field that could still change in a subclass — breaks the hash contract.

**Self-check question**
Write a `record Range(LocalDate start, LocalDate end)` that validates `start ≤ end`. Does it need a defensive copy of the `LocalDate` fields? Why or why not?
