### Topic: Aspect-Oriented Programming (AOP) — Level 3

> **Related:** [Level 2 — Design Patterns (Proxy)](../level-2/design_patterns.md) · [Level 2 — Beans (BeanPostProcessor)](../level-2/beans.md) · [Level 3 — Spring Boot](./spring_boot.md)

**Why it matters (Karat angle)**
AOP is how Spring implements `@Transactional`, `@Cacheable`, `@Secured`, and `@Async`. Interviewers ask "how does @Transactional work?" — the answer is AOP proxies. Knowing aspects, pointcuts, and join points proves you understand Spring's core architecture.

**Core concept**

| AOP Term | Meaning | Example |
|----------|---------|---------|
| **Aspect** | A cross-cutting concern module | Logging, security, transaction management |
| **Join point** | A point in execution (method call, field access) | `AccountService.transfer()` is called |
| **Pointcut** | An expression that selects join points | "All methods in `com.citi.service.*`" |
| **Advice** | Code to run at a join point | Log before method, time after method |
| **Weaving** | Applying aspects to target code | Spring uses runtime proxies (CGLIB) |

**Advice types:**

| Type | Annotation | When it runs |
|------|-----------|-------------|
| **Before** | `@Before` | Before the method |
| **After Returning** | `@AfterReturning` | After successful return |
| **After Throwing** | `@AfterThrowing` | After exception |
| **After (Finally)** | `@After` | Always (like finally) |
| **Around** | `@Around` | Wraps the method (most powerful) |

**Working code example:**

```java
// File: topics/level-3/AopDemo.java

@Aspect
@Component
public class LoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);

    // --- @Before: log method entry ---
    @Before("execution(* com.citi.service.*.*(..))")
    public void logBefore(JoinPoint jp) {
        log.info("→ {}.{}({})",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            Arrays.toString(jp.getArgs()));
    }

    // --- @AfterReturning: log return value ---
    @AfterReturning(pointcut = "execution(* com.citi.service.*.*(..))", returning = "result")
    public void logAfterReturning(JoinPoint jp, Object result) {
        log.info("← {}.{} returned: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            result);
    }

    // --- @AfterThrowing: log exceptions ---
    @AfterThrowing(pointcut = "execution(* com.citi.service.*.*(..))", throwing = "ex")
    public void logException(JoinPoint jp, Exception ex) {
        log.error("✘ {}.{} threw: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            ex.getMessage());
    }

    // --- @Around: measure execution time (most common use case) ---
    @Around("@annotation(com.citi.annotation.Timed)")
    public Object timeExecution(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();                // call the actual method
            return result;
        } finally {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.info("⏱ {}.{} took {}ms",
                pjp.getTarget().getClass().getSimpleName(),
                pjp.getSignature().getName(),
                elapsed);
        }
    }
}
```

**Pointcut expression syntax:**

```java
// File: topics/level-3/PointcutExpressions.java

// --- execution: match method signature ---
@Pointcut("execution(* com.citi.service.*.*(..))")        // any return, any method, any args in service package
@Pointcut("execution(public * com.citi..*Service.*(..))")  // public methods in classes ending with Service
@Pointcut("execution(* com.citi..*.transfer(..))") // any method named 'transfer'

// --- within: match class ---
@Pointcut("within(com.citi.service.*)")                    // all methods in service package (not sub-packages)
@Pointcut("within(com.citi.service..*)")                   // all methods in service package AND sub-packages

// --- @annotation: match by annotation ---
@Pointcut("@annotation(com.citi.annotation.Timed)")        // methods annotated with @Timed
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")

// --- @within: match class annotation ---
@Pointcut("@within(org.springframework.stereotype.Service)")  // all methods in @Service classes

// --- bean: match by bean name ---
@Pointcut("bean(*Service)")                                // beans with names ending in "Service"

// --- Combining ---
@Pointcut("execution(* com.citi.service.*.*(..)) && !execution(* com.citi.service.AuditService.*(..))")
// All service methods EXCEPT AuditService
```

**Custom annotation for AOP:**

```java
// File: topics/level-3/CustomAopAnnotation.java

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timed {
    String value() default "";
}

@Service
public class TransferService {

    @Timed("transfer-operation")
    public void transfer(Long from, Long to, BigDecimal amount) {
        // The @Around advice wraps this method — logs execution time
    }
}
```

**How Spring AOP relates to `@Transactional`:**
```
1. Spring scans for @Transactional annotations
2. BeanPostProcessor wraps the bean in a CGLIB proxy
3. Proxy intercepts method calls:
   - Before: begin transaction (TransactionInterceptor)
   - Proceed: call real method
   - After returning: commit
   - After throwing RuntimeException: rollback
4. This is EXACTLY @Around advice
```

**What to say in the interview (4-beat answer)**
1. **Definition:** AOP separates cross-cutting concerns (logging, transactions, security) from business logic. Aspects define pointcuts (which methods) and advice (what to do: before, after, around).
2. **Why/when:** `@Transactional` is an `@Around` aspect — intercepts method calls to manage transactions. Custom `@Timed` aspects log execution time without cluttering business code. AOP is the mechanism behind Spring's annotation-based magic.
3. **Example:** `@Around("@annotation(Timed)")` wraps any method annotated with `@Timed` — measures `System.nanoTime()` before and after `pjp.proceed()`, logs the duration.
4. **Gotcha/tradeoff:** AOP only works on proxy calls — `this.method()` bypasses the proxy, so aspects don't apply. Also: `@Around` advice must call `pjp.proceed()` — forgetting it silently prevents the method from executing.

**Common pitfalls**
- Forgetting `pjp.proceed()` in `@Around` — method never executes; returns null silently.
- Self-invocation bypassing proxy — `this.method()` doesn't trigger AOP advice.
- Pointcut too broad (`execution(* *.*(..))`) — aspects run on every method, massive performance hit.
- `@Aspect` without `@Component` — Spring doesn't detect it.
- Ordering: multiple aspects on the same method — use `@Order` to control execution order.

**Self-check question**
You have `@Around` logging and `@Around` transaction aspects on the same method. In what order do they execute? How does `@Order` control this? What happens if the logging aspect doesn't call `proceed()`?
