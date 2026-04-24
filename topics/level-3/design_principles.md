### Topic: Design Principles — Level 3

> **Foundation:** [Level 1 — Design Patterns](../level-1/design_patterns.md) · [Level 2 — Design Patterns (Façade, Proxy)](../level-2/design_patterns.md) · [Level 2 — Inheritance and Composition](../level-2/inheritance_and_composition.md)

**Why it matters (Karat angle)**
L1/L2 covered patterns (Singleton, Factory, Proxy). L3 is about the **principles** that tell you when and why to apply them. SOLID, KISS, DRY, YAGNI — interviewers throw these acronyms to see if you design systems or just code features.

**Core concept**

**SOLID — the five OOP design principles:**

| Principle | One-liner | Violation smell |
|-----------|----------|----------------|
| **S** — Single Responsibility | A class has one reason to change | God class doing validation + persistence + notification |
| **O** — Open/Closed | Open for extension, closed for modification | Adding `if/else` for every new type |
| **L** — Liskov Substitution | Subtypes must be substitutable for their base type | `Square extends Rectangle` breaks `setWidth`/`setHeight` |
| **I** — Interface Segregation | No client should depend on methods it doesn't use | Fat interface with 20 methods |
| **D** — Dependency Inversion | Depend on abstractions, not concretions | `new MySQLRepository()` hardcoded in service |

```java
// File: topics/level-3/SolidDemo.java

// --- S: Single Responsibility ---
// WRONG: one class does everything
class OrderProcessor {
    void validate(Order o) { /* ... */ }
    void save(Order o) { /* ... */ }
    void sendEmail(Order o) { /* ... */ }
    void generatePdf(Order o) { /* ... */ }
}
// RIGHT: separate concerns
class OrderValidator { void validate(Order o) { } }
class OrderRepository { void save(Order o) { } }
class OrderNotifier { void sendEmail(Order o) { } }

// --- O: Open/Closed ---
// WRONG: modify PaymentProcessor for every new payment type
class PaymentProcessor {
    void process(Payment p) {
        if (p.getType().equals("WIRE")) { /* ... */ }
        else if (p.getType().equals("ACH")) { /* ... */ }
        // Add new else-if for every type...
    }
}
// RIGHT: extend via new classes
interface PaymentStrategy { void process(Payment p); }
class WirePayment implements PaymentStrategy { /* ... */ }
class ACHPayment implements PaymentStrategy { /* ... */ }
// New type = new class, no modification to existing code

// --- L: Liskov Substitution ---
// WRONG: Square breaks Rectangle's contract
class Rectangle { void setWidth(int w) { } void setHeight(int h) { } }
class Square extends Rectangle {
    void setWidth(int w) { super.setWidth(w); super.setHeight(w); } // violates LSP
}
// RIGHT: use composition or separate abstractions

// --- I: Interface Segregation ---
// WRONG: fat interface
interface Worker { void code(); void test(); void manage(); void recruit(); }
// RIGHT: split
interface Developer { void code(); void test(); }
interface Manager { void manage(); void recruit(); }

// --- D: Dependency Inversion ---
// WRONG: depends on concrete class
class TransferService {
    private MySQLAccountRepo repo = new MySQLAccountRepo(); // tight coupling
}
// RIGHT: depends on abstraction
class TransferService {
    private final AccountRepository repo; // interface
    TransferService(AccountRepository repo) { this.repo = repo; } // injected
}
```

**Beyond SOLID — other key principles:**

| Principle | Meaning | Example |
|-----------|---------|---------|
| **KISS** | Keep It Simple, Stupid | Don't use reflection when a simple `if` works |
| **YAGNI** | You Aren't Gonna Need It | Don't build a plugin system for 2 payment types |
| **DRY** | Don't Repeat Yourself | Extract shared validation into a utility |
| **Separation of Concerns** | Each module handles one concern | Controller (HTTP) ≠ Service (business) ≠ Repository (data) |
| **Composition over Inheritance** | Prefer `has-a` over `is-a` | `PaymentService` has a `PaymentStrategy`, not extends `BaseProcessor` |
| **IoC** | Framework controls object creation | Spring DI — you declare, container wires |
| **CQS** | Commands (mutate) and Queries (read) are separate methods | `debit()` returns void; `getBalance()` returns value, no side effects |
| **Chain of Responsibility** | Pass request along a chain of handlers | Servlet filters, Spring interceptors, validation chains |

**Chain of Responsibility in code:**

```java
// File: topics/level-3/ChainOfResponsibilityDemo.java

abstract class PaymentValidator {
    private PaymentValidator next;

    public PaymentValidator setNext(PaymentValidator next) {
        this.next = next;
        return next;
    }

    public void validate(Payment p) {
        doValidate(p);
        if (next != null) next.validate(p);
    }

    protected abstract void doValidate(Payment p);
}

class AmountValidator extends PaymentValidator {
    protected void doValidate(Payment p) {
        if (p.getAmount().compareTo(BigDecimal.ZERO) <= 0)
            throw new ValidationException("Amount must be positive");
    }
}

class FraudValidator extends PaymentValidator {
    protected void doValidate(Payment p) {
        if (fraudService.isSuspicious(p))
            throw new ValidationException("Fraud detected");
    }
}

class AccountValidator extends PaymentValidator {
    protected void doValidate(Payment p) {
        if (!accountRepo.exists(p.getAccountId()))
            throw new ValidationException("Account not found");
    }
}

// Build chain:
PaymentValidator chain = new AmountValidator();
chain.setNext(new FraudValidator()).setNext(new AccountValidator());
chain.validate(payment); // runs all validators in sequence
```

**What to say in the interview (4-beat answer)**
1. **Definition:** SOLID guides class design — Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion. Complementary principles: KISS, YAGNI, DRY, Separation of Concerns, Composition over Inheritance, CQS.
2. **Why/when:** They prevent the problems patterns solve — SOLID tells you *when* to refactor; patterns tell you *how*. Violating SRP creates god classes; violating OCP creates `if/else` chains; violating DIP creates untestable code.
3. **Example:** Chain of Responsibility for validation — each validator checks one concern, passes to the next. Adding a new check = adding a new class (OCP). Each validator has one job (SRP). The chain is composed (Composition over Inheritance).
4. **Gotcha/tradeoff:** Over-applying principles creates complexity. YAGNI is the counter-balance — don't abstract until you have a real second use case. Three classes for a single validation is over-engineering.

**Common pitfalls**
- Applying SOLID dogmatically — not every class needs an interface.
- DRY leading to wrong abstractions — two similar-looking pieces of code may have different reasons to change. Duplication is cheaper than the wrong abstraction.
- YAGNI vs extensibility — balance "don't build it yet" with "make it easy to build later."
- Confusing CQS with CQRS — CQS is method-level (query vs command); CQRS is architecture-level (separate read/write models).

**Self-check question**
You have a `UserService` that validates input, saves to DB, sends a welcome email, and publishes a Kafka event. Which SOLID principle is violated? Refactor it — how many classes/services do you end up with, and what's the trade-off?
