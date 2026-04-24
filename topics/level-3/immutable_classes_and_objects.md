### Topic: Immutable Classes and Objects — Level 3

> **Foundation:** [Level 1 — Immutable Classes](../level-1/immutable_classes.md) · [Level 2 — Cloning](../level-2/cloning.md)

**Why it matters (Karat angle)**
L1 covered the definition and String example. L3 is about **creating** your own immutable classes correctly — defensive copying, handling mutable fields, and using Java 16+ records. Interviewers ask "make this class immutable" as a design exercise.

**Core concept**

**Rules for creating an immutable class:**
1. Declare the class `final` (prevent subclassing).
2. Make all fields `private final`.
3. No setters.
4. Provide values only through constructor.
5. **Defensive copy** all mutable fields on input AND output.

```java
// File: topics/level-3/ImmutableClassDemo.java

public final class Transaction {                          // 1. final class

    private final String id;                              // 2. private final
    private final BigDecimal amount;                      //    (BigDecimal is immutable — safe)
    private final LocalDateTime timestamp;                //    (LocalDateTime is immutable — safe)
    private final List<String> tags;                      //    (List is MUTABLE — needs defensive copy)
    private final Date legacyDate;                        //    (Date is MUTABLE — needs defensive copy)

    public Transaction(String id, BigDecimal amount, LocalDateTime timestamp,
                       List<String> tags, Date legacyDate) {
        this.id = id;
        this.amount = amount;
        this.timestamp = timestamp;
        this.tags = List.copyOf(tags);                    // 5. Defensive copy (unmodifiable)
        this.legacyDate = new Date(legacyDate.getTime()); // 5. Defensive copy
    }

    // 3. No setters — only getters

    public String getId() { return id; }
    public BigDecimal getAmount() { return amount; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public List<String> getTags() { return tags; }        // Already unmodifiable from List.copyOf
    public Date getLegacyDate() { return new Date(legacyDate.getTime()); } // 5. Defensive copy on output
}
```

**Without defensive copies — mutability leaks:**

```java
// File: topics/level-3/MutabilityLeakDemo.java

// WRONG — mutable field exposed
class BrokenImmutable {
    private final List<String> items;
    BrokenImmutable(List<String> items) { this.items = items; }      // stores reference
    List<String> getItems() { return items; }                         // returns reference
}

List<String> original = new ArrayList<>(List.of("a", "b"));
BrokenImmutable obj = new BrokenImmutable(original);
original.add("c");                                        // mutates the "immutable" object's state!
obj.getItems().add("d");                                  // also mutates internal state!
System.out.println(obj.getItems());                       // [a, b, c, d] — broken!

// RIGHT — defensive copies
class CorrectImmutable {
    private final List<String> items;
    CorrectImmutable(List<String> items) { this.items = List.copyOf(items); }
    List<String> getItems() { return items; }              // List.copyOf returns unmodifiable
}
```

**Java 16+ Records — immutable by design:**

```java
// File: topics/level-3/RecordImmutableDemo.java

// Records are final, fields are private final, getters auto-generated
public record Money(BigDecimal amount, String currency) {
    // Compact constructor — validation
    public Money {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount must be non-negative");
        if (currency == null || currency.length() != 3)
            throw new IllegalArgumentException("Currency must be 3-letter ISO code");
    }
}

// BUT: records with mutable fields still need defensive copies
public record AccountSnapshot(String id, List<Transaction> history) {
    public AccountSnapshot {
        history = List.copyOf(history);                   // defensive copy in compact constructor
    }
}

// Records provide: equals(), hashCode(), toString() based on all fields
Money m1 = new Money(BigDecimal.TEN, "USD");
Money m2 = new Money(BigDecimal.TEN, "USD");
System.out.println(m1.equals(m2));                        // true — value-based equality
```

**When to use immutables:**

| Use case | Why |
|----------|-----|
| Value objects (Money, Address) | Equality by value, thread-safe |
| DTOs / API responses | Prevent accidental mutation after creation |
| Map keys / Set elements | hashCode must not change after insertion |
| Concurrent sharing | No synchronisation needed |
| Caching | Cached object can't be corrupted by callers |

**What to say in the interview (4-beat answer)**
1. **Definition:** An immutable class is `final`, all fields are `private final`, no setters, and mutable fields are defensively copied on input and output. Java 16 records provide immutability by default (but still need defensive copies for mutable field types).
2. **Why/when:** Thread safety without synchronisation, safe as map keys, safe for caching. Value objects (Money, Address) should always be immutable.
3. **Example:** `List.copyOf(tags)` in the constructor prevents the caller from mutating the internal list. `new Date(date.getTime())` on the getter prevents the caller from mutating the internal Date.
4. **Gotcha/tradeoff:** Records look immutable but aren't if they contain mutable types like `List` or `Date` — you must still defensively copy in the compact constructor. `List.copyOf` rejects nulls — use `Collections.unmodifiableList(new ArrayList<>(list))` if nulls are expected.

**Common pitfalls**
- Storing a mutable reference directly — caller can mutate internal state.
- Returning a mutable reference from a getter — caller can mutate internal state.
- Assuming records are fully immutable — they are if all component types are immutable; otherwise, defensive copy is required.
- Using `Collections.unmodifiableList(list)` without copying — it's a view; the original list can still be mutated.
- Forgetting to make the class `final` — subclass can add mutable state.

**Self-check question**
You have `record Account(String id, Map<String, BigDecimal> balances)`. Someone calls `account.balances().put("USD", BigDecimal.ZERO)`. Does it modify the record's internal state? How do you prevent it?
