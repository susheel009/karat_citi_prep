### Topic: SQL Integration — Level 1

> **Advanced coverage:** [Level 2 — Stored Procedures](../level-2/stored_procedures.md) · [Level 2 — ORM Technologies](../level-2/orm_technologies.md) · [Level 2 — Data Sources](../level-2/data_sources.md) · [Level 2 — Pagination](../level-2/pagination.md)

**Why it matters (Karat angle)**
Every Citi backend service talks to a relational database. Interviewers expect you to know three paths: raw `JdbcTemplate`, Spring Data JPA repositories, and calling stored procedures. They'll also probe whether you understand `@Transactional`, prepared statements, and SQL injection prevention.

**Core concept**

Spring provides three levels of SQL integration:

| Approach | SQL control | Boilerplate | Use case |
|----------|:-----------:|:-----------:|----------|
| `JdbcTemplate` | Full (you write SQL) | Medium | Complex queries, reports, legacy DBs |
| Spring Data JPA | Minimal (generated from method names) | Low | Standard CRUD, rapid development |
| Stored procedures | Full (DB-side SQL) | Medium | Complex business logic in the DB, legacy systems |

All three handle connection management, parameter binding, and result set mapping for you — you never manually open/close a `Connection`.

**JdbcTemplate — SELECT, UPDATE, INSERT**

```java
// File: topics/level-1/JdbcIntegrationDemo.java

@Repository
public class TransactionRepository {

    private final JdbcTemplate jdbc;

    public TransactionRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    // ---------- SELECT: reading a result set ----------
    public List<Transaction> findByAccountId(Long accountId) {
        return jdbc.query(
            "SELECT id, account_id, amount, txn_date, status FROM transactions WHERE account_id = ?",
            (rs, rowNum) -> new Transaction(
                rs.getLong("id"),
                rs.getLong("account_id"),
                rs.getBigDecimal("amount"),
                rs.getTimestamp("txn_date").toLocalDateTime(),
                rs.getString("status")
            ),
            accountId                                              // binds to ?
        );
    }

    // ---------- SELECT single value ----------
    public BigDecimal getBalance(Long accountId) {
        return jdbc.queryForObject(
            "SELECT balance FROM accounts WHERE id = ?",
            BigDecimal.class,
            accountId
        );
    }

    // ---------- INSERT ----------
    public int insert(Transaction txn) {
        return jdbc.update(
            "INSERT INTO transactions (account_id, amount, txn_date, status) VALUES (?, ?, ?, ?)",
            txn.getAccountId(), txn.getAmount(), txn.getTxnDate(), txn.getStatus()
        );
    }

    // ---------- UPDATE ----------
    public int markSettled(Long txnId) {
        return jdbc.update(
            "UPDATE transactions SET status = 'SETTLED' WHERE id = ? AND status = 'PENDING'",
            txnId
        );
    }

    // ---------- BATCH INSERT (for EOD settlement) ----------
    public int[] batchInsert(List<Transaction> txns) {
        return jdbc.batchUpdate(
            "INSERT INTO transactions (account_id, amount, txn_date, status) VALUES (?, ?, ?, ?)",
            new BatchPreparedStatementSetter() {
                @Override public void setValues(PreparedStatement ps, int i) throws SQLException {
                    Transaction t = txns.get(i);
                    ps.setLong(1, t.getAccountId());
                    ps.setBigDecimal(2, t.getAmount());
                    ps.setTimestamp(3, Timestamp.valueOf(t.getTxnDate()));
                    ps.setString(4, t.getStatus());
                }
                @Override public int getBatchSize() { return txns.size(); }
            }
        );
    }
}
```

**Spring Data JPA — the same, without SQL**

```java
// File: topics/level-1/JpaIntegrationDemo.java

@Entity
@Table(name = "transactions")
public class Transaction {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long accountId;
    private BigDecimal amount;
    private LocalDateTime txnDate;
    private String status;
    // getters, setters
}

public interface TransactionRepository extends JpaRepository<Transaction, Long> {

    // Derived query — Spring generates SQL from method name
    List<Transaction> findByAccountId(Long accountId);

    List<Transaction> findByStatusAndTxnDateAfter(String status, LocalDateTime after);

    // Custom JPQL
    @Query("SELECT t FROM Transaction t WHERE t.accountId = :id AND t.amount > :min")
    List<Transaction> findLargeTransactions(@Param("id") Long accountId,
                                            @Param("min") BigDecimal minAmount);

    // Native SQL (when JPQL isn't enough)
    @Query(value = "SELECT * FROM transactions WHERE status = 'PENDING' AND txn_date < SYSDATE - 7",
           nativeQuery = true)
    List<Transaction> findStaleTransactions();
}
```

