### Topic: Custom Exceptions — Level 2

> **Foundation:** [Level 1 — Exception Handling](../level-1/exception_handling.md)

**Why it matters (Karat angle)**
Every Citi microservice needs custom exceptions for domain-specific error handling. Interviewers check whether you create meaningful exception hierarchies or just throw `RuntimeException("something went wrong")`. A senior answer includes exception chaining, HTTP status mapping, and when checked vs unchecked.

**Core concept**

L1 covered the exception hierarchy and try-catch-finally. L2 is about **designing** your own exceptions.

**Checked vs unchecked — when to use which:**

| | Checked (`extends Exception`) | Unchecked (`extends RuntimeException`) |
|--|-------------------------------|---------------------------------------|
| Compiler enforces | ✅ Must catch or declare `throws` | ❌ No enforcement |
| Caller must handle | Yes | No (but should) |
| Use when | Recoverable conditions (retry, fallback) | Programming errors, unrecoverable failures |
| Example | `InsufficientFundsException` | `InvalidAccountIdException` |
| Modern preference | Declining — verbose | Preferred in Spring/microservices |

**Spring convention:** Use unchecked exceptions. Spring's `@RestControllerAdvice` catches them and maps to HTTP status codes.

**Working code example**

```java
// File: topics/level-2/CustomExceptionDemo.java

// ---------- Base exception for the domain ----------
public class PaymentException extends RuntimeException {

    private final String errorCode;

    public PaymentException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public PaymentException(String errorCode, String message, Throwable cause) {
        super(message, cause);                           // exception chaining — preserves root cause
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// ---------- Specific exceptions extend the base ----------
public class InsufficientFundsException extends PaymentException {
    private final BigDecimal available;
    private final BigDecimal requested;

    public InsufficientFundsException(BigDecimal available, BigDecimal requested) {
        super("INSUFFICIENT_FUNDS",
              String.format("Available: %s, Requested: %s", available, requested));
        this.available = available;
        this.requested = requested;
    }

    public BigDecimal getAvailable() { return available; }
    public BigDecimal getRequested() { return requested; }
}

public class AccountNotFoundException extends PaymentException {
    public AccountNotFoundException(Long accountId) {
        super("ACCOUNT_NOT_FOUND", "Account not found: " + accountId);
    }
}

public class DuplicateTransactionException extends PaymentException {
    public DuplicateTransactionException(String txnRef) {
        super("DUPLICATE_TXN", "Transaction already exists: " + txnRef);
    }
}
```

**Mapping exceptions to HTTP responses:**

```java
// File: topics/level-2/ExceptionHandlerDemo.java

@RestControllerAdvice
public class PaymentExceptionHandler {

    @ExceptionHandler(AccountNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(AccountNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }

    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientFunds(InsufficientFundsException ex) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)             // 422
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }

    @ExceptionHandler(DuplicateTransactionException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateTransactionException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)                          // 409
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }

    @ExceptionHandler(PaymentException.class)
    public ResponseEntity<ErrorResponse> handleGenericPayment(PaymentException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }
}

record ErrorResponse(String code, String message) {}
```

**Exception chaining — always preserve the root cause:**

```java
// File: topics/level-2/ExceptionChainingDemo.java

public class PaymentService {

    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        try {
            Account from = accountRepo.findById(fromId)
                .orElseThrow(() -> new AccountNotFoundException(fromId));
            accountRepo.debit(from, amount);
        } catch (DataAccessException ex) {
            // WRONG: throw new PaymentException("DB_ERROR", ex.getMessage());
            //   → loses the stack trace of the original DB exception

            // RIGHT: chain the cause
            throw new PaymentException("DB_ERROR", "Transfer failed", ex);
            //   → ex.getCause() returns the original DataAccessException
            //   → full stack trace preserved in logs
        }
    }
}
```

**Edge case: checked exception in a lambda**

```java
// File: topics/level-2/CheckedInLambdaDemo.java

// Problem: Function<T,R> doesn't declare throws
// list.stream().map(s -> parseJson(s));                 // COMPILE ERROR if parseJson throws IOException

// Solution 1: Wrap in unchecked
list.stream()
    .map(s -> {
        try { return parseJson(s); }
        catch (IOException e) { throw new UncheckedIOException(e); }
    })
    .toList();

// Solution 2: Create a checked functional interface
@FunctionalInterface
interface CheckedFunction<T, R> {
    R apply(T t) throws Exception;
}

// Solution 3: Utility wrapper
static <T, R> Function<T, R> unchecked(CheckedFunction<T, R> fn) {
    return t -> {
        try { return fn.apply(t); }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}

list.stream().map(unchecked(s -> parseJson(s))).toList();
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Custom exceptions extend `RuntimeException` (unchecked, preferred in Spring) or `Exception` (checked). They carry domain-specific context — error codes, affected entity IDs, amounts.
2. **Why/when:** Generic exceptions (`RuntimeException("error")`) are useless in production logs. Custom exceptions let `@RestControllerAdvice` map them to specific HTTP status codes and structured error responses.
3. **Example:** `InsufficientFundsException` carries `available` and `requested` amounts. The handler maps it to 422 with a JSON body `{ "code": "INSUFFICIENT_FUNDS", "message": "Available: 50, Requested: 200" }`.
4. **Gotcha/tradeoff:** Always chain the root cause — `new CustomException("msg", cause)`. Losing the original exception's stack trace makes debugging impossible. Also: checked exceptions in lambdas don't compile — wrap them or use an `unchecked()` utility.

**Common pitfalls**
- `throw new RuntimeException(e.getMessage())` — loses the stack trace; use `new RuntimeException("context", e)`.
- Too many custom exception classes — one base + 3-5 specifics per domain is enough.
- Checked exceptions in stream lambdas — won't compile; wrap in unchecked.
- Exposing internal exception details in API responses — security risk; map to generic messages externally, log details internally.
- Not implementing `serialVersionUID` on custom exceptions — they're `Serializable` (inherited from `Throwable`).

**Self-check question**
You have `ServiceException extends RuntimeException` and `NotFoundException extends ServiceException`. In `@RestControllerAdvice`, you have handlers for both. A `NotFoundException` is thrown — which handler runs? What if you only have the `ServiceException` handler?
