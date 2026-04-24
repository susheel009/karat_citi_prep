### Topic: HashMap — Level 1

> **Advanced coverage:** [Level 3 — HashMap (internal implementation, thread safety)](../level-3/hashmap.md) · [Level 3 — Hash Collision](../level-3/hash_collision.md) · [Level 1 — hashCode/equals contract](./relationship_between_hash_code_and_equals_method.md)

**Why it matters (Karat angle)**
`HashMap` is the most-asked Java data structure — interviewers use it to probe your understanding of hashing, the `hashCode`/`equals` contract, and performance characteristics. Saying "it's O(1)" is not enough; explaining *why* and when it degrades is.

**Core concept**

A `HashMap` stores key-value entries in an **array of buckets**. Each bucket holds a small list (or tree, in Java 8+) of entries that hash to that bucket.

**Put mechanism:**
1. Compute `key.hashCode()`.
2. Apply a secondary hash spread: `hash = h ^ (h >>> 16)` — mixes high bits into low bits to defend against weak hash functions.
3. Pick bucket index: `index = hash & (n - 1)` where `n` is the array length (always a power of 2).
4. Walk the bucket's entries. For each, compare keys with `equals()`. If found → replace value. Otherwise → append a new entry.

**Get mechanism:** same hash computation → bucket → walk entries comparing with `equals()` → return value or null.

**Collision handling:**
- Multiple keys can hash to the same bucket (collision).
- Until Java 7: bucket was a **singly-linked list**. Worst case O(n) per bucket.
- From Java 8: when a bucket exceeds **TREEIFY_THRESHOLD = 8** entries (and total capacity ≥ 64), the bucket is converted to a **red-black tree** — worst case O(log n) per bucket. Untreeifies below 6.

**Resize (rehash):**
- **Load factor** (default 0.75) = threshold when size exceeds `capacity * loadFactor`.
- On threshold hit, capacity doubles and every entry is reinserted at its new index. O(n) but amortised O(1) per put.
- That's why initialising with a known size matters for large maps: `new HashMap<>(expectedSize * 4 / 3 + 1)`.

**Nulls and key rules:**
- `HashMap` allows one `null` key and any number of `null` values.
- `Hashtable` and `TreeMap` reject null keys; `ConcurrentHashMap` rejects both null keys and null values.

**Thread safety:**
- `HashMap` is NOT thread-safe. Concurrent writes in Java 7 could form an infinite loop inside a bucket (famous bug, fixed in Java 8, but still don't use concurrent).
- Use `ConcurrentHashMap` for shared maps.

**Mental model:** think of it as a coat-check. `hashCode` picks the aisle (bucket). `equals` walks the aisle until your coat is found. If aisles get crowded, aisles become sorted racks (trees) for faster search.

**Real-world use cases**
- **Cache:** map product ID → cached product DTO.
- **Config lookup:** map property name → value.
- **Counting / grouping:** `map.merge(word, 1, Integer::sum)` for word counts.
- **Deduplication:** put IDs into a `HashSet` (backed by `HashMap`) to check seen.
- **Index over a list:** pre-compute `Map<ID, Entity>` from a list for O(1) lookups.

**Working code example**
```java
// File: topics/level-1/HashMapDemo.java
import java.util.*;

public class HashMapDemo {

    public static void main(String[] args) {

        // Basic operations
        Map<String, Integer> ages = new HashMap<>();
        ages.put("Alice", 30);
        ages.put("Bob", 25);
        ages.put("Alice", 31);                    // overwrites; same key, new value
        System.out.println(ages.get("Alice"));    // 31
        System.out.println(ages.get("Charlie"));  // null
        System.out.println(ages.getOrDefault("Charlie", -1)); // -1

        // null key and null values allowed
        ages.put(null, 0);
        ages.put("missing", null);
        System.out.println(ages.get(null));       // 0

        // Common idioms
        ages.putIfAbsent("Dan", 40);              // only writes if absent
        ages.computeIfAbsent("Eve", k -> 50);     // only computes if absent
        ages.merge("Alice", 1, Integer::sum);     // 31 + 1 = 32

        // Iteration — no guaranteed order
        for (Map.Entry<String, Integer> e : ages.entrySet()) {
            System.out.println(e.getKey() + " -> " + e.getValue());
        }

        // Sizing for known load (avoid resize cost)
        Map<Integer, String> big = new HashMap<>(2000);  // fits ~1500 entries at 0.75 load
        for (int i = 0; i < 1000; i++) big.put(i, "v" + i);
    }
}
```

**Edge case: broken hashCode/equals makes HashMap unusable**
```java
// File: topics/level-1/BadKey.java
class BadKey {
    int id;
    BadKey(int id) { this.id = id; }
    // No hashCode/equals override — uses Object's default (memory address)
}

Map<BadKey, String> m = new HashMap<>();
m.put(new BadKey(1), "a");
System.out.println(m.get(new BadKey(1)));        // null — different Object identity, different hash
```
Moral: any custom class used as a `HashMap` key MUST override both `hashCode()` and `equals()` consistently.

**Edge case: mutating a key after insertion**
```java
class MutableKey {
    String name;
    @Override public int hashCode() { return name.hashCode(); }
    @Override public boolean equals(Object o) { return o instanceof MutableKey m && name.equals(m.name); }
}

MutableKey k = new MutableKey(); k.name = "alpha";
Map<MutableKey, Integer> map = new HashMap<>();
map.put(k, 1);
k.name = "beta";                                 // hashCode changes
System.out.println(map.get(k));                  // null — entry is orphaned in old bucket
```
Moral: keys should be immutable, or at least never mutate fields used in `hashCode`/`equals` after insertion.

**What to say in the interview (4-beat answer)**
1. **Definition:** `HashMap<K,V>` is an array-backed hash table: keys are hashed to bucket indices; collisions within a bucket are resolved with a linked list, promoted to a red-black tree (Java 8+) once the bucket exceeds eight entries.
2. **Why/when:** It's the default when you need O(1) average lookup by key. Use `LinkedHashMap` for insertion-order iteration, `TreeMap` for sorted keys, `ConcurrentHashMap` for multi-threaded access.
3. **Example:** On `put`, the map computes `hash(key)`, picks a bucket via `hash & (capacity - 1)`, and compares with `equals()` to either replace or append. Load factor of 0.75 triggers a doubling resize to keep collisions low.
4. **Gotcha/tradeoff:** Worst-case degrades to O(log n) per operation (Java 8+ trees) when hash quality is poor — before Java 8 it was O(n) via linked lists. Always override `hashCode`/`equals` together for custom keys, and never mutate keys after insertion.

**Common pitfalls**
- Using a mutable object as a key and mutating a field used in `hashCode` — the entry becomes orphaned.
- Overriding `equals` but not `hashCode` — breaks the contract; two "equal" objects can land in different buckets.
- Expecting iteration order — it's unordered; use `LinkedHashMap` if order matters.
- Ignoring initial capacity for known-size maps — triggers repeated O(n) resizes during loading.
- Using `HashMap` across threads — race conditions, possibly corrupted state (especially pre-Java 8).

**Self-check question**
You insert 100,000 entries into a `HashMap` at default capacity. Roughly how many resizes happen during loading, and what does each resize cost? How would you initialise the map to avoid this?
