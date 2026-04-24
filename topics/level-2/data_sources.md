### Topic: Data Sources — Level 2

> **Foundation:** [Level 1 — SQL Integration](../level-1/sql_integration.md) · [Level 1 — Profiles](../level-1/profiles.md)
> **Related:** [Level 2 — ORM Technologies](./orm_technologies.md) · [Level 2 — Custom Connection Pool](./custom_connection_pool.md)

**Why it matters (Karat angle)**
Every Citi service connects to a database. Interviewers ask about DataSource configuration, `@Transactional` nuances, and `readOnly` optimisation. Getting these right is the difference between a service that works and one that performs.

**Core concept**

**Spring Boot auto-configures a DataSource** from `application.properties`:

```properties
# File: src/main/resources/application.properties
spring.datasource.url=jdbc:oracle:thin:@//citi-db:1521/PROD
spring.datasource.username=app_user
spring.datasource.password=${DB_PASS}
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver

# HikariCP pool settings (Spring Boot default pool)
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

**`@Transactional` in depth:**

```java
// File: topics/level-2/TransactionalDetailsDemo.java

@Service
public class AccountService {

    // Default: REQUIRED propagation, rolls back on RuntimeException
    @Transactional
    public void transfer(Long from, Long to, BigDecimal amount) {
        accountRepo.debit(from, amount);
        accountRepo.credit(to, amount);
    }

    // Read-only: optimises DB and JPA behaviour
    @Transactional(readOnly = true)
    public AccountDto findById(Long id) {
        // JPA: skips dirty checking (no need to detect changes)
        // DB: may route to read replica
        // Hibernate: FlushMode set to MANUAL — no writes allowed
        return accountRepo.findById(id).map(this::toDto).orElseThrow();
    }

    // Explicit rollback control
    @Transactional(rollbackFor = Exception.class)          // roll back on ALL exceptions (including checked)
    public void processWithChecked() throws IOException {
        // Without rollbackFor, checked exceptions DON'T trigger rollback
    }

    // No rollback for specific exception
    @Transactional(noRollbackFor = EmailException.class)
    public void transferWithNotification(Long from, Long to, BigDecimal amount) {
        accountRepo.debit(from, amount);
        accountRepo.credit(to, amount);
        emailService.notify(from);                        // if this throws EmailException, txn still commits
    }

    // Timeout
    @Transactional(timeout = 5)                           // seconds — rolls back if exceeds
    public void longRunningOperation() { /* ... */ }
}
```

**`readOnly = true` benefits:**

| Layer | Optimisation |
|-------|-------------|
| **JPA/Hibernate** | Skips dirty checking; FlushMode = MANUAL |
| **JDBC** | `Connection.setReadOnly(true)` — hint to driver |
| **Database** | May route to read replica; optimise query plan |
| **Spring** | No flush at transaction end — faster |

**Custom DataSource bean:**

```java
// File: topics/level-2/CustomDataSourceConfig.java

@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("app.datasource")
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:oracle:thin:@//citi-db:1521/PROD");
        ds.setUsername("app_user");
        ds.setMaximumPoolSize(20);
        ds.setMinimumIdle(5);
        ds.setConnectionTestQuery("SELECT 1 FROM DUAL");
        ds.setLeakDetectionThreshold(60000);              // log warning if connection not returned in 60s
        return ds;
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Boot auto-configures a HikariCP DataSource from properties. `@Transactional` manages transaction boundaries — `readOnly=true` optimises read operations by skipping dirty checks and hinting the DB driver.
2. **Why/when:** Always use `readOnly=true` for read-only service methods — it improves performance and enables read-replica routing. Configure pool size based on `cores * 2 + spindles`.
3. **Example:** `@Transactional(readOnly = true)` sets JPA FlushMode to MANUAL and JDBC `Connection.setReadOnly(true)` — Hibernate skips dirty checking, and the DB can optimise the query plan.
4. **Gotcha/tradeoff:** `@Transactional` only rolls back on `RuntimeException` by default — checked exceptions (e.g., `IOException`) do NOT trigger rollback. Use `rollbackFor = Exception.class` to catch all.

**Common pitfalls**
- Writing data in a `readOnly = true` transaction — may silently fail or throw depending on JPA provider.
- Not setting `rollbackFor` with checked exceptions — transaction commits even on error.
- Forgetting HikariCP's `maxLifetime` — DB-side connection timeouts break stale connections.
- `@Transactional` on `private` methods — Spring proxy can't intercept.
- Self-invocation (calling `this.readMethod()` from `writeMethod()`) — `readOnly` not applied.

**Self-check question**
A `@Transactional` method calls `accountRepo.save(acc)` then throws a checked `IOException`. Does the save persist to the database? What if you add `rollbackFor = Exception.class`?
