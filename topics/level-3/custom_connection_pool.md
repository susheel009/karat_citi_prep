### Topic: Custom Connection Pool (Thread-Safe, Monitoring) — Level 3

> **Foundation:** [Level 2 — Custom Connection Pool (basic design)](../level-2/custom_connection_pool.md)
> **Related:** [Level 3 — Threads](./threads.md) · [Level 3 — Deadlocks](./deadlocks.md)

**Why it matters (Karat angle)**
L2 built a basic pool with `BlockingQueue`. L3 adds production-grade features: thread-safe checkout/return with `Semaphore`, connection validation, leak detection, and monitoring — the features HikariCP provides.

**Core concept**

**Production pool requirements:**
1. Thread-safe borrow/return (concurrent access)
2. Connection validation (dead connection detection)
3. Leak detection (connection not returned in time)
4. Pool statistics (active, idle, waiting)
5. Graceful shutdown (close all connections)

**Full implementation:**

```java
// File: topics/level-3/ProductionConnectionPool.java

public class ProductionConnectionPool implements AutoCloseable {

    private final BlockingDeque<Connection> idle;
    private final Set<Connection> active;
    private final Semaphore permits;
    private final String url, user, password;
    private final int maxSize;
    private final long leakThresholdMs;
    private final Map<Connection, Long> borrowTimestamps;
    private final ScheduledExecutorService leakDetector;
    private volatile boolean closed = false;

    // Monitoring counters
    private final AtomicInteger totalCreated = new AtomicInteger(0);
    private final AtomicLong totalBorrowTimeNs = new AtomicLong(0);
    private final AtomicInteger totalBorrows = new AtomicInteger(0);

    public ProductionConnectionPool(String url, String user, String password,
                                     int maxSize, long leakThresholdMs) throws SQLException {
        this.url = url;
        this.user = user;
        this.password = password;
        this.maxSize = maxSize;
        this.leakThresholdMs = leakThresholdMs;
        this.idle = new LinkedBlockingDeque<>(maxSize);
        this.active = ConcurrentHashMap.newKeySet();
        this.permits = new Semaphore(maxSize, true);       // fair — FIFO ordering
        this.borrowTimestamps = new ConcurrentHashMap<>();

        // Pre-populate min connections
        int minIdle = Math.min(5, maxSize);
        for (int i = 0; i < minIdle; i++) {
            idle.add(createConnection());
        }

        // Leak detection thread
        this.leakDetector = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "pool-leak-detector");
            t.setDaemon(true);
            return t;
        });
        leakDetector.scheduleAtFixedRate(this::detectLeaks, 30, 30, TimeUnit.SECONDS);
    }

    private Connection createConnection() throws SQLException {
        Connection conn = DriverManager.getConnection(url, user, password);
        totalCreated.incrementAndGet();
        return conn;
    }

    public Connection borrow(long timeoutMs) throws SQLException, InterruptedException {
        if (closed) throw new SQLException("Pool is closed");

        long start = System.nanoTime();
        if (!permits.tryAcquire(timeoutMs, TimeUnit.MILLISECONDS)) {
            throw new SQLException("Pool exhausted — waited " + timeoutMs + "ms. " +
                "Active: " + active.size() + ", Idle: " + idle.size());
        }

        try {
            Connection conn = idle.poll();
            if (conn == null || conn.isClosed() || !conn.isValid(2)) {
                if (conn != null) conn.close();            // discard invalid
                conn = createConnection();                 // create fresh
            }

            active.add(conn);
            borrowTimestamps.put(conn, System.currentTimeMillis());
            totalBorrows.incrementAndGet();
            totalBorrowTimeNs.addAndGet(System.nanoTime() - start);
            return conn;
        } catch (Exception e) {
            permits.release();                             // release permit on failure
            throw e;
        }
    }

    public void returnConnection(Connection conn) {
        if (conn == null) return;
        active.remove(conn);
        borrowTimestamps.remove(conn);

        try {
            if (!closed && !conn.isClosed() && conn.isValid(2)) {
                conn.setAutoCommit(true);                  // reset state
                idle.offer(conn);
            } else {
                conn.close();                              // discard bad connection
            }
        } catch (SQLException e) {
            try { conn.close(); } catch (SQLException ignored) {}
        } finally {
            permits.release();
        }
    }

    private void detectLeaks() {
        long now = System.currentTimeMillis();
        borrowTimestamps.forEach((conn, borrowTime) -> {
            long held = now - borrowTime;
            if (held > leakThresholdMs) {
                System.err.println("LEAK WARNING: Connection held for " + held + "ms (threshold: " + leakThresholdMs + "ms)");
            }
        });
    }

    // Monitoring
    public int getActiveCount() { return active.size(); }
    public int getIdleCount() { return idle.size(); }
    public int getWaitingCount() { return maxSize - permits.availablePermits() - active.size(); }
    public int getTotalCreated() { return totalCreated.get(); }
    public double getAvgBorrowTimeMs() {
        int borrows = totalBorrows.get();
        return borrows == 0 ? 0 : (totalBorrowTimeNs.get() / 1_000_000.0) / borrows;
    }

    @Override
    public void close() throws SQLException {
        closed = true;
        leakDetector.shutdown();
        Connection conn;
        while ((conn = idle.poll()) != null) conn.close();
        for (Connection c : active) {
            try { c.close(); } catch (SQLException ignored) {}
        }
    }
}
```

**Usage with try-with-resources:**
```java
// File: topics/level-3/PoolUsagePattern.java

try (Connection conn = pool.borrow(5000)) {
    // auto-returned? No — Connection.close() closes the connection, doesn't return to pool!
    // Need a wrapper:
}

// Proper pattern: wrap Connection to intercept close()
class PooledConnection implements Connection {
    private final Connection delegate;
    private final ProductionConnectionPool pool;

    @Override
    public void close() {
        pool.returnConnection(delegate);                   // return to pool instead of closing
    }
    // delegate all other methods to 'delegate'...
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A production connection pool uses `Semaphore` for thread-safe bounded access, validates connections before lending, detects leaks (connections held too long), and provides monitoring (active/idle/wait counts).
2. **Why/when:** This is what HikariCP does internally. Building one tests your understanding of thread safety (`Semaphore`, `ConcurrentHashMap`), resource lifecycle, and failure handling.
3. **Example:** `Semaphore.tryAcquire(timeout)` bounds concurrency. On borrow: validate with `conn.isValid(2)`, discard dead connections, create fresh ones. Leak detector runs every 30s, logs connections held beyond threshold.
4. **Gotcha/tradeoff:** `Connection.close()` in a pooled context should return to pool, not close the TCP connection. HikariCP uses a proxy wrapper. Also: always release the semaphore permit in a `finally` block — leaked permits exhaust the pool.

**Common pitfalls**
- Not releasing the semaphore permit on exception — pool permanently shrinks.
- Not validating connections before returning — handing out a dead connection.
- Not resetting connection state (`autoCommit`, isolation level) on return — contaminates next user.
- Leak detection without action — log the warning AND capture a stack trace of the borrower.

**Self-check question**
Your pool has `maxSize=10`. A thread borrows a connection but never returns it (code path misses the return). What happens? How does the pool detect this? What should it do about it?
