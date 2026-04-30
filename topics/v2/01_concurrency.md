# 01 — Concurrency & Threading

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## Rapid-Fire Q&A (drill these aloud, 30 sec each)

### Q1: What does `synchronized` do and what does it lock?
**A:** Acquires the intrinsic monitor lock on an object. Instance method → locks `this`. Static method → locks the `Class` object. Block → locks the specified object. It's reentrant — same thread can re-acquire without deadlocking.

### Q2: What does `volatile` guarantee? What doesn't it guarantee?
**A:** Guarantees **visibility** — writes by one thread are immediately visible to others. Establishes a happens-before edge. Does **not** guarantee atomicity. `volatile int x; x++` is a read-modify-write — still a race condition.

### Q3: Why is `volatile int x; x++` still a race condition?
**A:** `x++` is three operations: read x, add 1, write x. Another thread can read between the read and write, causing a lost update. `volatile` only guarantees each individual read/write is visible — not that the compound operation is atomic.

### Q4: What does `AtomicInteger.incrementAndGet()` do internally?
**A:** Uses a CPU-level CAS (compare-and-swap) loop. Read current value, compute new value, atomically write only if the value hasn't changed since the read. If it changed, retry. Lock-free — no thread ever blocks.

### Q5: `synchronized` vs `ReentrantLock` — when use which?
**A:** `ReentrantLock` adds: `tryLock(timeout)`, interruptible acquisition, fair ordering, multiple `Condition` objects. Cost: you must `unlock()` in a `finally` block — `synchronized` auto-unlocks on exception. Use `synchronized` for simple cases; `ReentrantLock` when you need timeout or fairness.

### Q6: What's a deadlock? How do you prevent one?
**A:** Two+ threads each hold a lock the other needs. Prevention: (1) always acquire locks in the same global order, (2) use `tryLock` with timeout, (3) minimize lock scope, (4) avoid nested locking.

### Q7: Race condition vs livelock vs starvation?
**A:** Race condition: outcome depends on thread interleaving — data corruption. Livelock: threads aren't blocked but keep yielding to each other, making no progress. Starvation: a thread can't get the lock because others keep grabbing it — fix with fair locking.

### Q8: What's the Java Memory Model? Define happens-before.
**A:** The JMM defines visibility and ordering guarantees between threads. If action A happens-before action B, then B sees A's effects. Established by: program order within a thread, `synchronized` unlock → lock on same monitor, `volatile` write → read, `Thread.start()` → actions in started thread, thread actions → `join()` return.

### Q9: `Future` vs `CompletableFuture`?
**A:** `Future` is poll/block-based — `get()` blocks the caller. `CompletableFuture` is composable: `supplyAsync → thenApply → thenCompose → exceptionally`. Supports chaining, combining (`thenCombine`, `allOf`, `anyOf`), and non-blocking callbacks.

### Q10: What does `ExecutorService.shutdown()` do vs `shutdownNow()`?
**A:** `shutdown()` stops accepting new tasks, lets queued tasks finish. `shutdownNow()` interrupts running tasks and returns the queue of unstarted tasks.

### Q11: `Runnable` vs `Callable`?
**A:** `Runnable.run()` returns `void`, can't throw checked exceptions. `Callable.call()` returns a value `T` and can throw `Exception`. Submit a `Callable` to get a `Future<T>`.

### Q12: `CountDownLatch` vs `CyclicBarrier` vs `Semaphore`?
**A:** `CountDownLatch`: one-shot — N threads count down, waiting threads proceed when count hits 0. `CyclicBarrier`: reusable — N threads wait at a barrier point, all proceed together. `Semaphore`: controls concurrent access to N permits — acquire/release. Use for rate limiting, connection pools.

### Q13: What's `wait/notify` vs `Condition`?
**A:** `wait/notify` requires `synchronized` on the same monitor — one condition per lock. `Condition` (from `ReentrantLock`) allows multiple conditions per lock. E.g., a bounded queue can have `notFull` and `notEmpty` conditions on the same lock. Always call `wait` in a `while` loop (spurious wakeups).

### Q14: What's `ReadWriteLock`?
**A:** Allows multiple concurrent readers OR one exclusive writer. Use when reads vastly outnumber writes. `StampedLock` (Java 8+) adds optimistic reads for even better read performance.

### Q15: What's `ThreadLocal`?
**A:** Per-thread storage — each thread gets its own copy. Common for request context, database connections, date formatters. **Must** remove in `finally` in thread pool environments — otherwise context leaks between requests.

### Q16: What's `ConcurrentHashMap.computeIfAbsent` and why is it important?
**A:** Atomic compound operation: check if key exists, compute and insert if absent — all in one atomic step. Replaces the racy `if (!map.containsKey(k)) map.put(k, v)` check-then-act pattern.

### Q17: What's `LongAdder` and when is it better than `AtomicLong`?
**A:** Shards the counter across cells to reduce CAS contention. Under high contention, `LongAdder` is significantly faster than `AtomicLong` because threads update different cells. Trade-off: `sum()` is eventually consistent — okay for metrics, not for exact counts.

### Q18: What's a `ForkJoinPool`?
**A:** Work-stealing thread pool. Divide tasks recursively (`fork`), combine results (`join`). Backing pool for parallel streams and `CompletableFuture.supplyAsync()` (common pool). Don't use for I/O — it's sized for CPU-bound work (`Runtime.availableProcessors()`).

---

## Key Code Patterns

### Double-Checked Locking Singleton
```java
public class Config {
    private static volatile Config instance;
    public static Config getInstance() {
        if (instance == null) {                    // 1st check — no lock
            synchronized (Config.class) {
                if (instance == null) {             // 2nd check — under lock
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
// volatile prevents instruction reordering — without it,
// another thread could see a partially constructed Config
```

### Thread-safe counter
```java
private final AtomicLong counter = new AtomicLong(0);
public long increment() { return counter.incrementAndGet(); } // CAS loop, lock-free
```

### CompletableFuture chain
```java
CompletableFuture.supplyAsync(() -> fetchUser(id), executor)
    .thenApply(user -> user.getEmail())
    .thenCompose(email -> sendEmailAsync(email))    // flatMap
    .thenCombine(fetchPrefs(id), (result, prefs) -> merge(result, prefs))
    .exceptionally(ex -> { log.error("Failed", ex); return fallback; });
```

### Producer-Consumer with BlockingQueue
```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
// Producer
queue.put(task);           // blocks if full
// Consumer
Task t = queue.take();     // blocks if empty
```

---

## Can you answer these cold?

- [ ] Explain `synchronized` — what it locks, why it's reentrant
- [ ] `volatile` guarantees vs limitations — give the `x++` example
- [ ] CAS — what it is, where Java uses it
- [ ] Four deadlock conditions and how to break each one
- [ ] `Future` vs `CompletableFuture` — write a chain from memory
- [ ] `ExecutorService` factory methods — when each is appropriate
- [ ] `wait/notify` — why `while` not `if` (spurious wakeups)
- [ ] Thread pool sizing: CPU-bound = cores+1, I/O-bound = cores × (1 + wait/compute)

[← Back to Index](./00_INDEX.md)
