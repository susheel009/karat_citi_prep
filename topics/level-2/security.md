### Topic: Security — Level 2

> **Foundation:** [Level 1 — REST APIs](../level-1/rest_apis.md) · [Level 2 — REST APIs (OAuth)](./rest_apis.md)

**Why it matters (Karat angle)**
Citi is a bank — security isn't optional. Interviewers expect JWT understanding, Spring Security configuration, SSL/TLS, endpoint-level role restrictions, and CORS. This is the most common "senior-level" Spring topic.

**Core concept**

**JWT (JSON Web Token) structure:**
```
Header.Payload.Signature

Header:  { "alg": "RS256", "typ": "JWT" }
Payload: { "sub": "user123", "roles": ["ADMIN"], "iat": 1680000000, "exp": 1680003600 }
Signature: RSASHA256(base64(header) + "." + base64(payload), privateKey)
```

| Part | Contains | Purpose |
|------|---------|---------|
| Header | Algorithm, token type | How to verify |
| Payload | Claims (sub, roles, exp) | Who, what, when |
| Signature | Cryptographic hash | Tamper detection |

**Spring Security filter chain:**

```java
// File: topics/level-2/SecurityConfigDemo.java

@Configuration
@EnableWebSecurity
@EnableMethodSecurity                                     // enables @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                 // disable for stateless APIs
            .sessionManagement(sm -> sm
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/accounts/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/accounts/**").hasRole("ADMIN")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

**Method-level security:**

```java
// File: topics/level-2/MethodSecurityDemo.java

@Service
public class AccountService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAccount(Long id) { /* ... */ }

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.subject")
    public AccountDto getAccount(Long id, String userId) { /* ... */ }

    @PostAuthorize("returnObject.ownerId == authentication.principal.subject")
    public AccountDto findById(Long id) {
        return accountRepo.findById(id).map(this::toDto).orElseThrow();
        // After return, Spring checks if the result's ownerId matches the JWT subject
    }
}
```

**Enabling SSL/TLS:**

```yaml
# application.yml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASS}
    key-store-type: PKCS12
    key-alias: banking-api
```

```java
// File: topics/level-2/HttpsRedirectDemo.java

// Force HTTPS redirect
@Configuration
public class HttpsConfig {

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addAdditionalTomcatConnectors(httpConnector());
        return tomcat;
    }

    private Connector httpConnector() {
        Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        connector.setRedirectPort(8443);
        return connector;
    }
}
```

**CORS configuration:**

```java
// File: topics/level-2/CorsConfigDemo.java

@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://banking.citi.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Authorization", "Content-Type")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Security secures endpoints via a filter chain. JWT tokens carry claims (subject, roles, expiry). `@PreAuthorize` enforces method-level access control. SSL/TLS encrypts traffic.
2. **Why/when:** Stateless APIs use JWT + OAuth 2.0. Role-based access (`hasRole('ADMIN')`) restricts endpoints. SSL is mandatory in banking — all data in transit must be encrypted.
3. **Example:** `SecurityFilterChain` with `.requestMatchers("/api/admin/**").hasRole("ADMIN")` restricts admin endpoints. `@PreAuthorize("#userId == authentication.principal.subject")` ensures users can only access their own data.
4. **Gotcha/tradeoff:** Disabling CSRF is correct for stateless JWT APIs but dangerous for cookie-based sessions. Always set `STATELESS` session policy when using JWT. Don't expose `/actuator/**` without authentication.

**Common pitfalls**
- Disabling CSRF without stateless sessions — CSRF protection is still needed for cookie-based auth.
- Using `permitAll()` on actuator endpoints — leaks sensitive info.
- Not validating JWT `exp` claim — expired tokens are accepted.
- Hardcoding CORS `allowedOrigins("*")` in production — security hole.
- Storing JWT in localStorage — XSS can steal it; use HttpOnly cookies for browser apps.

**Self-check question**
Your API has two roles: `USER` and `ADMIN`. You want `USER` to read accounts (GET) and `ADMIN` to create/delete (POST/DELETE). Write the `SecurityFilterChain` config and explain why order of `requestMatchers` matters.
