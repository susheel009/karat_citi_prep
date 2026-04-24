### Topic: Functional Interface and Lambda Expressions — Level 2

> **Foundation:** [Level 1 — Lambda Expressions](../level-1/lambda_expressions.md)
> **Related:** [Level 2 — Lambdas (effectively final, atomics)](./lambdas.md) · [Level 2 — Streams](./streams.md)

**Why it matters (Karat angle)**
Interviewers ask "what's a functional interface?" then follow up with "can it have default methods?" and "write a custom one." The distinction between `Function`, `Predicate`, `Consumer`, `Supplier`, and `UnaryOperator` must be instant.

**Core concept**

A **functional interface** has exactly **one abstract method** (SAM — Single Abstract Method). It can have any number of `default` and `static` methods. The `@FunctionalInterface` annotation is optional but enforces the rule at compile time.

**The four core functional interfaces:**

| Interface | Signature | Purpose | Example |
|----------|-----------|---------|---------|
| `Function<T,R>` | `R apply(T t)` | Transform T → R | `s -> s.length()` |
| `Predicate<T>` | `boolean test(T t)` | Test a condition | `n -> n > 0` |
| `Consumer<T>` | `void accept(T t)` | Side effect (no return) | `s -> log.info(s)` |
| `Supplier<T>` | `T get()` | Produce a value (no input) | `() -> UUID.randomUUID()` |

**Extended variants:**

| Interface | Signature | Use |
|----------|-----------|-----|
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs, one output |
| `BiPredicate<T,U>` | `boolean test(T t, U u)` | Two inputs, boolean |
| `BiConsumer<T,U>` | `void accept(T t, U u)` | Two inputs, no return |
| `UnaryOperator<T>` | `T apply(T t)` | `Function<T,T>` — same type in/out |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | `BiFunction<T,T,T>` |

**Working code example**

```java
// File: topics/level-2/FunctionalInterfaceDemo.java

// ---------- Using built-in functional interfaces ----------
Function<String, Integer> length = String::length;
Predicate<Integer> isPositive = n -> n > 0;
Consumer<String> printer = System.out::println;
Supplier<Instant> now = Instant::now;

System.out.println(length.apply("hello"));               // 5
System.out.println(isPositive.test(-3));                  // false
printer.accept("logged");                                 // "logged"
System.out.println(now.get());                            // 2026-04-23T...

// ---------- Composition ----------
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> pipeline = trim.andThen(upper);  // trim first, then uppercase
System.out.println(pipeline.apply("  hello  "));          // "HELLO"

Predicate<String> notEmpty = Predicate.not(String::isEmpty);
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> combined = notEmpty.and(startsWithA);
System.out.println(combined.test("Alice"));               // true
```

**Writing a custom functional interface:**

```java
// File: topics/level-2/CustomFunctionalInterface.java

@FunctionalInterface
public interface TransactionValidator {
    boolean validate(Transaction txn);                    // the ONE abstract method

    // Default methods are fine — don't count as abstract
    default TransactionValidator and(TransactionValidator other) {
        return txn -> this.validate(txn) && other.validate(txn);
    }

    // Static factory methods are fine too
    static TransactionValidator amountAbove(BigDecimal min) {
        return txn -> txn.getAmount().compareTo(min) > 0;
    }
}

// Usage:
TransactionValidator aboveThreshold = TransactionValidator.amountAbove(new BigDecimal("10000"));
TransactionValidator notPending = txn -> !"PENDING".equals(txn.getStatus());
TransactionValidator combined = aboveThreshold.and(notPending);

transactions.stream()
    .filter(combined::validate)                           // method reference to custom interface
    .forEach(System.out::println);
```

**Functional interface vs lambda — the relationship:**
```java
// A lambda is an IMPLEMENTATION of a functional interface
Runnable r = () -> System.out.println("run");
// Equivalent to:
Runnable r2 = new Runnable() {
    @Override public void run() { System.out.println("run"); }
};
// The lambda is syntactic sugar for an anonymous class implementing the SAM.
```

**Edge cases:**

```java
// ✅ One abstract method + defaults + statics = functional interface
@FunctionalInterface
interface Transformer<T> {
    T transform(T input);                                 // SAM
    default Transformer<T> andThen(Transformer<T> after) {
        return input -> after.transform(this.transform(input));
    }
    static <T> Transformer<T> identity() { return t -> t; }
}

// ❌ Two abstract methods = NOT a functional interface
interface NotFunctional {
    void doA();
    void doB();                                           // second abstract → can't be lambda target
}

// ✅ Inheriting one abstract method from a parent
interface A { void doSomething(); }
@FunctionalInterface
interface B extends A {}                                  // inherits doSomething() → valid SAM
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A functional interface has exactly one abstract method. It can have multiple `default` and `static` methods. `@FunctionalInterface` enforces this at compile time. Lambdas are anonymous implementations of functional interfaces.
2. **Why/when:** They enable functional-style programming — pass behaviour as arguments. Core to Streams, `CompletableFuture`, Spring's `@FunctionalInterface`-based callback patterns.
3. **Example:** `Predicate<T>` has one abstract method `test(T)`, plus `and()`, `or()`, `negate()` as defaults. You compose predicates: `isPositive.and(isEven)` — each is a lambda, combined into a pipeline.
4. **Gotcha/tradeoff:** An interface with one abstract method is a functional interface whether or not you annotate it. The annotation just gives you a compile error if someone accidentally adds a second abstract method.

**Common pitfalls**
- Thinking `default` methods disqualify a functional interface — only abstract methods count.
- Confusing `Function` (one input, one output) with `Consumer` (one input, no output) — wrong type breaks compilation.
- Forgetting primitive specialisations (`IntPredicate`, `LongFunction`) — avoid autoboxing in hot paths.
- Writing a custom functional interface when a built-in one (`Predicate`, `Function`) already fits.

**Self-check question**
Is `Comparator<T>` a functional interface? It has `compare(T, T)` as abstract, but also `equals(Object)` which overrides `Object.equals`. How many abstract methods does it really have?
