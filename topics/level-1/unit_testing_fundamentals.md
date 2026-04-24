### Topic: Unit Testing Fundamentals — Level 1

> **Related:** [Level 1 — Spring Boot Fundamentals](./spring_boot_fundamentals.md) · [Level 1 — Beans](./beans.md)

**Why it matters (Karat angle)**
Interviewers at Citi probe testing to see if you actually write production code or just demo code. They want to hear: JUnit 5 basics, Mockito for mocking dependencies, and how Spring Boot's `@SpringBootTest` vs `@WebMvcTest` differ. Saying "I test with Postman" is a red flag.

**Core concept**

Unit testing in Spring Boot sits on three layers:

| Layer | Framework | What it tests | Speed |
|-------|----------|--------------|-------|
| **Pure unit test** | JUnit 5 + Mockito | Single class, mocked deps | Milliseconds |
| **Slice test** | `@WebMvcTest`, `@DataJpaTest` | One Spring layer (web or repo) | Seconds |
| **Integration test** | `@SpringBootTest` | Full application context | Slow (full startup) |

**JUnit 5 essentials:**

| Annotation | Purpose |
|-----------|---------|
| `@Test` | Marks a test method |
| `@BeforeEach` / `@AfterEach` | Setup / teardown per test |
| `@BeforeAll` / `@AfterAll` | Setup / teardown per class (static methods) |
| `@DisplayName("...")` | Human-readable test name |
| `@ParameterizedTest` | Run same test with different inputs |
| `@Disabled` | Skip a test |

**Mockito essentials:**

| Method | Does what |
|--------|----------|
| `mock(Class)` or `@Mock` | Create a mock (fake) object |
| `when(mock.method()).thenReturn(value)` | Stub a method call |
| `verify(mock).method()` | Assert a method was called |
| `@InjectMocks` | Create the class under test and inject its `@Mock` dependencies |

**Working code example**

```java
// File: topics/level-1/PaymentServiceTest.java
import org.junit.jupiter.api.*;
import org.mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)                    // activate Mockito annotations
class PaymentServiceTest {

    @Mock
    private AccountRepository accountRepo;              // fake — no real DB

    @Mock
    private NotificationGateway notificationGateway;

    @InjectMocks
    private PaymentService paymentService;              // real class, mocked deps injected

    @Test
    @DisplayName("transfer should debit source and credit target")
    void transfer_success() {
        // Arrange
        Account source = new Account(1L, "Alice", new BigDecimal("1000"));
        Account target = new Account(2L, "Bob", new BigDecimal("500"));
        when(accountRepo.findById(1L)).thenReturn(Optional.of(source));
        when(accountRepo.findById(2L)).thenReturn(Optional.of(target));

        // Act
        paymentService.transfer(1L, 2L, new BigDecimal("200"));

        // Assert
        assertEquals(new BigDecimal("800"), source.getBalance());
        assertEquals(new BigDecimal("700"), target.getBalance());
        verify(accountRepo, times(2)).save(any(Account.class));
        verify(notificationGateway).send(anyString());
    }

    @Test
    @DisplayName("transfer should throw when source has insufficient balance")
    void transfer_insufficientBalance_throws() {
        Account source = new Account(1L, "Alice", new BigDecimal("50"));
        when(accountRepo.findById(1L)).thenReturn(Optional.of(source));

        assertThrows(InsufficientBalanceException.class, () ->
            paymentService.transfer(1L, 2L, new BigDecimal("200"))
        );

        verify(accountRepo, never()).save(any());       // no save should happen
    }
}
```

**Basic assertions cheat sheet:**

```java
// File: topics/level-1/AssertionsDemo.java

assertEquals(expected, actual);                        // equality
assertNotEquals(a, b);
assertTrue(condition);
assertFalse(condition);
assertNull(obj);
assertNotNull(obj);
assertThrows(IllegalArgumentException.class, () -> method());
assertDoesNotThrow(() -> method());
assertAll("grouped assertions",                        // run all, report all failures
    () -> assertEquals("Alice", name),
    () -> assertTrue(balance.compareTo(BigDecimal.ZERO) > 0)
);
assertTimeout(Duration.ofSeconds(2), () -> slowMethod());  // fails if > 2s
```

