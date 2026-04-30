### Topic: HashMap — Level 1

> **Advanced coverage:** [Level 3 — HashMap (internal implementation, thread safety)](../level-3/hashmap.md) · [Level 3 — Hash Collision](../level-3/hash_collision.md) · [Level 1 — hashCode/equals contract](./relationship_between_hash_code_and_equals_method.md)

---

## Part 1: Let's start from the name itself

**"Hash"** comes from the Old French *hacher* — meaning to chop. Same root as "hash browns" — you take potatoes, chop them up, mix them around, and what comes out is a jumbled result. In computing, a **hash function** does the same thing to data: takes any input (a string, a number, an object) and chops/mixes it into a fixed-size number called a **hash code**.

**"Map"** in computing means: a thing that **maps** one value to another — like a real-world map maps the name "Toronto" to coordinates. A Java `Map<String, Integer>` maps keys (strings) to values (integers).

So **HashMap** = a Map that uses a hash function to figure out where to store things.

That's literally the entire idea. Everything else is optimisation.

---

## Part 2: Why do we even need it?

Imagine you're building a phone book. Every person has a **name** (the key) and a **phone number** (the value). You have 1 million people. You want to look up "Alice Smith" and get her phone number fast.

**Approach 1 — dumb list:**
```
entries = [("Bob", "555-1"), ("Alice", "555-2"), ("Carol", "555-3"), ...]
```
To find Alice, you scan the whole list from the top. On average you check 500,000 entries before finding her. Slow: **O(n)**.

**Approach 2 — sort the list alphabetically:**
Now you can use binary search: jump to the middle, go left or right, repeat. About 20 checks for 1 million entries. Fast lookup: **O(log n)**. But now inserting a new person means shifting thousands of entries to keep the list sorted. Painful.

**Approach 3 — the HashMap idea:**
What if the name itself could **tell you** which slot to look in — no scanning, no searching, just go directly?

Here's the trick:
1. Take a big array with, say, 1000 empty slots.
2. Run "Alice Smith" through a hash function. It spits out some number — say, `42`.
3. Put `("Alice Smith", "555-2")` in slot 42.
4. To look up Alice later: hash her name again → 42 → go directly to slot 42.

**One step. No scanning. O(1).**

That's the magic.

---

## Part 3: But two names might hash to the same slot!

They will. It's unavoidable when you're mapping millions of possible inputs into 1000 slots (pigeonhole principle). Two different names can hash to the same number. This is called a **collision**.

**Fix:** each slot isn't a single entry — it's a tiny **list**. When two names collide, both go into the list at that slot.

```
Slot 42 → ("Alice Smith", "555-2") → ("Fred Jones", "555-9") → null
Slot 43 → null (empty)
Slot 44 → ("Bob Lee", "555-3") → null
```

When looking up Alice: hash → slot 42 → walk the tiny list (usually just 1-2 items) → compare keys → return value.

In Java, these slots are called **buckets**. Each bucket is a linked chain of entries.

Still effectively **O(1)** on average because each bucket has only 1-2 items if you size the array well.

---

## Part 4: What if too many names land in one slot?

Bad luck or a bad hash function could dump 50 names into one bucket. Now lookups at that bucket are O(50) — slow.

Java has two fixes:

**Fix 1: resize when the map gets too full.** Every HashMap tracks a **load factor**, default **0.75**. When the number of entries exceeds `capacity × 0.75`, Java doubles the array size and **rehashes** every entry into the new, bigger array. More slots = fewer collisions.

**Fix 2 (Java 8 onwards): turn a long chain into a tree.** If a single bucket gets more than 8 entries *and* the total capacity is ≥ 64, Java converts that bucket from a linked list into a **red-black tree** (a self-balancing binary search tree). Worst-case lookup at that bucket drops from O(n) to O(log n).

---

## Part 5: What's actually inside a Java HashMap?

Boil it down to the essentials:

```
HashMap<K, V>:
  Node<K, V>[] table          ← the array of buckets
  int size                    ← how many entries total
  float loadFactor            ← default 0.75
  int threshold               ← resize when size > threshold

Node<K, V>:                   ← each entry in a bucket chain
  int hash                    ← cached hash of the key
  K key
  V value
  Node<K, V> next             ← next Node in this bucket's chain (or null)
```

**When you call `map.put("Alice", 99)`:**
1. Compute `hash("Alice")` → some number.
2. Pick the bucket index with `hash & (table.length - 1)` — this is just a fast way to do `hash % table.length` when `table.length` is a power of 2.
3. Walk the chain at that bucket. For each Node, compare its key with `"Alice"` using `.equals()`.
4. If a matching key is found → replace the value. If not → append a new Node at the chain's end.
5. Increment `size`. If `size > threshold` → resize.

**When you call `map.get("Alice")`:**
1. Same hash + bucket calculation.
2. Walk the chain. Compare each key with `"Alice"` using `.equals()`.
3. Return the value when a match is found, or `null` if the chain ends.

---

## Part 6: Why you must override hashCode AND equals for custom keys

Look at step 3 above: **`.equals()` decides which key in the chain is "the one."** And `hashCode()` decides which bucket to go to in the first place.

If you use a custom class as a key (like `Money` or `UserId`) and **forget to override either method**, Java falls back to `Object`'s default:
- Default `hashCode` = based on memory address.
- Default `equals` = reference equality (`this == other`).

Two separate `Money(100, "USD")` objects will have DIFFERENT memory addresses, so:
- Their `hashCode`s differ → they land in DIFFERENT buckets.
- `map.put(m1, x); map.get(m2);` → returns null. The "same" money is invisible.

That's why the `hashCode` / `equals` contract matters. See [the dedicated topic](./relationship_between_hash_code_and_equals_method.md) for the full contract rules.

