### Topic: Types — Level 1

**Why it matters (Karat angle)**
Knowing when to use class vs abstract class vs interface vs enum vs record signals design fluency. Interviewers probe this to see if you understand Java's type system beyond "just write a class."

**Core concept**

Java offers several type constructs, each with a specific design purpose:

| Construct | State? | Constructor? | Multiple? | Instantiable? | Purpose |
|-----------|:------:|:-----------:|:---------:|:-------------:|---------|
| `class` | ✅ | ✅ | single `extends` | ✅ | Standard implementation |
| `abstract class` | ✅ | ✅ | single `extends` | ❌ | Shared code + enforced contract |
| `interface` | ❌ (only `static final`) | ❌ | multi `implements` | ❌ | Pure contract; capabilities |
| `enum` | ✅ | ✅ (private) | single (enum-specific) | Fixed instances only | Fixed set of named constants |
| `record` (Java 14+) | ✅ immutable | auto-generated | single (final) | ✅ | Transparent immutable data carrier |
| `annotation` (`@interface`) | N/A | N/A | applied, not implemented | N/A | Metadata for compile/runtime processing |
| anonymous class | ✅ | one instance | inline | ✅ once | Inline one-off implementation |

**When to pick what:**

| If you need... | Use |
|----------------|-----|
| Named constants with methods | `enum` |
| Immutable DTO / value object | `record` (Java 14+) or immutable class |
| A contract across unrelated hierarchies | `interface` |
| Shared code + an enforced hook | `abstract class` |
| A single inline implementation | lambda (preferred) or anonymous class |
| Attach metadata to code | `annotation` |

**Key rules worth memorising:**
- **Interface methods are implicitly `public abstract`** (unless `default`, `static`, or `private` in Java 9+).
- **Interface fields are implicitly `public static final`** — they're constants, not instance state.
- **Abstract classes** can have constructors (called via `super()` from subclasses) and instance fields.
- **Enums are implicitly `final`** (can't be extended) and their constructors are implicitly `private`.
- **Records** generate: all-args constructor, accessors (`name()` not `getName()`), `equals`, `hashCode`, `toString`. You can add methods and static factories.

**Mental model:** class = thing. Abstract class = partially-built thing. Interface = capability. Enum = closed set. Record = data-only thing. Annotation = sticky note on code.

**Real-world use cases**
- **Enum:** `TransactionStatus { PENDING, APPROVED, REJECTED }` with `isFinal()` method — type-safe state machines.
- **Record:** `public record AccountDTO(String id, long balance, LocalDate openedOn) {}` — REST API response bodies.
- **Interface:** `PaymentGateway`, `AuditLogger` — swap implementations via dependency injection.
- **Abstract class:** `BaseEntity` with shared `id`, `createdAt`, `updatedAt` plus abstract `validate()` — JPA entity hierarchies.
- **Annotation:** `@RetryOn(attempts = 3)` — processed by an AOP aspect at runtime.

**Working code example**
```java
// File: topics/level-1/TypesDemo.java
import java.time.LocalDate;

// ---------- Interface: capability contract ----------
interface Auditable {
    void audit();
    default String source() { return "SYSTEM"; }      // Java 8 default method
}

// ---------- Abstract class: shared code + hook ----------
abstract class BaseEntity {
    private final long id;
    protected BaseEntity(long id) { this.id = id; }

    public long getId() { return id; }
    public abstract String describe();                 // subclass must implement
}

// ---------- Enum: fixed domain with behaviour ----------
enum TransactionStatus {
    PENDING(false), APPROVED(true), REJECTED(true);

    private final boolean terminal;
    TransactionStatus(boolean terminal) { this.terminal = terminal; }

    public boolean isFinal() { return terminal; }
}

// ---------- Record: immutable data carrier (Java 14+) ----------
record TransactionDTO(long id, long amount, TransactionStatus status) {
    // Compact constructor — validation goes here
    public TransactionDTO {
        if (amount < 0) throw new IllegalArgumentException("Negative amount");
    }

    // Static factory
    public static TransactionDTO pending(long id, long amount) {
        return new TransactionDTO(id, amount, TransactionStatus.PENDING);
    }
}

// ---------- Annotation: metadata ----------
@interface ReviewedBy {
    String reviewer();
    String date();
}

// ---------- Class: combines it all ----------
@ReviewedBy(reviewer = "team-lead", date = "2026-04-23")
class Transaction extends BaseEntity implements Auditable {

    private final long amount;

    public Transaction(long id, long amount) {
        super(id);
        this.amount = amount;
    }

    @Override public String describe() { return "Txn#" + getId() + " $" + amount; }
    @Override public void audit()       { System.out.println("Auditing " + describe()); }
}

public class TypesDemo {
    public static void main(String[] args) {
        Transaction t = new Transaction(42L, 1000L);
        t.audit();

        System.out.println(TransactionStatus.PENDING.isFinal()); // false
        System.out.println(TransactionStatus.APPROVED.isFinal()); // true

        TransactionDTO dto = TransactionDTO.pending(1L, 500L);
        System.out.println(dto);                                 // auto toString
        System.out.println(dto.amount());                        // auto accessor

        // Lambda replaces anonymous class in Java 8+
        Auditable inline = () -> System.out.println("Ad-hoc audit");
        inline.audit();
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java has several type constructs — class, abstract class, interface, enum, record, annotation, and anonymous class — each designed for a specific modelling intent.
2. **Why/when:** Use `interface` for capabilities across unrelated classes (`Serializable`, `Comparable`), `abstract class` when subclasses share state or constructor logic, `enum` for closed sets, `record` for immutable DTOs.
3. **Example:** `Transaction extends BaseEntity implements Auditable` combines shared entity logic (abstract class) with a cross-cutting audit capability (interface). `TransactionDTO` as a record gives us equality/toString/accessors for free.
4. **Gotcha/tradeoff:** Since Java 8, `default` methods make interfaces look like abstract classes. The key remaining difference: abstract classes hold instance fields and constructors; interfaces hold only constants and method contracts.

**Common pitfalls**
- Using `String` or `int` constants instead of enums — loses type safety, no compile-time completeness check in switch.
- Forgetting that interface fields are `public static final` — making "state" in an interface isn't a thing.
- Implementing too many interfaces on one class — smells like too many responsibilities (SRP violation).
- Using a record when you actually need mutable state — records are final and fields are final.

**Self-check question**
You're designing a `Shape` hierarchy with shared `area()` formula logic and per-shape dimensions. Abstract class or interface? Why? What changes if you also need each shape to be `Serializable` and `Drawable`?
