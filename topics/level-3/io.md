### Topic: IO (NIO, File Operations, Object Serialization) — Level 3

> **Foundation:** [Level 2 — IO (Buffered Streams)](../level-2/io.md) · [Level 2 — Serialization](../level-2/serialization.md)

**Why it matters (Karat angle)**
L2 covered buffered I/O and the decorator pattern. L3 covers NIO (non-blocking I/O), `Path`/`Files` API, file channel operations, and full object serialization workflows — the modern Java file handling that replaced legacy I/O.

**Core concept**

**NIO vs legacy I/O:**

| | Legacy (java.io) | NIO (java.nio) |
|--|-----------------|----------------|
| API | `File`, `FileReader` | `Path`, `Files`, `Channel` |
| Blocking | ✅ Always | Configurable (non-blocking) |
| Buffer | Stream-oriented | Buffer-oriented (ByteBuffer) |
| Channels | ❌ | ✅ (FileChannel, SocketChannel) |
| Watch Service | ❌ | ✅ (file system events) |
| Modern usage | Legacy code | All new code |

**`Path` and `Files` — the modern API:**

```java
// File: topics/level-3/NioFilesDemo.java

// --- Creating and resolving paths ---
Path base = Path.of("/opt/citi/data");
Path file = base.resolve("transactions.csv");            // /opt/citi/data/transactions.csv
Path relative = Path.of("../config/app.yml");
Path absolute = relative.toAbsolutePath();

// --- Reading files ---
// Read entire file as string
String content = Files.readString(Path.of("config.json"), StandardCharsets.UTF_8);

// Read all lines
List<String> lines = Files.readAllLines(Path.of("data.csv"), StandardCharsets.UTF_8);

// Stream lines lazily (for large files — doesn't load everything into memory)
try (Stream<String> lineStream = Files.lines(Path.of("big_file.csv"), StandardCharsets.UTF_8)) {
    long count = lineStream
        .filter(line -> line.contains("ERROR"))
        .count();
}

// Read all bytes
byte[] bytes = Files.readAllBytes(Path.of("image.png"));

// --- Writing files ---
Files.writeString(Path.of("output.txt"), "Hello\n", StandardCharsets.UTF_8);
Files.write(Path.of("lines.txt"), List.of("line1", "line2"), StandardCharsets.UTF_8);

// Append
Files.writeString(Path.of("log.txt"), "new entry\n",
    StandardCharsets.UTF_8, StandardOpenOption.APPEND, StandardOpenOption.CREATE);

// --- File operations ---
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target, StandardCopyOption.ATOMIC_MOVE);
Files.delete(path);                                       // throws if not exists
Files.deleteIfExists(path);                               // no-throw

// --- Directory operations ---
Files.createDirectories(Path.of("/opt/citi/data/archive"));
try (Stream<Path> entries = Files.list(Path.of("/opt/citi/data"))) {
    entries.filter(Files::isRegularFile)
           .forEach(System.out::println);
}
// Walk directory tree recursively
try (Stream<Path> tree = Files.walk(Path.of("/opt/citi"), 3)) {  // maxDepth 3
    tree.filter(p -> p.toString().endsWith(".csv"))
        .forEach(System.out::println);
}
```

**`FileChannel` — high-performance file I/O:**

```java
// File: topics/level-3/FileChannelDemo.java

// FileChannel uses ByteBuffer for direct memory transfer
try (FileChannel channel = FileChannel.open(Path.of("data.bin"), StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(8192);
    while (channel.read(buffer) != -1) {
        buffer.flip();                                    // switch from write mode to read mode
        while (buffer.hasRemaining()) {
            byte b = buffer.get();                        // process byte
        }
        buffer.clear();                                   // reset for next read
    }
}

// Memory-mapped file (for very large files — maps file directly into memory)
try (FileChannel channel = FileChannel.open(Path.of("huge.dat"), StandardOpenOption.READ)) {
    MappedByteBuffer mapped = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
    // Access file contents as if they were in memory — OS handles paging
    byte first = mapped.get(0);
}

// File-to-file transfer (zero-copy — OS-level optimisation)
try (FileChannel src = FileChannel.open(Path.of("source.dat"), StandardOpenOption.READ);
     FileChannel dst = FileChannel.open(Path.of("dest.dat"), StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
    src.transferTo(0, src.size(), dst);                   // kernel-level copy, no user-space buffer
}
```

**Object serialization to/from files:**

```java
// File: topics/level-3/ObjectSerializationDemo.java

// Serialize object to file
Account account = new Account(1L, "Alice", new BigDecimal("5000"));
try (ObjectOutputStream oos = new ObjectOutputStream(
        Files.newOutputStream(Path.of("account.ser")))) {
    oos.writeObject(account);
}

// Deserialize object from file
try (ObjectInputStream ois = new ObjectInputStream(
        Files.newInputStream(Path.of("account.ser")))) {
    Account loaded = (Account) ois.readObject();
}

// JSON serialization (modern alternative — Jackson)
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());

// Object → JSON file
mapper.writeValue(Path.of("account.json").toFile(), account);

// JSON file → Object
Account fromJson = mapper.readValue(Path.of("account.json").toFile(), Account.class);
```

**WatchService — file system events:**

```java
// File: topics/level-3/WatchServiceDemo.java

WatchService watcher = FileSystems.getDefault().newWatchService();
Path dir = Path.of("/opt/citi/incoming");
dir.register(watcher, StandardWatchEventKinds.ENTRY_CREATE);

while (true) {
    WatchKey key = watcher.take();                        // blocks until event
    for (WatchEvent<?> event : key.pollEvents()) {
        Path newFile = (Path) event.context();
        System.out.println("New file: " + newFile);
        processIncomingFile(dir.resolve(newFile));
    }
    key.reset();
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** NIO replaces legacy `File`/`FileReader` with `Path`/`Files` (modern API), `FileChannel` (buffer-oriented, non-blocking), and `WatchService` (file system events).
2. **Why/when:** `Files.readString()` / `Files.lines()` for simple text. `FileChannel` with `MappedByteBuffer` for very large files (memory-mapped, zero-copy). `WatchService` for file drop monitoring.
3. **Example:** `Files.lines(path)` streams a large CSV lazily — filter and count without loading 10GB into memory. `FileChannel.transferTo()` copies files via kernel zero-copy — no user-space buffer needed.
4. **Gotcha/tradeoff:** `Files.readAllLines()` loads everything into memory — OOM for large files. Use `Files.lines()` (lazy stream) instead. Always close streams from `Files.lines()` / `Files.list()` — use try-with-resources.

**Common pitfalls**
- `Files.readAllLines` on a 10GB file — OOM. Use `Files.lines()` stream.
- Not closing `Files.lines()` / `Files.list()` — leaks file handles.
- `ByteBuffer.flip()` forgotten after writing to buffer — reads garbage.
- Memory-mapped files not unmapped — file locked on Windows until GC collects the buffer.
- Using legacy `java.io.File` in new code — prefer `Path` and `Files`.

**Self-check question**
You need to read a 50GB log file and count lines containing "ERROR". Which API would you use — `Files.readAllLines()`, `Files.lines()`, or `FileChannel` with `ByteBuffer`? Why? What's the memory profile of each approach?
