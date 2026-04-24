### Topic: Spring Framework vs Spring Boot — Level 1

> **Related:** [Level 1 — Spring Framework Fundamentals](./spring_framework_fundamentals.md) · [Level 1 — Spring Boot Fundamentals](./spring_boot_fundamentals.md)

**Why it matters (Karat angle)**
This is one of the most common screening questions — it immediately reveals whether a candidate has only ever used Boot (and doesn't know what it hides) or has worked with raw Spring and understands the plumbing. Interviewers also use it to segue into containerization questions.

**Core concept**

Spring Framework is the **engine** — DI, AOP, MVC, JDBC abstraction. Spring Boot is the **chassis** — it wraps the engine with auto-configuration, embedded servers, and opinionated defaults so you spend zero time on wiring XML.

**Head-to-head comparison:**

| Dimension | Spring Framework | Spring Boot |
|-----------|-----------------|-------------|
| Configuration | Explicit XML or `@Configuration` classes | Auto-configured; you override only what you need |
| Server | External (deploy WAR to Tomcat/WebLogic) | Embedded (Tomcat/Jetty baked into JAR) |
| Dependency mgmt | You pick compatible versions manually | Starter POMs with curated, tested version sets |
| Production readiness | You build health checks, metrics, etc. | Actuator gives `/health`, `/metrics`, `/info` out of the box |
| Learning curve | Higher — understand every bean definition | Lower — "it just works" until you need to customise |
| Startup speed | Depends on config | Slower than raw Spring (classpath scanning + auto-config reflection) but fast enough for microservices |

**Why Spring Boot?**

1. **Eliminates boilerplate** — no `web.xml`, no `DispatcherServlet` registration, no `DataSource` bean definition (unless you want custom).
2. **Embedded server** — `java -jar app.jar` is a self-contained process. Perfect for containers.
3. **Starter dependencies** — `spring-boot-starter-web` pulls in Tomcat, Jackson, Spring MVC, validation — all version-compatible.
4. **Actuator** — production monitoring (`/actuator/health`, `/actuator/metrics`) with zero code.
5. **Profiles** — `application-dev.yml`, `application-prod.yml` for environment-specific config.

**Why containerization? (the follow-up question)**

| Traditional deployment | Containerized deployment |
|----------------------|------------------------|
| WAR → external Tomcat on a VM | Fat JAR → Docker image → Kubernetes pod |
| Ops team configures Tomcat version, JVM flags per server | Everything is in the Dockerfile — reproducible anywhere |
| Scaling = more VMs, manual load balancer config | Scaling = `kubectl scale --replicas=5` |
| Env drift between dev/staging/prod | Same image in every environment; config via env vars |

Spring Boot's embedded server makes containerization trivial:
```dockerfile
# File: topics/level-1/Dockerfile
FROM eclipse-temurin:21-jre-alpine
COPY target/payment-service-1.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

No Tomcat install, no WAR deployment script. The JAR IS the server.

**Working code example — the same service, before and after Boot**

```xml
<!-- File: topics/level-1/spring-only-web.xml -->
<!-- WITHOUT Boot: you configure everything manually -->
<web-app>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

```java
// File: topics/level-1/SpringBootEquivalent.java
// WITH Boot: all of the above is auto-configured
@SpringBootApplication
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
        // DispatcherServlet: auto-registered
        // Embedded Tomcat: auto-started
        // Component scan: auto-triggered
        // Jackson: auto-configured for JSON serialisation
    }
}
```

**When to use raw Spring Framework (no Boot)?**
- Legacy enterprise apps deployed as WARs on WebLogic/WebSphere (Citi has these).
- When you need a non-web Spring context (batch processing, desktop app).
- When startup time is critical and you want to eliminate auto-config overhead (rare — Boot startup is typically < 5 seconds).

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Framework is the core DI/AOP/MVC engine. Spring Boot wraps it with auto-configuration, embedded servers, starters, and Actuator so you skip boilerplate and deploy as a standalone JAR.
2. **Why/when:** Use Boot for any new microservice — it eliminates XML, bundles compatible deps, and makes containerization trivial. Use raw Spring only for legacy WAR deployments or non-web contexts.
3. **Example:** In raw Spring, you write `web.xml`, register `DispatcherServlet`, configure a `DataSource` bean. In Boot, you add `spring-boot-starter-web` + `spring-boot-starter-data-jpa`, set `spring.datasource.url` in properties, and everything auto-configures.
4. **Gotcha/tradeoff:** Boot's auto-configuration is magic until it breaks — when it picks the wrong bean or version. Understanding `@ConditionalOnMissingBean` and `--debug` output is essential for debugging auto-config issues.

**Common pitfalls**
- Saying "Spring Boot is a different framework" — it's not; it's an opinionated layer ON TOP of Spring Framework.
- Not knowing what `@SpringBootApplication` actually does — it's three annotations combined (`@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`).
- Deploying a Boot app as a WAR to external Tomcat without excluding the embedded Tomcat starter — classpath conflicts.
- Assuming Boot replaces understanding Spring — when auto-config breaks, you need Spring fundamentals to debug.

**Self-check question**
A teammate deploys a Spring Boot app to an external Tomcat server and gets `ClassNotFoundException: javax.servlet.Servlet`. What went wrong, and how do you fix the POM?
