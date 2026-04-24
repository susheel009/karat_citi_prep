### Topic: DB Indexes — Level 2

> **Related:** [Level 2 — Pagination](./pagination.md) · [Level 2 — ORM Technologies](./orm_technologies.md)

**Why it matters (Karat angle)**
A query that takes 5 seconds without an index takes 5ms with one. Interviewers ask this to check if you've tuned queries in production — not just written them. Knowing index types, when they help, and when they hurt is senior-level knowledge.

**Core concept**

An **index** is a sorted data structure (B-tree, hash, etc.) that speeds up lookups by avoiding full table scans.

| Index type | Structure | Best for | Example |
|-----------|-----------|---------|---------|
| **B-tree** (default) | Balanced tree | Range queries, equality, ORDER BY | `WHERE balance > 1000` |
| **Hash** | Hash table | Equality only | `WHERE status = 'ACTIVE'` |
| **Composite** | B-tree on multiple columns | Multi-column WHERE/ORDER | `WHERE status = ? AND date > ?` |
| **Unique** | B-tree with uniqueness constraint | Primary keys, unique fields | `account_number` |
| **Covering** | Index includes all needed columns | Avoid table lookup (index-only scan) | `SELECT name FROM accounts WHERE id = ?` |
| **Partial** (PostgreSQL) | Index on a subset of rows | Selective queries | `WHERE status = 'ACTIVE'` only |

**How a B-tree index works (simplified):**
```
Without index: SELECT * FROM accounts WHERE id = 500
  → Full table scan: check every row → O(n)

With B-tree index on 'id':
  → Tree lookup: root → branch → leaf → pointer to row → O(log n)
```

**Composite index — column order matters:**
```sql
-- Index: CREATE INDEX idx_status_date ON transactions(status, txn_date);

-- ✅ Uses index (leftmost prefix)
SELECT * FROM transactions WHERE status = 'PENDING';

-- ✅ Uses index (both columns)
SELECT * FROM transactions WHERE status = 'PENDING' AND txn_date > '2026-01-01';

-- ❌ Cannot use index (skips leftmost column)
SELECT * FROM transactions WHERE txn_date > '2026-01-01';
-- Must include 'status' to use the composite index
```

**When indexes help vs hurt:**

| Helps | Hurts |
|-------|-------|
| `WHERE` clauses (equality, range) | `INSERT` — index must be updated |
| `ORDER BY` / `GROUP BY` | `UPDATE` on indexed columns — re-index |
| `JOIN` conditions | `DELETE` — index entries removed |
| Unique constraints | Storage — each index adds disk space |
| Covering queries (index-only scan) | Too many indexes → write-heavy tables slow down |

**Working code example — creating indexes:**
```sql
-- File: topics/level-2/db_indexes.sql

-- Single column
CREATE INDEX idx_account_status ON accounts(status);

-- Composite (order matters!)
CREATE INDEX idx_txn_status_date ON transactions(status, txn_date DESC);

-- Unique
CREATE UNIQUE INDEX idx_account_number ON accounts(account_number);

-- Partial (PostgreSQL only)
CREATE INDEX idx_active_accounts ON accounts(name) WHERE status = 'ACTIVE';

-- Covering (include non-indexed columns to avoid table lookup)
CREATE INDEX idx_txn_covering ON transactions(account_id) INCLUDE (amount, status);
```

**EXPLAIN PLAN — how to check if your index is used:**
```sql
EXPLAIN ANALYZE SELECT * FROM transactions WHERE status = 'PENDING' AND txn_date > '2026-01-01';

-- Look for:
-- "Index Scan using idx_txn_status_date" → ✅ index used
-- "Seq Scan on transactions"             → ❌ full table scan
```

**Spring/JPA index annotations:**
```java
// File: topics/level-2/JpaIndexDemo.java

@Entity
@Table(name = "transactions", indexes = {
    @Index(name = "idx_status", columnList = "status"),
    @Index(name = "idx_status_date", columnList = "status, txnDate DESC"),
    @Index(name = "idx_account", columnList = "accountId")
})
public class Transaction {
    @Id @GeneratedValue private Long id;
    private String status;
    private LocalDateTime txnDate;
    private Long accountId;
    private BigDecimal amount;
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Indexes are sorted data structures (typically B-trees) that speed up lookups from O(n) full table scans to O(log n) tree traversals. They trade write performance and storage for read speed.
2. **Why/when:** Add indexes on columns used in WHERE, JOIN, ORDER BY. Don't add on write-heavy, low-cardinality columns (e.g., boolean fields). Composite indexes must match query patterns — leftmost prefix rule.
3. **Example:** A composite index on `(status, txn_date)` supports `WHERE status = 'PENDING' AND txn_date > X` but NOT `WHERE txn_date > X` alone — the leftmost column must appear in the query.
4. **Gotcha/tradeoff:** Over-indexing slows writes — each INSERT/UPDATE/DELETE must update every relevant index. Use EXPLAIN PLAN to verify indexes are actually used; the query optimizer may choose a full scan if selectivity is low.

**Common pitfalls**
- Adding indexes without checking EXPLAIN PLAN — the optimizer might not use them.
- Wrong column order in composite indexes — `(status, date)` vs `(date, status)` produce different query coverage.
- Indexing low-cardinality columns (boolean, status with 3 values) — B-tree index isn't selective enough; full scan may be faster.
- Too many indexes on a write-heavy table — inserts/updates become slow.
- Not indexing foreign key columns — JOINs become full table scans.

**Self-check question**
You have a table with 10 million rows and a composite index on `(region, created_date)`. Query: `SELECT * FROM orders WHERE created_date > '2026-01-01' ORDER BY region`. Does the index help? Why or why not?