**Edge case: testing a REST controller without starting the full app**

```java
// File: topics/level-1/AccountControllerTest.java

@WebMvcTest(AccountController.class)                   // only loads web layer — no DB, no services
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;                            // simulates HTTP requests

    @MockBean                                           // Spring-aware mock (replaces bean in context)
    private AccountService accountService;

    @Test
    void getAccount_returns200() throws Exception {
        when(accountService.findById(1L))
            .thenReturn(Optional.of(new AccountDto(1L, "Alice", "1000")));

        mockMvc.perform(get("/api/v1/accounts/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.holderName").value("Alice"))
            .andExpect(jsonPath("$.balance").value("1000"));
    }

    @Test
    void getAccount_returns404_whenNotFound() throws Exception {
        when(accountService.findById(99L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/v1/accounts/99"))
            .andExpect(status().isNotFound());
    }

    @Test
    void createAccount_returns400_whenValidationFails() throws Exception {
        String invalidJson = """
            { "holderName": "", "initialDeposit": -100 }
            """;

        mockMvc.perform(post("/api/v1/accounts")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson))
            .andExpect(status().isBadRequest());
    }
}
```

**`@MockBean` vs `@Mock`:**

| | `@Mock` (Mockito) | `@MockBean` (Spring Boot) |
|--|-------------------|--------------------------|
| Context | Pure unit test — no Spring | Slice/integration test — needs Spring context |
| Replaces | Nothing — you wire manually | Actual bean in the application context |
| Use with | `@InjectMocks` | `@WebMvcTest`, `@SpringBootTest` |

**Real-world use cases**
- **Pure unit tests (`@Mock`)** — service layer logic, calculators, validators. Fast, no Spring context.
- **`@WebMvcTest`** — controller tests. Validates request mappings, validation, error handling, JSON serialisation.
- **`@DataJpaTest`** — repository tests against an embedded H2 DB. Validates queries, pagination.
- **`@SpringBootTest`** — full integration test. Validates the whole chain (controller → service → repo → DB). Slow — use sparingly.

**What to say in the interview (4-beat answer)**
1. **Definition:** Spring Boot testing uses JUnit 5 for assertions, Mockito for mocking dependencies, and Spring's test annotations (`@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`) for different test scopes.
2. **Why/when:** Unit tests with `@Mock` are fastest — test one class in isolation. Slice tests (`@WebMvcTest`) test one layer with a partial Spring context. Integration tests (`@SpringBootTest`) test everything but are slow.
3. **Example:** A service test mocks the repository with `@Mock`, injects it via `@InjectMocks`, stubs `findById()` with `when().thenReturn()`, then verifies with `assertEquals` and `verify`.
4. **Gotcha/tradeoff:** `@MockBean` replaces the real bean in the Spring context — if you use it in `@SpringBootTest`, it creates a new context per test class (slow). Prefer `@Mock` + `@InjectMocks` for service-layer tests.

**Common pitfalls**
- Using `@SpringBootTest` for everything — it's slow; prefer `@WebMvcTest` or `@DataJpaTest` for slice tests.
- Confusing `@Mock` and `@MockBean` — `@Mock` is pure Mockito, `@MockBean` is Spring's version that replaces beans in the context.
- Not using `@ExtendWith(MockitoExtension.class)` — `@Mock` annotations won't be initialised.
- Testing implementation details (verify every method call) instead of behaviour (verify the outcome).
- Forgetting `@Transactional` on `@DataJpaTest` — tests won't roll back and may pollute the DB for subsequent tests.

**Self-check question**
You're testing a `@Service` that calls a `@Repository` and a third-party REST client. Which test annotation do you use, and what do you mock? What changes if you also need to test the controller layer's JSON serialisation?
