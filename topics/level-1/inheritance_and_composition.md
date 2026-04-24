### Topic: Inheritance and Composition — Level 1

> **Advanced coverage:** [Level 2 — Inheritance and Composition](../level-2/inheritance_and_composition.md)

**Why it matters (Karat angle)**
"Favour composition over inheritance" (Joshua Bloch, *Effective Java*) is one of the most-cited design principles. Interviewers ask this to see whether you've been burned by inheritance in production — or whether you only know textbook OOP.

**Core concept**

| | Inheritance | Composition |
|-|------------|-------------|
| Relationship | "is-a" | "has-a" |
| Mechanism | `extends` or `implements` | Field holding another object |
| Coupling | Tight — parent changes ripple to children | Loose — swap the field, caller unaffected |
| Flexibility | Fixed at compile time | Can swap at runtime |
| Java limit | Single class inheritance, multiple interface | Unlimited |
| Test isolation | Hard — must test with real parent or complex mock | Easy — inject a mock field |

**The fragile base class problem:** when you inherit, you depend on the parent's internal implementation, not just its public API. A parent method that adds a new call to another method can silently change subclass behaviour. With composition, you only depend on the other class's public interface — explicit and stable.

**Rule of thumb:**
- **Inheritance** is right when the subclass genuinely IS-A specialised version of the parent AND the hierarchy is shallow (1-2 levels).
- **Composition** is right when you want to mix capabilities, swap implementations, or avoid being locked into a class hierarchy.

**Mental model:** Inheritance is marriage (permanent, hard to undo). Composition is hiring a contractor (swap any time).

**Real-world use cases**

**Reach for inheritance when:**
- Framework requires it — `extends HttpServlet`, `extends JpaRepository<Entity, Long>`.
- Genuine specialisation — `SavingsAccount extends Account` where savings is unambiguously a kind of account.
- Abstract template method pattern — parent defines the skeleton, subclass fills in steps.

**Reach for composition when:**
- Multiple capabilities needed — a `User` that is both `Auditable` and `Cacheable`; use interfaces + field references.
- Strategy pattern — `PaymentService` holds a `PaymentGateway` field; swap `Stripe` for `PayPal` via config.
- Decorator pattern — wrap a base `Repository` with a `CachingRepository` that holds the base as a field.
- You expect the "parent" class to change often and don't want subclasses to break.

**Working code example**
```java
// File: topics/level-1/InheritanceVsComposition.java

// ---------- Inheritance approach ----------
// Fine when it's truly "is-a" and the hierarchy is shallow.
class Vehicle {
    public void start() { System.out.println("Vehicle started"); }
}

class Car extends Vehicle {                    // Car IS-A Vehicle — OK
    public void honk() { System.out.println("Beep!"); }
}

// ---------- Composition approach ----------
// Preferred when behaviour varies or must be swappable.
interface Engine {
    void start();
}

class ElectricEngine implements Engine {
    public void start() { System.out.println("Silent hum"); }
}

class PetrolEngine implements Engine {
    public void start() { System.out.println("Roar!"); }
}

// Truck HAS-A Engine — can swap engine type without modifying Truck
class Truck {
    private final Engine engine;

    public Truck(Engine engine) { this.engine = engine; }

    public void start() { engine.start(); }
}

public class InheritanceVsComposition {
    public static void main(String[] args) {
        new Car().start();                       // inherited
        new Truck(new ElectricEngine()).start(); // composed — electric
        new Truck(new PetrolEngine()).start();   // composed — petrol, same Truck class
    }
}
```

**Edge case: diamond problem avoided by Java**
```java
// File: topics/level-1/DiamondViaInterface.java

interface Swimmer { default void move() { System.out.println("Swim"); } }
interface Walker  { default void move() { System.out.println("Walk"); } }

// Multiple interface inheritance forces you to resolve the conflict explicitly
class Amphibian implements Swimmer, Walker {
    @Override
    public void move() { Swimmer.super.move(); }   // pick one — compiler-enforced
}
```
Java disallows multiple class inheritance but allows multiple interface inheritance — when two interfaces have conflicting default methods, the implementing class MUST override and pick one.

**What to say in the interview (4-beat answer)**
1. **Definition:** Inheritance is an "is-a" relationship via `extends`; composition is a "has-a" relationship by holding another object as a field.
2. **Why/when:** Prefer composition when you need runtime flexibility, want to test in isolation, or suspect the base will change. Use inheritance only when the relationship is unambiguously "is-a" and the hierarchy is shallow.
3. **Example:** `Truck` with a pluggable `Engine` field can switch from electric to petrol without modifying `Truck` itself; with inheritance that would require a new subclass per engine type.
4. **Gotcha/tradeoff:** Java allows only single class inheritance but multiple interface inheritance — deep class hierarchies eat that single "extends" slot for future flexibility.

**Common pitfalls**
- Using inheritance just to share utility code — creates a subclass that inherits irrelevant behaviour.
- Overriding a parent method and calling `super.method()` — your subclass now depends on the parent's internal call order (fragile).
- Treating `extends` as free — every superclass method becomes part of your public API.
- Using inheritance when a field (composition) would serve — ask: "Would I swap this at runtime?"

**Self-check question**
A new requirement arrives: a `FlyingCar` that both drives and flies. Why is composition the right choice over multiple inheritance (which Java doesn't allow anyway)?
