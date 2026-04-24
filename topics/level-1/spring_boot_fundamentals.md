### Topic: Spring Boot Fundamentals — Level 1

> **Advanced coverage:** [Level 2 — Spring Boot (JPA, Pagination, Transactions, Actuator)](../level-2/spring_boot.md) · [Level 3 — Spring Boot (custom error codes, scheduling, transactional errors)](../level-3/spring_boot.md) · [Level 1 — Spring Boot Annotations](./spring_boot_annotations.md)

**Why it matters (Karat angle)**
Every Citi Java service is Spring Boot. Interviewers ask this to see if you understand what Boot actually does for you (auto-configuration, embedded server, opinionated defaults) versus raw Spring — and whether you've actually configured a production app, not just accepted the defaults.

**Core concept**

Spring Boot = Spring Framework + **auto-configuration** + **embedded server** + **opinionated starters**. It eliminates XML config and boilerplate so you can go from zero to a running REST service in minutes.

**Four things Boot gives you:**

| Feature | What it does | Without Boot |
|---------|-------------|-------------|
| **Auto-configuration** | Detects classpath jars and configures beans automatically | You write `@Bean` definitions for every DataSource, EntityManager, etc. |
| **Starters** | Curated dependency sets (`spring-boot-starter-web`, `-data-jpa`, `-security`) | You hunt compatible versions yourself |
| **Embedded server** | Tomcat/Jetty/Undertow runs inside your JAR | You deploy a WAR to an external Tomcat |
| **Externalized config** | `application.properties` / `application.yml` + profiles | Manual property loading |

**Auto-configuration — how it works**

1. `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
2. `@EnableAutoConfiguration` triggers `spring.factories` / `AutoConfiguration.imports` — a list of `@Configuration` classes.
3. Each auto-config class has `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc. — it only activates if the right jar is on the classpath AND you haven't already defined that bean.

**Mental model:** Boot's auto-config is a stack of "if you haven't done it yourself, I'll do it for you" rules. Define your own `DataSource` bean → Boot backs off. Don't define one but have `spring-boot-starter-data-jpa` on the classpath → Boot creates one from `application.properties`.

**Application properties — the essentials**

```properties
# File: src/main/resources/application.properties

# Server
server.port=8080
server.servlet.context-path=/api

# DataSource
spring.datasource.url=jdbc:postgresql://localhost:5432/citidb
spring.datasource.username=app_user
spring.datasource.password=${DB_PASSWORD}          # env var injection

# JPA
spring.jpa.hibernate.ddl-auto=validate             # never 'create' in prod
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true

# Logging
logging.level.root=INFO
logging.level.com.citi.myservice=DEBUG

# Actuator
management.endpoints.web.exposure.include=health,info,metrics
```

**Environment and property resolution order (highest wins):**

| Priority | Source |
|----------|--------|
| 1 (highest) | Command-line args (`--server.port=9090`) |
| 2 | OS environment variables (`SERVER_PORT=9090`) |
| 3 | `application-{profile}.properties` |
| 4 | `application.properties` |
| 5 | `@PropertySource` annotations |
| 6 | Defaults |

**Working code example**

```java
// File: topics/level-1/SpringBootApp.java

@SpringBootApplication                              // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class PaymentServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
        // Embedded Tomcat starts, component scan finds @Controller/@Service/@Repository
    }
}

@RestController
@RequestMapping("/payments")
public class PaymentController {

    private final PaymentService service;

    public PaymentController(PaymentService service) { this.service = service; }  // constructor DI

    @GetMapping("/{id}")
    public ResponseEntity<Payment> get(@PathVariable Long id) {
        return service.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<Payment> create(@Valid @RequestBody PaymentRequest req) {
        Payment p = service.create(req);
        URI location = URI.create("/payments/" + p.getId());
        return ResponseEntity.created(location).body(p);
    }
}
```

**Edge case: overriding auto-configuration**
```java
// File: topics/level-1/CustomDataSourceConfig.java
// Boot auto-configures a DataSource from application.properties.
// Define your own bean to override it — Boot backs off via @ConditionalOnMissingBean.

@Configuration
public class CustomDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:oracle:thin:@//citi-db:1521/PROD");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(3000);
        return new HikariDataSource(config);
    }
}
```

**Real-world use cases**
- **`application-dev.properties` / `application-prod.properties`** — different DB URLs, log levels, feature flags per environment. Boot picks the right file based on active profile.
- **`@Value("${payment.retry.max:3}")`** — inject config with a default. Overridden in prod via env var `PAYMENT_RETRY_MAX=5`.
- **Fat JAR deployment** — `mvn package` → single JAR with embedded Tomcat → `java -jar app.jar`. No WAR, no external server. This is how Citi deploys to OpenShift/Kubernetes.

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Boot is an opinionated layer on top of Spring Framework that provides auto-configuration, embedded servers, and starter dependencies to eliminate boilerplate.
2. **Why/when:** It lets you stand up a production-ready microservice in minutes — auto-configured DataSource, embedded Tomcat, health endpoints via Actuator — without writing XML or manual bean definitions.
3. **Example:** Adding `spring-boot-starter-data-jpa` and setting `spring.datasource.url` in properties is enough — Boot auto-configures a `DataSource`, `EntityManagerFactory`, and `TransactionManager`. Define your own `@Bean DataSource` and Boot backs off.
4. **Gotcha/tradeoff:** `spring.jpa.hibernate.ddl-auto=update` is convenient in dev but dangerous in prod — it can silently alter table schemas. Always use `validate` or `none` in production, with Flyway/Liquibase for migrations.

**Common pitfalls**
- Leaving `ddl-auto=create-drop` in production — drops and recreates tables on every restart.
- Not understanding property resolution order — a command-line arg beats `application.properties`, which can cause unexpected behaviour in CI/CD.
- Assuming `@SpringBootApplication` scans the whole project — it only scans its own package and sub-packages. Put your app class at the root package.
- Injecting with `@Value` without a default — app crashes on startup if the property is missing. Use `@Value("${key:default}")`.

**Self-check question**
You add `spring-boot-starter-security` to your pom.xml but don't write any security configuration. What happens when you hit any endpoint? Why?
