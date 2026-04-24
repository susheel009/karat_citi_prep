### Topic: Collections (Internals) — Level 2

> **Foundation:** [Level 1 — Collections](../level-1/collections.md) · [Level 1 — HashMap](../level-1/hashmap.md)
> **Advanced:** [Level 3 — HashMap (internals, thread safety)](../level-3/hashmap.md) · [Level 3 — Hash Collision](../level-3/hash_collision.md)

**Why it matters (Karat angle)**
L1 covered which collection to pick. L2 is about *why* — internal data structures, resize mechanics, and immutable collections. Interviewers probe this to separate seniors who understand performance characteristics from juniors who just call `.add()`.

**Core concept**

**ArrayList internals**
- Backed by `Object[]`. Default initial capacity: **10**.
- When full, grows by **50%** (`newCapacity = oldCapacity + (oldCapacity >> 1)`).
- Growth = allocate new array + `Arrays.copyOf` (O(n) per resize).
- Amortised O(1) for `add` at end; worst case O(n) on resize.

```java
// File: topics/level-2/ArrayListInternals.java
// Avoid repeated resizes with initial capacity
List<Transaction> txns = new ArrayList<>(10_000);    // one allocation, not ~25 resizes
```

**HashMap internals (L2 depth — L3 covers thread safety)**
- Array of `Node<K,V>[]` buckets. Default capacity: **16**, load factor: **0.75**.
- Resize triggers when `size > capacity * loadFactor` → doubles capacity, rehashes all entries.
- **Java 8+ treeification:** when a bucket exceeds **8** entries AND total capacity ≥ 64, the linked list converts to a red-black tree (O(log n) lookup instead of O(n)).
- Tree reverts to list when bucket shrinks to **6** entries (UNTREEIFY_THRESHOLD).

```
put("Alice", 100):
  1. hashCode("Alice") → spread bits → bucket index
  2. If bucket empty → insert Node
  3. If bucket has entries → walk chain, check equals()
     - Match found → overwrite value
     - No match → append (or treeify if > 8)
  4. If size > threshold → resize (double capacity, rehash all)
```

| Parameter | Default | Effect |
|-----------|---------|--------|
| Initial capacity | 16 | Fewer resizes if set higher |
| Load factor | 0.75 | Lower = more space, fewer collisions; higher = denser, more collisions |
| Treeify threshold | 8 | Bucket converts list → tree |
| Untreeify threshold | 6 | Bucket converts tree → list |

**Pre-sizing to avoid resizes:**
```java
// File: topics/level-2/HashMapPresize.java
// Need to store 1000 entries without resize:
// threshold = capacity * 0.75, so capacity >= 1000 / 0.75 ≈ 1334 → next power of 2 = 2048
Map<String, Account> cache = new HashMap<>(2048);
// Or simpler: new HashMap<>(1000, 1.0f) — load factor 1.0, capacity = 1000 (resize at 1000)
```

**LinkedHashMap internals**
- Extends `HashMap` with a **doubly-linked list** through all entries.
- Insertion-order by default; access-order if constructed with `accessOrder=true` (used for LRU caches — covered in L1).
- Overhead: two extra pointers (`before`, `after`) per entry.

**TreeMap internals**
- Red-black tree (self-balancing BST). All operations O(log n).
- Keys must implement `Comparable` or provide a `Comparator` at construction.
- Supports `NavigableMap` operations: `ceilingKey`, `floorKey`, `headMap`, `tailMap`, `subMap`.

**Immutable collections — three approaches**

| Approach | Since | Truly immutable? | Null elements? |
|----------|-------|:----------------:|:--------------:|
| `Collections.unmodifiableList(list)` | Java 2 | ❌ View — original can still be modified | ✅ |
| `List.of(a, b, c)` | Java 9 | ✅ Structural + content (new list) | ❌ throws NPE |
| `List.copyOf(existingList)` | Java 10 | ✅ Defensive copy | ❌ throws NPE |

```java
// File: topics/level-2/ImmutableCollectionsDemo.java

// --- unmodifiableList: a VIEW, not a copy ---
List<String> original = new ArrayList<>(List.of("a", "b"));
List<String> view = Collections.unmodifiableList(original);
// view.add("c");                                      // UnsupportedOperationException
original.add("c");                                      // succeeds!
System.out.println(view);                               // [a, b, c] — view reflects mutation

// --- List.of: truly immutable ---
List<String> fixed = List.of("a", "b", "c");
// fixed.add("d");                                      // UnsupportedOperationException
// No backing list to mutate — genuinely immutable

// --- List.copyOf: defensive copy ---
List<String> source = new ArrayList<>(List.of("x", "y"));
List<String> copy = List.copyOf(source);
source.add("z");
System.out.println(copy);                               // [x, y] — unaffected

// --- Map.of / Map.ofEntries ---
Map<String, Integer> m = Map.of("a", 1, "b", 2);       // up to 10 entries
Map<String, Integer> big = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)                                   // any number of entries
);
```

**Edge case: structural sharing in immutable collections**
```java
// File: topics/level-2/ImmutableEdgeCase.java
List<String> a = List.of("x", "y");
List<String> b = List.copyOf(a);
System.out.println(a == b);                              // true — copyOf detects already-immutable, returns same reference
// No unnecessary copy — optimisation baked into the JDK.
```

**Real-world use cases**
- **Pre-sized `HashMap`** — bulk loading reference data (currency codes, branch configs) at startup.
- **`List.of()` / `Map.of()`** — constant lookup tables, enum-like mappings, API response metadata.
- **`List.copyOf()`** — defensive return from a getter that exposes internal state.
- **`TreeMap`** — time-series data keyed by `Instant`, range queries for audit logs.

**What to say in the interview (4-beat answer)**
1. **Definition:** `ArrayList` is backed by a resizable array (50% growth); `HashMap` is an array of buckets with linked lists that treeify at 8 entries (Java 8+). Both resize by doubling capacity.
2. **Why/when:** Understanding internals matters for performance — pre-sizing avoids O(n) resizes, load factor tuning controls collision rates, and immutable collections (`List.of`, `List.copyOf`) prevent unintended mutation vs `Collections.unmodifiable*` which is just a view.
3. **Example:** Loading 10,000 accounts into a `HashMap<>(16)` triggers ~10 resizes (16→32→…→16384), each rehashing all entries. Pre-sizing to `new HashMap<>(14_000)` avoids all resizes.
4. **Gotcha/tradeoff:** `Collections.unmodifiableList(list)` is NOT immutable — the original list can still be mutated and the view reflects it. Use `List.copyOf()` for true defensive copies.

**Common pitfalls**
- Confusing `Collections.unmodifiableList` (view) with `List.of` (truly immutable) — the view leaks mutations.
- Not pre-sizing `ArrayList` or `HashMap` for bulk loads — repeated resizes are O(n) each.
- Assuming `HashMap` iteration order is stable — it changes on resize.
- Using `null` elements with `List.of()` / `Map.of()` — throws `NullPointerException`, unlike `ArrayList` or `HashMap`.
- Forgetting that treeification requires keys to be `Comparable` — otherwise bucket stays as a linked list even at 8+ entries.

**Self-check question**
You create `HashMap<>(4, 0.5f)` and insert 3 entries. At which insertion does the first resize happen? What's the new capacity and threshold after that resize?
