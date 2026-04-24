### Topic: Servlets (Dispatcher Servlet) — Level 2

> **Foundation:** [Level 1 — Spring Framework Fundamentals](../level-1/spring_framework_fundamentals.md) · [Level 1 — REST APIs](../level-1/rest_apis.md)
> **Advanced:** [Level 3 — Spring MVC (Dispatcher Servlet internals, regex request mapping)](../level-3/spring_mvc.md)

**Why it matters (Karat angle)**
"How does a request reach your `@RestController`?" is a standard question. If you can't explain the servlet container, `DispatcherServlet`, and handler mapping pipeline, you're operating Spring Boot like a black box — not a senior-level answer.

**Core concept**

**What's a servlet?**
A servlet is a Java class that handles HTTP requests. The servlet container (Tomcat, Jetty) manages servlet lifecycle and routes requests to them.

**Spring MVC replaces manual servlets with one master servlet: `DispatcherServlet`.**

**Request flow:**
```
Client → Tomcat → DispatcherServlet → HandlerMapping → Controller → View/Response
```

```
1. Client sends HTTP request to Tomcat (port 8080)
2. Tomcat looks up the servlet mapped to the URL → DispatcherServlet (mapped to "/")
3. DispatcherServlet calls HandlerMapping to find the right @Controller method
4. HandlerAdapter invokes the controller method
5. Controller returns data (or ModelAndView for non-REST)
6. HttpMessageConverter serialises to JSON (Jackson)
7. Response sent back through Tomcat to client
```

```java
// File: topics/level-2/DispatcherServletFlow.java

// You write this:
@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    @GetMapping("/{id}")
    public AccountDto getAccount(@PathVariable Long id) {
        return accountService.findById(id);
    }
}

// Spring does this behind the scenes:
// 1. DispatcherServlet receives GET /api/v1/accounts/42
// 2. RequestMappingHandlerMapping maps it to getAccount() method
// 3. RequestMappingHandlerAdapter invokes it, binding @PathVariable id=42
// 4. Jackson's MappingJackson2HttpMessageConverter converts AccountDto → JSON
// 5. Response: 200 OK, Content-Type: application/json, body: {"id": 42, ...}
```

**Key components in the DispatcherServlet pipeline:**

| Component | Responsibility |
|----------|---------------|
| `HandlerMapping` | URL → controller method lookup |
| `HandlerAdapter` | Invoke the handler, resolve method args |
| `HttpMessageConverter` | Convert request body → Java object and Java → response body |
| `HandlerInterceptor` | Pre/post-processing (logging, auth, timing) |
| `HandlerExceptionResolver` | Map exceptions to HTTP responses |

**Interceptors — the servlet filter alternative in Spring:**

```java
// File: topics/level-2/InterceptorDemo.java

@Component
public class RequestTimingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("startTime", System.nanoTime());
        return true;                                      // continue to controller
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res,
                                 Object handler, Exception ex) {
        long start = (long) req.getAttribute("startTime");
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.println(req.getRequestURI() + " → " + elapsed + "ms");
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new RequestTimingInterceptor())
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/health");
    }
}
```

**Filters vs Interceptors:**

| | Servlet Filter | HandlerInterceptor |
|--|---------------|-------------------|
| Level | Servlet container (Tomcat) | Spring MVC (DispatcherServlet) |
| Access to | `HttpServletRequest/Response` | Spring handler + model objects |
| Ordering | `@Order` or `FilterRegistrationBean` | `InterceptorRegistry.order()` |
| Use case | Security, CORS, compression | Logging, auth, request timing |

**What to say in the interview (4-beat answer)**
1. **Definition:** `DispatcherServlet` is Spring MVC's front controller — a single servlet that receives all requests, delegates to `HandlerMapping` to find the right controller, `HandlerAdapter` to invoke it, and `HttpMessageConverter` to serialise the response.
2. **Why/when:** It replaces manual servlet programming with annotation-driven controllers. You never write a raw `doGet()`/`doPost()` in Spring Boot — DispatcherServlet handles everything.
3. **Example:** `GET /api/accounts/42` → DispatcherServlet → `RequestMappingHandlerMapping` finds `AccountController.getAccount(42)` → adapter invokes it → Jackson converts `AccountDto` to JSON → response sent.
4. **Gotcha/tradeoff:** Filters run before DispatcherServlet (Tomcat level); Interceptors run after (Spring level). Security filters (`OncePerRequestFilter`) should be filters, not interceptors, because they need to run before Spring dispatching.

**Common pitfalls**
- Confusing Filters (servlet level) with Interceptors (Spring MVC level) — use the right one for each concern.
- Not registering interceptors via `WebMvcConfigurer` — they won't run.
- Assuming `@RestController` is magic — it's `@Controller` + `@ResponseBody`; the DispatcherServlet + Jackson do the actual JSON conversion.
- Blocking the DispatcherServlet thread with long operations — use `@Async` or reactive endpoints for slow I/O.

**Self-check question**
A request hits `/api/accounts/42` but returns 404 even though the controller method exists. What are three things in the DispatcherServlet pipeline that could cause this?
