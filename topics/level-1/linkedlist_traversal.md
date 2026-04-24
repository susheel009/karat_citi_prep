### Topic: LinkedList Traversal — Level 1

**Why it matters (Karat angle)**
Linked list traversal is a Codility staple and an interview warm-up. The real test is whether you reach for the right Java APIs (`ListIterator`, `descendingIterator`) instead of reinventing the wheel — and whether you can reverse a list in place without losing nodes.

**Core concept**

Java's `LinkedList<E>` is a **doubly-linked list** — each node holds `prev` and `next` pointers. That's why both forward AND reverse traversal are O(n), and insertion/removal at either end is O(1).

**Traversal options (all O(n))**

| Direction | Technique | Code |
|-----------|-----------|------|
| Forward | Enhanced for | `for (E e : list) ...` |
| Forward | Iterator | `Iterator<E> it = list.iterator()` |
| Forward | Stream | `list.stream().forEach(...)` |
| Forward/Backward | ListIterator | `ListIterator<E> lit = list.listIterator(list.size())` |
| Backward | descendingIterator | `Iterator<E> di = list.descendingIterator()` |
| Backward | Java 21 sequenced | `list.reversed().forEach(...)` |

**Forward vs reverse traversal (standard Java `LinkedList`)**

Forward:
```java
for (E e : list) { ... }                          // cleanest, O(n)
```

Reverse:
```java
Iterator<E> it = list.descendingIterator();       // O(n), built-in
while (it.hasNext()) { E e = it.next(); ... }

// Or (less efficient for singly-linked lists — N/A here since LinkedList is doubly linked)
ListIterator<E> lit = list.listIterator(list.size());
while (lit.hasPrevious()) { E e = lit.previous(); ... }
```

**Classic: reverse a singly-linked list in place**

If the interviewer asks about a **custom** singly-linked list (where you only have `next` pointers), this is the canonical O(n), O(1) space algorithm using three pointers:

```
prev  <- null
curr  <- head
while curr != null:
    next = curr.next          // save
    curr.next = prev          // flip pointer
    prev = curr               // advance
    curr = next
return prev                    // new head
```

**Mental model:** traversal = walking nodes. Reversing = flipping arrows as you walk.

**Real-world use cases**
- **Deque / queue:** `ArrayDeque` is usually faster; `LinkedList` is a fallback that's still a valid `Deque`.
- **Undo history:** insertion/removal at the head is O(1); forward traversal shows the history chain.
- **Interview algo drills:** reversing, detecting cycles (Floyd's tortoise-and-hare), finding middle node.
- **Pipeline stages:** chain of processors where each holds a reference to the next.

**Working code example**
```java
// File: topics/level-1/LinkedListTraversalDemo.java
import java.util.*;

public class LinkedListTraversalDemo {

    public static void main(String[] args) {

        LinkedList<String> list = new LinkedList<>(List.of("a", "b", "c", "d"));

        // ---------- Forward traversal ----------
        System.out.print("Forward: ");
        for (String s : list) System.out.print(s + " ");
        System.out.println();                       // a b c d

        // ---------- Reverse traversal via descendingIterator ----------
        System.out.print("Reverse: ");
        Iterator<String> di = list.descendingIterator();
        while (di.hasNext()) System.out.print(di.next() + " ");
        System.out.println();                       // d c b a

        // ---------- Reverse via ListIterator (doubly-linked — cheap) ----------
        System.out.print("Reverse (ListIter): ");
        ListIterator<String> lit = list.listIterator(list.size());
        while (lit.hasPrevious()) System.out.print(lit.previous() + " ");
        System.out.println();                       // d c b a

        // ---------- In-place reverse via Collections ----------
        Collections.reverse(list);
        System.out.println("After Collections.reverse: " + list);  // [d, c, b, a]
    }
}
```

**Edge case: custom singly-linked list reversal**
```java
// File: topics/level-1/SinglyReverseDemo.java
class Node {
    int val;
    Node next;
    Node(int val) { this.val = val; }
}

public class SinglyReverseDemo {

    // Iterative — three-pointer technique, O(n) time, O(1) space
    static Node reverse(Node head) {
        Node prev = null, curr = head;
        while (curr != null) {
            Node next = curr.next;   // save the rest of the list
            curr.next = prev;        // flip the pointer
            prev = curr;             // advance prev
            curr = next;             // advance curr
        }
        return prev;                  // new head
    }

    // Recursive — same result, O(n) time, O(n) stack
    static Node reverseRecursive(Node head) {
        if (head == null || head.next == null) return head;
        Node newHead = reverseRecursive(head.next);
        head.next.next = head;
        head.next = null;
        return newHead;
    }

    static void printList(Node head) {
        while (head != null) { System.out.print(head.val + " "); head = head.next; }
        System.out.println();
    }

    public static void main(String[] args) {
        Node head = new Node(1);
        head.next = new Node(2);
        head.next.next = new Node(3);
        head.next.next.next = new Node(4);

        printList(head);              // 1 2 3 4
        Node rev = reverse(head);
        printList(rev);               // 4 3 2 1
    }
}
```

**Edge case: detecting a cycle (Floyd's algorithm)**
```java
static boolean hasCycle(Node head) {
    Node slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;                // one step
        fast = fast.next.next;           // two steps
        if (slow == fast) return true;   // cycle detected
    }
    return false;                         // reached null → no cycle
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** `LinkedList<E>` in Java is a doubly-linked list, so forward and reverse traversal are both O(n); insertion/removal at either end is O(1).
2. **Why/when:** Forward traversal = for-each; reverse = `descendingIterator` or `ListIterator.hasPrevious()`. For custom singly-linked lists, the three-pointer reverse pattern (prev/curr/next) is O(n) time and O(1) space.
3. **Example:** The three-pointer algorithm walks the list once, flipping each `next` pointer to the previous node, and returns `prev` as the new head.
4. **Gotcha/tradeoff:** `LinkedList` in Java is almost never the right choice — `ArrayDeque` wins on cache locality and raw speed. Use `LinkedList` for deque semantics only when you need list + deque + queue in one object.

**Common pitfalls**
- Reversing by building a new list in reverse order — O(n) space and an extra allocation; in-place is better for custom lists.
- Losing the tail pointer during reversal — forget to save `curr.next` before overwriting.
- Using recursion on long lists — O(n) stack → `StackOverflowError`; prefer iterative.
- Confusing `LinkedList.get(i)` with O(1) — it's O(n) because it walks from the nearer end.

**Self-check question**
Given only a reference to the middle node of a singly-linked list (not the head), can you delete that node in O(1)? How, and what's the trick?
