### Topic: Profiles — Level 1

> **Related:** [Level 1 — Spring Boot Fundamentals](./spring_boot_fundamentals.md) · [Level 2 — Configuration Properties Service](../level-2/configuration_properties_service.md)

**Why it matters (Karat angle)**
Profiles are how you separate dev, staging, and production configuration without code changes. Interviewers ask this to see if you've deployed to multiple environments — a senior dev who's only ever run on `localhost` hasn't used profiles properly.

**Core concept**

A **profile** is a named configuration set that Spring activates at runtime. Each profile can have its own properties, beans, and behaviour.

**How it works:**
1. Define profile-specific property files: `application-dev.properties`, `application-prod.properties`.
2. Activate a profile at runtime: Spring loads `application.properties` PLUS `application-{profile}.properties`.
3. Profile-specific properties **override** base properties.

**Activation methods (highest priority wins):**

| Method | Example | Priority |
|--------|---------|----------|
| Command-line arg | `--spring.profiles.active=prod` | Highest |
| Environment variable | `SPRING_PROFILES_ACTIVE=prod` | High |
| `application.properties` | `spring.profiles.active=dev` | Medium |
| Programmatic | `SpringApplication.setAdditionalProfiles("dev")` | Low |
| `@ActiveProfiles("test")` | In test classes only | Test only |

**Working code example**

```properties
# File: src/main/resources/application.properties (base — always loaded)
app.name=PaymentService
server.port=8080
spring.profiles.active=dev                            # default profile
```

```properties
# File: src/main/resources/application-dev.properties
spring.datasource.url=jdbc:h2:mem:devdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
logging.level.com.citi=DEBUG
app.feature.fraud-check=false                         # skip in dev
```

```properties
# File: src/main/resources/application-prod.properties
spring.datasource.url=jdbc:oracle:thin:@//citi-db:1521/PROD
spring.datasource.username=${DB_USER}                 # from env var
spring.datasource.password=${DB_PASS}
spring.jpa.hibernate.ddl-auto=validate                # never auto-DDL in prod
logging.level.com.citi=WARN
app.feature.fraud-check=true
```

**Profile-specific beans:**

```java
// File: topics/level-1/ProfileBeansDemo.java

// Bean only created when 'dev' profile is active
@Configuration
@Profile("dev")
public class DevConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .addScript("schema.sql")
                .build();
    }
}

// Bean only created when 'prod' profile is active
@Configuration
@Profile("prod")
public class ProdConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:oracle:thin:@//citi-db:1521/PROD");
        config.setMaximumPoolSize(30);
        return new HikariDataSource(config);
    }
}

// Bean created for ALL profiles EXCEPT 'prod'
@Component
@Profile("!prod")
public class MockEmailGateway implements EmailGateway {
    @Override
    public void send(String to, String body) {
        System.out.println("[MOCK] Email to " + to + ": " + body);
    }
}
```

**Edge case: multiple active profiles**

```bash
# Activate multiple profiles — properties from both are merged, later wins on conflicts
java -jar app.jar --spring.profiles.active=prod,metrics

# application-prod.properties is loaded first, then application-metrics.properties
# If both define server.port, metrics wins (last profile wins)
```

```java
// File: topics/level-1/MultiProfileCheck.java
@Service
public class StartupLogger {

    @Autowired
    private Environment env;

    @PostConstruct
    public void log() {
        String[] profiles = env.getActiveProfiles();
        System.out.println("Active profiles: " + Arrays.toString(profiles));
        // Check if a specific profile is active:
        if (env.acceptsProfiles(Profiles.of("prod"))) {
            System.out.println("Running in PRODUCTION mode");
        }
    }
}
```

**Profile usage in tests:**

```java
// File: topics/level-1/ProfileTestDemo.java
@SpringBootTest
@ActiveProfiles("test")                               // loads application-test.properties
class PaymentServiceIntegrationTest {
    // test with test-specific DB, mock services, etc.
}
```

**Real-world use cases**
- **`dev` profile:** H2 in-memory DB, DEBUG logging, mock external services, `ddl-auto=create-drop`.
- **`prod` profile:** Oracle/Postgres with connection pool, WARN logging, real external services, `ddl-auto=validate`.
- **`test` profile:** H2 or Testcontainers, `@ActiveProfiles("test")`, separate data setup.
- **Feature flags via profiles:** `app.feature.fraud-check=true` in prod, `false` in dev — toggle behaviour without code changes.

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring profiles let you define environment-specific configuration (properties, beans) that activate at runtime based on the active profile name.
2. **Why/when:** Different environments need different DB URLs, log levels, and feature flags. Profiles keep this config externalised — no code changes between dev and prod.
3. **Example:** `application-dev.properties` uses H2 with `ddl-auto=create-drop`; `application-prod.properties` uses Oracle with `ddl-auto=validate`. Activated via `--spring.profiles.active=prod` at deployment.
4. **Gotcha/tradeoff:** Setting `spring.profiles.active=dev` in `application.properties` and forgetting to override it in production — the app runs with dev config in prod. Always set the active profile externally (env var or command-line) in production environments.

**Common pitfalls**
- Hardcoding `spring.profiles.active=dev` in `application.properties` and deploying to prod without overriding — dev config leaks into production.
- Not having a `@Profile("!prod")` mock for external services — tests and dev inadvertently call real production APIs.
- Assuming profile-specific properties files completely replace the base — they don't; they merge and override only the keys they define.
- Using `@ActiveProfiles` outside of tests — it only works in test classes, not in production code.

**Self-check question**
You have `application.properties` with `server.port=8080` and `application-prod.properties` with `server.port=9090`. You start with `--spring.profiles.active=prod --server.port=7070`. What port does the server run on, and why?
