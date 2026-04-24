### Topic: Spring Boot Annotations — Level 1

> **Related:** [Level 1 — Spring Boot Fundamentals](./spring_boot_fundamentals.md) · [Level 1 — Beans](./beans.md) · [Level 1 — REST APIs](./rest_apis.md)

**Why it matters (Karat angle)**
Interviewers use annotations as a rapid-fire knowledge check. They'll ask "what does `@Transactional` do?" or "difference between `@Controller` and `@RestController`?" and expect a crisp one-liner, not a lecture. Knowing 15–20 core annotations cold is table stakes for a senior role.

**Core concept**

Spring Boot annotations fall into four functional groups:

| Group | Purpose | Key annotations |
|-------|---------|----------------|
| **Bootstrap** | App startup and configuration | `@SpringBootApplication`, `@EnableAutoConfiguration`, `@ComponentScan`, `@Configuration` |
| **Stereotype** | Mark beans for component scan | `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController` |
| **Web / REST** | Map HTTP requests to methods | `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody`, `@ResponseBody` |
| **DI / Lifecycle** | Wire beans and manage lifecycle | `@Autowired`, `@Qualifier`, `@Value`, `@Bean`, `@Scope`, `@Lazy`, `@PostConstruct`, `@PreDestroy` |

---

**Bootstrap annotations — what each actually does**

```java
@SpringBootApplication
// = @Configuration       → this class is a source of bean definitions
// + @EnableAutoConfiguration → activate Spring Boot's auto-config magic
// + @ComponentScan           → scan this package + sub-packages for @Component classes
```

| Annotation | Does what |
|-----------|----------|
| `@SpringBootApplication` | Combines the three above; entry point |
| `@EnableAutoConfiguration` | Triggers conditional auto-config (`spring.factories`) |
| `@ComponentScan` | Tells Spring where to look for `@Component` classes |
| `@Configuration` | Marks a class as a bean definition source (replaces XML) |
| `@ConfigurationProperties(prefix="app")` | Binds a whole properties block to a POJO |

---

**Stereotype annotations — when to use which**

| Annotation | Layer | Special behaviour |
|-----------|-------|-------------------|
| `@Component` | Generic | None — just registers as a bean |
| `@Service` | Business logic | None beyond semantics (no extra behaviour) |
| `@Repository` | Data access | Enables **exception translation** — vendor SQL exceptions → Spring's `DataAccessException` |
| `@Controller` | Web (view) | Returns view names → resolved by `ViewResolver` |
| `@RestController` | Web (API) | = `@Controller` + `@ResponseBody` on every method → returns JSON directly |

**The key question: `@Controller` vs `@RestController`**
- `@Controller` returns a **view name** (e.g., `"index"`) that a `ViewResolver` maps to an HTML template.
- `@RestController` returns the **object itself**, serialised to JSON by Jackson. No view resolution.
- If you use `@Controller` but want JSON, add `@ResponseBody` on each method.

---

**Web / REST annotations**

```java
// File: topics/level-1/AnnotationDemo.java

@RestController
@RequestMapping("/api/v1/transactions")
public class TransactionController {

    // @PathVariable — extracts from URL path
    @GetMapping("/{id}")
    public Transaction get(@PathVariable Long id) { ... }

    // @RequestParam — extracts from query string (?status=ACTIVE)
    @GetMapping
    public List<Transaction> list(@RequestParam(defaultValue = "ACTIVE") String status) { ... }

    // @RequestBody — deserialises JSON body to object
    @PostMapping
    public Transaction create(@Valid @RequestBody TransactionRequest req) { ... }

    // @ResponseStatus — override default status code
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)           // 204 instead of 200
    public void delete(@PathVariable Long id) { ... }
}
```

---

**DI / Lifecycle annotations**

```java
// File: topics/level-1/DiAnnotationDemo.java

@Configuration
public class AppConfig {

    @Bean                                            // registers return value as a bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean
    @Scope("prototype")                              // new instance each injection
    public AuditEntry auditEntry() { return new AuditEntry(); }
}

@Service
public class PaymentService {

    private final AccountRepo repo;
    private final ObjectMapper mapper;

    @Autowired                                       // constructor injection — preferred
    public PaymentService(AccountRepo repo, ObjectMapper mapper) {
        this.repo = repo;
        this.mapper = mapper;
    }

    @Value("${payment.retry.max:3}")                 // inject property with default
    private int maxRetries;

    @PostConstruct                                   // called after DI complete
    public void init() { System.out.println("PaymentService ready"); }

    @PreDestroy                                      // called before bean destruction
    public void shutdown() { System.out.println("PaymentService shutting down"); }
}

// When two beans of the same type exist:
@Service
public class NotificationService {

    @Autowired
    @Qualifier("smsGateway")                         // disambiguate by name
    private NotificationGateway gateway;
}
```

---

**Quick-reference cheat sheet**

| Annotation | One-liner |
|-----------|----------|
| `@SpringBootApplication` | Bootstrap + auto-config + scan |
| `@RestController` | `@Controller` + `@ResponseBody` on all methods |
| `@Autowired` | Inject a dependency (prefer constructor) |
| `@Qualifier("name")` | Pick a specific bean when multiple match |
| `@Value("${key:default}")` | Inject a property value |
| `@Bean` | Register a method's return value as a bean |
| `@Scope("prototype")` | New instance per injection |
| `@Lazy` | Don't create until first use |
| `@PostConstruct` | Run after DI is complete |
| `@PreDestroy` | Run before bean is destroyed |
| `@Transactional` | Wrap method in a DB transaction |
| `@Valid` | Trigger Bean Validation on `@RequestBody` |
| `@ResponseStatus(HttpStatus.X)` | Override HTTP status code |

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Boot uses annotations to replace XML configuration. `@SpringBootApplication` bootstraps the app; stereotype annotations (`@Service`, `@Repository`, `@RestController`) mark beans for specific layers; DI annotations (`@Autowired`, `@Bean`) wire them together.
2. **Why/when:** Annotations make the architecture self-documenting — you can read a class's role from its annotation. `@Repository` adds exception translation; `@RestController` adds JSON serialisation; `@Transactional` adds rollback behaviour.
3. **Example:** `@RestController` = `@Controller` + `@ResponseBody`. A `@GetMapping("/{id}")` method with `@PathVariable` extracts the ID from the URL and returns a JSON-serialised object.
4. **Gotcha/tradeoff:** `@Autowired` on fields works but is hard to unit test (you need reflection or Spring context). Constructor injection is preferred — it makes dependencies explicit and allows plain `new` in tests.

**Common pitfalls**
- Using `@Controller` when you want JSON — you'll get a `ViewResolver` error. Use `@RestController`.
- Field injection everywhere — makes testing painful; constructor injection is the modern standard.
- Missing `@ComponentScan` base package — Boot only scans the `@SpringBootApplication` class's package and below. Put the main class at the root.
- Confusing `@Component` and `@Bean` — `@Component` goes on your own classes; `@Bean` goes on factory methods in `@Configuration` for third-party objects you can't annotate.
- Forgetting `@EnableAutoConfiguration` when not using `@SpringBootApplication` — no auto-config activates.

**Self-check question**
You have two implementations of `NotificationGateway` — `SmsGateway` and `EmailGateway` — both annotated `@Component`. You `@Autowired NotificationGateway` in a service. What happens at startup, and how do you fix it?
