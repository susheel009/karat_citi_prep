### Topic: REST APIs — Level 1

> **Advanced coverage:** [Level 2 — REST APIs (OAuth security)](../level-2/rest_apis.md) · [Level 2 — Security (JWT, SSL, Spring Security)](../level-2/security.md)

**Why it matters (Karat angle)**
REST is the interface layer of every Citi microservice. Interviewers expect you to define endpoints correctly (method + path + status code), validate input, handle errors gracefully, and know basic security — not just return 200 for everything.

**Core concept**

REST (Representational State Transfer) maps **HTTP methods to CRUD operations** on resources:

| HTTP Method | CRUD | Idempotent? | Request body? | Typical status codes |
|-------------|------|:-----------:|:-------------:|---------------------|
| `GET` | Read | ✅ | ❌ | 200 OK, 404 Not Found |
| `POST` | Create | ❌ | ✅ | 201 Created, 400 Bad Request |
| `PUT` | Replace (full) | ✅ | ✅ | 200 OK, 204 No Content |
| `PATCH` | Update (partial) | ❌ | ✅ | 200 OK |
| `DELETE` | Delete | ✅ | ❌ | 204 No Content, 404 Not Found |

**Default HTTP method for a form submit?** `GET` — but REST APIs typically use `POST` for creates.

**HTTP status code families:**

| Range | Meaning | Examples |
|-------|---------|---------|
| `2xx` | Success | 200 OK, 201 Created, 204 No Content |
| `3xx` | Redirect | 301 Moved, 304 Not Modified |
| `4xx` | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity |
| `5xx` | Server error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

**Common errors and when to use them:**
- **400** — malformed JSON, missing required field.
- **401** — no credentials or expired token.
- **403** — valid credentials but insufficient role.
- **404** — resource doesn't exist.
- **409** — conflict (e.g., duplicate account number).
- **422** — syntactically valid but semantically invalid (e.g., negative balance).

**Spring Boot REST mapping annotations:**

| Annotation | Shortcut for |
|-----------|-------------|
| `@GetMapping("/path")` | `@RequestMapping(value="/path", method=GET)` |
| `@PostMapping` | `@RequestMapping(method=POST)` |
| `@PutMapping` | `@RequestMapping(method=PUT)` |
| `@PatchMapping` | `@RequestMapping(method=PATCH)` |
| `@DeleteMapping` | `@RequestMapping(method=DELETE)` |

**Working code example**

```java
// File: topics/level-1/RestApiDemo.java

@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    private final AccountService service;

    public AccountController(AccountService service) { this.service = service; }

    // ---------- GET: read ----------
    @GetMapping("/{id}")
    public ResponseEntity<AccountDto> getById(@PathVariable Long id) {
        return service.findById(id)
                .map(ResponseEntity::ok)                          // 200
                .orElse(ResponseEntity.notFound().build());        // 404
    }

    @GetMapping
    public List<AccountDto> list(@RequestParam(defaultValue = "ACTIVE") String status) {
        return service.findByStatus(status);                      // 200
    }

    // ---------- POST: create ----------
    @PostMapping
    public ResponseEntity<AccountDto> create(@Valid @RequestBody CreateAccountRequest req) {
        AccountDto created = service.create(req);
        URI location = URI.create("/api/v1/accounts/" + created.getId());
        return ResponseEntity.created(location).body(created);    // 201 + Location header
    }

    // ---------- PUT: full replace ----------
    @PutMapping("/{id}")
    public ResponseEntity<AccountDto> replace(@PathVariable Long id,
                                               @Valid @RequestBody UpdateAccountRequest req) {
        return ResponseEntity.ok(service.replace(id, req));       // 200
    }

    // ---------- DELETE ----------
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();                // 204
    }
}
```

**Request validation with Bean Validation:**

```java
// File: topics/level-1/AccountRequest.java

public class CreateAccountRequest {

    @NotBlank(message = "Account holder name is required")
    private String holderName;

    @NotNull
    @Positive(message = "Initial deposit must be positive")
    private BigDecimal initialDeposit;

    @Email(message = "Invalid email format")
    private String email;

    @Pattern(regexp = "^[A-Z]{3}$", message = "Currency must be 3-letter ISO code")
    private String currency;

    // getters, setters
}
// Spring returns 400 + field-level error messages automatically when @Valid fails
```

**Global error handling:**

```java
// File: topics/level-1/GlobalExceptionHandler.java

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AccountNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(AccountNotFoundException ex) {
        return ResponseEntity.status(404)
                .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String errors = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_FAILED", errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(500)
                .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
    }
}

record ErrorResponse(String code, String message) {}
```

**API security basics (Level 1 depth)**
- **Authentication:** who are you? (token, basic auth)
- **Authorization:** what can you do? (roles, permissions)
- Simplest approach: Spring Security + HTTP Basic for internal APIs.
- Production: OAuth2 / JWT tokens (covered in Level 2).
- Always use HTTPS in production — never send credentials over plain HTTP.

**Real-world use cases**
- **`@RestController` + `@Service` + `@Repository`** — the standard three-layer architecture for every Citi microservice.
- **`@Valid`** — request validation at the controller layer; fail fast with 400 before hitting business logic.
- **`@RestControllerAdvice`** — centralised error handling; consistent error response format across all endpoints.
- **Versioned paths (`/api/v1/`)** — allows breaking changes in v2 without affecting existing clients.

**What to say in the interview (4-beat answer)**
1. **Definition:** REST APIs map HTTP methods (GET/POST/PUT/DELETE) to CRUD operations on resources, returning appropriate HTTP status codes. In Spring Boot, `@RestController` handles this with `@GetMapping`, `@PostMapping`, etc.
2. **Why/when:** REST is the standard interface for microservices communication. Proper status codes (201 for create, 204 for delete, 404 for missing) make APIs self-documenting and debuggable.
3. **Example:** A `POST /api/v1/accounts` with `@Valid @RequestBody` validates the request, creates the resource, and returns 201 with a `Location` header. Invalid input returns 400 with field-level error messages.
4. **Gotcha/tradeoff:** Returning 200 for everything (including errors) is a common anti-pattern — clients can't tell success from failure without parsing the body. Use proper HTTP status codes.

**Common pitfalls**
- Returning 200 for errors with an error message in the body — breaks HTTP semantics.
- Using `GET` for operations that mutate state — `GET` must be safe and idempotent.
- Not validating request bodies — lets malformed data reach the service/DB layer.
- Exposing internal exceptions in error responses — security risk; wrap them with `@RestControllerAdvice`.
- Forgetting `@PathVariable` or `@RequestParam` and getting `null` — Spring won't bind automatically without the annotation.

**Self-check question**
Your API receives a `POST` with a valid JSON body, but the account number already exists in the database. What HTTP status code should you return, and why not 400?
