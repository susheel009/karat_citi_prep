### Topic: Parallel Service Calls — Level 2

> **Related:** [Level 2 — Multithreading](./multithreading_in_java.md) · [Level 2 — Circuit Breaker](./circuit_breaker_design_pattern.md) · [Level 2 — Microservices](./microservices.md)

**Why it matters (Karat angle)**
A microservice that calls 3 downstream services sequentially wastes time. Making those calls in parallel cuts latency from sum(all) to max(slowest). Interviewers test this to see if you can use `CompletableFuture` or virtual threads effectively.

**Core concept**

| Approach | Sequential | Parallel |
|----------|:----------:|:--------:|
| Call A (200ms) + Call B (300ms) + Call C (100ms) | 600ms total | 300ms total (limited by slowest) |

**`CompletableFuture` — the standard approach:**

```java
// File: topics/level-2/ParallelCallsDemo.java

@Service
public class DashboardService {

    private final AccountClient accountClient;
    private final TransactionClient txnClient;
    private final NotificationClient notifClient;

    private final Executor executor = Executors.newFixedThreadPool(10);

    public DashboardDto getDashboard(Long userId) {
        // Launch all three calls in parallel
        CompletableFuture<AccountDto> accountFuture =
            CompletableFuture.supplyAsync(() -> accountClient.getAccount(userId), executor);

        CompletableFuture<List<TransactionDto>> txnFuture =
            CompletableFuture.supplyAsync(() -> txnClient.getRecent(userId), executor);

        CompletableFuture<List<NotificationDto>> notifFuture =
            CompletableFuture.supplyAsync(() -> notifClient.getUnread(userId), executor);

        // Wait for all to complete
        CompletableFuture.allOf(accountFuture, txnFuture, notifFuture).join();

        // Assemble result
        return new DashboardDto(
            accountFuture.join(),
            txnFuture.join(),
            notifFuture.join()
        );
    }
}
```

**With timeout and error handling:**

```java
// File: topics/level-2/ParallelWithTimeoutDemo.java

public DashboardDto getDashboardSafe(Long userId) {

    CompletableFuture<AccountDto> accountFuture =
        CompletableFuture.supplyAsync(() -> accountClient.getAccount(userId), executor)
            .orTimeout(3, TimeUnit.SECONDS)               // fail if > 3 seconds
            .exceptionally(ex -> {
                log.warn("Account call failed: {}", ex.getMessage());
                return AccountDto.empty();                 // fallback
            });

    CompletableFuture<List<TransactionDto>> txnFuture =
        CompletableFuture.supplyAsync(() -> txnClient.getRecent(userId), executor)
            .orTimeout(3, TimeUnit.SECONDS)
            .exceptionally(ex -> List.of());               // fallback: empty list

    CompletableFuture<List<NotificationDto>> notifFuture =
        CompletableFuture.supplyAsync(() -> notifClient.getUnread(userId), executor)
            .orTimeout(3, TimeUnit.SECONDS)
            .exceptionally(ex -> List.of());

    CompletableFuture.allOf(accountFuture, txnFuture, notifFuture).join();

    return new DashboardDto(
        accountFuture.join(),
        txnFuture.join(),
        notifFuture.join()
    );
}
```

**Spring's `@Async` for parallel execution:**

```java
// File: topics/level-2/AsyncParallelDemo.java

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("serviceExecutor")
    public Executor serviceExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("svc-");
        executor.initialize();
        return executor;
    }
}

@Service
public class AccountService {

    @Async("serviceExecutor")
    public CompletableFuture<AccountDto> getAccountAsync(Long userId) {
        AccountDto account = accountClient.getAccount(userId);
        return CompletableFuture.completedFuture(account);
    }
}
```

**Java 21 virtual threads — simplest approach:**

```java
// File: topics/level-2/VirtualThreadParallelDemo.java

// application.yml: spring.threads.virtual.enabled=true

public DashboardDto getDashboard(Long userId) {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var accountTask = scope.fork(() -> accountClient.getAccount(userId));
        var txnTask     = scope.fork(() -> txnClient.getRecent(userId));
        var notifTask   = scope.fork(() -> notifClient.getUnread(userId));

        scope.join();                                     // wait for all
        scope.throwIfFailed();                            // propagate first failure

        return new DashboardDto(
            accountTask.get(),
            txnTask.get(),
            notifTask.get()
        );
    }
}
// Virtual threads: lightweight, no thread pool sizing needed, blocks without wasting OS threads
```

| Approach | Thread model | Ease | Error handling |
|----------|-------------|:----:|:-------------:|
| `CompletableFuture` | Platform threads + executor | Medium | `exceptionally()`, `handle()` |
| `@Async` | Platform threads + Spring executor | Easy | Return `CompletableFuture` |
| Virtual threads (Java 21) | Virtual threads (JVM-managed) | Easy | Structured task scope |

**What to say in the interview (4-beat answer)**
1. **Definition:** Parallel service calls use `CompletableFuture.supplyAsync()` to launch multiple downstream calls concurrently, then `allOf().join()` to wait for all. Total latency = slowest call, not sum of all.
2. **Why/when:** Dashboard pages that aggregate data from multiple services. APIs that call payment + account + notification services. Any case where independent calls can overlap.
3. **Example:** Three calls at 200ms, 300ms, 100ms — sequential = 600ms, parallel = 300ms. Add `orTimeout(3, SECONDS)` and `exceptionally()` for resilience.
4. **Gotcha/tradeoff:** Thread pool sizing matters — too few threads and calls queue up; too many and you exhaust memory. Virtual threads (Java 21) eliminate this concern. Always handle individual call failures with fallbacks — don't let one failure block the entire response.

**Common pitfalls**
- Using `ForkJoinPool.commonPool()` (default) for I/O-bound work — it's sized for CPU-bound work; provide a custom executor.
- Not handling individual failures — one failing call blocks `allOf().join()` unless you use `exceptionally()`.
- Fire-and-forget without observing the future — swallowed exceptions.
- `@Async` without `@EnableAsync` — methods run synchronously; no error at compile time.
- Thread pool too small for the number of parallel calls — calls serialize.

**Self-check question**
You have 5 parallel calls with a 3-second timeout each. The thread pool has 3 threads. What's the maximum latency for all 5 calls? How do virtual threads change this?
