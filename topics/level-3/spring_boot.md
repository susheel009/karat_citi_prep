### Topic: Spring Boot (Custom Error Codes, Scheduling, Transactional Errors) — Level 3

> **Foundation:** [Level 2 — Spring Boot (JPA, caching, Actuator)](../level-2/spring_boot.md) · [Level 2 — Custom Exceptions](../level-2/custom_exceptions.md)

**Why it matters (Karat angle)**
L2 covered JPA, caching, and Actuator. L3 covers the tricky edge cases interviewers love: custom HTTP error codes with `@ResponseStatus`, `@Scheduled` task configuration, and transactional behaviour during errors — specifically which exceptions trigger rollback.

**Core concept**

**Custom HTTP error codes:**

```java
// File: topics/level-3/CustomErrorCodeDemo.java

// --- Approach 1: @ResponseStatus on exception ---
@ResponseStatus(HttpStatus.TOO_MANY_REQUESTS)             // 429
public class RateLimitExceededException extends RuntimeException {
    public RateLimitExceededException(String msg) { super(msg); }
}

@ResponseStatus(HttpStatus.GONE)                           // 410
public class ResourceRetiredException extends RuntimeException {
    public ResourceRetiredException(String msg) { super(msg); }
}

// Thrown from service → Spring automatically returns the annotated status

// --- Approach 2: ResponseEntity for dynamic codes ---
@GetMapping("/accounts/{id}")
public ResponseEntity<?> getAccount(@PathVariable Long id) {
    return accountRepo.findById(id)
        .<ResponseEntity<?>>map(ResponseEntity::ok)
        .orElse(ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", "Account " + id + " not found")));
}

// --- Approach 3: @ExceptionHandler with custom status ---
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .toList();
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)  // 422
            .body(new ErrorResponse("VALIDATION_FAILED", String.join("; ", errors)));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraint(DataIntegrityViolationException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)              // 409
            .body(new ErrorResponse("DUPLICATE", "Record already exists"));
    }
}
```

**Scheduling:**

```java
// File: topics/level-3/SchedulingDemo.java

@Configuration
@EnableScheduling
public class ScheduleConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);                          // concurrent scheduled tasks
        scheduler.setThreadNamePrefix("sched-");
        scheduler.setErrorHandler(t -> log.error("Scheduled task failed", t));
        return scheduler;
    }
}

@Service
public class ScheduledJobs {

    // Fixed rate: every 60 seconds (measured from START of previous execution)
    @Scheduled(fixedRate = 60_000)
    public void pollExternalSystem() {
        log.info("Polling...");
    }

    // Fixed delay: 60 seconds AFTER previous execution COMPLETES
    @Scheduled(fixedDelay = 60_000, initialDelay = 10_000)
    public void processQueue() {
        log.info("Processing queue...");
    }

    // Cron: 8 AM every weekday
    @Scheduled(cron = "0 0 8 * * MON-FRI")
    public void morningReport() {
        log.info("Generating morning report...");
    }

    // Cron: every 5 minutes
    @Scheduled(cron = "0 */5 * * * *")
    public void healthCheck() {
        externalService.ping();
    }

    // Configurable via properties
    @Scheduled(fixedRateString = "${app.job.rate:30000}")
    public void configurableJob() {
        log.info("Rate from config...");
    }
}
```

**Cron expression format:** `second minute hour day-of-month month day-of-week`
```
0 0 8 * * MON-FRI  → 8:00 AM, Mon–Fri
0 */5 * * * *      → every 5 minutes
0 0 0 1 * *        → midnight on the 1st of every month
0 30 9 * * *       → 9:30 AM daily
```

**Transactional error behaviour:**

```java
// File: topics/level-3/TransactionalErrorDemo.java

@Service
public class PaymentService {

    // DEFAULT: rolls back on RuntimeException, NOT on checked Exception
    @Transactional
    public void processA() throws IOException {
        accountRepo.debit(fromId, amount);
        throw new IOException("file error");              // DOES NOT ROLLBACK (checked exception)
        // The debit is committed!
    }

    // Fix: explicitly declare rollback for checked exceptions
    @Transactional(rollbackFor = Exception.class)
    public void processB() throws IOException {
        accountRepo.debit(fromId, amount);
        throw new IOException("file error");              // ROLLS BACK (rollbackFor = Exception)
    }

    // No rollback for a specific runtime exception
    @Transactional(noRollbackFor = EmailDeliveryException.class)
    public void processC() {
        accountRepo.debit(fromId, amount);
        emailService.send(receipt);                       // EmailDeliveryException → no rollback
        // Debit is committed even though email failed
    }

    // NESTED: inner failure → partial rollback to savepoint
    @Transactional
    public void processD() {
        accountRepo.debit(fromId, amount);
        try {
            auditService.log(txnDetails);                 // @Transactional(propagation = NESTED)
        } catch (Exception e) {
            // Inner NESTED txn rolls back to savepoint, outer txn continues
            log.warn("Audit failed but payment proceeds");
        }
    }
}
```

| Exception type | Default rollback? | How to change |
|---------------|:-----------------:|-------------|
| `RuntimeException` / `Error` | ✅ Rolls back | `noRollbackFor = MyException.class` |
| Checked `Exception` | ❌ Commits | `rollbackFor = Exception.class` |
| `@Transactional(rollbackFor = Exception.class)` | ✅ All exceptions | — |

**Edge case: `@Transactional` and self-invocation:**

```java
// File: topics/level-3/SelfInvocationDemo.java

@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        orderRepo.save(order);
        this.sendNotification(order);                     // self-call — proxy BYPASSED!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        // This does NOT run in a new transaction because 'this' bypasses the proxy
    }

    // Fix 1: Inject self
    // @Autowired private OrderService self;
    // self.sendNotification(order); → goes through proxy

    // Fix 2: Separate into another @Service
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Custom error codes use `@ResponseStatus` on exceptions, `ResponseEntity<>` for dynamic status, or `@ExceptionHandler` for centralised mapping. `@Scheduled` supports `fixedRate`, `fixedDelay`, and cron. `@Transactional` only rolls back on `RuntimeException` by default.
2. **Why/when:** Banking APIs need precise error codes (409 for duplicates, 422 for validation, 429 for rate limiting). Scheduled jobs poll external systems, generate reports, run health checks. Transaction rollback rules must be explicit for financial correctness.
3. **Example:** `@Transactional(rollbackFor = Exception.class)` ensures checked exceptions (like `IOException`) also trigger rollback — critical when a file I/O failure should undo a database write.
4. **Gotcha/tradeoff:** Self-invocation bypasses the Spring proxy — `this.method()` won't apply `@Transactional` or `@Scheduled` annotations. Inject the bean into itself or extract to a separate service. Also: `fixedRate` with a slow task can cause overlapping executions.

**Common pitfalls**
- Checked exception in `@Transactional` method → transaction commits despite error. Always use `rollbackFor`.
- `@Scheduled fixedRate` with task longer than the rate → overlapping executions. Use `fixedDelay` or add `@Async`.
- Self-invocation bypassing proxy — `@Transactional`, `@Cacheable`, `@Async` all fail on `this.method()`.
- `@ResponseStatus` on non-exception classes — does nothing. It must be on an exception class thrown from a handler.

**Self-check question**
A `@Transactional` method calls `accountRepo.save()`, then calls a REST API that returns 500, and your code throws `HttpClientErrorException` (extends `RuntimeException`). Does the save roll back? What if the REST call throws `IOException` instead?
