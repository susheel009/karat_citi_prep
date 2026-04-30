# Codility / DSA Patterns — Java Reference

**Purpose:** 12 patterns covering ~80% of Codility-level problems and Karat's live-coding probes. Each pattern has:
- **When to recognize it** — signals in the problem statement
- **Template code** — Java, compile-ready
- **Example problems** — Codility tasks + common names
- **Complexity**

**How to use this file:**
- Read all patterns once (90 min).
- For each, type out the template by hand in IntelliJ — muscle memory matters.
- Attempt 1 easy problem per pattern. Log in `codility/attempts/`.
- Redo the weak patterns day-over-day, not all at once.

---

## Pattern 1 — Two Pointers (opposite ends)

**Recognize when:** sorted array, find pair/triple with a target property (sum == K, closest to target), palindrome check, reverse in place.

```java
int left = 0, right = arr.length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) return new int[]{left, right};
    if (sum < target) left++;
    else              right--;
}
return new int[]{-1, -1};
```

**Example problems:** Codility `Distinct`, `PassingCars`; LeetCode `Two Sum II`, `Valid Palindrome`, `Container With Most Water`.
**Complexity:** O(n) time, O(1) space.

---

## Pattern 2 — Two Pointers (fast / slow)

**Recognize when:** linked list cycle detection, find middle of list, remove duplicates in place from a sorted array.

```java
// Middle of linked list
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
}
return slow;            // middle node (or second-middle if even length)

// Cycle detection (Floyd's)
ListNode s = head, f = head;
while (f != null && f.next != null) {
    s = s.next;
    f = f.next.next;
    if (s == f) return true;
}
return false;
```

**Example problems:** LeetCode `Linked List Cycle`, `Middle of the Linked List`, `Remove Duplicates from Sorted Array`.
**Complexity:** O(n) time, O(1) space.

---

## Pattern 3 — Sliding Window (fixed size)

**Recognize when:** find max/min/sum in every window of size K in an array.

```java
int sum = 0, maxSum = Integer.MIN_VALUE;
for (int i = 0; i < k; i++) sum += arr[i];        // first window
maxSum = sum;
for (int i = k; i < arr.length; i++) {
    sum += arr[i] - arr[i - k];                   // slide by one
    maxSum = Math.max(maxSum, sum);
}
return maxSum;
```

**Example problems:** Codility `MaxCounters` (variant); LeetCode `Maximum Sum Subarray of Size K`, `Average of Subarrays of Size K`.
**Complexity:** O(n) time, O(1) space.

---

## Pattern 4 — Sliding Window (variable size)

**Recognize when:** longest/shortest substring or subarray satisfying a condition (no repeated chars, sum ≥ K, at most K distinct).

```java
int left = 0, best = 0;
Map<Character, Integer> window = new HashMap<>();
for (int right = 0; right < s.length(); right++) {
    char c = s.charAt(right);
    window.merge(c, 1, Integer::sum);

    while (window.get(c) > 1) {                   // violation
        char leftChar = s.charAt(left);
        window.merge(leftChar, -1, Integer::sum);
        if (window.get(leftChar) == 0) window.remove(leftChar);
        left++;
    }
    best = Math.max(best, right - left + 1);
}
return best;
```

**Example problems:** LeetCode `Longest Substring Without Repeating Characters`, `Longest Substring with At Most K Distinct`, `Minimum Window Substring`.
**Complexity:** O(n) time, O(k) space.

---

## Pattern 5 — HashMap Frequency

**Recognize when:** count occurrences, group-by, "first non-repeating", anagram check, majority element.

```java
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);

// First non-repeating character
for (int i = 0; i < s.length(); i++) {
    if (freq.get(s.charAt(i)) == 1) return i;
}
return -1;

// Anagram check of two strings (same length assumed)
Map<Character, Integer> diff = new HashMap<>();
for (int i = 0; i < a.length(); i++) {
    diff.merge(a.charAt(i),  1, Integer::sum);
    diff.merge(b.charAt(i), -1, Integer::sum);
}
return diff.values().stream().allMatch(v -> v == 0);
```

**Example problems:** Codility `OddOccurrencesInArray`, `FrogJmp`; LeetCode `Valid Anagram`, `Group Anagrams`, `First Unique Character`.
**Complexity:** O(n) time, O(k) space.

---

## Pattern 6 — Prefix Sum + HashMap

**Recognize when:** subarray with sum K, count of subarrays with specific sum, running totals.

```java
// Number of subarrays with sum == k
Map<Integer, Integer> seen = new HashMap<>();
seen.put(0, 1);                                     // empty prefix
int sum = 0, count = 0;
for (int n : arr) {
    sum += n;
    count += seen.getOrDefault(sum - k, 0);         // any prefix that gives the diff
    seen.merge(sum, 1, Integer::sum);
}
return count;
```

