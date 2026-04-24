### Topic: ORM Technologies — Level 2

> **Foundation:** [Level 1 — SQL Integration](../level-1/sql_integration.md)
> **Related:** [Level 2 — Spring Boot (JPA/Hibernate)](./spring_boot.md) · [Level 2 — Data Sources](./data_sources.md) · [Level 2 — Locking](./locking.md)

**Why it matters (Karat angle)**
ORM configuration — custom DataSource, TransactionManager, propagation, and isolation levels — is what separates a Spring Boot user from someone who understands the database layer. Interviewers ask "what happens if two transactions read the same row?" to test isolation level knowledge.

**Core concept**

**Custom DataSource and TransactionManager:**

```java
// File: topics/level-2/CustomDataSourceDemo.java

@Configuration
public class DatabaseConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
    }

    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);     // JDBC
        // or: return new JpaTransactionManager(entityManagerFactory); // JPA
    }
}
```

**Multiple DataSources (common at Citi — read replica + write primary):**

```java
// File: topics/level-2/MultiDataSourceDemo.java

@Configuration
public class MultiDbConfig {

    @Bean @Primary
    @ConfigurationProperties("spring.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean @Primary
    public PlatformTransactionManager writeTransactionManager(
            @Qualifier("writeDataSource") DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }

    @Bean
    public PlatformTransactionManager readTransactionManager(
            @Qualifier("readDataSource") DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
}

// Usage:
@Service
public class AccountService {

    @Transactional("writeTransactionManager")
    public void debit(Long id, BigDecimal amount) { /* write DB */ }

    @Transactional(value = "readTransactionManager", readOnly = true)
    public AccountDto findById(Long id) { /* read replica */ }
}
```

**Transaction isolation levels:**

| Level | Dirty read | Non-repeatable read | Phantom read | Performance |
|-------|:----------:|:-------------------:|:------------:|:-----------:|
| `READ_UNCOMMITTED` | ✅ Possible | ✅ Possible | ✅ Possible | Fastest |
| `READ_COMMITTED` | ❌ Prevented | ✅ Possible | ✅ Possible | Default (PostgreSQL, Oracle) |
| `REPEATABLE_READ` | ❌ | ❌ Prevented | ✅ Possible | Default (MySQL) |
| `SERIALIZABLE` | ❌ | ❌ | ❌ Prevented | Slowest |

**Problems explained:**
- **Dirty read:** Reading uncommitted data from another transaction.
- **Non-repeatable read:** Reading the same row twice and getting different values (another txn committed between reads).
- **Phantom read:** Re-running a query and getting different rows (another txn inserted between queries).

```java
// File: topics/level-2/IsolationDemo.java

@Transactional(isolation = Isolation.READ_COMMITTED)     // most common in banking
public void transferFunds(Long from, Long to, BigDecimal amount) {
    // At READ_COMMITTED:
    // - Can see committed changes from other transactions
    // - Same SELECT may return different values if re-run (non-repeatable read)
    // - New rows from other transactions may appear (phantom read)
}

@Transactional(isolation = Isolation.SERIALIZABLE)        // strongest — use for critical operations
public void calculateInterest(Long accountId) {
    // At SERIALIZABLE:
    // - Fully isolated — as if transactions run one at a time
    // - Potential for lock contention and deadlocks
}
```

**Propagation behaviour (recap with detailed examples):**

```java
// File: topics/level-2/PropagationDemo.java

@Service
public class OrderService {

    @Transactional                                        // REQUIRED — creates txn
    public void createOrder(Order order) {
        orderRepo.save(order);
        paymentService.processPayment(order);             // REQUIRED → joins this txn
        auditService.logOrder(order);                     // REQUIRES_NEW → separate txn
    }
}

@Service
public class PaymentService {

    @Transactional(propagation = Propagation.REQUIRED)    // joins caller's txn
    public void processPayment(Order order) {
        // If this throws, the ENTIRE outer txn rolls back
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)   // NEW txn — independent
    public void logOrder(Order order) {
        // Commits independently — audit is saved even if outer txn rolls back
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** ORM maps Java objects to database tables. Spring Boot auto-configures DataSource, EntityManager, and TransactionManager. Custom configuration supports multiple data sources, specific isolation levels, and propagation control.
2. **Why/when:** Multiple data sources for read/write splitting. Isolation levels control concurrent access — `READ_COMMITTED` for most operations, `SERIALIZABLE` for financial calculations. Propagation controls transaction boundaries across service calls.
3. **Example:** `REQUIRES_NEW` for audit logging — it commits independently, so the audit record exists even if the outer business transaction rolls back.
4. **Gotcha/tradeoff:** Higher isolation = more locking = lower throughput. `SERIALIZABLE` prevents all anomalies but causes deadlocks under high concurrency. `READ_COMMITTED` is the pragmatic default for most banking operations.

**Common pitfalls**
- Using default isolation without knowing what it is — varies by DB (Postgres: READ_COMMITTED, MySQL: REPEATABLE_READ).
- `REQUIRES_NEW` in a method called internally (self-invocation) — proxy is bypassed, new transaction is NOT created.
- Not specifying `readOnly = true` for read operations — misses DB optimisations (skip dirty checking, use read replica).
- Multiple DataSources without `@Primary` — Spring doesn't know which to auto-wire; startup fails.

**Self-check question**
Transaction A reads account balance ($1000) at isolation `READ_COMMITTED`. Transaction B deducts $200 and commits. Transaction A re-reads the balance. What value does it see? What if isolation were `REPEATABLE_READ`?
