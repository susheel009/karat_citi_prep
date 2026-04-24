### Topic: Binary Tree (Balanced Trees, Advanced Operations) — Level 3

> **Foundation:** [Level 2 — Binary Tree (BST, traversals)](../level-2/binary_tree.md)

**Why it matters (Karat angle)**
L2 covered BST implementation and traversals. L3 covers **balanced trees** (AVL, red-black) — why they exist, how rotations work, and the connection to `TreeMap`/`TreeSet`. Also covers advanced operations: lowest common ancestor, tree validation, and serialization — common Codility/Karat questions.

**Core concept**

**Why balancing matters:**
| Tree type | Insert/Search | Worst case |
|-----------|:------------:|:----------:|
| BST (unbalanced) | O(log n) average | O(n) — degenerate |
| AVL tree | O(log n) | O(log n) — strict balance |
| Red-black tree | O(log n) | O(log n) — relaxed balance |

**AVL tree — strictly balanced (height difference ≤ 1):**
```
Balance factor = height(left) - height(right)
Valid: -1, 0, +1
Invalid: -2, +2 → rotation needed
```

**Red-black tree (used by `TreeMap`, `TreeSet`, HashMap buckets):**
Rules:
1. Every node is red or black.
2. Root is black.
3. No two consecutive red nodes (parent-child).
4. Every path from root to null has the same number of black nodes.

Guarantees: longest path ≤ 2× shortest path → O(log n).

**Rotations (the balancing mechanism):**

```java
// File: topics/level-3/RotationDemo.java

// RIGHT ROTATION (when left-heavy):
//       30              20
//      /  \            /  \
//     20    35  →    10    30
//    /  \                 /  \
//   10   25             25    35

Node rightRotate(Node y) {
    Node x = y.left;
    Node T2 = x.right;
    x.right = y;
    y.left = T2;
    // update heights
    y.height = 1 + Math.max(height(y.left), height(y.right));
    x.height = 1 + Math.max(height(x.left), height(x.right));
    return x;                                             // x is new root
}

// LEFT ROTATION (when right-heavy):
//     10                 20
//    /  \              /    \
//   5    20    →     10      30
//       /  \        /  \
//      15   30     5    15

Node leftRotate(Node x) {
    Node y = x.right;
    Node T2 = y.left;
    y.left = x;
    x.right = T2;
    x.height = 1 + Math.max(height(x.left), height(x.right));
    y.height = 1 + Math.max(height(y.left), height(y.right));
    return y;
}
```

**Advanced operations:**

```java
// File: topics/level-3/AdvancedTreeOps.java

// --- Validate BST (is it actually a valid BST?) ---
boolean isValidBST(Node node, long min, long max) {
    if (node == null) return true;
    if (node.key <= min || node.key >= max) return false;
    return isValidBST(node.left, min, node.key)
        && isValidBST(node.right, node.key, max);
}
// Usage: isValidBST(root, Long.MIN_VALUE, Long.MAX_VALUE)

// --- Lowest Common Ancestor (LCA) in BST ---
Node lcaBST(Node root, int p, int q) {
    if (root == null) return null;
    if (p < root.key && q < root.key) return lcaBST(root.left, p, q);
    if (p > root.key && q > root.key) return lcaBST(root.right, p, q);
    return root;                                          // split point → LCA
}

// --- LCA in general binary tree (not BST) ---
Node lcaGeneral(Node root, int p, int q) {
    if (root == null || root.key == p || root.key == q) return root;
    Node left = lcaGeneral(root.left, p, q);
    Node right = lcaGeneral(root.right, p, q);
    if (left != null && right != null) return root;       // split point
    return left != null ? left : right;
}

// --- Tree height / depth ---
int height(Node node) {
    if (node == null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
}

// --- Check if balanced ---
boolean isBalanced(Node node) {
    if (node == null) return true;
    int diff = Math.abs(height(node.left) - height(node.right));
    return diff <= 1 && isBalanced(node.left) && isBalanced(node.right);
}

// --- Serialize/Deserialize (pre-order) ---
void serialize(Node node, StringBuilder sb) {
    if (node == null) { sb.append("null,"); return; }
    sb.append(node.key).append(",");
    serialize(node.left, sb);
    serialize(node.right, sb);
}

Node deserialize(Queue<String> tokens) {
    String val = tokens.poll();
    if (val == null || "null".equals(val)) return null;
    Node node = new Node(Integer.parseInt(val));
    node.left = deserialize(tokens);
    node.right = deserialize(tokens);
    return node;
}
```

**Connection to Java Collections:**
- `TreeMap` and `TreeSet` use red-black trees → O(log n) for all operations, sorted order maintained.
- `HashMap` treeifies buckets with >8 entries using red-black tree nodes.

**What to say in the interview (4-beat answer)**
1. **Definition:** Balanced trees (AVL, red-black) guarantee O(log n) operations by maintaining height balance through rotations. Red-black trees are used by Java's `TreeMap`, `TreeSet`, and HashMap's treeified buckets.
2. **Why/when:** Unbalanced BSTs degrade to O(n) with sorted input. Balanced trees prevent this. Common interview operations: validate BST, find LCA, serialize/deserialize.
3. **Example:** LCA in a BST — if both targets are less than root, go left; both greater, go right; otherwise root is the split point. O(log n) for balanced BST.
4. **Gotcha/tradeoff:** AVL trees are more strictly balanced (faster lookups) but more rotations on insert/delete. Red-black trees are less balanced but fewer rotations — better for write-heavy workloads. Java chose red-black for this reason.

**Common pitfalls**
- Validating BST by only checking immediate children — must pass min/max range down recursively.
- Confusing BST LCA (use BST property) with general tree LCA (use recursion on both subtrees).
- Forgetting to update heights after rotations — breaks balance factor calculations.
- Implementing iterative tree operations without an explicit stack — recursion limit on deep trees.

**Self-check question**
Insert the sequence [1, 2, 3, 4, 5] into a plain BST and then into an AVL tree. Draw both trees. How many rotations does the AVL tree perform?
