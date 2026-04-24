### Topic: Iterators — Level 1

**Why it matters (Karat angle)**
Fail-fast vs fail-safe is a classic collections question. It tests whether you understand what actually happens when code modifies a list mid-iteration — a bug that surfaces as `ConcurrentModificationException` in production and baffles juniors.

**Core concept**

Java iterators fall into two behavioural categories:

| Category | Behaviour on concurrent modification | Examples | Cost |
|---------|---------------------------------------|----------|------|
| **Fail-fast** | Throws `ConcurrentModificationException` (CME) on next `next()` | `ArrayList`, `HashMap`, `LinkedList`, `HashSet` (standard) | None at runtime |
| **Fail-safe** | Iterates over a snapshot or concurrent structure — no CME, but may miss recent changes | `CopyOnWriteArrayList`, `ConcurrentHashMap`, `CopyOnWriteArraySet` | Memory (snapshot) or write cost |

**How fail-fast works (the modCount trick):** every structural modification (`add`, `remove` — not `set`) increments a `modCount` counter on the collection. The iterator captures `expectedModCount` at creation. On each `next()`, it checks `modCount == expectedModCount`; mismatch → throw CME. This is **best-effort** — not synchronised, so concurrent threads may miss the check and silently corrupt iteration.

**How fail-safe works:**
- `CopyOnWriteArrayList` — every mutation copies the entire underlying array. Iterators see the old snapshot; they never throw, but they also don't see concurrent writes.
- `ConcurrentHashMap` — segmented/locking internals; iterators are weakly consistent (reflect some but not necessarily all changes since creation).

**Mental model:** fail-fast says "don't modify me while you're iterating." Fail-safe says "I'll let you, but your iterator is working on yesterday's snapshot."

**Real-world use cases**

**Fail-fast (most of the time):**
- Default for ordinary ArrayList/HashMap iteration in a single thread.
- Protects against a common bug: modifying a list inside a for-each loop.

**Fail-safe:**
- `CopyOnWriteArrayList` for listener registries (many reads, rare writes) — e.g., Spring event listeners, observer patterns.
- `ConcurrentHashMap` for shared caches — millions of reads, occasional writes, no desire to throw on race conditions.

**Correct ways to modify during iteration**

```java
// 1. Iterator.remove() — safe even on fail-fast collections
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (shouldRemove(it.next())) it.remove();   // does NOT trigger CME
}

// 2. removeIf() — Java 8+ one-liner
list.removeIf(s -> shouldRemove(s));

// 3. Build a new list (no mutation of original)
List<String> kept = list.stream()
                        .filter(s -> !shouldRemove(s))
                        .collect(toList());
```

**Working code example**
```java
// File: topics/level-1/IteratorDemo.java
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

public class IteratorDemo {

    public static void main(String[] args) {

        // ---------- Fail-fast demo ----------
        List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
        try {
            for (String s : list) {
                if (s.equals("b")) list.remove(s);    // modifies during iteration
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("CME caught (fail-fast): " + e.getClass().getSimpleName());
        }

        // ---------- Correct: Iterator.remove ----------
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            if (it.next().equals("a")) it.remove();   // safe
        }
        System.out.println("After Iterator.remove: " + list);   // [b, c]

        // ---------- Correct: removeIf ----------
        List<String> list2 = new ArrayList<>(Arrays.asList("a", "b", "c"));
        list2.removeIf(s -> s.equals("b"));
        System.out.println("After removeIf: " + list2);         // [a, c]

        // ---------- Fail-safe demo ----------
        List<String> safe = new CopyOnWriteArrayList<>(Arrays.asList("x", "y", "z"));
        for (String s : safe) {
            if (s.equals("y")) safe.remove(s);        // NO exception
        }
        System.out.println("Fail-safe result: " + safe);        // [x, z]
        // Note: the iterator was iterating the OLD snapshot — it still saw "y"
    }
}
```

**Edge case: modification on a sub-list view**
```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4, 5));
List<Integer> sub  = list.subList(1, 4);         // view, not copy
list.add(6);                                     // mutates backing list
// sub.get(0);                                    // throws CME — view sees modCount mismatch
```
`subList` returns a **view** — any structural change to the original after the view is created invalidates the view.

**What to say in the interview (4-beat answer)**
1. **Definition:** Fail-fast iterators throw `ConcurrentModificationException` on structural modification during iteration; fail-safe iterators operate on a snapshot or concurrent structure and never throw.
2. **Why/when:** Fail-fast catches bugs early in single-threaded code. Fail-safe (`CopyOnWriteArrayList`, `ConcurrentHashMap`) is for read-heavy concurrent scenarios where occasional staleness is acceptable.
3. **Example:** A for-each loop that calls `list.remove(x)` triggers CME because `list.remove` increments `modCount`, which the iterator detects. Using `Iterator.remove()` bypasses this because the iterator updates `expectedModCount` in sync.
4. **Gotcha/tradeoff:** `CopyOnWriteArrayList` is only cheap for reads — every write copies the entire array, making it O(n) per write. Suitable for listener lists, not for high-write workloads.

**Common pitfalls**
- Calling `list.remove(element)` inside a for-each — always use `Iterator.remove()` or `removeIf()`.
- Assuming fail-fast is a thread-safety guarantee — it's best-effort; concurrent threads can silently corrupt.
- Treating `ConcurrentHashMap` iterators as giving a consistent snapshot — they're weakly consistent.
- Using `CopyOnWriteArrayList` for write-heavy workloads — it gets slow quadratically.

**Self-check question**
You need to remove all negative numbers from a large `ArrayList<Integer>` in one pass, in a single thread. Name two safe approaches and explain why each avoids CME.
