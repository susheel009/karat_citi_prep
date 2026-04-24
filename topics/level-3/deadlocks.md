### Topic: Deadlocks — Level 3

> **Foundation:** [Level 2 — Multithreading (wait/notify, monitors)](../level-2/multithreading_in_java.md) · [Level 2 — Locking](../level-2/locking.md)
> **Related:** [Level 3 — Threads](./threads.md) · [Level 3 — Critical Section of Code](./critical_section_of_code.md)

**Why it matters (Karat angle)**
Deadlocks freeze banking services silently — no error, no crash, just threads stuck forever. Interviewers ask "how do you detect and prevent deadlocks?" to see if you've diagnosed them in production.

**Core concept**

**Four conditions for deadlock (all must be true):**

| Condition | Meaning | How to break it |
|-----------|---------|----------------|
| **Mutual exclusion** | Only one thread can hold the resource | Use lock-free algorithms (CAS) |
| **Hold and wait** | Thread holds one lock, waits for another | Acquire all locks atomically |
| **No preemption** | Locks can't be forcibly taken | Use `tryLock` with timeout |
| **Circular wait** | A→B, B→A | Always lock in consistent order |

**Classic deadlock:**

```java
// File: topics/level-3/DeadlockDemo.java

public class DeadlockDemo {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void method1() {
        synchronized (lockA) {                             // Thread 1 holds lockA
            sleep(100);                                    // simulate work
            synchronized (lockB) {                         // Thread 1 waits for lockB
                System.out.println("method1");
            }
        }
    }

    public void method2() {
        synchronized (lockB) {                             // Thread 2 holds lockB
            sleep(100);
            synchronized (lockA) {                         // Thread 2 waits for lockA → DEADLOCK
                System.out.println("method2");
            }
        }
    }

    public static void main(String[] args) {
        DeadlockDemo demo = new DeadlockDemo();
        new Thread(demo::method1).start();
        new Thread(demo::method2).start();
        // Both threads stuck forever — no error, no exception, silent hang
    }
}
```

**Fix 1: Consistent lock ordering (best approach):**

```java
// File: topics/level-3/DeadlockFixOrdering.java

// Always lock in the same order (e.g., by object hash or ID)
public void transfer(Account from, Account to, BigDecimal amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**Fix 2: `tryLock` with timeout (ReentrantLock):**

```java
// File: topics/level-3/DeadlockFixTryLock.java

private final ReentrantLock lockA = new ReentrantLock();
private final ReentrantLock lockB = new ReentrantLock();

public void safeMethod() {
    boolean gotBoth = false;
    while (!gotBoth) {
        boolean gotA = false, gotB = false;
        try {
            gotA = lockA.tryLock(1, TimeUnit.SECONDS);
            gotB = lockB.tryLock(1, TimeUnit.SECONDS);
            gotBoth = gotA && gotB;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return;
        } finally {
            if (!gotBoth) {
                if (gotA) lockA.unlock();
                if (gotB) lockB.unlock();
                // Back off and retry
                try { Thread.sleep(ThreadLocalRandom.current().nextInt(100)); }
                catch (InterruptedException ignored) {}
            }
        }
    }
    try {
        // Critical section — both locks held
        doWork();
    } finally {
        lockB.unlock();
        lockA.unlock();
    }
}
```

**Detection tools:**

| Tool | Command | What it shows |
|------|---------|-------------|
| `jstack` | `jstack <pid>` | Thread dump with "Found one Java-level deadlock" |
| `jcmd` | `jcmd <pid> Thread.print` | Thread dump (newer API) |
| `JVisualVM` | GUI | Thread states + deadlock detection |
| `ThreadMXBean` | Programmatic | Runtime deadlock detection |

```java
// File: topics/level-3/DeadlockDetection.java

// Runtime deadlock detection
public class DeadlockDetector {

    @Scheduled(fixedRate = 30_000)
    public void detectDeadlocks() {
        ThreadMXBean tmx = ManagementFactory.getThreadMXBean();
        long[] deadlockedIds = tmx.findDeadlockedThreads();
        if (deadlockedIds != null) {
            ThreadInfo[] infos = tmx.getThreadInfo(deadlockedIds, true, true);
            StringBuilder sb = new StringBuilder("DEADLOCK DETECTED:\n");
            for (ThreadInfo ti : infos) {
                sb.append(ti.getThreadName())
                  .append(" blocked on ").append(ti.getLockName())
                  .append(" held by ").append(ti.getLockOwnerName())
                  .append("\n");
            }
            log.error(sb.toString());
            // Alert ops team
        }
    }
}
```

**`jstack` output for a deadlock:**
```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x000000..., a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00007f... (object 0x000000..., a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
"Thread-1":
    at DeadlockDemo.method2(DeadlockDemo.java:18)
    - waiting to lock <0x000000...> (a java.lang.Object)
    - locked <0x000000...> (a java.lang.Object)
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Deadlock = two+ threads each holding a lock the other needs. Four conditions: mutual exclusion, hold-and-wait, no preemption, circular wait. Break any one to prevent deadlock.
2. **Why/when:** Deadlocks are silent — no exception, no crash, threads just hang. In banking, a deadlocked payment service means transactions stop processing. Detection via `jstack`, `ThreadMXBean`, or scheduled deadlock detector.
3. **Example:** Thread 1 locks account A then waits for account B. Thread 2 locks account B then waits for account A. Fix: always lock accounts in ascending ID order — eliminates circular wait.
4. **Gotcha/tradeoff:** `tryLock` with timeout prevents deadlock but adds retry logic and potential livelock (threads keep retrying without progress). Consistent ordering is simpler and more reliable.

**Common pitfalls**
- Logging "deadlock" when you mean "contention" — deadlock is permanent; contention is temporary blocking.
- Nested `synchronized` blocks without consistent ordering — the #1 cause of deadlocks.
- Using `synchronized` when `ReentrantLock.tryLock()` would be safer — can't timeout with `synchronized`.
- Not monitoring for deadlocks in production — `ThreadMXBean.findDeadlockedThreads()` is cheap to call.

**Self-check question**
You have 3 threads and 3 locks (A, B, C). Thread 1 locks A→B, Thread 2 locks B→C, Thread 3 locks C→A. Is this a deadlock? How would you fix it?
