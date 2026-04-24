### Topic: GC Algorithms — Level 3

> **Related:** [Level 3 — Garbage Collector](./garbage_collector.md) · [Level 3 — JVM Memory Settings](./jvm_memory_settings.md) · [Level 3 — Performance Issues](./performance_issues.md)

**Why it matters (Karat angle)**
"Which GC algorithm do you use and why?" is a deep L3 question. The answer depends on your latency vs throughput requirements. Citi's payment services need low-latency GC (G1, ZGC); batch jobs need throughput (Parallel GC).

**Core concept**

**GC algorithm comparison:**

| Algorithm | Throughput | Max pause | Heap size | Use case | JVM flag |
|-----------|:---------:|:---------:|:---------:|---------|---------|
| **Serial** | Low | High | Small (<100MB) | Single-core, client apps | `-XX:+UseSerialGC` |
| **Parallel** | Highest | Medium (100ms–1s) | Medium (1–8GB) | Batch processing, throughput | `-XX:+UseParallelGC` |
| **G1** | High | Low (10–200ms) | Large (4–64GB) | General-purpose (default) | `-XX:+UseG1GC` |
| **ZGC** | High | Very low (<10ms) | Very large (8GB–16TB) | Ultra-low latency | `-XX:+UseZGC` |
| **Shenandoah** | High | Very low (<10ms) | Large | Low latency (RedHat) | `-XX:+UseShenandoahGC` |

**Serial GC** — simplest, single-threaded:
```
Stop all app threads → Mark → Sweep → Compact → Resume
One GC thread does everything → long pauses
Use: small apps, embedded, testing
```

**Parallel GC** — throughput-optimised:
```
Stop all app threads → Mark (parallel) → Sweep (parallel) → Compact → Resume
Multiple GC threads work simultaneously → shorter pauses than Serial
But still stop-the-world → pauses scale with heap size
Use: batch jobs where throughput > latency
```

**G1 GC** — balanced (default since Java 9):
```
Heap divided into ~2000 regions (not fixed generations)
Each region is Eden, Survivor, Old, or Humongous

1. Young GC: collect Eden + Survivor regions (stop-the-world, parallel)
2. Concurrent marking: trace live objects concurrently with app threads
3. Mixed GC: collect Young + some Old regions with most garbage
4. Evacuation: copy live objects to empty regions, compact

Key: G1 prioritises regions with most garbage → "Garbage-First"
Target: -XX:MaxGCPauseMillis=200 → G1 adapts region selection to meet target
```

```java
// File: topics/level-3/G1ConfigDemo.java

// Production G1 configuration
// java -XX:+UseG1GC
//      -XX:MaxGCPauseMillis=200          target pause time
//      -XX:G1HeapRegionSize=4m           region size (auto-calculated if omitted)
//      -XX:InitiatingHeapOccupancyPercent=45   start concurrent marking at 45% heap
//      -XX:G1ReservePercent=10           reserve 10% for promotions
```

**ZGC** — ultra-low latency (Java 15+):
```
- Concurrent mark, relocate, and remap — almost no STW pauses
- Colored pointers: uses unused bits in 64-bit pointers for GC metadata
- Load barriers: intercept object reads to handle relocated objects
- Pauses: <10ms regardless of heap size (even 16TB)
- Trade-off: slightly lower throughput than G1 (~5-10% overhead)
```

```bash
# ZGC configuration
java -XX:+UseZGC \
     -XX:+ZGenerational \                  # Generational ZGC (Java 21+)
     -Xmx8g \
     -jar low-latency-service.jar
```

**Choosing the right GC:**

| Service type | GC choice | Why |
|-------------|-----------|-----|
| REST API (general) | **G1** | Good balance of throughput and latency |
| Payment processing (low-latency) | **ZGC** | <10ms pauses, even under load |
| Batch ETL job | **Parallel** | Max throughput, pauses don't matter |
| Microservice on small container | **Serial** or **G1** | Low memory overhead |
| In-memory cache (large heap) | **ZGC** | Handles >32GB without long pauses |

**GC log analysis:**

```bash
# Enable GC logging (Java 9+)
java -Xlog:gc*:file=gc.log:time,level,tags:filecount=5,filesize=10m

# Sample G1 GC log:
# [2026-04-23T20:30:00.123] GC(42) Pause Young (Normal) 2048M->1024M(4096M) 12.345ms
#                                   ^type        ^before ^after  ^heap     ^pause

# Key metrics to watch:
# - Pause time: should be < MaxGCPauseMillis target
# - Frequency: too often = heap too small or too much allocation
# - Full GC: should be rare (0 ideally) — indicates heap exhaustion
# - Promotion rate: objects moving Young → Old too fast
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Serial (single-thread), Parallel (multi-thread, throughput), G1 (region-based, balanced), ZGC (concurrent, <10ms pauses). G1 is the default since Java 9. ZGC is for ultra-low latency.
2. **Why/when:** G1 for most Spring Boot services (200ms pause target). ZGC for payment/trading services needing <10ms pauses. Parallel for batch jobs where throughput matters more than latency.
3. **Example:** G1 with `-XX:MaxGCPauseMillis=200` — G1 adapts region selection to keep pauses under 200ms. If pauses exceed this, increase heap or reduce allocation rate.
4. **Gotcha/tradeoff:** ZGC has the lowest pauses but ~5–10% throughput overhead. G1's `MaxGCPauseMillis` is a target, not a guarantee — large heaps with high allocation rates will exceed it. Full GC on G1 is a stop-the-world fallback.

**Common pitfalls**
- Using Parallel GC for latency-sensitive services — long pauses.
- Setting `MaxGCPauseMillis=10` on G1 — G1 can't achieve this consistently; use ZGC instead.
- Not monitoring Full GC count — should be 0 in normal operation; indicates heap pressure.
- Ignoring GC logs — "it works fine" until a 5-second Full GC pause causes timeouts.
- Oversizing heap to "fix" GC pauses — larger heap = longer Full GC when it happens.

**Self-check question**
Your G1-configured service has `MaxGCPauseMillis=200` but GC logs show occasional 500ms pauses with "Pause Full (Allocation Failure)." What does this mean? What are three things you'd try to fix it?
