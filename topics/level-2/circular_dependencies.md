### Topic: Circular Dependencies — Level 2

> **Related:** [Level 2 — Spring IoC Container](./spring.md) · [Level 2 — Beans](./beans.md)

**Why it matters (Karat angle)**
Circular dependency errors during startup are common in large Spring projects. Interviewers ask this to see if you've hit it, understood why it happens, and know the fixes — not just the workaround.

**Core concept**

A **circular dependency** occurs when Bean A depends on Bean B, and Bean B depends on Bean A (directly or transitively). Spring can't complete construction of either — deadlock.

```
BeanA → needs BeanB → needs BeanA → 💥 BeanCurrentlyInCreationException
```

**Spring Boot 2.6+ default:** Circular dependencies cause startup failure. Before 2.6, Spring silently resolved some via early reference injection (which hid design problems).

**Three types:**

| Type | Example | Can Spring resolve? |
|------|---------|:-------------------:|
| **Constructor circular** | A(B), B(A) both via constructor | ❌ Always fails |
| **Field/setter circular** | A has `@Autowired B`, B has `@Autowired A` | ⚠️ With `@Lazy` or setter injection |
| **Transitive** | A → B → C → A | ❌ Same problem, harder to spot |

**Working code example — the problem:**

```java
// File: topics/level-2/CircularDependencyProblem.java

@Service
public class OrderService {
    private final PaymentService paymentService;
    public OrderService(PaymentService paymentService) {   // constructor injection
        this.paymentService = paymentService;
    }
}

@Service
public class PaymentService {
    private final OrderService orderService;
    public PaymentService(OrderService orderService) {     // constructor injection
        this.orderService = orderService;
    }
}

// Result: BeanCurrentlyInCreationException at startup
// Spring tries to create OrderService → needs PaymentService → needs OrderService → loop
```

**Fix 1: Redesign (preferred) — extract shared logic:**

```java
// File: topics/level-2/CircularFixRedesign.java

// Extract the shared concern into a third service
@Service
public class OrderService {
    private final OrderPaymentMediator mediator;
    public OrderService(OrderPaymentMediator mediator) { this.mediator = mediator; }
}

@Service
public class PaymentService {
    private final OrderPaymentMediator mediator;
    public PaymentService(OrderPaymentMediator mediator) { this.mediator = mediator; }
}

@Service
public class OrderPaymentMediator {
    // Shared logic that both services need — no cycle
}
```

**Fix 2: `@Lazy` — defer proxy injection:**

```java
// File: topics/level-2/CircularFixLazy.java

@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(@Lazy PaymentService paymentService) {  // inject lazy proxy
        this.paymentService = paymentService;
    }
    // At construction, Spring injects a proxy (not the real bean).
    // The real bean is created on first method call.
}
```

**Fix 3: Setter injection — break the constructor cycle:**

```java
// File: topics/level-2/CircularFixSetter.java

@Service
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
// Spring creates OrderService (no deps needed at construction),
// then creates PaymentService, then injects OrderService into PaymentService,
// then injects PaymentService into OrderService via setter.
```

**Fix 4: Events — decouple completely:**

```java
// File: topics/level-2/CircularFixEvents.java

@Service
public class OrderService {
    private final ApplicationEventPublisher events;

    public OrderService(ApplicationEventPublisher events) { this.events = events; }

    public void createOrder(Order order) {
        // ... create order
        events.publishEvent(new OrderCreatedEvent(order));    // no direct dep on PaymentService
    }
}

@Service
public class PaymentService {

    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // process payment — no direct dep on OrderService
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Circular dependency = Bean A needs Bean B at construction, Bean B needs Bean A. Spring can't resolve this with constructor injection — `BeanCurrentlyInCreationException`.
2. **Why/when:** It signals a design problem — two services are too tightly coupled. The fix should be architectural (extract common logic, use events) not just technical (`@Lazy`).
3. **Example:** `OrderService` and `PaymentService` both inject each other via constructor. Fix: extract shared logic into `OrderPaymentMediator`, or use `ApplicationEventPublisher` to decouple.
4. **Gotcha/tradeoff:** `@Lazy` is a band-aid — it hides the cycle. Events fully decouple but add complexity and make the flow harder to trace. Redesigning the dependency graph is the correct answer.

**Common pitfalls**
- Using `@Lazy` everywhere to hide design problems — cycles are a code smell; fix the architecture.
- Assuming setter injection is safe — it breaks immutability (fields can't be `final`) and makes testing harder.
- Not detecting transitive cycles (A → B → C → A) — they're harder to spot until startup fails.
- Allowing circular deps globally (`spring.main.allow-circular-references=true`) — suppresses the safety check without fixing the problem.

**Self-check question**
`ServiceA` needs `ServiceB` for one method, and `ServiceB` needs `ServiceA` for one method. Neither needs the other at construction time. What's the cleanest fix that doesn't use `@Lazy` or setter injection?
