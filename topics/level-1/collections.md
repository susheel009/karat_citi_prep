### Topic: Collections — Level 1

> **Advanced coverage:** [Level 2 — Collections (internals, immutable)](../level-2/collections.md) · [Level 1 — HashMap](./hashmap.md) · [Level 3 — HashMap (internals)](../level-3/hashmap.md)

**Why it matters (Karat angle)**
Collections are the most-used data structures in Java backend work. Interviewers ask this to check whether you can pick the right one for a scenario — not just that `ArrayList` exists. A senior answer frames each choice in terms of access pattern and Big-O.

**Core concept**

The Collections Framework is organised around three top-level interfaces:

| Interface | Allows duplicates? | Ordered? | Keyed? |
|-----------|:-------------------:|:--------:|:------:|
| `List<E>` | ✅ | ✅ (by index) | ❌ |
| `Set<E>` | ❌ | depends | ❌ |
| `Map<K,V>` | values: yes / keys: no | depends | ✅ |

**List implementations**

| Impl | Backing | Get by index | Insert at end | Insert middle | Thread-safe? |
|------|---------|:------------:|:-------------:|:-------------:|:------------:|
| `ArrayList` | Resizable array | O(1) | O(1) amortised | O(n) | ❌ |
| `LinkedList` | Doubly linked | O(n) | O(1) | O(1) once located | ❌ |
| `Vector` | Resizable array | O(1) | O(1) | O(n) | ✅ (obsolete) |
| `CopyOnWriteArrayList` | Array (copied on write) | O(1) | O(n) | O(n) | ✅ |

**Set implementations**

| Impl | Ordering | Lookup | Null allowed? |
|------|----------|:------:|:-------------:|
| `HashSet` | None (hash order) | O(1) | ✅ one null |
| `LinkedHashSet` | Insertion order | O(1) | ✅ one null |
| `TreeSet` | Sorted (natural/Comparator) | O(log n) | ❌ |

**Map implementations**

| Impl | Ordering | Lookup | Null keys? | Thread-safe? |
|------|----------|:------:|:----------:|:------------:|
| `HashMap` | None | O(1) avg | ✅ one null | ❌ |
| `LinkedHashMap` | Insertion (or access) order | O(1) avg | ✅ | ❌ |
| `TreeMap` | Sorted keys | O(log n) | ❌ (compares keys) | ❌ |
| `Hashtable` | None | O(1) | ❌ | ✅ (obsolete) |
| `ConcurrentHashMap` | None | O(1) | ❌ | ✅ |

**HashSet vs HashMap — the structural connection:** `HashSet` is literally implemented as a `HashMap<E, Object>` where every value is a dummy constant. `set.add("x")` becomes `map.put("x", PRESENT)`. This is why both give O(1) membership tests.

**Mental model:** picking a collection = answering three questions:
1. Do I need order? (Insertion? Sorted? None?)
2. Do I need uniqueness? (Set vs List)
3. Am I looking up by key or scanning? (Map vs List/Set)

**Real-world use cases**
- **`ArrayList`** — default for any ordered, indexed data (DTO lists returned from REST, JDBC result sets).
- **`LinkedList`** — rarely worth it; only when you need O(1) insertion at both ends AND don't need indexed access. Use `ArrayDeque` instead most of the time.
- **`HashMap`** — caches, ID lookups, config stores.
- **`TreeMap`** — when you need to iterate keys in order (e.g., time-series data keyed by `Instant`).
- **`LinkedHashMap`** — LRU cache base (override `removeEldestEntry`), or preserving request-order for audit logs.
- **`HashSet`** — membership tests (already-seen IDs, tag sets).
- **`ConcurrentHashMap`** — shared in-memory cache across threads (alternative to a real cache for small data).

