### Topic: Debugging — Level 1

**Why it matters (Karat angle)**
Interviewers ask this to gauge whether you actually debug production issues, or just println your way through tests. A senior answer includes IDE debugger specifics AND command-line tools (`jstack`, `jmap`) that you'd reach for in a prod incident.

**Core concept**

Debugging in Java spans three layers: **IDE-level** (local, interactive), **logging-level** (remote / batch analysis), and **JVM tools** (live process inspection).

**IDE debugging — core techniques (IntelliJ / Eclipse / VS Code)**

| Technique | Use case | Shortcut (IntelliJ) |
|-----------|----------|---------------------|
| Line breakpoint | Stop on a specific line | Click gutter / F8 to step |
| Conditional breakpoint | Stop only when `x > 100` | Right-click breakpoint → Condition |
| Exception breakpoint | Stop when any/specific exception is thrown | Run → View Breakpoints → + |
| Method breakpoint | Stop on entry/exit of a method | Click gutter at method signature |
| Field watchpoint | Stop on read/write of a field | Right-click breakpoint on a field |
| Step Over (F8) | Run current line, don't enter calls | |
| Step Into (F7) | Enter the next method call | |
| Step Out (Shift+F8) | Run until current method returns | |
| Resume (F9) | Continue until next breakpoint | |
| Evaluate Expression | Compute arbitrary code at breakpoint | Alt+F8 |
| Watches panel | Auto-evaluate expressions at each stop | |
| Drop Frame | Rewind to the start of the current frame | (JVM allows this!) |
| Hot Swap | Modify method body while paused, continue | Auto-saved; Ctrl+Shift+F9 |

**Logging — the async debugger**

Always prefer SLF4J + Logback/Log4j2. Use **parameterised messages** (no string concat on hot paths):
```java
log.debug("Processing {} items for user {}", items.size(), userId);  // cheap if debug off
log.error("Payment failed for txn " + txnId);                        // ❌ concat runs even if off
```

Log levels: `TRACE < DEBUG < INFO < WARN < ERROR`. Production runs at `INFO` typically; drop to `DEBUG`/`TRACE` for a specific package when troubleshooting.

**JVM tooling (in-prod or post-mortem)**

| Tool | What it shows | When to use |
|------|---------------|-------------|
| `jps` | Java process IDs on this host | "What's running?" |
| `jstack <pid>` | All thread stacks | Hung process, deadlock detection |
| `jmap -dump:live,file=heap.hprof <pid>` | Heap dump | Memory leak investigation |
| `jmap -histo <pid>` | Object count + size by class | Quick "what's eating memory" |
| `jstat -gc <pid>` | Live GC stats | GC pressure diagnosis |
| `jcmd <pid> GC.heap_info` | Modern unified CLI — everything | Java 8+ go-to |
| `jdb` | Command-line debugger | Remote / server-only environments |
| Java Flight Recorder (`-XX:StartFlightRecording`) | Full profile | Deep perf analysis |

**Remote debug (JDWP)** — attach your IDE to a running JVM:
```bash
# Start JVM with debug agent on port 5005
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -jar app.jar
```
Then in IntelliJ: Run → Attach to Process, or create a Remote JVM Debug configuration.

**Mental model:** the IDE debugger is for local known-unknowns. Logs are for unknown-unknowns that happened in prod. JVM tools are for "process is acting weird RIGHT NOW."

**Real-world use cases**
- **Reproducing a user bug:** conditional breakpoint at the entry point, condition on the user ID.
- **Data corruption:** field watchpoint on the field being corrupted — fires the moment any code writes to it.
- **Deadlock:** `jstack` looks for `Found one Java-level deadlock` and prints the waiting cycle.
- **Heap leak:** heap dump with `jmap`, open in Eclipse MAT or VisualVM, look at the dominator tree.
- **Flaky test:** start logging at TRACE for the package under test; often a timing issue is visible in the order of log lines.

**Working code example**
```java
// File: topics/level-1/DebuggingDemo.java
import java.util.*;
import org.slf4j.*;

public class DebuggingDemo {

    private static final Logger log = LoggerFactory.getLogger(DebuggingDemo.class);

    static int computeTotal(List<Integer> prices) {
        log.debug("computeTotal called with {} items", prices.size());

        int total = 0;
        for (int i = 0; i < prices.size(); i++) {
            int price = prices.get(i);
            // ▶ Set a CONDITIONAL breakpoint here: price < 0
            //   IDE only stops when this evaluates true.
            total += price;

            log.trace("after item {}: running total = {}", i, total);
        }

        log.info("Total computed: {}", total);
        return total;
    }

    public static void main(String[] args) {
        try {
            int t = computeTotal(Arrays.asList(10, 20, -5, 30));
            System.out.println("Total: " + t);
        } catch (Exception e) {
            log.error("Failed to compute total", e);    // always log exception as 2nd arg
        }
    }
}
```

**Edge case: println debugging when you must**
Sometimes you're in a container and can't attach a debugger. Use temporary guarded logs:
```java
log.debug("state={}, input={}, path={}", state, input, Thread.currentThread().getStackTrace()[2]);
```
Or use the Java 14+ `--enable-preview` stack walker: `StackWalker.getInstance().walk(...)` for selective trace.

**What to say in the interview (4-beat answer)**
1. **Definition:** Debugging in Java uses three complementary layers — IDE interactive (breakpoints, step, watch, evaluate), structured logging (SLF4J with parameterised messages), and JVM tools (`jstack`, `jmap`, `jcmd`) for live process inspection.
2. **Why/when:** IDE for local reproducible bugs. Logs for production traceability. JVM tools for "is this process deadlocked / leaking / in GC storm RIGHT NOW."
3. **Example:** To catch a hard-to-repro bug, I set a conditional breakpoint on the suspect line with the reproducing input condition — stops only when the bad case occurs, skipping the noise.
4. **Gotcha/tradeoff:** Debugger slows the app dramatically and stops GC — fine locally, never attach a stop-the-world breakpoint to prod. Use JDWP in suspend=n mode for safer live inspection.

**Common pitfalls**
- Using `System.out.println` instead of a logger — no level control, no timestamps, no correlation id.
- Logging with string concatenation — `log.debug("x=" + x)` always builds the string even if debug is off.
- Catching an exception and logging only the message — `log.error(e.getMessage())` loses the stack trace; pass `e` as the second argument.
- Leaving breakpoints in code — they're an IDE artifact, not a code construct, but checking in debug println statements is the equivalent crime.

**Self-check question**
A service is hung and you can't attach a debugger. Name three commands you'd run in order to diagnose whether it's a deadlock, GC pause, or stuck thread — and what each command shows.
