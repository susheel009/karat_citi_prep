### Topic: Cloning — Level 2

> **Related:** [Level 2 — Serialization](./serialization.md) · [Level 1 — Immutable Classes](../level-1/immutable_classes.md)

**Why it matters (Karat angle)**
Cloning tests whether you understand reference semantics in Java. "Shallow vs deep copy" is a standard interview question — and the correct answer involves explaining why `Object.clone()` is broken by design and what alternatives exist.

**Core concept**

| Copy type | What's copied | Nested objects | Performance |
|-----------|--------------|:---------------:|:-----------:|
| **Reference copy** | Just the pointer | Shared | O(1) |
| **Shallow clone** | Top-level fields | Shared (same references) | O(1) |
| **Deep clone** | Everything, recursively | Independent copies | O(n) |

```
Original:  Account { name: "Alice", address: Address { city: "Toronto" } }

Reference copy:  ref2 → same object as ref1
Shallow clone:   new Account { name: "Alice", address: → SAME Address object }
Deep clone:      new Account { name: "Alice", address: new Address { city: "Toronto" } }
```

**`Object.clone()` — the built-in (and problematic) approach**

```java
// File: topics/level-2/ShallowCloneDemo.java

public class Account implements Cloneable {

    private String name;                                 // immutable — shared safely
    private BigDecimal balance;                          // immutable — shared safely
    private List<Transaction> history;                   // MUTABLE — shallow clone shares this!

    @Override
    public Account clone() {
        try {
            return (Account) super.clone();              // shallow copy — field-by-field copy
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();                  // can't happen if Cloneable
        }
    }
}

// The problem:
Account original = new Account("Alice", BigDecimal.TEN, new ArrayList<>());
Account shallow = original.clone();
shallow.getHistory().add(new Transaction("deposit"));
System.out.println(original.getHistory().size());        // 1 — MUTATED THE ORIGINAL!
```

**Deep clone — manual approach**

```java
// File: topics/level-2/DeepCloneDemo.java

public class Account implements Cloneable {

    private String name;
    private BigDecimal balance;
    private List<Transaction> history;
    private Address address;

    @Override
    public Account clone() {
        try {
            Account copy = (Account) super.clone();
            // Deep-copy mutable fields:
            copy.history = new ArrayList<>(this.history);          // new list, same Transaction refs
            // If Transaction is also mutable, clone each:
            copy.history = this.history.stream()
                .map(Transaction::clone)
                .collect(Collectors.toList());
            copy.address = this.address.clone();                  // clone nested object
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

**Why `clone()` is considered broken (Effective Java, Item 13):**
1. `Cloneable` is a marker interface with no methods — `clone()` is on `Object`.
2. `clone()` doesn't call constructors — bypasses invariant checks.
3. Shallow by default — you must manually deep-copy every mutable field.
4. `CloneNotSupportedException` is checked — forces awkward try-catch.
5. Fragile with inheritance — subclass must remember to override `clone()`.

**Better alternatives:**

| Alternative | How | When |
|------------|-----|------|
| **Copy constructor** | `new Account(other)` | Preferred for most classes |
| **Static factory** | `Account.copyOf(other)` | When you want a different return type |
| **Serialization** | Serialize → deserialize | Deep copy of complex graphs (slow) |
| **Builder pattern** | `account.toBuilder().name("Bob").build()` | Immutable objects with modifications |

```java
// File: topics/level-2/CopyConstructorDemo.java

public class Account {

    private final String name;
    private final BigDecimal balance;
    private final List<Transaction> history;

    // Copy constructor — deep copy, explicit, no Cloneable mess
    public Account(Account other) {
        this.name = other.name;                          // immutable — share
        this.balance = other.balance;                    // immutable — share
        this.history = new ArrayList<>(other.history);   // defensive copy
    }

    // Static factory
    public static Account copyOf(Account other) {
        return new Account(other);
    }
}

// Usage:
Account copy = new Account(original);                   // clear, safe
Account copy2 = Account.copyOf(original);               // equally clear
```

**Edge case: deep copy via serialization**
```java
// File: topics/level-2/SerializationCloneDemo.java

// Works for any Serializable graph — no manual field copying
@SuppressWarnings("unchecked")
public static <T extends Serializable> T deepCopy(T obj) throws Exception {
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    try (ObjectOutputStream oos = new ObjectOutputStream(bos)) {
        oos.writeObject(obj);
    }
    ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
    try (ObjectInputStream ois = new ObjectInputStream(bis)) {
        return (T) ois.readObject();
    }
}
// Slow but thorough — useful for testing, not production hot paths.
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Shallow clone copies field values (references shared for objects); deep clone copies everything recursively. `Object.clone()` does shallow by default.
2. **Why/when:** Use copy constructors instead of `clone()` — they're explicit, type-safe, and work with inheritance. `clone()` is broken by design (no constructor call, shallow by default, checked exception).
3. **Example:** Shallow-cloning an `Account` with a `List<Transaction>` shares the list — adding a transaction to the clone mutates the original. Deep clone requires `new ArrayList<>(original.history)` and cloning each `Transaction`.
4. **Gotcha/tradeoff:** Immutable fields (String, BigDecimal) don't need deep copying — they can be safely shared. Only mutable fields (lists, arrays, custom objects with setters) need defensive copies.

**Common pitfalls**
- Assuming `clone()` is deep — it's shallow by default.
- Forgetting to clone nested mutable objects — one shared reference undermines the entire copy.
- Using `clone()` with inheritance — subclass adds a field but doesn't override `clone()` → field not copied.
- Deep-copying immutable objects (String, Integer, LocalDate) — wasteful; share references.
- Using serialization-based deep copy in hot paths — too slow for production traffic.

**Self-check question**
You have a `Department` with a `List<Employee>` where each `Employee` has an `Address`. You shallow-clone the `Department`. Then you call `clone.getEmployees().get(0).getAddress().setCity("Montreal")`. Does the original's first employee's city change? Why?
