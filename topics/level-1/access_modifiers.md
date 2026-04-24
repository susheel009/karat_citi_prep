### Topic: Access Modifiers — Level 1

**Why it matters (Karat angle)**
Access modifiers are encapsulation in action. Interviewers ask this to check whether you understand the principle of least privilege — and whether you know that `protected` is wider than most developers think.

**Core concept**

Java has four access levels, from strictest to most permissive:

| Modifier | Same class | Same package | Subclass (different package) | Everywhere |
|----------|:---------:|:-----------:|:----------------------------:|:----------:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(default)* package-private | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

**Key points that trip people up:**
- **No keyword = package-private**, not public. Easy to forget when typing fast.
- `protected` gives **package access AND subclass access** — it's NOT "subclass only." A class in the same package can reach a `protected` field without inheriting.
- `private` is **class-level, not instance-level** — one `Account` can read another `Account`'s private fields directly (same class).
- From **Java 9+** there's also the **module system** (`module-info.java`) — a `public` class is only accessible across modules if the package is `exports`ed.

**Mental model:** think of it as concentric circles — class, package, hierarchy, world. Each modifier opens the next ring outward.

**Principle of least privilege:** default every field to `private`; every method to `private` or package-private; widen only when a real consumer needs it. `public` on a field or class is a commitment — once published, removal breaks every caller.

**Real-world use cases**
- **`private` fields with public getters/setters:** standard JavaBean; protects invariants.
- **Package-private utility classes:** helpers that shouldn't leak outside a package (e.g., internal DTO converters in a `service` package).
- **`protected` template methods:** used with the Template Method pattern — parent defines skeleton, subclass fills in `protected` hooks.
- **`public` interfaces + package-private implementations:** expose behaviour via interface, hide the concrete class (Spring's convention for many services).

**Working code example**
```java
// File: topics/level-1/base/Parent.java
package com.example.base;

public class Parent {
    public    int pub   = 1;   // anyone with a reference
    protected int prot  = 2;   // same package OR any subclass
              int pkg   = 3;   // same package only
    private   int priv  = 4;   // this class only

    public void show() {
        System.out.println(pub + " " + prot + " " + pkg + " " + priv);
    }
}

// Same-package access — everything except private
class SamePackageBuddy {
    void peek(Parent p) {
        System.out.println(p.pub);     // ✅ public
        System.out.println(p.prot);    // ✅ protected (same pkg)
        System.out.println(p.pkg);     // ✅ package-private
        // System.out.println(p.priv); // ❌ compile error
    }
}
```

```java
// File: topics/level-1/child/Child.java
package com.example.child;              // DIFFERENT package
import com.example.base.Parent;

class Child extends Parent {
    void peek() {
        System.out.println(pub);        // ✅ public
        System.out.println(prot);       // ✅ protected — subclass
        // System.out.println(pkg);     // ❌ different package AND not inherited as protected
        // System.out.println(priv);    // ❌ private
    }

    void peekOther(Parent p) {
        // Subtle: protected access from a subclass works on "this" and subclass-typed refs,
        // NOT on a Parent-typed reference in a different package.
        // System.out.println(p.prot);  // ❌ compile error even though Child extends Parent
    }
}
```

**Edge case: `protected` on a Parent reference in a subclass**

The last comment is the trap. From a different package, a subclass can access `protected` members **only on references of its own type or a more specific type** — not on a reference of the parent type. This rule prevents a subclass from using its access rights to poke at an unrelated `Parent` subclass's data.

**What to say in the interview (4-beat answer)**
1. **Definition:** Java has four access levels: `private` (class only), package-private (same package), `protected` (same package + subclasses anywhere), `public` (everywhere — modulo the module system in Java 9+).
2. **Why/when:** Narrower access means smaller API surface, fewer callers to break when refactoring, and stronger encapsulation. Default everything to `private` and widen only on demand.
3. **Example:** `protected prot` is visible to a subclass in a different package, but package-private `pkg` is not — unless the subclass is in the same package too.
4. **Gotcha/tradeoff:** `protected` is wider than "subclass only" — it includes every class in the same package too. That's why `protected` on a sensitive field can leak to unrelated utilities sharing the package.

**Common pitfalls**
- Forgetting no-keyword = package-private, not public.
- Thinking `protected` = "subclasses only" — it also grants same-package access.
- Making fields `public` for "convenience" — every caller becomes coupled to your representation.
- Not knowing that Java 9 modules add another layer — `public` is only cross-module if `exports`ed.

**Self-check question**
You have `protected void audit()` in class `A` in package `com.base`. Class `B` in package `com.reporting` extends `A`. Class `C` in `com.reporting` does NOT extend `A`. Can `B` call `audit()`? Can `C` call it (given an `A` reference)? Can a class `D` in `com.base` (not a subclass) call it?
