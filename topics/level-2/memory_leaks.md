### Topic: Memory Leaks — Level 2

> **Advanced:** [Level 3 — Memory Leaks (cyclic deps, JVM GC handling, prevention)](../level-3/memory_leaks.md) · [Level 3 — Memory Leak Debugging](../level-3/memory_leak_debugging.md)
> **Related:** [Level 2 — Heap and Stack](./heap_and_stack.md) · [Level 3 — Garbage Collector](../level-3/garbage_collector.md)

**Why it matters (Karat angle)**
Memory leaks in banking services cause OOM crashes during peak hours — the worst time. L2 covers what OOM is, common Java causes, and basic identification. L3 goes deeper into cyclic deps, GC internals, and heap dump analysis.

**Core concept**

A **memory leak** in Java means objects are no longer needed but still referenced — GC can't collect them. Memory grows until `OutOfMemoryError`.

**Types of OOM:**

| Error | Cause | Fix |
|-------|-------|-----|
| `OutOfMemoryError: Java heap space` | Too many objects retained | Fix leak; increase `-Xmx` |
| `OutOfMemoryError: Metaspace` | Too many loaded classes (classloader leak) | Fix classloader leak; `-XX:MaxMetaspaceSize` |
| `OutOfMemoryError: GC overhead limit exceeded` | GC consuming >98% time, recovering <2% heap | Fix leak; heap is nearly full of live objects |
| `OutOfMemoryError: unable to create native thread` | Too many threads (OS limit) | Reduce thread count; `-Xss` to shrink stack |

**Common causes of memory leaks in Java:**

| Cause | Example | Fix |
|-------|---------|-----|
| **Static collections** | `static List<Event> events = new ArrayList<>()` that grows forever | Use bounded collections or eviction |
| **Unclosed resources** | `Connection`, `InputStream` not closed | try-with-resources |
| **Listener/callback not removed** | Register observer but never unregister | `removeListener()` on cleanup |
| **Thread-local variables** | `ThreadLocal<BigObject>` in thread pool — never `remove()` | Always call `threadLocal.remove()` in finally |
| **String.intern() abuse** | Interning millions of unique strings | Don't intern user data |
| **HashMap with bad keys** | Custom key without `hashCode`/`equals` — entries pile up | Implement equals/hashCode |
| **Inner class holding outer ref** | Non-static inner class retains outer instance | Use static inner class |

**Working code example — common leak patterns:**

```java
// File: topics/level-2/MemoryLeakDemo.java

// --- Leak 1: Static collection grows forever ---
public class EventLogger {
    private static final List<String> logs = new ArrayList<>();  // never cleared!

    public void log(String event) {
        logs.add(event);                                         // grows until OOM
    }
    // Fix: use bounded queue, periodic cleanup, or external logging (ELK)
}

// --- Leak 2: ThreadLocal not cleaned ---
public class RequestContext {
    private static final ThreadLocal<Map<String, Object>> ctx = new ThreadLocal<>();

    public void init() { ctx.set(new HashMap<>()); }
    public void set(String key, Object val) { ctx.get().put(key, val); }

    // MISSING: ctx.remove() — in a thread pool, the Thread is reused,
    // so the ThreadLocal value persists to the next request!

    // Fix:
    public void cleanup() { ctx.remove(); }
    // Call cleanup() in a servlet filter's finally block or @PreDestroy
}

// --- Leak 3: Unclosed connection ---
public class AccountDao {
    public Account findById(Long id) throws SQLException {
        Connection conn = dataSource.getConnection();
        PreparedStatement ps = conn.prepareStatement("SELECT * FROM accounts WHERE id = ?");
        ps.setLong(1, id);
        ResultSet rs = ps.executeQuery();
        // ... process result
        return account;
        // BUG: conn never closed! Connection pool exhausted → eventual OOM
    }

    // Fix: try-with-resources
    public Account findByIdFixed(Long id) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement("SELECT * FROM accounts WHERE id = ?")) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                // ... process result
                return account;
            }
        }
    }
}
```

**Basic identification — how to spot a leak:**

| Tool | What it shows | Command |
|------|-------------|---------|
| **JVM flags** | GC activity | `-verbose:gc -Xlog:gc*` |
| **jmap** | Heap summary | `jmap -heap <pid>` |
| **jstat** | GC stats over time | `jstat -gcutil <pid> 1000` |
| **VisualVM** | Heap graph, object count | GUI tool, connect to process |
| **Heap dump** | Full object graph | `jmap -dump:format=b,file=heap.hprof <pid>` |
| **MAT (Eclipse)** | Analyse heap dump | Open `.hprof`, find retained objects |

**Quick diagnosis flow:**
```
1. App slowing down → check heap usage: jstat -gcutil <pid> 1000
2. Heap growing over time → take heap dump: jmap -dump:format=b,file=heap.hprof <pid>
3. Open in MAT → Dominator Tree → find objects retaining the most memory
4. Trace the reference chain back to root → that's your leak
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A memory leak in Java means objects are still referenced (so GC can't collect them) but no longer needed. Heap grows until `OutOfMemoryError`.
2. **Why/when:** Common causes: static collections that grow forever, ThreadLocal not removed in thread pools, unclosed resources, listener/callback registration without cleanup.
3. **Example:** `static List<Event> events` that's appended to on every request but never cleared. After 1M requests, it holds 1M events. Fix: bounded `LinkedBlockingQueue` or external log sink.
4. **Gotcha/tradeoff:** `ThreadLocal` in a thread pool is a subtle leak — the Thread is reused, so the value persists across requests. Always call `threadLocal.remove()` in a finally block or servlet filter.

**Common pitfalls**
- Assuming GC prevents all leaks — GC only collects unreachable objects; leaked objects are still reachable.
- Increasing `-Xmx` without fixing the leak — delays the crash, doesn't fix it.
- Using `finalize()` for cleanup — deprecated, unreliable, can delay GC. Use try-with-resources.
- Storing large objects in HTTP session — sessions live for 30 min; 10K concurrent users × 10 MB = 100 GB.
- Not monitoring heap in production — leaks are invisible until OOM crashes the service.

**Self-check question**
Your service runs fine for 2 days, then crashes with `OutOfMemoryError: Java heap space`. Restarting fixes it for another 2 days. What does this pattern tell you? What's your first diagnostic step?
