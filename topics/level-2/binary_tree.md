### Topic: Binary Tree — Level 2

> **Advanced:** [Level 3 — Binary Tree (balanced trees, advanced operations)](../level-3/binary_tree.md) · [Level 3 — LRU Cache](../level-3/lru_cache.md)

**Why it matters (Karat angle)**
Binary tree questions are a Codility and Karat staple. L2 expects you to implement a BST from scratch — insert, search, and all three traversal orders. It also tests whether you can articulate why `TreeMap` uses a red-black tree (L3 depth) and when a BST degrades to O(n).

**Core concept**

A **Binary Search Tree (BST)** is a binary tree where:
- Left subtree contains only nodes with keys **less than** the parent.
- Right subtree contains only nodes with keys **greater than** the parent.

| Operation | Average case | Worst case (degenerate/linear) |
|-----------|:----------:|:----------------------------:|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |
| In-order traversal | O(n) | O(n) |

Worst case happens when data is inserted in sorted order → tree becomes a linked list.

**Traversal orders:**

| Order | Visit sequence | Use case |
|-------|---------------|----------|
| **In-order** (LNR) | Left → Node → Right | Sorted output |
| **Pre-order** (NLR) | Node → Left → Right | Copy/serialize tree |
| **Post-order** (LRN) | Left → Right → Node | Delete tree / evaluate expressions |
| **Level-order** (BFS) | Level by level | Print tree by depth |

**Working code example**

```java
// File: topics/level-2/BinarySearchTree.java

public class BinarySearchTree {

    static class Node {
        int key;
        Node left, right;
        Node(int key) { this.key = key; }
    }

    private Node root;

    // ---------- Insert ----------
    public void insert(int key) {
        root = insertRec(root, key);
    }

    private Node insertRec(Node node, int key) {
        if (node == null) return new Node(key);
        if (key < node.key) node.left = insertRec(node.left, key);
        else if (key > node.key) node.right = insertRec(node.right, key);
        // duplicates ignored
        return node;
    }

    // ---------- Search ----------
    public boolean search(int key) {
        return searchRec(root, key);
    }

    private boolean searchRec(Node node, int key) {
        if (node == null) return false;
        if (key == node.key) return true;
        return key < node.key ? searchRec(node.left, key) : searchRec(node.right, key);
    }

    // ---------- In-order traversal (sorted) ----------
    public List<Integer> inOrder() {
        List<Integer> result = new ArrayList<>();
        inOrderRec(root, result);
        return result;
    }

    private void inOrderRec(Node node, List<Integer> result) {
        if (node == null) return;
        inOrderRec(node.left, result);
        result.add(node.key);                            // visit node between left and right
        inOrderRec(node.right, result);
    }

    // ---------- Pre-order traversal ----------
    public List<Integer> preOrder() {
        List<Integer> result = new ArrayList<>();
        preOrderRec(root, result);
        return result;
    }

    private void preOrderRec(Node node, List<Integer> result) {
        if (node == null) return;
        result.add(node.key);                            // visit node first
        preOrderRec(node.left, result);
        preOrderRec(node.right, result);
    }

    // ---------- Post-order traversal ----------
    public List<Integer> postOrder() {
        List<Integer> result = new ArrayList<>();
        postOrderRec(root, result);
        return result;
    }

    private void postOrderRec(Node node, List<Integer> result) {
        if (node == null) return;
        postOrderRec(node.left, result);
        postOrderRec(node.right, result);
        result.add(node.key);                            // visit node last
    }

    // ---------- Level-order (BFS) ----------
    public List<Integer> levelOrder() {
        List<Integer> result = new ArrayList<>();
        if (root == null) return result;
        Queue<Node> queue = new LinkedList<>();
        queue.add(root);
        while (!queue.isEmpty()) {
            Node node = queue.poll();
            result.add(node.key);
            if (node.left != null) queue.add(node.left);
            if (node.right != null) queue.add(node.right);
        }
        return result;
    }

    // ---------- Delete ----------
    public void delete(int key) {
        root = deleteRec(root, key);
    }

    private Node deleteRec(Node node, int key) {
        if (node == null) return null;
        if (key < node.key) node.left = deleteRec(node.left, key);
        else if (key > node.key) node.right = deleteRec(node.right, key);
        else {
            // Node found — three cases:
            if (node.left == null) return node.right;    // case 1 & 2: 0 or 1 child
            if (node.right == null) return node.left;
            // case 3: two children — replace with in-order successor
            Node successor = findMin(node.right);
            node.key = successor.key;
            node.right = deleteRec(node.right, successor.key);
        }
        return node;
    }

    private Node findMin(Node node) {
        while (node.left != null) node = node.left;
        return node;
    }

    public static void main(String[] args) {
        BinarySearchTree bst = new BinarySearchTree();
        int[] values = {50, 30, 70, 20, 40, 60, 80};
        for (int v : values) bst.insert(v);

        System.out.println("In-order:    " + bst.inOrder());     // [20, 30, 40, 50, 60, 70, 80]
        System.out.println("Pre-order:   " + bst.preOrder());    // [50, 30, 20, 40, 70, 60, 80]
        System.out.println("Post-order:  " + bst.postOrder());   // [20, 40, 30, 60, 80, 70, 50]
        System.out.println("Level-order: " + bst.levelOrder());  // [50, 30, 70, 20, 40, 60, 80]
        System.out.println("Search 40: " + bst.search(40));      // true
        bst.delete(30);
        System.out.println("After delete 30: " + bst.inOrder()); // [20, 40, 50, 60, 70, 80]
    }
}
```

**Connection to Java Collections:**
- `TreeMap` / `TreeSet` use a **red-black tree** (self-balancing BST) — guarantees O(log n) always (no degenerate case). Covered in L3.
- `HashMap` treeifies buckets into red-black trees when bucket size > 8 (Java 8+).

**What to say in the interview (4-beat answer)**
1. **Definition:** A BST is a binary tree where left children are smaller and right children are larger. Insert, search, and delete are O(log n) average, O(n) worst case (degenerate tree).
2. **Why/when:** BSTs are the foundation of `TreeMap`/`TreeSet`. In interviews, they test recursion, pointer manipulation, and understanding of time complexity degradation.
3. **Example:** In-order traversal of a BST produces sorted output — `[20, 30, 40, 50, 60, 70, 80]`. Delete with two children uses the in-order successor to maintain BST invariant.
4. **Gotcha/tradeoff:** A BST built from sorted input degenerates into a linked list (O(n) operations). Self-balancing trees (red-black, AVL) prevent this — that's why `TreeMap` uses red-black, not plain BST.

**Common pitfalls**
- Inserting sorted data into a plain BST — O(n) depth, O(n) per operation.
- Forgetting the three delete cases (0 children, 1 child, 2 children) — the 2-children case requires in-order successor.
- Confusing in-order (sorted) with pre-order (serialization) — they produce different sequences.
- Recursive traversal on very deep trees — can cause `StackOverflowError`; use iterative with explicit stack.

**Self-check question**
Given a BST with nodes [50, 30, 70, 20, 40, 60, 80], you delete node 50 (root). Which node replaces it, and what does the tree look like after deletion?
