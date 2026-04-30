# 05 — Strings & Immutability

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## Rapid-Fire Q&A

### Q1: Why is `String` immutable?
**A:** Security (can't modify file paths, class names), thread safety (shared without synchronisation), String pool works because interned strings can't change, `hashCode` can be cached (used as HashMap keys). `String` is `final` — can't subclass.

### Q2: What's the String pool?
**A:** JVM maintains a pool of unique string literals in the heap (since Java 7 — was in PermGen before). `"hello"` and `"hello"` point to the same object. `new String("hello")` creates a separate object on the heap (not in pool). `intern()` forces a string into the pool.

### Q3: `==` vs `equals()` for Strings?
**A:** `==` compares references. `equals()` compares characters. `"hello" == "hello"` → true (same pool object). `new String("hello") == "hello"` → false (different objects). Always use `equals()`.

### Q4: `String` vs `StringBuilder` vs `StringBuffer`?
**A:** `String`: immutable — concatenation creates new objects. `StringBuilder`: mutable, not thread-safe, fast. `StringBuffer`: mutable, thread-safe (synchronized), slower. Use `StringBuilder` for loops/building. `+` in a single expression is optimised by the compiler.

### Q5: Why not concatenate strings in a loop?
**A:** Each `+=` creates a new `String` object. N iterations = N intermediate strings = O(n²) copying. `StringBuilder.append()` is O(n) total — single buffer.

### Q6: What does `String.intern()` do?
**A:** Adds the string to the pool (or returns the existing pooled instance). After `intern()`, you can use `==` for comparison. Rarely needed in application code — mainly for memory optimisation in frameworks.

### Q7: What's immutability in general? How do you make a class immutable?
**A:** Object state can't change after construction. Rules: `final` class, `private final` fields, no setters, values only via constructor, defensive copy mutable fields (input AND output). Java 16 `record` gives most of this for free.

### Q8: `char[]` vs `String` for passwords?
**A:** `String` stays in memory until GC collects it (and in the string pool potentially forever). `char[]` can be explicitly zeroed after use (`Arrays.fill(chars, '0')`). Security-sensitive — OWASP recommends `char[]`.

---

## Can you answer these cold?

- [ ] String immutability — why it matters (security, thread safety, pool, hashCode caching)
- [ ] String pool — where it lives, `intern()`, `new String()` vs literal
- [ ] `StringBuilder` vs `StringBuffer` — when each
- [ ] Loop concatenation performance — O(n²) problem
- [ ] How to create an immutable class — 5 rules

[← Back to Index](./00_INDEX.md)