**Working code example**
```java
// File: topics/level-1/CollectionsDemo.java
import java.util.*;

public class CollectionsDemo {

    public static void main(String[] args) {

        // ---------- List ----------
        List<String> list = new ArrayList<>();
        list.add("apple"); list.add("banana"); list.add("apple");
        System.out.println(list);                  // [apple, banana, apple]
        System.out.println(list.get(1));           // banana — O(1)
        list.sort(Comparator.naturalOrder());
        System.out.println(list);                  // [apple, apple, banana]

        // ---------- Set ----------
        Set<String> hashSet = new HashSet<>(list);
        Set<String> treeSet = new TreeSet<>(list);
        System.out.println(hashSet);               // [apple, banana] — unordered
        System.out.println(treeSet);               // [apple, banana] — alphabetical

        // ---------- Map ----------
        Map<String, Integer> scores = new HashMap<>();
        scores.put("Alice", 95);
        scores.put("Bob", 87);
        scores.put("Alice", 99);                   // overwrites
        System.out.println(scores.get("Alice"));   // 99
        System.out.println(scores.getOrDefault("Charlie", 0)); // 0

        // ---------- Iteration patterns ----------
        for (Map.Entry<String, Integer> e : scores.entrySet()) {
            System.out.println(e.getKey() + " -> " + e.getValue());
        }
        scores.forEach((k, v) -> System.out.println(k + "=" + v));

        // ---------- Java 9+: immutable factories ----------
        List<String> fixed = List.of("a", "b", "c");               // immutable
        Map<String, Integer> m  = Map.of("x", 1, "y", 2);          // immutable
        // fixed.add("d");                                          // UnsupportedOperationException

        // ---------- LinkedHashMap: preserves insertion order ----------
        Map<String, Integer> ordered = new LinkedHashMap<>();
        ordered.put("first", 1); ordered.put("second", 2); ordered.put("third", 3);
        System.out.println(ordered);                                // insertion order preserved

        // ---------- TreeMap: sorted keys ----------
        NavigableMap<Integer, String> tree = new TreeMap<>();
        tree.put(3, "c"); tree.put(1, "a"); tree.put(2, "b");
        System.out.println(tree);                                   // {1=a, 2=b, 3=c}
        System.out.println(tree.firstEntry());                      // 1=a
        System.out.println(tree.ceilingKey(2));                     // 2 — smallest key >= 2
    }
}
```

**Edge case: LinkedHashMap as an LRU cache**
```java
// File: topics/level-1/LruCacheDemo.java
Map<String, String> lru = new LinkedHashMap<>(16, 0.75f, true) {    // accessOrder=true
    @Override protected boolean removeEldestEntry(Map.Entry<String,String> e) {
        return size() > 3;                                          // cap at 3
    }
};
lru.put("a", "1"); lru.put("b", "2"); lru.put("c", "3");
lru.get("a");                                                       // touch "a"
lru.put("d", "4");                                                  // evicts "b" (least recently used)
System.out.println(lru);                                            // {c=3, a=1, d=4}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** The Collections Framework provides `List` (ordered, duplicates allowed), `Set` (unique), and `Map` (key-value) interfaces with multiple implementations tuned for different access patterns.
2. **Why/when:** Choose `ArrayList` for random access, `LinkedList` for ends-insertion, `HashMap` for O(1) key lookup, `TreeMap` when sorted key order matters.
3. **Example:** `HashSet` is internally a `HashMap` with dummy values — `set.add("x")` is `map.put("x", PRESENT)`, which is why both give O(1) membership tests.
4. **Gotcha/tradeoff:** `HashMap` allows one `null` key; `TreeMap` rejects null keys because it must compare them. `Hashtable` and `Vector` are obsolete — use `ConcurrentHashMap` / `Collections.synchronizedList` / `CopyOnWriteArrayList` instead.

**Common pitfalls**
- Reaching for `LinkedList` by default — `ArrayList` is faster for almost every real workload. Only use `LinkedList` if you insert/remove at the head heavily.
- Using `Vector` or `Hashtable` in new code — legacy; use concurrent alternatives.
- Iterating a `HashMap` and expecting a consistent order — it's unordered by design.
- Mutating a key's hashCode after inserting into a `HashMap` — the entry becomes unreachable.

**Self-check question**
You need O(1) lookup by ID AND iteration in insertion order. Which Map implementation? What about lookup by ID AND iteration in sorted key order?
