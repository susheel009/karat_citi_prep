# Data Structures in Java: From the Ground Up

A walkthrough of every data structure that matters for interview-level DSA in Java — starting with where the word came from, what it looks like in everyday life, what problem it actually solves, how Java exposes it, and the kinds of LeetCode patterns it shows up in.

---

## Foundation: What "Data Structure" Even Means

**Etymology.** *Data* is Latin, plural of *datum* — literally "a thing given." *Structure* comes from Latin *struere*, "to build, to arrange in layers" (same root as *construct* and *destroy*). A data structure is, word-for-word, **"an arrangement of given things."**

**Why we need different ones.** Memory is just a long row of bytes. Every data structure is a *convention* layered on top of that row to make some operation cheap (lookup, insert, sort, find-min) at the cost of making other operations more expensive. Picking a data structure is picking which trade-offs you want.

---

## 1. Array

### Etymology
From Old French *areer / arei*, meaning **"to arrange in order"** — especially troops in battle formation. "An array of soldiers" was a row of soldiers in fixed positions. The meaning leaked into mathematics and then computing.

### Real-world example
An **egg carton**. Twelve numbered slots, fixed at manufacture time. To get egg #7, you don't search — you reach directly to slot 7. The slots have a known size and a known position relative to the start of the carton.

### Problem it solves
"I have N items of the same kind. I want to grab item #i instantly without looking at the others." Without arrays, you'd have to walk through every item one by one.

### In Java
```java
int[] arr = new int[10];
arr[3] = 42;           // O(1) write
int x = arr[3];        // O(1) read
int len = arr.length;  // size baked in, not a method
```
- **Fixed size** — set at creation, can't grow.
- **Contiguous memory** — fast, cache-friendly. The JVM computes `address = base + i * sizeof(int)`, which is why indexing is O(1).
- **Insert/delete in the middle is O(n)** — everything to the right has to shift.

### Expected DSA problem types
- **Two-pointer** — *Container With Most Water, Trapping Rain Water, 3Sum*
- **Sliding window** — *Maximum Sum Subarray of Size K, Longest Substring Without Repeating Characters*
- **Prefix sum** — *Subarray Sum Equals K, Range Sum Query*
- **Cyclic sort** — *Find Missing Number, Find All Duplicates in an Array*
- **In-place rearrangement** — *Move Zeroes, Rotate Array*
- **Kadane's algorithm** — *Maximum Subarray*

---

## 2. Dynamic Array (`ArrayList`)

### Etymology
*Dynamic* is Greek *dynamis* — "power, capability." A dynamic array has the *capability* to grow. "ArrayList" is a Java naming choice — under the hood it's still an array.

### Real-world example
A **bookshelf you keep adding to**. When it fills up, you move every book to a bigger shelf. You don't move them on every add — only when you outgrow the current shelf. So most adds are cheap; occasional adds are expensive.

### Problem it solves
Arrays have fixed size. Real problems rarely give you the size upfront.

### In Java
```java
List<Integer> list = new ArrayList<>();
list.add(42);          // amortized O(1)
list.get(3);           // O(1)
list.remove(2);        // O(n) — shifts everything right of index 2
list.contains(42);     // O(n) — has to scan
```
- Backed by `Object[]`. When full, it allocates a new array (typically 1.5x size in Java) and copies. This is why "amortized O(1)" works — the expensive copy is rare enough to average out.
- **Random access is still O(1).** This is the key reason `ArrayList` beats `LinkedList` for most real workloads.

### Expected DSA problem types
- Anywhere you'd use an array but don't know the output size
- Building results in DFS/BFS where the count varies
- Result accumulation in problems like *Subsets, Permutations, Combinations*

---

## 3. Linked List

### Etymology
*List* is Old English *liste* — originally "a strip, a border, a catalog written on a strip of parchment." *Linked* is straightforward — chained together.

### Real-world example
A **scavenger hunt**. Each clue tells you only where the *next* clue is. You can't skip ahead to clue #5 without reading clues 1 through 4. But adding a new clue is trivial — write a note pointing to the next location, and have the previous clue point to your new note instead.

