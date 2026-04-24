### Topic: Strings (Formatting) — Level 3

> **Foundation:** [Level 1 — String Class](../level-1/string_class.md) · [Level 2 — String Class (encoding, ASCII)](../level-2/string_class.md)

**Why it matters (Karat angle)**
L1 covered immutability/pool, L2 covered encoding. L3 is about `String.format`, `printf`, `NumberFormat`, and `DecimalFormat` — formatting floats, currency, and dates for banking output (reports, receipts, API responses).

**Core concept**

**`String.format` / `System.out.printf` specifiers:**

| Specifier | Type | Example | Output |
|-----------|------|---------|--------|
| `%s` | String | `format("%s", "hi")` | `hi` |
| `%d` | Integer | `format("%d", 42)` | `42` |
| `%f` | Float/Double | `format("%f", 3.14)` | `3.140000` |
| `%.2f` | Float (2 decimals) | `format("%.2f", 3.14159)` | `3.14` |
| `%e` | Scientific | `format("%e", 123456.789)` | `1.234568e+05` |
| `%n` | Newline (platform-safe) | | OS-dependent newline |
| `%,d` | Thousands separator | `format("%,d", 1000000)` | `1,000,000` |
| `%10d` | Right-aligned (width 10) | `format("%10d", 42)` | `        42` |
| `%-10s` | Left-aligned (width 10) | `format("%-10s", "hi")` | `hi        ` |
| `%05d` | Zero-padded | `format("%05d", 42)` | `00042` |
| `%+.2f` | Force sign | `format("%+.2f", 3.14)` | `+3.14` |

**Working code example**

```java
// File: topics/level-3/StringFormattingDemo.java

public class StringFormattingDemo {

    public static void main(String[] args) {

        // --- float/double formatting ---
        double balance = 1234567.891;
        System.out.printf("Balance: $%,.2f%n", balance);         // Balance: $1,234,567.89
        System.out.printf("Rate: %.4f%%%n", 0.035);              // Rate: 0.0350%
        System.out.printf("Scientific: %e%n", balance);          // Scientific: 1.234568e+06

        // --- Integer formatting ---
        int txnId = 42;
        System.out.printf("TXN-%05d%n", txnId);                  // TXN-00042
        System.out.printf("Amount: %,d%n", 5_000_000);           // Amount: 5,000,000

        // --- String alignment (table formatting) ---
        System.out.printf("%-20s %10s %15s%n", "Account", "Type", "Balance");
        System.out.printf("%-20s %10s %,15.2f%n", "Alice Checking", "CHK", 12345.67);
        System.out.printf("%-20s %10s %,15.2f%n", "Bob Savings", "SAV", 98765.43);

        // --- String.format (returns String, doesn't print) ---
        String receipt = String.format("TXN %05d: $%,.2f on %tF", txnId, balance, LocalDate.now());
        // TXN 00042: $1,234,567.89 on 2026-04-23
    }
}
```

**`NumberFormat` and `DecimalFormat` — locale-aware formatting:**

```java
// File: topics/level-3/NumberFormatDemo.java

// --- Currency formatting ---
NumberFormat usd = NumberFormat.getCurrencyInstance(Locale.US);
System.out.println(usd.format(1234567.89));                      // $1,234,567.89

NumberFormat eur = NumberFormat.getCurrencyInstance(Locale.GERMANY);
System.out.println(eur.format(1234567.89));                      // 1.234.567,89 €

// --- Percentage formatting ---
NumberFormat pct = NumberFormat.getPercentInstance();
pct.setMinimumFractionDigits(2);
System.out.println(pct.format(0.0375));                          // 3.75%

// --- DecimalFormat (custom patterns) ---
DecimalFormat df = new DecimalFormat("#,##0.00");
System.out.println(df.format(1234567.891));                      // 1,234,567.89

DecimalFormat accounting = new DecimalFormat("$#,##0.00;($#,##0.00)");
System.out.println(accounting.format(1234.5));                   // $1,234.50
System.out.println(accounting.format(-1234.5));                  // ($1,234.50) — accounting style
```

**Date/Time formatting:**

```java
// File: topics/level-3/DateFormatDemo.java

LocalDateTime now = LocalDateTime.now();
DateTimeFormatter iso = DateTimeFormatter.ISO_LOCAL_DATE_TIME;
DateTimeFormatter custom = DateTimeFormatter.ofPattern("dd-MMM-yyyy HH:mm:ss");
DateTimeFormatter banking = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSXXX");

System.out.println(now.format(iso));                              // 2026-04-23T20:30:00
System.out.println(now.format(custom));                           // 23-Apr-2026 20:30:00

// Parsing
LocalDate parsed = LocalDate.parse("2026-04-23", DateTimeFormatter.ISO_LOCAL_DATE);
```

**Float vs Double precision:**

```java
// File: topics/level-3/PrecisionDemo.java

// float: ~7 significant digits
float f = 1234567.89f;
System.out.printf("float: %.2f%n", f);                           // 1234567.88 — precision lost!

// double: ~15 significant digits
double d = 1234567.89;
System.out.printf("double: %.2f%n", d);                          // 1234567.89

// NEVER use float/double for money — use BigDecimal
BigDecimal amount = new BigDecimal("1234567.89");
System.out.println(amount.setScale(2, RoundingMode.HALF_UP));    // 1234567.89 — exact
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `String.format`/`printf` use specifiers (`%d`, `%.2f`, `%,d`, `%s`) for type-safe formatting. `NumberFormat`/`DecimalFormat` provide locale-aware currency, percentage, and number formatting.
2. **Why/when:** Banking reports need exact formatting — `$1,234,567.89`, zero-padded transaction IDs (`TXN-00042`), percentage rates (`3.75%`). Locale matters — US uses `$1,234.56`, Germany uses `1.234,56 €`.
3. **Example:** `String.format("$%,.2f", 1234567.89)` → `$1,234,567.89`. `DecimalFormat("$#,##0.00;($#,##0.00)")` formats negatives in accounting style: `($1,234.50)`.
4. **Gotcha/tradeoff:** Never use `float` or `double` for money — precision loss is guaranteed. Use `BigDecimal` with explicit `RoundingMode`. `float` has ~7 digits of precision; `double` has ~15.

**Common pitfalls**
- Using `%f` without precision — defaults to 6 decimal places.
- `float` for currency — `0.1f + 0.2f != 0.3f` (floating-point representation error).
- Hardcoding `,` as thousands separator — breaks in locales that use `.` (Germany, Brazil).
- `new BigDecimal(0.1)` vs `new BigDecimal("0.1")` — the first uses the imprecise double representation.

**Self-check question**
Format `BigDecimal("1234567.895")` as US currency with 2 decimal places using `NumberFormat`. What does rounding mode `HALF_UP` produce vs `HALF_EVEN` (banker's rounding)?
