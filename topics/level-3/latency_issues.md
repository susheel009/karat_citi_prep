### Topic: Latency Issues — Level 3

> **Related:** [Level 3 — Performance Issues](./performance_issues.md) · [Level 3 — Thread Pool Size](./thread_pool_size.md) · [Level 2 — Monitoring and Alerting](../level-2/monitoring_and_alerting.md)

**Why it matters (Karat angle)**
"Your p95 latency doubled after a deployment. How do you debug it?" This is the quintessential senior production question. The answer is a systematic approach: isolate the layer, identify the cause, fix, and prevent regression.

**Core concept**

**Latency percentiles:**

| Percentile | Meaning | Why it matters |
|:----------:|---------|---------------|
| p50 (median) | Half of requests are faster | Typical user experience |
| p95 | 95% of requests are faster | "Most users" experience |
| p99 | 99% of requests are faster | Tail latency — worst 1% |
| p99.9 | 99.9% are faster | SLA-critical, edge cases |

**Latency budget breakdown (typical REST API):**
```
Total latency: 200ms
├── Network (ingress → app): 5ms
├── Spring filter chain: 2ms
├── Controller → Service: 1ms
├── DB query: 50ms
├── External REST call: 100ms
├── Serialization (JSON): 5ms
├── Network (app → client): 5ms
└── Overhead (GC, contention): ~32ms
```

**Systematic debugging approach:**

```
1. WHERE: Which layer is slow?
   → Add timing at each layer boundary
   → Distributed tracing (Zipkin/Jaeger) shows per-span latency

2. WHAT: What changed?
   → Recent deployment? Config change? Traffic spike?
   → Compare "before" and "after" metrics

3. WHY: Root cause analysis
   → DB: slow query log, EXPLAIN PLAN, missing index
   → GC: GC logs show long pauses
   → Thread pool: thread dump shows threads WAITING/BLOCKED
   → External: downstream service latency increased
   → Code: N+1 queries, unnecessary serialization, hot loop
```

**Common latency causes and fixes:**

| Cause | Symptom | Fix |
|-------|---------|-----|
| Missing DB index | Query time > 1s | Add index, verify with EXPLAIN |
| N+1 queries | Hundreds of small queries | JOIN FETCH, @EntityGraph |
| GC pauses | Periodic latency spikes | Tune GC, reduce allocation |
| Thread pool exhaustion | Requests queued, high latency | Increase pool, add bulkhead |
| Downstream slow | Specific span is slow in trace | Circuit breaker, timeout, cache |
| Lock contention | Threads in BLOCKED state | Reduce lock scope, use concurrent data structures |
| Connection pool exhaustion | "Waiting for connection" logs | Increase pool, fix leaks |
| Large payloads | Serialization time high | Paginate, compress, reduce fields |

**Adding observability:**

```java
// File: topics/level-3/LatencyObservability.java

// --- Manual timing ---
@Around("@annotation(Timed)")
public Object timeMethod(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.nanoTime();
    try {
        return pjp.proceed();
    } finally {
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        log.info("{}.{} took {}ms", pjp.getTarget().getClass().getSimpleName(),
            pjp.getSignature().getName(), elapsed);
    }
}

// --- Micrometer timer (exports to Prometheus) ---
@Service
public class PaymentService {

    private final Timer paymentTimer;

    public PaymentService(MeterRegistry registry) {
        this.paymentTimer = Timer.builder("payment.process.duration")
            .tag("type", "wire")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public PaymentResult process(PaymentRequest request) {
        return paymentTimer.record(() -> doProcess(request));
    }
}
```

**Distributed tracing (Spring Boot + Micrometer Tracing):**

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0                                    # sample 100% (use 0.1 in prod)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

```java
// File: topics/level-3/TracingDemo.java

// Trace propagation: each service adds a span
// GET /dashboard → span: "dashboard-controller" (5ms)
//   → REST call to account-service → span: "account-client" (50ms)
//   → DB query → span: "accountRepo.findById" (10ms)
//   → REST call to notification-service → span: "notif-client" (200ms) ← bottleneck!

// Zipkin UI shows the full trace with per-span timing
// Identifies: notification-service is the bottleneck (200ms)
```

**Thread dump analysis for latency:**

```bash
# Capture thread dump
jstack <pid> > threads.txt

# Look for:
# 1. Many threads in BLOCKED state → lock contention
# 2. Many threads in WAITING on a pool → pool exhausted
# 3. Many threads in TIMED_WAITING on I/O → slow downstream

# Example: thread pool exhausted
"http-nio-8080-exec-193" WAITING
   at sun.misc.Unsafe.park
   at java.util.concurrent.locks.LockSupport.park
   at java.util.concurrent.LinkedBlockingQueue.take       ← waiting for connection
   at com.zaxxer.hikari.pool.HikariPool.getConnection     ← connection pool exhausted!
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Latency debugging is systematic: identify which layer is slow (tracing), what changed (deployment diff), and why (profiling, thread dumps, GC logs). Percentiles (p50, p95, p99) measure user experience better than averages.
2. **Why/when:** p95 latency matters more than average — 5% of users see the worst experience. Averages hide tail latency. Distributed tracing (Zipkin) shows per-service latency breakdown.
3. **Example:** Trace shows 200ms on notification-service call. Thread dump shows threads blocked on HikariCP → connection pool exhausted. Fix: increase pool size, fix connection leak, or add caching.
4. **Gotcha/tradeoff:** Adding tracing adds ~1–5% overhead — sample at 10% in production. Thread dumps are free but are snapshots — take 3–5 dumps 10 seconds apart to see patterns.

**Common pitfalls**
- Optimising average latency instead of p95/p99 — average hides spikes.
- No distributed tracing — can't identify which service is slow in a microservice chain.
- Looking at application metrics when the issue is infrastructure (DNS resolution, network, disk I/O).
- Missing correlation IDs — can't trace a request across services.
- Thread dump taken only once — need multiple snapshots to see patterns.

**Self-check question**
Your service p95 went from 100ms to 2 seconds. CPU is 30%, memory is 60%, GC pauses are 5ms. Thread dump shows 180 out of 200 `http-nio` threads in `TIMED_WAITING` at `SocketInputStream.read()`. What's wrong? Where do you look next?
