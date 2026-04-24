### Topic: Stored Procedures — Level 2

> **Foundation:** [Level 1 — SQL Integration](../level-1/sql_integration.md)

**Why it matters (Karat angle)**
Citi legacy systems have critical business logic in stored procedures. Interviewers ask this to see if you've called procs from Java — not just written SQL queries. Knowing `SimpleJdbcCall`, JPA `@NamedStoredProcedureQuery`, and OUT parameter handling is expected.

**Core concept**

L1 briefly showed stored procedure calls. L2 covers the full spectrum — IN/OUT/INOUT parameters, result sets from procs, and error handling.

**Three approaches in Spring:**

| Approach | Best for | Complexity |
|----------|---------|:----------:|
| `JdbcTemplate.call()` | Simple procs, one-off calls | Low |
| `SimpleJdbcCall` | Structured calls, parameter metadata | Medium |
| JPA `@NamedStoredProcedureQuery` | JPA-centric apps | Medium |

**Working code example**

```java
// File: topics/level-2/StoredProcDemo.java

// ---------- Approach 1: SimpleJdbcCall ----------
@Repository
public class InterestRepository {

    private final SimpleJdbcCall calculateInterest;
    private final SimpleJdbcCall getAccountSummary;

    public InterestRepository(DataSource dataSource) {
        this.calculateInterest = new SimpleJdbcCall(dataSource)
            .withProcedureName("sp_calculate_interest")
            .declareParameters(
                new SqlParameter("p_account_id", Types.BIGINT),           // IN
                new SqlOutParameter("p_interest", Types.DECIMAL),         // OUT
                new SqlOutParameter("p_status", Types.VARCHAR)            // OUT
            );

        this.getAccountSummary = new SimpleJdbcCall(dataSource)
            .withProcedureName("sp_account_summary")
            .returningResultSet("accounts",                               // named result set
                (rs, rowNum) -> new AccountSummary(
                    rs.getLong("id"),
                    rs.getString("name"),
                    rs.getBigDecimal("balance")
                ));
    }

    // IN + OUT parameters
    public InterestResult calculateInterest(Long accountId) {
        Map<String, Object> result = calculateInterest.execute(
            new MapSqlParameterSource("p_account_id", accountId)
        );
        return new InterestResult(
            (BigDecimal) result.get("p_interest"),
            (String) result.get("p_status")
        );
    }

    // Proc that returns a result set
    @SuppressWarnings("unchecked")
    public List<AccountSummary> getAccountSummary(String region) {
        Map<String, Object> result = getAccountSummary.execute(
            new MapSqlParameterSource("p_region", region)
        );
        return (List<AccountSummary>) result.get("accounts");
    }
}
```

```java
// File: topics/level-2/JpaStoredProcDemo.java

// ---------- Approach 2: JPA @NamedStoredProcedureQuery ----------
@Entity
@NamedStoredProcedureQuery(
    name = "Account.calculateInterest",
    procedureName = "sp_calculate_interest",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN,  name = "p_account_id", type = Long.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "p_interest",    type = BigDecimal.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "p_status",      type = String.class)
    }
)
public class Account { /* ... */ }

// Call from repository:
@Repository
public class AccountRepository {

    @PersistenceContext
    private EntityManager em;

    public BigDecimal calculateInterest(Long accountId) {
        StoredProcedureQuery query = em.createNamedStoredProcedureQuery("Account.calculateInterest");
        query.setParameter("p_account_id", accountId);
        query.execute();
        return (BigDecimal) query.getOutputParameterValue("p_interest");
    }
}
```

```java
// File: topics/level-2/JdbcCallableDemo.java

// ---------- Approach 3: Raw CallableStatement (for edge cases) ----------
@Repository
public class LegacyProcRepository {

    private final JdbcTemplate jdbc;

    public LegacyProcRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    public BigDecimal calculateInterest(Long accountId) {
        return jdbc.execute((ConnectionCallback<BigDecimal>) conn -> {
            try (CallableStatement cs = conn.prepareCall("{call sp_calculate_interest(?, ?)}")) {
                cs.setLong(1, accountId);                         // IN
                cs.registerOutParameter(2, Types.DECIMAL);        // OUT
                cs.execute();
                return cs.getBigDecimal(2);
            }
        });
    }
}
```

**Error handling with stored procedures:**
```java
// File: topics/level-2/ProcErrorHandling.java

// Procs may set error codes instead of throwing SQL exceptions
Map<String, Object> result = procCall.execute(params);
String status = (String) result.get("p_status");
if ("ERROR".equals(status)) {
    String errorMsg = (String) result.get("p_error_message");
    throw new PaymentException("PROC_ERROR", errorMsg);
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Stored procedures are pre-compiled SQL routines stored in the database. Java calls them via `SimpleJdbcCall` (Spring JDBC), `@NamedStoredProcedureQuery` (JPA), or raw `CallableStatement`.
2. **Why/when:** Banking legacy systems keep critical business logic in procs (interest calculation, EOD batch). Java calls them with IN/OUT parameters and handles result sets or error codes.
3. **Example:** `SimpleJdbcCall` with `SqlOutParameter` — call `sp_calculate_interest` with account ID, get back interest amount and status as OUT parameters.
4. **Gotcha/tradeoff:** Stored procedures move business logic into the database — harder to test, version, and deploy than Java code. But they're faster for bulk data operations and already exist in legacy systems. Don't rewrite them unless there's a migration mandate.

**Common pitfalls**
- Not registering OUT parameters — `NullPointerException` when reading results.
- Mismatched SQL types (`Types.DECIMAL` vs `Types.NUMERIC`) — silent data corruption or `SQLException`.
- Forgetting to handle proc-level error codes — assuming the call succeeded just because no SQL exception was thrown.
- Testing procs in isolation but not the Java-to-proc integration — parameter ordering bugs are common.

**Self-check question**
A stored procedure has an INOUT parameter — you pass an account ID (IN) and it returns the updated balance in the same parameter (OUT). How would you declare and handle this with `SimpleJdbcCall`?
