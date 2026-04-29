# Citi/Karat — 4-Day Intensive Prep Plan

**The single source of truth for the next 96 hours.** Everything you need is here. Other docs (`java-data-structures-from-the-ground-up.md`, `leetcode-easy-by-data-structure.md`) are reference material — this doc is your operating manual.

---

## Table of Contents

1. [The Strategic Frame](#1-the-strategic-frame)
2. [Karat Interview Structure (Know This Cold)](#2-karat-interview-structure-know-this-cold)
3. [The Four Non-Negotiable Rules](#3-the-four-non-negotiable-rules)
4. [Day-by-Day Plan](#4-day-by-day-plan)
    - [Day 1: Stack, HashMap, Array Foundations](#day-1-stack-hashmap-array-foundations)
    - [Day 2: Trees, Linked Lists, Two Pointers/Sliding Window](#day-2-trees-linked-lists-two-pointerssliding-window)
    - [Day 3: Heap, Graph, Small-Class Design](#day-3-heap-graph-small-class-design)
    - [Day 4: Mock + Polish + Light Review](#day-4-mock--polish--light-review)
5. [The UMPIRE Process (Apply to Every Problem)](#5-the-umpire-process-apply-to-every-problem)
6. [The Karat Talk Track (Memorize This)](#6-the-karat-talk-track-memorize-this)
7. [Pattern Recognition Cheat Sheet](#7-pattern-recognition-cheat-sheet)
8. [Java Fundamentals — One-Page Cheat Sheet](#8-java-fundamentals--one-page-cheat-sheet)
9. [Java Syntax You Must Have Cold](#9-java-syntax-you-must-have-cold)
10. [Small-Class Design — The 3 to Drill](#10-small-class-design--the-3-to-drill)
11. [What I'm Cutting and Why](#11-what-im-cutting-and-why)
12. [Day-of-Interview Playbook](#12-day-of-interview-playbook)
13. [Failure Mode Checklist](#13-failure-mode-checklist)

---

## 1. The Strategic Frame

You have 4 days. You're an experienced engineer with rusty pattern recognition (Copilot atrophy) and one known gap (Stack — the parens problem). Karat will give you 1-2 coding problems plus Java fundamentals plus possibly a small-class design.

**The strategy is built on three claims:**

1. **Pattern recognition matters more than algorithm depth.** At Karat-Easy/Medium level, every problem reduces to ~10 patterns. Recognizing the pattern in the first 60 seconds is 70% of the battle.
2. **Communication is half the grade.** Karat literally scores it. A B-tier solution explained brilliantly beats an A-tier solution mumbled through.
3. **Your gap areas (Stack, recognition reflex) need disproportionate time.** A 4-day plan that gives equal time to everything is a 4-day plan that fixes nothing.

**What you'll be able to do after 4 days:** Reliably solve Easy problems and ~70% of Mediums in the canonical patterns. Articulate trade-offs. Speak Java fundamentals fluently for ~15 minutes. Talk through your thought process the way Karat scores highest.

**What you won't be able to do:** Hard DP, advanced graphs (Dijkstra, MST), tricky bit manipulation. That's fine — those are rare at Karat.

---

## 2. Karat Interview Structure (Know This Cold)

Karat is a third-party interviewing platform. The interviewer is a contractor running a fixed Citi-approved script. They will not deviate. They will not tailor to your resume.

**Total duration: 60-90 minutes.**

| Section | Duration | What happens |
|---|---|---|
| Intro | ~3-5 min | Brief — name, background, one sentence on why interested. Do not over-talk. |
| Java/CS fundamentals | ~15-20 min | Rapid-fire questions. Collections internals, concurrency, OOP, exceptions, JVM basics. They want crisp 1-3 sentence answers. |
| Coding problem 1 | ~25-35 min | Medium-flavored DSA problem. They share a problem statement; you code in their browser editor. |
| Coding problem 2 *(if time)* | ~15-25 min | Either a follow-up extension to problem 1, or a smaller second problem. Often a small-class design. |
| Wrap | ~2 min | "Any questions for me?" — have one ready. |

**What's recorded:** Everything. Your screen, your video, your audio. A separate Citi engineer reviews the recording later. **The interviewer does not make the hiring decision.** This is critical: you're performing for the recording, not for the person on the call.

**The Karat editor:** Plain-text browser editor. No autocomplete. No syntax highlighting beyond basic. No Copilot. No IntelliJ. **Practice in LeetCode's editor — it's the closest match.**

**Languages:** Java is supported. Stick to Java unless the question dictates otherwise.

**What they score (Citi's published rubric format):**
- Problem comprehension (did you clarify before coding?)
- Communication (did you talk through your thinking?)
- Correctness (does the code work?)
- Code quality (naming, structure, edge cases)
- Complexity analysis (can you state and justify big-O?)
- Java fluency (separate fundamentals section)

**Highest-leverage insight:** You can lose points on correctness and still pass if you nail comprehension, communication, and complexity. You cannot pass with great code and silent flailing.

---

## 3. The Four Non-Negotiable Rules

These are not suggestions. Violating these is what causes the candidate-with-better-skills to lose to the candidate-with-discipline.

### Rule 1: No Copilot. No AI. No "View Solution."

For the next 4 days, you are training under interview conditions. If you let Copilot help during prep, you've defeated the prep. Disable Copilot in IntelliJ. Don't ask ChatGPT/Claude to solve the problem. Don't click "view solution" on LeetCode.

**The only allowed lookups:**
- Java method signatures and syntax (Baeldung, official docs)
- LeetCode's "Hints" tab (1-3 hints, not the full solution) — only after 25+ minutes stuck
- *This document*

**Why this matters:** You discovered (correctly) that Copilot did your pattern recognition for years. The reflex you need to rebuild only rebuilds when you struggle without help. Comfort = no learning. Friction = the reflex coming back.

### Rule 2: Talk Out Loud. Always.

Every problem, even when alone. Narrate your thinking. State your plan. Say "I'm choosing a HashMap because I need O(1) lookup." Even when no one's listening. Especially when no one's listening — that's when the habit forms.

If you can record yourself (phone, OBS, Zoom-to-self), do it. Listen to one recording per day. The first time you hear yourself, you'll cringe. By Day 4 you'll be fluent.

### Rule 3: UMPIRE Every Problem

**U**nderstand → **M**atch → **P**lan → **I**mplement → **R**eview → **E**valuate

Spend 5 minutes on U-M-P **before typing a single line of code**. Skipping this is the #1 reason senior engineers fail Karat. Full template in Section 5.

### Rule 4: After Every Problem, Write the Pattern Entry

One line. *"When I see [signal], reach for [structure] because [reason]."*

By Day 4 you'll have 30-40 entries. This is your highest-value artifact. Review it the morning of the interview.

---

## 4. Day-by-Day Plan

**~5-6 hours/day of focused work.** Not 12 hours of unfocused grinding. Sleep is when consolidation happens. Take a 60-minute break in the middle of each day.

### Day 1: Stack, HashMap, Array Foundations

**Theme: rebuild the recognition reflex on the highest-frequency patterns. Stack first because that's your known gap.**

**Morning Block (2.5 hrs) — STACK (priority area)**

Your gap is here. Spend disproportionate time.

1. Read Stack section in `java-data-structures-from-the-ground-up.md` (10 min)
2. Drill Java syntax cold (15 min): Without looking, write code to:
    - Create a stack using `ArrayDeque`
    - Push, pop, peek
    - Iterate from top to bottom
    - Check empty
      Then verify your work.
3. Solve these 4 problems in order, in LeetCode editor only:
    - **Valid Parentheses (#20)** — your nemesis. Solve it. Then close the tab and solve it again from scratch.
    - **Min Stack (#155)** — design problem with stack mechanics
    - **Baseball Game (#682)** — pure stack semantics
    - **Remove All Adjacent Duplicates In String (#1047)** — stack hidden in a string problem
4. After each: write the pattern entry. For Valid Parentheses, the entry should literally be: *"Matching pairs / brackets / valid expression → STACK. Always stack. No exceptions."*

**Afternoon Block (2.5 hrs) — HASHMAP + ARRAY**

Highest frequency in interviews. You probably remember more here than you think.

5. HashMap syntax drill (10 min): Without looking, write:
    - `Map<String, Integer>` declaration with `HashMap`
    - `put`, `get`, `containsKey`, `getOrDefault`
    - `computeIfAbsent` for adjacency lists
    - Iterate keys, values, entries
6. Solve these 4 problems:
    - **Two Sum (#1)** — the most-asked problem in history. Two passes acceptable, one pass with HashMap is the target.
    - **Contains Duplicate (#217)** — pure HashSet
    - **Valid Anagram (#242)** — frequency map (or 26-char array)
    - **Best Time to Buy and Sell Stock (#121)** — single-pass scan, running min
7. Pattern entries after each.

**Evening Block (45 min) — Java Fundamentals Cheat Sheet (Section 8)**

Read it once, slowly. Don't try to memorize. Just expose your brain to it.

**Day 1 success criterion:** When you see a problem statement involving brackets, parentheses, or "matching" anything, your immediate reaction is **STACK** — no thinking required. If that's true at end of day, Day 1 worked.

---

### Day 2: Trees, Linked Lists, Two Pointers/Sliding Window

**Theme: the next-highest-frequency patterns at Karat. Trees especially — they show up in 40%+ of senior Karat coding problems.**

**Morning Block (2.5 hrs) — TREES**

This is the single most important pattern after Stack/HashMap for senior interviews.

1. Re-solve **Valid Parentheses** from scratch as warm-up (10 min). If you can't, you have a real problem — go back to Day 1.
2. Read Tree section in `java-data-structures-from-the-ground-up.md` (10 min)
3. Internalize the recursion question: *"What does this node need — what its children figured out, or what its ancestors decided?"*
4. Solve these problems in order:
    - **Maximum Depth of Binary Tree (#104)** — simplest possible recursion
    - **Invert Binary Tree (#226)** — structural manipulation
    - **Same Tree (#100)** — two-tree recursion
    - **Binary Tree Level Order Traversal (#102)** — *Medium*, but the canonical BFS-on-tree pattern. Cannot skip. Memorize the `queue.size()` snapshot pattern.
    - **Diameter of Binary Tree (#543)** — post-order recursion with global state. Common Karat pattern.
5. Pattern entries after each.

**The level-order template** (memorize cold):
```java
Queue<TreeNode> q = new ArrayDeque<>();
if (root != null) q.offer(root);
while (!q.isEmpty()) {
    int size = q.size();  // snapshot — critical
    for (int i = 0; i < size; i++) {
        TreeNode node = q.poll();
        // process node
        if (node.left != null) q.offer(node.left);
        if (node.right != null) q.offer(node.right);
    }
}
```

**Afternoon Block (2 hrs) — LINKED LIST**

Lower interview frequency than trees but always shows up at least once in mocks.

6. Linked list syntax drill (10 min): Without looking, write the dummy-node pattern.
7. Solve these problems:
    - **Reverse Linked List (#206)** — foundational. Do iteratively, then recursively.
    - **Merge Two Sorted Lists (#21)** — dummy node pattern
    - **Linked List Cycle (#141)** — Floyd's tortoise & hare. Must-know.
    - **Middle of the Linked List (#876)** — fast/slow pointer
8. Pattern entries.

**The dummy-node trick** (drill until automatic):
```java
ListNode dummy = new ListNode(0);
dummy.next = head;
ListNode prev = dummy;
// ...modify list...
return dummy.next;
```

**Evening Block (1 hr) — TWO POINTERS / SLIDING WINDOW (light touch)**

9. Solve 2 problems:
    - **Move Zeroes (#283)** — classic two-pointer
    - **Best Time to Buy and Sell Stock (#121)** — already done; redo from scratch as a sliding-window exercise

10. Read Section 7 (Pattern Recognition Cheat Sheet) end-to-end. By now most should feel intuitive.

**Day 2 success criterion:** When you see a tree problem, you can immediately say "post-order vs pre-order vs BFS" within 30 seconds.

---

### Day 3: Heap, Graph, Small-Class Design

**Theme: round out remaining high-value patterns + the design problem that's likely at senior level.**

**Morning Block (2 hrs) — HEAP (PriorityQueue)**

11. Re-solve **Diameter of Binary Tree** from scratch as warm-up (10 min).
12. Read Heap section in `java-data-structures-from-the-ground-up.md` (10 min)
13. Heap syntax drill (10 min) — write all three variants from memory:
    ```java
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    PriorityQueue<int[]> custom = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    ```
14. Solve these problems:
    - **Kth Largest Element in a Stream (#703)** — canonical "min-heap of size K"
    - **Last Stone Weight (#1046)** — pure max-heap simulation
    - **K Closest Points to Origin (#973)** — *Medium*, custom comparator
    - **Top K Frequent Elements (#347)** — *Medium*, HashMap + Heap combo. Most-asked combo problem.
15. Pattern entries.

**Afternoon Block (2 hrs) — GRAPH (BFS/DFS on grids only)**

We're cutting full graph theory. Just grids.

16. Read Graph section in `java-data-structures-from-the-ground-up.md` (10 min)
17. Memorize the grid DFS template:
    ```java
    void dfs(int[][] grid, int r, int c) {
        if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length || grid[r][c] != target) return;
        grid[r][c] = visited;
        dfs(grid, r+1, c); dfs(grid, r-1, c); dfs(grid, r, c+1); dfs(grid, r, c-1);
    }
    ```
18. Solve these problems:
    - **Flood Fill (#733)** — simplest grid DFS
    - **Number of Islands (#200)** — *Medium*. The single most-asked graph problem ever. Cannot skip.
    - **Max Area of Island (#695)** — *Medium*, variant with size tracking
19. Pattern entries.

**Evening Block (1.5 hrs) — SMALL-CLASS DESIGN**

20. Read Section 10. Drill these 3:
    - **LRU Cache (#146)** — *Medium*. HashMap + Doubly Linked List. Classic. Practice articulating: "I need O(1) lookup → HashMap. I need O(1) eviction of oldest → DLL. I link them: HashMap maps keys to DLL nodes."
    - **Min Stack (#155)** — already done Day 1, but re-articulate the design
    - **Logger Rate Limiter (#359)** — message rate limiting. HashMap-based.
21. For each, write out the design choices in plain English before coding. This is what Karat scores on design problems.

**Day 3 success criterion:** You can explain LRU Cache's design — choosing structures, complexity per method, why DLL not singly-linked — in under 90 seconds without coding.

---

### Day 4: Mock + Polish + Light Review

**Theme: integrate everything. Don't learn anything new. The interview is tomorrow (or today).**

**Morning Block (2 hrs) — FULL MOCK INTERVIEW**

This is the highest-leverage hour of the four days.

22. Pick ONE problem from the list below. Don't look at it first. Set a 35-min timer.
23. **Talk out loud** the entire time. Apply UMPIRE strictly. Pretend the recording matters (it does — review it after).
24. Pick from:
    - **Group Anagrams (#49)** — Medium, HashMap pattern
    - **Validate Binary Search Tree (#98)** — Medium, tree recursion with bounds
    - **Number of Islands (#200)** — Medium, redo as mock if you didn't fully internalize Day 3
    - **Top K Frequent Elements (#347)** — Medium, HashMap + Heap combo
25. After the timer: review honestly. Where did you fumble? Where did silences happen? Where did you miss the pattern signal? Write 5 bullet points of what to fix.

**Afternoon Block (1.5 hrs) — JAVA FUNDAMENTALS DRILLING**

26. Read Section 8 (Java Fundamentals Cheat Sheet) twice, slowly.
27. **Self-quiz aloud:** Cover the answers and ask yourself each question. Speak the answer in 1-3 sentences. If you can't, re-read.
28. The most likely Karat fundamentals areas (drill these specifically):
    - HashMap internals (buckets, hashing, Java 8 treeification)
    - `equals` and `hashCode` contract
    - `synchronized` vs `volatile` vs `Atomic*`
    - `ConcurrentHashMap`
    - ArrayList vs LinkedList
    - Checked vs unchecked exceptions
    - `final` keyword
    - Generics + type erasure (lighter touch)

**Evening Block (1.5 hrs) — POLISH**

29. **Re-read your pattern table.** Every entry. This is your interview-day reference.
30. **Re-read the Talk Track (Section 6).** Read it aloud once. Make it muscle memory.
31. **Re-solve ONE problem from each pattern** as a final sweep — pick the one you found hardest from each day. This is retrieval practice, not new learning.
32. **Stop early if you're tired.** A rested brain on Day 5 is worth more than 2 extra hours on Day 4.

**Day 4 success criterion:** You feel slightly bored, not panicked. That means it's stuck.

---

## 5. The UMPIRE Process (Apply to Every Problem)

The single most important habit. Apply this without exception, in prep and in the interview.

### U — Understand (60-90 seconds)

- Restate the problem in your own words. *"So we have an array of integers and need to return..."*
- Ask about edge cases out loud:
    - Empty input?
    - Single element?
    - Duplicates?
    - Negatives?
    - Overflow possible?
- Confirm input/output types and constraints.
- If unsure: ask. Karat interviewers will answer clarifying questions and you score points for asking.

### M — Match (30-60 seconds)

- *"This looks like a [pattern] problem because [signal]."*
- The signal might be: input shape, keywords ("contiguous," "sorted," "top K"), constraint sizes.
- If multiple patterns could fit, mention both: *"This could be solved with sorting or with a heap — let me think which is better here."*

### P — Plan (60-90 seconds)

- State the brute force first. *"Naive approach: O(n²) — for each element, scan the rest."*
- State the optimization and why it works. *"We can do better with a HashMap because we're recomputing complement lookups."*
- Walk through the plan in 3-5 steps in plain English.
- State expected complexity before coding.
- Trace through one example verbally.

### I — Implement (15-25 minutes)

- Now you can code.
- Narrate decisions as you type. *"I'll initialize the HashMap outside the loop. For each element, I check if its complement exists. If yes, return; if no, add it."*
- Use clear variable names. Not `i`, `j`, `tmp` for everything.
- Handle edge cases inline.

### R — Review (60-90 seconds)

- Trace through the code with a small example. Out loud.
- Check edge cases: empty input, single element, duplicates.
- Look for off-by-one errors.

### E — Evaluate (60 seconds)

- State final time complexity.
- State final space complexity.
- Mention one alternative approach and trade-off. *"An alternative is sorting + two pointers, which is O(n log n) time but O(1) space. I picked HashMap because the constraints don't suggest space pressure."*

**Total UMPIRE overhead:** ~5 minutes before coding, ~2 minutes after. In a 25-minute slot, that leaves ~18 minutes to code. Plenty.

---

## 6. The Karat Talk Track (Memorize This)

Read this aloud until it's muscle memory. Use it as the literal script for every problem.

> **[Receive problem]**
>
> *"Let me make sure I understand. We have [restate the problem]. The output should be [restate the output]. Just to confirm a few edge cases — should I handle empty input? What about duplicates? Are negative numbers possible?"*
>
> **[Wait for clarifications]**
>
> *"Got it. The brute force approach would be [X], which is O(n²). I think we can do better because we're [redundancy observation]."*
>
> *"This looks like a [pattern] problem because [signal — e.g., 'we need to find a pair that sums to a target']. My plan is:*
> *Step 1: [action]*
> *Step 2: [action]*
> *Step 3: [action]*
> *Expected complexity is O(n) time, O(n) space."*
>
> *"Before I code, let me trace through with [small example]. We start with [...], we do [...], and we end up with [expected output]. Plan looks correct."*
>
> **[Code, narrating decisions]**
>
> *"I'll use a HashMap because I need O(1) complement lookup. Let me initialize it outside the loop... For each element, I check if the complement exists... if yes I return the indices, if no I add this element to the map..."*
>
> **[Finish coding]**
>
> *"Let me trace through with [example] to verify. We start with i=0... at i=2 we find the complement... return [0, 2]. Correct."*
>
> *"Edge cases: empty input returns empty array; single element returns empty; duplicates handled because we check the map before adding."*
>
> *"Final complexity: O(n) time, O(n) space. An alternative would be sorting and using two pointers — O(n log n) time but O(1) space. I picked the HashMap because it's a single pass and the problem doesn't suggest space is constrained."*

If you do *only this script* — even with a slightly suboptimal solution — you outscore candidates with better solutions and worse communication. **This is the single highest-leverage habit in your prep.**

---

## 7. Pattern Recognition Cheat Sheet

The mental lookup table. Drill until automatic.

### Input → Pattern

| Input characteristic | First-instinct pattern |
|---|---|
| Sorted array | Binary search OR two pointers |
| Array + "subarray/substring" | Sliding window OR prefix sum |
| Array + "find pair/triplet summing to X" | HashMap (unsorted) OR two pointers (sorted) |
| Linked list + "find middle / detect cycle" | Fast & slow pointers |
| Tree + "level by level" | BFS with Queue |
| Tree + "path / depth / recursive structure" | DFS with recursion |
| Graph/grid + "shortest path, unweighted" | BFS |
| Graph/grid + "all paths / connectivity / regions" | DFS or Union-Find |
| "Top K" / "Kth largest/smallest" | Heap (PriorityQueue) |
| "Next greater / smaller element" | Monotonic stack |
| "Find max/min in every window" | Monotonic deque |
| "Matching pairs / valid expression / brackets" | **STACK. Always stack.** |
| "All combinations / permutations / subsets" | Backtracking |
| "Number of ways" / "min/max value of..." | Dynamic programming |
| "Prefix matching / autocomplete" | Trie |
| "Connected components / merge groups" | Union-Find |

### Keyword Triggers

- **"contiguous"** → sliding window or prefix sum
- **"subsequence"** (non-contiguous) → DP
- **"sorted"** → binary search or two pointers
- **"top K"** / **"Kth"** → heap
- **"shortest"** / **"minimum steps"** → BFS or DP
- **"all possible"** / **"enumerate"** → backtracking
- **"count the number of ways"** → DP
- **"in-place"** / **"O(1) space"** → two pointers
- **"streaming"** / **"online"** → heap
- **"connected"** / **"groups"** / **"islands"** → DFS/BFS/Union-Find
- **"prefix"** → Trie
- **"palindrome"** → expand-around-center or DP

### Constraint Hints

- n ≤ 20 → brute force, backtracking, bitmask
- n ≤ 500 → O(n³) acceptable, think DP
- n ≤ 10⁵ → need O(n log n) or O(n)
- n ≤ 10⁹ → O(log n) or O(1) only — binary search or math

### Multi-Pattern Decision

When two patterns work, articulate the trade-off:

**Kth largest:**
- Sort → O(n log n) time, O(1) space — simple baseline
- Min-heap of size K → O(n log K) time — better for streaming
- Quickselect → O(n) average, harder to code under pressure

State all three. Justify your pick. **This is what 8 YOE looks like.**

---

## 8. Java Fundamentals — One-Page Cheat Sheet

Be able to answer all of these in 1-3 sentences, crisply.

### Collections

**HashMap internals:** Array of buckets. Each key is hashed; hash modulo array size = bucket index. Collisions form a linked list. Since Java 8, when a bucket has ≥8 entries, it's converted to a balanced tree (O(log n) worst case instead of O(n)).

**HashMap vs Hashtable vs ConcurrentHashMap:** Hashtable is fully synchronized (legacy, slow). HashMap is unsynchronized. ConcurrentHashMap uses fine-grained locking (per-bucket since Java 8) — safe for multi-threaded use without global lock.

**ArrayList vs LinkedList:** ArrayList is array-backed — O(1) random access, amortized O(1) append, O(n) middle insert. LinkedList is doubly-linked — O(1) insert/remove at known position, O(n) random access. **In practice ArrayList wins almost always** because of cache locality.

**ArrayDeque vs Stack vs LinkedList:** ArrayDeque is the modern choice for both stack and queue use. Stack class is legacy (Java 1.0, synchronized via Vector). LinkedList implements Deque but with worse constants.

**TreeMap / TreeSet:** Red-Black tree (self-balancing BST). All ops O(log n). Useful when you need sorted iteration or range queries (`floorKey`, `ceilingKey`, `subMap`).

### `equals` and `hashCode`

**The contract:** If `a.equals(b)`, then `a.hashCode() == b.hashCode()`. The reverse need not hold (collisions are allowed).

**What breaks if you override `equals` without `hashCode`:** Object becomes unfindable in HashMap/HashSet — you put it in, but `get` or `contains` returns false because the bucket lookup uses `hashCode`.

**Why immutability matters:** If a key's hash code changes after it's in a map, it's effectively lost.

### Concurrency

**`synchronized`:** Acquires an intrinsic lock on an object's monitor. Reentrant (same thread can re-enter). Synchronized method on instance = lock on `this`. Synchronized static method = lock on the Class object.

**`volatile`:** Guarantees visibility (writes are seen by all threads immediately) but NOT atomicity. `volatile int x; x++` is still a race condition because `++` is read-modify-write.

**`AtomicInteger` etc.:** Lock-free atomic operations using CPU CAS (compare-and-swap). Faster than synchronized for simple counters.

**Race condition:** Multiple threads access shared state, outcome depends on timing.

**Deadlock:** Two+ threads waiting on each other's locks.

**Livelock:** Threads aren't blocked but can't make progress (e.g., both keep yielding).

**`ExecutorService`:** Manages thread pool. `Executors.newFixedThreadPool(n)` is the most common. Submit `Runnable` or `Callable`. Don't create threads manually in production code.

**`CompletableFuture`:** Async chaining. `supplyAsync(() -> ...).thenApply(...).thenCompose(...)`. Beats `Future` for composition.

### Generics

**Type erasure:** Generics exist at compile time only. At runtime, `List<String>` is just `List`.

**Wildcards (PECS):** `? extends T` (Producer — read-only), `? super T` (Consumer — write). "Producer Extends, Consumer Super."

**Why `List<String>` is not `List<Object>`:** Java generics are invariant by default. If they were covariant, you could put a `Integer` into a `List<String>` reference and it would compile.

### Exceptions

**Checked vs unchecked:** Checked must be declared or caught (IOException, SQLException). Unchecked are RuntimeExceptions (NullPointerException, IllegalArgumentException) — don't need declaration.

**Try-with-resources:** Auto-closes anything implementing `AutoCloseable`. Replaces verbose try/finally.

**When to use which:** Checked for recoverable conditions where caller must handle. Unchecked for programming errors.

### JVM Basics

**Heap vs stack memory:** Heap stores objects. Stack stores method frames and local primitives. Each thread has its own stack; heap is shared.

**Garbage collection:** Generational hypothesis — most objects die young. Young gen (Eden + Survivors) collected frequently and fast. Old gen collected less often but more expensive.

**`final`:** On variable = can't reassign. On method = can't override. On class = can't subclass.

### OOP / Design

**SOLID:**
- **S**ingle Responsibility — one reason to change
- **O**pen/Closed — open for extension, closed for modification
- **L**iskov Substitution — subtypes substitutable for base type
- **I**nterface Segregation — many small interfaces > one big one
- **D**ependency Inversion — depend on abstractions, not concretions

**Composition vs inheritance:** Modern preference for composition. Inheritance creates rigid hierarchies; composition lets you mix behavior.

**Interface vs abstract class:** Interface = pure contract (default methods allowed since Java 8). Abstract class = partial implementation + state.

---

## 9. Java Syntax You Must Have Cold

Because no Copilot in the interview. Drill these until they're automatic.

### Collections Initialization

```java
List<Integer> list = new ArrayList<>();
Set<Integer> set = new HashSet<>();
Map<String, Integer> map = new HashMap<>();
Deque<Integer> stack = new ArrayDeque<>();   // for stack use
Deque<Integer> queue = new ArrayDeque<>();   // for queue use
Queue<Integer> queue2 = new LinkedList<>();  // alternative
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
```

### Stack Operations (use ArrayDeque)

```java
stack.push(x);     // add to top
stack.pop();       // remove from top, returns value
stack.peek();      // look at top, doesn't remove
stack.isEmpty();
```

### Queue Operations

```java
queue.offer(x);    // add to back
queue.poll();      // remove from front, returns null if empty
queue.peek();      // look at front
```

### Map Operations

```java
map.put(key, value);
map.get(key);                              // null if missing
map.getOrDefault(key, 0);                  // your most-used method
map.containsKey(key);
map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);  // adjacency lists

// Iteration
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String k = entry.getKey();
    Integer v = entry.getValue();
}
```

### Frequency Counting

```java
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) {
    freq.put(c, freq.getOrDefault(c, 0) + 1);
}
// OR
for (char c : s.toCharArray()) {
    freq.merge(c, 1, Integer::sum);
}
```

### Heap with Custom Comparator

```java
// Sort by second element of int[]
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);

// Sort by frequency (HashMap value)
PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> freq.get(a) - freq.get(b));
```

### String Manipulation

```java
String s = "hello";
char[] chars = s.toCharArray();
StringBuilder sb = new StringBuilder();
sb.append('a').append("bc");
sb.reverse();
String result = sb.toString();
s.substring(1, 4);     // chars at index 1, 2, 3 (exclusive end)
s.charAt(0);
s.length();            // method, not field
s.equals(other);       // never use ==
Character.isDigit(c);
Character.isLetter(c);
Character.toLowerCase(c);
```

### Array

```java
int[] arr = new int[10];
arr.length;           // field, not method
Arrays.sort(arr);
Arrays.fill(arr, -1);
int[][] grid = new int[m][n];
int rows = grid.length;
int cols = grid[0].length;
```

### TreeNode and ListNode (LeetCode standard)

```java
public class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}

public class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}
```

### Common Idioms

```java
// Min/max
int max = Math.max(a, b);
int min = Math.min(a, b);

// Sort with comparator
Arrays.sort(arr, (a, b) -> a - b);  // ascending
Collections.sort(list, (a, b) -> b - a);  // descending

// Convert array <-> list
List<Integer> list = Arrays.asList(1, 2, 3);  // fixed-size!
Integer[] arr = list.toArray(new Integer[0]);
```

---

## 10. Small-Class Design — The 3 to Drill

Karat at senior level often includes one of these. Be ready.

### LRU Cache (#146) — The Classic

**The design articulation (practice saying this aloud):**
> *"I need O(1) get and O(1) put. HashMap gives me O(1) lookup. But I also need to evict the least-recently-used in O(1) — that requires fast removal of arbitrary nodes plus fast tracking of the oldest. Doubly linked list gives me that. So I combine them: HashMap maps key → DLL node. On get, I look up the node and move it to the head (most recent). On put, if at capacity, I remove the tail node from both the DLL and the HashMap, then add the new node at the head."*

**Implementation outline:**
```java
class LRUCache {
    class Node {
        int key, val;
        Node prev, next;
    }
    Map<Integer, Node> map;
    Node head, tail;  // dummy nodes
    int capacity;
    
    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        remove(node);
        addToHead(node);
        return node.val;
    }
    
    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.val = value;
            remove(node);
            addToHead(node);
        } else {
            if (map.size() == capacity) {
                map.remove(tail.prev.key);
                remove(tail.prev);
            }
            Node node = new Node(key, value);
            addToHead(node);
            map.put(key, node);
        }
    }
    
    private void remove(Node node) { /* unlink from DLL */ }
    private void addToHead(Node node) { /* insert after head */ }
}
```

### Min Stack (#155)

**Design articulation:**
> *"I need push, pop, top in O(1) — that's a stack. I also need getMin in O(1). I can't search the stack, so I need to track minimums alongside. I use a second stack of minimums: every time I push, I push min(new value, current min) onto the min stack. Every pop pops both. getMin returns the top of the min stack."*

### Logger Rate Limiter (#359)

**Design articulation:**
> *"I need to check whether a message has been printed in the last 10 seconds. HashMap from message → last-printed timestamp gives me O(1) check and update. On shouldPrintMessage(timestamp, message), if the message isn't in the map or its last timestamp + 10 ≤ current timestamp, update and return true. Otherwise return false."*

---

## 11. What I'm Cutting and Why

Time discipline. These are deliberately *not* in the plan.

| Topic | Why cut |
|---|---|
| Spring/Spring Boot | Not on Karat. Matters for post-Karat onsite, separate prep. |
| Backtracking | Lower Karat frequency at senior. If asked, you can fall back to recursion + global state. |
| Dynamic Programming | Hardest pattern, lowest ROI in 4 days. If asked, articulate brute-force recursion + memoization — that's 70% of DP problems. |
| Advanced graphs (Dijkstra, MST, Bellman-Ford) | Rare at Karat. |
| Trie, Union-Find | Rare at Karat-Medium. |
| Segment trees, Fenwick trees | Never at Karat. |
| Bit manipulation | Niche. If asked, you'll fall back on first principles. |
| System design | Not at Karat. Critical for onsite — separate prep. |
| 5-hour YouTube DSA course | Wasted time relative to active practice. |
| Reading *Cracking the Coding Interview* cover-to-cover | Useful as reference, terrible as primary prep. |

If the interviewer hits one of these and you don't know it: **stay calm, articulate what you'd do from first principles, ask clarifying questions, and try the brute force.** A thoughtful struggle beats a panicked surrender.

---

## 12. Day-of-Interview Playbook

### Night Before

- **Light review only.** Re-read your pattern table, the Talk Track (Section 6), and the Java Fundamentals Cheat Sheet (Section 8). Do *not* attempt new problems.
- Test the Karat platform setup if they've sent a practice link.
- Lay out water, paper, pen, charger.
- Phone on silent, in another room.
- Sleep early. Rested brain > extra prep.

### 30 Minutes Before

- Re-read the pattern table once.
- Re-read the Talk Track once.
- Stand up. Walk around. Drink water.
- Don't review code. At this point, more reading hurts.

### During the Interview

**The first 60 seconds matter most.** When the interviewer reads the problem, do NOT start coding. Do not even start thinking about code.

1. Restate the problem.
2. Ask 1-2 clarifying questions about edge cases. *Always.* Even if the problem is clear. This signals seniority.
3. State the brute force.
4. State the pattern.
5. State the plan.
6. *Now* code.

**If you get stuck mid-coding:**
- Don't go silent. Verbalize: *"I'm thinking about whether this needs a HashMap or a Set. The difference is..."*
- Silence reads as panic. Articulated uncertainty reads as engineering.

**If you finish early:**
- Trace through your code with another example.
- Ask: *"Want me to discuss how I'd extend this — for instance, to handle duplicates, or to be thread-safe?"*

**For Java fundamentals questions:**
- Answer in 1-3 sentences. Don't ramble.
- If unsure, say: *"I'm not 100% sure of the specific term but my understanding is..."* — better than a confident wrong answer.

**Final 2 minutes:**
- They'll ask if you have questions. Have one ready. Examples:
    - *"What does success look like for someone in this role at 6 months?"*
    - *"What does the team's tech stack and on-call rotation look like?"*
    - *"What's the most interesting technical challenge the team is working on right now?"*
- Avoid: salary, benefits, "do I need to know X?" — these are for the recruiter call.

### Immediately After

- Write down everything you can remember about the questions while it's fresh. Useful for next round and for any future Karat-style interviews.
- Don't replay it endlessly. The decision is out of your hands now.

---

## 13. Failure Mode Checklist

If you notice these failure patterns during prep, course-correct immediately.

- ❌ **Looking up solutions when stuck for 5 minutes.** Sit with it for 25 minutes minimum before any hint.
- ❌ **Coding before stating the plan.** Brain wants the dopamine of typing. Resist.
- ❌ **Solving silently.** If your prep is silent, your interview will be too. Force the talk-out-loud habit.
- ❌ **Skipping the pattern table entry.** Without it, Day 1 problems are forgotten by Day 4.
- ❌ **Doing problems out of pattern order, jumping around.** Pattern grouping is what builds recognition. Random shuffle is worse.
- ❌ **Re-reading vs. retrieval-practicing.** Re-reading feels productive, doesn't stick. Re-solving from scratch sticks.
- ❌ **Practicing in IntelliJ with autocomplete on.** That's not the interview environment. LeetCode editor only.
- ❌ **Cramming the night before.** Hurts more than helps.
- ❌ **Spending 20+ min on Java fundamentals when DSA needs work.** DSA is the larger grade component.

---

## TL;DR

- **Day 1:** Stack (priority — your gap), HashMap, Array.
- **Day 2:** Trees, Linked Lists, Two Pointers/Sliding Window.
- **Day 3:** Heap, Graph, Small-Class Design (LRU especially).
- **Day 4:** Mock interview, Java fundamentals review, light polish. Stop early.

**Non-negotiables:** No Copilot. Talk out loud. UMPIRE every problem. Pattern table entry after each.

**The talk track in Section 6 is your single highest-leverage habit.** Memorize it. It alone closes 30% of the gap between you and a candidate with 2 more years of prep.

You're not aiming for perfection. You're aiming for: pattern recognized in first 90 seconds, brute force articulated, optimization explained, code written while narrating, complexity stated. That candidate passes Karat at 8 YOE. That candidate is reachable in 4 days.

Go.