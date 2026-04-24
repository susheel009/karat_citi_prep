### Topic: IO (Buffered Streams) — Level 2

> **Advanced:** [Level 3 — IO (NIO, file read/write, object serialization)](../level-3/io.md)
> **Related:** [Level 2 — Serialization](./serialization.md)

**Why it matters (Karat angle)**
I/O is slow. Buffering is the single biggest performance lever for file and network operations. Interviewers ask this to see if you understand why wrapping a `FileReader` in a `BufferedReader` can be 100x faster — and whether you can pick the right stream type.

**Core concept**

Java I/O has two families:

| Family | Unit | Classes | Use case |
|--------|------|---------|----------|
| **Byte streams** | `byte` (8-bit) | `InputStream` / `OutputStream` | Binary data (images, serialised objects, raw network) |
| **Character streams** | `char` (16-bit) | `Reader` / `Writer` | Text data (CSV, JSON, logs) |

**Buffered wrappers add an internal buffer** — reads/writes happen in bulk (default 8 KB), reducing OS-level system calls.

| Unbuffered | Buffered | Speedup |
|-----------|---------|---------|
| `FileInputStream` | `BufferedInputStream` | 10–100x for sequential reads |
| `FileOutputStream` | `BufferedOutputStream` | 10–100x for sequential writes |
| `FileReader` | `BufferedReader` | Adds `readLine()` + bulk reads |
| `FileWriter` | `BufferedWriter` | Adds `newLine()` + bulk writes |

**Why buffering matters — system call overhead:**
```
Without buffer: read 1 byte → 1 OS syscall → context switch → 1 byte returned
With buffer:    read 1 byte → read 8192 bytes in one syscall → serve from buffer
```

**Working code example**

```java
// File: topics/level-2/BufferedIoDemo.java
import java.io.*;
import java.nio.charset.StandardCharsets;

public class BufferedIoDemo {

    // ---------- Reading text: BufferedReader ----------
    static void readFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(
                new FileReader(path, StandardCharsets.UTF_8))) {
            String line;
            while ((line = br.readLine()) != null) {     // readLine() — only on BufferedReader
                System.out.println(line);
            }
        }
        // br auto-closed by try-with-resources
    }

    // ---------- Writing text: BufferedWriter ----------
    static void writeFile(String path, List<String> lines) throws IOException {
        try (BufferedWriter bw = new BufferedWriter(
                new FileWriter(path, StandardCharsets.UTF_8))) {
            for (String line : lines) {
                bw.write(line);
                bw.newLine();                            // platform-independent newline
            }
        }
        // Buffer flushed + file closed automatically
    }

    // ---------- Copying binary: BufferedInputStream/OutputStream ----------
    static void copyBinary(String src, String dst) throws IOException {
        try (BufferedInputStream in  = new BufferedInputStream(new FileInputStream(src));
             BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(dst))) {
            byte[] buf = new byte[8192];                 // explicit buffer for bulk copy
            int bytesRead;
            while ((bytesRead = in.read(buf)) != -1) {
                out.write(buf, 0, bytesRead);
            }
        }
    }
}
```

**The decorator pattern in action:**
```
BufferedReader
  └── InputStreamReader (byte → char conversion, charset)
        └── FileInputStream (raw bytes from disk)
```
```java
// File: topics/level-2/DecoratorStackDemo.java
// Full stack for reading UTF-8 text from a byte stream:
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("data.csv"),
            StandardCharsets.UTF_8))) {
    // br.readLine() reads bytes from FileInputStream,
    // decodes to chars via InputStreamReader,
    // buffers via BufferedReader
}
```

**Custom buffer size:**
```java
// Default buffer: 8192 bytes/chars
// For large files, increase it:
BufferedReader br = new BufferedReader(new FileReader(path), 65536);  // 64 KB buffer
```

**Edge case: flush matters for output**
```java
// File: topics/level-2/FlushDemo.java
BufferedWriter bw = new BufferedWriter(new FileWriter("log.txt"));
bw.write("important data");
// Data is in the buffer, NOT on disk yet!
// If the process crashes here, data is lost.
bw.flush();                                              // forces buffer → disk
// try-with-resources calls close(), which calls flush() automatically.
```

**PrintWriter — convenience wrapper**
```java
// File: topics/level-2/PrintWriterDemo.java
try (PrintWriter pw = new PrintWriter(
        new BufferedWriter(new FileWriter("report.txt")))) {
    pw.println("Account: " + accountId);
    pw.printf("Balance: $%,.2f%n", balance);             // formatted output
    // Auto-flushing available: new PrintWriter(..., true)
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java I/O has byte streams (`InputStream/OutputStream`) and character streams (`Reader/Writer`). Buffered wrappers (`BufferedReader`, `BufferedOutputStream`) add an internal buffer to reduce system calls.
2. **Why/when:** Unbuffered I/O makes one syscall per read/write — 100x slower for sequential operations. Always wrap file/network streams in buffered versions. `BufferedReader` also adds `readLine()` which is essential for text processing.
3. **Example:** `new BufferedReader(new InputStreamReader(new FileInputStream("data.csv"), UTF_8))` stacks three decorators — raw bytes → charset-decoded chars → buffered reads.
4. **Gotcha/tradeoff:** Buffered output doesn't write to disk until the buffer is full or `flush()` is called. A crash before flush loses data. `try-with-resources` handles this via `close()` → `flush()`, but manual resource management misses it.

**Common pitfalls**
- Not buffering file I/O — `FileReader.read()` one char at a time is catastrophically slow.
- Forgetting to flush `BufferedWriter` before process exit (without try-with-resources) — data loss.
- Using `FileReader` without explicit charset — uses platform default, breaks on cross-platform deployment.
- Double-buffering (wrapping `BufferedReader` in another `BufferedReader`) — no benefit, wastes memory.
- Mixing byte and char streams incorrectly — use `InputStreamReader` as the bridge.

**Self-check question**
You're reading a 1 GB CSV file line by line. What's the minimum decorator stack you need for correct, fast, UTF-8 text reading? What happens if you skip the `BufferedReader` wrapper?
