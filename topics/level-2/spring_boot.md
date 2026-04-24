### Topic: Spring Boot (Advanced Features) — Level 2

> **Foundation:** [Level 1 — Spring Boot Fundamentals](../level-1/spring_boot_fundamentals.md) · [Level 1 — Spring Boot Annotations](../level-1/spring_boot_annotations.md)
> **Advanced:** [Level 3 — Spring Boot (custom error codes, scheduling, transactional errors)](../level-3/spring_boot.md)

**Why it matters (Karat angle)**
L1 covered auto-configuration and basics. L2 covers the features you use daily in a production Citi service: JPA/Hibernate, transactions, optimistic locking, caching, and Actuator. Interviewers expect fluency with all of these.

**Core concept**

**JPA/Hibernate in Spring Boot:**

```java
// File: topics/level-2/JpaAdvancedDemo.java

@Entity
@Table(name = "accounts")
public class Account {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 20)
    private String accountNumber;

    @Column(precision = 15, scale = 2)
    private BigDecimal balance;

    @Version                                              // optimistic locking
    private Long version;

    @OneToMany(mappedBy = "account", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Transaction> transactions = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private AccountStatus status;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

**Optimistic locking with `@Version`:**
```java
// File: topics/level-2/OptimisticLockingDemo.java

@Service
public class AccountService {

    @Transactional
    public void debit(Long accountId, BigDecimal amount) {
        Account acc = accountRepo.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));

        acc.setBalance(acc.getBalance().subtract(amount));
        accountRepo.save(acc);
        // JPA generates: UPDATE accounts SET balance=?, version=version+1 WHERE id=? AND version=?
        // If another thread changed the version → OptimisticLockException → transaction rolls back
    }
}

// Handling the conflict:
@RestControllerAdvice
public class LockingHandler {
    @ExceptionHandler(OptimisticLockException.class)
    public ResponseEntity<ErrorResponse> handle(OptimisticLockException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("CONCURRENT_MODIFICATION", "Data was modified by another user. Please retry."));
    }
}
```

**Transactions in depth:**

| Propagation | Behaviour |
|------------|-----------|
| `REQUIRED` (default) | Join existing txn, or create new one |
| `REQUIRES_NEW` | Suspend current txn, create new one |
| `SUPPORTS` | Join if exists, otherwise run non-transactional |
| `NOT_SUPPORTED` | Suspend current txn, run non-transactional |
| `MANDATORY` | Must have existing txn, else throw |
| `NEVER` | Must NOT have existing txn, else throw |

```java
// File: topics/level-2/TransactionPropagationDemo.java

@Service
public class TransferService {

    @Transactional                                        // REQUIRED by default
    public void transfer(Long from, Long to, BigDecimal amount) {
        accountService.debit(from, amount);
        accountService.credit(to, amount);
        auditService.logTransfer(from, to, amount);       // must succeed with transfer
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)   // separate txn
    public void logTransfer(Long from, Long to, BigDecimal amount) {
        // This commits independently — audit log is saved even if the outer txn rolls back
    }
}
```

**Caching with `@Cacheable`:**

```java
// File: topics/level-2/CachingDemo.java

@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("accounts", "exchangeRates");
    }
}

@Service
public class AccountService {

    @Cacheable(value = "accounts", key = "#id")
    public AccountDto findById(Long id) {
        // Only called on cache miss — result stored in cache
        return accountRepo.findById(id).map(this::toDto).orElseThrow();
    }

    @CachePut(value = "accounts", key = "#result.id")
    public AccountDto update(AccountUpdateRequest req) {
        // Always called — result REPLACES cache entry
        Account acc = accountRepo.findById(req.getId()).orElseThrow();
        acc.setBalance(req.getBalance());
        return toDto(accountRepo.save(acc));
    }

    @CacheEvict(value = "accounts", key = "#id")
    public void delete(Long id) {
        accountRepo.deleteById(id);
        // Cache entry removed after method executes
    }

    @CacheEvict(value = "accounts", allEntries = true)
    @Scheduled(fixedRate = 3600000)                       // every hour
    public void clearCache() {
        // Evict all entries periodically
    }
}
```

**Actuator endpoints:**

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, env, loggers
  endpoint:
    health:
      show-details: when-authorized
```

| Endpoint | URL | What it shows |
|----------|-----|-------------|
| `/actuator/health` | Health check (UP/DOWN) | DB, disk, custom checks |
| `/actuator/metrics` | JVM metrics, HTTP stats | `jvm.memory.used`, `http.server.requests` |
| `/actuator/env` | Environment properties | All config (redacted secrets) |
| `/actuator/loggers` | Runtime log level control | Change log level without restart |
| `/actuator/info` | App info | Build version, git commit |

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Boot's advanced features include JPA/Hibernate for ORM, `@Version` for optimistic locking, `@Cacheable` for result caching, and Actuator for production monitoring and health checks.
2. **Why/when:** Optimistic locking prevents concurrent writes from silently overwriting each other. Caching reduces DB load for read-heavy endpoints. Actuator exposes health and metrics for Kubernetes probes and dashboards.
3. **Example:** `@Version` on an entity field — JPA adds `AND version = ?` to every UPDATE. If another transaction modified the row, `OptimisticLockException` is thrown — you retry or return 409 Conflict.
4. **Gotcha/tradeoff:** `@Cacheable` on a method that returns mutable objects — mutating the returned object mutates the cache entry. Return DTOs or immutable objects from cached methods.

**Common pitfalls**
- `@Cacheable` returning entities with lazy-loaded associations — `LazyInitializationException` when accessed outside the transaction. Return DTOs.
- `@Transactional` on a `private` method — Spring proxy can't intercept it; annotation is ignored.
- Forgetting `@EnableCaching` — `@Cacheable` annotations are silently ignored.
- Exposing all Actuator endpoints in production — security risk; limit to `/health` and `/info` publicly.
- Not handling `OptimisticLockException` — the user sees a 500 instead of a meaningful "retry" response.

**Self-check question**
You have a `@Cacheable` method that returns an entity with `@OneToMany(fetch = LAZY)`. A controller calls this method and accesses the lazy collection. What exception do you get on a cache hit, and why? What's the fix?
