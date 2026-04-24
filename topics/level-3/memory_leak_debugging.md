### Topic: Memory Leak Debugging — Level 3

> **Foundation:** [Level 2 — Memory Leaks (causes, identification)](../level-2/memory_leaks.md) · [Level 3 — Memory Leaks (references, cyclic deps)](./memory_leaks.md)
> **Related:** [Level 3 — JVM Memory Settings](./jvm_memory_settings.md) · [Level 3 — Garbage Collector](./garbage_collector.md)

**Why it matters (Karat angle)**
L2/L3 memory_leaks covered causes and reference types. This topic covers the **debugging workflow** — the exact steps, tools, and commands to find and fix a memory leak in production. Interviewers ask "how would you debug an OOM in prod?"

**Core concept**

**Memory leak vs high memory usage:**

| | Memory leak | High memory usage |
|--|------------|-------------------|
| Behaviour | Heap grows continuously, never reclaims | Heap is large but stable |
| GC effect | GC runs, recovers almost nothing | GC runs, recovers memory successfully |
| Outcome | Eventually OOM | Stable (but might need more heap) |
| Fix | Find and remove the retaining reference | Tune heap size or reduce working set |

**Debugging workflow:**

```
1. DETECT: Monitoring shows heap growing over time
   → jstat -gcutil <pid> 1000 → Old Gen % increasing across Full GCs

2. CAPTURE: Take heap dump
   → jmap -dump:live,format=b,file=heap.hprof <pid>
   → or: auto on OOM with -XX:+HeapDumpOnOutOfMemoryError

3. ANALYSE: Open in MAT (Eclipse Memory Analyzer)
   → Dominator tree: what objects retain the most memory?
   → Leak suspects report: automated analysis
   → Path to GC root: why is this object not collected?

4. IDENTIFY: Find the retaining path
   → e.g., static HashMap<String, UserSession> → 500,000 entries → 2GB
   → The map is never cleared → sessions accumulate → leak

5. FIX: Remove the retaining reference
   → Add TTL eviction, use WeakHashMap, add cleanup logic

6. VERIFY: Deploy fix, monitor heap trend → Old Gen stabilises
```

**Step-by-step tool usage:**

```bash
# File: topics/level-3/MemoryDebugCommands.sh

# --- Step 1: Monitor GC in real-time ---
jstat -gcutil <pid> 1000
# Watch: O (Old Gen %) after each FGC (Full GC)
# If O% increases after every Full GC → LEAK

# --- Step 2: Capture heap dump ---
# Live heap dump (triggers GC first — only live objects)
jmap -dump:live,format=b,file=/opt/dumps/heap_$(date +%s).hprof <pid>

# Or via jcmd (preferred, newer API)
jcmd <pid> GC.heap_dump /opt/dumps/heap.hprof

# Size warning: heap dump = ~heap size. 4GB heap = 4GB file.

# --- Step 3: Class histogram (quick check) ---
jmap -histo:live <pid> | head -30
# Shows top classes by instance count and size
#  num   instances  bytes    class name
#    1:   5000000   200000000 byte[]
#    2:   2500000   60000000  java.lang.String
#    3:   1000000   48000000  com.citi.model.UserSession  ← suspicious!

# --- Step 4: Analyse in MAT ---
# Open heap.hprof in Eclipse Memory Analyzer Tool
# Run "Leak Suspects Report" → auto-detects largest retaining trees
# Use "Dominator Tree" → shows which objects prevent GC
# Use "Path to GC Roots" → shows WHY object is retained
```

**Common leak scenarios and MAT findings:**

| Leak scenario | MAT finding | Fix |
|-------------|-------------|-----|
| Static `HashMap` growing | Dominator: `HashMap` → 500K entries → GC root: `static` field | Add TTL eviction or use Caffeine cache |
| `ThreadLocal` not removed | Dominator: `Thread` → `ThreadLocalMap` → leaked values | `finally { threadLocal.remove(); }` |
| Unclosed `InputStream` | Dominator: `BufferedInputStream` → `byte[8192]` × 10K | Use try-with-resources |
| Listener not unregistered | Path to GC root: `EventBus` → `Listener` → large object graph | Unregister in `@PreDestroy` |
| Class loader leak | Metaspace OOM, thousands of generated classes | Fix dynamic class generation, limit reflection |

**Programmatic heap monitoring:**

```java
// File: topics/level-3/HeapMonitorDemo.java

@Component
public class HeapMonitor {

    @Scheduled(fixedRate = 60_000)
    public void monitor() {
        MemoryMXBean mem = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = mem.getHeapMemoryUsage();

        double usedMB = heap.getUsed() / 1_048_576.0;
        double maxMB = heap.getMax() / 1_048_576.0;
        double pct = (heap.getUsed() * 100.0) / heap.getMax();

        if (pct > 85) {
            log.warn("HEAP: {}/{} MB ({:.1f}%) — investigate potential leak", usedMB, maxMB, pct);
        }

        // Track trend: if used MB after GC keeps increasing → leak
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            log.info("GC {}: count={}, time={}ms", gc.getName(), gc.getCollectionCount(), gc.getCollectionTime());
        }
    }
}
```

**Differentiating leak types:**

| Symptom | Likely cause | Tool |
|---------|------------|------|
| `OutOfMemoryError: Java heap space` | Object leak on heap | Heap dump + MAT |
| `OutOfMemoryError: Metaspace` | Class loader leak (generated classes) | `jmap -histo` for class count |
| `OutOfMemoryError: unable to create native thread` | Too many threads | `jstack <pid>` → count threads |
| `OutOfMemoryError: Direct buffer memory` | Unclosed NIO channels | `-XX:MaxDirectMemorySize` + code review |
| OS OOM-kill (exit code 137) | Total JVM memory > container limit | Reduce `-Xmx`, add off-heap accounting |

**What to say in the interview (4-beat answer)**
1. **Definition:** Memory leak debugging follows: detect (jstat shows Old Gen growing after Full GCs) → capture (heap dump) → analyse (MAT: dominator tree, leak suspects, path to GC root) → fix (remove retaining reference) → verify (heap stabilises).
2. **Why/when:** `jstat -gcutil` is the first diagnostic. `jmap -histo:live` gives a quick class histogram. MAT's dominator tree shows which objects prevent GC. Path to GC root reveals WHY.
3. **Example:** MAT shows `static HashMap → 500K UserSession entries → 2GB`. The map is never cleared. Fix: replace with Caffeine cache (TTL, max size) or use `@SessionScope`.
4. **Gotcha/tradeoff:** Taking a heap dump pauses the JVM (seconds for large heaps). In production, use `-XX:+HeapDumpOnOutOfMemoryError` to capture automatically. For live diagnosis, use `jmap -histo` (quick) before a full heap dump (slow).

**Common pitfalls**
- Taking heap dump without `live` flag — includes garbage, file is larger, harder to analyse.
- Ignoring Metaspace OOM — not a heap issue; caused by class loader leaks or excessive reflection.
- "Adding more memory" as a fix — delays OOM but doesn't fix the leak.
- Not enabling `-XX:+HeapDumpOnOutOfMemoryError` in production — OOM happens once, no data to diagnose.
- Heap dump file too large for MAT — increase MAT's memory: `MemoryAnalyzer.ini -Xmx8g`.

**Self-check question**
`jstat -gcutil` shows: after every Full GC, Old Gen usage is 30%, 35%, 42%, 50%, 58%. Is this a leak? You take a heap dump and MAT shows `ConcurrentHashMap → 200K entries of type CacheEntry`. The map is in a `@Service` bean. What's your fix?
