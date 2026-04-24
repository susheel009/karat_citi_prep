### Topic: High Availability Alerts — Level 3

> **Related:** [Level 2 — Monitoring and Alerting](../level-2/monitoring_and_alerting.md) · [Level 3 — Performance Issues](./performance_issues.md) · [Level 3 — Latency Issues](./latency_issues.md)

**Why it matters (Karat angle)**
Banking services have SLAs — 99.9% uptime means only 8.7 hours downtime per year. Interviewers ask about alert types, on-call response, and how you design for high availability. A senior dev must know what to alert on and what to ignore.

**Core concept**

**Alert categories:**

| Category | Examples | Severity |
|----------|---------|:--------:|
| **Availability** | Service down, health check failed, pod crash-looping | Critical |
| **Latency** | p95 > 2s, p99 > 5s, DB query > 10s | Warning/Critical |
| **Error rate** | 5xx rate > 1%, 4xx spike, exception rate spike | Warning/Critical |
| **Saturation** | CPU > 85%, memory > 90%, disk > 80%, thread pool exhausted | Warning |
| **Business** | Payment failure rate > 0.1%, zero transactions in 5 min | Critical |

**The four golden signals (Google SRE):**

| Signal | What to measure | Alert when |
|--------|----------------|-----------|
| **Latency** | Request duration (p50, p95, p99) | p95 > SLA threshold |
| **Traffic** | Requests/second | Unusual spike or drop |
| **Errors** | Error rate (5xx/total) | > baseline + 2σ |
| **Saturation** | Resource usage (CPU, memory, connections) | > 80% sustained |

**Alerting rules (Prometheus + Grafana):**

```yaml
# File: topics/level-3/alert_rules.yml

groups:
  - name: payment-service-alerts
    rules:
      # --- Availability ---
      - alert: ServiceDown
        expr: up{job="payment-service"} == 0
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "Payment service is DOWN"
          runbook: "https://wiki.citi.com/runbooks/payment-service-down"

      # --- Latency ---
      - alert: HighLatencyP95
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{service="payment"}[5m])) > 2
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "p95 latency > 2s for 5 minutes"

      # --- Error Rate ---
      - alert: HighErrorRate
        expr: sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
              / sum(rate(http_server_requests_seconds_count[5m])) > 0.01
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "5xx error rate > 1% for 5 minutes"

      # --- Saturation ---
      - alert: HighCpuUsage
        expr: process_cpu_usage{service="payment"} > 0.85
        for: 10m
        labels: { severity: warning }

      - alert: HeapMemoryHigh
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 5m
        labels: { severity: warning }

      # --- Business ---
      - alert: NoTransactions
        expr: rate(transfers_total[5m]) == 0
        for: 10m
        labels: { severity: critical }
        annotations:
          summary: "Zero transactions in 10 minutes — possible outage"
```

**High availability design:**

| Pattern | How | Failure it handles |
|---------|-----|-------------------|
| **Redundancy** | Multiple instances behind load balancer | Instance failure |
| **Health checks** | Liveness + readiness probes | Detect and remove sick instances |
| **Circuit breaker** | Fail fast on downstream failure | Cascading failure |
| **Retry with backoff** | Retry transient failures | Network blips |
| **Rate limiting** | Cap requests per client | DoS, traffic spikes |
| **Graceful degradation** | Return cached/partial data | Partial outage |
| **Multi-region** | Deploy across data centres | Data centre failure |

**On-call response framework:**

```
1. DETECT: Alert fires → PagerDuty/OpsGenie notification
2. TRIAGE: Check dashboards → determine scope (one user? all users? one region?)
3. MITIGATE: Quick fix (rollback, scale up, restart, toggle feature flag)
4. DIAGNOSE: Root cause analysis (logs, traces, heap dumps)
5. FIX: Deploy permanent fix
6. POSTMORTEM: Document what happened, why, and how to prevent
```

**What to say in the interview (4-beat answer)**
1. **Definition:** HA alerts monitor the four golden signals: latency, traffic, errors, saturation. Alerts have severity levels (warning, critical) with escalation policies and runbooks.
2. **Why/when:** Banking SLAs require 99.9%+ uptime. Alerts detect anomalies before customers notice. Business alerts (zero transactions) catch outages that infrastructure metrics miss.
3. **Example:** `HighErrorRate: 5xx > 1% for 5 minutes → critical alert → page on-call engineer`. Runbook links to investigation steps: check logs, recent deployments, downstream status.
4. **Gotcha/tradeoff:** Too many alerts → alert fatigue → engineers ignore them. Only alert on actionable conditions. Use warning for "investigate soon" and critical for "act now." Every alert should have a runbook.

**Common pitfalls**
- Alert on every 5xx → alert fatigue. Alert on error *rate* exceeding a threshold.
- No runbook linked to alerts → engineer spends 10 minutes finding documentation.
- Alerting on CPU usage alone → CPU spikes are normal during deployment; use sustained threshold.
- No business-level alerts → infrastructure looks fine but payments are failing.
- No `for` duration → alerts fire on transient spikes and auto-resolve.

**Self-check question**
Your payment service has 99.9% SLA (8.7 hours/year downtime). You get 10 alerts per day, 8 of which resolve automatically within 2 minutes. What's wrong with this alerting setup? How would you fix it?
