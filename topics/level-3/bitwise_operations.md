### Topic: Bitwise Operations — Level 3

> **Related:** [Level 1 — Types](../level-1/types.md) · [Level 1 — Java Fundamentals](../level-1/java_fundamentals.md)

**Why it matters (Karat angle)**
Bitwise operations appear in Codility/HackerRank challenges and interview whiteboard questions. XOR, shifts, and bitmasks are efficient solutions for problems like "find the unique number," "check if power of 2," and "swap without temp." Knowing them signals algorithmic depth.

**Core concept**

| Operator | Symbol | What it does | Example (binary) |
|----------|:------:|-------------|-------------------|
| AND | `&` | 1 if both bits are 1 | `0110 & 1010 = 0010` |
| OR | `\|` | 1 if either bit is 1 | `0110 \| 1010 = 1110` |
| XOR | `^` | 1 if bits differ | `0110 ^ 1010 = 1100` |
| NOT | `~` | Flip all bits | `~0110 = 1001` (in 2's complement) |
| Left shift | `<<` | Shift bits left, fill 0s | `0011 << 1 = 0110` (multiply by 2) |
| Right shift | `>>` | Shift bits right, preserve sign | `1100 >> 1 = 1110` (signed) |
| Unsigned right shift | `>>>` | Shift right, fill 0s | `1100 >>> 1 = 0110` |

**Key identity tricks:**

| Expression | Result | Use |
|-----------|--------|-----|
| `x ^ x` | `0` | Cancel self |
| `x ^ 0` | `x` | Identity |
| `x & 1` | Last bit (0 or 1) | Check odd/even |
| `x & (x - 1)` | Clears lowest set bit | Check power of 2 |
| `x << n` | `x * 2^n` | Fast multiply |
| `x >> n` | `x / 2^n` | Fast divide |

**Working code example**

```java
// File: topics/level-3/BitwiseDemo.java

public class BitwiseDemo {

    // --- XOR: Find the single unique number in array where every other appears twice ---
    static int findUnique(int[] nums) {
        int result = 0;
        for (int n : nums) result ^= n;
        // Pairs cancel: a ^ a = 0, so only the unique number remains
        return result;
    }

    // --- Swap two numbers without temp ---
    static void swap(int a, int b) {
        a = a ^ b;
        b = a ^ b;  // (a^b)^b = a
        a = a ^ b;  // (a^b)^a = b
        System.out.println("a=" + a + ", b=" + b);
    }

    // --- Check if power of 2 ---
    static boolean isPowerOfTwo(int n) {
        return n > 0 && (n & (n - 1)) == 0;
        // Power of 2 has exactly one bit set: 1000 & 0111 = 0
    }

    // --- Count set bits (Kernighan's algorithm) ---
    static int countBits(int n) {
        int count = 0;
        while (n != 0) {
            n = n & (n - 1);  // clear lowest set bit
            count++;
        }
        return count;
    }

    // --- Check if bit at position k is set ---
    static boolean isBitSet(int n, int k) {
        return (n & (1 << k)) != 0;
    }

    // --- Set bit at position k ---
    static int setBit(int n, int k) {
        return n | (1 << k);
    }

    // --- Clear bit at position k ---
    static int clearBit(int n, int k) {
        return n & ~(1 << k);
    }

    // --- Toggle bit at position k ---
    static int toggleBit(int n, int k) {
        return n ^ (1 << k);
    }

    public static void main(String[] args) {
        System.out.println(findUnique(new int[]{2, 3, 5, 3, 2}));  // 5
        System.out.println(isPowerOfTwo(16));                       // true
        System.out.println(isPowerOfTwo(18));                       // false
        System.out.println(countBits(0b1011));                      // 3
        System.out.println(isBitSet(0b1010, 1));                    // true (bit 1 is set)
        swap(7, 11);                                                // a=11, b=7
    }
}
```

**Bitmask for permissions (real-world banking use case):**

```java
// File: topics/level-3/BitmaskPermissions.java

public class Permissions {
    static final int READ    = 1;       // 0001
    static final int WRITE   = 1 << 1;  // 0010
    static final int DELETE  = 1 << 2;  // 0100
    static final int ADMIN   = 1 << 3;  // 1000

    static boolean hasPermission(int userPerms, int required) {
        return (userPerms & required) == required;
    }

    static int grant(int userPerms, int perm) { return userPerms | perm; }
    static int revoke(int userPerms, int perm) { return userPerms & ~perm; }

    public static void main(String[] args) {
        int perms = READ | WRITE;                         // 0011
        System.out.println(hasPermission(perms, READ));   // true
        System.out.println(hasPermission(perms, DELETE)); // false
        perms = grant(perms, DELETE);                     // 0111
        perms = revoke(perms, WRITE);                     // 0101
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Bitwise operations work on individual bits — AND (`&`), OR (`|`), XOR (`^`), NOT (`~`), and shifts (`<<`, `>>`, `>>>`). They're O(1) and hardware-level fast.
2. **Why/when:** XOR finds unique elements, `n & (n-1)` checks power of 2, bitmasks encode permission flags compactly. Common in Codility challenges and system-level code.
3. **Example:** `findUnique([2,3,5,3,2])` → XOR all elements → pairs cancel → `5` remains. O(n) time, O(1) space.
4. **Gotcha/tradeoff:** `>>` preserves the sign bit (arithmetic shift); `>>>` fills with zeros (logical shift). For negative numbers, they produce different results. XOR swap is clever but unreadable — use a temp variable in production.

**Common pitfalls**
- Confusing `>>` (sign-preserving) with `>>>` (zero-fill) for negative numbers.
- Using `==` instead of `!=0` when checking bit flags: `(perms & READ) == true` doesn't compile.
- Forgetting that `~` on an `int` flips all 32 bits — `~0 == -1`, not 1.
- Shift overflow: `1 << 32` wraps to `1` (shift by 32 is modulo 32 for int).

**Self-check question**
Given an array where every number appears three times except one which appears once, can you find it using bitwise operations? (Hint: XOR won't work — you need to count bits modulo 3.)
