### Topic: Thread — Level 1

> **Advanced coverage:** [Level 2 — Multithreading (producer-consumer, monitor)](../level-2/multithreading_in_java.md) · [Level 3 — Threads (async, synchronized, critical region)](../level-3/threads.md) · [Level 1 — Single-Threaded Application](./single_threaded_application.md)

**Why it matters (Karat angle)**
Thread questions separate candidates who've only written synchronous code from those who understand concurrency. Interviewers want lifecycle names, `wait/notify` mechanics, and knowledge of why you should use a `ThreadPool` instead of `new Thread()`.

**Core concept**

**Two ways to create a thread**

| Approach | Code | When |
|----------|------|------|
| Implement `Runnable` | `new Thread(() -> work())` | **Preferred** — decouples task from thread; works with `ExecutorService` |
| Extend `Thread` | `class Worker extends Thread` | Rarely; ties you to single inheritance |

Prefer `Runnable` (or `Callable<V>` when you need a return value) — it separates the *what* (task) from the *how* (execution).

**Thread lifecycle (states in `Thread.State` enum)**

| State | How you get there |
|-------|-------------------|
| `NEW` | `new Thread(r)` — not yet started |
| `RUNNABLE` | `thread.start()` — running or ready-to-run (OS scheduler decides) |
| `BLOCKED` | Waiting to acquire a `synchronized` lock |
| `WAITING` | `wait()`, `join()`, `LockSupport.park()` — until notified |
| `TIMED_WAITING` | `sleep(ms)`, `wait(ms)`, `join(ms)` |
| `TERMINATED` | `run()` returned or threw |

**Key methods (memorise these)**

| Method | Does |
|--------|------|
| `start()` | Starts the thread; JVM calls `run()` on the new thread |
| `run()` | The task body; calling `run()` directly runs on the CURRENT thread — NOT a new thread |
| `sleep(ms)` | Pauses current thread for `ms`; does NOT release locks |
| `join()` | Current thread waits until target thread terminates |
| `wait()` | Releases the lock, waits in the monitor's wait set; MUST be inside `synchronized(obj)` |
| `notify()` / `notifyAll()` | Wakes one / all waiting threads on the monitor; must be inside `synchronized(obj)` |
| `interrupt()` | Sets the thread's interrupt flag; cooperating threads check and exit gracefully |

**wait/notify vs sleep** — the fundamental pair:

| | `sleep(ms)` | `wait()` |
|-|-------------|----------|
| Needs to be in `synchronized`? | No | Yes |
| Releases lock? | No — holds it | Yes — releases during wait, reacquires on wake |
| Resumes on | Timer | `notify` / `notifyAll` (or timer if `wait(ms)`) |

**ThreadPool — why you almost always want it**

Creating a new `Thread` for every task costs ~1 MB of stack and a system call. At scale, this OOMs or bogs the scheduler. A **thread pool** (via `Executors` or `new ThreadPoolExecutor(...)`) keeps a fixed set of worker threads, hands them tasks from a queue, and reuses them.

| Factory | Behaviour |
|---------|-----------|
| `Executors.newFixedThreadPool(n)` | Fixed `n` threads; unbounded task queue (can OOM) |
| `Executors.newCachedThreadPool()` | Grows unbounded; threads die after 60s idle (risky for bursty) |
| `Executors.newSingleThreadExecutor()` | One thread, serial tasks |
| `Executors.newScheduledThreadPool(n)` | Supports `schedule(delay)` and `scheduleAtFixedRate` |
| `new ThreadPoolExecutor(...)` | Full control — bounded queue, rejection policy, custom thread factory |

**In production, prefer `new ThreadPoolExecutor(...)` with a bounded queue** — otherwise a task backlog can exhaust heap.

**Mental model:** a thread is a worker. Creating one is expensive. A pool is a team of workers that gets tasks from an inbox.

