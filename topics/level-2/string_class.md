### Topic: String Class (Encoding & ASCII) — Level 2

> **Foundation:** [Level 1 — String Class](../level-1/string_class.md)
> **Advanced:** [Level 3 — Strings (formatting for data types)](../level-3/strings.md)

**Why it matters (Karat angle)**
Encoding bugs are silent and deadly — garbled characters in API responses, broken CSV exports, security bypasses via malformed Unicode. Interviewers ask this to see if you've debugged encoding issues in production or just used `String` without thinking about bytes.

**Core concept**

L1 covered immutability, pool, StringBuilder. L2 is about what's **inside** a String and how characters map to bytes.

**Internal representation:**
- **Java 8 and earlier:** backed by `char[]` — each char is 2 bytes (UTF-16).
- **Java 9+ (Compact Strings):** backed by `byte[]` + a coder flag. Latin-1 strings (ASCII-compatible) use 1 byte per char; non-Latin strings fall back to UTF-16 (2 bytes).

| Java version | Internal array | "hello" memory | "日本語" memory |
|-------------|---------------|:-------------:|:--------------:|
| ≤ 8 | `char[]` | 10 bytes | 6 bytes |
| ≥ 9 | `byte[]` + LATIN1/UTF16 flag | 5 bytes | 6 bytes |

**ASCII — the foundation**

ASCII maps 128 characters to integers 0–127. Every Java `char` includes ASCII as a subset.

```java
// File: topics/level-2/AsciiDemo.java

char c = 'A';
int ascii = (int) c;                                    // 65
char back = (char) 65;                                  // 'A'

// Common ASCII ranges:
// '0'-'9' → 48-57
// 'A'-'Z' → 65-90
// 'a'-'z' → 97-122
// Lowercase = uppercase + 32

// Quick digit check without Character.isDigit()
boolean isDigit = c >= '0' && c <= '9';

// Case conversion via arithmetic
char lower = (char)(c + 32);                            // 'a' (only for A-Z)
char upper = (char)(c - 32);                            // 'A' (only for a-z)

// Char-to-int for digit: '7' - '0' = 7
int digit = '7' - '0';                                 // 7
```

**Encoding and charsets**

| Charset | Bytes per char | Supports | Use case |
|---------|:--------------:|----------|----------|
| ASCII | 1 | English + control chars (128 total) | Legacy systems |
| ISO-8859-1 (Latin-1) | 1 | Western European (256 chars) | Old DBs, HTTP headers |
| UTF-8 | 1–4 | All Unicode (1.1M+ chars) | Web, APIs, modern default |
| UTF-16 | 2 or 4 | All Unicode | Java internal (pre-9), Windows |

**UTF-8 is the universal default.** Every modern API, file format, and database should use it.

```java
// File: topics/level-2/EncodingDemo.java

String text = "Café";

// String → bytes (encoding)
byte[] utf8  = text.getBytes(StandardCharsets.UTF_8);    // [67, 97, 102, -61, -87] — 5 bytes
byte[] latin = text.getBytes(StandardCharsets.ISO_8859_1); // [67, 97, 102, -23] — 4 bytes

// bytes → String (decoding) — MUST use the same charset
String fromUtf8  = new String(utf8, StandardCharsets.UTF_8);    // "Café" ✅
String mismatch  = new String(utf8, StandardCharsets.ISO_8859_1); // "CafÃ©" ❌ — mojibake!

// Platform default charset (dangerous — varies by OS)
byte[] platformBytes = text.getBytes();                  // DON'T — use explicit charset
```

**The mojibake problem:**
```
Write bytes as UTF-8 → Read as ISO-8859-1 → "Café" becomes "CafÃ©"
Write bytes as ISO-8859-1 → Read as UTF-8 → "Café" becomes "Caf?" or throws
```
Fix: always specify charset explicitly. Never rely on `Charset.defaultCharset()`.

**Surrogate pairs — characters beyond U+FFFF**
```java
// File: topics/level-2/SurrogatePairDemo.java

String emoji = "😀";                                     // U+1F600
System.out.println(emoji.length());                      // 2 (two UTF-16 code units)
System.out.println(emoji.codePointCount(0, emoji.length())); // 1 (one actual character)
System.out.println(emoji.charAt(0));                     // ? (high surrogate)
System.out.println(emoji.charAt(1));                     // ? (low surrogate)

// Safe iteration over code points:
emoji.codePoints().forEach(cp -> System.out.println(Character.toString(cp)));  // 😀

// Interview trap: "hello😀".length() == 7, not 6
```

**Edge case: encoding in HTTP / REST**
```java
// File: topics/level-2/HttpEncodingDemo.java

// Spring Boot defaults to UTF-8 for JSON responses (via Jackson)
// But for plain text or XML, you may need to set it explicitly:
@GetMapping(value = "/report", produces = "text/plain;charset=UTF-8")
public String report() { return "Résumé — 日本語"; }

// URL encoding for query params:
String encoded = URLEncoder.encode("café & thé", StandardCharsets.UTF_8);
// "caf%C3%A9+%26+th%C3%A9"
String decoded = URLDecoder.decode(encoded, StandardCharsets.UTF_8);
// "café & thé"
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java Strings are backed by `byte[]` (Java 9+, compact strings) with UTF-16 encoding for non-Latin characters. ASCII is a 7-bit subset (0–127); UTF-8 is the standard encoding for APIs and files.
2. **Why/when:** Encoding mismatches cause mojibake (garbled text). Always specify `StandardCharsets.UTF_8` when converting between `String` and `byte[]`. Never rely on platform default.
3. **Example:** `"Café".getBytes(UTF_8)` produces 5 bytes (é = 2 bytes in UTF-8). Reading those bytes with ISO-8859-1 produces "CafÃ©" — a classic production bug when DB charset doesn't match app charset.
4. **Gotcha/tradeoff:** `String.length()` returns UTF-16 code units, not characters. An emoji like 😀 has `length() == 2` but `codePointCount() == 1`. Use `codePoints()` for safe iteration over actual characters.

**Common pitfalls**
- Using `getBytes()` without specifying charset — behaviour varies by OS.
- Assuming `String.length()` equals character count — fails for emoji, CJK supplementary characters.
- Storing UTF-8 data in a Latin-1 database column — silent truncation or corruption.
- URL-encoding with the wrong charset — breaks international domain names and query parameters.
- Comparing bytes from different encodings — two representations of the same string won't match.

**Self-check question**
`"hello😀".length()` returns what? `"hello😀".codePointCount(0, "hello😀".length())` returns what? Why are they different?
