### Topic: Serialization — Level 2

> **Advanced:** [Level 3 — IO (Serializing/Deserializing objects)](../level-3/io.md)
> **Related:** [Level 2 — Cloning](./cloning.md)

**Why it matters (Karat angle)**
Serialization is how Java converts objects to bytes (for caching, messaging, or persistence) and back. Interviewers ask about `transient`, `serialVersionUID`, and the security risks — especially in banking where serialized objects cross network boundaries.

**Core concept**

**Serialization** = object → byte stream. **Deserialization** = byte stream → object.

Two approaches:

| Approach | Mechanism | Use case |
|----------|----------|----------|
| **Java native** (`Serializable`) | JVM reflection-based binary format | Legacy systems, RMI, Hibernate L2 cache |
| **JSON/XML** (Jackson, Gson) | Text-based, human-readable | REST APIs, message queues, modern apps |

**`Serializable` interface:**
```java
// File: topics/level-2/SerializationDemo.java

public class Account implements Serializable {

    private static final long serialVersionUID = 1L;     // version control for deserialization

    private Long id;
    private String holderName;
    private BigDecimal balance;

    private transient String sessionToken;               // NOT serialized
    private transient Connection dbConnection;           // NOT serialized — not serializable anyway

    private static String bankName = "Citi";             // NOT serialized — static belongs to class, not instance
}
```

**`transient` keyword:**
Marks fields that should be **excluded** from serialization.

| Use `transient` for | Reason |
|---------------------|--------|
| Passwords, tokens, secrets | Security — don't persist credentials |
| Cached/derived values | Can be recalculated after deserialization |
| Non-serializable fields | `Connection`, `Thread`, `Socket` — can't be serialized |
| Logger fields | `private transient Logger log = ...;` |

After deserialization, `transient` fields are set to their **default values** (`null`, `0`, `false`).

**Working code example**

```java
// File: topics/level-2/SerializeDemo.java
import java.io.*;

public class SerializeDemo {

    public static void main(String[] args) throws Exception {

        Account acc = new Account(1L, "Alice", new BigDecimal("5000"));
        acc.setSessionToken("secret-123");

        // ---------- Serialize ----------
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream("account.ser"))) {
            oos.writeObject(acc);
        }

        // ---------- Deserialize ----------
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream("account.ser"))) {
            Account loaded = (Account) ois.readObject();
            System.out.println(loaded.getHolderName());     // "Alice"
            System.out.println(loaded.getBalance());        // 5000
            System.out.println(loaded.getSessionToken());   // null — transient, not serialized
        }
    }
}
```

**`serialVersionUID` — version compatibility:**

```java
// If you DON'T declare serialVersionUID:
// - JVM auto-generates one based on class structure (fields, methods, etc.)
// - ANY change to the class → new UID → InvalidClassException on deserialization

// If you DO declare it:
private static final long serialVersionUID = 1L;
// - You control compatibility
// - Adding a new field with a default → old serialized objects still deserialize
// - Removing or changing a field type → you must bump the version
```

| Scenario | Declared UID | Auto-generated UID |
|----------|:------------:|:------------------:|
| Add a field | ✅ Works (new field gets default) | ❌ `InvalidClassException` |
| Remove a field | ✅ Works (old data ignored) | ❌ `InvalidClassException` |
| Change field type | ❌ `InvalidClassException` | ❌ `InvalidClassException` |
| Rename a field | Old field ignored, new field gets default | ❌ `InvalidClassException` |

**Custom serialization — `readObject` / `writeObject`:**

```java
// File: topics/level-2/CustomSerializationDemo.java

public class SecureAccount implements Serializable {

    private static final long serialVersionUID = 2L;
    private Long id;
    private String holderName;
    private transient char[] password;                   // transient — but we encrypt it manually

    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();                        // serialize non-transient fields
        oos.writeObject(encrypt(password));              // manually serialize encrypted password
    }

    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();                         // deserialize non-transient fields
        this.password = decrypt((byte[]) ois.readObject()); // manually decrypt password
    }

    private byte[] encrypt(char[] pw) { /* ... */ return new byte[0]; }
    private char[] decrypt(byte[] data) { /* ... */ return new char[0]; }
}
```

**Edge case: Externalizable — full control**
```java
// File: topics/level-2/ExternalizableDemo.java

public class CompactAccount implements Externalizable {

    private Long id;
    private String name;

    public CompactAccount() {}                           // required — Externalizable needs no-arg constructor

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeLong(id);
        out.writeUTF(name);                              // you control exact byte format
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        this.id = in.readLong();
        this.name = in.readUTF();
    }
}
// Faster than Serializable (no reflection) but more maintenance.
```

**Security risk — deserialization attacks:**
- Deserialization creates objects WITHOUT calling constructors → validation logic is bypassed.
- Malicious payloads can trigger arbitrary code execution via gadget chains.
- **Rule:** never deserialize untrusted data with Java native serialization. Use JSON (Jackson) with type validation.

**What to say in the interview (4-beat answer)**
1. **Definition:** Serialization converts an object to bytes via `Serializable` + `ObjectOutputStream`. The `transient` keyword excludes fields (passwords, connections, caches). `serialVersionUID` controls version compatibility.
2. **Why/when:** Used for Hibernate L2 cache, JMS messages (legacy), session replication. Modern systems use JSON (Jackson) instead — safer, human-readable, cross-language.
3. **Example:** `transient String sessionToken` — not serialized because it's request-specific and sensitive. After deserialization, it's `null` — the app must re-authenticate.
4. **Gotcha/tradeoff:** Without an explicit `serialVersionUID`, any class change breaks deserialization with `InvalidClassException`. Declare it always. Also: Java native deserialization is a security risk — never deserialize untrusted byte streams.

**Common pitfalls**
- Not declaring `serialVersionUID` — any refactor breaks all cached/serialized objects.
- Serializing non-serializable fields — throws `NotSerializableException` at runtime; use `transient`.
- Assuming `transient` fields have their initialized values after deserialization — they get default values (`null`, `0`).
- Storing passwords in serializable fields without `transient` — they end up in cache files, logs, or message queues.
- Using `Serializable` for REST API payloads — use Jackson/JSON instead.

**Self-check question**
You add a new field `private String email;` to a `Serializable` class that already has `serialVersionUID = 1L`. An old serialized object (without `email`) is deserialized. What value does `email` have? What if you hadn't declared the UID?