**Example problems:** Codility `GenomicRangeQuery`; LeetCode `Subarray Sum Equals K`, `Continuous Subarray Sum`.
**Complexity:** O(n) time, O(n) space.

---

## Pattern 7 — Stack (including monotonic stack)

**Recognize when:** balanced parentheses, next greater/smaller element, stock span, histogram area, reverse Polish evaluation.

```java
// Balanced parentheses
Deque<Character> stack = new ArrayDeque<>();
for (char c : s.toCharArray()) {
    if (c == '(' || c == '[' || c == '{') stack.push(c);
    else {
        if (stack.isEmpty()) return false;
        char open = stack.pop();
        if (!matches(open, c)) return false;
    }
}
return stack.isEmpty();

// Next greater element (monotonic decreasing stack)
int[] result = new int[arr.length];
Deque<Integer> st = new ArrayDeque<>();             // stack of indices
for (int i = arr.length - 1; i >= 0; i--) {
    while (!st.isEmpty() && arr[st.peek()] <= arr[i]) st.pop();
    result[i] = st.isEmpty() ? -1 : arr[st.peek()];
    st.push(i);
}
```

**Example problems:** Codility `Brackets`, `Fish`, `StoneWall`; LeetCode `Valid Parentheses`, `Daily Temperatures`, `Largest Rectangle in Histogram`.
**Complexity:** O(n) time, O(n) space.

---

## Pattern 8 — BFS (queue, level-order)

**Recognize when:** shortest path in unweighted graph, level order of a tree, "minimum number of steps to..."

```java
// Level order traversal of binary tree
List<List<Integer>> levels = new ArrayList<>();
if (root == null) return levels;

Deque<TreeNode> queue = new ArrayDeque<>();
queue.offer(root);

while (!queue.isEmpty()) {
    int size = queue.size();                        // freeze the level boundary
    List<Integer> level = new ArrayList<>(size);
    for (int i = 0; i < size; i++) {
        TreeNode node = queue.poll();
        level.add(node.val);
        if (node.left  != null) queue.offer(node.left);
        if (node.right != null) queue.offer(node.right);
    }
    levels.add(level);
}
return levels;

// Shortest path in unweighted graph (BFS from src to dst)
int[] dist = new int[n];
Arrays.fill(dist, -1);
dist[src] = 0;
Deque<Integer> q = new ArrayDeque<>();
q.offer(src);
while (!q.isEmpty()) {
    int u = q.poll();
    for (int v : adj[u]) {
        if (dist[v] == -1) {
            dist[v] = dist[u] + 1;
            q.offer(v);
        }
    }
}
return dist[dst];
```

**Example problems:** LeetCode `Binary Tree Level Order Traversal`, `Word Ladder`, `01 Matrix`, `Rotting Oranges`.
**Complexity:** O(V + E) time, O(V) space.

---

## Pattern 9 — DFS (recursive or stack)

**Recognize when:** all paths, connected components, flood fill, tree path sum, backtracking building.

```java
// Count connected components in a grid (4-direction flood fill)
int count = 0;
int[][] dirs = {{0,1},{1,0},{0,-1},{-1,0}};

for (int i = 0; i < grid.length; i++) {
    for (int j = 0; j < grid[0].length; j++) {
        if (grid[i][j] == '1') {
            count++;
            dfs(grid, i, j, dirs);
        }
    }
}
return count;

void dfs(char[][] grid, int i, int j, int[][] dirs) {
    if (i < 0 || j < 0 || i >= grid.length || j >= grid[0].length) return;
    if (grid[i][j] != '1') return;
    grid[i][j] = '0';                               // mark visited
    for (int[] d : dirs) dfs(grid, i + d[0], j + d[1], dirs);
}
```

**Example problems:** LeetCode `Number of Islands`, `Max Area of Island`, `Flood Fill`, `Path Sum`.
**Complexity:** O(V + E) time, O(V) stack.

---

## Pattern 10 — Binary Search (classic + on answer)

**Recognize when:**
- **Classic:** sorted array, find value / insertion point / first occurrence / last occurrence.
- **On answer:** find min/max value that satisfies a monotonic predicate (min days to complete, max weight to split, etc.).

```java
// Classic: first occurrence of target
int lo = 0, hi = arr.length - 1, result = -1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;                   // overflow-safe midpoint
    if (arr[mid] == target) {
        result = mid;
        hi = mid - 1;                               // keep searching left
    } else if (arr[mid] < target) lo = mid + 1;
    else                          hi = mid - 1;
}
return result;

// Binary search on the answer (e.g. min days to ship within capacity K)
int lo = maxWeight, hi = totalWeight;               // bounds of possible answers
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (canShipWithin(mid, days)) hi = mid;         // mid works, try smaller
    else                          lo = mid + 1;
}
return lo;
```

