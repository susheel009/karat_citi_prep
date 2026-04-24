### Topic: Volatile Keyword — Level 2

> **Related:** [Level 2 — Multithreading](./multithreading_in_java.md) · [Level 2 — Lambdas (Atomic types)](./lambdas.md)
> **Advanced:** [Level 3 — Threads (synchronized, critical region)](../level-3/threads.md) · [Level 3 — Deadlocks](../level-3/deadlocks.md)

**Why it matters (Karat angle)**
`volatile` is a subtle but critical keyword that tests your understanding of the Java Memory Model. Interviewers ask "when would you use volatile instead of synchronized?" to distinguish seniors who understand visibility vs atomicity from juniors who just use locks everywhere.

**Core concept**

**The problem: CPU caches and visibility**

Each CPU core has its own cache. Without `volatile`, a thread may read a stale value from its local cache — never seeing updates made by another thread.

```
Thread A (Core 1)        Shared Memory        Thread B (Core 2)
┌──────────┐                                  ┌──────────┐
│ flag = true │  ← written but not flushed  │ flag = false │ ← stale cache
└──────────┘                                  └──────────┘
```

**`volatile` guarantees:**
1. **Visibility:** Every read of a volatile variable goes directly to main memory (not CPU cache). Every write is immediately flushed to main memory.
2. **Ordering:** Prevents instruction reordering around volatile reads/writes (happens-before relationship).
3. **NOT atomicity:** `volatile int count; count++` is still NOT atomic (read + increment + write = 3 operations).

```java
// File: topics/level-2/VolatileDemo.java

public class ShutdownFlag {

    private volatile boolean running = true;               // volatile = visible across threads

    public void run() {
        while (running) {                                  // always reads from main memory
            // do work
        }
        System.out.println("Stopped");
    }

    public void stop() {
        running = false;                                   // immediately visible to run()
    }
}

// Without volatile:
// The JIT compiler may optimise the while(running) loop into while(true)
// because it sees that 'running' is never modified in the SAME thread.
// The loop would never exit — even after stop() is called from another thread.
```

**When to use `volatile` vs `synchronized` vs `Atomic*`:**

| | `volatile` | `synchronized` | `AtomicInteger` |
|--|-----------|---------------|-----------------|
| Visibility | ✅ | ✅ | ✅ |
| Atomicity | ❌ | ✅ | ✅ (single operation) |
| Blocking | ❌ | ✅ (lock) | ❌ (CAS) |
| Use case | Flags, status indicators | Compound operations | Counters, accumulators |

```java
// File: topics/level-2/VolatileVsAtomicDemo.java

// WRONG: volatile doesn't make ++ atomic
private volatile int count = 0;
count++;                                                   // NOT atomic: read → increment → write
// Two threads: both read 5, both write 6 → lost update

// RIGHT: use AtomicInteger for counters
private final AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();                                   // atomic CAS operation

// RIGHT: use volatile for flags
private volatile boolean shutdownRequested = false;
// Only one writer sets it to true; multiple readers check it
```

**Double-checked locking — the classic volatile use case:**

```java
// File: topics/level-2/DoubleCheckedLocking.java

public class Singleton {
    private static volatile Singleton instance;            // volatile prevents partial construction

    public static Singleton getInstance() {
        if (instance == null) {                            // first check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {                    // second check (with lock)
                    instance = new Singleton();
                    // Without volatile, another thread may see a partially constructed object
                    // because the JVM can reorder: allocate memory → assign reference → call constructor
                }
            }
        }
        return instance;
    }
}
// Modern alternative: use an enum Singleton or static holder pattern — simpler, safer.
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `volatile` ensures visibility — reads/writes go to main memory, not CPU cache. It prevents stale reads across threads. It does NOT provide atomicity — `count++` is still a race condition.
2. **Why/when:** Use for simple flags (shutdown, status) read by many threads and written by one. For counters, use `AtomicInteger`. For compound operations, use `synchronized`.
3. **Example:** A `volatile boolean running` flag — one thread sets it to `false`, all reader threads see the change immediately. Without `volatile`, the JIT may cache the value and the loop never exits.
4. **Gotcha/tradeoff:** `volatile` is lighter than `synchronized` (no lock, no blocking) but only works for single read/write operations. `count++` on a volatile field is NOT safe — it's three operations (read, increment, write).

**Common pitfalls**
- Using `volatile` for counters — `volatile int count; count++` is NOT atomic.
- Assuming `volatile` replaces `synchronized` — it doesn't for compound operations.
- Not using `volatile` on a flag shared between threads — the JIT may cache the value, causing the reader to never see updates.
- Double-checked locking without `volatile` — allows partially constructed object to be visible.

**Self-check question**
You have `volatile long x = 0;` and two threads: Thread A writes `x = Long.MAX_VALUE`, Thread B reads `x`. On a 32-bit JVM, is the read guaranteed to see either `0` or `Long.MAX_VALUE` — or could it see a torn value (high 32 bits from one write, low 32 from another)?
