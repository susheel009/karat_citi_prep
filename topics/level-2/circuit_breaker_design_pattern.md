### Topic: Circuit Breaker Design Pattern — Level 2

> **Related:** [Level 2 — Microservices](./microservices.md) · [Level 2 — Monitoring and Alerting](./monitoring_and_alerting.md)

**Why it matters (Karat angle)**
In a microservice architecture, one slow or failing downstream service can cascade failures across the system. Circuit breakers prevent this. Interviewers ask this to see if you've designed resilient services — not just happy-path ones.

**Core concept**

The circuit breaker is a **state machine** that wraps calls to external services:

```
CLOSED (normal) → failures exceed threshold → OPEN (fail fast)
OPEN → wait timeout expires → HALF_OPEN (test with a few requests)
HALF_OPEN → success → CLOSED / failure → OPEN
```

| State | Behaviour | Duration |
|-------|----------|----------|
| **CLOSED** | Requests pass through normally; failures are counted | Until failure threshold hit |
| **OPEN** | Requests fail immediately (fallback) — no call to downstream | configurable wait duration |
| **HALF_OPEN** | Limited requests allowed through to test recovery | Until success/failure threshold |

**Resilience4j — the Spring Boot standard:**

```java
// File: topics/level-2/CircuitBreakerDemo.java

// application.yml
// resilience4j:
//   circuitbreaker:
//     instances:
//       paymentGateway:
//         sliding-window-size: 10
//         failure-rate-threshold: 50       # % — open after 50% failures in window
//         wait-duration-in-open-state: 30s # stay open for 30 seconds
//         permitted-number-of-calls-in-half-open-state: 3
//         register-health-indicator: true  # expose in /actuator/health

@Service
public class PaymentService {

    private final PaymentGatewayClient gateway;

    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallbackPayment")
    public PaymentResult processPayment(PaymentRequest request) {
        return gateway.charge(request);                   // may fail or timeout
    }

    // Fallback — called when circuit is OPEN or call fails
    private PaymentResult fallbackPayment(PaymentRequest request, Throwable ex) {
        log.warn("Circuit breaker fallback: {}", ex.getMessage());
        return PaymentResult.queued("Payment queued for retry");
        // Options: return cached result, queue for async retry, return degraded response
    }
}
```

**Combining with retry and timeout:**

```java
// File: topics/level-2/ResilienceComboDemo.java

// Order matters: Retry → CircuitBreaker → TimeLimiter → Bulkhead (outer to inner)

@Service
public class ExchangeRateService {

    @Retry(name = "rateApi", fallbackMethod = "cachedRate")
    @CircuitBreaker(name = "rateApi", fallbackMethod = "cachedRate")
    @TimeLimiter(name = "rateApi")
    public CompletableFuture<BigDecimal> getRate(String from, String to) {
        return CompletableFuture.supplyAsync(() -> rateApi.fetchRate(from, to));
    }

    private CompletableFuture<BigDecimal> cachedRate(String from, String to, Throwable ex) {
        return CompletableFuture.completedFuture(cacheService.getLastKnownRate(from, to));
    }
}
```

```yaml
# application.yml
resilience4j:
  retry:
    instances:
      rateApi:
        max-attempts: 3
        wait-duration: 500ms
  circuitbreaker:
    instances:
      rateApi:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
  timelimiter:
    instances:
      rateApi:
        timeout-duration: 3s
```

**Bulkhead — limit concurrent calls:**

| Type | How it limits | Use case |
|------|-------------|---------|
| **Semaphore** | Max concurrent calls | Lightweight, same thread |
| **Thread pool** | Separate thread pool | Isolate slow calls from fast ones |

```java
@Bulkhead(name = "paymentGateway", type = Bulkhead.Type.SEMAPHORE)
public PaymentResult processPayment(PaymentRequest request) {
    // Max 10 concurrent calls (configured in YAML)
    // 11th call gets BulkheadFullException → fallback
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A circuit breaker wraps calls to an external service. When failures exceed a threshold, it opens (fails fast) to prevent cascading failures. After a wait period, it half-opens to test recovery.
2. **Why/when:** In microservices, a downstream service outage can cascade — every caller blocks and eventually fails. Circuit breakers fail fast and return a fallback (cached data, queued retry, degraded response).
3. **Example:** Resilience4j's `@CircuitBreaker` with `failureRateThreshold=50` opens after 50% of the last 10 calls fail. Fallback returns a cached exchange rate instead of a 500 error.
4. **Gotcha/tradeoff:** Circuit breakers hide failures — monitor them via Actuator health indicators and alerts. A permanently open circuit breaker means the downstream service is down and nobody noticed. Combine with retry (for transient failures) and bulkhead (for isolation).

**Common pitfalls**
- No fallback method — open circuit throws exception to caller instead of degrading gracefully.
- Fallback signature mismatch — must match the original method's parameters plus a `Throwable`.
- Circuit breaker around internal method calls — self-invocation bypasses the proxy (same as `@Transactional`).
- Not monitoring circuit state — open circuits go unnoticed in production.
- Retry inside circuit breaker — retries count as failures, causing the circuit to open faster. Put retry outside.

**Self-check question**
Your circuit breaker has `slidingWindowSize=10, failureRateThreshold=50, waitDurationInOpenState=30s`. You make 10 calls: 5 succeed, 5 fail. What state is the circuit in? You wait 30 seconds and make 3 more calls — 2 succeed, 1 fails. What happens?
