### Topic: Multithreading in Java — Level 2

> **Foundation:** [Level 1 — Thread](../level-1/thread.md) · [Level 1 — Single-Threaded Application](../level-1/single_threaded_application.md)
> **Advanced:** [Level 3 — Threads (async, synchronized, critical region)](../level-3/threads.md) · [Level 3 — Deadlocks](../level-3/deadlocks.md) · [Level 3 — Critical Section of Code](../level-3/critical_section_of_code.md)

**Why it matters (Karat angle)**
L1 covered thread creation and lifecycle. L2 is about **coordination** — producer-consumer, monitor locking, and `wait`/`notify` mechanics. These are the building blocks of every concurrent system at Citi (message processing, batch settlement, queue consumers).

**Core concept**

**Producer-consumer pattern:** One or more threads produce work items; one or more threads consume them. A shared buffer (queue) sits between them.

**Monitor locking (`synchronized`):** Every Java object has an intrinsic lock (monitor). `synchronized` acquires it — only one thread can hold it at a time.

**`wait` / `notify` / `notifyAll`:**

| Method | What it does | Must hold lock? |
|--------|-------------|:--------------:|
| `wait()` | Release lock, sleep until notified | ✅ |
| `notify()` | Wake one waiting thread | ✅ |
| `notifyAll()` | Wake all waiting threads | ✅ |

**Producer-consumer with `wait`/`notify`:**

```java
// File: topics/level-2/ProducerConsumerDemo.java

public class BoundedBuffer<T> {

    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) { this.capacity = capacity; }

    // Producer: add item, wait if full
    public synchronized void produce(T item) throws InterruptedException {
        while (queue.size() == capacity) {               // WHILE, not IF — spurious wakeups
            wait();                                       // release lock, sleep
        }
        queue.add(item);
        notifyAll();                                      // wake consumers
    }

    // Consumer: take item, wait if empty
    public synchronized T consume() throws InterruptedException {
        while (queue.isEmpty()) {                         // WHILE, not IF
            wait();
        }
        T item = queue.poll();
        notifyAll();                                      // wake producers
        return item;
    }
}

// Usage:
BoundedBuffer<String> buffer = new BoundedBuffer<>(10);

// Producer thread
new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        try { buffer.produce("item-" + i); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}).start();

// Consumer thread
new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        try { System.out.println(buffer.consume()); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}).start();
```

**Why `while` and not `if` for wait conditions:**
- Spurious wakeups: the JVM spec allows `wait()` to return without `notify()`.
- Multiple consumers: one consumer may grab the item before another wakes up.
- Always re-check the condition after waking.

**Modern alternative: `BlockingQueue`**

```java
// File: topics/level-2/BlockingQueueDemo.java

// java.util.concurrent — does the wait/notify for you
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// Producer
new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        try { queue.put("item-" + i); }                  // blocks if full
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}).start();

// Consumer
new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        try { System.out.println(queue.take()); }        // blocks if empty
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}).start();
```

| `BlockingQueue` impl | Bounded? | Ordering | Use case |
|---------------------|:--------:|----------|----------|
| `ArrayBlockingQueue` | ✅ | FIFO | Fixed-capacity buffer |
| `LinkedBlockingQueue` | Optional | FIFO | Unbounded (or bounded) task queues |
| `PriorityBlockingQueue` | ❌ | Priority | Scheduled tasks |
| `SynchronousQueue` | No buffer | Direct handoff | Thread pools (`CachedThreadPool`) |

**Monitor locking — how `synchronized` works:**

```java
// File: topics/level-2/MonitorLockingDemo.java

public class AccountService {

    // Method-level lock: locks on 'this'
    public synchronized void debit(Account acc, BigDecimal amount) {
        acc.setBalance(acc.getBalance().subtract(amount));
    }

    // Block-level lock: lock on a specific object
    private final Object transferLock = new Object();

    public void transfer(Account from, Account to, BigDecimal amount) {
        synchronized (transferLock) {                     // only one transfer at a time
            from.setBalance(from.getBalance().subtract(amount));
            to.setBalance(to.getBalance().add(amount));
        }
    }

    // Static synchronized: locks on Class object
    public static synchronized void globalAudit() {
        // only one thread across all instances
    }
}
```

| Lock scope | Lock object | When |
|-----------|------------|------|
| `synchronized` method | `this` (instance) | Protect instance state |
| `synchronized(obj)` block | Any `Object` | Fine-grained locking |
| `static synchronized` method | `Class` object | Protect static state |

**Edge case: `ReentrantLock` — more control than `synchronized`**

```java
// File: topics/level-2/ReentrantLockDemo.java

private final ReentrantLock lock = new ReentrantLock();

public void transferWithTimeout(Account from, Account to, BigDecimal amount)
        throws InterruptedException {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {              // try with timeout
        try {
            from.setBalance(from.getBalance().subtract(amount));
            to.setBalance(to.getBalance().add(amount));
        } finally {
            lock.unlock();                                // ALWAYS in finally
        }
    } else {
        throw new TimeoutException("Could not acquire lock in 5 seconds");
    }
}
```

| | `synchronized` | `ReentrantLock` |
|--|----------------|----------------|
| Syntax | Block/method keyword | Explicit lock/unlock |
| Timeout | ❌ Blocks forever | ✅ `tryLock(timeout)` |
| Interruptible | ❌ | ✅ `lockInterruptibly()` |
| Fair | ❌ (barging) | ✅ `new ReentrantLock(true)` |
| Condition variables | One (wait/notify) | Multiple (`lock.newCondition()`) |

**What to say in the interview (4-beat answer)**
1. **Definition:** Producer-consumer uses a shared buffer with wait/notify (or `BlockingQueue`) for coordination. `synchronized` acquires an object's intrinsic monitor lock — only one thread holds it at a time.
2. **Why/when:** Producer-consumer decouples production rate from consumption rate — essential for message processing, batch jobs, request buffering. Monitor locking protects shared mutable state.
3. **Example:** `BlockingQueue<T>` replaces manual `wait`/`notifyAll` — `put()` blocks when full, `take()` blocks when empty. `ArrayBlockingQueue(100)` gives a bounded 100-item buffer.
4. **Gotcha/tradeoff:** Always use `while` (not `if`) for wait conditions — spurious wakeups are allowed by the JVM spec. Prefer `BlockingQueue` over hand-rolled `wait`/`notify` — it's tested, correct, and simpler.

**Common pitfalls**
- `if` instead of `while` around `wait()` — misses spurious wakeups and competing consumers.
- Calling `wait()`/`notify()` without holding the monitor — throws `IllegalMonitorStateException`.
- `notify()` instead of `notifyAll()` — may wake the wrong thread (a producer when you wanted a consumer).
- Forgetting to `unlock()` in a `finally` block with `ReentrantLock` — deadlock on exception.
- Unbounded `LinkedBlockingQueue` with fast producers — OOM when consumers can't keep up.

**Self-check question**
You have 3 producer threads and 1 consumer thread sharing an `ArrayBlockingQueue(5)`. A producer calls `put()` when the queue is full. What happens to that producer? What if you used `offer()` instead?
