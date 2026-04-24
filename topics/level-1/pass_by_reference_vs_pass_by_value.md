### Topic: Pass-by-Reference vs Pass-by-Value — Level 1

**Why it matters (Karat angle)**
"Does Java pass by reference?" is a trick question. The correct answer is **no, Java is strictly pass-by-value** — and many candidates answer "it depends" or "by reference for objects", which is technically wrong. Nailing this signals you understand the JVM memory model.

**Core concept**

**Java is always pass-by-value.** What differs is what the value IS:

| Argument type | Value passed | Caller sees changes to... |
|---------------|--------------|---------------------------|
| Primitive (`int`, `double`, `boolean`, ...) | Copy of the value | Nothing — caller's variable untouched |
| Object reference | Copy of the reference (a pointer) | Yes, if method mutates the shared object's state |
| Object reference | Copy of the reference | No, if method reassigns the parameter to a new object |

**The key mental model:** imagine the parameter variable as a Post-it note with the memory address written on it. You hand the method a **photocopy** of that Post-it. The method can:
- Walk to the address and rearrange what's there → caller sees it (same address).
- Scribble a new address on its own copy → caller's original Post-it is unchanged.

**Why this matters:**
- Mutating a method parameter object is a hidden side effect — easy to miss in code review.
- You can't write a `swap(a, b)` that swaps two caller variables (for primitives or references) without wrapping them.
- Immutable types (`String`, `Integer`, `LocalDate`) side-step the mutation problem entirely.

**Real-world use cases**
- **Defensive copying:** if a method accepts a `List<T>` and doesn't want to be affected by later caller mutations, it copies (`new ArrayList<>(param)`).
- **Builder pattern returning `this`:** the caller's builder reference keeps working because we never reassign, we mutate.
- **Understanding mutation bugs:** a service method that takes a DTO and mutates it is a landmine — the controller's DTO is now changed too.
- **Explaining Spring bean lifecycle:** a singleton bean passed around is one shared instance; mutations to its state affect every caller.

**Working code example**
```java
// File: topics/level-1/PassByValueDemo.java

class Counter {
    int count;
    Counter(int count) { this.count = count; }
}

public class PassByValueDemo {

    // Primitive — copy of value, caller unaffected
    static void doubleIt(int x) {
        x = x * 2;                               // modifies local copy only
    }

    // Object reference — copy of the pointer; mutation visible
    static void increment(Counter c) {
        c.count++;                               // same heap object as caller's
    }

    // Object reference — reassign the parameter; caller unaffected
    static void replace(Counter c) {
        c = new Counter(999);                    // local variable now points elsewhere
        c.count = 1_000_000;                     // modifies a throwaway object
    }

    // Attempted swap — doesn't work
    static void swap(Counter a, Counter b) {
        Counter tmp = a;
        a = b;
        b = tmp;                                 // local only; callers unchanged
    }

    // Workaround — swap via array wrapper
    static void swapInArray(Counter[] arr, int i, int j) {
        Counter tmp = arr[i];
        arr[i] = arr[j];
        arr[j] = tmp;
    }

    public static void main(String[] args) {
        int n = 5;
        doubleIt(n);
        System.out.println(n);                   // 5 — primitive unchanged

        Counter counter = new Counter(10);
        increment(counter);
        System.out.println(counter.count);       // 11 — mutation visible

        replace(counter);
        System.out.println(counter.count);       // 11, NOT 1_000_000

        Counter x = new Counter(1), y = new Counter(2);
        swap(x, y);
        System.out.println(x.count + " " + y.count); // 1 2 — NOT swapped

        Counter[] arr = { x, y };
        swapInArray(arr, 0, 1);
        System.out.println(arr[0].count + " " + arr[1].count); // 2 1 — array contents swapped
    }
}
```

**Edge case: mutating a parameter object in a deep method**
```java
// Surprise side effect — caller's list grew by 1000 elements
void loadDefaults(List<String> codes) {
    for (int i = 0; i < 1000; i++) codes.add("DEFAULT-" + i);
}

// Safer signature — return a new list
List<String> withDefaults(List<String> codes) {
    List<String> result = new ArrayList<>(codes);
    for (int i = 0; i < 1000; i++) result.add("DEFAULT-" + i);
    return result;
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Java is strictly pass-by-value. For primitives, the value is copied. For objects, the reference (a pointer) is copied — both method and caller now hold references to the same heap object.
2. **Why/when:** This matters any time you mutate a parameter — mutations to the referenced object's state are visible to the caller, but reassigning the parameter to a new object is not.
3. **Example:** `increment(counter)` changes `counter.count` because both refs point at the same `Counter`. `replace(counter)` fails to change the caller's reference because reassigning inside the method only updates the local copy.
4. **Gotcha/tradeoff:** You cannot write a Java method that swaps two caller variables — `swap(a, b)` reassigning locally doesn't propagate. Wrap in an array, list, or return a new pair.

**Common pitfalls**
- Saying "Java passes objects by reference" — this is wrong; it passes references by value.
- Attempting a swap function — doesn't work without a wrapper.
- Unexpectedly mutating a caller's collection/DTO — defensive copy if you don't own the object's lifecycle.
- Thinking `final` parameters change the semantics — `final` just prevents reassigning inside the method; mutation of the referenced object is still possible.

**Self-check question**
You write `void clear(List<String> list) { list = new ArrayList<>(); }`. After `clear(myList)`, what does `myList` look like? Now write a version that actually empties the caller's list.