**Real-world use cases**
- **Async REST endpoint:** return immediately, process in a pool, respond later via WebSocket / callback.
- **Batch job:** split N files across a fixed pool of workers.
- **Scheduled task:** `ScheduledExecutorService` fires a cleanup job every minute.
- **Producer-consumer:** one thread enqueues, several dequeue and process — classic integration pattern.
- **Virtual threads (Java 21):** thread-per-request for I/O-bound services; see [Level 1 — Java Versions](./java_versions.md).

**Working code example**
```java
// File: topics/level-1/ThreadDemo.java
import java.util.concurrent.*;

public class ThreadDemo {

    public static void main(String[] args) throws Exception {

        // ---------- Runnable via ExecutorService (preferred) ----------
        ExecutorService pool = Executors.newFixedThreadPool(3);

        for (int i = 0; i < 5; i++) {
            int id = i;
            pool.submit(() -> {
                System.out.println("task " + id + " on " + Thread.currentThread().getName());
                try { Thread.sleep(100); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();     // restore flag — important idiom
                }
            });
        }

        // ---------- Callable — returns a value ----------
        Future<Integer> future = pool.submit(() -> {
            Thread.sleep(50);
            return 42;
        });
        System.out.println("Result: " + future.get());      // blocks until done

        pool.shutdown();                                    // stop accepting new tasks
        pool.awaitTermination(1, TimeUnit.SECONDS);

        // ---------- join: wait for a thread to finish ----------
        Thread t = new Thread(() -> System.out.println("background work"));
        t.start();
        t.join();                                           // main waits for t
        System.out.println("t finished: " + t.getState()); // TERMINATED

        // ---------- Common bug: calling run() instead of start() ----------
        Thread bug = new Thread(() -> System.out.println("on " + Thread.currentThread().getName()));
        bug.run();     // runs on MAIN thread — NOT a new thread
        bug.start();   // correct — runs on a new thread
    }
}
```

**Edge case: wait/notify pattern**
```java
// File: topics/level-1/WaitNotifyDemo.java
class Box {
    private String item;

    public synchronized void put(String x) {
        while (item != null) {                       // always wait in a loop (spurious wakeups!)
            try { wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
        }
        item = x;
        notifyAll();                                 // wake waiters
    }

    public synchronized String take() {
        while (item == null) {
            try { wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); return null; }
        }
        String r = item;
        item = null;
        notifyAll();
        return r;
    }
}
```
**Always `wait()` in a loop, never an `if`** — threads can wake spuriously (JVM is allowed to). Re-check the condition after waking.

**What to say in the interview (4-beat answer)**
1. **Definition:** A Java thread is a unit of parallel execution managed by the JVM. You create it by implementing `Runnable` (preferred) or extending `Thread`, and start it with `start()` — which internally calls `run()` on a new OS thread.
2. **Why/when:** For any real app, avoid creating raw `Thread` objects — use an `ExecutorService` from `Executors` or `new ThreadPoolExecutor()` for pooling. Pools amortise thread creation and bound concurrency.
3. **Example:** `pool.submit(() -> task())` hands a `Runnable` to a worker thread; `Callable<V>` returns a `Future<V>` you can `.get()` on to retrieve the result.
4. **Gotcha/tradeoff:** Calling `run()` directly executes on the current thread — no parallelism at all. `sleep()` holds the lock; `wait()` releases it. Always `wait()` in a loop to handle spurious wakeups.

**Common pitfalls**
- Calling `thread.run()` instead of `thread.start()` — no new thread is created.
- `wait()` in an `if` instead of a `while` — wakes spuriously and proceeds with invalid state.
- Using `Executors.newFixedThreadPool` with no regard for queue size — the default unbounded queue can OOM under load.
- Swallowing `InterruptedException` — always restore the flag with `Thread.currentThread().interrupt()` or propagate.
- Sharing mutable state without `synchronized` / `java.util.concurrent` — data races, inconsistent reads.

**Self-check question**
Why is `Executors.newCachedThreadPool()` dangerous in a high-traffic service, and what's the safer alternative for an unpredictable bursty workload?
