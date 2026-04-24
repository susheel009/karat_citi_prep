### Topic: Custom Connection Pool — Level 2

> **Advanced:** [Level 3 — Custom Connection Pool (thread-safe, monitoring)](../level-3/custom_connection_pool.md)
> **Related:** [Level 2 — Multithreading](./multithreading_in_java.md) · [Level 1 — Spring Framework Fundamentals](../level-1/spring_framework_fundamentals.md)

**Why it matters (Karat angle)**
Connection pooling is critical for banking app performance — every Citi service uses HikariCP or a similar pool. Interviewers ask "design a connection pool" to test your understanding of resource management, thread safety, and why opening a new connection per request is expensive.

**Core concept**

**Why pool connections?**

| Approach | Cost per request | Concurrent support |
|----------|:---------------:|:------------------:|
| New connection each time | ~50–200ms (TCP handshake + auth) | Poor (OS limit ~1000 connections) |
| Connection pool | ~0.1ms (borrow from pool) | Good (reuse M connections for N requests) |

A pool pre-creates a fixed number of connections and lends them to threads on demand.

**Design elements:**
1. **Fixed pool of connections** — pre-allocated at startup.
2. **Borrow / return** — thread takes a connection, uses it, returns it.
3. **Blocking when exhausted** — wait or timeout when all connections are in use.
4. **Validation** — check connection health before lending.

**Basic implementation:**

```java
// File: topics/level-2/SimpleConnectionPool.java

public class SimpleConnectionPool {

    private final BlockingQueue<Connection> pool;
    private final String url, user, password;
    private final int maxSize;

    public SimpleConnectionPool(String url, String user, String password, int maxSize)
            throws SQLException {
        this.url = url;
        this.user = user;
        this.password = password;
        this.maxSize = maxSize;
        this.pool = new ArrayBlockingQueue<>(maxSize);

        // Pre-populate the pool
        for (int i = 0; i < maxSize; i++) {
            pool.add(createConnection());
        }
    }

    private Connection createConnection() throws SQLException {
        return DriverManager.getConnection(url, user, password);
    }

    // Borrow a connection (blocks if pool empty)
    public Connection borrowConnection(long timeoutMs) throws InterruptedException, SQLException {
        Connection conn = pool.poll(timeoutMs, TimeUnit.MILLISECONDS);
        if (conn == null) {
            throw new SQLException("Connection pool exhausted — timeout after " + timeoutMs + "ms");
        }
        // Validate before returning
        if (conn.isClosed()) {
            conn = createConnection();                    // replace dead connection
        }
        return conn;
    }

    // Return a connection to the pool
    public void returnConnection(Connection conn) {
        if (conn != null) {
            pool.offer(conn);                             // non-blocking return
        }
    }

    // Shutdown — close all connections
    public void shutdown() throws SQLException {
        Connection conn;
        while ((conn = pool.poll()) != null) {
            conn.close();
        }
    }
}
```

**Usage pattern:**
```java
// File: topics/level-2/PoolUsageDemo.java

SimpleConnectionPool pool = new SimpleConnectionPool(
    "jdbc:oracle:thin:@//citi-db:1521/PROD", "app", "secret", 20
);

// In a service method:
Connection conn = null;
try {
    conn = pool.borrowConnection(5000);                  // 5-second timeout
    PreparedStatement ps = conn.prepareStatement("SELECT * FROM accounts WHERE id = ?");
    ps.setLong(1, accountId);
    ResultSet rs = ps.executeQuery();
    // ... process results
} catch (Exception e) {
    // handle error
} finally {
    pool.returnConnection(conn);                         // ALWAYS return, even on error
}
```

**Production pools — HikariCP (what Citi actually uses):**

```java
// File: topics/level-2/HikariConfigDemo.java

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:oracle:thin:@//citi-db:1521/PROD");
config.setUsername("app_user");
config.setPassword("${DB_PASS}");

// Pool sizing
config.setMaximumPoolSize(20);                           // max connections
config.setMinimumIdle(5);                                // min idle connections

// Timeouts
config.setConnectionTimeout(30_000);                     // wait for connection (ms)
config.setIdleTimeout(600_000);                          // close idle connections after 10 min
config.setMaxLifetime(1_800_000);                        // recycle connections every 30 min

// Validation
config.setConnectionTestQuery("SELECT 1 FROM DUAL");     // Oracle health check

HikariDataSource ds = new HikariDataSource(config);

// Spring Boot auto-configures HikariCP if it's on the classpath:
// spring.datasource.hikari.maximum-pool-size=20
// spring.datasource.hikari.minimum-idle=5
```

**Pool sizing formula (rule of thumb):**
```
optimal pool size = number of CPU cores * 2 + effective spindle count
```
For a 4-core server with SSD: `4 * 2 + 1 = 9` connections. More connections ≠ more throughput — too many means thread contention on the DB side.

**What to say in the interview (4-beat answer)**
1. **Definition:** A connection pool pre-creates and reuses a fixed set of database connections. Threads borrow from the pool, use the connection, and return it — avoiding the 50–200ms cost of creating a new connection per request.
2. **Why/when:** Every production Java service uses one — HikariCP is the default in Spring Boot. Pool sizing, timeouts, and validation are critical for stability under load.
3. **Example:** A basic pool uses `BlockingQueue<Connection>` — `poll(timeout)` for borrowing (blocks when empty), `offer()` for returning. HikariCP adds health checks, connection recycling, and leak detection.
4. **Gotcha/tradeoff:** Pool too small → threads wait for connections → latency spikes. Pool too large → DB overwhelmed with connections → slower for everyone. HikariCP's recommendation: start with `cores * 2 + spindles`.

**Common pitfalls**
- Not returning connections on exception — wrap in try-finally or try-with-resources.
- Pool too large (50+ connections) — overwhelms the database with concurrent queries.
- No connection validation — a stale/dead connection gets returned, causing `SQLException`.
- No timeout on borrow — thread blocks forever if pool is exhausted.
- Forgetting `maxLifetime` — connections that live too long get killed by database-side timeouts.

**Self-check question**
Your connection pool has 10 connections and 15 threads are trying to borrow. What happens to threads 11–15? How does the pool's timeout setting affect behaviour?
