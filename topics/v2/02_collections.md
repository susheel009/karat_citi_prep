# 02 — Collections Internals

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## Rapid-Fire Q&A

### Q1: How does `HashMap` work internally?
**A:** Array of buckets (default capacity 16, load factor 0.75). Hash the key → spread hash (`hashCode ^ (hashCode >>> 16)`) → bucket index = `hash & (capacity - 1)`. Collisions form a linked list. When a bucket has ≥8 entries AND capacity ≥64, it converts to a red-black tree (O(log n) instead of O(n)). Reverts to list at ≤6 entries. Resizes (doubles capacity, rehashes all entries) when `size > capacity × loadFactor`.

### Q2: `HashMap` vs `Hashtable` vs `Collections.synchronizedMap` vs `ConcurrentHashMap`?
**A:**
| | Thread-safe | Locking | null key/value | Use |
|--|:-:|----------|:-:|-----|
| `HashMap` | ❌ | None | ✅/✅ | Single thread |
| `Hashtable` | ✅ | Global lock on `this` | ❌/❌ | Legacy — don't use |
| `synchronizedMap` | ✅ | Global mutex | ✅/✅ | Quick retrofit |
| `ConcurrentHashMap` | ✅ | Per-bucket (CAS + sync on head) | ❌/❌ | Production concurrent |

`ConcurrentHashMap` also provides atomic compound ops: `computeIfAbsent`, `compute`, `merge`.

### Q3: Why is `ConcurrentHashMap` faster than `synchronizedMap`?
**A:** `synchronizedMap` holds a single global lock — all ops serialised. `ConcurrentHashMap` uses per-bucket locking: CAS for empty buckets, `synchronized` on bucket head node for collisions. Multiple threads can read/write different buckets concurrently.

### Q4: What changed in `ConcurrentHashMap` between Java 7 and 8?
**A:** Java 7: segment-based locking (16 fixed segments). Java 8: eliminated segments — uses CAS + `synchronized` on individual bucket head nodes. Better concurrency granularity and reduced memory overhead.

### Q5: `ArrayList` vs `LinkedList` — when use which?
**A:** `ArrayList` almost always. O(1) random access, contiguous memory (cache-friendly). `LinkedList` has O(n) random access, poor cache locality, extra memory per node (prev/next pointers). `LinkedList` is only better for frequent insertions/removals at known positions via iterator — which is rare.

### Q6: What's the `equals`/`hashCode` contract?
**A:** If `a.equals(b)` → `a.hashCode() == b.hashCode()` (must hold). Reverse is NOT required (collisions are allowed). If you override `equals` without `hashCode`, objects become unfindable in `HashMap`/`HashSet` — they go to different buckets. Keys must be effectively immutable — if hashCode changes after insertion, the entry is lost.

### Q7: Why must keys in a `HashMap` be effectively immutable?
**A:** The bucket is determined by `hashCode()` at insertion time. If you mutate the key later, its hashCode changes, but the entry stays in the old bucket. `get()` computes the new hash, looks in the wrong bucket, returns null. The entry is effectively lost.

### Q8: `TreeMap` — complexity? When use it?
**A:** Red-black tree (self-balancing BST). All ops O(log n). Maintains sorted key order. Use for range queries (`subMap`, `floorKey`, `ceilingKey`) or when you need sorted iteration. `NavigableMap` interface.

### Q9: `LinkedHashMap` — what's special?
**A:** `HashMap` + doubly-linked list maintaining insertion order (or access order if constructed with `accessOrder=true`). Override `removeEldestEntry()` to build an LRU cache. Iteration order is predictable — unlike `HashMap`.

### Q10: `CopyOnWriteArrayList` — when use it?
**A:** Every write copies the entire backing array. Reads are lock-free. Use for many-read, few-write scenarios: listener lists, configuration snapshots. Never for write-heavy workloads — O(n) per write.

### Q11: `ArrayDeque` — when use it?
**A:** Resizable array-based deque. Faster than `Stack` (legacy) and `LinkedList` (poor cache locality) for both stack and queue use cases. Default choice for stack (`push/pop`) and queue (`offer/poll`).

### Q12: `Set` implementations — which and when?
**A:** `HashSet`: O(1) add/contains, backed by `HashMap`. `LinkedHashSet`: maintains insertion order. `TreeSet`: sorted order, O(log n) ops, backed by `TreeMap`. `EnumSet`: bitfield-backed, fastest for enum values.

### Q13: `PriorityQueue` — what is it?
**A:** Min-heap. `poll()` always returns the smallest element. O(log n) insert and remove-min. Not thread-safe — use `PriorityBlockingQueue` for concurrent access. Custom ordering via `Comparator`.

### Q14: What's `Collections.unmodifiableList` vs `List.of` vs `List.copyOf`?
**A:** `unmodifiableList(list)`: unmodifiable *view* — original can still be mutated. `List.of(...)`: truly immutable, rejects nulls. `List.copyOf(list)`: immutable copy, rejects nulls. In Java 10+, prefer `List.copyOf` for defensive copies.

### Q15: How does `HashMap` resize work?
**A:** When `size > capacity × loadFactor`, capacity doubles. Every entry is rehashed — bucket index recalculated for new capacity. Each old bucket splits into at most 2 new buckets (old index or old index + old capacity). This is O(n) — expensive.

### Q16: What's `IdentityHashMap`?
**A:** Uses `==` (reference equality) instead of `equals()` for key comparison. Rare use: serialisation frameworks, interning, when you explicitly want reference semantics.

---

## Key Code Patterns

### ConcurrentHashMap — atomic compound operation
```java
// WRONG — check-then-act race
if (!map.containsKey(key)) map.put(key, compute(key));

// RIGHT — atomic
map.computeIfAbsent(key, k -> compute(k));
```

### LRU Cache with LinkedHashMap
```java
Map<String, Object> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
        return size() > MAX_ENTRIES;
    }
};
```

### Immutable collections (Java 10+)
```java
List<String> immutable = List.of("a", "b", "c");          // truly immutable
Map<String, Integer> map = Map.of("one", 1, "two", 2);    // immutable map
Set<String> set = Set.copyOf(mutableSet);                  // immutable copy
```

---

## Can you answer these cold?

- [ ] Walk through `HashMap.put()` — hashing, bucket index, collision, treeification
- [ ] Four-way map comparison — HashMap/Hashtable/synchronizedMap/ConcurrentHashMap
- [ ] `equals`/`hashCode` contract — what breaks if violated
- [ ] `ArrayList` vs `LinkedList` — cache locality argument
- [ ] `ConcurrentHashMap` locking — Java 7 segments vs Java 8 CAS+sync
- [ ] When to use `TreeMap`, `LinkedHashMap`, `CopyOnWriteArrayList`
- [ ] Build an LRU cache with `LinkedHashMap` in 5 lines

[← Back to Index](./00_INDEX.md)
