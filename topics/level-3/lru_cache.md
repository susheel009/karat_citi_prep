### Topic: LRU Cache Implementation — Level 3

> **Related:** [Level 2 — Caching](../level-2/caching.md) · [Level 2 — Collections](../level-2/collections.md) · [Level 3 — Binary Tree](./binary_tree.md)

**Why it matters (Karat angle)**
"Implement an LRU cache" is a classic interview coding question (Leetcode 146). It tests your ability to design a data structure with O(1) get and put using a HashMap + doubly linked list. Citi interviewers ask this to test algorithmic design skills.

**Core concept**

**LRU (Least Recently Used)** cache evicts the item that hasn't been accessed for the longest time when the cache is full.

**Required operations — both O(1):**
| Operation | Action |
|-----------|--------|
| `get(key)` | Return value and mark as most recently used |
| `put(key, value)` | Insert/update and mark as most recently used; evict LRU if full |

**Data structure: HashMap + Doubly Linked List:**
```
HashMap<Key, Node>          Doubly Linked List (MRU ↔ LRU)
┌──────────────┐            HEAD ↔ Node(A) ↔ Node(B) ↔ Node(C) ↔ TAIL
│ "A" → Node(A)│                   (MRU)                  (LRU)
│ "B" → Node(B)│
│ "C" → Node(C)│
└──────────────┘

get("B"): HashMap → Node(B) → move to front → O(1)
put("D"): cache full → remove TAIL.prev (LRU) → add Node(D) after HEAD → O(1)
```

**Full implementation:**

```java
// File: topics/level-3/LRUCache.java

public class LRUCache<K, V> {

    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;                        // sentinel — MRU side
    private final Node<K, V> tail;                        // sentinel — LRU side

    static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;

        Node() {}                                         // sentinel constructor
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>();
        this.tail = new Node<>();
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;                    // cache miss
        moveToFront(node);                                // mark as recently used
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;                       // update value
            moveToFront(existing);                        // mark as recently used
            return;
        }

        if (map.size() >= capacity) {
            Node<K, V> lru = tail.prev;                   // least recently used
            remove(lru);
            map.remove(lru.key);
        }

        Node<K, V> newNode = new Node<>(key, value);
        addToFront(newNode);
        map.put(key, newNode);
    }

    private void addToFront(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToFront(Node<K, V> node) {
        remove(node);
        addToFront(node);
    }

    public int size() { return map.size(); }

    // --- Demo ---
    public static void main(String[] args) {
        LRUCache<String, Integer> cache = new LRUCache<>(3);
        cache.put("A", 1);                                // [A]
        cache.put("B", 2);                                // [B, A]
        cache.put("C", 3);                                // [C, B, A]
        cache.get("A");                                   // [A, C, B] — A moved to front
        cache.put("D", 4);                                // [D, A, C] — B evicted (LRU)
        System.out.println(cache.get("B"));               // null — evicted
        System.out.println(cache.get("C"));               // 3 — still in cache
    }
}
```

**Java's built-in alternative: `LinkedHashMap`**

```java
// File: topics/level-3/LinkedHashMapLRU.java

// LinkedHashMap with accessOrder=true is a built-in LRU cache
public class SimpleLRU<K, V> extends LinkedHashMap<K, V> {

    private final int capacity;

    public SimpleLRU(int capacity) {
        super(capacity, 0.75f, true);                     // accessOrder = true (LRU order)
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;                          // evict when over capacity
    }
}

// Usage:
SimpleLRU<String, Integer> lru = new SimpleLRU<>(3);
lru.put("A", 1); lru.put("B", 2); lru.put("C", 3);
lru.get("A");                                             // A is most recently used
lru.put("D", 4);                                          // B evicted (LRU)
```

**Thread-safe LRU cache:**

```java
// File: topics/level-3/ConcurrentLRU.java

// Option 1: Wrap with Collections.synchronizedMap (simple but coarse)
Map<String, Integer> syncLru = Collections.synchronizedMap(new SimpleLRU<>(1000));

// Option 2: Use Caffeine (production-grade)
Cache<String, Integer> caffeineCache = Caffeine.newBuilder()
    .maximumSize(1000)                                    // LRU eviction
    .expireAfterAccess(Duration.ofMinutes(10))
    .recordStats()
    .build();
```

**What to say in the interview (4-beat answer)**
1. **Definition:** An LRU cache evicts the least recently accessed item when full. Implemented with a HashMap (O(1) lookup) + Doubly Linked List (O(1) move-to-front, O(1) evict-from-tail).
2. **Why/when:** Classic interview question (Leetcode 146). In production, use Caffeine or Guava Cache. The custom implementation tests your ability to design composite data structures.
3. **Example:** `get("A")` → HashMap finds the node → move to front of list (mark as recently used). `put("D")` when full → remove `tail.prev` (LRU) from list and map → add new node at front.
4. **Gotcha/tradeoff:** `LinkedHashMap` with `accessOrder=true` is a built-in LRU — simpler but not thread-safe. For production, use Caffeine (concurrent, bounded, TTL, stats).

**Common pitfalls**
- Forgetting sentinel head/tail nodes — boundary conditions cause NPE.
- Not removing from HashMap when evicting from linked list — stale entries pile up.
- Not moving to front on `get()` — access pattern doesn't update recency.
- Thread-safety: raw `HashMap` + linked list is not thread-safe — use `Caffeine` or synchronise.

**Self-check question**
You have an LRU cache with capacity 3. Operations: `put(1,1), put(2,2), put(3,3), get(1), put(4,4), get(2)`. After all operations, which keys are in the cache and in what order (MRU to LRU)?
