### Topic: Garbage Collector — Level 3

> **Foundation:** [Level 2 — Heap and Stack](../level-2/heap_and_stack.md) · [Level 2 — Memory Leaks](../level-2/memory_leaks.md)
> **Related:** [Level 3 — GC Algorithms](./gc_algorithms.md) · [Level 3 — JVM Memory Settings](./jvm_memory_settings.md)

**Why it matters (Karat angle)**
"How does garbage collection work?" is a standard L3 question. Interviewers expect you to explain reachability, generational collection, GC roots, and stop-the-world pauses — and how they affect latency in a banking application.

**Core concept**

**GC fundamentals: what is garbage?**
An object is "garbage" (eligible for collection) when it's **unreachable** — no path exists from any GC root to the object.

**GC roots — the starting points for reachability analysis:**

| GC Root | Where |
|---------|-------|
| Local variables | Current stack frames (method variables) |
| Active threads | `Thread` objects and their stacks |
| Static fields | `Class` static references |
| JNI references | Native method references |
| Synchronized monitors | Objects used in `synchronized` blocks |

```
GC Root (stack variable)
  └── AccountService
        └── AccountRepository
              └── DataSource
                    └── Connection pool
                          └── Connection objects
                          
All reachable → NOT collected.

GC Root (stack variable)
  └── (method returned, variable popped)
        └── Account object → UNREACHABLE → eligible for GC
```

**Generational collection — the heap is divided:**

```
HEAP
├── Young Generation
│   ├── Eden Space      (new objects created here)
│   ├── Survivor 0 (S0) (survived 1+ GC)
│   └── Survivor 1 (S1) (survived 1+ GC)
├── Old Generation (Tenured)
│   └── Long-lived objects (survived ~15 young GCs)
└── Metaspace (off-heap)
    └── Class metadata, method info
```

**Why generational?**
- **Weak generational hypothesis:** Most objects die young (temporary objects, method locals).
- Collecting only Eden (Minor GC) is fast — small area, mostly dead objects.
- Collecting Old Generation (Major/Full GC) is expensive — large area, more live objects.

**GC cycle — Minor GC:**
```
1. Eden fills up → Minor GC triggered
2. Mark: trace from GC roots → mark reachable objects in Young Gen
3. Copy: live objects from Eden → Survivor S0 (or S1, alternating)
4. Clear: Eden is wiped (dead objects gone instantly)
5. Objects that survive multiple Minor GCs (age threshold, default 15) → promoted to Old Gen
```

**GC cycle — Major/Full GC:**
```
1. Old Generation fills up → Major GC triggered
2. Mark: trace ALL reachable objects (Young + Old)
3. Sweep/Compact: reclaim dead objects, compact live objects
4. This is SLOW — stop-the-world pause
```

**Stop-the-world (STW) pauses:**
All application threads are paused while GC runs. Minor GC pauses: ~1-10ms. Full GC pauses: ~100ms–seconds.

```java
// File: topics/level-3/GcDemo.java

// Force GC (suggestion only — JVM may ignore)
System.gc();

// finalize() — deprecated since Java 9, removed in Java 18
// Don't use it — unpredictable, delays GC, blocks finalizer thread

// Monitor GC activity:
// java -verbose:gc -Xlog:gc*:file=gc.log MyApp
// or: jstat -gcutil <pid> 1000
```

**Object lifecycle:**

```
Created (new) → Reachable → Unreachable → Finalized → Collected
                  ↑                  ↓
                  └── reassigned ←───┘ (resurrect — don't do this)
```

**What to say in the interview (4-beat answer)**
1. **Definition:** GC automatically reclaims memory from unreachable objects. The heap is generational: Eden (new objects) → Survivor (survived minor GC) → Old Gen (long-lived). Reachability is traced from GC roots (stack, statics, active threads).
2. **Why/when:** Minor GC (Eden collection) runs frequently but is fast (~ms). Major/Full GC (Old Gen) is infrequent but causes stop-the-world pauses (~100ms+) — critical for low-latency banking services.
3. **Example:** A method creates a temporary `List<Transaction>` to process results, then returns. When the method returns, the local variable is popped from the stack → list is unreachable → collected in the next Minor GC.
4. **Gotcha/tradeoff:** Most objects die in Eden — Minor GC is cheap. But objects that leak into Old Gen (static collections, caches without TTL) trigger Full GC — long pauses. Tuning generational sizes (`-Xmn` for Young Gen) balances frequency vs pause time.

**Common pitfalls**
- Calling `System.gc()` in production — triggers Full GC with long pause; JVM manages GC better than manual calls.
- Relying on `finalize()` — deprecated, unpredictable, delays collection.
- Allocating huge objects (> half of Eden) — goes directly to Old Gen, triggers Full GC sooner.
- Not monitoring GC pauses — latency-sensitive services need GC log analysis.
- Assuming GC prevents memory leaks — GC only collects unreachable objects; leaks keep objects reachable.

**Self-check question**
You create 10 million `new Transaction()` objects per second, each used for 1ms then discarded. Are these collected in Minor GC or Major GC? What happens if your Eden space is too small?
