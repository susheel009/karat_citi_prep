### Topic: Pagination — Level 2

> **Foundation:** [Level 1 — SQL Integration](../level-1/sql_integration.md)
> **Related:** [Level 2 — Spring Boot (JPA pagination)](./spring_boot.md)

**Why it matters (Karat angle)**
Every API that returns lists needs pagination. Interviewers check whether you know OFFSET/LIMIT, keyset pagination, and Spring Data's `Pageable` — and whether you understand why OFFSET-based pagination degrades on large datasets.

**Core concept**

| Approach | SQL pattern | Performance | Use case |
|----------|-----------|:-----------:|---------|
| **OFFSET/LIMIT** | `LIMIT n OFFSET m` | Degrades on large offsets | Small datasets, simple UIs |
| **Keyset (cursor)** | `WHERE id > :lastId LIMIT n` | Constant time | Large datasets, infinite scroll |
| **Spring Data Pageable** | Abstraction over both | Depends on impl | Standard Spring Boot APIs |

**OFFSET/LIMIT — simple but slow at scale:**
```sql
-- Page 1 (items 1-20)
SELECT * FROM transactions ORDER BY id LIMIT 20 OFFSET 0;

-- Page 500 (items 9981-10000)
SELECT * FROM transactions ORDER BY id LIMIT 20 OFFSET 9980;
-- DB scans and discards 9980 rows just to return 20 → O(offset + limit)
```

**Keyset pagination — constant performance:**
```sql
-- Page 1
SELECT * FROM transactions ORDER BY id LIMIT 20;
-- Returns IDs 1..20. Client remembers lastId = 20

-- Next page
SELECT * FROM transactions WHERE id > 20 ORDER BY id LIMIT 20;
-- Index seek → O(limit), not O(offset + limit)
```

**Spring Data `Pageable`:**

```java
// File: topics/level-2/PaginationDemo.java

// Repository
public interface TransactionRepository extends JpaRepository<Transaction, Long> {

    Page<Transaction> findByStatus(String status, Pageable pageable);

    @Query("SELECT t FROM Transaction t WHERE t.amount > :min ORDER BY t.timestamp DESC")
    Page<Transaction> findLargeTransactions(@Param("min") BigDecimal min, Pageable pageable);
}

// Controller
@GetMapping("/transactions")
public Page<Transaction> list(
        @RequestParam(defaultValue = "SETTLED") String status,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "timestamp,desc") String[] sort) {

    Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "timestamp"));
    return transactionRepo.findByStatus(status, pageable);
}

// Response includes: content[], totalElements, totalPages, number, size, sort
```

**Keyset pagination in Spring:**
```java
// File: topics/level-2/KeysetPaginationDemo.java

// No built-in Spring support — implement manually
@GetMapping("/transactions")
public List<Transaction> list(
        @RequestParam(required = false) Long afterId,
        @RequestParam(defaultValue = "20") int size) {

    if (afterId == null) {
        return transactionRepo.findTopN(size);
    }
    return transactionRepo.findAfterId(afterId, size);
}

// Repository with native query
@Query(value = "SELECT * FROM transactions WHERE id > :afterId ORDER BY id LIMIT :size",
       nativeQuery = true)
List<Transaction> findAfterId(@Param("afterId") Long afterId, @Param("size") int size);
```

**What to say in the interview (4-beat answer)**
1. **Definition:** OFFSET/LIMIT is simple but degrades at high page numbers (DB scans discarded rows). Keyset/cursor pagination uses a WHERE clause on the last seen ID — constant O(limit) performance.
2. **Why/when:** Use OFFSET for admin UIs with small datasets. Use keyset for mobile infinite scroll, large transaction histories, public APIs.
3. **Example:** Spring Data's `Pageable` with `PageRequest.of(page, size, sort)` generates OFFSET queries. For keyset, use a native query with `WHERE id > :lastId LIMIT :size`.
4. **Gotcha/tradeoff:** OFFSET pagination can return duplicates or skip rows if data is inserted/deleted between page fetches. Keyset pagination is stable but doesn't support "jump to page 50."

**Common pitfalls**
- Using OFFSET for large datasets (millions of rows) — page 10,000 is painfully slow.
- Not including an `ORDER BY` with pagination — results are non-deterministic without it.
- Returning `Page<Entity>` instead of `Page<DTO>` — exposes internal entity structure in the API.
- Forgetting `totalElements` query cost — `Page<>` runs a COUNT query; use `Slice<>` if you don't need total count.

**Self-check question**
Your API returns paginated transactions. A new transaction is inserted between page 2 and page 3 fetches. With OFFSET, what problem occurs? Does keyset pagination have the same issue?
