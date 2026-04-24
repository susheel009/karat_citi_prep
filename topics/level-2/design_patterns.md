### Topic: Design Patterns — Level 2

> **Foundation:** [Level 1 — Design Patterns (Singleton, Factory, Adapter, Observer, Decorator, Bridge, Visitor, Template)](../level-1/design_patterns.md)
> **Advanced:** [Level 3 — Design Principles (SOLID, KISS, YAGNI)](../level-3/design_principles.md)

**Why it matters (Karat angle)**
L1 covered eight patterns with code. L2 adds **Façade** and **Proxy** — two patterns Spring uses extensively. Interviewers expect you to identify them in Spring code and explain when each applies.

**Core concept**

L2 adds two structural patterns to the L1 set:

| Pattern | Problem it solves | Spring example |
|---------|-------------------|---------------|
| **Façade** | Simplify a complex subsystem behind one interface | `JdbcTemplate`, `RestTemplate`, `KafkaTemplate` |
| **Proxy** | Control access to an object (add behaviour transparently) | `@Transactional`, `@Cacheable`, AOP proxies |

---

**Façade — simplify a complex API**

The Façade wraps multiple subsystems and exposes one clean interface. The caller doesn't know (or care) about the internal complexity.

```java
// File: topics/level-2/FacadeDemo.java

// Complex subsystems
class AccountValidator { boolean validate(String acctId) { return true; } }
class FraudChecker { boolean check(BigDecimal amount) { return amount.compareTo(new BigDecimal("50000")) < 0; } }
class LedgerService { void record(String from, String to, BigDecimal amount) { /* ... */ } }
class NotificationService { void notify(String acctId, String msg) { /* ... */ } }

// Façade — one method, four subsystems
public class TransferFacade {

    private final AccountValidator validator;
    private final FraudChecker fraud;
    private final LedgerService ledger;
    private final NotificationService notifier;

    public TransferFacade(AccountValidator v, FraudChecker f, LedgerService l, NotificationService n) {
        this.validator = v; this.fraud = f; this.ledger = l; this.notifier = n;
    }

    public void transfer(String from, String to, BigDecimal amount) {
        if (!validator.validate(from) || !validator.validate(to))
            throw new IllegalArgumentException("Invalid account");
        if (!fraud.check(amount))
            throw new SecurityException("Fraud check failed");
        ledger.record(from, to, amount);
        notifier.notify(from, "Debited " + amount);
        notifier.notify(to, "Credited " + amount);
    }
}
// Caller: transferFacade.transfer("A001", "A002", BigDecimal.TEN);
// Doesn't know about validator, fraud, ledger, notifier.
```

**Spring's Façades:**
- `JdbcTemplate` — façade over `Connection`, `PreparedStatement`, `ResultSet`, exception translation.
- `RestTemplate` / `WebClient` — façade over HTTP client, serialisation, error handling.
- `KafkaTemplate` — façade over producer config, serialisation, send callbacks.

---

**Proxy — intercept and add behaviour**

A Proxy wraps the real object and adds cross-cutting behaviour (logging, security, transactions) **transparently** — the caller thinks it's calling the real object.

```java
// File: topics/level-2/ProxyDemo.java

// Interface
interface PaymentService {
    void process(Payment payment);
}

// Real implementation
class PaymentServiceImpl implements PaymentService {
    public void process(Payment payment) {
        System.out.println("Processing payment: " + payment.getAmount());
    }
}

// Static proxy — adds logging
class LoggingPaymentProxy implements PaymentService {

    private final PaymentService real;

    LoggingPaymentProxy(PaymentService real) { this.real = real; }

    @Override
    public void process(Payment payment) {
        System.out.println("LOG: before process");
        long start = System.nanoTime();
        real.process(payment);                            // delegate to real
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.println("LOG: after process (" + elapsed + "ms)");
    }
}

// Usage:
PaymentService service = new LoggingPaymentProxy(new PaymentServiceImpl());
service.process(payment);                                 // caller doesn't know about proxy
```

**Spring's proxies (dynamic — created at runtime):**

```java
// File: topics/level-2/SpringProxyDemo.java

@Service
public class PaymentService {

    @Transactional                                        // Spring wraps this bean in a proxy
    public void transfer(Long from, Long to, BigDecimal amount) {
        accountRepo.debit(from, amount);
        accountRepo.credit(to, amount);
    }
}

// At runtime, Spring creates:
// PaymentService$$EnhancerBySpringCGLIB → proxy
//   → calls TransactionInterceptor.invoke()
//     → begins transaction
//       → calls real PaymentService.transfer()
//     → commits (or rolls back on exception)
```

**Two types of Spring proxies:**

| | JDK Dynamic Proxy | CGLIB Proxy |
|--|-------------------|------------|
| Requires | Target implements an interface | No interface needed |
| Mechanism | `java.lang.reflect.Proxy` | Bytecode generation (subclass) |
| Limitation | Only proxies interface methods | Can't proxy `final` classes/methods |
| Default in Spring Boot | ❌ | ✅ (since Spring Boot 2.0) |

---

**Façade vs Proxy vs Decorator — disambiguating**

| Pattern | Intent | Structural difference |
|---------|--------|----------------------|
| **Façade** | Simplify many subsystems into one interface | Wraps multiple objects |
| **Proxy** | Control access (same interface as real) | Wraps one object, same interface |
| **Decorator** | Add behaviour (same interface, stackable) | Wraps one object, same interface, composable |

- Proxy and Decorator look similar structurally but differ in **intent**: Proxy controls access (security, lazy loading, transaction boundaries); Decorator adds features (encryption, compression, logging that stacks).

**What to say in the interview (4-beat answer)**
1. **Definition:** Façade simplifies a complex subsystem behind one interface. Proxy wraps an object to add behaviour (transactions, security, logging) transparently — the caller doesn't know it's talking to a proxy.
2. **Why/when:** `JdbcTemplate` is a Façade over JDBC boilerplate. `@Transactional` creates a Proxy around your service bean — intercepting method calls to manage transactions. Both are fundamental to Spring.
3. **Example:** A `TransferFacade` hides validation, fraud checks, ledger, and notifications behind one `transfer()` call. A `@Transactional` proxy wraps the real service — begins a transaction before the method, commits after, rolls back on exception.
4. **Gotcha/tradeoff:** Spring proxies don't intercept **self-calls** — if `methodA()` calls `this.methodB()`, and `methodB()` has `@Transactional`, the transaction is NOT applied because the call bypasses the proxy. Fix: inject the bean into itself or use `AopContext.currentProxy()`.

**Common pitfalls**
- Confusing Façade (simplifies) with Adapter (converts interface) — Façade wraps multiple objects; Adapter wraps one to change its interface.
- Assuming `@Transactional` works on `private` methods — CGLIB proxies can't intercept private methods.
- Self-invocation bypassing the proxy — `this.method()` doesn't go through Spring's proxy.
- Making a class `final` and expecting Spring to proxy it — CGLIB can't subclass `final` classes.

**Self-check question**
You have a `@Service` with `methodA()` calling `methodB()` which is `@Transactional`. Does `methodB` run in a transaction when called from `methodA`? What about when called from a `@Controller`? Explain the proxy mechanism behind both scenarios.
