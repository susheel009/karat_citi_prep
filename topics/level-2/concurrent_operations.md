### Topic: Concurrent Operations on Collections — Level 2

> **Foundation:** [Level 1 — Iterators](../level-1/iterators.md) · [Level 1 — Collections](../level-1/collections.md)
> **Related:** [Level 2 — Multithreading](./multithreading_in_java.md) · [Level 3 — Critical Section of Code](../level-3/critical_section_of_code.md)

**Why it matters (Karat angle)**
Banking services handle concurrent reads and writes to shared data structures (caches, session stores, counters). Interviewers ask this to see if you know the thread-safe collection options and their trade-offs — not just "use `ConcurrentHashMap`."

**Core concept**

Standard collections (`ArrayList`, `HashMap`) are **not thread-safe**. Three approaches exist:

| Approach | How | Read perf | Write perf | Consistency |
|----------|-----|:---------:|:----------:|:-----------:|
| `synchronized` wrapper | `Collections.synchronizedList(list)` | Slow (global lock) | Slow (global lock) | Strong |
| `CopyOnWrite*` | `CopyOnWriteArrayList` | Fast (no lock) | Slow (copies array) | Eventual (stale reads) |
| `Concurrent*` | `ConcurrentHashMap` | Fast (segment/stripe locks) | Fast (fine-grained) | Weakly consistent |

**`Collections.synchronizedList/Map/Set`:**

```java
// File: topics/level-2/SynchronizedWrapperDemo.java

List<String> syncList = Collections.synchronizedList(new ArrayList<>());
syncList.add("a");                                       // lock acquired, add, lock released
syncList.get(0);                                         // lock acquired, get, lock released

// DANGER: compound operations are NOT atomic
// if (!syncList.contains("b")) syncList.add("b");       // RACE CONDITION

// Fix: manually synchronize compound operations
synchronized (syncList) {
    if (!syncList.contains("b")) {
        syncList.add("b");
    }
}

// Iteration also needs external sync:
synchronized (syncList) {
    for (String s : syncList) {                          // iterator is NOT thread-safe
        System.out.println(s);
    }
}
```

**`ConcurrentHashMap`:**

```java
// File: topics/level-2/ConcurrentHashMapDemo.java

ConcurrentHashMap<String, AtomicInteger> counters = new ConcurrentHashMap<>();

// Atomic compound operations built-in:
counters.putIfAbsent("logins", new AtomicInteger(0));    // atomic check-and-insert
counters.get("logins").incrementAndGet();                 // atomic increment

// compute: atomic read-modify-write
counters.compute("logins", (key, val) -> {
    if (val == null) return new AtomicInteger(1);
    val.incrementAndGet();
    return val;
});

// merge: atomic upsert
ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();
scores.merge("Alice", 10, Integer::sum);                 // "Alice" -> 10
scores.merge("Alice", 5, Integer::sum);                  // "Alice" -> 15

// forEach with parallelism threshold:
scores.forEach(1,                                        // parallelism threshold
    (key, val) -> System.out.println(key + "=" + val)
);
```

| `ConcurrentHashMap` method | What it does | Atomic? |
|---------------------------|-------------|:-------:|
| `putIfAbsent(k, v)` | Insert only if key missing | ✅ |
| `compute(k, remapping)` | Read-modify-write | ✅ |
| `computeIfAbsent(k, fn)` | Insert computed value if key missing | ✅ |
| `merge(k, v, remapping)` | Upsert with merge function | ✅ |
| `replace(k, old, new)` | CAS-style replace | ✅ |

**`CopyOnWriteArrayList`:**

```java
// File: topics/level-2/CopyOnWriteDemo.java

CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();
listeners.add("listener-1");
listeners.add("listener-2");

// Read: no locking, reads the current snapshot
for (String l : listeners) {                             // safe — iterates a snapshot
    System.out.println(l);
}

// Write: copies the ENTIRE array
listeners.add("listener-3");                             // O(n) — new array created

// Concurrent modification during iteration: NO exception, but iterator sees old data
```

| | `ConcurrentHashMap` | `CopyOnWriteArrayList` | `synchronizedList` |
|--|---------------------|----------------------|-------------------|
| Lock granularity | Stripe/segment | None (read) / full copy (write) | Global lock |
| Iteration | Weakly consistent | Snapshot (stale OK) | Must externally sync |
| Best for | Frequent read + write | Frequent read, rare write | Legacy/simple cases |
| `null` keys/values | ❌ | ✅ | Depends on backing collection |

**Edge case: `ConcurrentHashMap` size and bulk operations are approximate**

```java
// File: topics/level-2/ConcurrentMapEdgeCase.java

ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
// map.size() is NOT exact under concurrent modification — it's an estimate
// Use mappingCount() for a long-valued (more accurate) estimate
long approxSize = map.mappingCount();

// Bulk operations (forEach, search, reduce) use a parallelism threshold:
// threshold = 1 → maximize parallelism
// threshold = Long.MAX_VALUE → sequential (single-threaded)
String found = map.search(1, (k, v) -> v.contains("CRITICAL") ? v : null);
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Standard collections aren't thread-safe. Options: `Collections.synchronized*` (global lock), `ConcurrentHashMap` (striped locks, atomic compound ops), `CopyOnWriteArrayList` (snapshot reads, copy-on-write).
2. **Why/when:** `ConcurrentHashMap` for concurrent caches and counters. `CopyOnWriteArrayList` for listener/observer lists (many reads, rare writes). `synchronizedList` only for legacy compatibility.
3. **Example:** `map.computeIfAbsent("key", k -> expensiveLoad(k))` — atomically checks if the key exists and inserts a computed value if not. No external synchronisation needed.
4. **Gotcha/tradeoff:** `Collections.synchronizedList` only synchronises individual method calls — compound operations (check-then-act) still need external `synchronized`. `ConcurrentHashMap` doesn't allow `null` keys or values — unlike `HashMap`.

**Common pitfalls**
- Using `synchronizedList` and assuming iteration is safe — it's not; you must wrap in `synchronized(list)`.
- Using `CopyOnWriteArrayList` for write-heavy workloads — each write copies the entire array; O(n) per write.
- `if (!map.containsKey(k)) map.put(k, v)` on a `ConcurrentHashMap` — NOT atomic; use `putIfAbsent`.
- Assuming `ConcurrentHashMap.size()` is exact — it's an estimate under concurrency.
- Putting `null` in `ConcurrentHashMap` — throws `NullPointerException`.

**Self-check question**
You have a shared `Map<String, List<Transaction>>` where multiple threads add transactions. `ConcurrentHashMap` makes `put`/`get` thread-safe, but is `map.get("acct").add(txn)` thread-safe? Why or why not? How would you fix it?