**Calling stored procedures from Java**

```java
// File: topics/level-1/StoredProcCallDemo.java

// --- Approach 1: JdbcTemplate ---
@Repository
public class InterestCalculator {

    private final JdbcTemplate jdbc;

    public BigDecimal calculateInterest(Long accountId) {
        return jdbc.queryForObject(
            "CALL sp_calculate_interest(?)",
            BigDecimal.class,
            accountId
        );
    }
}

// --- Approach 2: SimpleJdbcCall (more structured) ---
@Repository
public class InterestCalculatorV2 {

    private final SimpleJdbcCall call;

    public InterestCalculatorV2(DataSource ds) {
        this.call = new SimpleJdbcCall(ds).withProcedureName("sp_calculate_interest");
    }

    public BigDecimal calculateInterest(Long accountId) {
        Map<String, Object> result = call.execute(
            new MapSqlParameterSource("p_account_id", accountId)
        );
        return (BigDecimal) result.get("p_interest");
    }
}

// --- Approach 3: JPA @NamedStoredProcedureQuery ---
@Entity
@NamedStoredProcedureQuery(
    name = "calculateInterest",
    procedureName = "sp_calculate_interest",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "p_account_id", type = Long.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "p_interest", type = BigDecimal.class)
    }
)
public class Account { /* ... */ }
```

**`@Transactional` — the must-know annotation**

```java
// File: topics/level-1/TransactionalDemo.java

@Service
public class TransferService {

    @Transactional                                     // all-or-nothing: rollback on any RuntimeException
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        accountRepo.debit(fromId, amount);             // UPDATE 1
        accountRepo.credit(toId, amount);              // UPDATE 2
        auditRepo.log(fromId, toId, amount);           // INSERT
        // If any of these throw, ALL three roll back
    }
}
```

| `@Transactional` attribute | Default | What it controls |
|---------------------------|---------|-----------------|
| `readOnly` | `false` | Hints DB to optimise for reads (no dirty checks in JPA) |
| `rollbackFor` | `RuntimeException` | Which exceptions trigger rollback |
| `propagation` | `REQUIRED` | Join existing txn or create new one |
| `isolation` | DB default | Read committed, repeatable read, etc. |

**SQL injection prevention:**

```java
// WRONG — string concatenation → SQL injection
String sql = "SELECT * FROM accounts WHERE name = '" + userInput + "'";  // NEVER

// RIGHT — parameterised query → safe
jdbc.query("SELECT * FROM accounts WHERE name = ?", mapper, userInput);  // ALWAYS
```
JdbcTemplate and JPA both use prepared statements — parameters are bound, never interpolated.

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring integrates with SQL databases via `JdbcTemplate` (raw SQL with automatic resource management), Spring Data JPA (generated queries from method names), and stored procedure calls via `SimpleJdbcCall` or JPA annotations.
2. **Why/when:** Use `JdbcTemplate` for complex SQL and legacy DBs. Use Spring Data JPA for standard CRUD — it eliminates boilerplate. Use stored procedures when business logic lives in the DB (common in banking).
3. **Example:** `JdbcTemplate.query()` takes a SQL string, a `RowMapper` lambda, and parameters. It handles connection pooling, parameter binding, and result set mapping — no manual `try/catch/finally` for resource cleanup.
4. **Gotcha/tradeoff:** JPA's generated queries are convenient but can produce inefficient SQL for complex joins — always check the generated SQL with `spring.jpa.show-sql=true` during development. For performance-critical queries, drop to `@Query` with native SQL.

**Common pitfalls**
- String concatenation for SQL parameters — SQL injection vulnerability. Always use `?` placeholders.
- Forgetting `@Transactional` on service methods with multiple writes — partial commits on failure.
- Using `@Transactional` on a `private` method — Spring's proxy-based AOP can't intercept private methods; the annotation is silently ignored.
- `@Transactional(readOnly = true)` on a method that writes — the write may succeed or silently fail depending on the JPA provider.
- Not closing resources with raw JDBC — `JdbcTemplate` handles this, but if you ever use raw `Connection`/`Statement`, always use try-with-resources.

**Self-check question**
You have a service method that reads from DB, calls a remote REST API, then writes to DB. Should the `@Transactional` wrap the entire method or just the DB operations? What's the risk of wrapping the REST call?
