### Topic: Beans — Level 1

> **Advanced coverage:** [Level 2 — Beans (scopes, ApplicationContext vs BeanFactory, post-processors)](../level-2/beans.md) · [Level 2 — Circular Dependencies](../level-2/circular_dependencies.md)

**Why it matters (Karat angle)**
Beans ARE Spring. If you can't explain the lifecycle, scopes, and the difference between XML and Java-based definition, an interviewer will doubt your Spring experience. This topic also opens the door to IoC, DI, and AOP follow-ups.

**Core concept**

A **bean** is any object managed by the Spring IoC container. The container handles creation, wiring (dependency injection), lifecycle callbacks, and destruction.

**Two ways to define beans:**

| Approach | When to use | Example |
|----------|------------|---------|
| **Annotation-based** (modern) | New projects, Spring Boot | `@Component`, `@Service`, `@Repository`, `@Controller` on classes; `@Bean` in `@Configuration` |
| **XML-based** (legacy) | Legacy codebases, Citi older projects | `<bean id="..." class="...">` in `applicationContext.xml` |

**Bean lifecycle (simplified):**

```
Container starts
  → Instantiate bean (constructor)
    → Inject dependencies (@Autowired / setter / constructor)
      → @PostConstruct / InitializingBean.afterPropertiesSet()
        → Bean is ready — container serves it
          → @PreDestroy / DisposableBean.destroy()
            → Container shuts down
```

**Bean scopes:**

| Scope | Instances created | Lifecycle | Default? |
|-------|------------------|-----------|----------|
| `singleton` | One per container | App startup → shutdown | ✅ |
| `prototype` | New one every time `getBean()` is called | Created on demand; NOT destroyed by container | ❌ |
| `request` | One per HTTP request | Start → end of request | ❌ (web only) |
| `session` | One per HTTP session | Session create → invalidate | ❌ (web only) |

**Mental model:** singleton = shared service (stateless). Prototype = throwaway worker (stateful per use). Most beans are singleton by default — and they should be stateless.

**Working code example**

```java
// File: topics/level-1/BeanDefinitionDemo.java

// ---------- Annotation-based (modern) ----------
@Component                                          // generic bean
public class NotificationGateway { }

@Service                                            // semantic: business logic layer
public class PaymentService {
    private final NotificationGateway gateway;

    @Autowired                                      // constructor injection (preferred)
    public PaymentService(NotificationGateway gateway) {
        this.gateway = gateway;
    }

    @PostConstruct
    public void init() {
        System.out.println("PaymentService ready — dependencies injected");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("PaymentService shutting down");
    }
}

// ---------- @Bean in @Configuration (third-party classes you can't annotate) ----------
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
                .setConnectTimeout(Duration.ofSeconds(3))
                .build();
    }

    @Bean
    @Scope("prototype")                              // new instance each time
    public AuditEvent auditEvent() {
        return new AuditEvent();
    }
}
```

```xml
<!-- File: topics/level-1/beans-xml-definition.xml -->
<!-- ---------- XML-based (legacy — you'll see this in Citi older projects) ---------- -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="paymentService" class="com.citi.PaymentService">
        <constructor-arg ref="notificationGateway"/>
    </bean>

    <bean id="notificationGateway" class="com.citi.NotificationGateway"/>

    <!-- lazy init — created on first use, not at startup -->
    <bean id="reportGenerator" class="com.citi.ReportGenerator" lazy-init="true"/>
</beans>
```

**Edge case: eager vs lazy initialization**
```java
// File: topics/level-1/LazyBeanDemo.java

@Component
@Lazy                                                // don't create at startup
public class HeavyReportEngine {
    public HeavyReportEngine() {
        System.out.println("HeavyReportEngine created (expensive!)");
        // loads ML model, reads large config, etc.
    }
}

@Service
public class ReportService {
    private final HeavyReportEngine engine;

    @Autowired
    @Lazy                                            // inject a proxy; real object created on first method call
    public ReportService(HeavyReportEngine engine) {
        this.engine = engine;
    }
}
// HeavyReportEngine is NOT created until ReportService actually calls a method on engine.
```

**Real-world use cases**
- **Singleton (default):** `@Service`, `@Repository`, `@Controller` — stateless business logic and data access.
- **Prototype:** audit event objects, request-scoped builders, anything that carries mutable state per use.
- **Lazy:** heavy initialisation beans (connection pools, ML models) that aren't needed on every startup path.
- **XML definition:** legacy Citi Spring 3.x apps; gradual migration path — you can mix XML and annotation config.

**What to say in the interview (4-beat answer)**
1. **Definition:** A bean is an object managed by the Spring IoC container — created, wired, and destroyed by the container. Defined via `@Component`/`@Service`/`@Bean` (modern) or XML (legacy).
2. **Why/when:** Beans decouple object creation from usage. The container handles the dependency graph, making code testable and modular.
3. **Example:** A `@Service PaymentService` depends on a `@Repository AccountRepo`. Spring creates both as singletons, injects the repo into the service via constructor, calls `@PostConstruct` for initialisation, and `@PreDestroy` on shutdown.
4. **Gotcha/tradeoff:** Singleton beans must be stateless — storing request-specific data in a singleton causes thread-safety bugs because all requests share the same instance.

**Common pitfalls**
- Putting mutable state (fields that change per request) in a singleton bean — race conditions in production.
- Confusing `@Component` with `@Bean` — `@Component` goes on YOUR classes; `@Bean` goes on methods in `@Configuration` for third-party classes you can't annotate.
- Forgetting that prototype-scoped beans injected into singletons are created ONCE — use `ObjectProvider<T>` or `@Lookup` to get a fresh prototype each time.
- Using `@PostConstruct` for logic that depends on the full application context being ready — use `ApplicationRunner` or `@EventListener(ApplicationReadyEvent.class)` instead.

**Self-check question**
You inject a prototype-scoped bean into a singleton service via constructor injection. How many instances of the prototype bean will exist for the lifetime of the app? How do you fix it to get a new instance on each call?
