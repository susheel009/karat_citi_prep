### Topic: Spring IoC Container — Level 2

> **Foundation:** [Level 1 — Spring Framework Fundamentals](../level-1/spring_framework_fundamentals.md) · [Level 1 — Beans](../level-1/beans.md)
> **Related:** [Level 2 — Beans (scopes, post-processors)](./beans.md) · [Level 2 — Circular Dependencies](./circular_dependencies.md)

**Why it matters (Karat angle)**
"Explain IoC" is the single most common Spring question. L1 covered DI basics. L2 expects you to explain the **container architecture** — `BeanFactory` vs `ApplicationContext`, how beans are created, wired, and lifecycle-managed. This is what separates "I use Spring" from "I understand Spring."

**Core concept**

**Inversion of Control (IoC)** = the framework controls object creation and wiring, not your code. **Dependency Injection (DI)** is the mechanism — Spring creates objects and injects their dependencies.

**Container hierarchy:**

```
BeanFactory (interface)
  └── ApplicationContext (interface, extends BeanFactory)
        ├── AnnotationConfigApplicationContext   (Java config)
        ├── ClassPathXmlApplicationContext        (XML config — legacy)
        └── WebApplicationContext                 (Spring MVC / Boot)
```

| | `BeanFactory` | `ApplicationContext` |
|--|--------------|---------------------|
| Bean instantiation | Lazy (on first `getBean()`) | Eager (all singletons at startup) |
| Event publishing | ❌ | ✅ `ApplicationEventPublisher` |
| Internationalization (i18n) | ❌ | ✅ `MessageSource` |
| AOP | Limited | Full |
| Use | Memory-constrained environments (rare) | Everything else (99% of apps) |

**Bean lifecycle in the IoC container:**

```
1. BeanDefinition loaded (from @Component scan or @Bean method)
2. Bean instantiated (constructor)
3. Dependencies injected (@Autowired fields/setters)
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct method called
6. InitializingBean.afterPropertiesSet() (if implemented)
7. Custom init-method (if specified)
8. BeanPostProcessor.postProcessAfterInitialization()
   ─── Bean is ready for use ───
9. @PreDestroy method called (on shutdown)
10. DisposableBean.destroy() (if implemented)
11. Custom destroy-method (if specified)
```

```java
// File: topics/level-2/BeanLifecycleDemo.java

@Component
public class PaymentGateway implements InitializingBean, DisposableBean {

    @Autowired
    private AccountRepository accountRepo;               // Step 3: injected

    @PostConstruct
    public void init() {                                  // Step 5
        System.out.println("@PostConstruct: gateway ready");
    }

    @Override
    public void afterPropertiesSet() {                    // Step 6
        System.out.println("InitializingBean: properties set");
    }

    @PreDestroy
    public void cleanup() {                               // Step 9
        System.out.println("@PreDestroy: releasing resources");
    }

    @Override
    public void destroy() {                               // Step 10
        System.out.println("DisposableBean: final cleanup");
    }
}
```

**DI styles — constructor vs field vs setter:**

| Style | Recommended? | Testable? | Immutable? |
|-------|:-----------:|:---------:|:----------:|
| **Constructor** | ✅ Preferred | ✅ Easy to mock | ✅ Fields can be `final` |
| **Field** (`@Autowired`) | ❌ | ❌ Requires reflection to test | ❌ |
| **Setter** | Occasional | ✅ | ❌ |

```java
// File: topics/level-2/ConstructorInjectionDemo.java

@Service
public class TransferService {

    private final AccountRepository accountRepo;
    private final AuditService auditService;

    // Constructor injection — Spring auto-detects (no @Autowired needed since Spring 4.3)
    public TransferService(AccountRepository accountRepo, AuditService auditService) {
        this.accountRepo = accountRepo;
        this.auditService = auditService;
    }
    // Fields are final → immutable after construction
    // Test: new TransferService(mockRepo, mockAudit) — no Spring needed
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** The IoC container (`ApplicationContext`) creates beans, resolves dependencies, and manages their lifecycle. It inverts control — your code declares what it needs; the container provides it.
2. **Why/when:** `ApplicationContext` is the standard — it eagerly creates singletons, supports events, i18n, and full AOP. `BeanFactory` is the low-level interface, rarely used directly.
3. **Example:** Bean lifecycle: constructor → DI → `@PostConstruct` → ready → `@PreDestroy` → shutdown. Constructor injection is preferred — fields can be `final`, no Spring dependency in tests.
4. **Gotcha/tradeoff:** `ApplicationContext` eagerly creates all singletons — startup is slower but fails fast on misconfiguration. `BeanFactory` is lazy — errors appear at runtime on first `getBean()`.

**Common pitfalls**
- Using field injection (`@Autowired private Repo repo;`) — untestable without reflection, fields can't be `final`.
- Not understanding eager vs lazy — `ApplicationContext` creates all singletons at startup; lazy beans (`@Lazy`) are deferred.
- Assuming `@PostConstruct` runs before ALL beans are ready — it runs after THIS bean's dependencies are injected, not after the entire context is ready. Use `ApplicationReadyEvent` for post-startup logic.
- Confusing `BeanFactory` with `FactoryBean` — `BeanFactory` is the container; `FactoryBean` is a pattern for creating complex beans.

**Self-check question**
You have `@PostConstruct void init()` in a `@Service` that calls another `@Service` which hasn't been created yet. What happens? How does Spring's startup order work?
