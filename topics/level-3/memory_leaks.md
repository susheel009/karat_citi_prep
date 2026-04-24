### Topic: Memory Leaks (Cyclic Dependencies, JVM GC Handling, Prevention) — Level 3

> **Foundation:** [Level 2 — Memory Leaks (OOM types, common causes)](../level-2/memory_leaks.md)
> **Related:** [Level 3 — Garbage Collector](./garbage_collector.md) · [Level 3 — Memory Leak Debugging](./memory_leak_debugging.md)

**Why it matters (Karat angle)**
L2 covered OOM types and common causes. L3 covers the deeper mechanics — how GC handles circular references, `WeakReference`/`SoftReference` for cache-friendly patterns, and systematic prevention strategies.

**Core concept**

**Cyclic dependencies and GC:**

```java
// File: topics/level-3/CyclicGcDemo.java

// Does Java GC handle circular references?
class A { B b; }
class B { A a; }

A objA = new A();
B objB = new B();
objA.b = objB;
objB.a = objA;                                            // circular reference: A → B → A

objA = null;
objB = null;
// Both are now unreachable from any GC root
// Java GC traces from roots → neither A nor B is reachable → BOTH collected
// Java does NOT use reference counting → cycles are NOT a problem

// Contrast with Python/Swift: reference counting needs cycle detection
// Java's tracing GC handles this automatically
```

**Reference types for memory-sensitive caching:**

| Type | Collected when | Use case |
|------|---------------|---------|
| **Strong** | Never (while reachable) | Normal variables |
| **Soft** `SoftReference<T>` | When memory is low (before OOM) | Memory-sensitive caches |
| **Weak** `WeakReference<T>` | At next GC (regardless of memory) | Canonicalizing maps, listeners |
| **Phantom** `PhantomReference<T>` | After finalization, before reclamation | Post-mortem cleanup |

```java
// File: topics/level-3/ReferenceTypesDemo.java

// --- SoftReference: cache that yields to GC pressure ---
Map<String, SoftReference<byte[]>> imageCache = new HashMap<>();

public byte[] getImage(String key) {
    SoftReference<byte[]> ref = imageCache.get(key);
    byte[] data = (ref != null) ? ref.get() : null;
    if (data == null) {
        data = loadFromDisk(key);                          // cache miss
        imageCache.put(key, new SoftReference<>(data));
    }
    return data;
}
// When heap is low, GC clears SoftReferences → prevents OOM
// But: HashMap entries (keys) still accumulate → use WeakHashMap for keys too

// --- WeakReference: auto-cleanup when no strong references ---
Map<Thread, WeakReference<RequestContext>> contextMap = new ConcurrentHashMap<>();
// When Thread is GC'd → WeakReference is cleared → entry becomes stale
// WeakHashMap does this automatically for keys

// --- WeakHashMap: keys are weakly referenced ---
WeakHashMap<Object, String> weakMap = new WeakHashMap<>();
Object key = new Object();
weakMap.put(key, "value");
key = null;                                                // key is now weakly reachable
System.gc();                                               // GC clears the weak key
System.out.println(weakMap.size());                        // 0 — entry auto-removed
```

**Systematic leak prevention:**

```java
// File: topics/level-3/LeakPreventionDemo.java

// --- 1. Always use try-with-resources ---
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    // process
}

// --- 2. Always remove ThreadLocal ---
private static final ThreadLocal<UserContext> ctx = new ThreadLocal<>();
public void process(Request req) {
    try {
        ctx.set(new UserContext(req.getUserId()));
        doWork();
    } finally {
        ctx.remove();                                      // CRITICAL in thread pools
    }
}

// --- 3. Bounded caches with eviction ---
Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(30))
    .build();

// --- 4. Unregister listeners/callbacks ---
eventBus.register(listener);
// ... later:
eventBus.unregister(listener);                             // prevent listener leak

// --- 5. Use @PreDestroy for cleanup ---
@Component
public class ResourceHolder {
    private ExecutorService executor = Executors.newFixedThreadPool(5);

    @PreDestroy
    public void shutdown() {
        executor.shutdown();                               // cleanup on bean destruction
    }
}
```

**Production leak detection:**

```java
// File: topics/level-3/LeakDetectionConfig.java

// HikariCP leak detection
// spring.datasource.hikari.leak-detection-threshold=60000  // 60 seconds

// Custom leak detection with JMX
@Component
public class MemoryLeakMonitor {

    @Scheduled(fixedRate = 60_000)
    public void checkHeapUsage() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memBean.getHeapMemoryUsage();
        double usedPct = (double) heap.getUsed() / heap.getMax() * 100;
        if (usedPct > 85) {
            log.warn("HEAP WARNING: {}% used ({}/{})",
                String.format("%.1f", usedPct),
                heap.getUsed() / 1_048_576 + "MB",
                heap.getMax() / 1_048_576 + "MB");
        }
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java GC uses tracing (not reference counting), so circular references are collected automatically. Memory leaks occur when objects are reachable but no longer needed — the GC can't help.
2. **Why/when:** `SoftReference` for memory-sensitive caches (GC clears when heap is low). `WeakReference`/`WeakHashMap` for canonicalizing maps (auto-cleanup when key is GC'd). Strong references in static collections cause the most common leaks.
3. **Example:** `WeakHashMap<Thread, Context>` — when a Thread is GC'd, the entry is automatically removed. Contrast with `HashMap` — the Thread's entry persists even after the thread dies, leaking the Context.
4. **Gotcha/tradeoff:** `SoftReference` cache entries are cleared unpredictably by GC — not suitable when you need guaranteed availability. Use Caffeine with explicit TTL and size limits instead.

**Common pitfalls**
- Assuming circular references cause leaks in Java — they don't; tracing GC handles them.
- `SoftReference` cache with `HashMap` keys — keys accumulate even after values are cleared. Use `Caffeine` instead.
- Forgetting `ThreadLocal.remove()` in thread pool environments — context leaks across requests.
- Using `finalize()` for cleanup — deprecated, unreliable, and can actually cause leaks (finalizer queue backlog).

**Self-check question**
You have a `ConcurrentHashMap<String, SoftReference<BigObject>>`. After GC, many `SoftReference.get()` return null but the map entries remain. Is this a leak? How would you clean up the stale entries?