### Problem it solves
Arrays are slow to insert into the middle (everything shifts). Linked lists make insert/delete O(1) **if you already have a pointer to the spot** — at the cost of losing random access.

### In Java
```java
// Singly linked
class Node {
    int val;
    Node next;
}

// Java's built-in is doubly linked
LinkedList<Integer> ll = new LinkedList<>();
ll.addFirst(1);     // O(1)
ll.addLast(2);      // O(1)
ll.get(3);          // O(n) — has to walk from head
ll.removeFirst();   // O(1)
```
- `LinkedList` in Java implements both `List` and `Deque`.
- **Trade-off**: O(1) at the ends, O(n) anywhere else.
- **In practice**, `ArrayList` outperforms `LinkedList` for most real workloads because of cache locality. The pointer-chasing in linked lists kills CPU cache performance.

### Expected DSA problem types
This is its own LeetCode category. Patterns:
- **Reverse a linked list** — *Reverse Linked List, Reverse Nodes in K-Group*
- **Fast & slow pointers (Floyd's)** — *Linked List Cycle, Find Middle of Linked List*
- **Merge two lists** — *Merge Two Sorted Lists, Merge K Sorted Lists*
- **Remove Nth from end** (two pointers, gap of N)
- **Add two numbers** (digits stored in reverse)
- **LRU Cache** — combines doubly linked list + HashMap

---

## 4. Stack

### Etymology
From Old Norse *stakkr* — "a haystack, a pile." The English meaning of "a vertical pile of things" came from the shape of stacked hay.

### Real-world example
**Stack of plates in a cafeteria**. You put a clean plate on top, you take a plate from the top. The bottom plate has been there longest. **LIFO** — Last In, First Out. You physically *can't* take the bottom plate without removing everything above it first.

### Problem it solves
Anywhere you need to **remember things in reverse order** of when you saw them. Function call frames, undo histories, matching parentheses, backtracking — they all need "give me the most recent thing I haven't dealt with yet."

### In Java
```java
// Don't use java.util.Stack — it's legacy, synchronized, slow
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);     // O(1)
stack.push(2);
int top = stack.peek();  // 2, doesn't remove
int x = stack.pop();     // 2, removes
```
- **Use `ArrayDeque` for stack semantics.** The `Stack` class extends `Vector` from Java 1.0 and is universally considered a mistake.
- All operations are O(1).

### Expected DSA problem types
- **Balanced parentheses** — *Valid Parentheses, Minimum Remove to Make Valid*
- **Monotonic stack** — *Next Greater Element, Daily Temperatures, Largest Rectangle in Histogram*
- **Expression evaluation** — *Evaluate Reverse Polish Notation, Basic Calculator*
- **Backtracking simulation** — *Decode String*
- **DFS iterative** — any tree/graph DFS without recursion
- **Min Stack** (classic system design-y question)

---

## 5. Queue and Deque

### Etymology
*Queue* is from Latin *cauda* — "tail." Came through French where *queue* still means "tail" or "ponytail." A British queue at a bus stop is literally a "tail" of people.

*Deque* is short for **D**ouble-**E**nded **Que**ue, pronounced "deck."

### Real-world example
A **line at a coffee shop**. First person in line gets served first. **FIFO** — First In, First Out. You can't cut to the front, and the barista can't randomly serve someone in the middle.

A **deque** is a line where people can be added or removed from *either* end — like a passenger train where people board and exit from both the front and back doors.

### Problem it solves
- **Queue**: process things in arrival order. Critical for BFS, scheduling, buffering.
- **Deque**: when you need fast access to *both* ends — sliding window problems where you add to the back and remove from either side.

### In Java
```java
// Queue
Queue<Integer> q = new ArrayDeque<>();
q.offer(1);      // add to back, O(1)
q.poll();        // remove from front, O(1), returns null if empty
q.peek();        // look at front

// Deque
Deque<Integer> dq = new ArrayDeque<>();
dq.offerFirst(1);  dq.offerLast(2);
dq.pollFirst();    dq.pollLast();
```
- `ArrayDeque` is the modern, fast choice for both queue and stack.
- `LinkedList` also implements `Deque` but with worse constants.

### Expected DSA problem types
- **BFS** — *Number of Islands, Word Ladder, Rotting Oranges, Binary Tree Level Order Traversal*
- **Topological sort** (Kahn's algorithm) — *Course Schedule*
- **Multi-source BFS** — *Walls and Gates, 01 Matrix*
- **Sliding window maximum** (uses deque) — *Sliding Window Maximum*
- **Monotonic deque** — *Shortest Subarray with Sum at Least K*

---

## 6. Priority Queue (Heap)

### Etymology
*Heap* is Old English *heap* — "a pile, a mass." The data structure is named after the *visual* of a binary heap — a pile-like tree where the heaviest (or lightest) thing sits on top. *Priority queue* is a modern compound — a queue ordered by priority, not arrival.

### Real-world example
A **hospital ER triage**. People don't get seen in order of arrival — they get seen by severity. The patient with the gunshot wound jumps the line. A new arrival gets inserted at the right priority spot, and the next-most-critical patient is always next.

### Problem it solves
"Give me the smallest (or largest) element right now, and let me keep adding new elements." A sorted list would solve it, but inserts would be O(n). A heap gives O(log n) insert and O(1) peek-min.

### In Java
```java
// Min-heap (default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);  minHeap.offer(2);  minHeap.offer(8);
minHeap.peek();    // 2, O(1)
minHeap.poll();    // 2, O(log n)

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
```
- Implemented as a **binary heap** in an array. Parent of `i` is `(i-1)/2`; children are `2i+1` and `2i+2`.
- `offer` and `poll` are O(log n). `peek` is O(1). `contains` and arbitrary `remove` are O(n) — this is a common gotcha.

### Expected DSA problem types
This is one of the **highest-frequency** structures in interviews — Citi/Karat-style especially loves it.
- **Top-K problems** — *Top K Frequent Elements, Kth Largest Element in an Array*
- **K-way merge** — *Merge K Sorted Lists, Smallest Range Covering Elements from K Lists*
- **Two heaps** — *Find Median from Data Stream, Sliding Window Median*
- **Scheduling** — *Task Scheduler, Meeting Rooms II*
- **Dijkstra's shortest path**
- **Prim's MST**

---

## 7. Hash Table (`HashMap`, `HashSet`)

### Etymology
*Hash* is from French *hacher* — "to chop, to mince." Same root as "hatchet" and the breakfast dish "hash browns" — chopped potatoes. The hash *function* "chops up" your key into a number.

*Map* is from Latin *mappa* — originally "a cloth or napkin," then "a chart drawn on cloth." The data structure maps keys to values the way a paper map maps places to coordinates.

### Real-world example
A **library card catalog** (or its modern descendant, a contact list on your phone). You don't search every shelf for a book — you look up the title, get a code, and walk straight to that shelf. The catalog *maps* a title to a location.

A **hash function** is like a function that takes "John Smith" and tells you "drawer #47, slot #3" — directly, without any searching.

### Problem it solves
"I want to look something up by name (or any key) in O(1) time, not O(n)." This is arguably the most important data structure ever invented. It transforms search from a linear walk into a direct jump.

### In Java
```java
// HashMap — key/value
Map<String, Integer> map = new HashMap<>();
map.put("alice", 30);
map.get("alice");           // 30, O(1) average
map.containsKey("alice");
map.getOrDefault("bob", 0);  // your most-used method in DSA
map.computeIfAbsent("c", k -> new ArrayList<>()).add(1);  // graph adjacency lists

// HashSet — keys only, for membership tests
Set<Integer> seen = new HashSet<>();
seen.add(42);
seen.contains(42);  // O(1) average
```
- Average case O(1), worst case O(n) (bad hash distribution causing chains).
- Since Java 8, when a bucket has too many collisions (≥8), the chain is converted to a balanced tree, making worst case **O(log n)**.
- **No ordering guarantee.** Iteration order is not insertion order, not sorted, and can change between JVM versions.

### Expected DSA problem types
The single most-used structure in LeetCode Easy/Medium.
- **Two Sum** and every variant
- **Frequency counting** — *Group Anagrams, Top K Frequent Elements*
- **Seen/visited tracking** — *Contains Duplicate, Longest Consecutive Sequence*
- **Caching/memoization** in DP
- **Sliding window with character counts** — *Longest Substring Without Repeating Characters, Minimum Window Substring*
- **Adjacency lists** for graphs
- **Two-sum-style "complement"** — *Two Sum, 3Sum, 4Sum*

---

## 8. LinkedHashMap / LinkedHashSet

### Etymology
The "linked" prefix tells you how it's implemented — a hash table *plus* a doubly linked list weaving through the entries to remember insertion order.

### Real-world example
A **chronological diary with an index**. You can flip to any date instantly (the index = hash table), but if you read the entries in physical order, they're in the order you wrote them (the linked list).

### Problem it solves
"Hash map speed, but I want to iterate in insertion order." Or: "I want LRU cache behavior — quickly evict the oldest entry."

### In Java
```java
Map<String, Integer> ordered = new LinkedHashMap<>();
// Iteration follows insertion order

// LRU mode: access order = true, override removeEldestEntry
Map<Integer, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > CAPACITY;
    }
};
```

### Expected DSA problem types
- **LRU Cache** — classic interview question; can be solved with `LinkedHashMap` directly or implemented from scratch (HashMap + DoublyLinkedList).

---

## 9. TreeMap / TreeSet (Red-Black Tree)

### Etymology
*Tree* is Old English *treow* — the actual plant. Computer scientists borrowed the metaphor for any branching hierarchy. *Red-Black* is just the coloring rule used to keep the tree balanced.

### Real-world example
A **dictionary**. Words are sorted alphabetically. You can find a word by binary searching, you can find all words starting with "pre-", and you can ask "what's the word that comes right after 'apple' alphabetically?" — none of which a hash map can do.

### Problem it solves
You need hash-map-like fast lookup *plus* one of: sorted iteration, range queries, "next key greater than X," or "key just below X."

### In Java
```java
TreeMap<Integer, String> tm = new TreeMap<>();
tm.put(5, "five"); tm.put(2, "two"); tm.put(8, "eight");

tm.firstKey();        // 2 — smallest
tm.lastKey();         // 8 — largest
tm.ceilingKey(6);     // 8 — smallest key >= 6
tm.floorKey(6);       // 5 — largest key <= 6
tm.subMap(3, 7);      // range view
```
- All operations: **O(log n)**.
- Backed by a Red-Black Tree (a self-balancing BST).
- Slower than `HashMap` by a constant factor, but gives you ordering for free.

### Expected DSA problem types
- **Range queries** — *My Calendar I, II, III*
- **Floor / ceiling lookups** — *Find Right Interval*
- **Sliding window with sorted state** — *Sliding Window Median (alternative to two heaps)*
- **Sweep line algorithms**
- **Stock market / event scheduling problems**

---

## 10. Tree (General) and Binary Search Tree

### Etymology
Same *treow* as above. The CS metaphor — root, branches, leaves — maps so cleanly that the term stuck immediately.

### Real-world example
A **family tree** (general tree, any number of children). A **decision flowchart** where every question has exactly two answers — yes/no — is a **binary tree**. A **company org chart** where each manager has multiple reports is a general tree.

### Problem it solves
Hierarchies. Anything that branches. Plus, with the *binary search* property (left subtree < root < right subtree), you get O(log n) search/insert/delete on a balanced tree.

### In Java
There's no built-in `BinaryTree` class — you write the node yourself.
```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```
For a self-balancing BST with built-in operations, use `TreeMap` / `TreeSet`.

### Expected DSA problem types
Massive interview category.
- **Traversals** — preorder, inorder, postorder, level-order (BFS)
- **Path problems** — *Path Sum, Binary Tree Maximum Path Sum, Diameter of Binary Tree*
- **Construction** — *Construct Binary Tree from Preorder and Inorder*
- **Lowest Common Ancestor** — *LCA of a Binary Tree, LCA of a BST*
- **BST validation** — *Validate Binary Search Tree*
- **Serialization** — *Serialize and Deserialize Binary Tree*
- **Views** — *Right Side View, Top View, Bottom View*
- **Recursion patterns** — almost every tree problem is "solve for the children, combine for the parent"

---

## 11. Trie (Prefix Tree)

### Etymology
**Coined in 1960 by Edward Fredkin** from the middle syllable of *re**trie**val*. He wanted it pronounced "tree," but everyone else says "try" to disambiguate from regular trees.

### Real-world example
**Phone keypad autocomplete** or **search bar suggestions**. Type "ja" and Java, JavaScript, Jakarta, January all surface. The structure is a tree where each node is one letter, and following a path spells a word.

### Problem it solves
Searching by **prefix**. A hash map can tell you if "java" is a stored word, but it can't tell you "give me all words starting with 'jav-'" without scanning every key.

### In Java
No built-in. You implement it.
```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    boolean isEnd;
}

class Trie {
    TrieNode root = new TrieNode();
    
    void insert(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (node.children[i] == null) node.children[i] = new TrieNode();
            node = node.children[i];
        }
        node.isEnd = true;
    }
}
```
- Insert / search: **O(L)** where L is word length.
- Memory-heavy — each node holds 26 pointers (or a HashMap).

### Expected DSA problem types
- **Implement Trie**
- **Word Search II** (Trie + DFS on grid — classic medium-hard)
- **Replace Words, Add and Search Word**
- **Autocomplete / typeahead**
- **Longest common prefix** of many strings

---

## 12. Graph

### Etymology
Greek *graphein* — "to write, to draw." Originally meant any drawing. In math, **Euler's 1736 paper on the Königsberg bridges** is considered the birth of graph theory — he drew the bridges and landmasses as dots and lines, calling it a *graph*.

### Real-world example
**A road map.** Cities are *nodes*; roads between them are *edges*. **Social networks** — people are nodes, friendships are edges. **The internet** — pages are nodes, hyperlinks are (directed) edges.

### Problem it solves
Anything where the entities matter *and* the connections between them matter. Trees are a special case of graphs (no cycles, one root).

### In Java
No built-in graph class — you build one out of `Map` and `List`.
```java
// Adjacency list — most common
Map<Integer, List<Integer>> graph = new HashMap<>();
graph.computeIfAbsent(u, k -> new ArrayList<>()).add(v);

// For weighted edges
Map<Integer, List<int[]>> weighted = new HashMap<>();
weighted.computeIfAbsent(u, k -> new ArrayList<>()).add(new int[]{v, weight});

// Adjacency matrix — when graph is dense
int[][] adj = new int[n][n];
adj[u][v] = 1;
```
- **Adjacency list**: O(V+E) space. Good for sparse graphs (most real graphs).
- **Adjacency matrix**: O(V²) space. Faster edge lookup but wasteful unless dense.

### Expected DSA problem types
- **BFS** — *Number of Islands, Rotting Oranges, Word Ladder, Open the Lock*
- **DFS** — *Number of Islands (DFS variant), Surrounded Regions, Pacific Atlantic Water Flow*
- **Topological sort** — *Course Schedule I & II, Alien Dictionary*
- **Shortest path (Dijkstra)** — *Network Delay Time, Cheapest Flights Within K Stops*
- **Cycle detection** — both directed (DFS coloring) and undirected (Union-Find)
- **Connected components** — *Number of Provinces, Number of Connected Components in an Undirected Graph*

---

## 13. Union-Find (Disjoint Set Union)

### Etymology
Descriptive — a structure for *finding* which set something belongs to and *uniting* sets together. Sometimes called **DSU** for Disjoint Set Union.

### Real-world example
**Merging companies.** Each company has employees. When two companies merge, all their employees now belong to the same parent organization. Given any two employees, you can ask: "Do they work for the same parent company now?"

### Problem it solves
"Given a stream of merge operations and connectivity queries, answer them all fast." Naive approach is BFS each time — O(V+E) per query. Union-Find is **near-O(1) per query** with path compression + union by rank.

### In Java
No built-in. Standard implementation:
```java
class UnionFind {
    int[] parent, rank;
    
    UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);  // path compression
        return parent[x];
    }
    
    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
        return true;
    }
}
```

### Expected DSA problem types
- **Connectivity / components** — *Number of Provinces, Number of Connected Components*
- **Cycle detection in undirected graph** — *Graph Valid Tree, Redundant Connection*
- **Kruskal's MST**
- **Account merging** — *Accounts Merge*
- **Dynamic connectivity** problems where edges are added incrementally

---

## 14. Strings (Java's Quirks)

### Etymology
*String* is Old English *streng* — "a cord, a line." Came to mean "a sequence" — a string of pearls, a string of letters.

### Why strings deserve their own section in Java
- **Immutable.** Every `+`, `replace`, or `substring` creates a new object. In a loop, this is O(n²).
- **Use `StringBuilder`** for concatenation in loops — O(n) total.
- **`String.equals()`** for content comparison; `==` checks reference identity.
- **`charAt(i)`** is O(1). `substring(i, j)` in Java 7+ creates a copy — O(n).

### Expected DSA problem types
- **Palindrome problems** — *Valid Palindrome, Longest Palindromic Substring*
- **Anagrams** — *Group Anagrams, Valid Anagram*
- **Pattern matching** — *Implement strStr (KMP), Repeated String Pattern*
- **String DP** — *Edit Distance, Longest Common Subsequence*
- **Sliding window over chars** — *Longest Substring Without Repeating Characters, Minimum Window Substring*

---

## 15. Advanced (Brief Mention)

These show up at the harder end and are **not Karat-priority** but worth knowing they exist.

### Segment Tree
Range queries (sum, min, max) with point updates in O(log n). Used in *Range Sum Query - Mutable*. Java has no built-in; you write an array-based version.

### Fenwick Tree (Binary Indexed Tree, BIT)
Lighter alternative to segment tree for prefix sums with updates. O(log n) for both. Slick bit-manipulation implementation.

### Suffix Array / Suffix Tree
Advanced string structures. Mostly competitive programming territory.

### Skip List
Probabilistic alternative to balanced BSTs. Used in `ConcurrentSkipListMap`.

---

## Cheat Sheet for Karat-Style Interviews

| Pattern you see in the problem | Reach for |
|---|---|
| "Find pair / triple summing to X" | HashMap |
| "Top K / Kth largest" | PriorityQueue |
| "Shortest path in unweighted graph" | Queue (BFS) |
| "Levels of a tree" | Queue (BFS) |
| "All paths in a tree" | Recursion / Stack (DFS) |
| "Balanced parens / next greater" | Stack |
| "Sliding window max" | Deque |
| "Range / sorted lookups" | TreeMap |
| "Prefix lookup" | Trie |
| "Connected components / dynamic merge" | Union-Find |
| "Insertion order matters with O(1) ops" | LinkedHashMap |
| "Merge K sorted things" | PriorityQueue |
| "Cycle in linked list / find middle" | Two pointers |

---

## How to Internalize This

The structures aren't independent — they compose:
- **LRU Cache** = HashMap + Doubly Linked List
- **Min Stack** = Stack + Stack (or Stack of pairs)
- **Top K Frequent** = HashMap (count) + PriorityQueue (rank)
- **Word Search II** = Trie + DFS + visited set
- **Dijkstra** = Graph + PriorityQueue + HashMap (distances)

When you see a problem, the question isn't "which one structure" — it's "which combination gives me the right operations cheaply." That mental model is what separates fluent DSA from memorized DSA.