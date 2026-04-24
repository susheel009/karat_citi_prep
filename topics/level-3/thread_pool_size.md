### Topic: Thread Pool Size — Level 3

> **Related:** [Level 3 — Threads (ExecutorService)](./threads.md) · [Level 3 — Performance Issues](./performance_issues.md) · [Level 2 — Custom Connection Pool](../level-2/custom_connection_pool.md)

**Why it matters (Karat angle)**
"How do you size a thread pool?" is a senior-level question. Too few threads = underutilised CPU. Too many = context-switch overhead, OOM. The answer depends on whether the work is CPU-bound or I/O-bound.

**Core concept**

**The two formulas:**

| Workload | Formula | Example (8 cores) |
|----------|---------|:------------------:|
| **CPU-bound** | `cores + 1` | 9 threads |
| **I/O-bound** | `cores × (1 + wait/compute)` | 8 × (1 + 10) = 88 threads |
| **Mixed** | Separate pools for CPU and I/O | CPU: 9, I/O: 88 |

**Why these formulas work:**
- **CPU-bound:** Each thread uses a full core. More threads than cores = context switching overhead. `+1` covers the case when one thread pauses (page fault, GC).
- **I/O-bound:** Threads spend most time waiting (network, DB). While one thread waits, another can use the core. `wait/compute` ratio determines how many threads can share a core.

**Practical examples:**

```java
// File: topics/level-3/ThreadPoolSizingDemo.java

int cores = Runtime.getRuntime().availableProcessors();    // e.g., 8

// --- CPU-bound: image processing, encryption, data transformation ---
ExecutorService cpuPool = Executors.newFixedThreadPool(cores + 1);

// --- I/O-bound: REST calls, DB queries, file I/O ---
// If average REST call: 200ms total, 10ms compute, 190ms waiting
// wait/compute = 190/10 = 19
// Optimal threads = 8 × (1 + 19) = 160
ExecutorService ioPool = Executors.newFixedThreadPool(cores * 20);

// --- Production: use ThreadPoolExecutor for control ---
ThreadPoolExecutor production = new ThreadPoolExecutor(
    10,                                                    // corePoolSize — always alive
    50,                                                    // maxPoolSize — grows under load
    60, TimeUnit.SECONDS,                                  // idle timeout for threads > core
    new LinkedBlockingQueue<>(200),                         // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy()               // rejection: caller thread runs it
);
```

**Rejection policies (when queue is full AND max threads reached):**

| Policy | What happens |
|--------|-------------|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Caller thread executes the task (backpressure) |
| `DiscardPolicy` | Silently drops the task |
| `DiscardOldestPolicy` | Drops the oldest queued task, submits new one |

**ThreadPoolExecutor behaviour:**
```
Submit task →
  1. Thread count < corePoolSize? → Create new thread
  2. Queue not full? → Add to queue
  3. Thread count < maxPoolSize? → Create new thread
  4. Queue full AND maxPoolSize reached → Rejection policy
```

```java
// File: topics/level-3/ThreadPoolBehaviourDemo.java

// Example: core=5, max=10, queue=20
// Task 1-5:   create 5 core threads (threads running: 5, queued: 0)
// Task 6-25:  queue fills up to 20 (threads running: 5, queued: 20)
// Task 26-30: create 5 more threads (threads running: 10, queued: 20)
// Task 31+:   REJECTED — CallerRunsPolicy → caller thread executes
```

**Spring Boot `@Async` thread pool:**

```java
// File: topics/level-3/SpringThreadPoolConfig.java

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-");
        executor.setRejectionPolicy(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) ->
            log.error("Async exception in {}: {}", method.getName(), throwable.getMessage());
    }
}
```

**Virtual threads (Java 21) — sizing is irrelevant:**

```java
// File: topics/level-3/VirtualThreadPoolDemo.java

// Virtual threads: JVM manages scheduling — no pool sizing needed
ExecutorService vThreadPool = Executors.newVirtualThreadPerTaskExecutor();
// Creates a new virtual thread per task — millions of concurrent tasks
// OS threads are shared transparently
// Perfect for I/O-bound: 10,000 concurrent HTTP calls, each on its own virtual thread
```

**What to say in the interview (4-beat answer)**
1. **Definition:** CPU-bound: `cores + 1` threads. I/O-bound: `cores × (1 + wait/compute)`. Use `ThreadPoolExecutor` with bounded queue and explicit rejection policy — never `newCachedThreadPool()` in production.
2. **Why/when:** Wrong sizing kills performance. Too few threads: CPU idle during I/O waits. Too many: context-switch overhead, memory waste (~1MB stack per thread). Separate pools for CPU and I/O workloads.
3. **Example:** 8 cores, REST calls with 200ms total/10ms compute = `8 × 20 = 160` threads. For CPU-intensive image processing on the same server: separate pool of `8 + 1 = 9` threads.
4. **Gotcha/tradeoff:** `Executors.newCachedThreadPool()` has no upper bound — under load, it can create thousands of threads → OOM. Java 21 virtual threads eliminate the sizing problem for I/O-bound workloads.

**Common pitfalls**
- `newCachedThreadPool()` in production — unbounded thread creation → OOM.
- Unbounded `LinkedBlockingQueue` — tasks queue forever, memory grows, latency spikes.
- Same pool for CPU and I/O work — I/O tasks starve CPU tasks.
- Not monitoring pool metrics (active, queued, rejected) — silent saturation.
- Sizing I/O pools like CPU pools (`cores + 1`) — threads sit idle while waiting for I/O.

**Self-check question**
You have a service with 4 cores. Each request makes 2 DB calls (50ms each) and 1 REST call (200ms), with 5ms of CPU work between calls. What's the optimal thread pool size? What happens if you set it to 5?
