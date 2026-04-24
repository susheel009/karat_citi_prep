### Topic: JVM, JRE, and JDK — Level 1

> **Advanced coverage:** [Level 3 — GC Algorithms](../level-3/gc_algorithms.md) · [Level 3 — JVM Memory Settings](../level-3/jvm_memory_settings.md) · [Level 3 — Garbage Collector](../level-3/garbage_collector.md)

**Why it matters (Karat angle)**
This is asked to see if you understand how Java code actually runs. At banks, JVM tuning is the difference between a payment system that handles peak load and one that GC-pauses mid-transaction.

**Core concept**

These three terms are commonly confused — they're nested, not interchangeable:

| Component | Contains | Who uses it |
|-----------|----------|-------------|
| **JDK** (Development Kit) | JRE + compiler (`javac`) + dev tools (`jar`, `jshell`, `jstack`, `jmap`, `jdb`) | Developers |
| **JRE** (Runtime Environment) | JVM + standard class libraries | Runtime of the app (historically) |
| **JVM** (Virtual Machine) | Classloader + bytecode verifier + interpreter + JIT + GC + memory areas | The engine that actually runs bytecode |

**Note:** Since Java 11, Oracle no longer ships a standalone JRE. Production images now bundle a slimmed JDK built with `jlink`, or a distroless JRE image from OpenJDK distros.

**What the JVM actually does when you run `java MyApp`:**

1. **Classloader** reads `.class` bytecode from classpath, loads it into memory.
2. **Bytecode verifier** validates bytecode safety (no illegal casts, stack operations well-formed).
3. **Execution engine** runs bytecode — first via **interpreter**, then via **JIT** compiler for hot methods.
4. **Runtime data areas** manage memory: heap (objects), stack (method frames), metaspace (class metadata).
5. **GC** reclaims unreachable heap objects in the background.

**JVM memory areas**

| Area | What lives there | GC'd? | OOM flavour |
|------|------------------|-------|-------------|
| **Heap** | All objects created with `new` | Yes (GC's job) | `OutOfMemoryError: Java heap space` |
| **Stack** (per thread) | Method frames: local vars, operand stack, return address | No — pops on method return | `StackOverflowError` (infinite recursion) |
| **Metaspace** | Class metadata, method bytecode, static fields, string pool | Rarely (only on class unload) | `OutOfMemoryError: Metaspace` |
| **PC Register** (per thread) | Instruction pointer | No | N/A |
| **Native method stack** | Native (JNI) calls | No | Native OOM |

**JIT (Just-In-Time) compilation** — the JVM starts by interpreting bytecode (slow but fast to start). As methods execute repeatedly, the JIT compiles them to native machine code and replaces the interpreted path. HotSpot JVM has two tiers: **C1** (fast compile, modest optimisation) and **C2** (slower compile, aggressive optimisation). This is why the first few calls to a method are slow and subsequent calls are fast ("warm-up").

**JVM implementations worth knowing:**
- **HotSpot** (Oracle/OpenJDK) — the default; dominant in industry.
- **OpenJ9** (IBM) — lower memory footprint, better for containers.
- **GraalVM** — supports native image compilation (ahead-of-time), near-instant startup.

**Real-world use cases**
- **Docker images:** use `eclipse-temurin:17-jre` (JRE only) for production — smaller image, no compiler.
- **Diagnosing OOM:** look at the message — "heap" vs "metaspace" vs "direct buffer" each point to a different cause.
- **Tuning a banking service:** `-Xms4g -Xmx4g` (fixed heap, no resize), `-XX:+UseG1GC`, `-XX:MaxGCPauseMillis=200` for predictable pauses.
- **Profiling in prod:** `jstack <pid>` for thread dumps, `jmap -dump` for heap dumps, `jcmd` for everything else.

**Working code example**
```java
// File: topics/level-1/JvmDemo.java

public class JvmDemo {

    // Each call = one stack frame. Infinite recursion → StackOverflowError.
    static int factorial(int n) {
        if (n <= 1) return 1;
        return n * factorial(n - 1);
    }

    // Each iteration allocates a new object on the heap.
    // Without GC this would OOM; with GC these become collectable after loop exits.
    static void allocateGarbage() {
        for (int i = 0; i < 100_000; i++) {
            byte[] buf = new byte[1024];              // 1 KB per iteration
        }
    }

    public static void main(String[] args) {
        Runtime rt = Runtime.getRuntime();
        long mb = 1024L * 1024L;
        System.out.println("Max heap:   " + rt.maxMemory()   / mb + " MB");
        System.out.println("Total heap: " + rt.totalMemory() / mb + " MB (currently allocated)");
        System.out.println("Free heap:  " + rt.freeMemory()  / mb + " MB");
        System.out.println("CPU cores:  " + rt.availableProcessors());

        System.out.println("factorial(10) = " + factorial(10));
        allocateGarbage();
        System.out.println("After allocation, free heap: " + rt.freeMemory() / mb + " MB");
    }
}
```

**Common JVM flags worth knowing**
```bash
-Xms4g -Xmx4g                    # Initial and max heap (set equal in prod to avoid resize)
-Xss256k                          # Per-thread stack size
-XX:+UseG1GC                      # Use G1 garbage collector (default on Java 9+)
-XX:MaxGCPauseMillis=200          # Target pause time for G1
-XX:MaxMetaspaceSize=512m         # Cap metaspace (unbounded by default!)
-XX:+HeapDumpOnOutOfMemoryError   # Dump heap on OOM — essential in prod
-XX:HeapDumpPath=/var/log/app/    # Where to dump
-verbose:gc -Xlog:gc*             # GC logging (Java 9+ uses -Xlog)
```

**What to say in the interview (4-beat answer)**
1. **Definition:** JDK is for developers (includes compiler and dev tools); JRE is for running Java apps (JVM + libraries); JVM is the engine that loads, verifies, and executes bytecode.
2. **Why/when:** Production Docker images use JRE-only for smaller size; developers need the full JDK; JVM tuning flags directly control heap, GC behaviour, and pause times.
3. **Example:** `Runtime.getRuntime().maxMemory()` shows the heap cap the JVM is working with — critical to check when an app OOMs in prod but not locally.
4. **Gotcha/tradeoff:** Metaspace (replaced PermGen in Java 8) is **unbounded by default** — class-heavy frameworks like Spring can leak metaspace through dynamic proxies; always set `-XX:MaxMetaspaceSize` in prod.

**Common pitfalls**
- Confusing `StackOverflowError` (stack exhausted, usually infinite recursion) with `OutOfMemoryError` (heap full).
- Running in prod with no `-Xmx` — default is 25% of system RAM, which might be far too much or far too little.
- Leaving `-XX:+HeapDumpOnOutOfMemoryError` off — when OOM hits at 3am you have no forensic data.
- Thinking JRE still ships as a separate download — it doesn't since Java 11; use `jlink` to build one.

**Self-check question**
A Spring Boot service crashes with `OutOfMemoryError: Metaspace`. What does this tell you about the cause (versus heap OOM), and what's one flag you'd add to diagnose?
