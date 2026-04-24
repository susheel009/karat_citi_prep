### Topic: Hash Collision — Level 3

> **Foundation:** [Level 2 — Collections (HashMap internals)](../level-2/collections.md)
> **Related:** [Level 3 — HashMap (internal implementation)](./hashmap.md)

**Why it matters (Karat angle)**
This is the deepest HashMap question interviewers ask: "What happens when two keys hash to the same bucket? Why did Java 8 change the internal implementation? What's the DoS vulnerability?" Knowing the answer proves you understand data structure internals.

**Core concept**

**What is a hash collision?**
Two distinct keys produce the same bucket index: `hash(keyA) & (capacity - 1) == hash(keyB) & (capacity - 1)`.

**Collision resolution in HashMap across Java versions:**

| Java version | Bucket structure | Worst-case lookup |
|:------------:|-----------------|:-----------------:|
| Java 1–7 | Linked list only | O(n) |
| Java 8+ | Linked list → Red-black tree (at 8+) | O(log n) |

**Why Java 8 changed: the HashDoS attack**

In Java 7, all entries in a bucket were in a linked list — O(n) lookup. An attacker could craft keys with the same hashCode, forcing all entries into one bucket → O(n²) for n insertions → denial of service.

```java
// File: topics/level-3/HashDosDemo.java

// Strings designed to collide (same hashCode in Java):
// "Aa".hashCode() == "BB".hashCode() == 2112
// "AaAa", "AaBB", "BBAa", "BBBB" — all hashCode 2031744

// Attacker sends 10,000 keys that all hash to the same bucket
// Java 7: 10,000 entries in one linked list → get() is O(10,000)
// Java 8: at 8 entries, bucket treeifies → get() is O(log 10,000) ≈ 13 comparisons
```

**Collision handling step by step (Java 8+):**

```
put("Alice", value1):
  1. hash("Alice") = 92668751
  2. index = 92668751 & 15 = 7  (capacity 16)
  3. Bucket 7 is empty → place Node directly

put("Bob", value2):
  1. hash("Bob") = 66965
  2. index = 66965 & 15 = 5
  3. Bucket 5 is empty → place Node directly

put("CollisionKey", value3):  // same hash index as "Alice"
  1. hash("CollisionKey") = ... 
  2. index = ... & 15 = 7  (same as Alice!)
  3. Bucket 7 has Node("Alice") → COLLISION
  4. Walk the chain: equals("Alice", "CollisionKey") → false
  5. Append to end of chain: Node("Alice") → Node("CollisionKey")
  
After 8+ collisions in bucket 7:
  6. Chain length > TREEIFY_THRESHOLD (8) AND table.length ≥ 64
  7. Convert linked list → red-black tree
  8. Nodes become TreeNodes, sorted by hash (then by Comparable if implemented)
```

**How `equals()` and `hashCode()` contract matters:**

```java
// File: topics/level-3/EqualsHashCodeDemo.java

// CONTRACT:
// 1. If a.equals(b) → a.hashCode() == b.hashCode()  (MUST)
// 2. If a.hashCode() == b.hashCode() → a.equals(b) is NOT guaranteed (collision)

// BROKEN: override equals without hashCode
class BrokenKey {
    String name;
    @Override public boolean equals(Object o) { return name.equals(((BrokenKey) o).name); }
    // hashCode NOT overridden → uses Object.hashCode() (memory address)
}

BrokenKey k1 = new BrokenKey("Alice");
BrokenKey k2 = new BrokenKey("Alice");
Map<BrokenKey, String> map = new HashMap<>();
map.put(k1, "value");
System.out.println(map.get(k2));                          // null! — k2 has different hashCode → different bucket

// CORRECT: consistent equals + hashCode
class CorrectKey {
    String name;
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CorrectKey c)) return false;
        return Objects.equals(name, c.name);
    }
    @Override public int hashCode() { return Objects.hash(name); }
}
```

**How a good `hashCode()` reduces collisions:**

```java
// File: topics/level-3/GoodHashCodeDemo.java

public class Account {
    private Long id;
    private String accountNumber;
    private String region;

    @Override
    public int hashCode() {
        // Objects.hash uses 31-based polynomial: ((id * 31 + accountNumber) * 31 + region)
        return Objects.hash(id, accountNumber, region);
    }

    // Why 31? Prime, odd, JIT optimises 31*x as (x << 5) - x
}

// Distribution check: a good hashCode spreads evenly across buckets
// Bad hashCode example: return 1; → everything in bucket 1 → O(n) for everything
// Good hashCode: spread bits across the int range → even bucket distribution
```

**Red-black tree requirements for collision resolution:**

When a bucket treeifies, the tree needs to order nodes. It uses:
1. Hash value comparison first.
2. If hashes are equal and keys implement `Comparable` → `compareTo()`.
3. If not Comparable → `System.identityHashCode()` as tiebreaker.

```java
// If your key class implements Comparable, HashMap uses it for tree ordering:
class AccountKey implements Comparable<AccountKey> {
    String id;
    @Override public int compareTo(AccountKey o) { return id.compareTo(o.id); }
}
// This makes treeified buckets more efficient — consistent ordering
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A hash collision occurs when two keys map to the same bucket. Java 7 used linked lists (O(n) lookup). Java 8+ treeifies buckets with >8 entries into red-black trees (O(log n)), introduced to mitigate HashDoS attacks.
2. **Why/when:** Collisions are inevitable — any hash function mapping infinite keys to finite buckets will collide. Good `hashCode()` implementations minimise collisions by distributing evenly.
3. **Example:** `"Aa".hashCode() == "BB".hashCode()` — these collide. In Java 7, 10,000 such keys → O(n) per lookup. In Java 8, treeification reduces worst case to O(log n).
4. **Gotcha/tradeoff:** Overriding `equals()` without `hashCode()` breaks HashMap — equal objects go to different buckets. Always override both together. Implementing `Comparable` on key classes improves treeified bucket performance.

**Common pitfalls**
- Overriding `equals` without `hashCode` — HashMap can't find entries.
- `hashCode()` returning a constant — all entries in one bucket, O(n) for everything.
- Mutable fields in `hashCode()` — if the hash changes after insertion, the entry is lost in the wrong bucket.
- Not implementing `Comparable` on key classes — red-black tree falls back to identity hash tiebreaker, less efficient.

**Self-check question**
You have a `HashMap<Employee, String>`. `Employee` overrides `equals()` (by name + department) but NOT `hashCode()`. You put an entry with key `new Employee("Alice", "IT")` and try to get it with `new Employee("Alice", "IT")`. What happens and why?
