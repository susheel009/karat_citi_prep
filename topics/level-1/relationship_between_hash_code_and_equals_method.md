### Topic: Relationship Between hashCode and equals — Level 1

> **Related:** [Level 1 — HashMap](./hashmap.md) · [Level 3 — Hash Collision](../level-3/hash_collision.md)

**Why it matters (Karat angle)**
The `hashCode`/`equals` contract is a classic gotcha question. Interviewers ask this to check whether you understand *why* the contract exists (hash-based collections depend on it) — not just the rules.

**Core concept**

**The contract** (`java.lang.Object` specifies):

1. **`equals` consistency:** `a.equals(b)` must return the same result on repeated calls (as long as neither is mutated).
2. **`equals` is reflexive:** `a.equals(a)` is always `true`.
3. **`equals` is symmetric:** `a.equals(b) == b.equals(a)`.
4. **`equals` is transitive:** if `a.equals(b)` and `b.equals(c)`, then `a.equals(c)`.
5. **`equals(null)`:** always `false`.
6. **The critical rule:** if `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` MUST be true. *The converse is not required* — two unequal objects CAN share a hash code (collision).

**What this means practically:**
- Always override **both** or **neither**. Overriding only one breaks the contract.
- The `hashCode` must be derived from the same fields `equals` uses.

**Why the contract exists: HashMap mechanics**

`HashMap.get(key)` does:
1. Compute `key.hashCode()` → pick bucket.
2. Walk the bucket, comparing with `equals()`.

If `equals` says two objects are equal but their hash codes differ, the map puts them in different buckets — and `get` on either never finds the other.

```java
class BadCity {
    String name;
    @Override public boolean equals(Object o) {
        return o instanceof BadCity b && name.equals(b.name);
    }
    // ❌ No hashCode override — uses Object's (memory address)
}

BadCity a = new BadCity(); a.name = "NYC";
BadCity b = new BadCity(); b.name = "NYC";
System.out.println(a.equals(b));        // true
System.out.println(a.hashCode() == b.hashCode()); // FALSE — memory addresses differ
Map<BadCity, Integer> m = new HashMap<>();
m.put(a, 1);
System.out.println(m.get(b));            // null — a and b land in different buckets
```

**How to implement both correctly**

**Modern idiom (Java 7+):**
```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Money m)) return false;
    return amount == m.amount && currency.equals(m.currency);
}

@Override public int hashCode() {
    return Objects.hash(amount, currency);    // null-safe, combines fields
}
```

**Key points:**
- Use `instanceof` pattern (Java 16+) for type check + auto-cast.
- Compare primitives with `==`, objects with `.equals()` (use `Objects.equals(a, b)` for null safety).
- Use **the same fields** for both `equals` and `hashCode`.
- For performance-critical code, use `31 * h + field.hashCode()` manually (classic recipe) — `Objects.hash(...)` boxes into a varargs array.

**Default implementations (Object class)**
- `equals` — reference equality (`this == o`).
- `hashCode` — derived from memory address (roughly).

**Shortcuts that handle it for you:**

| Tool | How |
|------|-----|
| IDE generator | IntelliJ/Eclipse → Generate → equals() and hashCode() |
| Lombok | `@EqualsAndHashCode` on the class |
| Records (Java 14+) | Auto-generated — uses all record components |

**Mental model:** `hashCode` picks the filing cabinet drawer. `equals` rummages through that drawer. If two "equal" objects pick different drawers, `get` never finds them.

**Real-world use cases**
- Any class used as a `HashMap` key or `HashSet` element.
- Value objects (`Money`, `EmailAddress`, `AccountId`) where equality is by content.
- Cache keys — `Map<CacheKey, CachedValue>` fails catastrophically if the contract is broken.
- Event deduplication — `Set<Event>` won't dedupe if events don't implement the contract.

**Working code example**
```java
// File: topics/level-1/HashEqualsDemo.java
import java.util.*;

public class HashEqualsDemo {

    // Proper value object
    static final class Money {
        private final long amount;
        private final String currency;

        Money(long amount, String currency) {
            this.amount = amount; this.currency = currency;
        }

        @Override public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Money m)) return false;
            return amount == m.amount && Objects.equals(currency, m.currency);
        }

        @Override public int hashCode() {
            return Objects.hash(amount, currency);
        }

        @Override public String toString() { return amount + " " + currency; }
    }

    public static void main(String[] args) {
        Money a = new Money(100, "USD");
        Money b = new Money(100, "USD");
        Money c = new Money(100, "EUR");

        System.out.println(a.equals(b));              // true  — same content
        System.out.println(a.hashCode() == b.hashCode());  // true  — contract held
        System.out.println(a.equals(c));              // false — currency differs

        // Works as a HashMap key because contract is correct
        Map<Money, String> m = new HashMap<>();
        m.put(a, "first deposit");
        System.out.println(m.get(b));                 // "first deposit" — b matches a

        // Works in a HashSet too (HashSet is backed by HashMap)
        Set<Money> seen = new HashSet<>();
        seen.add(a);
        System.out.println(seen.contains(b));         // true
    }
}
```

**Edge case: mutable fields in hashCode**

```java
class MutableBadKey {
    String id;
    @Override public int hashCode() { return id.hashCode(); }
    @Override public boolean equals(Object o) {
        return o instanceof MutableBadKey k && id.equals(k.id);
    }
}

MutableBadKey k = new MutableBadKey();
k.id = "alpha";
Map<MutableBadKey, Integer> m = new HashMap<>();
m.put(k, 1);
k.id = "beta";                         // hashCode changed
System.out.println(m.get(k));          // null — entry lost in the original bucket
System.out.println(m.size());          // 1 — still one entry, just orphaned
```

Moral: keys must be immutable, or at minimum, fields used in `hashCode`/`equals` must never change after insertion.

**What to say in the interview (4-beat answer)**
1. **Definition:** The `equals`/`hashCode` contract requires that if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must also be true. Equal hashes don't imply equal objects (collisions are allowed), but the reverse MUST hold.
2. **Why/when:** Because `HashMap.get` computes the hash first to pick a bucket, then uses `equals` to find within the bucket. If the hash code doesn't match, `get` can never find an object that would otherwise be equal.
3. **Example:** Override both `equals` and `hashCode` using the same fields — `Objects.hash(amount, currency)` in `hashCode` matches the fields compared in `equals`. Records give this for free.
4. **Gotcha/tradeoff:** Override one without the other, or mutate a key's hash-contributing field after insertion — both silently corrupt `HashMap` lookups. Records and Lombok's `@EqualsAndHashCode` are the safest modern options.

**Common pitfalls**
- Overriding `equals` but not `hashCode` — IDE often warns; the test fails when the class becomes a `HashMap` key.
- Using fields in `equals` that aren't used in `hashCode` — violates the contract.
- Mutating a field used in `hashCode` after insertion — the entry becomes orphaned.
- Using `==` to compare object fields inside `equals` — fails for `String`/`Integer` outside Integer cache.
- Custom `hashCode` that returns a constant (like `return 0`) — technically valid but all entries land in one bucket (O(n) lookups).

**Self-check question**
Can two non-equal objects share a hash code? Can two equal objects have different hash codes? Explain what each case implies for `HashMap` behaviour.
