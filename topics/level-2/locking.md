### Topic: Locking — Level 2

> **Related:** [Level 2 — Spring Boot (optimistic locking)](./spring_boot.md) · [Level 2 — ORM Technologies](./orm_technologies.md) · [Level 3 — Deadlocks](../level-3/deadlocks.md)

**Why it matters (Karat angle)**
Concurrent access to shared data is a daily reality in banking. Interviewers ask about optimistic vs pessimistic locking, and distributed locking with Redis — these are core to preventing double-spend and race conditions.

**Core concept**

| | Optimistic Locking | Pessimistic Locking |
|--|-------------------|-------------------|
| Assumption | Conflicts are rare | Conflicts are expected |
| Mechanism | Version check on write | Lock row on read |
| Blocking | ❌ No locks held | ✅ Other transactions wait |
| Throughput | High (no contention) | Lower (locks serialise access) |
| Conflict handling | Detect + retry | Prevent (but risk deadlocks) |
| Use case | Read-heavy, rare writes | Write-heavy, frequent conflicts |

**Optimistic locking with JPA `@Version`:**

```java
// File: topics/level-2/OptimisticLockDemo.java

@Entity
public class Account {
    @Id private Long id;
    private BigDecimal balance;

    @Version
    private Long version;                                 // auto-incremented by JPA
}

// Generated SQL on update:
// UPDATE accounts SET balance = ?, version = version + 1
// WHERE id = ? AND version = ?
//                  ^^^^^^^^^^^ if version changed, 0 rows updated → OptimisticLockException

@Service
public class AccountService {

    @Transactional
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    public void debit(Long id, BigDecimal amount) {
        Account acc = accountRepo.findById(id).orElseThrow();
        acc.setBalance(acc.getBalance().subtract(amount));
        accountRepo.save(acc);
    }
}
```

**Pessimistic locking with JPA:**

```java
// File: topics/level-2/PessimisticLockDemo.java

public interface AccountRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)                 // SELECT ... FOR UPDATE
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);

    @Lock(LockModeType.PESSIMISTIC_READ)                  // SELECT ... FOR SHARE
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdShared(@Param("id") Long id);
}

@Service
public class TransferService {

    @Transactional
    public void transfer(Long from, Long to, BigDecimal amount) {
        // Lock both accounts — always lock in consistent order to avoid deadlocks
        Long first = Math.min(from, to);
        Long second = Math.max(from, to);

        Account a1 = accountRepo.findByIdForUpdate(first).orElseThrow();
        Account a2 = accountRepo.findByIdForUpdate(second).orElseThrow();

        // Debit/credit with exclusive locks held — no other transaction can modify these rows
        if (first.equals(from)) { a1.debit(amount); a2.credit(amount); }
        else { a2.debit(amount); a1.credit(amount); }
    }
}
```

**Distributed locking with Redis (when you have multiple app instances):**

```java
// File: topics/level-2/RedisDistributedLockDemo.java

// Using Redisson (Redis client with distributed locks)
@Service
public class DistributedTransferService {

    private final RedissonClient redisson;

    public void transfer(Long from, Long to, BigDecimal amount) {
        String lockKey = "lock:transfer:" + Math.min(from, to) + ":" + Math.max(from, to);
        RLock lock = redisson.getLock(lockKey);

        try {
            if (lock.tryLock(10, 30, TimeUnit.SECONDS)) { // wait 10s, hold 30s max
                try {
                    accountService.debit(from, amount);
                    accountService.credit(to, amount);
                } finally {
                    lock.unlock();
                }
            } else {
                throw new LockAcquisitionException("Could not acquire lock for transfer");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Lock interrupted", e);
        }
    }
}
```

| | JPA Locking | Distributed Lock (Redis) |
|--|------------|-------------------------|
| Scope | Single database | Multiple app instances + DBs |
| Mechanism | DB row locks / version column | Redis key with TTL |
| Coordination | DB handles it | Your code handles it |
| Use case | Single-DB apps | Microservices, multi-instance |

**What to say in the interview (4-beat answer)**
1. **Definition:** Optimistic locking uses a `@Version` field — detect conflicts on write, retry. Pessimistic locking uses `SELECT FOR UPDATE` — lock rows on read, prevent conflicts. Distributed locking (Redis/Redisson) extends this to multi-instance deployments.
2. **Why/when:** Use optimistic for read-heavy workloads (rare conflicts). Pessimistic for write-heavy or critical financial operations. Distributed for microservices where multiple app instances access the same resources.
3. **Example:** JPA optimistic: `WHERE version = ?` on UPDATE — 0 rows affected → `OptimisticLockException` → retry. Pessimistic: `SELECT FOR UPDATE` holds row lock until transaction commits.
4. **Gotcha/tradeoff:** Pessimistic locking can deadlock if two transactions lock rows in different order. Always lock in a consistent order (e.g., by ascending ID). Distributed locks need TTL to prevent stale locks from blocking forever.

**Common pitfalls**
- Not retrying on `OptimisticLockException` — the client sees 500 instead of a successful retry.
- Locking rows in inconsistent order → deadlocks (A locks row 1 then row 2; B locks row 2 then row 1).
- Distributed lock without TTL → lock held forever if the app crashes.
- Using `@Version` on an entity with bulk update queries — bulk updates bypass JPA version checks.
- Pessimistic lock with long transactions → other transactions timeout waiting.

**Self-check question**
Two users simultaneously edit the same account. User A reads version=1 and changes the name. User B reads version=1 and changes the balance. Both save. With optimistic locking, who succeeds and who fails? With pessimistic locking?
