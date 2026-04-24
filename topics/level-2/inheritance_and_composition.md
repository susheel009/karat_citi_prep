### Topic: Inheritance and Composition — Level 2

> **Foundation:** [Level 1 — Inheritance and Composition](../level-1/inheritance_and_composition.md) · [Level 1 — Object-Oriented Programming](../level-1/object_oriented_programming_oop.md)

**Why it matters (Karat angle)**
L1 covered the definitions. L2 is about the **design decision** — when to extend, when to compose, and how to refactor inheritance into composition. Interviewers use this to test architectural maturity — juniors extend everything; seniors compose.

**Core concept**

**The rule: favour composition over inheritance** (Effective Java, Item 18).

| | Inheritance (`extends`) | Composition (`has-a`) |
|--|------------------------|----------------------|
| Coupling | Tight — subclass depends on parent internals | Loose — delegate via interface |
| Flexibility | Fixed at compile time | Swap implementations at runtime |
| Code reuse | Fragile — parent changes break subclass | Stable — delegate contract is independent |
| Testing | Hard to mock parent | Easy to mock the delegate |
| Multiple behaviors | Single inheritance limit | Compose any number of behaviors |

**When inheritance IS appropriate:**
- True "is-a" relationships: `Dog extends Animal`, `IOException extends Exception`.
- You control both classes (same team, same module).
- The parent was **designed** for extension (non-final, documented extension points).

**When composition is better (most of the time):**
- "Has-a" or "uses-a" relationships.
- You need to combine behaviours from multiple sources.
- The "parent" class wasn't designed for inheritance (no `protected` extension points).
- You want to change behaviour at runtime (Strategy pattern).

**The fragile base class problem:**

```java
// File: topics/level-2/FragileBaseClassDemo.java

// Parent — not designed for extension
public class InstrumentedList<E> extends ArrayList<E> {

    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);                          // BUG: ArrayList.addAll calls add() internally
    }

    public int getAddCount() { return addCount; }
}

// Usage:
InstrumentedList<String> list = new InstrumentedList<>();
list.addAll(List.of("a", "b", "c"));
System.out.println(list.getAddCount());                  // 6, not 3!
// addAll increments by 3, then ArrayList.addAll calls add() 3 times → 3 more.
// Parent's internal implementation leaked into subclass.
```

**Fix with composition:**

```java
// File: topics/level-2/CompositionFixDemo.java

public class InstrumentedList<E> implements List<E> {

    private final List<E> delegate;                      // composition — wraps any List
    private int addCount = 0;

    public InstrumentedList(List<E> delegate) {
        this.delegate = delegate;
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return delegate.add(e);                          // delegate — no inheritance fragility
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c);                       // delegate.addAll does NOT call our add()
    }

    public int getAddCount() { return addCount; }

    // Delegate all other List methods...
    @Override public int size() { return delegate.size(); }
    @Override public E get(int i) { return delegate.get(i); }
    // ... (use IDE "generate delegate methods" or Lombok @Delegate)
}
```

**Strategy pattern — composition for runtime flexibility:**

```java
// File: topics/level-2/StrategyCompositionDemo.java

// Instead of: PaymentProcessor with subclasses WireProcessor, ACHProcessor, CheckProcessor
// Use composition:

interface PaymentStrategy {
    void process(Payment payment);
}

class WireStrategy implements PaymentStrategy {
    public void process(Payment p) { /* SWIFT network */ }
}
class ACHStrategy implements PaymentStrategy {
    public void process(Payment p) { /* ACH batch */ }
}

class PaymentProcessor {
    private final PaymentStrategy strategy;              // composed, not inherited

    public PaymentProcessor(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void execute(Payment p) {
        validate(p);
        strategy.process(p);                             // delegate to strategy
        audit(p);
    }
}

// Swap at runtime:
PaymentProcessor wire = new PaymentProcessor(new WireStrategy());
PaymentProcessor ach  = new PaymentProcessor(new ACHStrategy());
```

**Spring does this everywhere:**
```java
// Spring's RestTemplate uses composition:
// RestTemplate has-a HttpMessageConverter (Jackson for JSON, JAXB for XML)
// RestTemplate has-a ClientHttpRequestFactory (switch HTTP client at runtime)
// RestTemplate has-a ErrorHandler

// Your @Service composes @Repository — not extends
@Service
public class PaymentService {
    private final PaymentRepository repo;                // has-a, not is-a
    private final NotificationGateway gateway;           // another composed dependency
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Inheritance creates an "is-a" relationship with tight coupling to the parent. Composition creates a "has-a" relationship by delegating to an interface — loose coupling, runtime flexibility.
2. **Why/when:** Inheritance is fragile — changing the parent's internal implementation can break subclasses (fragile base class problem). Composition is preferred because it delegates through a stable interface, is testable (mock the delegate), and supports multiple behaviours.
3. **Example:** Instead of `InstrumentedList extends ArrayList` (which breaks because `addAll` internally calls `add`), wrap an `ArrayList` via a `delegate` field and implement the `List` interface — no inheritance surprises.
4. **Gotcha/tradeoff:** Composition requires more boilerplate (delegate all interface methods). Use `@Delegate` (Lombok) or IDE generation. The stability and testability are worth the extra lines.

**Common pitfalls**
- Extending utility classes (`ArrayList`, `HashMap`) — you inherit their internal implementation, which you don't control.
- Using inheritance for code reuse alone (not "is-a") — extract shared logic into a helper class and compose.
- Forgetting that Java only allows single class inheritance — if you extend one class, you can't extend another. Composition has no such limit.
- Not defining an interface for the delegate — locks you into one implementation.

**Self-check question**
You have `EmailNotifier extends Notifier` and `SmsNotifier extends Notifier`. Now you need an `EmailAndSmsNotifier`. Why can't you use inheritance? Show how composition solves it.
