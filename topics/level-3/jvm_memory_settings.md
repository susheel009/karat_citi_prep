### Topic: JVM Memory Settings — Level 3

> **Related:** [Level 3 — Garbage Collector](./garbage_collector.md) · [Level 3 — GC Algorithms](./gc_algorithms.md) · [Level 2 — Heap and Stack](../level-2/heap_and_stack.md)

**Why it matters (Karat angle)**
Production JVM services need tuned memory settings. Interviewers ask "what JVM flags do you use?" to see if you've operated Java in production — not just developed locally with defaults.

**Core concept**

**JVM memory regions:**

```
┌──────────────────────────────────────────────────────────┐
│ JVM Process Memory                                       │
├───────────────────────────┬──────────────────────────────┤
│ HEAP (-Xmx)              │ OFF-HEAP                     │
│ ├── Young Gen (-Xmn)     │ ├── Metaspace               │
│ │   ├── Eden              │ │   (-XX:MaxMetaspaceSize)  │
│ │   ├── Survivor S0       │ ├── Thread stacks           │
│ │   └── Survivor S1       │ │   (-Xss per thread)      │
│ └── Old Gen               │ ├── Direct buffers          │
│                           │ │   (-XX:MaxDirectMemorySize)│
│                           │ ├── Code cache              │
│                           │ └── JNI, mapped files       │
└───────────────────────────┴──────────────────────────────┘
```

**Essential JVM flags:**

| Flag | Purpose | Typical value |
|------|---------|:------------:|
| `-Xms` | Initial heap size | 2g |
| `-Xmx` | Maximum heap size | 4g |
| `-Xmn` | Young generation size | 1g (or use ratio) |
| `-Xss` | Thread stack size | 512k |
| `-XX:MaxMetaspaceSize` | Max class metadata | 256m |
| `-XX:MaxDirectMemorySize` | Max direct ByteBuffers | 256m |
| `-XX:+UseG1GC` | G1 garbage collector | (default in Java 9+) |
| `-XX:MaxGCPauseMillis` | Target GC pause time | 200 |
| `-XX:+HeapDumpOnOutOfMemoryError` | Auto heap dump on OOM | Always enable |
| `-XX:HeapDumpPath` | Where to write heap dump | /opt/dumps/ |

**Production JVM configuration:**

```bash
# File: topics/level-3/jvm_production_flags.sh

java \
  -Xms4g -Xmx4g \                        # fixed heap (avoid resize pauses)
  -Xmn1g \                                # 25% of heap for Young Gen
  -Xss512k \                              # stack per thread (default 1M — reduce if many threads)
  -XX:MaxMetaspaceSize=256m \             # cap class metadata
  -XX:+UseG1GC \                          # G1 collector (balanced throughput + latency)
  -XX:MaxGCPauseMillis=200 \              # target 200ms max GC pause
  -XX:+HeapDumpOnOutOfMemoryError \       # capture heap dump on OOM
  -XX:HeapDumpPath=/opt/dumps/ \          # dump location
  -Xlog:gc*:file=/opt/logs/gc.log:time,level,tags \  # GC logging
  -XX:+ExitOnOutOfMemoryError \           # kill JVM on OOM (let Kubernetes restart)
  -jar payment-service.jar
```

**Setting `Xms = Xmx` — why?**
- Avoids heap resize pauses during startup and load ramps.
- The OS allocates all memory upfront — no fragmentation.
- Standard practice in containerised (Kubernetes) environments.

**Container-aware JVM settings (Kubernetes/Docker):**

```bash
# Container has 8GB memory limit
# JVM should use ~75% of container memory for heap (rest for off-heap)

java \
  -XX:MaxRAMPercentage=75.0 \             # 75% of container memory as max heap
  -XX:InitialRAMPercentage=75.0 \         # start at 75%
  -XX:+UseContainerSupport \              # respect cgroup limits (default since Java 10)
  -jar app.jar

# DO NOT hardcode -Xmx in containers if the memory limit varies
# Use MaxRAMPercentage instead — adapts to container size
```

**Diagnosing memory settings:**

```bash
# Check current JVM flags
jcmd <pid> VM.flags

# Check heap usage
jstat -gcutil <pid> 1000                   # GC stats every second
# Output: S0=0.00 S1=85.3 E=45.2 O=62.1 M=94.3 YGC=543 YGCT=12.3 FGC=3 FGCT=4.5 GCT=16.8

# Heap summary
jmap -heap <pid>

# Live memory breakdown
jcmd <pid> GC.heap_info
```

| Column | Meaning |
|--------|---------|
| S0, S1 | Survivor space usage % |
| E | Eden usage % |
| O | Old gen usage % |
| M | Metaspace usage % |
| YGC/YGCT | Young GC count / total time |
| FGC/FGCT | Full GC count / total time |

**Sizing guidelines:**

| Scenario | Heap | Young Gen | Notes |
|----------|:----:|:---------:|-------|
| Spring Boot REST API | 2–4g | 25–33% | Most objects are short-lived |
| Batch processing | 4–8g | 40% | Large datasets in memory |
| Low-latency trading | 2–8g | 50% | Minimise GC pauses |
| In-memory cache (Hazelcast) | 8–32g | 10–15% | Most data in Old Gen |

**What to say in the interview (4-beat answer)**
1. **Definition:** `-Xms`/`-Xmx` control heap size. `-Xmn` sets Young Gen. `-Xss` sets thread stack. Set `Xms = Xmx` in production to avoid resize pauses. Use `-XX:MaxRAMPercentage` in containers.
2. **Why/when:** Undersized heap → frequent GC, OOM. Oversized → wasted resources, longer GC pauses. Always enable `-XX:+HeapDumpOnOutOfMemoryError` and GC logging in production.
3. **Example:** Container with 8GB limit: `-XX:MaxRAMPercentage=75.0` gives 6GB heap + 2GB for off-heap (Metaspace, threads, direct buffers). If you hardcode `-Xmx7g`, the container is OOM-killed by the OS.
4. **Gotcha/tradeoff:** JVM total memory > heap. A 4GB `-Xmx` JVM may use 5–6GB total (Metaspace, thread stacks, direct buffers, code cache). In containers, set `-Xmx` to ~75% of memory limit, not 100%.

**Common pitfalls**
- `-Xmx` = container memory limit → OS OOM-kills the JVM (off-heap not accounted for).
- Not setting `-XX:+HeapDumpOnOutOfMemoryError` → OOM happens, no data to diagnose.
- Default `-Xss` (1MB) with 500 threads → 500MB just for stacks. Reduce to 512k if not deeply recursive.
- Not logging GC → can't diagnose pause issues after the fact.
- Ignoring Metaspace → unlimited by default → can grow indefinitely (class loader leaks).

**Self-check question**
Your Kubernetes pod has 4GB memory limit. You set `-Xmx3g`. The JVM is OOM-killed by the OS (exit code 137). What happened? What's the correct `-Xmx` value?
