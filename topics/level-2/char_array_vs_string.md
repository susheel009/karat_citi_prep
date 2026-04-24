### Topic: Char Array vs String for Passwords — Level 2

> **Foundation:** [Level 1 — String Class](../level-1/string_class.md)
> **Related:** [Level 2 — Heap and Stack](./heap_and_stack.md) · [Level 2 — Serialization](./serialization.md)

**Why it matters (Karat angle)**
This is a standard security question in banking interviews. The answer is one sentence, but the reasoning behind it tests your understanding of the String pool, GC behaviour, and memory dumps — all critical for Citi's security posture.

**Core concept**

**Rule: Use `char[]` for passwords, not `String`.**

| | `String` | `char[]` |
|--|---------|---------|
| Mutability | Immutable — can't be erased | Mutable — can zero-fill when done |
| Memory lifetime | Lives until GC (potentially minutes/hours) | Cleared immediately by your code |
| String pool | May be interned → survives even longer | Not pooled |
| Heap dump visibility | Plaintext in any memory dump | Plaintext until you clear it |
| Logging risk | Accidentally logged via `toString()` | `toString()` shows `[C@1a2b3c` (not content) |

**The problem with `String` for passwords:**
```java
String password = "secret123";
// 1. "secret123" is immutable — you CANNOT erase it
// 2. It may be in the String pool — lives until class is unloaded
// 3. Even without pool, it sits in heap until GC runs (unpredictable)
// 4. A heap dump (jmap -dump) shows it in plaintext
// 5. password = null; // reference gone, but the bytes remain in memory
```

**The `char[]` solution:**
```java
// File: topics/level-2/CharArrayPasswordDemo.java

public class AuthService {

    public boolean authenticate(char[] password) {
        try {
            boolean valid = checkPassword(password);
            return valid;
        } finally {
            // Zero-fill immediately after use
            Arrays.fill(password, '\0');
            // Now the password bytes are overwritten in memory
        }
    }

    // Java's own APIs use char[]:
    // - Console.readPassword() returns char[]
    // - JPasswordField.getPassword() returns char[]
    // - KeyStore.load(InputStream, char[]) takes char[]
}
```

**Why Java APIs use `char[]` for passwords:**
```java
// File: topics/level-2/ConsolePasswordDemo.java

Console console = System.console();
if (console != null) {
    char[] password = console.readPassword("Password: ");    // no echo, returns char[]
    try {
        authenticate(password);
    } finally {
        Arrays.fill(password, '\0');                          // wipe immediately
    }
}
// Compare: Scanner.nextLine() returns String — password lingers in memory
```

**Edge case: what about `SecretKeySpec`?**
```java
// File: topics/level-2/SecretKeyDemo.java
// Even crypto APIs take byte[] or char[], not String:
char[] passphrase = console.readPassword();
SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
KeySpec spec = new PBEKeySpec(passphrase, salt, 65536, 256);   // char[]
SecretKey key = factory.generateSecret(spec);
((PBEKeySpec) spec).clearPassword();                           // wipe the spec too
Arrays.fill(passphrase, '\0');                                  // wipe original
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Passwords should be stored in `char[]` instead of `String` because `char[]` is mutable — you can zero-fill it after use. `String` is immutable and lingers in heap memory until GC.
2. **Why/when:** In banking apps, a heap dump or memory analysis could expose passwords stored as Strings. `char[]` lets you erase the password from memory immediately, minimising the attack window.
3. **Example:** `Console.readPassword()` returns `char[]`, not `String`. After authentication, call `Arrays.fill(password, '\0')` to overwrite the memory.
4. **Gotcha/tradeoff:** Converting `char[]` to `String` for comparison defeats the purpose — the `String` re-enters the heap. Compare character-by-character or use `MessageDigest` directly with the `char[]`.

**Common pitfalls**
- Converting `char[]` to `String` for convenience (logging, comparison) — negates the security benefit.
- Forgetting to `Arrays.fill()` after use — the `char[]` is still in memory.
- Using `String.toCharArray()` thinking it helps — the original `String` still exists.
- Not clearing `PBEKeySpec` after generating a key — `clearPassword()` exists for a reason.

**Self-check question**
You receive a password as `char[]`, hash it, and store the hash. You then call `Arrays.fill(password, '\0')`. Is the password now fully gone from JVM memory? What about if GC moved the array before you cleared it?
