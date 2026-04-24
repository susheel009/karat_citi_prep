### Topic: Lambdas (Effectively Final & Atomic Types) — Level 2

> **Foundation:** [Level 1 — Lambda Expressions](../level-1/lambda_expressions.md)
> **Related:** [Level 2 — Functional Interface and Lambda Expressions](./functional_interface_and_lambda_expressions.md) · [Level 2 — Streams](./streams.md)

**Why it matters (Karat angle)**
L1 covered lambda syntax and common functional interfaces. L2 is about the constraints — why variables captured by lambdas must be effectively final, what "effectively final" means, and how to work around it with `Atomic*` types when you actually need mutation.

**Core concept**

**The rule:** A lambda can only capture local variables that are **final or effectively final** — assigned once and never reassigned.

```java
// File: topics/level-2/EffectivelyFinalDemo.java

// ✅ Effectively final — assigned once, never reassigned
String prefix = "TXN-";
List<String> ids = List.of("001", "002");
ids.forEach(id -> System.out.println(prefix + id));     // compiles — prefix is effectively final

// ❌ NOT effectively final — reassigned
String label = "before";
label = "after";
// ids.forEach(id -> System.out.println(label + id));   // COMPILE ERROR: variable must be final or effectively final
```

**Why this rule exists:**
Lambdas capture the **value** of local variables, not a reference to the stack slot. Local variables live on the thread's stack and disappear when the method returns — but the lambda may outlive the method (passed to another thread, stored in a field). If the lambda could see mutations, it would read stale stack memory. Effectively final = safe to copy the value once.

**What's NOT restricted:**
- **Instance fields and static fields** — they live on the heap, accessible to any thread (but require your own synchronisation).
- **The content of a captured reference** — you can mutate the object a variable points to, just not reassign the variable itself.

```java
// File: topics/level-2/MutableContentDemo.java

// ✅ Captured reference is effectively final, but content is mutable
List<String> results = new ArrayList<>();               // effectively final (never reassigned)
ids.forEach(id -> results.add(prefix + id));            // mutating list content — allowed
System.out.println(results);                            // [TXN-001, TXN-002]

// ✅ Array workaround (ugly but works)
int[] counter = {0};                                    // array ref is effectively final
ids.forEach(id -> counter[0]++);                        // mutating array content — allowed
System.out.println(counter[0]);                         // 2
```

**Atomic types — the proper workaround for thread-safe mutation**

When you need a mutable counter/flag inside a lambda (especially with parallel streams or async code), use `java.util.concurrent.atomic.*`:

| Type | Wraps | Key methods |
|------|-------|------------|
| `AtomicInteger` | `int` | `get()`, `set()`, `incrementAndGet()`, `compareAndSet()` |
| `AtomicLong` | `long` | Same as above |
| `AtomicBoolean` | `boolean` | `get()`, `set()`, `compareAndSet()` |
| `AtomicReference<T>` | `T` | `get()`, `set()`, `compareAndSet()`, `updateAndGet()` |

```java
// File: topics/level-2/AtomicDemo.java

// Thread-safe counter in a lambda
AtomicInteger count = new AtomicInteger(0);
transactions.parallelStream()
    .filter(t -> t.getAmount().compareTo(BigDecimal.ZERO) > 0)
    .forEach(t -> count.incrementAndGet());              // atomic — safe across threads
System.out.println("Positive transactions: " + count.get());

// AtomicReference for accumulating a non-primitive
AtomicReference<BigDecimal> total = new AtomicReference<>(BigDecimal.ZERO);
transactions.forEach(t ->
    total.updateAndGet(current -> current.add(t.getAmount()))
);
// Note: for sums, Collectors.reducing() is cleaner than AtomicReference
```

**How atomics work — CAS (Compare-And-Swap):**
```
1. Read current value
2. Compute new value
3. CAS: "if current value is still X, set it to Y" (hardware atomic instruction)
4. If CAS fails (another thread changed it), retry from step 1
```
No locks, no blocking — lock-free concurrency. Much faster than `synchronized` for simple counters.

**Edge case: capturing loop variables**
```java
// File: topics/level-2/LoopCaptureDemo.java

// ❌ for-loop variable is reassigned each iteration
for (int i = 0; i < 5; i++) {
    // Runnable r = () -> System.out.println(i);         // COMPILE ERROR — i is not effectively final
}

// ✅ Fix: capture in a local variable
for (int i = 0; i < 5; i++) {
    final int captured = i;
    Runnable r = () -> System.out.println(captured);     // each iteration creates a new effectively-final captured
    new Thread(r).start();
}

// ✅ Enhanced for-loop — loop variable is effectively final per iteration
for (String name : names) {
    Runnable r = () -> System.out.println(name);         // each iteration is a new scope
    new Thread(r).start();
}
```

**When to use what:**

| Scenario | Approach |
|----------|---------|
| Read-only capture | Just use the variable (effectively final) |
| Single-threaded mutation in lambda | `int[] counter = {0}` or mutable container |
| Multi-threaded mutation | `AtomicInteger` / `AtomicLong` |
| Complex accumulation | `Collectors.reducing()` or `stream().reduce()` |
| Flag for cancellation | `AtomicBoolean` |

**What to say in the interview (4-beat answer)**
1. **Definition:** Lambdas can only capture local variables that are effectively final — assigned exactly once. This is because lambdas capture the value, not the stack slot, and the lambda may outlive the method scope.
2. **Why/when:** The restriction prevents reading stale stack memory. When you need mutation inside a lambda, use `AtomicInteger`/`AtomicReference` for thread safety, or a single-element array for single-threaded cases.
3. **Example:** `AtomicInteger count = new AtomicInteger(0); stream.forEach(x -> count.incrementAndGet())` — the reference `count` is effectively final; the atomic's internal value is mutated safely via CAS.
4. **Gotcha/tradeoff:** `int[] counter = {0}` works but isn't thread-safe. In a `parallelStream`, use `AtomicInteger`. For sums/aggregations, prefer `Collectors.reducing()` — it's cleaner and avoids side effects entirely.

**Common pitfalls**
- Trying to reassign a local variable inside a lambda — compile error.
- Using `int[] counter` in a parallel stream — race condition (not atomic).
- Using `AtomicReference` for complex operations that aren't CAS-friendly — `updateAndGet` retries can starve under contention.
- Capturing a loop variable from a `for(int i=...)` loop — must copy to a local `final int`.
- Mutating instance fields from a lambda without synchronisation — compiles but has race conditions.

**Self-check question**
Why does `for (String s : list) { executor.submit(() -> process(s)); }` compile, but `for (int i = 0; i < list.size(); i++) { executor.submit(() -> process(list.get(i))); }` does not? What's the fix?
