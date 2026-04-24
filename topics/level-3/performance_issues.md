### Topic: Performance Issues — Level 3

> **Related:** [Level 3 — Latency Issues](./latency_issues.md) · [Level 3 — Memory Leak Debugging](./memory_leak_debugging.md) · [Level 2 — Monitoring and Alerting](../level-2/monitoring_and_alerting.md) · [Level 3 — GC Algorithms](./gc_algorithms.md)

**Why it matters (Karat angle)**
"How do you identify and resolve performance issues?" is an open-ended senior question. The answer isn't just "add more RAM" — it's a systematic approach: metrics → profiling → root cause → fix. Citi interviewers want to hear about CPU profiling, N+1 queries, thread contention, and GC tuning.

**Core concept**

**Performance issue categories:**

| Category | Symptoms | Common causes |
|----------|---------|---------------|
| **CPU** | High CPU %, slow response | Inefficient algorithms, tight loops, regex backtracking |
| **Memory** | Growing heap, GC pauses, OOM | Memory leaks, large caches, object churn |
| **I/O** | Slow queries, network timeouts | N+1 queries, missing indexes, unbuffered I/O |
| **Concurrency** | Thread starvation, deadlocks | Lock contention, undersized thread pools |
| **GC** | Long pauses, high GC frequency | Object churn, wrong GC algorithm, too-small heap |

**Systematic diagnosis:**

```
1. OBSERVE: metrics (Grafana, Actuator)
   → Response time increasing? Error rate up? CPU/memory spiking?

2. NARROW: which layer?
   → Is it the app, DB, network, or downstream service?
   → Add timing logs at each layer boundary

3. PROFILE: pinpoint the bottleneck
   → CPU: async-profiler, JFR (Java Flight Recorder)
   → Memory: heap dump + MAT
   → DB: slow query log, EXPLAIN PLAN
   → Threads: thread dump, jstack

4. FIX: address root cause
5. VERIFY: confirm fix with same metrics
```

**Top performance anti-patterns and fixes:**

```java
// File: topics/level-3/PerformanceAntiPatterns.java

// --- 1. N+1 Query Problem ---
// WRONG: loads 100 accounts, then 100 separate queries for transactions
List<Account> accounts = accountRepo.findAll();           // 1 query
for (Account a : accounts) {
    List<Transaction> txns = a.getTransactions();          // N queries (lazy loading!)
}

// FIX: @EntityGraph or JOIN FETCH
@Query("SELECT a FROM Account a JOIN FETCH a.transactions WHERE a.status = :status")
List<Account> findByStatusWithTransactions(@Param("status") String status); // 1 query

// --- 2. String concatenation in loops ---
// WRONG: O(n²) — creates n String objects
String result = "";
for (String s : items) result += s;                       // immutable → copy every iteration

// FIX: StringBuilder
StringBuilder sb = new StringBuilder();
for (String s : items) sb.append(s);                      // O(n), one buffer

// --- 3. Blocking the event loop / main thread ---
// WRONG: synchronous HTTP call inside a request handler
@GetMapping("/dashboard")
public DashboardDto getDashboard() {
    AccountDto acc = restTemplate.getForObject("/accounts/me", AccountDto.class); // blocks
    List<TxnDto> txns = restTemplate.getForObject("/transactions", List.class);   // blocks
    return new DashboardDto(acc, txns);
}
// Total: 200ms + 300ms = 500ms

// FIX: parallel calls
CompletableFuture<AccountDto> accF = CompletableFuture.supplyAsync(() -> accountClient.get());
CompletableFuture<List<TxnDto>> txnF = CompletableFuture.supplyAsync(() -> txnClient.get());
CompletableFuture.allOf(accF, txnF).join();               // Total: max(200, 300) = 300ms

// --- 4. Unbounded cache ---
// WRONG: cache that never evicts
Map<String, Object> cache = new HashMap<>();
cache.put(key, value);                                     // grows until OOM

// FIX: bounded cache with TTL
Caffeine.newBuilder().maximumSize(10_000).expireAfterWrite(Duration.ofMinutes(10)).build();

// --- 5. Autoboxing in hot loops ---
// WRONG: creates millions of Integer wrapper objects
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) sum += i;            // autoboxing on every iteration

// FIX: use primitives
long sum = 0L;
for (long i = 0; i < 1_000_000; i++) sum += i;
```

**Profiling tools:**

| Tool | What it profiles | How |
|------|-----------------|-----|
| `jstack <pid>` | Thread dump — shows what each thread is doing | Snapshot |
| `jmap -heap <pid>` | Heap summary | Snapshot |
| `jstat -gcutil <pid> 1000` | GC stats every second | Continuous |
| **JFR** (Java Flight Recorder) | CPU, memory, I/O, locks — low overhead | `java -XX:StartFlightRecording` |
| **async-profiler** | CPU flame graphs — shows hot methods | Attach to running JVM |
| **MAT** (Eclipse Memory Analyzer) | Analyse heap dumps for leaks | Open .hprof file |
| **Grafana + Prometheus** | Metrics over time | Dashboard |

**What to say in the interview (4-beat answer)**
1. **Definition:** Performance issues fall into CPU (algorithms), Memory (leaks/GC), I/O (queries/network), and Concurrency (locks/pool sizing). Diagnosis is systematic: observe → narrow → profile → fix → verify.
2. **Why/when:** Use JFR and async-profiler for CPU profiling. Use heap dumps + MAT for memory leaks. Use EXPLAIN PLAN for slow queries. Use thread dumps for lock contention and deadlocks.
3. **Example:** N+1 query: 100 lazy-loaded associations = 101 SQL queries. Fix: `JOIN FETCH` or `@EntityGraph` reduces to 1 query. Autoboxing in a hot loop: `Long sum += i` creates 1M wrapper objects. Fix: use `long`.
4. **Gotcha/tradeoff:** Premature optimisation is the root of all evil — profile first, optimise the actual bottleneck. A 10ms method called once doesn't need optimisation; a 1ms method called 10,000 times does.

**Common pitfalls**
- Optimising without profiling — fixing the wrong thing.
- N+1 queries hidden by lazy loading — only visible under load or with SQL logging.
- Increasing heap (`-Xmx`) to "fix" a memory leak — delays OOM, doesn't fix it.
- Thread pool too small for I/O-bound work — threads block waiting for responses.

**Self-check question**
Your service's p95 latency jumped from 200ms to 2 seconds after a deployment. The CPU is normal. The database is normal. What's your debugging approach, and what are the top 3 things you'd check?
