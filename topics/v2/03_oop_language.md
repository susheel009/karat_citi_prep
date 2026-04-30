# 03 — OOP & Language Fundamentals

[← Back to Index](./00_INDEX.md) | **Priority: 🔴 Critical**

---

## Rapid-Fire Q&A

### Q1: Interface vs abstract class — when each?
**A:** Interface = pure contract. Since Java 8: default methods (shared behaviour) and static methods — but no instance state. Abstract class = partial implementation + instance state. Modern preference: program to interfaces. Use abstract class only when you need shared mutable state or a template method pattern.

### Q2: What changed about interfaces in Java 8?
**A:** Added `default` methods (implementation in interface — avoids breaking existing implementations). Added `static` methods. Java 9 added `private` methods in interfaces. This moved Java closer to traits / mix-in patterns.

### Q3: Method overloading vs overriding?
**A:** Overloading = same name, different parameters. Resolved at **compile time** (static dispatch). Overriding = subclass replaces superclass method (same signature). Resolved at **runtime** (dynamic dispatch / virtual method lookup). `@Override` annotation catches typos at compile time.

### Q4: `final` keyword — three contexts?
**A:** `final` variable: can't reassign (reference is fixed, but object contents can still change). `final` method: can't be overridden by subclasses. `final` class: can't be subclassed (e.g., `String`, `Integer`, records).

### Q5: `static` — what does it mean? Why is static mutable state dangerous?
**A:** Class-level, not instance-level. Shared across all instances. Static mutable state is dangerous because: (1) all threads share it — race conditions, (2) test pollution — one test's state leaks to another, (3) tight coupling — hard to mock.

### Q6: Checked vs unchecked exceptions?
**A:** Checked (`IOException`, `SQLException`): must be declared or caught. Use for recoverable conditions. Unchecked (`RuntimeException` subtypes — `NullPointerException`, `IllegalArgumentException`): compiler doesn't enforce. Use for programming errors. Modern bias: lean on unchecked — checked exceptions don't compose well with lambdas/streams.

### Q7: Try-with-resources — what interface does it require?
**A:** `AutoCloseable` (or the older `Closeable`). Compiler generates `finally` block that calls `close()`. If both the try block and `close()` throw, the close exception is **suppressed** (attached to the primary exception via `getSuppressed()`).

### Q8: Generics — what is type erasure? Why can't you do `new T()`?
**A:** Generics are compile-time only. At runtime, `List<String>` is just `List` (raw type). `T` doesn't exist at runtime — the compiler erases it. So `new T()` is impossible (no type info). Also can't do `instanceof T`, `new T[]`, or catch `T`.

### Q9: PECS — what does it stand for?
**A:** Producer Extends, Consumer Super. `? extends T` — read-only (produce T-or-subtypes out). `? super T` — write-only (consume T-or-subtypes in). `Collection<? extends Number>` lets you read `Number` but not write (compiler can't know if it's `List<Integer>` or `List<Double>`).

### Q10: `Optional` — when appropriate? When wrong?
**A:** Good: return type for "might not exist" lookups (`findById`). Bad: don't use for fields, method parameters, or collections (use empty collection instead). Prefer `orElse`, `orElseGet`, `map`, `ifPresent` over `get()` + `isPresent()`.

### Q11: `equals` vs `==`?
**A:** `==` compares references (same object in memory). `equals()` compares logical equality (value). `String` pool: `"hello" == "hello"` is true (same interned instance), but `new String("hello") == "hello"` is false. Always use `equals()` for objects.

### Q12: What's an enum? When use it?
**A:** Type-safe constant set. Can have methods, fields, constructors. Use instead of `int` constants. Can implement interfaces. `EnumSet` and `EnumMap` are optimised implementations. Can be used in `switch` statements. Implicitly `final` and serialisation-safe singletons.

### Q13: What is `record` (Java 16+)?
**A:** Immutable data carrier. Auto-generates: `private final` fields, constructor, `equals`, `hashCode`, `toString`, getters (`name()` not `getName()`). No setters. Can have compact constructors for validation. Still need defensive copies for mutable field types.

### Q14: What are sealed classes (Java 17+)?
**A:** Restrict which classes can extend/implement. `sealed class Shape permits Circle, Rectangle`. Forces exhaustive pattern matching in `switch`. Between `final` (no subclasses) and open (any subclass).

### Q15: What's the diamond problem and how does Java handle it?
**A:** Two interfaces provide conflicting `default` methods. Java forces the implementing class to override and resolve explicitly. Abstract class can only have single inheritance — no diamond problem there.

### Q16: Composition vs Inheritance — when each?
**A:** Modern preference: composition ("has-a"). Inheritance ("is-a") creates rigid hierarchies, breaks encapsulation (subclass depends on superclass implementation). Composition is more flexible — swap behaviour at runtime. Use inheritance only when Liskov Substitution genuinely holds.

### Q17: What's `instanceof` and pattern matching (Java 16+)?
**A:** `instanceof` checks type at runtime. Pattern matching eliminates the cast: `if (obj instanceof String s)` — `s` is already typed. Combines with sealed classes for exhaustive `switch` (Java 21).

### Q18: Immutability — how to create an immutable class?
**A:** `final` class, `private final` fields, no setters, values via constructor only, defensive copy mutable fields on input AND output. Records give you most of this for free (but still need defensive copies for mutable types like `List`).

---

## Can you answer these cold?

- [ ] Interface vs abstract class — with Java 8 default methods context
- [ ] Overloading (compile time) vs overriding (runtime) — explain dispatch
- [ ] `final` in three contexts — variable, method, class
- [ ] Type erasure — what's lost at runtime, what can't you do
- [ ] PECS — explain with a method signature example
- [ ] Checked vs unchecked — when to use each, why streams prefer unchecked
- [ ] Composition vs inheritance — give a real refactoring example
- [ ] `record` vs regular class — what you get for free, what you don't

[← Back to Index](./00_INDEX.md)
