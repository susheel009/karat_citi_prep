### Topic: REST APIs — Securing with OAuth — Level 2

> **Foundation:** [Level 1 — REST APIs](../level-1/rest_apis.md)
> **Related:** [Level 2 — Security](./security.md) · [Level 2 — Spring Boot](./spring_boot.md)

**Why it matters (Karat angle)**
Citi's APIs use OAuth 2.0 and JWT tokens. Interviewers ask "how do you secure a REST API?" and expect OAuth flows, token validation, and Spring Security integration — not just "we use Spring Security."

**Core concept**

**OAuth 2.0** is an authorization framework. It separates authentication (who you are) from authorization (what you can do).

**Key roles:**

| Role | What it is | Example |
|------|-----------|---------|
| **Resource Owner** | The user | Bank customer |
| **Client** | The app requesting access | Mobile banking app |
| **Authorization Server** | Issues tokens | Keycloak, Okta, Azure AD |
| **Resource Server** | API that validates tokens | Your Spring Boot service |

**OAuth 2.0 flows:**

| Flow | Use case | How |
|------|---------|-----|
| **Authorization Code** | Web apps (server-side) | Redirect to auth server → code → exchange for token |
| **Client Credentials** | Service-to-service (no user) | Client ID + secret → token directly |
| **PKCE** | Mobile/SPA (public client) | Authorization Code + code verifier |
| **Password** (deprecated) | Legacy only | Username + password → token |

**For Citi microservices (service-to-service): Client Credentials flow.**
**For user-facing apps: Authorization Code with PKCE.**

**Spring Boot as a Resource Server (validating JWT tokens):**

```java
// File: topics/level-2/OAuthResourceServerDemo.java

// pom.xml dependency:
// spring-boot-starter-oauth2-resource-server

// application.yml
// spring:
//   security:
//     oauth2:
//       resourceserver:
//         jwt:
//           issuer-uri: https://auth.citi.com/realms/banking

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            )
            .build();
    }

    // Map JWT claims to Spring Security roles
    private JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");     // JWT claim containing roles
        authorities.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }
}
```

**Accessing JWT claims in a controller:**

```java
// File: topics/level-2/JwtClaimsDemo.java

@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    @GetMapping("/me")
    public AccountDto getMyAccount(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();                 // "sub" claim
        String email = jwt.getClaimAsString("email");
        List<String> roles = jwt.getClaimAsStringList("roles");
        return accountService.findByUserId(userId);
    }

    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/{id}")
    public void deleteAccount(@PathVariable Long id) {
        accountService.delete(id);
    }
}
```

**Service-to-service call with OAuth (Client Credentials):**

```java
// File: topics/level-2/ServiceToServiceOAuth.java

// application.yml
// spring:
//   security:
//     oauth2:
//       client:
//         registration:
//           payment-service:
//             client-id: order-service
//             client-secret: ${OAUTH_SECRET}
//             authorization-grant-type: client_credentials
//             scope: payment.read,payment.write
//         provider:
//           payment-service:
//             token-uri: https://auth.citi.com/oauth/token

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient paymentWebClient(
            OAuth2AuthorizedClientManager clientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(clientManager);
        oauth.setDefaultClientRegistrationId("payment-service");

        return WebClient.builder()
            .baseUrl("https://payment-service.citi.com")
            .filter(oauth)                                // auto-attaches Bearer token
            .build();
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** OAuth 2.0 separates auth from authorization. The auth server issues JWT tokens; the resource server (your API) validates them. Spring Boot's `oauth2-resource-server` starter handles JWT validation, claim extraction, and role mapping.
2. **Why/when:** Service-to-service uses Client Credentials flow (no user involved). User-facing apps use Authorization Code with PKCE. Both produce JWTs that the resource server validates.
3. **Example:** `@AuthenticationPrincipal Jwt jwt` gives access to the token's claims. `@PreAuthorize("hasRole('ADMIN')")` restricts endpoints. JWT audience, issuer, and signature are validated automatically by Spring.
4. **Gotcha/tradeoff:** JWTs are stateless — you can't revoke them before expiry. Use short-lived access tokens (5–15 min) + refresh tokens. Also: always validate the `aud` (audience) claim to prevent token misuse across services.

**Common pitfalls**
- Not validating JWT audience — a token issued for Service A could be used against Service B.
- Long-lived JWTs (hours/days) — can't be revoked; use short expiry + refresh.
- Hardcoding client secrets — use env vars or vault.
- Mixing authentication and authorization — OAuth is primarily authorization; OpenID Connect adds identity.

**Self-check question**
Your API validates JWTs issued by Keycloak. A user's roles are in the `realm_access.roles` claim (nested JSON). How do you configure `JwtGrantedAuthoritiesConverter` to extract roles from a nested claim?
