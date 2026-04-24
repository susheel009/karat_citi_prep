### Topic: Lambda Expressions — Level 1

> **Advanced coverage:** [Level 2 — Lambdas (final/Atomic)](../level-2/lambdas.md) · [Level 2 — Functional Interface and Lambda Expressions](../level-2/functional_interface_and_lambda_expressions.md)

**Why it matters (Karat angle)**
Lambdas are the foundation of modern Java. Every codebase uses them — in streams, comparators, Spring `@Async` callbacks, Reactor pipelines. Interviewers expect syntax fluency AND the mental model (what's a functional interface, why does effectively-final matter).

**Core concept**

A **lambda** is a concise way to implement a **functional interface** — an interface with exactly one abstract method (SAM). Syntax forms:

| Form | Example |
|------|---------|
| No args | `() -> 42` |
| Single arg, no type | `x -> x * 2` |
| Multiple args | `(a, b) -> a + b` |
| Typed args | `(int a, int b) -> a + b` |
| Block body | `(a, b) -> { int s = a + b; return s; }` |
| Annotated arg (Java 11+) | `(@NonNull var x) -> x.trim()` |

**What a lambda is NOT:**
- Not an anonymous class under the hood (it uses `invokedynamic` — cheaper).
- Not a closure in the full functional-programming sense — captured variables must be **effectively final**.

**Built-in functional interfaces (`java.util.function`)**

| Interface | Abstract method | When to use |
|-----------|-----------------|-------------|
| `Predicate<T>` | `boolean test(T)` | Filters, validation |
| `Function<T,R>` | `R apply(T)` | Transformations |
| `Consumer<T>` | `void accept(T)` | Side effects (logging, mutation) |
| `Supplier<T>` | `T get()` | Lazy producers, defaults |
| `BiFunction<T,U,R>` | `R apply(T, U)` | Two-input transform |
| `UnaryOperator<T>` | `T apply(T)` | Same-type transform (subtype of Function) |

**Method references** — a shorter lambda form for four cases:

| Kind | Example | Equivalent lambda |
|------|---------|-------------------|
| Static | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Instance on a known object | `logger::info` | `msg -> logger.info(msg)` |
| Instance on an unbound type | `String::length` | `s -> s.length()` |
| Constructor | `ArrayList::new` | `() -> new ArrayList<>()` |

**Effectively final explained:** any variable captured by a lambda must either be declared `final` OR never reassigned after its initial assignment. The compiler treats "never reassigned" as implicitly final. Why the rule? Because lambdas can be passed to other threads and executed later — if the captured variable kept changing, you'd have visibility/race issues. Java sidesteps this by freezing capture.

**Mental model:** a lambda is a tiny object carrying its captured variables in its frozen state, plus a method pointer to the code you wrote.

**Real-world use cases**
- **Stream pipelines:** `list.stream().filter(x -> x.age > 18).map(User::name).collect(toList())`.
- **Comparator:** `list.sort(Comparator.comparing(User::lastName).thenComparing(User::firstName))`.
- **Event callbacks:** `button.addActionListener(e -> handleClick())`.
- **Spring:** `@Async` methods returning `CompletableFuture`; `RestTemplate` error handlers; transaction templates.
- **Retry/lazy defaults:** `Map.computeIfAbsent(key, k -> expensiveLookup(k))`.

**Working code example**
```java
// File: topics/level-1/LambdaDemo.java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class LambdaDemo {

    // Custom functional interface
    @FunctionalInterface
    interface Validator<T> {
        boolean validate(T value);
    }

    public static void main(String[] args) {

        // ---------- Lambda = implementation of a SAM interface ----------
        Validator<String> notEmpty = s -> !s.isEmpty();
        System.out.println(notEmpty.validate("hello"));   // true
        System.out.println(notEmpty.validate(""));        // false

        // ---------- Built-in functional interfaces ----------
        Predicate<Integer>        isEven = n -> n % 2 == 0;
        Function<String, Integer> len    = String::length;                 // method ref
        Consumer<String>          print  = System.out::println;
        Supplier<List<String>>    newList = ArrayList::new;                // constructor ref

        // ---------- Combining with streams ----------
        List<String> names = List.of("Alice", "Bob", "Charlie", "Dan");
        names.stream()
             .filter(n -> n.length() > 3)
             .map(String::toUpperCase)
             .sorted()
             .forEach(print);
        // Output: ALICE, CHARLIE

        // ---------- Predicate composition (and, or, negate) ----------
        Predicate<Integer> positive = n -> n > 0;
        Predicate<Integer> positiveEven = positive.and(isEven);
        System.out.println(positiveEven.test(4));    // true
        System.out.println(positiveEven.test(-2));   // false

        // ---------- Effectively final ----------
        String prefix = "TXN-";                       // never reassigned — effectively final
        Function<Integer, String> tag = id -> prefix + id;
        System.out.println(tag.apply(42));            // TXN-42
        // prefix = "NEW-";                           // uncommenting this breaks the lambda

        // ---------- Comparator with method refs ----------
        List<String> mutable = new ArrayList<>(names);
        mutable.sort(Comparator.comparing(String::length).thenComparing(Comparator.naturalOrder()));
        System.out.println(mutable);                  // sorted by length, then alphabetical
    }
}
```

**Edge case: mutable state workaround**
```java
// Can't do this — counter must be effectively final
// int counter = 0;
// list.forEach(x -> counter++);   // compile error

// Workaround 1 — AtomicInteger (thread-safe too)
AtomicInteger counter = new AtomicInteger();          // reference is final; value mutates
list.forEach(x -> counter.incrementAndGet());

// Workaround 2 — single-element array (hack, but valid)
int[] counterArr = {0};
list.forEach(x -> counterArr[0]++);                    // array ref is final; contents mutate
```

**What to say in the interview (4-beat answer)**
1. **Definition:** A lambda is an inline implementation of a functional interface (single abstract method), enabling functional-style programming without anonymous-class boilerplate.
2. **Why/when:** Whenever you pass behaviour as a parameter — stream predicates, sort comparators, callbacks, async tasks. It replaces five-line anonymous classes with a one-line expression.
3. **Example:** `list.stream().filter(x -> x > 0).map(x -> x * 2).sum()` — three readable lines that would otherwise be a for-loop with an accumulator.
4. **Gotcha/tradeoff:** Captured variables must be effectively final. If you need mutable state inside a lambda, use `AtomicInteger` (thread-safe) or an array wrapper (single-thread). The lambda implementation uses `invokedynamic`, so it's typically cheaper than a compiled anonymous class.

**Common pitfalls**
- Trying to reassign a captured variable inside a lambda — compile error; use `AtomicReference`, `AtomicInteger`, or a one-element array.
- Over-using method references when a lambda reads more clearly — `s -> s.trim().toLowerCase()` is clearer than chained refs.
- Forgetting that a lambda needs a functional interface target type — `Object f = () -> 42;` doesn't compile; `Runnable f = () -> System.out.println("ok");` does.
- Thinking `@FunctionalInterface` is required — it's not; it's a compile-time check that the interface has exactly one abstract method.

**Self-check question**
Define a functional interface. Can an interface with one abstract method and two `default` methods be used as a lambda target? What about two abstract methods?
