### Topic: Autoboxing / Auto-unboxing — Level 1

> **Advanced coverage:** [Level 2 — Autoboxing (compiler errors, truncation)](../level-2/autoboxing_auto_unboxing.md)

**Why it matters (Karat angle)**
Autoboxing is the hidden performance trap in Java. Interviewers ask this to check whether you know that `Integer` ≠ `int`, that boxing allocates, and that the Integer cache makes `==` work for small numbers by accident.

**Core concept**

**Autoboxing:** automatic conversion from primitive → wrapper (`int` → `Integer`).
**Auto-unboxing:** automatic conversion wrapper → primitive (`Integer` → `int`).

The compiler inserts `Integer.valueOf(n)` and `integer.intValue()` behind the scenes.

**Primitive ↔ wrapper pairs**

| Primitive | Wrapper |
|-----------|---------|
| `boolean` | `Boolean` |
| `byte` | `Byte` |
| `short` | `Short` |
| `char` | `Character` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |

**Why wrappers exist**
- Generics only work on objects (`List<int>` is illegal; `List<Integer>` works).
- `Map<K, V>` values and `Collection<E>` elements must be objects.
- Null is meaningful (primitive has no "absent" state; wrapper allows `null`).

**The Integer cache — the `==` trap**

`Integer.valueOf(n)` caches values for `-128` to `127` (configurable up via `-XX:AutoBoxCacheMax`). Same literal value in this range returns the **same cached object**:

```java
Integer a = 100, b = 100;
System.out.println(a == b);     // true  — both refer to the cached object
Integer c = 200, d = 200;
System.out.println(c == d);     // false — outside the cache; different objects
System.out.println(c.equals(d));// true  — always use .equals()
```

This is why you NEVER use `==` on wrappers — the result depends on the value.

**The null-unbox NPE trap**

Unboxing `null` throws `NullPointerException`:

```java
Map<String, Integer> scores = new HashMap<>();
int alice = scores.get("Alice");    // Alice isn't in the map → returns null → NPE on unbox
```

**Performance cost — autoboxing in a loop**

```java
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) sum += i;    // ❌ 1M Long allocations
// Each `sum += i` does: long s = sum.longValue() + i; sum = Long.valueOf(s);
```
Fix: use `long sum = 0L;` — no boxing in the loop.

**Mental model:** autoboxing is the compiler silently wrapping and unwrapping. Free syntax, but every box = one heap allocation.

**Real-world use cases**
- **Generics:** `List<Integer>`, `Map<String, Long>` — boxing is unavoidable here.
- **`Optional<Integer>`** — requires a wrapper; or use `OptionalInt` to avoid boxing.
- **Streams of primitives:** use `IntStream`, `LongStream`, `DoubleStream` instead of `Stream<Integer>` to avoid boxing on every element.
- **Nullable DB column → entity field:** use `Integer` (nullable) not `int` (0 by default).

**Working code example**
```java
// File: topics/level-1/AutoboxingDemo.java
import java.util.*;
import java.util.stream.*;

public class AutoboxingDemo {

    public static void main(String[] args) {

        // ---------- Autobox on assignment ----------
        int primitive = 42;
        Integer boxed = primitive;            // compiler inserts Integer.valueOf(primitive)
        int back = boxed;                      // compiler inserts boxed.intValue()

        // ---------- Autobox on collection insertion ----------
        List<Integer> nums = new ArrayList<>();
        nums.add(1);                           // 1 (int) → Integer.valueOf(1)
        int first = nums.get(0);               // unbox

        // ---------- Integer cache trap ----------
        Integer a = 100, b = 100;
        Integer c = 200, d = 200;
        System.out.println(a == b);            // true  — cache hit
        System.out.println(c == d);            // false — outside cache
        System.out.println(c.equals(d));       // true  — correct way
        System.out.println(Objects.equals(c, d)); // true  — null-safe correct way

        // ---------- Null-unbox NPE ----------
        Map<String, Integer> scores = new HashMap<>();
        try {
            int score = scores.get("Alice");   // null → NPE
        } catch (NullPointerException e) {
            System.out.println("Caught NPE from unboxing null");
        }
        // Safer: Integer score = scores.get("Alice"); if (score != null) ...

        // ---------- Performance trap: boxing in a loop ----------
        Long slowSum = 0L;
        for (long i = 0; i < 100_000; i++) slowSum += i;   // boxes on every iteration
        System.out.println("Slow sum: " + slowSum);

        long fastSum = 0L;
        for (long i = 0; i < 100_000; i++) fastSum += i;   // no boxing
        System.out.println("Fast sum: " + fastSum);

        // ---------- Stream primitive specialisation ----------
        long streamSum = IntStream.rangeClosed(1, 100).sum();    // no boxing
        System.out.println("Stream sum: " + streamSum);
    }
}
```

**Edge case: data truncation on narrowing conversion**
```java
long big = 10_000_000_000L;
Integer intRef = (int) big;           // explicit cast truncates to int range
System.out.println(intRef);           // truncated — data loss
// Auto-unbox of Integer → long is safe (widening), but Long → int requires explicit cast
```

**Edge case: `==` on same-expression autoboxing inside a ternary**
```java
Integer i = true ? Integer.valueOf(1) : 0;   // 0 is int; result promoted — unboxes i
// Subtle: the ternary's result type unifies the branches, triggering unbox
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Autoboxing is the compiler's silent conversion from primitive to wrapper (`int` → `Integer` via `Integer.valueOf`); auto-unboxing is the reverse (`intValue()`).
2. **Why/when:** It exists so generics (`List<Integer>`) and nullable fields can work. Unavoidable in collections and streams of objects — but use primitive-specialised streams (`IntStream`, `LongStream`) when you can.
3. **Example:** `Integer a = 1000, b = 1000; a == b` is `false` — these are outside Java's `-128..127` cache, so they're different objects. Use `.equals()` or `Objects.equals()` for wrapper equality.
4. **Gotcha/tradeoff:** Unboxing a null throws NPE — a common bug when a `Map.get()` returns null and you assign to a primitive. And autoboxing in a tight loop allocates heavily; use primitive types for hot loops.

**Common pitfalls**
- Using `==` on wrappers — works by accident for cached values (`-128..127`), fails for larger values.
- Unboxing a null (e.g., `int x = map.get(key)` when key absent) — NPE.
- Using wrapper types in a hot loop — each operation allocates a new object.
- `Integer` as a method parameter when primitive would do — adds allocation cost and accepts null.

**Self-check question**
Why does `Long a = 127L; Long b = 127L; a == b` return `true` but `Long a = 128L; Long b = 128L; a == b` return `false`? What's the underlying Java mechanism?
