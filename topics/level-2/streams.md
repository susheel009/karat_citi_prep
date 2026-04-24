### Topic: Streams — Level 2

> **Foundation:** [Level 1 — Lambda Expressions](../level-1/lambda_expressions.md) · [Level 1 — Collections](../level-1/collections.md)
> **Advanced:** [Level 3 — Streams (findFirst/findAny, orElse, exception handling)](../level-3/streams.md)

**Why it matters (Karat angle)**
Streams are the single most-tested Java 8+ feature. L2 interviewers expect fluent pipelines — map, filter, collect, flatMap, groupingBy, reduce — and the ability to refactor imperative loops into declarative streams on the spot. Saying "I know streams" then writing a for-loop is a red flag.

**Core concept**

A stream is a **lazy, single-use pipeline** over data. No data is processed until a **terminal operation** triggers execution.

```
Source → intermediate ops (lazy) → terminal op (triggers execution) → result
```

| Operation type | Examples | Lazy? | Returns |
|---------------|---------|:-----:|---------|
| **Intermediate** | `filter`, `map`, `flatMap`, `sorted`, `distinct`, `peek`, `limit`, `skip` | ✅ | `Stream<T>` |
| **Terminal** | `collect`, `forEach`, `reduce`, `count`, `toList`, `toArray`, `anyMatch`, `noneMatch` | ❌ | Result / void |

**map — transform each element**
```java
// File: topics/level-2/StreamsDemo.java
List<String> names = List.of("Alice", "Bob", "Charlie");
List<String> upper = names.stream()
    .map(String::toUpperCase)
    .toList();                                          // [ALICE, BOB, CHARLIE]
```

**filter — keep matching elements**
```java
List<Transaction> large = transactions.stream()
    .filter(t -> t.getAmount().compareTo(new BigDecimal("10000")) > 0)
    .toList();
```

**flatMap — flatten nested structures**
```java
// Each account has a list of transactions
List<Account> accounts = getAccounts();
List<Transaction> allTxns = accounts.stream()
    .flatMap(a -> a.getTransactions().stream())          // Stream<List<Txn>> → Stream<Txn>
    .toList();
```

**collect with Collectors**

| Collector | Result |
|----------|--------|
| `toList()` | `List<T>` (Java 16+ shorthand: `.toList()`) |
| `toSet()` | `Set<T>` |
| `toMap(keyFn, valueFn)` | `Map<K,V>` |
| `joining(", ")` | Single `String` |
| `groupingBy(classifier)` | `Map<K, List<V>>` |
| `partitioningBy(predicate)` | `Map<Boolean, List<V>>` |
| `counting()` | `Long` |
| `summarizingInt/Long/Double` | Stats (count, sum, min, avg, max) |

```java
// File: topics/level-2/CollectorsDemo.java

// groupingBy — group transactions by status
Map<String, List<Transaction>> byStatus = transactions.stream()
    .collect(Collectors.groupingBy(Transaction::getStatus));
// { "SETTLED": [...], "PENDING": [...] }

// groupingBy with downstream collector — count per status
Map<String, Long> countByStatus = transactions.stream()
    .collect(Collectors.groupingBy(Transaction::getStatus, Collectors.counting()));
// { "SETTLED": 42, "PENDING": 7 }

// groupingBy with sum — total amount per status
Map<String, BigDecimal> totalByStatus = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::getStatus,
        Collectors.reducing(BigDecimal.ZERO, Transaction::getAmount, BigDecimal::add)
    ));

// toMap — build lookup by ID (throws on duplicate keys!)
Map<Long, Transaction> byId = transactions.stream()
    .collect(Collectors.toMap(Transaction::getId, Function.identity()));

// toMap with merge function — handle duplicates
Map<Long, Transaction> byIdLatest = transactions.stream()
    .collect(Collectors.toMap(
        Transaction::getId,
        Function.identity(),
        (existing, replacement) -> replacement            // keep latest on collision
    ));

// joining — CSV line
String csv = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));         // [Alice, Bob, Charlie]

// partitioningBy — split into two groups
Map<Boolean, List<Transaction>> settled = transactions.stream()
    .collect(Collectors.partitioningBy(t -> "SETTLED".equals(t.getStatus())));
// { true: [settled txns], false: [non-settled txns] }
```

**reduce — fold elements into a single result**
```java
// File: topics/level-2/ReduceDemo.java

// Sum of amounts
BigDecimal total = transactions.stream()
    .map(Transaction::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// Max amount (returns Optional)
Optional<BigDecimal> max = transactions.stream()
    .map(Transaction::getAmount)
    .reduce(BigDecimal::max);
```

**Chaining a real pipeline — banking example**
```java
// File: topics/level-2/StreamPipelineDemo.java

// "Give me the top 5 largest settled transactions from today, sorted descending"
List<Transaction> top5 = transactions.stream()
    .filter(t -> "SETTLED".equals(t.getStatus()))
    .filter(t -> t.getTimestamp().toLocalDate().equals(LocalDate.now()))
    .sorted(Comparator.comparing(Transaction::getAmount).reversed())
    .limit(5)
    .toList();
```

**Edge case: streams are single-use**
```java
// File: topics/level-2/StreamReuseError.java
Stream<String> s = names.stream().filter(n -> n.length() > 3);
s.count();                                              // works
s.toList();                                             // IllegalStateException: stream already operated on
// Fix: create a new stream each time, or use a Supplier<Stream<T>>
Supplier<Stream<String>> supplier = () -> names.stream().filter(n -> n.length() > 3);
supplier.get().count();
supplier.get().toList();
```

**When NOT to use streams**
- Simple iteration with side effects — `for` loop is clearer.
- Performance-critical tight loops — stream overhead (object allocation, lambda captures) matters at scale.
- When you need to break/continue mid-iteration — streams don't support this cleanly (use `takeWhile`/`dropWhile` Java 9+ for some cases).

**What to say in the interview (4-beat answer)**
1. **Definition:** Streams are lazy, single-use pipelines with intermediate (map, filter, flatMap — lazy) and terminal (collect, reduce, count — triggers execution) operations.
2. **Why/when:** They replace imperative loops with declarative pipelines — more readable, composable, and parallelisable. `Collectors.groupingBy` replaces manual map-building loops.
3. **Example:** Grouping transactions by status with a downstream `counting()` collector: `stream().collect(groupingBy(Txn::getStatus, counting()))` — one line replaces a 10-line loop.
4. **Gotcha/tradeoff:** `Collectors.toMap` throws `IllegalStateException` on duplicate keys — always provide a merge function. Streams are single-use — reusing a consumed stream throws `IllegalStateException`.

**Common pitfalls**
- Reusing a stream after a terminal operation — single-use by design.
- `toMap` without a merge function when duplicates are possible — runtime exception.
- Using `peek()` for business logic — it's for debugging only; side effects in streams are fragile.
- Forgetting that `sorted()` is a stateful intermediate operation — it buffers the entire stream.
- `flatMap` returning `null` instead of `Stream.empty()` — NPE.

**Self-check question**
Given a `List<Order>` where each order has a `List<LineItem>`, write a single stream pipeline that returns a `Map<String, BigDecimal>` — product name to total revenue across all orders. Use flatMap, map, and a groupingBy with a reducing downstream.
