### Topic: Critical Section of Code — Level 3

> **Related:** [Level 3 — Deadlocks](./deadlocks.md) · [Level 3 — Threads](./threads.md) · [Level 2 — Volatile Keyword](../level-2/volatile_keyword.md)

**Why it matters (Karat angle)**
Race conditions in banking code = double-spend, lost updates, corrupted balances. Interviewers ask about critical sections to test whether you can identify shared mutable state, protect it correctly, and choose the right synchronisation tool.

**Core concept**

A **critical section** is code that accesses shared mutable state and must be executed by only one thread at a time.

**Race condition — the problem:**

```java
// File: topics/level-3/RaceConditionDemo.java

public class BankAccount {

    private int balance = 1000;                            // shared mutable state

    // NOT thread-safe — critical section unprotected
    public void withdraw(int amount) {
        if (balance >= amount) {                           // CHECK
            // Thread A checks balance=1000, pauses here
            // Thread B checks balance=1000, proceeds
            balance -= amount;                             // ACT
            // Both threads withdraw — balance goes negative!
        }
    }

    // This is a "check-then-act" race condition
    // The check and the act must be ATOMIC (indivisible)
}
```

**Protecting critical sections:**

| Mechanism | Scope | Blocking | Fairness | Reentrant |
|-----------|:-----:|:--------:|:--------:|:---------:|
| `synchronized` | Block/method | Yes | No | Yes |
| `ReentrantLock` | Explicit lock/unlock | Yes (tryLock = optional) | Configurable | Yes |
| `ReadWriteLock` | Read shared, write exclusive | Yes | Configurable | Yes |
| `StampedLock` | Optimistic read, write exclusive | Optional | No | No |
| `Atomic*` | Single variable | No (CAS) | N/A | N/A |
| `volatile` | Visibility only | No | N/A | N/A |

```java
// File: topics/level-3/CriticalSectionFixes.java

// --- Fix 1: synchronized block ---
public class SafeAccount {
    private int balance = 1000;
    private final Object lock = new Object();

    public void withdraw(int amount) {
        synchronized (lock) {                              // only one thread at a time
            if (balance >= amount) {
                balance -= amount;
            }
        }
    }
}

// --- Fix 2: ReentrantLock with tryLock ---
public class LockAccount {
    private int balance = 1000;
    private final ReentrantLock lock = new ReentrantLock();

    public boolean withdraw(int amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
                return true;
            }
            return false;
        } finally {
            lock.unlock();                                 // ALWAYS in finally
        }
    }
}

// --- Fix 3: AtomicInteger (lock-free, single variable) ---
public class AtomicAccount {
    private final AtomicInteger balance = new AtomicInteger(1000);

    public boolean withdraw(int amount) {
        while (true) {
            int current = balance.get();
            if (current < amount) return false;
            if (balance.compareAndSet(current, current - amount)) return true;
            // CAS failed → another thread modified → retry
        }
    }
}
```

**ReadWriteLock — optimising for read-heavy workloads:**

```java
// File: topics/level-3/ReadWriteLockDemo.java

public class AccountCache {
    private final Map<Long, Account> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public Account read(Long id) {
        rwLock.readLock().lock();                           // multiple readers concurrently
        try {
            return cache.get(id);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void write(Long id, Account account) {
        rwLock.writeLock().lock();                          // exclusive — blocks all readers and writers
        try {
            cache.put(id, account);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

**Common race condition patterns:**

| Pattern | Example | Fix |
|---------|---------|-----|
| Check-then-act | `if (map.get(k) == null) map.put(k, v)` | `map.putIfAbsent(k, v)` |
| Read-modify-write | `count++` | `AtomicInteger.incrementAndGet()` |
| Compound action | `if (q.size() < max) q.add(item)` | Use `BlockingQueue.offer()` |
| Lazy init | `if (instance == null) instance = new Singleton()` | Double-checked locking with `volatile` |

```java
// File: topics/level-3/CompoundAtomicDemo.java

// ConcurrentHashMap — atomic compound operations
ConcurrentHashMap<String, AtomicLong> counters = new ConcurrentHashMap<>();

// WRONG: race between check and put
if (!counters.containsKey(key)) counters.put(key, new AtomicLong(0));
counters.get(key).incrementAndGet();

// RIGHT: atomic compound operation
counters.computeIfAbsent(key, k -> new AtomicLong(0)).incrementAndGet();
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A critical section is code accessing shared mutable state that must execute atomically. Race conditions occur when the critical section is unprotected — check-then-act and read-modify-write are the two classic patterns.
2. **Why/when:** Use `synchronized` for simple mutual exclusion. `ReentrantLock` for tryLock/timeout. `ReadWriteLock` for read-heavy caches. `AtomicInteger` for single-variable counters. `ConcurrentHashMap.computeIfAbsent` for atomic compound map operations.
3. **Example:** `balance >= amount` then `balance -= amount` — two separate operations. Without synchronisation, two threads can both pass the check and both subtract, overdrawing the account.
4. **Gotcha/tradeoff:** `synchronized` is simple but can't timeout or be interrupted. `ReentrantLock` is more flexible but you MUST unlock in `finally`. `AtomicInteger` is lock-free but only works for single-variable operations.

**Common pitfalls**
- Synchronising on the wrong object — `synchronized(this)` doesn't protect if different instances are used.
- Forgetting `finally { lock.unlock() }` — lock held forever on exception.
- Using `volatile` for compound operations — `volatile int count; count++` is NOT atomic.
- Synchronising on a non-final field — reassignment changes the lock object.
- Lock granularity too coarse (locking entire method) — reduces throughput.

**Self-check question**
Two threads call `map.computeIfAbsent("key", k -> expensiveCompute())` simultaneously. Does `expensiveCompute()` run once or twice? What if it were `if (!map.containsKey("key")) map.put("key", expensiveCompute())` instead?
