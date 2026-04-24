### Topic: Spring Framework Fundamentals — Level 1

> **Advanced coverage:** [Level 2 — Spring (IoC Container)](../level-2/spring.md) · [Level 2 — Servlets (Dispatcher Servlet)](../level-2/servlets.md) · [Level 3 — Spring MVC](../level-3/spring_mvc.md)

**Why it matters (Karat angle)**
Spring is the backbone of every Citi Java service. Interviewers expect you to explain MVC, IoC/DI, and the data-access layer (JDBC template vs JPA) — not just say "Spring handles it." Knowing the layered architecture signals real project experience.

**Core concept**

Spring Framework is a **dependency injection (DI) container** that wires objects together at runtime. Everything else — MVC, JDBC, AOP — builds on that core.

**Three pillars you need to know:**

| Pillar | What it does | Key class / annotation |
|--------|-------------|----------------------|
| **IoC / DI** | Container creates and injects beans; you don't `new` them | `ApplicationContext`, `@Autowired`, `@Component` |
| **Spring MVC** | Request → DispatcherServlet → Controller → View/JSON | `@Controller`, `@RequestMapping`, `@ResponseBody` |
| **Data Access** | Abstracts JDBC boilerplate; integrates ORM | `JdbcTemplate`, `JpaRepository`, `@Transactional` |

**IoC / Dependency Injection — the mental model**

Without DI:
```java
OrderService service = new OrderService(new OrderRepository(new DataSource(...))); // you build the graph
```
With DI:
```java
@Service
public class OrderService {
    private final OrderRepository repo;
    @Autowired  // Spring resolves this at startup
    public OrderService(OrderRepository repo) { this.repo = repo; }
}
```
The container owns the object graph. You declare what you need; Spring supplies it. This makes testing trivial — swap the real repo for a mock.

**Spring MVC request lifecycle (simplified)**

```
HTTP request
  → DispatcherServlet (front controller — single entry point)
    → HandlerMapping (which controller method?)
      → Controller method executes
        → ViewResolver / @ResponseBody (render response)
          → HTTP response
```

Every `@RestController` method bypasses the ViewResolver and writes JSON directly via `HttpMessageConverter` (Jackson by default).

**Data access — JDBC Template vs JPA**

| | `JdbcTemplate` | Spring Data JPA (`JpaRepository`) |
|--|----------------|----------------------------------|
| SQL control | You write raw SQL | Generated from method names or `@Query` |
| Mapping | Manual `RowMapper` | Automatic via `@Entity` annotations |
| Performance tuning | Full control | Need `@Query` or native queries for complex SQL |
| When to use | Legacy DBs, complex reporting queries | Standard CRUD, rapid development |

**Working code example**

```java
// File: topics/level-1/SpringJdbcDemo.java
// --- JDBC Template: running a SELECT, executing an UPDATE ---

@Repository
public class AccountRepository {

    private final JdbcTemplate jdbc;

    public AccountRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    // SELECT — reading a result set
    public List<Account> findByStatus(String status) {
        return jdbc.query(
            "SELECT id, name, balance FROM accounts WHERE status = ?",
            (rs, rowNum) -> new Account(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getBigDecimal("balance")
            ),
            status
        );
    }

    // UPDATE — executing a write
    public int deactivate(long accountId) {
        return jdbc.update(
            "UPDATE accounts SET status = 'INACTIVE' WHERE id = ?",
            accountId
        );
    }
}
```

```java
// File: topics/level-1/SpringJpaDemo.java
// --- JPA: same operations, zero SQL ---

@Entity
@Table(name = "accounts")
public class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private BigDecimal balance;
    private String status;
    // getters, setters
}

public interface AccountRepository extends JpaRepository<Account, Long> {
    List<Account> findByStatus(String status);           // derived query
    @Modifying @Query("UPDATE Account a SET a.status = 'INACTIVE' WHERE a.id = :id")
    int deactivate(@Param("id") Long id);                // custom JPQL
}
```

**Edge case: JdbcTemplate with stored procedure call**
```java
// File: topics/level-1/StoredProcDemo.java
SimpleJdbcCall call = new SimpleJdbcCall(dataSource)
    .withProcedureName("sp_calculate_interest");

Map<String, Object> result = call.execute(
    new MapSqlParameterSource("account_id", 42)
);
BigDecimal interest = (BigDecimal) result.get("interest_amount");
```

**Real-world use cases**
- **`JdbcTemplate`** — Citi legacy systems with hand-tuned SQL, batch inserts for EOD settlement, calling stored procedures.
- **Spring Data JPA** — new microservices with standard CRUD; rapid prototyping; pagination built-in via `Pageable`.
- **Spring MVC** — every REST endpoint you'll build; the `@RestController` + `@Service` + `@Repository` layered architecture is the expected answer in interviews.

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Framework is a DI container that manages the object lifecycle and wires dependencies. Spring MVC is its web layer; Spring JDBC / JPA is its data-access layer.
2. **Why/when:** DI makes code testable (swap real DB for mock), MVC standardises request handling, JdbcTemplate eliminates JDBC boilerplate while keeping SQL control.
3. **Example:** A `@Service` declares a constructor dependency on a `@Repository`; Spring resolves it at startup. The repo uses `JdbcTemplate.query()` with a `RowMapper` lambda to map rows to POJOs.
4. **Gotcha/tradeoff:** JPA is faster to code but hides SQL — in performance-critical banking queries (EOD batch, reporting), `JdbcTemplate` with hand-tuned SQL often wins.

**Common pitfalls**
- Field injection (`@Autowired` on a field) — works but makes unit testing harder; prefer constructor injection.
- Forgetting `@Transactional` on service methods that do multiple writes — partial commits on failure.
- Using JPA for everything — N+1 query problem; for complex joins, drop to `@Query` with native SQL or use `JdbcTemplate`.
- Not closing `ResultSet`/`Connection` manually when using raw JDBC — `JdbcTemplate` handles this; raw JDBC does not.

**Self-check question**
You have a `@Service` that calls two `@Repository` methods — one debit, one credit. The credit fails. What happens if there's no `@Transactional`? What changes if you add it?
