### Topic: Exception Handling — Level 1

> **Advanced coverage:** [Level 2 — Custom Exceptions](../level-2/custom_exceptions.md)

**Why it matters (Karat angle)**
Exception handling is where good developers are separated from code-dumpers. Interviewers look for: correct hierarchy knowledge, checked vs unchecked judgement, try-with-resources fluency, and avoidance of the two cardinal sins (`catch(Exception)` swallow, re-throwing without context).

**Core concept**

**The hierarchy:**

```
Throwable
├── Error             (JVM-level, usually unrecoverable: OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException  (unchecked — programming bugs: NullPointerException, IllegalArgumentException)
    └── (everything else) (checked — recoverable conditions: IOException, SQLException)
```

**Checked vs unchecked — when to use which**

| | Checked (`extends Exception`) | Unchecked (`extends RuntimeException`) |
|-|-----------------------------|----------------------------------------|
| Must declare/catch? | ✅ yes | ❌ no |
| Represents | Recoverable external condition | Programming error or contract violation |
| Examples | `IOException`, `SQLException`, `InterruptedException` | `NullPointerException`, `IllegalArgumentException`, `IllegalStateException` |
| Compiler enforces? | ✅ | ❌ |

**Rule of thumb:** checked exceptions should be rare. Modern Java style (Spring, Kotlin-style) leans towards unchecked — checked exceptions pollute every method signature up the call stack. Use checked when the caller genuinely has a sensible recovery path (retry, fall back); use unchecked for everything else.

**try-catch-finally semantics**

| Clause | When it runs |
|--------|--------------|
| `try` block | Always (until an exception or normal completion) |
| `catch` | When a matching exception is thrown |
| `finally` | Always — even after return or uncaught throw. Used for cleanup. |

**try-with-resources (Java 7+):** any object implementing `AutoCloseable` can be declared in the `try(...)` header; it's closed automatically when the block exits, even on exception. Replaces the verbose manual `finally { resource.close(); }` dance.

**Multi-catch (Java 7+):** `catch (IOException | SQLException e)` — handle several types the same way without repetition.

**Exception chaining:** always preserve the original cause when wrapping: `throw new ServiceException("Processing failed", e);` — otherwise you lose the stack trace at the real root cause.

**Mental model:** `try` is the attempt, `catch` is the contingency, `finally` is the cleanup — and `try-with-resources` is the modern sugar that removes the cleanup boilerplate.

**Real-world use cases**
- **REST controller:** let exceptions bubble to a `@ControllerAdvice` handler that maps them to HTTP status codes.
- **Database call:** catch `SQLException`, wrap in a `DataAccessException` (Spring does this), re-throw with context.
- **File I/O:** `try (BufferedReader r = Files.newBufferedReader(path))` — no manual close.
- **Retry logic:** catch a narrow exception (e.g., `TimeoutException`), log, retry up to N times.
- **Custom domain exceptions:** `InsufficientFundsException extends RuntimeException` — semantic, caller decides how to handle.

**Working code example**
```java
// File: topics/level-1/ExceptionHandlingDemo.java
import java.io.*;
import java.nio.file.*;
import java.sql.SQLException;

public class ExceptionHandlingDemo {

    // ---------- try-with-resources (Java 7+) ----------
    static String readFile(Path path) throws IOException {
        try (BufferedReader reader = Files.newBufferedReader(path)) {
            return reader.readLine();
            // reader.close() called automatically, even on exception
        }
    }

    // ---------- multi-catch + chaining ----------
    static void process(Path path) {
        try {
            String line = readFile(path);
            System.out.println(line);
        } catch (IOException | SecurityException e) {
            // Multi-catch: handle two unrelated exceptions identically
            throw new RuntimeException("Failed to process " + path, e);   // preserve cause
        }
    }

    // ---------- custom unchecked exception ----------
    static class InsufficientFundsException extends RuntimeException {
        public InsufficientFundsException(String msg) { super(msg); }
    }

    static void withdraw(long balance, long amount) {
        if (amount > balance) {
            throw new InsufficientFundsException(
                "Requested " + amount + " but balance is " + balance);
        }
        System.out.println("Withdrew " + amount);
    }

    // ---------- finally runs even on return ----------
    static int withFinally() {
        try {
            return 1;
        } finally {
            System.out.println("finally ran");     // prints BEFORE return completes
        }
    }

    public static void main(String[] args) {
        try {
            withdraw(100, 200);
        } catch (InsufficientFundsException e) {
            System.out.println("Caught: " + e.getMessage());
        }

        System.out.println(withFinally());         // prints "finally ran" then 1
    }
}
```

**Edge case: exception swallow — the cardinal sin**
```java
// NEVER do this
try {
    riskyOperation();
} catch (Exception e) {
    // silent — stack trace lost forever
}

// Minimum acceptable
try {
    riskyOperation();
} catch (Exception e) {
    log.error("riskyOperation failed", e);       // always log the exception
    throw new ProcessingException("Op failed", e);  // or rethrow with cause
}
```

**Edge case: try-with-resources multi-resource**
```java
try (InputStream in = Files.newInputStream(src);
     OutputStream out = Files.newOutputStream(dst)) {
    in.transferTo(out);
    // Closed in REVERSE declaration order; both closed even if one throws
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java exceptions descend from `Throwable` → `Error` (JVM-level, usually fatal) and `Exception` → `RuntimeException` (unchecked, programming bugs) vs the rest (checked, recoverable external conditions).
2. **Why/when:** Checked forces handling at compile time — use only when a caller has a meaningful recovery. Unchecked for all other failure modes. Always preserve the cause when wrapping.
3. **Example:** `try-with-resources` eliminates manual `finally { close(); }` blocks — the `BufferedReader` is closed automatically, even if an exception is thrown.
4. **Gotcha/tradeoff:** `catch (Exception e)` without logging or rethrowing hides bugs. At minimum, log with the exception (`log.error(msg, e)` not `log.error(e.getMessage())`) to preserve the stack trace.

**Common pitfalls**
- Swallowing exceptions silently (empty catch block) — bug becomes invisible.
- `catch (Exception e) { throw new RuntimeException(e.getMessage()); }` — loses the stack trace; always pass `e` as the cause.
- Catching `Throwable` — you catch `Error` too (`OutOfMemoryError`, `StackOverflowError`), which you usually can't recover from.
- Declaring too many checked exceptions — consider wrapping in an unchecked domain exception at the boundary.
- Forgetting `finally` runs even when `try` returns — a `return` in `finally` overrides a `return` in `try`.

**Self-check question**
What's the difference between `catch(IOException e) { throw new ServiceException(e.getMessage()); }` and `catch(IOException e) { throw new ServiceException("context", e); }`? Which do you always prefer and why?
