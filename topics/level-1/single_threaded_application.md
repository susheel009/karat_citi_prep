### Topic: Single-Threaded Application — Level 1

> **Related:** [Level 1 — Thread](./thread.md) · [Level 2 — Multithreading in Java](../level-2/multithreading_in_java.md) · [Level 3 — Threads (advanced)](../level-3/threads.md)

**Why it matters (Karat angle)**
This sounds basic, but the interviewer is checking two things: can you write and reason about the simplest case, AND do you know *when* single-threaded is actually the right choice? "Just add threads" isn't always the answer.

**Core concept**

A single-threaded application does all its work on **one thread** — by default the JVM's `main` thread (and a few JVM-internal threads like GC). Execution is strictly sequential: line N finishes before line N+1 starts.

**JVM structure of a simple app**

When you run `java App`:
1. JVM starts.
2. Main thread (`Thread.currentThread().getName() == "main"`) enters `main(String[] args)`.
3. Executes each statement in order.
4. When `main` returns AND all non-daemon threads have finished, the JVM exits.

**Threads present even in a "single-threaded" app:**
- `main` — your code.
- GC threads — reclaim memory in background.
- Finalizer, Reference Handler — legacy cleanup.
- Signal Dispatcher — OS signals.

So "single-threaded" really means "your application logic is on one thread."

**When single-threaded is actually the right choice**

| Scenario | Why single-threaded wins |
|----------|-------------------------|
| Small CLI tools, scripts | Simpler code; no concurrency bugs to debug |
| Batch data transformation | Often I/O or DB-bound; threads don't help if the bottleneck is external |
| Event loop frameworks | Node.js model — one thread per CPU, async I/O (Vert.x, Netty handlers) |
| Sequential business logic | Ordering constraints (financial transactions in event order) |
| Tests | Fewer variables = more deterministic |

**When to add threads**

| Scenario | Why multi-threading helps |
|----------|-------------------------|
| Multiple independent I/O operations | Overlap blocking time (parallel HTTP calls) |
| CPU-bound parallelisable work | Use all cores (data parallel, matrix ops, encoding) |
| Responsive UI | Keep event thread free; do work in background |
| Server handling concurrent requests | One thread (or virtual thread) per request |

**Java 21 twist: virtual threads change the calculus.** With virtual threads, you can write thread-per-request code that looks synchronous — the JVM handles mounting/unmounting to OS threads during blocking I/O. This lets you stay in a simple "one logical thread per request" model with massive concurrency.

**Mental model:** single-threaded is a single cashier. Multi-threaded is multiple cashiers. Adding cashiers only helps if there are customers waiting — if the queue is always short, you just pay more cashiers for nothing (and they fight over the till → locking).

**Real-world use cases**
- **Command-line tools:** `javac`, `jshell`, Maven goals.
- **ETL scripts:** read file → transform → write output.
- **Simple REST service under low traffic:** Tomcat's default thread pool handles concurrency for you, but your `@RestController` methods are typically straight-line single-threaded logic.
- **Event loop servers:** Netty event loop, each loop is single-threaded; you scale by running N loops (one per core).

**Working code example**
```java
// File: topics/level-1/SingleThreadedDemo.java
import java.nio.file.*;
import java.util.*;
import java.util.stream.*;

public class SingleThreadedDemo {

    public static void main(String[] args) throws Exception {

        System.out.println("Running on: " + Thread.currentThread().getName()); // main

        // ---------- Sequential pipeline ----------
        List<Integer> nums = IntStream.rangeClosed(1, 10).boxed().toList();

        int sumOfSquaresOfEvens = nums.stream()
            .filter(n -> n % 2 == 0)
            .mapToInt(n -> n * n)
            .sum();

        System.out.println("Sum of squares of evens: " + sumOfSquaresOfEvens);

        // ---------- A typical single-threaded file transform ----------
        // Not actually writing to disk — replace with Files.write(...) for real use
        List<String> upper = Files.lines(Paths.get("README.md"))    // lazy read
            .filter(line -> !line.isBlank())
            .map(String::toUpperCase)
            .collect(Collectors.toList());
        System.out.println("Processed " + upper.size() + " lines");

        // ---------- What happens if you ignore threads ----------
        // No races, no locks needed. Easy to reason about.
        int counter = 0;
        for (int i = 0; i < 1_000_000; i++) counter++;
        System.out.println("counter = " + counter);   // always 1,000,000 — no races

        // By contrast, the same counter across threads without synchronisation
        // could produce any number below 1,000,000 — that's the bug we avoid here.
    }
}
```

**Edge case: "single-threaded" under Spring Boot**

When your Spring Boot app starts, you're used to writing code that looks single-threaded inside a `@RestController` method. But Tomcat runs **multiple request threads concurrently** — your controller method is called on different threads for different requests. The "single-threaded feel" is an illusion at the per-request level.

**Implication:** anything stored in a `@Component` that's mutated must be thread-safe (or stateless). A field like `private int counter;` in a `@Service` is a race condition waiting to happen.

**What to say in the interview (4-beat answer)**
1. **Definition:** A single-threaded application executes its logic on one thread, typically `main`; statements run in strict sequential order. Even so, the JVM has background threads (GC, signal handler) running alongside.
2. **Why/when:** It's the right default for CLI tools, batch scripts, sequential business logic with ordering constraints, and anywhere the bottleneck is external I/O not CPU parallelism.
3. **Example:** A pipeline like `files.stream().filter(...).map(...).forEach(...)` runs on `main` — clean, no concurrency bugs, easy to debug.
4. **Gotcha/tradeoff:** Spring Boot REST controllers look single-threaded in the code but run concurrently across request threads — shared mutable state in a `@Service` is a bug. Java 21 virtual threads bring back a single-threaded *feel* per request with massive concurrency underneath.

**Common pitfalls**
- Assuming "looks sequential" means "is sequential" in server code — Tomcat calls your controller on many threads.
- Adding threads prematurely — if work is external-I/O bound, threading adds complexity without speedup (look at virtual threads instead).
- Using `new Thread()` for a one-off async task — use `CompletableFuture.supplyAsync` or a pool.
- Writing sequential code that mutates shared fields inside a `@Component` — compiles fine, breaks under load.

**Self-check question**
A `@Service` in a Spring Boot app has `private int requestCount = 0;` that it increments on every request. What goes wrong, and how would you fix it without introducing a lock?
