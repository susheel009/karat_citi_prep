# Self-Audit — Level 1 + Level 2 (G/Y/R)

**Purpose:** Rate comfort on each topic before study starts. Result decides study order.

**Scale:**
- **G** (Green) — can explain the concept AND write clean code cold in a Karat interview.
- **Y** (Yellow) — know the idea, rusty on internals, gotchas, or syntax.
- **R** (Red) — need full coverage from scratch or last touched years ago.

**How to run it:**
1. 25 min, pen-and-paper first. First instinct wins — don't deliberate.
2. For each topic: close your eyes, imagine the Karat interviewer asking "Tell me about X." Could you answer the 4-beat template (definition → why → example → gotcha)? If yes → G. If partial → Y. If no → R.
3. Transfer ratings into the tables below by replacing the blank cell with G, Y, or R.
4. Jot surprises in the Notes section — forgotten internals, unexpected Rs, etc.

**Study priority after audit:**
1. R in L2 (primary focus, highest interview weight)
2. Y in L2
3. R in L1 (warmup gaps — fill fast, don't linger)
4. Y in L1 (quick refreshers only)
5. G refreshers skipped unless time remains

---

## Level 1 — Standard (warmup, 2-3 yrs experience)

| # | Topic | Scope hint | Rating |
|---|-------|------------|--------|
| 1 | Object-Oriented Programming (OOP) | Concepts + practical usage | |
| 2 | Inheritance and Composition | Explanation, differences | |
| 3 | Java Fundamentals | Versions 8 and 11, Collections, Exception Handling | |
| 4 | String class | String vs StringBuffer vs StringBuilder; pool vs heap | |
| 5 | Iterators | Fail-fast vs fail-safe | |
| 6 | Access Modifiers | public/protected/default/private | |
| 7 | Types | Classes, Interfaces, Enums, Abstract, Annotations, Anonymous | |
| 8 | Pass-by-Reference vs Pass-by-Value | Java's actual behavior | |
| 9 | Java Versions | New features in 8, 11, 17, 21 | |
| 10 | JVM, JRE, JDK | Differences, what each contains | |
| 11 | Lambda Expressions | Usage, examples | |
| 12 | Collections | ArrayList, Map, Vector, Set; HashSet vs HashMap | |
| 13 | Exception Handling | Hierarchy, custom, chained catch | |
| 14 | Debugging | Basic techniques | |
| 15 | HashMap | Storage and retrieval mechanism | |
| 16 | Thread | ThreadPool, Runnable, wait/notify/sleep/start/run/join | |
| 17 | Immutable Classes | Creation + usage | |
| 18 | LinkedList Traversal | Forward and reverse | |
| 19 | Autoboxing / Auto-unboxing | Explanation, usage | |
| 20 | hashCode ↔ equals | Contract between the two | |
| 21 | Design Patterns | Singleton, Factory, Adapter, Observer, Decorator, Bridge, Visitor, Template | |
| 22 | Single-Threaded Application | Implementation | |
| 23 | Spring Framework Fundamentals | MVC, JDBC, JPA/Hibernate basics | |
| 24 | Spring Boot Fundamentals | Annotations, properties, environment, AutoConfiguration | |
| 25 | Spring Framework vs Spring Boot | Differences, why Boot, why containerize | |
| 26 | Beans | Lifecycle, XML vs Java config, scope types | |
| 27 | REST APIs | Writing, validation, security basics, HTTP methods, status codes | |
| 28 | Spring Boot Annotations | @SpringBootApplication, @ComponentScan, @Controller, @Service, @Repository | |
| 29 | Unit Testing Fundamentals | Spring Boot test setup, assertions | |
| 30 | Profiles | Active profiles in Spring Boot | |
| 31 | SQL Integration | Queries / updates / procedures from Java | |
| 32 | Code Repository | Bitbucket, Github | |
| 33 | Agile Methodology | Ceremonies, roles, artifacts | |

---

## Level 2 — Senior (PRIMARY FOCUS, 3-5 yrs experience)

| # | Topic | Scope hint | Rating |
|---|-------|------------|--------|
| 1 | Advanced Java | Java 17/21, enhanced switch, try-with-resources | |
| 2 | Collections | Internal workings, immutable collections | |
| 3 | Streams | map, filter, collect, flatten, reduce | |
| 4 | String class (advanced) | Encoding, ASCII codes | |
| 5 | Block types | Static block, instance block | |
| 6 | Autoboxing (advanced) | Compiler errors/warnings, data truncation | |
| 7 | IO | Buffered IO streams | |
| 8 | Lambdas (advanced) | Final / effectively final, Atomic types | |
| 9 | Serialization | transient modifier, mechanics | |
| 10 | Cloning | Deep vs shallow | |
| 11 | Heap and Stack | Differences, what lives where | |
| 12 | Char Array vs String | Password-storage preference | |
| 13 | Functional Interface + Lambdas | Differences, usage | |
| 14 | Custom Exceptions | Creation and usage | |
| 15 | Inheritance and Composition (advanced) | When to prefer which | |
| 16 | Multithreading in Java | Thread, producer-consumer, monitor locking | |
| 17 | Concurrent Operations | Concurrent read/write on collections | |
| 18 | Binary Tree | Implementation in Java | |
| 19 | Custom Connection Pool | Design and creation | |
| 20 | Design Patterns (full set) | ... + Façade, Proxy | |
| 21 | Stored Procedures | Calling from Java | |
| 22 | Pagination | SQL query implementation | |
| 23 | DB Indexes | Types and usage | |
| 24 | Memory Leaks | OOM, identification | |
| 25 | Spring | IoC Container | |
| 26 | Servlets | Dispatcher servlet | |
| 27 | Circular Dependencies | Handling in Spring | |
| 28 | Beans (advanced) | Prototype scope, ApplicationContext vs BeanFactory, post-processors | |
| 29 | Microservices | Fundamentals, orchestration, pros/cons, implementation parameters | |
| 30 | Spring Boot (advanced) | JPA, pagination, transactions, optimistic locking, caching, actuator | |
| 31 | REST APIs (secure) | OAuth | |
| 32 | ORM Technologies | Custom DataSource, TransactionManager, propagation, isolation | |
| 33 | Locking | Optimistic JPA, distributed Redis | |
| 34 | Caching | Caching, eviction | |
| 35 | Monitoring and Alerting | Actuator, custom health endpoints | |
| 36 | Security | JWT, SSL, Spring Security config, matchers, roles | |
| 37 | SOAP Calls | Implementation, SSL cert handling | |
| 38 | Data Sources | Config, @Transactional, read-only | |
| 39 | Configuration Properties Service | Spring Boot impl | |
| 40 | Circuit Breaker | Pattern explanation | |
| 41 | Volatile Keyword | Usage in Java | |
| 42 | Cloud Deployment | Deploying Spring Boot | |
| 43 | Parallel Service Calls | Spring Boot impl | |
| 44 | CI/CD | Github, Jenkins/GH Actions, Openshift, Docker, Kubernetes | |

---

## Notes / gotchas flagged during audit

_Jot anything that surprised you — forgotten internals, unexpected Rs, topics you thought you knew but couldn't explain cold._

-
-
-
-

---

## Agreed study order (fill in after audit)

1.
2.
3.
4.
5.