---

## Part 7: Quick reference table

| Property | Value |
|----------|-------|
| Average lookup | O(1) |
| Worst-case lookup (Java 7) | O(n) per bucket (linked list) |
| Worst-case lookup (Java 8+) | O(log n) per bucket (tree when > 8) |
| Default capacity | 16 |
| Default load factor | 0.75 |
| Treeify threshold | 8 entries per bucket |
| Allows null key? | ✅ one null key |
| Allows null values? | ✅ any number |
| Thread-safe? | ❌ (use `ConcurrentHashMap`) |
| Iteration order | None — unordered |
| Since when? | Java 1.2; trees added Java 8 |

---

## Real-world use cases

- **Cache:** `Map<ProductId, ProductDTO>` for product lookups.
- **Config store:** property name → value, loaded once at startup.
- **Counting / grouping:** `map.merge(word, 1, Integer::sum)` for word frequencies.
- **Deduplication:** `Set<OrderId>` (backed by HashMap) to check "have I seen this event?".
- **Index over a list:** pre-compute `Map<Id, Entity>` from a list for O(1) lookup.

---

## Working code example

```java
// File: topics/level-1/HashMapDemo.java
import java.util.*;

public class HashMapDemo {

    public static void main(String[] args) {

        // Basic operations
        Map<String, Integer> ages = new HashMap<>();
        ages.put("Alice", 30);
        ages.put("Bob",   25);
        ages.put("Alice", 31);                      // overwrites; same key, new value
        System.out.println(ages.get("Alice"));      // 31
        System.out.println(ages.get("Charlie"));    // null — not present
        System.out.println(ages.getOrDefault("Charlie", -1)); // -1

        // null key and null values are allowed in HashMap
        ages.put(null, 0);
        ages.put("missing", null);
        System.out.println(ages.get(null));         // 0

        // Common idioms (Java 8+)
        ages.putIfAbsent("Dan", 40);                // only writes if absent
        ages.computeIfAbsent("Eve", k -> 50);       // lazy create if absent
        ages.merge("Alice", 1, Integer::sum);       // 31 + 1 = 32

        // Iteration — order is not guaranteed
        for (Map.Entry<String, Integer> e : ages.entrySet()) {
            System.out.println(e.getKey() + " -> " + e.getValue());
        }

        // Sizing for known load — avoid repeated resizes
        Map<Integer, String> big = new HashMap<>(2000);  // fits ~1500 entries at 0.75 load
        for (int i = 0; i < 1000; i++) big.put(i, "v" + i);
    }
}
```

---

## Edge case: broken custom key (this is the trap)

```java
// File: topics/level-1/BadKey.java
class BadKey {
    int id;
    BadKey(int id) { this.id = id; }
    // Forgot to override hashCode and equals!
}

Map<BadKey, String> m = new HashMap<>();
m.put(new BadKey(1), "Alice");
System.out.println(m.get(new BadKey(1)));   // null — the "same" key isn't found
```

The two `BadKey(1)` instances are different objects in memory. Default `hashCode` differs → different buckets → `get` can't find it.

**Fix:** override both, using the same field(s):
```java
@Override public int hashCode() { return Integer.hashCode(id); }
@Override public boolean equals(Object o) {
    return o instanceof BadKey b && b.id == this.id;
}
```

---

## Edge case: don't mutate a key after insertion

```java
class MutableKey {
    String name;
    @Override public int hashCode() { return name.hashCode(); }
    @Override public boolean equals(Object o) {
        return o instanceof MutableKey m && name.equals(m.name);
    }
}

MutableKey k = new MutableKey(); k.name = "alpha";
Map<MutableKey, Integer> map = new HashMap<>();
map.put(k, 1);
k.name = "beta";                            // hashCode now returns a different value
System.out.println(map.get(k));             // null — entry is orphaned
System.out.println(map.size());             // 1 — still there, just unreachable
```

Keys should be **immutable**. Or at minimum, never mutate a field used in `hashCode`/`equals` after insertion.

---

## What to say in the interview (4-beat answer)

1. **Definition:** `HashMap<K,V>` is an array-backed hash table. Keys are hashed to bucket indices; collisions within a bucket are handled by chaining — a linked list in Java 7, promoted to a red-black tree from Java 8 once a bucket exceeds 8 entries.
2. **Why/when:** It's the default when you need O(1) average key-based lookup. For insertion-order iteration use `LinkedHashMap`; for sorted keys use `TreeMap`; for thread-safe use `ConcurrentHashMap`.
3. **Example:** On `put`, the map computes `hash(key)`, picks a bucket via `hash & (capacity - 1)`, then compares with `equals()` to either replace or append. Load factor 0.75 triggers a doubling resize to keep chains short.
4. **Gotcha/tradeoff:** Java 8 trees cap worst case at O(log n) per bucket instead of O(n). But the bigger trap is custom keys — you MUST override `hashCode` and `equals` together and never mutate keyed fields after insertion.

---

## Common pitfalls

- **Mutable key with mutated hash-field after insertion** → entry orphaned.
- **Override `equals` but forget `hashCode`** → contract broken; `get` returns null for "equal" objects.
- **Expect iteration order** → `HashMap` is unordered. Use `LinkedHashMap` if you need insertion order.
- **Initial capacity ignored for large known sizes** → triggers repeated O(n) resize during bulk load.
- **Use `HashMap` across threads** → race conditions, lost writes, possible infinite loop in Java 7 during resize.

---

## Self-check question

You insert 100,000 entries into a `HashMap` at default capacity (16). Roughly how many times does it resize during loading, and what's the cost of each resize? How would you initialise the map to avoid those resizes?