**Example problems:** Codility `CountDiv`; LeetCode `Binary Search`, `Search Insert Position`, `Capacity to Ship Packages within D Days`, `Koko Eating Bananas`.
**Complexity:** O(log n) for classic; O(n log range) for answer-search.

---

## Pattern 11 — Heap (PriorityQueue)

**Recognize when:** Kth largest / smallest, merge K sorted lists, top-K frequent, scheduling.

```java
// Kth largest element — min-heap of size K
PriorityQueue<Integer> minHeap = new PriorityQueue<>();      // natural order
for (int n : arr) {
    minHeap.offer(n);
    if (minHeap.size() > k) minHeap.poll();                  // evict smallest
}
return minHeap.peek();                                       // Kth largest

// Top-K frequent elements
Map<Integer, Integer> freq = new HashMap<>();
for (int n : arr) freq.merge(n, 1, Integer::sum);

PriorityQueue<Map.Entry<Integer, Integer>> heap =
    new PriorityQueue<>(Comparator.comparingInt(Map.Entry::getValue));
for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
    heap.offer(e);
    if (heap.size() > k) heap.poll();
}
List<Integer> result = new ArrayList<>();
while (!heap.isEmpty()) result.add(heap.poll().getKey());
```

**Example problems:** LeetCode `Kth Largest Element in Array`, `Top K Frequent Elements`, `Merge K Sorted Lists`.
**Complexity:** O(n log k).

---

## Pattern 12 — Basic Dynamic Programming (1D)

**Recognize when:** "count the number of ways," "min/max cost to reach X," overlapping subproblems.

```java
// Climbing stairs: ways to reach step n taking 1 or 2 steps
int[] dp = new int[n + 1];
dp[0] = 1; dp[1] = 1;
for (int i = 2; i <= n; i++) dp[i] = dp[i - 1] + dp[i - 2];
return dp[n];

// House robber: max money without robbing two adjacent houses
int prev2 = 0, prev1 = 0;
for (int money : houses) {
    int curr = Math.max(prev1, prev2 + money);
    prev2 = prev1;
    prev1 = curr;
}
return prev1;

// Coin change: min coins to make amount (or -1)
int[] dp = new int[amount + 1];
Arrays.fill(dp, amount + 1);                        // sentinel > any real answer
dp[0] = 0;
for (int a = 1; a <= amount; a++) {
    for (int coin : coins) {
        if (coin <= a) dp[a] = Math.min(dp[a], dp[a - coin] + 1);
    }
}
return dp[amount] > amount ? -1 : dp[amount];
```

**Example problems:** LeetCode `Climbing Stairs`, `House Robber`, `Coin Change`, `Maximum Subarray` (Kadane), `Longest Increasing Subsequence`.
**Complexity:** usually O(n) or O(n × m).

---

## Difficulty ladder (attempt in this order)

**Day 1-2 (easy, confidence builds):**
1. Two Sum II — two pointers (Pattern 1)
2. Valid Palindrome — two pointers (Pattern 1)
3. Maximum Sum Subarray of Size K — sliding fixed (Pattern 3)
4. First Unique Character — hashmap freq (Pattern 5)
5. Valid Parentheses — stack (Pattern 7)
6. Number of Islands — DFS flood fill (Pattern 9)

**Day 3-4 (medium, bulk practice):**
7. Longest Substring Without Repeating — sliding variable (Pattern 4)
8. Subarray Sum Equals K — prefix sum + hashmap (Pattern 6)
9. Binary Tree Level Order — BFS (Pattern 8)
10. Daily Temperatures — monotonic stack (Pattern 7)
11. Kth Largest Element — heap (Pattern 11)
12. House Robber — 1D DP (Pattern 12)

**Day 5-6 (timed mediums, mix patterns):**
13. Longest Increasing Subsequence
14. Koko Eating Bananas (binary search on answer, Pattern 10)
15. Top K Frequent Elements (Pattern 11)
16. Word Ladder (BFS, Pattern 8)

---

## Codility-specific quirks

- Time limit is tight; O(n log n) usually passes, O(n²) usually does not.
- Edge cases love zero-length arrays and single-element arrays.
- Most Codility problems reduce to: array scan + hashmap OR two pointers OR stack.
- Read the full problem statement including the "performance" section — it tells you the target complexity.

---

## Template: log each attempt in `codility/attempts/YYYY-MM-DD-problem-name.md`

```markdown
# Problem: [name]

**Source:** Codility / LeetCode / other
**Pattern used:** [Two Pointers / Sliding Window / ...]
**Time attempted:** [e.g. 25 min]
**Result:** [passed / partial / failed]

## Approach
[2-3 sentences]

## What I struggled with
[specific — off-by-one, wrong data structure, missed edge case]

## Final code
```java
// working solution
```

## Lesson for next time
[one line]
```
