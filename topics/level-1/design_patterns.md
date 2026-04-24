### Topic: Design Patterns — Level 1

> **Advanced coverage:** [Level 2 — Design Patterns (adds Façade, Proxy)](../level-2/design_patterns.md) · [Level 3 — Design Principles (SOLID, KISS, YAGNI)](../level-3/design_principles.md)

**Why it matters (Karat angle)**
Design patterns are the shared vocabulary of OOP. Interviewers expect you to recognise them in code (Spring uses a dozen), name them when describing a solution, and know which problem each one solves — not recite Gang-of-Four definitions.

**Core concept**

Patterns fall into three families (GoF classification):

| Family | Intent | Patterns in the cheatsheet |
|--------|--------|----------------------------|
| **Creational** | How objects are created | Singleton, Factory |
| **Structural** | How objects are composed | Adapter, Decorator, Bridge |
| **Behavioural** | How objects interact | Observer, Visitor, Template |

### The eight Level-1 patterns

**1. Singleton** — one instance per JVM.
```java
public class Config {
    private static final Config INSTANCE = new Config();       // eager + final = thread-safe
    private Config() {}
    public static Config getInstance() { return INSTANCE; }
}
```
*Modern alternative:* an enum is the simplest thread-safe Singleton:
```java
public enum Config {
    INSTANCE;
    public String getUrl() { return "..."; }
}
```
*When:* config, logging, caches, connection pools.
*Pitfalls:* hides dependencies; hard to test — prefer dependency injection (Spring bean with default singleton scope).

**2. Factory** — creation logic behind a method; hides `new`.
```java
public interface Shape { double area(); }
public class Circle  implements Shape { public double area() { return 3.14; } }
public class Square  implements Shape { public double area() { return 4.0;  } }

public class ShapeFactory {
    public static Shape create(String type) {
        return switch (type) {
            case "circle" -> new Circle();
            case "square" -> new Square();
            default -> throw new IllegalArgumentException(type);
        };
    }
}
```
*When:* you want to decouple callers from concrete classes.

**3. Adapter** — make an incompatible interface compatible.
```java
// Existing (third-party) class
class LegacyLogger {
    public void logRaw(String msg) { System.out.println(msg); }
}

// Target interface you want
interface Logger { void info(String msg); }

// Adapter
class LegacyAdapter implements Logger {
    private final LegacyLogger legacy;
    public LegacyAdapter(LegacyLogger legacy) { this.legacy = legacy; }
    public void info(String msg) { legacy.logRaw("[INFO] " + msg); }
}
```
*When:* integrating legacy or third-party APIs without changing them.

**4. Observer** — publisher-subscriber.
```java
interface Listener { void onEvent(String e); }

class EventBus {
    private final List<Listener> listeners = new CopyOnWriteArrayList<>();
    public void register(Listener l) { listeners.add(l); }
    public void publish(String event) { listeners.forEach(l -> l.onEvent(event)); }
}
```
*When:* UI events, Spring `ApplicationEvent`, Kafka consumers, JMX notifications.

**5. Decorator** — wrap an object to add behaviour at runtime.
```java
interface DataSource { String read(); }
class FileDataSource implements DataSource { public String read() { return "raw"; } }

class EncryptedDataSource implements DataSource {
    private final DataSource wrapped;
    public EncryptedDataSource(DataSource wrapped) { this.wrapped = wrapped; }
    public String read() { return decrypt(wrapped.read()); }
    private String decrypt(String s) { return "[decrypted] " + s; }
}

// Usage — stack decorators
DataSource ds = new EncryptedDataSource(new FileDataSource());
```
*When:* Java I/O (`BufferedReader` wraps `FileReader` wraps `InputStreamReader`), Spring `HandlerInterceptor` chain.

