### Topic: Configuration Properties Service — Level 2

> **Foundation:** [Level 1 — Spring Boot Fundamentals](../level-1/spring_boot_fundamentals.md) · [Level 1 — Profiles](../level-1/profiles.md)

**Why it matters (Karat angle)**
Externalised configuration is how you make a Spring Boot service deployable across environments without code changes. Interviewers test whether you know `@ConfigurationProperties` (type-safe) vs `@Value` (fragile), validation, and how to bind complex YAML structures.

**Core concept**

| Approach | Type-safe? | Validated? | Use case |
|----------|:---------:|:----------:|---------|
| `@Value("${key}")` | ❌ Strings, manual casting | ❌ | One-off values, simple cases |
| `@ConfigurationProperties` | ✅ Strongly typed, nested objects | ✅ with `@Validated` | Grouped configuration (DB, API, feature flags) |

**`@ConfigurationProperties` — the standard approach:**

```java
// File: topics/level-2/ConfigPropertiesDemo.java

@Configuration
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentProperties {

    @NotBlank
    private String gatewayUrl;

    @Min(1) @Max(60)
    private int timeoutSeconds = 30;                      // default value

    @NotNull
    private Retry retry = new Retry();                    // nested object

    private Map<String, BigDecimal> limits = new HashMap<>();  // map binding

    public static class Retry {
        private int maxAttempts = 3;
        private long delayMs = 1000;
        // getters/setters
    }

    // getters/setters for all fields
}
```

```yaml
# application.yml
app:
  payment:
    gateway-url: https://payment.citi.com/api
    timeout-seconds: 15
    retry:
      max-attempts: 5
      delay-ms: 2000
    limits:
      wire: 50000
      ach: 10000
      internal: 1000000
```

**Usage in a service:**

```java
// File: topics/level-2/ConfigUsageDemo.java

@Service
public class PaymentService {

    private final PaymentProperties props;

    public PaymentService(PaymentProperties props) {
        this.props = props;                               // injected, type-safe, validated
    }

    public void processPayment(String type, BigDecimal amount) {
        BigDecimal limit = props.getLimits().get(type);
        if (amount.compareTo(limit) > 0) {
            throw new LimitExceededException(type, limit);
        }

        // Use timeout from config
        HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(props.getTimeoutSeconds()))
            .build();
    }
}
```

**`@Value` — simpler but fragile:**

```java
// File: topics/level-2/ValueAnnotationDemo.java

@Service
public class NotificationService {

    @Value("${app.notification.enabled:true}")            // default = true
    private boolean enabled;

    @Value("${app.notification.recipients}")
    private List<String> recipients;                       // comma-separated in properties

    @Value("${app.notification.template:Hello #{name}}")
    private String template;

    // Problems:
    // - Typo in key → runtime failure (no compile-time check)
    // - No validation
    // - No grouping — scattered @Value across classes
}
```

**When to use which:**

| Scenario | Use |
|----------|-----|
| Single value, one place | `@Value` |
| Related config group | `@ConfigurationProperties` |
| Nested objects, lists, maps | `@ConfigurationProperties` |
| Validation required | `@ConfigurationProperties` + `@Validated` |
| Many services share config | `@ConfigurationProperties` (inject the bean) |

**Binding complex types from YAML:**

```yaml
# Lists
app.notification.recipients:
  - admin@citi.com
  - ops@citi.com

# Maps
app.feature-flags:
  fraud-check: true
  new-ui: false
  beta-payment: true
```

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private Map<String, Boolean> featureFlags;            // binds from YAML map
    private List<String> notificationRecipients;          // binds from YAML list
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `@ConfigurationProperties` binds a group of properties to a type-safe Java class with validation. `@Value` injects individual values but is fragile and unvalidated.
2. **Why/when:** Use `@ConfigurationProperties` for any grouped config (DB, gateway, feature flags). Use `@Value` for one-off values. Properties are externalised — same code, different config per environment.
3. **Example:** `PaymentProperties` binds `app.payment.*` to a class with `gatewayUrl`, `timeoutSeconds`, nested `Retry`, and a `Map<String, BigDecimal>` for payment limits. `@Validated` with `@NotBlank` catches misconfiguration at startup.
4. **Gotcha/tradeoff:** `@ConfigurationProperties` requires getters/setters (or `@ConstructorBinding` for immutable). Typos in YAML keys are silently ignored — enable `spring.config.import` validation or use IDE YAML support.

**Common pitfalls**
- Typos in YAML keys — silently ignored; field keeps its default (or null).
- Missing `@EnableConfigurationProperties` or `@ConfigurationPropertiesScan` — properties class not registered.
- `@Value` with missing key and no default — startup failure.
- Not using `@Validated` — invalid values (negative timeout, blank URL) cause runtime errors instead of startup failures.

**Self-check question**
You have `@ConfigurationProperties(prefix = "app.db")` with a field `maxPoolSize`. In YAML, does the property name need to be `maxPoolSize`, `max-pool-size`, or `MAX_POOL_SIZE`? What about as an environment variable?
