# 04 — Exception Handling

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## Rapid-Fire Q&A

### Q1: Checked vs Unchecked exceptions — hierarchy?
**A:** `Throwable` → `Error` (JVM-level: `OutOfMemoryError`, `StackOverflowError` — don't catch) and `Exception`. `Exception` → Checked (`IOException`, `SQLException`) + `RuntimeException` (unchecked: `NullPointerException`, `IllegalArgumentException`, `IllegalStateException`).

### Q2: When would you create a custom exception?
**A:** When built-in exceptions don't convey domain meaning. E.g., `InsufficientFundsException extends RuntimeException`. Include context: account ID, attempted amount, available balance. Extend `RuntimeException` (unchecked) unless caller must handle it.

### Q3: Try-with-resources — how does it work?
**A:** Resources declared in `try(...)` must implement `AutoCloseable`. Compiler generates `finally` with `close()` calls in reverse declaration order. If both try body and `close()` throw, the close exception is suppressed (attached via `addSuppressed`).

### Q4: `finally` vs `try-with-resources`?
**A:** `try-with-resources` is strictly better for resource cleanup — handles the suppressed-exception edge case correctly, is less verbose. Use `finally` only for non-resource cleanup (resetting state, logging). Never use `finally` to return a value — it overrides the try block's return.

### Q5: What happens if you catch `Exception` broadly?
**A:** Catches all checked AND unchecked exceptions — too broad. Hides bugs (catches `NullPointerException` you didn't expect). Catch the most specific exception possible. If you must catch broadly, re-throw or log with the full stack trace.

### Q6: `throw` vs `throws`?
**A:** `throw` — actually throw an exception instance (`throw new IllegalArgumentException("bad")`). `throws` — method signature declaration that checked exceptions may be thrown (`void read() throws IOException`).

### Q7: Can you catch multiple exceptions in one block?
**A:** Yes. Multi-catch: `catch (IOException | ParseException e)`. The variable `e` is implicitly `final`. Cannot catch parent and child in same multi-catch (compiler error).

### Q8: What's exception chaining?
**A:** Wrapping a lower-level exception as the cause of a higher-level one: `throw new ServiceException("Failed", ioe)`. Preserves the full stack trace. Accessed via `getCause()`.

### Q9: Best practices for exception handling?
**A:** (1) Catch specific, not broad. (2) Don't swallow exceptions — at minimum log them. (3) Use try-with-resources for all resources. (4) Throw early, catch late. (5) Include context in exception messages. (6) Don't use exceptions for flow control (expensive — fills stack trace).

---

## Can you answer these cold?

- [ ] Exception hierarchy — Throwable → Error / Exception → RuntimeException
- [ ] Checked vs unchecked — when each is appropriate
- [ ] Try-with-resources — AutoCloseable, suppressed exceptions
- [ ] Custom exception — when, how, what to include
- [ ] Multi-catch syntax

[← Back to Index](./00_INDEX.md)
