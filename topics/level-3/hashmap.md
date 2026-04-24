### Topic: HashMap — Internal Implementation and Thread Safety — Level 3

> **Foundation:** [Level 1 — Collections](../level-1/collections.md) · [Level 2 — Collections (resize, treeification)](../level-2/collections.md)
> **Related:** [Level 3 — Hash Collision](./hash_collision.md) · [Level 2 — Concurrent Operations](../level-2/concurrent_operations.md)

**Why it matters (Karat angle)**
"How does HashMap work internally?" is the single most asked Java data structure question. L2 covered resize/load factor. L3 goes deeper — hashing, bucket structure, treeification threshold, and thread-safety concerns that cause infinite loops in production.

**Core concept**

**Internal structure (Java 8+):**
```
HashMap
├── Node<K,V>[] table          // bucket array (power of 2 size)
│   ├── [0] null
│   ├── [1] Node → Node → null  // linked list (≤8 entries)
│   ├── [2] TreeNode (red-black tree, >8 entries)
│   ├── [3] null
│   └── ...
├── int size                    // total entries
├── int threshold               // capacity * loadFactor (resize trigger)
└── float loadFactor            // default 0.75
```

**How `put(key, value)` works internally:**

```java
// File: topics/level-3/HashMapInternalsDemo.java

// Step 1: Compute hash
// HashMap doesn't use key.hashCode() directly — it applies a spread function:
static int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    // XOR high bits into low bits — reduces collisions for poor hashCode() implementations
}

// Step 2: Find bucket index
// index = hash & (capacity - 1)
// This is equivalent to hash % capacity BUT only when capacity is a power of 2
// That's WHY HashMap capacity is always a power of 2

// Step 3: Insert
// If bucket is empty → new Node placed directly
// If bucket has entries → check for key equality:
//   - If key found → replace value
//   - If key not found → append to end of list/tree
//   - If list length > TREEIFY_THRESHOLD (8) AND table.length >= 64 → convert to red-black tree
```

**Treeification lifecycle:**

| Event | Threshold | Action |
|-------|:---------:|--------|
| Bucket chain grows | > 8 nodes | Convert linked list → red-black tree |
| Table is too small | < 64 buckets | Resize table instead of treeifying |
| Bucket tree shrinks | ≤ 6 nodes | Convert red-black tree → linked list |

```
Bucket 7 (before treeification):
  Node("Alice") → Node("Bob") → Node("Carol") → ... → Node(9th entry)
  └── 9 entries → TREEIFY! Convert to red-black tree

Bucket 7 (after treeification):
  TreeNode("Eve")
  ├── TreeNode("Alice")
  │   └── TreeNode("Bob")
  └── TreeNode("Carol")
      └── TreeNode("Dave")
  → O(log n) lookup instead of O(n)
```

**Resize internals:**

```java
// File: topics/level-3/HashMapResizeDemo.java

// Resize triggers when: size > capacity * loadFactor
// Default: capacity=16, loadFactor=0.75 → resize at 13th entry

// During resize:
// 1. New table = 2x capacity
// 2. For each entry in old table:
//    - Recalculate bucket: hash & (newCapacity - 1)
//    - Because capacity doubled, each bucket splits into two:
//      old index stays or moves to (old index + old capacity)
// 3. This is O(n) — all entries rehashed

// Example:
// Capacity 16 → 32
// Entry with hash 17: 17 & 15 = 1 (old bucket), 17 & 31 = 17 (new bucket)
// Entry with hash 33: 33 & 15 = 1 (old bucket), 33 & 31 = 1 (stays in bucket 1)
```

**Thread-safety problems:**

```java
// File: topics/level-3/HashMapThreadUnsafe.java

// HashMap is NOT thread-safe. Concurrent modifications cause:

// 1. Lost updates — two threads put at the same bucket, one overwrites the other
// 2. Infinite loop (Java 7) — concurrent resize causes circular linked list
//    Thread A and Thread B both trigger resize → both rehash same bucket
//    → linked list pointers form a cycle → get() spins forever
// 3. Corrupted state (Java 8+) — infinite loop fixed, but still data corruption

// Java 7 infinite loop reproduction:
// Thread A: resize(), rehashing bucket 3: A→B→C becomes C→B→A
// Thread B: simultaneously rehashing same bucket: C→B→A→B (cycle!)
// get("B") on this bucket → follows B→A→B→A→... forever

// Java 8 fixed this by using tail-insertion (append) instead of head-insertion during resize
// But concurrent put/get still causes:
// - Missed entries (put during resize → entry dropped)
// - ConcurrentModificationException during iteration
```

**Thread-safe alternatives:**

| | `Hashtable` | `synchronizedMap` | `ConcurrentHashMap` |
|--|------------|-------------------|---------------------|
| Locking | One global lock | One global lock | Striped/bucket locks |
| Read concurrency | Blocked by any write | Blocked by any write | Lock-free reads |
| Write concurrency | Serialised | Serialised | Concurrent (different segments) |
| null key/value | ❌ / ❌ | ✅ / ✅ | ❌ / ❌ |
| Performance | Slow | Slow | Fast |
| Use | Legacy (don't use) | Quick fix | Production |

**What to say in the interview (4-beat answer)**
1. **Definition:** HashMap stores entries in a `Node[] table`. The hash is spread via `hashCode() ^ (hashCode >>> 16)`, bucket index is `hash & (capacity - 1)`. Collisions form linked lists that treeify to red-black trees above 8 entries.
2. **Why/when:** O(1) average for get/put, O(log n) worst case (after treeification). Resize doubles capacity when `size > capacity * 0.75`, rehashing all entries.
3. **Example:** Two keys with hash collision go into the same bucket as a linked list. At 9 entries, the bucket converts to a red-black tree. At 6 or fewer, it reverts to a linked list.
4. **Gotcha/tradeoff:** HashMap is not thread-safe. In Java 7, concurrent resize caused infinite loops (circular linked lists). Java 8 fixed this with tail-insertion, but concurrent operations still corrupt data. Use `ConcurrentHashMap` for thread safety.

**Common pitfalls**
- Using `HashMap` from multiple threads without synchronisation — data corruption, lost updates.
- Mutable keys — if hashCode changes after insertion, the entry becomes unreachable (wrong bucket).
- Specifying initial capacity that's not a power of 2 — HashMap rounds up to the next power of 2.
- Not understanding that `hashCode()` alone doesn't determine the bucket — the spread function and capacity mask matter.

**Self-check question**
Two keys have hashCode() values of 16 and 32. In a HashMap with default initial capacity (16), do they collide? What about after the HashMap resizes to 32? Show the bucket index calculation for each.
