### Topic: Spring MVC (Dispatcher Servlet Internals, Regex Request Mapping) — Level 3

> **Foundation:** [Level 2 — Servlets (Dispatcher Servlet)](../level-2/servlets.md) · [Level 2 — Spring IoC](../level-2/spring.md)

**Why it matters (Karat angle)**
L2 covered the request flow at a high level. L3 covers the internal workings — how `DispatcherServlet` initialises, how `HandlerMapping` resolves complex URL patterns (regex, Ant patterns), and how argument resolvers bind request data to method parameters.

**Core concept**

**DispatcherServlet initialization sequence:**

```
1. Tomcat starts → creates ServletContext
2. Spring Boot registers DispatcherServlet as "/" mapping
3. DispatcherServlet.init() called:
   a. Creates WebApplicationContext (child of root context)
   b. Initialises strategy beans:
      - HandlerMapping (which handler for which URL)
      - HandlerAdapter (how to invoke the handler)
      - HandlerExceptionResolver (how to handle exceptions)
      - ViewResolver (for MVC views, not REST)
      - HttpMessageConverter (JSON/XML conversion)
4. All @Controller/@RestController beans registered in HandlerMapping
```

**Request processing internals:**

```java
// File: topics/level-3/DispatcherServletInternals.java

// Simplified DispatcherServlet.doDispatch():
protected void doDispatch(HttpServletRequest req, HttpServletResponse res) {
    // 1. Find handler
    HandlerExecutionChain chain = getHandler(req);
    // iterates through HandlerMappings:
    //   RequestMappingHandlerMapping → checks @RequestMapping annotations
    //   SimpleUrlHandlerMapping → static resource mappings

    // 2. Find adapter
    HandlerAdapter adapter = getHandlerAdapter(chain.getHandler());
    // RequestMappingHandlerAdapter for @Controller methods

    // 3. Run interceptor pre-handle
    if (!chain.applyPreHandle(req, res)) return;

    // 4. Invoke handler
    ModelAndView mv = adapter.handle(req, res, chain.getHandler());
    // For @RestController: return value serialised by HttpMessageConverter

    // 5. Run interceptor post-handle
    chain.applyPostHandle(req, res, mv);

    // 6. Process dispatch result (render view or write response)
    processDispatchResult(req, res, chain, mv, null);
}
```

**Regex and Ant patterns in `@RequestMapping`:**

```java
// File: topics/level-3/RegexRequestMappingDemo.java

@RestController
@RequestMapping("/api/v1")
public class AdvancedMappingController {

    // --- Ant-style patterns ---
    @GetMapping("/accounts/*")                            // matches /accounts/123 but NOT /accounts/123/txns
    public String singleWildcard() { return "single"; }

    @GetMapping("/accounts/**")                           // matches /accounts/123/txns/456 (any depth)
    public String doubleWildcard() { return "deep"; }

    @GetMapping("/reports/*/summary")                     // matches /reports/2026/summary
    public String middleWildcard() { return "summary"; }

    // --- Regex in path variables ---
    @GetMapping("/accounts/{id:\\d+}")                    // id must be digits only
    public String numericId(@PathVariable Long id) {
        return "Account: " + id;
    }

    @GetMapping("/accounts/{code:[A-Z]{3}\\d{7}}")       // e.g., USD1234567
    public String accountCode(@PathVariable String code) {
        return "Code: " + code;
    }

    @GetMapping("/files/{filename:.+}")                   // dot-inclusive (e.g., report.pdf)
    public String fileName(@PathVariable String filename) {
        return "File: " + filename;
    }

    // --- Matrix variables ---
    // GET /api/v1/accounts/filter;status=ACTIVE;region=NA
    @GetMapping("/accounts/filter")
    public String matrixFilter(
            @MatrixVariable String status,
            @MatrixVariable String region) {
        return "Status: " + status + ", Region: " + region;
    }
}
```

**Argument resolvers — how Spring binds request data:**

```java
// File: topics/level-3/ArgumentResolverDemo.java

@RestController
public class ResolverDemoController {

    @GetMapping("/search")
    public List<Transaction> search(
            @RequestParam String status,                  // ?status=PENDING
            @RequestParam(defaultValue = "0") int page,   // ?page=0
            @RequestParam(required = false) String region, // optional
            @RequestHeader("X-Correlation-Id") String corrId, // from header
            @CookieValue(name = "session", required = false) String session,
            @AuthenticationPrincipal Jwt jwt,              // from security context
            HttpServletRequest rawRequest                  // raw servlet request
    ) {
        return List.of();
    }

    @PostMapping("/accounts")
    public Account create(
            @RequestBody @Valid AccountCreateRequest body, // JSON body → object
            @RequestHeader("Content-Type") String contentType,
            UriComponentsBuilder uriBuilder                // for building Location header
    ) {
        Account saved = accountService.create(body);
        // ...
        return saved;
    }
}
```

**Custom `HandlerMethodArgumentResolver`:**

```java
// File: topics/level-3/CustomArgumentResolver.java

// Example: resolve the current user from JWT into a domain User object
public class CurrentUserResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter param) {
        return param.hasParameterAnnotation(CurrentUser.class)
            && param.getParameterType().equals(User.class);
    }

    @Override
    public User resolveArgument(MethodParameter param, ModelAndViewContainer mvc,
                                 NativeWebRequest request, WebDataBinderFactory binder) {
        Jwt jwt = (Jwt) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return userService.findBySubject(jwt.getSubject());
    }
}

// Register:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserResolver());
    }
}

// Usage:
@GetMapping("/me")
public UserDto getMe(@CurrentUser User user) { return toDto(user); }
```

**What to say in the interview (4-beat answer)**
1. **Definition:** DispatcherServlet initialises strategy beans (HandlerMapping, HandlerAdapter, MessageConverter) at startup. On each request, it resolves the handler, runs interceptors, invokes the controller, and converts the response.
2. **Why/when:** Regex in `@RequestMapping` enables precise URL matching — `{id:\\d+}` ensures path variable is numeric. Custom argument resolvers inject domain objects (current user) directly into controller parameters.
3. **Example:** `@GetMapping("/accounts/{code:[A-Z]{3}\\d{7}}")` matches `GET /accounts/USD1234567` but rejects `GET /accounts/invalid`. Spring resolves the regex, extracts the path variable, and passes it to the method.
4. **Gotcha/tradeoff:** URL pattern priority: exact match > longest path > regex. If two patterns match, Spring picks the most specific. `/**` is the lowest priority catch-all. Custom argument resolvers are powerful but can hide complexity — document them.

**Common pitfalls**
- Path variable with dots (e.g., `filename.pdf`) — Spring truncates after the last dot. Use `{filename:.+}` to include the extension.
- Regex escape in annotations — `\\d` not `\d` because Java strings need double backslash.
- Conflicting patterns — `/**` catches everything and can shadow specific mappings.
- Custom argument resolvers that hit the database on every request — add caching or request-scoped beans.

**Self-check question**
You have `@GetMapping("/users/{id}")` and `@GetMapping("/users/admin")`. A request comes in for `GET /users/admin`. Which handler runs? What about `GET /users/123`? What determines the priority?