**6. Bridge** — separate abstraction from implementation so they can vary independently.
```java
// Abstraction
abstract class Notification {
    protected final Channel channel;
    Notification(Channel channel) { this.channel = channel; }
    abstract void send(String msg);
}

// Implementation
interface Channel { void deliver(String msg); }
class Email  implements Channel { public void deliver(String m) { /* SMTP */ } }
class SMS    implements Channel { public void deliver(String m) { /* Twilio */ } }

// Concrete abstractions
class Alert  extends Notification { Alert(Channel c){super(c);}  void send(String m){channel.deliver("🚨 "+m);} }
class Report extends Notification { Report(Channel c){super(c);} void send(String m){channel.deliver("📊 "+m);} }

// Mix and match: Alert+Email, Alert+SMS, Report+Email — no combinatorial subclass explosion
```
*When:* you'd otherwise need M × N subclasses.

**7. Visitor** — add operations to a class hierarchy without modifying the classes.
```java
interface Shape { double accept(ShapeVisitor v); }
interface ShapeVisitor { double visit(Circle c); double visit(Square s); }

class Circle implements Shape {
    double radius;
    public double accept(ShapeVisitor v) { return v.visit(this); }
}
class Square implements Shape {
    double side;
    public double accept(ShapeVisitor v) { return v.visit(this); }
}

class AreaVisitor implements ShapeVisitor {
    public double visit(Circle c) { return Math.PI * c.radius * c.radius; }
    public double visit(Square s) { return s.side * s.side; }
}
```
*When:* stable hierarchy + evolving operations (compilers' AST processing).

**8. Template Method** — parent defines the skeleton; subclass fills in the steps.
```java
abstract class ReportGenerator {
    public final String generate() {       // final — subclass can't reorder
        String data = fetch();
        String rendered = render(data);
        return store(rendered);
    }
    protected abstract String fetch();
    protected abstract String render(String data);
    protected abstract String store(String rendered);
}

class PdfReport extends ReportGenerator {
    protected String fetch()  { return "rows"; }
    protected String render(String d) { return "[PDF]" + d; }
    protected String store(String r)  { return "saved:" + r; }
}
```
*When:* the algorithm structure is fixed but individual steps vary (JDBC `JdbcTemplate`, Spring `TransactionTemplate`).

**Pattern quick-reference**

| Pattern | Problem it solves |
|---------|-------------------|
| Singleton | One global instance |
| Factory | Hide `new`, decouple from concrete class |
| Adapter | Make incompatible interface fit |
| Observer | One-to-many event notification |
| Decorator | Stack features at runtime |
| Bridge | Avoid M×N subclass explosion |
| Visitor | Add operations without modifying classes |
| Template | Reuse algorithm skeleton; vary steps |

**Real-world use cases (you'll see these in Spring)**
- **Singleton:** every `@Service`/`@Component` bean by default.
- **Factory:** `BeanFactory`, `ApplicationContext.getBean`.
- **Adapter:** `HandlerAdapter` in Spring MVC.
- **Observer:** `ApplicationEventPublisher` / `@EventListener`.
- **Decorator:** `HandlerInterceptor` chain, servlet filters.
- **Template:** `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`.
- **Proxy:** (L2) Spring AOP, `@Transactional` wraps beans in a proxy.

**What to say in the interview (4-beat answer)**
1. **Definition:** Design patterns are reusable solutions to common design problems, organised into Creational (how to create), Structural (how to compose), and Behavioural (how to interact).
2. **Why/when:** They give teams a shared vocabulary ("let's make that a Strategy"), help spot the right Java feature (interface + field = composition = Strategy or Bridge), and surface in frameworks — Spring alone uses Singleton, Factory, Template, Proxy, Observer, Decorator.
3. **Example:** `JdbcTemplate` is a classic Template Method — it handles connection management, transaction boundaries, and exception translation, letting you supply only the SQL and a `RowMapper`.
4. **Gotcha/tradeoff:** Patterns are tools, not goals. Over-patterning adds classes and ceremony — reach for them when a specific problem appears, not preemptively.

**Common pitfalls**
- Making everything a Singleton "for convenience" — global state, testability nightmare.
- Confusing Adapter (make incompatible fit) with Facade (simplify complex) — different intents.
- Decorators without composition via the common interface — ends up as tight-coupled extends.
- Rolling your own Singleton with double-checked locking — enum or eager static field is safer.

**Self-check question**
Pick any three of the eight patterns. For each, name a real Spring/JDK class that implements it and describe the problem it solves there.
