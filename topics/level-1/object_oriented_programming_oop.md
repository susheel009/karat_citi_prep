### Topic: Object-Oriented Programming (OOP) — Level 1

**Why it matters (Karat angle)**
Every Java interview opens here — it's the baseline filter. Interviewers want the four pillars named, defined, and demonstrated in code. A senior-level answer ties each pillar to a real design decision (why you chose it, what it buys you) instead of just reciting definitions.

**Core concept**

Java OOP is built on **four pillars** — each solves a specific problem in managing complexity:

| Pillar | What it is | What problem it solves | Java mechanism |
|--------|-----------|------------------------|----------------|
| **Encapsulation** | Bundle state + behaviour in a class; hide internals | Prevent callers from corrupting invariants (e.g., negative balance) | `private` fields, controlled access via methods |
| **Inheritance** | Subclass reuses parent via `extends` | Share code across "is-a" relationships | Single class inheritance; multiple interface inheritance |
| **Polymorphism** | One interface, many runtime behaviours | Decouple callers from concrete implementations | Method overriding (runtime), overloading (compile-time) |
| **Abstraction** | Expose what, hide how | Let consumers depend on contracts, not details | Abstract classes, interfaces |

**Mental model:** encapsulation = "the class protects its own state"; inheritance = "child gets parent's wiring"; polymorphism = "caller says `speak()`, runtime picks Dog.speak or Cat.speak"; abstraction = "the doorknob, not the lock mechanism."

Runtime mechanics worth knowing: polymorphism works via the **vtable** — each class has a table of method pointers, and `animal.speak()` dispatches through that table based on the runtime type of `animal`, not the declared type. Overloading is resolved by the compiler using argument types; overriding is resolved by the JVM using the object's actual class.

**Real-world use cases**
- **Domain modelling:** `Account`, `Transaction`, `Customer` classes encapsulate business invariants (e.g., `Account.withdraw()` rejects overdrafts).
- **Framework integration:** Spring beans implement framework interfaces (`Filter`, `HandlerInterceptor`); JPA entities inherit from a `BaseEntity` with shared `id`, `createdAt`.
- **Plugin/strategy architectures:** a `PaymentProcessor` interface with `StripeProcessor`, `PayPalProcessor` implementations — swap at runtime via config.
- **Testability:** polymorphism lets you substitute a real `EmailService` with a `MockEmailService` in unit tests.

**Working code example**
```java
// File: topics/level-1/OOPDemo.java

// Abstraction — defines the contract, no implementation
abstract class Account {
    private final String id;                 // Encapsulation — private, final
    private long balance;                    // Encapsulation — guarded by methods

    public Account(String id, long initial) {
        if (initial < 0) throw new IllegalArgumentException("Negative balance");
        this.id = id;
        this.balance = initial;
    }

    public final String getId()   { return id; }
    public final long getBalance() { return balance; }

    // Controlled mutation — invariant enforced here
    public void deposit(long amount) {
        if (amount <= 0) throw new IllegalArgumentException("Non-positive deposit");
        balance += amount;
    }

    public abstract long monthlyFee();       // subclasses define fee policy
}

// Inheritance — SavingsAccount IS-A Account
class SavingsAccount extends Account {
    public SavingsAccount(String id, long initial) { super(id, initial); }

    @Override
    public long monthlyFee() { return 0; }   // Polymorphism — runtime dispatch
}

class CheckingAccount extends Account {
    public CheckingAccount(String id, long initial) { super(id, initial); }

    @Override
    public long monthlyFee() { return 500; } // different policy
}

public class OOPDemo {
    public static void main(String[] args) {
        Account[] accounts = {
            new SavingsAccount("S-01", 10_000),
            new CheckingAccount("C-01", 5_000)
        };

        for (Account a : accounts) {
            // Same call — runtime picks the right monthlyFee()
            System.out.println(a.getId() + " fee: " + a.monthlyFee());
        }

        // Encapsulation in action — invariants cannot be violated
        try {
            accounts[0].deposit(-100);
        } catch (IllegalArgumentException e) {
            System.out.println("Blocked: " + e.getMessage());
        }
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** OOP organises code around objects that bundle state and behaviour, governed by four pillars: encapsulation, inheritance, polymorphism, and abstraction.
2. **Why/when:** It maps naturally to business domains, enforces invariants at the class level, and enables substitution — swap implementations without changing callers.
3. **Example:** In the code above, `Account` abstracts the concept, `SavingsAccount` inherits the common logic, `monthlyFee()` dispatches polymorphically, and `balance` is encapsulated so no caller can set it negative.
4. **Gotcha/tradeoff:** Deep inheritance hierarchies become rigid — the "fragile base class" problem means a change to `Account` can break every subclass. Prefer composition when behaviour varies independently.

**Common pitfalls**
- Confusing **overloading** (compile-time, same class, different parameter lists) with **overriding** (runtime, subclass, same signature, `@Override`).
- Exposing mutable fields via getters — the caller can mutate the referenced object, bypassing encapsulation. Return defensive copies or use immutable types.
- Writing `public` fields "just this once" — once published, removing them breaks every caller.
- Forgetting that `private` is class-level, not instance-level — one `Account` can read another `Account`'s private fields directly.

**Self-check question**
Without looking: name each pillar, give one example of how it's expressed in Java syntax, and say which line of the code above demonstrates it.
