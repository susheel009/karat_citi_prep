### Topic: Streams (Terminal Operations) — Level 3

> **Foundation:** [Level 1 — Streams](../level-1/streams.md) · [Level 2 — Streams (groupingBy, reduce, pipeline)](../level-2/streams.md)

**Why it matters (Karat angle)**
L2 covered `map`/`filter`/`collect`/`reduce`/`groupingBy`. L3 covers `findFirst`/`findAny`, `Optional` chaining, `orElse` vs `orElseGet` vs `orElseThrow`, and exception handling in streams — the exact edge cases interviewers test.

**Core concept**

**Find operations and Optional:**

| Method | Returns | Short-circuits? | Use case |
|--------|---------|:---------------:|---------|
| `findFirst()` | `Optional<T>` — first element in encounter order | ✅ | Deterministic result (ordered stream) |
| `findAny()` | `Optional<T>` — any element | ✅ | Performance (parallel streams) |
| `anyMatch(pred)` | `boolean` | ✅ | "Does any element match?" |
| `allMatch(pred)` | `boolean` | ✅ | "Do all elements match?" |
| `noneMatch(pred)` | `boolean` | ✅ | "Does no element match?" |

```java
// File: topics/level-3/StreamFindDemo.java

List<Transaction> txns = getTransactions();

// findFirst — deterministic order
Optional<Transaction> firstLarge = txns.stream()
    .filter(t -> t.getAmount().compareTo(new BigDecimal("10000")) > 0)
    .findFirst();

// findAny — faster in parallel
Optional<Transaction> anyLarge = txns.parallelStream()
    .filter(t -> t.getAmount().compareTo(new BigDecimal("10000")) > 0)
    .findAny();

// anyMatch — boolean check, short-circuits
boolean hasFraud = txns.stream()
    .anyMatch(t -> "FRAUD".equals(t.getStatus()));

// allMatch
boolean allSettled = txns.stream()
    .allMatch(t -> "SETTLED".equals(t.getStatus()));
```

**`Optional` — `orElse` vs `orElseGet` vs `orElseThrow`:**

| Method | Argument | Evaluates when? | Use |
|--------|---------|:---------------:|-----|
| `orElse(T)` | Default value | **Always** (even if Optional is present) | Constant defaults |
| `orElseGet(Supplier<T>)` | Lambda/supplier | Only if empty | Expensive computation |
| `orElseThrow(Supplier<X>)` | Exception supplier | Only if empty | Fail-fast |
| `or(Supplier<Optional<T>>)` | Alternative Optional | Only if empty | Fallback chain |

```java
// File: topics/level-3/OptionalDemo.java

// orElse — ALWAYS evaluates the default (even if value is present!)
Account account = accountRepo.findById(id)
    .orElse(createDefaultAccount());                     // createDefaultAccount() runs ALWAYS

// orElseGet — only evaluates if empty (lazy)
Account account2 = accountRepo.findById(id)
    .orElseGet(() -> createDefaultAccount());            // only runs if empty

// orElseThrow — throw if empty
Account account3 = accountRepo.findById(id)
    .orElseThrow(() -> new AccountNotFoundException(id));

// or — try another Optional
Account account4 = primaryRepo.findById(id)
    .or(() -> secondaryRepo.findById(id))                // fallback to secondary
    .orElseThrow(() -> new AccountNotFoundException(id));

// map + orElse chaining
String email = accountRepo.findById(id)
    .map(Account::getOwner)
    .map(Owner::getEmail)
    .orElse("unknown@citi.com");

// filter + ifPresent
accountRepo.findById(id)
    .filter(a -> a.getBalance().compareTo(BigDecimal.ZERO) > 0)
    .ifPresent(a -> process(a));
```

**Exception handling in streams:**

```java
// File: topics/level-3/StreamExceptionDemo.java

// Problem: checked exceptions don't work in lambdas
// list.stream().map(s -> parseJson(s));  // COMPILE ERROR if parseJson throws IOException

// --- Solution 1: Wrapper function ---
static <T, R> Function<T, R> unchecked(CheckedFunction<T, R> fn) {
    return t -> {
        try { return fn.apply(t); }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}

List<Account> accounts = jsonStrings.stream()
    .map(unchecked(json -> objectMapper.readValue(json, Account.class)))
    .toList();

// --- Solution 2: Collect errors instead of throwing ---
record Result<T>(T value, Exception error) {
    static <T> Result<T> success(T value) { return new Result<>(value, null); }
    static <T> Result<T> failure(Exception e) { return new Result<>(null, e); }
}

List<Result<Account>> results = jsonStrings.stream()
    .map(json -> {
        try { return Result.success(objectMapper.readValue(json, Account.class)); }
        catch (Exception e) { return Result.<Account>failure(e); }
    })
    .toList();

List<Account> successes = results.stream().filter(r -> r.error() == null).map(Result::value).toList();
List<Exception> failures = results.stream().filter(r -> r.error() != null).map(Result::error).toList();

// --- Solution 3: flatMap to skip errors ---
List<Account> accounts2 = jsonStrings.stream()
    .flatMap(json -> {
        try { return Stream.of(objectMapper.readValue(json, Account.class)); }
        catch (Exception e) { log.warn("Parse failed: {}", json); return Stream.empty(); }
    })
    .toList();
```

**Advanced collectors — `toMap` edge case:**

```java
// File: topics/level-3/ToMapEdgeCase.java

// toMap throws IllegalStateException on duplicate keys by default
Map<String, Account> byNumber = accounts.stream()
    .collect(Collectors.toMap(Account::getAccountNumber, Function.identity()));
// If two accounts have the same number → IllegalStateException

// Fix: merge function
Map<String, Account> byNumberSafe = accounts.stream()
    .collect(Collectors.toMap(
        Account::getAccountNumber,
        Function.identity(),
        (existing, replacement) -> existing              // keep first on conflict
    ));
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `findFirst()`/`findAny()` return `Optional`, short-circuiting the stream. `orElse` always evaluates its argument; `orElseGet` evaluates lazily; `orElseThrow` fails fast.
2. **Why/when:** Use `findFirst` when order matters, `findAny` for parallel performance. Use `orElseThrow` for mandatory lookups (account by ID). Use `orElseGet` when the default is expensive to compute.
3. **Example:** `accountRepo.findById(id).orElseThrow(() -> new AccountNotFoundException(id))` — fails fast with a specific exception. `primaryRepo.findById(id).or(() -> secondaryRepo.findById(id))` — fallback chain.
4. **Gotcha/tradeoff:** `orElse(expensiveCall())` always evaluates the argument — even when the Optional has a value. Use `orElseGet(() -> expensiveCall())` to avoid the wasted computation. Also: checked exceptions in stream lambdas require a wrapper or try-catch.

**Common pitfalls**
- `orElse` with expensive side-effect — always runs, even when not needed.
- `findFirst` on an unordered parallel stream — no performance benefit over `findAny`.
- `Optional.get()` without `isPresent()` — throws `NoSuchElementException`. Prefer `orElseThrow`.
- `Collectors.toMap` with duplicate keys — throws `IllegalStateException` unless you provide a merge function.
- Swallowing exceptions in stream lambdas with empty catch — silent data loss.

**Self-check question**
`accountRepo.findById(id).orElse(new Account())` vs `accountRepo.findById(id).orElseGet(Account::new)` — when the account IS found, how many Account objects are created in each case? Why does it matter?
