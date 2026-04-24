### Topic: Heap and Stack — Level 2

> **Related:** [Level 1 — JVM, JRE, and JDK](../level-1/jvm_jre_and_jdk.md) · [Level 3 — JVM Memory Settings](../level-3/jvm_memory_settings.md) · [Level 3 — Garbage Collector](../level-3/garbage_collector.md)

**Why it matters (Karat angle)**
This is where Java memory model questions start. Interviewers ask "where does this variable live?" to see if you understand how method calls, object creation, and garbage collection interact. Wrong answers here signal you've never debugged an OOM or tuned a JVM.

**Core concept**

| | Stack | Heap |
|--|-------|------|
| **Stores** | Method frames: local variables, references, return addresses | Objects (`new`), instance fields, arrays |
| **Lifetime** | Created on method entry, destroyed on return | Lives until no references → GC collects |
| **Thread scope** | One stack per thread (private) | Shared across all threads |
| **Size** | Small (default ~512 KB–1 MB per thread) | Large (configurable: `-Xmx`) |
| **Speed** | Very fast (LIFO push/pop) | Slower (allocation + GC) |
| **Overflow error** | `StackOverflowError` (deep recursion) | `OutOfMemoryError` (heap full) |
| **Structure** | Contiguous, LIFO | Fragmented, managed by GC |

**What goes where:**

```java
// File: topics/level-2/HeapStackDemo.java

public class HeapStackDemo {

    private static final String BANK = "Citi";          // String "Citi" → heap (String pool in Metaspace)
                                                         // static ref BANK → method area (part of heap)

    public static void main(String[] args) {             // main stack frame pushed
        int count = 10;                                  // count → stack (primitive)
        String name = "Alice";                           // "Alice" → heap (String pool); name ref → stack
        Account acc = new Account("Alice", 5000);        // Account object → heap; acc ref → stack

        process(acc);                                    // new stack frame pushed for process()
    }                                                    // main frame popped; acc ref gone → Account eligible for GC

    static void process(Account a) {                     // a → stack (copy of reference, same heap object)
        BigDecimal balance = a.getBalance();              // balance ref → stack; BigDecimal object → heap
        System.out.println(balance);
    }                                                    // process frame popped; a, balance refs gone
}
```

**Visual model:**

```
STACK (per thread)                    HEAP (shared)
┌─────────────────┐                  ┌───────────────────────┐
│ process() frame  │                  │ Account {             │
│   a ─────────────│──────────────────│   name ──→ "Alice"    │
│   balance ───────│──────────┐       │   balance: 5000       │
├─────────────────┤          │       │ }                     │
│ main() frame     │          │       │                       │
│   count: 10      │          └──────→│ BigDecimal { 5000 }   │
│   name ──────────│──────────────────│ "Alice" (String pool) │
│   acc ───────────│──────────────────│                       │
└─────────────────┘                  └───────────────────────┘
```

**Key insight:** References live on the stack; objects live on the heap. When a method returns, its stack frame is popped — references die, but heap objects survive until GC.

**Escape analysis (JVM optimization, Java 6+):**
The JIT compiler can detect objects that don't "escape" a method — they're never returned, stored in a field, or passed to another thread. These objects can be:
- **Stack-allocated** — never hit the heap at all (no GC).
- **Scalar-replaced** — fields unpacked into separate stack variables.

```java
// This object may be stack-allocated (escape analysis):
void calculate() {
    Point p = new Point(3, 4);                          // p doesn't escape this method
    double d = Math.sqrt(p.x * p.x + p.y * p.y);       // JIT may replace with: x=3, y=4 on stack
}
```

**StackOverflowError vs OutOfMemoryError:**

| Error | Cause | Fix |
|-------|-------|-----|
| `StackOverflowError` | Infinite/deep recursion (stack frames exhausted) | Fix the recursion; increase `-Xss` (thread stack size) |
| `OutOfMemoryError: Java heap space` | Too many objects, not enough heap | Fix memory leak; increase `-Xmx` |
| `OutOfMemoryError: Metaspace` | Too many loaded classes (classloader leak) | Fix classloader leak; increase `-XX:MaxMetaspaceSize` |

```java
// File: topics/level-2/StackOverflowDemo.java
void recurse() {
    recurse();                                           // StackOverflowError — stack frames exhausted
}
// Default stack size: ~512 KB. At ~40 bytes per frame → ~12,000 frames before overflow.
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Stack stores method frames (local variables, references, return addresses) — one per thread, LIFO, fast. Heap stores objects — shared across threads, managed by GC, slower.
2. **Why/when:** Understanding this explains why primitives are fast (stack), objects need GC (heap), and deep recursion causes `StackOverflowError` (stack exhausted). It's the foundation for JVM tuning (`-Xmx`, `-Xss`).
3. **Example:** `Account acc = new Account()` — `acc` (reference) goes on the stack, the `Account` object goes on the heap. When the method returns, `acc` is popped from the stack; the object becomes GC-eligible if unreachable.
4. **Gotcha/tradeoff:** Escape analysis can stack-allocate objects that don't escape a method — a significant JIT optimisation. But you can't rely on it — always code assuming objects go to the heap.

**Common pitfalls**
- Assuming all variables are on the heap — primitives and references live on the stack.
- Thinking "stack is faster therefore use primitives everywhere" — the JIT and escape analysis handle this; write clean code first.
- Confusing `StackOverflowError` (deep recursion) with `OutOfMemoryError` (heap full) — different root causes, different fixes.
- Creating thousands of threads with default stack size (1 MB each) — 1000 threads = 1 GB just for stacks. Use `-Xss256k` or virtual threads.

**Self-check question**
You have `String s = new String("hello")`. How many objects are created and where does each live? What about `String s = "hello"`?
