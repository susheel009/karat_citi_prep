### Topic: Monitoring and Alerting — Level 2

> **Related:** [Level 2 — Spring Boot (Actuator)](./spring_boot.md) · [Level 2 — Microservices](./microservices.md)
> **Advanced:** [Level 3 — Performance Issues](../level-3/performance_issues.md) · [Level 3 — Latency Issues](../level-3/latency_issues.md)

**Why it matters (Karat angle)**
Production services that don't expose health and metrics are blind. Interviewers ask this to see if you've operated services in production — not just built them. Custom health endpoints and alerting are standard senior-dev responsibilities.

**Core concept**

**Spring Boot Actuator** provides production-ready endpoints:

| Endpoint | Path | Purpose |
|----------|------|---------|
| `health` | `/actuator/health` | Liveness/readiness probes (Kubernetes) |
| `metrics` | `/actuator/metrics` | JVM, HTTP, DB, custom metrics |
| `info` | `/actuator/info` | Build version, git commit |
| `loggers` | `/actuator/loggers` | View/change log levels at runtime |
| `threaddump` | `/actuator/threaddump` | All thread stack traces |
| `heapdump` | `/actuator/heapdump` | Download heap dump (.hprof) |

**Custom health indicator:**

```java
// File: topics/level-2/CustomHealthIndicator.java

@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient gateway;

    @Override
    public Health health() {
        try {
            boolean reachable = gateway.ping();
            if (reachable) {
                return Health.up()
                    .withDetail("gateway", "reachable")
                    .withDetail("latency", gateway.getLatency() + "ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("gateway", "unreachable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}

// Response: GET /actuator/health
// {
//   "status": "UP",
//   "components": {
//     "db": { "status": "UP" },
//     "diskSpace": { "status": "UP" },
//     "paymentGateway": { "status": "UP", "details": { "latency": "45ms" } }
//   }
// }
```

**Custom metrics with Micrometer:**

```java
// File: topics/level-2/CustomMetricsDemo.java

@Service
public class TransferService {

    private final Counter transferCounter;
    private final Timer transferTimer;
    private final AtomicInteger activeTransfers;

    public TransferService(MeterRegistry registry) {
        this.transferCounter = Counter.builder("transfers.total")
            .description("Total number of transfers")
            .tag("type", "wire")
            .register(registry);

        this.transferTimer = Timer.builder("transfers.duration")
            .description("Transfer processing time")
            .register(registry);

        this.activeTransfers = registry.gauge("transfers.active",
            new AtomicInteger(0));
    }

    public void transfer(Long from, Long to, BigDecimal amount) {
        activeTransfers.incrementAndGet();
        transferTimer.record(() -> {
            // ... business logic
            transferCounter.increment();
        });
        activeTransfers.decrementAndGet();
    }
}

// Metrics available at: /actuator/metrics/transfers.total
// Prometheus scrape: /actuator/prometheus (with micrometer-registry-prometheus)
```

**Prometheus + Grafana stack (standard at Citi):**

```
App (Micrometer) → /actuator/prometheus → Prometheus (scrapes metrics) → Grafana (dashboards + alerts)
```

```yaml
# application.yml — Prometheus export
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: payment-service
      environment: prod
```

**Alerting rules (Prometheus/Grafana):**
```yaml
# Prometheus alerting rule example
- alert: HighErrorRate
  expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High 5xx error rate on {{ $labels.instance }}"

- alert: HighLatency
  expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
  for: 5m
  labels:
    severity: warning
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Actuator exposes health, metrics, and operational endpoints. Micrometer provides a vendor-neutral metrics API (counters, timers, gauges) that exports to Prometheus, Datadog, or CloudWatch.
2. **Why/when:** Custom health indicators check downstream dependencies (DB, payment gateway, message broker). Custom metrics track business KPIs (transfers/sec, error rates). Prometheus + Grafana visualise and alert.
3. **Example:** A `PaymentGatewayHealthIndicator` pings the gateway and returns UP/DOWN with latency details. Kubernetes uses this for readiness probes — unhealthy instances are removed from the load balancer.
4. **Gotcha/tradeoff:** Don't expose `/actuator/heapdump` or `/actuator/env` publicly — they leak sensitive data. Restrict via Spring Security or management port separation (`management.server.port=9090`).

**Common pitfalls**
- Exposing all actuator endpoints publicly — security risk (env vars, heap dumps).
- Health check that's too slow — Kubernetes kills pods that fail readiness probes.
- Custom metrics with unbounded tag cardinality (e.g., `userId` as tag) — Prometheus cardinality explosion.
- Not alerting on error rates — you find out about outages from customers, not dashboards.

**Self-check question**
Your health check calls an external API. That API is slow (5s timeout). Kubernetes readiness probe timeout is 3s. What happens to your pod? How do you fix it?
