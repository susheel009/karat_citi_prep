### Topic: Block Types — Level 2

> **Related:** [Level 1 — Java Fundamentals](../level-1/java_fundamentals.md) · [Level 1 — Object-Oriented Programming](../level-1/object_oriented_programming_oop.md)

**Why it matters (Karat angle)**
Static and instance blocks are rarely written in new code but show up in legacy Citi codebases and are a favourite Karat "what runs when?" question. Knowing execution order signals understanding of the JVM class-loading lifecycle.

**Core concept**

Java has two block types that run automatically:

| Block | Syntax | When it runs | How many times | Use case |
|-------|--------|-------------|:--------------:|----------|
| **Static block** | `static { ... }` | When the class is **loaded** (first reference) | Once per class | Load native libs, populate static lookup tables |
| **Instance block** | `{ ... }` | Before **every** constructor, after `super()` | Once per object | Common init shared across multiple constructors (rare) |

**Execution order:**

```
1. Static blocks (top to bottom, once per class load)
2. Instance blocks (top to bottom, before constructor body)
3. Constructor body
```

```java
// File: topics/level-2/BlockTypesDemo.java

public class BlockTypesDemo {

    // 1. Static block — runs once when class is loaded
    static {
        System.out.println("A: static block");
    }

    // 2. Static field initialiser (interleaved with static blocks in source order)
    private static final String CONFIG = initConfig();
    private static String initConfig() {
        System.out.println("B: static field init");
        return "loaded";
    }

    // 3. Instance block — runs before every constructor
    {
        System.out.println("C: instance block");
    }

    // 4. Instance field initialiser
    private String name = initName();
    private String initName() {
        System.out.println("D: instance field init");
        return "default";
    }

    // 5. Constructor
    public BlockTypesDemo() {
        System.out.println("E: constructor");
    }

    public static void main(String[] args) {
        System.out.println("F: main starts");
        new BlockTypesDemo();
        System.out.println("---");
        new BlockTypesDemo();
    }
}
```

**Output:**
```
A: static block
B: static field init
F: main starts
C: instance block
D: instance field init
E: constructor
---
C: instance block
D: instance field init
E: constructor
```

Key observations:
- Static block + static field init run **before** `main` (on class load).
- They run **once** — second `new` doesn't repeat them.
- Instance block runs **every** time, before the constructor body.

**Static block — real-world uses**

```java
// File: topics/level-2/StaticBlockUses.java

// 1. Loading native libraries
public class Crypto {
    static {
        System.loadLibrary("crypto_jni");
    }
}

// 2. Populating a static lookup table
public class CurrencyLookup {
    private static final Map<String, String> CODES;
    static {
        Map<String, String> map = new HashMap<>();
        map.put("USD", "US Dollar");
        map.put("EUR", "Euro");
        map.put("GBP", "British Pound");
        CODES = Collections.unmodifiableMap(map);
    }
    public static String getName(String code) { return CODES.get(code); }
}
// Modern alternative: Map.of("USD", "US Dollar", "EUR", "Euro", ...)
```

**Instance block — rare but valid**
```java
// File: topics/level-2/InstanceBlockUse.java

// When multiple constructors share init logic
public class AuditEntry {
    private final Instant created;
    private final String traceId;

    {
        // Common to ALL constructors — no duplication
        this.created = Instant.now();
        this.traceId = UUID.randomUUID().toString();
    }

    public AuditEntry() {}
    public AuditEntry(String detail) { /* detail-specific logic */ }
}
// Modern alternative: initialise fields inline or use a private init method.
```

**Edge case: static blocks and exceptions**
```java
// File: topics/level-2/StaticBlockException.java
public class BadInit {
    static {
        if (System.getenv("DB_URL") == null) {
            throw new ExceptionInInitializerError("DB_URL not set");
            // JVM marks this class as unusable — any future reference throws NoClassDefFoundError
        }
    }
}
// Second reference: NoClassDefFoundError (NOT ExceptionInInitializerError again)
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Static blocks run once when the class is loaded; instance blocks run before every constructor. Both execute in source order, interleaved with their respective field initialisers.
2. **Why/when:** Static blocks are for one-time setup (native libs, static lookup maps). Instance blocks share init logic across multiple constructors. Both are uncommon in modern code but appear in legacy systems.
3. **Example:** A static block loads a currency code map at class load time — `Map.of` is the modern replacement but doesn't work for complex initialisation logic (conditionals, error handling).
4. **Gotcha/tradeoff:** An exception in a static block throws `ExceptionInInitializerError` and permanently marks the class as broken — all future references throw `NoClassDefFoundError`. Fail-fast, no recovery.

**Common pitfalls**
- Assuming static blocks run at application startup — they run on first reference to the class (lazy loading).
- Using instance blocks in classes with a single constructor — pointless; put the logic in the constructor.
- Heavy work in static blocks (DB calls, HTTP) — slows class loading and can't be retried on failure.
- Circular static init between two classes — can cause subtle `null` values or `ExceptionInInitializerError`.

**Self-check question**
Class `A` has a static block, an instance block, and a constructor. Class `B extends A` has the same three. You call `new B()`. In what exact order do all six blocks execute?
