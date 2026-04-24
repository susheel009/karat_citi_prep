### Topic: Beans (Scopes, AppContext vs BeanFactory, Post-processors) — Level 2

> **Foundation:** [Level 1 — Beans](../level-1/beans.md)
> **Related:** [Level 2 — Spring IoC](./spring.md) · [Level 2 — Circular Dependencies](./circular_dependencies.md)

**Why it matters (Karat angle)**
L1 covered bean definition and basic scopes. L2 digs into scope behaviour in practice — especially the singleton-prototype trap, `ApplicationContext` vs `BeanFactory`, and `BeanPostProcessor` which is how Spring implements `@Transactional`, `@Cacheable`, and AOP.

**Core concept**

**Bean scopes:**

| Scope | Instances | Lifecycle | Use case |
|-------|:---------:|-----------|----------|
| `singleton` (default) | 1 per context | App lifetime | Services, repos, configs |
| `prototype` | New per `getBean()` / injection | Not managed by Spring after creation | Stateful, short-lived objects |
| `request` | 1 per HTTP request | Request lifetime | Request-scoped data |
| `session` | 1 per HTTP session | Session lifetime | User session state |
| `application` | 1 per `ServletContext` | App lifetime (like singleton) | Shared across DispatcherServlets |

**The singleton-prototype trap:**

```java
// File: topics/level-2/SingletonPrototypeTrap.java

@Component
@Scope("prototype")
public class PaymentRequest {
    private BigDecimal amount;
    // Each injection should give a NEW instance...
}

@Service // singleton by default
public class PaymentService {

    @Autowired
    private PaymentRequest request;                      // injected ONCE at startup!

    public void process(BigDecimal amount) {
        request.setAmount(amount);                       // reuses same instance every call — BUG
    }
}
// PaymentRequest is prototype, but it's injected into a singleton.
// The singleton gets ONE instance at construction — prototype scope is useless.
```

**Fixes for the singleton-prototype problem:**

```java
// File: topics/level-2/PrototypeFixes.java

// Fix 1: ObjectFactory / Provider — lazy lookup
@Service
public class PaymentService {

    @Autowired
    private ObjectFactory<PaymentRequest> requestFactory;

    public void process(BigDecimal amount) {
        PaymentRequest req = requestFactory.getObject();  // new instance each call
        req.setAmount(amount);
    }
}

// Fix 2: @Lookup method injection
@Service
public abstract class PaymentServiceV2 {

    @Lookup
    protected abstract PaymentRequest createRequest();    // Spring overrides this at runtime

    public void process(BigDecimal amount) {
        PaymentRequest req = createRequest();              // new instance each call
        req.setAmount(amount);
    }
}

// Fix 3: Provider<T> (JSR-330 standard)
@Service
public class PaymentServiceV3 {

    @Autowired
    private Provider<PaymentRequest> requestProvider;

    public void process(BigDecimal amount) {
        PaymentRequest req = requestProvider.get();        // new instance each call
        req.setAmount(amount);
    }
}
```

**BeanPostProcessor — how Spring adds magic:**

A `BeanPostProcessor` intercepts every bean before and after initialization. Spring uses it internally for:
- `@Autowired` injection → `AutowiredAnnotationBeanPostProcessor`
- `@Transactional` proxy creation → `InfrastructureAdvisorAutoProxyCreator`
- `@Scheduled` task registration → `ScheduledAnnotationBeanPostProcessor`

```java
// File: topics/level-2/CustomBeanPostProcessor.java

@Component
public class AuditBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Runs BEFORE @PostConstruct
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Runs AFTER @PostConstruct — this is where proxies are created
        if (bean.getClass().isAnnotationPresent(Audited.class)) {
            // Wrap bean in a logging proxy
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    System.out.println("AUDIT: " + beanName + "." + method.getName());
                    return method.invoke(bean, args);
                }
            );
        }
        return bean;
    }
}
```

**BeanFactoryPostProcessor — modify bean definitions before instantiation:**

```java
// File: topics/level-2/BeanFactoryPostProcessorDemo.java

@Component
public class PropertyOverrideProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        // Modify bean definitions BEFORE any bean is created
        BeanDefinition def = factory.getBeanDefinition("dataSource");
        def.getPropertyValues().add("maxPoolSize", 50);
    }
}
```

| | `BeanPostProcessor` | `BeanFactoryPostProcessor` |
|--|---------------------|---------------------------|
| When | After bean instantiation | Before any beans are created |
| What | Intercept bean instances (wrap, modify) | Modify bean definitions (metadata) |
| Example | AOP proxy creation | Property placeholder resolution |

**What to say in the interview (4-beat answer)**
1. **Definition:** Bean scopes control how many instances exist. `BeanPostProcessor` intercepts bean creation — it's how Spring implements `@Transactional`, `@Autowired`, and AOP proxies.
2. **Why/when:** The singleton-prototype trap is a common bug — a prototype bean injected into a singleton is only created once. Fix with `ObjectFactory`, `Provider`, or `@Lookup`.
3. **Example:** `BeanPostProcessor.postProcessAfterInitialization()` can wrap a bean in a proxy — this is exactly how `@Transactional` works: the post-processor replaces the real bean with a CGLIB proxy.
4. **Gotcha/tradeoff:** `BeanPostProcessor` runs for every bean — keep it fast. `BeanFactoryPostProcessor` modifies definitions before instantiation — it can change scopes, properties, or conditionally register beans.

**Common pitfalls**
- Injecting prototype into singleton without `ObjectFactory`/`Provider` — prototype scope is silently ignored.
- Confusing `BeanPostProcessor` (instance-level) with `BeanFactoryPostProcessor` (definition-level).
- Implementing `BeanPostProcessor` in a `@Configuration` class — it may process its own config class prematurely.
- Assuming `@PreDestroy` runs on prototype beans — Spring doesn't manage prototype lifecycle after creation.

**Self-check question**
You have a `@Scope("prototype")` bean with a `@PreDestroy` method. You call `context.getBean()` 100 times. When does `@PreDestroy` run? How many times?
