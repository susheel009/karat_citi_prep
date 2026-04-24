### Topic: Threads (Async, Synchronized, Critical Region) — Level 3

> **Foundation:** [Level 1 — Thread](../level-1/thread.md) · [Level 2 — Multithreading (producer-consumer, wait/notify)](../level-2/multithreading_in_java.md)
> **Related:** [Level 3 — Deadlocks](./deadlocks.md) · [Level 3 — Critical Section of Code](./critical_section_of_code.md) · [Level 3 — Thread Pool Size](./thread_pool_size.md)

**Why it matters (Karat angle)**
L2 covered producer-consumer and monitor locking. L3 covers building multi-threaded applications with `ExecutorService`, `CompletableFuture` (chaining, error handling), `synchronized` internals, and the `@Async` Spring pattern — production-level concurrency.

**Core concept**

**`ExecutorService` — thread pool management:**

```java
// File: topics/level-3/ExecutorServiceDemo.java

// --- Thread pool types ---
ExecutorService fixed    = Executors.newFixedThreadPool(10);        // fixed N threads
ExecutorService cached   = Executors.newCachedThreadPool();         // grows/shrinks on demand
ExecutorService single   = Executors.newSingleThreadExecutor();     // one thread (ordered)
ScheduledExecutorService sched = Executors.newScheduledThreadPool(5);

// --- Custom thread pool (production — always prefer this) ---
ThreadPoolExecutor custom = new ThreadPoolExecutor(
    5,                                                     // corePoolSize
    20,                                                    // maximumPoolSize
    60, TimeUnit.SECONDS,                                  // keepAliveTime for idle threads
    new LinkedBlockingQueue<>(100),                         // work queue (bounded!)
    new ThreadPoolExecutor.CallerRunsPolicy()               // rejection policy
);

// --- Submit tasks ---
Future<String> future = fixed.submit(() -> {
    Thread.sleep(1000);
    return "done";
});
String result = future.get(5, TimeUnit.SECONDS);           // blocks with timeout

// --- Shutdown ---
fixed.shutdown();                                          // stop accepting, finish pending
fixed.awaitTermination(30, TimeUnit.SECONDS);              // wait for completion
// fixed.shutdownNow();                                    // interrupt running tasks
```

**`CompletableFuture` — advanced async chaining:**

```java
// File: topics/level-3/CompletableFutureAdvanced.java

// --- Chain: fetch → transform → consume ---
CompletableFuture.supplyAsync(() -> accountClient.getAccount(userId), executor)
    .thenApply(account -> enrichWithHistory(account))      // transform (same thread or fork)
    .thenApply(account -> calculateRisk(account))
    .thenAccept(risk -> log.info("Risk: {}", risk))        // consume (no return)
    .exceptionally(ex -> {
        log.error("Pipeline failed: {}", ex.getMessage());
        return null;
    });

// --- Combine two independent futures ---
CompletableFuture<AccountDto> accF = CompletableFuture.supplyAsync(() -> getAccount(id));
CompletableFuture<List<TxnDto>> txnF = CompletableFuture.supplyAsync(() -> getTxns(id));

CompletableFuture<DashboardDto> dashboard = accF.thenCombine(txnF,
    (acc, txns) -> new DashboardDto(acc, txns));           // called when BOTH complete

// --- First completed (race) ---
CompletableFuture<String> fastest = CompletableFuture.anyOf(
    CompletableFuture.supplyAsync(() -> primaryService.call()),
    CompletableFuture.supplyAsync(() -> fallbackService.call())
).thenApply(Object::toString);

// --- Handle (success or failure in one callback) ---
CompletableFuture.supplyAsync(() -> riskyOperation())
    .handle((result, ex) -> {
        if (ex != null) return "fallback";
        return result;
    });

// --- Compose (chain of futures) ---
CompletableFuture.supplyAsync(() -> getAccountId(userId))
    .thenCompose(accId -> CompletableFuture.supplyAsync(() -> getBalance(accId)));
    // thenCompose = flatMap for futures (avoids CompletableFuture<CompletableFuture<T>>)
```

**`synchronized` internals — biased, thin, fat locks:**

```java
// File: topics/level-3/SynchronizedInternals.java

// JVM uses escalating lock levels:
// 1. BIASED LOCK:    single thread repeatedly enters → almost free (no CAS)
// 2. THIN LOCK (CAS): another thread contends → CAS spin lock
// 3. FAT LOCK (OS):   heavy contention → OS mutex (expensive context switch)

// Object header contains the lock state:
// [unused:1 | age:4 | biased:1 | tag:2]
// tag: 01 = unlocked, 00 = biased, 10 = thin lock, 11 = fat lock

// Biased locking removed in Java 15+ (-XX:+UseBiasedLocking was deprecated)

// Reentrant: same thread can enter synchronized block multiple times
synchronized (lock) {
    synchronized (lock) {                                  // same thread → reentrant, no deadlock
        // works fine — lock count incremented
    }
}
```

**Spring `@Async`:**

```java
// File: topics/level-3/SpringAsyncDemo.java

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(5);
        exec.setMaxPoolSize(20);
        exec.setQueueCapacity(50);
        exec.setRejectionPolicy(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.setThreadNamePrefix("async-");
        exec.initialize();
        return exec;
    }
}

@Service
public class NotificationService {

    @Async("taskExecutor")
    public CompletableFuture<Void> sendEmailAsync(String to, String body) {
        emailClient.send(to, body);                        // runs on taskExecutor thread
        return CompletableFuture.completedFuture(null);
    }

    @Async
    public void fireAndForget(String message) {
        // Returns void — caller can't track completion
        // Exception goes to AsyncUncaughtExceptionHandler
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `ExecutorService` manages thread pools; `CompletableFuture` chains async operations with `thenApply`, `thenCombine`, `exceptionally`. `synchronized` uses escalating locks (biased → thin → fat). Spring's `@Async` runs methods on a configured thread pool.
2. **Why/when:** Use `CompletableFuture` for parallel downstream calls with error handling and timeouts. Use `@Async` for fire-and-forget tasks (emails, audit logs). Always use a bounded thread pool with a rejection policy — unbounded pools cause OOM.
3. **Example:** `supplyAsync(getAccount).thenCombine(supplyAsync(getTxns), DashboardDto::new)` — two parallel calls, combined when both complete.
4. **Gotcha/tradeoff:** `@Async` on `this.method()` doesn't work — proxy is bypassed. `CompletableFuture` without `exceptionally()` swallows exceptions silently. `Executors.newCachedThreadPool()` is unbounded — can create thousands of threads under load.

**Common pitfalls**
- `Executors.newCachedThreadPool()` creating unlimited threads → OOM. Use fixed or custom pools.
- Not shutting down `ExecutorService` → threads live forever, app won't exit.
- `CompletableFuture` default uses `ForkJoinPool.commonPool()` — shared across app, not for I/O-bound tasks.
- `@Async` without `@EnableAsync` — methods run synchronously; no error.
- Ignoring the `Future` return value — exceptions are silently swallowed.

**Self-check question**
You submit 100 tasks to a `ThreadPoolExecutor(5, 10, 60s, LinkedBlockingQueue(20))`. At what point does the pool create new threads beyond the core 5? At what point does the rejection policy fire? How many tasks are active at maximum?
