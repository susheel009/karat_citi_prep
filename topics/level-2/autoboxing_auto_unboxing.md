### Topic: Autoboxing / Auto-unboxing (Compiler Issues) — Level 2

> **Foundation:** [Level 1 — Autoboxing / Auto-unboxing](../level-1/autoboxing_auto_unboxing.md)

**Why it matters (Karat angle)**
L1 covered what autoboxing is and the Integer cache. L2 is about the bugs it causes — compiler errors you'll see in real code, data truncation you won't catch until production, and performance traps seniors are expected to avoid.

**Core concept**

**Compiler errors and warnings around autoboxing:**

```java
// File: topics/level-2/AutoboxingCompilerDemo.java

// 1. Ambiguous method calls
void process(int x)     { System.out.println("primitive"); }
void process(Integer x) { System.out.println("wrapper"); }
// process(5);           // calls process(int) — no boxing needed, primitive wins
// process(null);        // calls process(Integer) — null can't be int

// But with overloads like:
void calc(long x)       { }
void calc(Integer x)    { }
// calc(5);              // AMBIGUOUS — compiler error! Widening (int→long) vs boxing (int→Integer)
```

**Widening vs boxing precedence:**
1. **Exact match** → `int` param for `int` arg.
2. **Widening** → `long` param for `int` arg.
3. **Boxing** → `Integer` param for `int` arg.
4. **Varargs** → `int...` param for `int` arg (lowest priority).

Boxing and widening together (e.g., `int → Long`) **never** happens implicitly.

```java
// File: topics/level-2/WideningVsBoxing.java
Long x = 5;              // ❌ COMPILE ERROR — int can't auto-box to Long
Long y = 5L;             // ✅ long → Long (boxing)
long z = new Integer(5); // ✅ Integer → int (unboxing) → long (widening)
```

**Data truncation:**

```java
// File: topics/level-2/TruncationDemo.java

// Narrowing cast — silent data loss
int big = 130;
byte b = (byte) big;                                    // -126 (wraps: 130 - 256 = -126)

// Double → int — decimal truncation
double price = 99.99;
int rounded = (int) price;                              // 99 — not 100

// Long → int — overflow
long bigLong = 3_000_000_000L;
int truncated = (int) bigLong;                          // -1294967296 (wraps)

// Float precision loss
float f = 16_777_217;                                   // stored as 16_777_216.0 (lost 1)
// float has ~7 significant digits; int has up to 10

// BigDecimal → int
BigDecimal bd = new BigDecimal("99.95");
int val = bd.intValue();                                // 99 — silent truncation
int safe = bd.intValueExact();                          // ArithmeticException if fractional
```

**NullPointerException from unboxing:**

```java
// File: topics/level-2/NullUnboxingDemo.java

Integer count = null;
int x = count;                                          // NPE! null can't unbox to int

// Common in real code:
Map<String, Integer> scores = new HashMap<>();
int score = scores.get("missing");                      // NPE — get() returns null, unboxed to int

// Fix: use getOrDefault or check for null
int safe = scores.getOrDefault("missing", 0);
```

**Performance traps:**

```java
// File: topics/level-2/AutoboxingPerfDemo.java

// WRONG — autoboxing on every iteration
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i;                                            // Long = Long + long → unbox, add, rebox
}
// Creates ~1M Long objects — 10x slower than using primitive long

// RIGHT
long sum2 = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum2 += i;                                           // pure primitive — no boxing
}
```

**Edge case: equality traps beyond the cache**

```java
// File: topics/level-2/EqualityTraps.java

// Integer cache: -128 to 127 → == works by accident
Integer a = 127;  Integer b = 127;
System.out.println(a == b);                              // true (cached)

Integer c = 128;  Integer d = 128;
System.out.println(c == d);                              // false (new objects)

// Boolean cache: always cached (only two values)
Boolean t1 = true; Boolean t2 = true;
System.out.println(t1 == t2);                            // true (always)

// Double: NEVER cached
Double d1 = 1.0; Double d2 = 1.0;
System.out.println(d1 == d2);                            // false (always new objects)

// Comparing across types:
Integer i = 0;
System.out.println(i.equals(0L));                        // false — Integer.equals(Long) is type mismatch
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Autoboxing compiler issues include ambiguous overloads (widening vs boxing), NPE from unboxing null wrappers, and silent data truncation when narrowing (long→int, double→int, float precision loss).
2. **Why/when:** These cause production bugs — a `Map.get()` returning `null` that's unboxed to `int` is an NPE. A `Long sum = 0L` in a loop creates millions of objects.
3. **Example:** `int x = map.get("key")` compiles fine but throws NPE at runtime if the key is missing, because `null` can't unbox. Use `getOrDefault` or check for null.
4. **Gotcha/tradeoff:** Widening + boxing can't combine — `int → Long` doesn't compile. But unboxing + widening works — `Integer → int → long` is fine. Know the precedence: exact > widen > box > varargs.

**Common pitfalls**
- Unboxing `null` from `Map.get()` — the most common source of autoboxing NPEs in production.
- Using wrapper types for loop counters — pointless boxing overhead.
- `Integer.equals(Long)` returning `false` even for the same numeric value — type mismatch.
- Assuming `==` works for all `Integer` values — only cached range (-128 to 127).
- Silent `double → int` truncation — use `Math.round()` or `BigDecimal.intValueExact()`.

**Self-check question**
What's the output of: `Integer a = 1; Long b = 1L; System.out.println(a.equals(b));`? Why? How would you safely compare them?
