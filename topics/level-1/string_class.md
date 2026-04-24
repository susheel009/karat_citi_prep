### Topic: String Class — Level 1

> **Advanced coverage:** [Level 2 — String Class (encoding, ASCII)](../level-2/string_class.md) · [Level 3 — Strings (formatting)](../level-3/strings.md)

**Why it matters (Karat angle)**
String questions are universal warm-up material. Interviewers use them to probe four things at once: immutability, memory model (heap vs pool), thread safety (Buffer vs Builder), and your habit around `==` vs `.equals()`.

**Core concept**

Strings are **immutable** in Java. Every "modification" (`concat`, `replace`, `substring`) returns a new `String` object; the original is untouched. This enables aggressive sharing: the **String pool** in the JVM metaspace deduplicates identical literals so `"hello" == "hello"` is `true`.

**String pool mechanics**
- Compile-time literals go in the pool automatically.
- `new String("x")` always allocates a fresh heap object, bypassing the pool.
- `"x".intern()` forces a string into the pool and returns the pooled reference.

| Class | Mutable? | Thread-safe? | Relative speed | Use when |
|-------|---------|--------------|----------------|----------|
| `String` | ❌ | N/A (immutable) | Fast read | Constants, method params, return types |
| `StringBuffer` | ✅ | ✅ (synchronized methods) | Slower (locks) | Building strings across threads (rare) |
| `StringBuilder` | ✅ | ❌ | Fastest | Building strings in a loop, single thread |

**Why immutability matters beyond syntax:**
- **Thread safety for free** — strings can be shared across threads without locks.
- **Safe as HashMap keys** — hashcode can't change after insertion.
- **Security** — a `String` password that's been handed to a library can't be modified by that library. (But the string sits in memory until GC — use `char[]` for truly sensitive data; see Level 2.)

**Performance trap: concat in a loop**
```java
String s = "";
for (int i = 0; i < n; i++) s += i;        // O(n²) — each += creates a new String
```
Each `+=` allocates a new `String` of length `previous + i`. For `n=10_000`, that's ~50 million characters copied. Use `StringBuilder` for O(n).

**Real-world use cases**
- **Building SQL or log lines in a loop:** `StringBuilder` (single thread).
- **Shared mutable text across threads:** `StringBuffer` (rare — usually redesign with immutability instead).
- **String constants, enum name, config keys:** `String` literals — auto-pooled, free deduplication.
- **JSON serialisation output:** let Jackson/Gson handle it, but internally they use `StringBuilder`.

**Working code example**
```java
// File: topics/level-1/StringDemo.java

public class StringDemo {

    public static void main(String[] args) {

        // ---------- String pool ----------
        String a = "hello";
        String b = "hello";
        System.out.println(a == b);               // true  — same pool reference
        System.out.println(a.equals(b));          // true  — content

        // new String() always creates a heap object outside the pool
        String c = new String("hello");
        System.out.println(a == c);               // false — different ref
        System.out.println(a.equals(c));          // true  — same content
        System.out.println(a == c.intern());      // true  — intern returns pooled ref

        // ---------- Immutability ----------
        String s = "Java";
        String t = s.concat(" 21");               // new object; s unchanged
        System.out.println(s);                    // Java
        System.out.println(t);                    // Java 21

        // ---------- StringBuilder: efficient for loops ----------
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 5; i++) sb.append(i).append(",");
        System.out.println(sb);                   // 0,1,2,3,4,

        // ---------- Common operations ----------
        String csv = "a,b,c,d";
        String[] parts = csv.split(",");          // ["a","b","c","d"]
        String joined = String.join("-", parts);  // "a-b-c-d"
        System.out.println(joined);

        // ---------- equals vs == pitfall ----------
        Integer id = 100;
        String idA = "user-" + id;
        String idB = "user-" + id;
        System.out.println(idA == idB);           // false — compiled concat, not pooled
        System.out.println(idA.equals(idB));      // true
    }
}
```

**Edge case: String pool and performance**
```java
// File: topics/level-1/StringPoolDemo.java
// Reading a large file of repeated codes? intern() saves memory if many dups.
List<String> codes = ...;                        // millions of "TXN1", "TXN2", ...
List<String> interned = codes.stream()
                             .map(String::intern)
                             .toList();
// All duplicates now point to the same pool reference — heap use drops.
// Tradeoff: intern() is synchronised internally; don't intern in hot paths.
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `String` is an immutable char sequence backed by a `byte[]` (or `char[]` pre-Java 9). `StringBuilder` is mutable, single-thread; `StringBuffer` is mutable, thread-safe via synchronised methods.
2. **Why/when:** Use `String` for constants and cross-boundary data; `StringBuilder` when building a string in a loop; `StringBuffer` only when actually shared across threads (usually redesign instead).
3. **Example:** Concatenating 1000 strings with `+=` creates 1000 new objects (O(n²) copying); with `StringBuilder`, one buffer grows in place (O(n)).
4. **Gotcha/tradeoff:** `==` compares references, `.equals()` compares content — `new String("x") == "x"` is `false` even though content is identical, because `new` bypasses the pool.

**Common pitfalls**
- Using `==` to compare string content — works for pooled literals by coincidence, fails for dynamic strings.
- Concatenating in a loop with `+=` — O(n²); use `StringBuilder`.
- Over-using `intern()` — synchronised, can cause metaspace pressure at scale.
- Storing passwords in `String` — they linger in memory until GC; use `char[]` and clear after use (covered in Level 2).

**Self-check question**
Given `String a = "x" + "y";` and `String b = new String("xy");`, which of `a == b`, `a.equals(b)`, `a == b.intern()` are true, and why?
