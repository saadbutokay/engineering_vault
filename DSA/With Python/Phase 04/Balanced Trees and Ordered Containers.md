## Overview

The previous note ended with a warning: a plain binary search tree is only fast when its shape is reasonable, and ordinary data such as timestamps and ascending IDs produces the worst shape, a chain. A self-balancing tree solves this by repairing its own shape after every insertion and deletion, so the height stays proportional to `log2(n)` no matter what order the keys arrive in. Search, insertion, and deletion then cost `O(log n)` in the worst case, not just on average.

Several families of structures achieve this. AVL trees keep the tree very strictly balanced using small local rearrangements called rotations. Red-black trees balance more loosely and rearrange less often, and they sit inside many standard libraries. B-trees pack many keys into each node so that a lookup touches only a few nodes, which is what makes them the standard structure for databases and file systems, where each node access is a disk read. Skip lists get balanced behavior from randomness instead of rotations.

In everyday Python you will almost never write any of these. You will use `dict` for lookups, `bisect` on a list for read-mostly sorted data, a database index when data lives in a database, and the third-party `sortedcontainers` package when you need a sorted collection in memory with many updates. This note explains why balance matters, how AVL rotations work, how red-black trees, B-trees, and skip lists differ, how `sortedcontainers` is built and when to choose it, and what an ordered map is used for in practice.

## Core concepts

### Why balance matters

Every tree operation costs about the height of the tree. A balanced tree with a million keys has height around 20. A degenerate one has height a million. A self-balancing tree guarantees the first outcome by checking and repairing balance during every update.

### Balance factor

In an AVL tree, the balance factor of a node is the height of its left subtree minus the height of its right subtree. The AVL rule is that every node's balance factor must be `-1`, `0`, or `1`. When an insertion or deletion makes some node's balance factor `2` or `-2`, the tree is repaired with one or two rotations at that node.

### Rotations

A rotation is a local change to three links that lifts one node above its parent while keeping the inorder order of the keys unchanged. Because the order is unchanged, the BST property survives, and because the shape changes, the height changes.

- Right rotation: the left child moves up and the node becomes its right child. It fixes a left-left imbalance (a long chain down the left).
- Left rotation: the right child moves up and the node becomes its left child. It fixes a right-right imbalance.
- Left-right case: the left child is itself right-heavy. Rotate that child left first, turning it into a left-left case, then rotate the node right.
- Right-left case: the mirror image. Rotate the right child right first, then rotate the node left.

### AVL trees

An AVL tree is a BST that restores the balance rule with rotations after each insertion and deletion, walking back up the path from the changed node and recomputing heights. Its height is at most about `1.44 * log2(n)`. Insertion needs at most one rotation (single or double). Deletion may need a rotation at each level along the path, up to `O(log n)` of them.

### Red-black trees (overview)

A red-black tree colors each node red or black and enforces rules: the root is black, a red node never has a red child, and every path from a node to its empty descendants passes through the same number of black nodes. Together these rules guarantee that the longest path is at most twice the shortest, so the height is at most `2 * log2(n + 1)`. Red-black trees are balanced less strictly than AVL trees, so lookups are slightly slower, but insertions and deletions need fewer rotations (at most two for an insertion and three for a deletion). That trade made them the common choice for library ordered maps: C++ `std::map`, Java `TreeMap`, and the Linux kernel's scheduler use them.

### B-trees

A B-tree is a balanced search tree whose nodes hold many keys, not just one, and have many children. A node with `k` keys has `k + 1` children, and the keys inside a node are kept sorted. To search, you binary search within a node to find which child to descend into. All leaves are at the same depth, and the tree stays balanced by splitting full nodes and merging sparse ones.

Why databases and file systems use them: reading a node means reading a disk page (typically 4 to 16 kilobytes), which costs far more than comparing keys in memory. A B-tree node is sized to fill one page and holds hundreds of keys, so the tree is very shallow. A B-tree with a minimum degree of 100 holds a million keys in three levels, so a lookup reads about three pages. Most database indexes are B-trees or the B+ tree variant, which keeps all the data in the leaves and links the leaves together so range scans are fast.

### Skip lists

A skip list is a sorted linked list with extra "express lanes". Each node is promoted to the next level up with probability one half, so level 1 holds about half of the nodes, level 2 about a quarter, and so on. A search starts at the top lane, moves right while the next key is smaller, drops down a level when it would overshoot, and finishes at the bottom. The expected cost is `O(log n)` for search, insert, and delete, with no rotations. They are simple to implement and easy to make concurrent, which is why Redis uses one for sorted sets.

### sortedcontainers

`sortedcontainers` is a pure-Python library that provides `SortedList`, `SortedDict`, and `SortedSet`. It does not use a tree at all. It keeps a list of sorted lists, each holding about a thousand items, plus a small index of the maximum of each sublist. Finding a position uses binary search on the index and then on one sublist, and inserting shifts items only inside one short list, which is a fast operation in C. For realistic sizes, this beats tree implementations written in Python, and it is what most Python programmers use when they need an ordered container.

### Ordered maps in practice

An ordered map keeps keys sorted and supports, besides lookup, queries a `dict` cannot answer: smallest and largest key, the next key above or below a given one (floor and ceiling), and all keys in a range. They appear as time-ordered event indexes, price ladders in trading systems, interval and scheduling structures, leaderboards, and every database index.

## Basic syntax examples

### AVL rotations and insertion

The code below uses plain integer keys to keep the mechanics visible. Every function that changes a subtree returns the new root of that subtree, so the caller can reattach it.

```python
from dataclasses import dataclass


@dataclass(slots=True)
class AVLNode:
    key: int
    height: int = 1  # counted in nodes: a single node has height 1
    left: "AVLNode | None" = None
    right: "AVLNode | None" = None


def node_height(node: AVLNode | None) -> int:
    return node.height if node is not None else 0


def update_height(node: AVLNode) -> None:
    node.height = 1 + max(node_height(node.left), node_height(node.right))


def balance_factor(node: AVLNode) -> int:
    return node_height(node.left) - node_height(node.right)


def rotate_right(y: AVLNode) -> AVLNode:
    """The left child x moves up. y becomes x's right child."""
    x = y.left
    assert x is not None
    y.left = x.right  # x's right subtree lies between x and y, so y adopts it
    x.right = y
    update_height(y)  # y is lower now, so update it first
    update_height(x)
    return x


def rotate_left(x: AVLNode) -> AVLNode:
    """The right child y moves up. x becomes y's left child."""
    y = x.right
    assert y is not None
    x.right = y.left
    y.left = x
    update_height(x)
    update_height(y)
    return y


def rebalance(node: AVLNode) -> AVLNode:
    update_height(node)
    balance = balance_factor(node)
    if balance > 1:  # left side is too tall
        assert node.left is not None
        if balance_factor(node.left) < 0:  # left-right case
            node.left = rotate_left(node.left)
        return rotate_right(node)
    if balance < -1:  # right side is too tall
        assert node.right is not None
        if balance_factor(node.right) > 0:  # right-left case
            node.right = rotate_right(node.right)
        return rotate_left(node)
    return node


def insert(node: AVLNode | None, key: int) -> AVLNode:
    if node is None:
        return AVLNode(key)
    if key < node.key:
        node.left = insert(node.left, key)
    elif key > node.key:
        node.right = insert(node.right, key)
    else:
        return node  # duplicates are ignored
    return rebalance(node)  # repair on the way back up


def inorder(node: AVLNode | None) -> list[int]:
    if node is None:
        return []
    return inorder(node.left) + [node.key] + inorder(node.right)


root: AVLNode | None = None
for key in range(1, 8):  # sorted input, the worst case for a plain BST
    root = insert(root, key)

assert root is not None
print(inorder(root))  # [1, 2, 3, 4, 5, 6, 7]
print(root.key, root.height)  # 4 3
```

Inserting `1, 2, 3` into an empty tree creates a right-right chain and triggers a left rotation:

```text
insert 1, 2, 3                         after rotate_left at 1

    1   balance factor -2                   2
     \                                    /   \
      2                                  1     3
       \
        3
```

The other three cases:

```text
Left-left (insert 3, 2, 1)           Right-left (insert 1, 3, 2)
    3                                    1
   /                                      \
  2     rotate right at 3                  3     rotate right at 3, then left at 1
 /      gives root 2                      /      gives root 2
1                                        2

Left-right (insert 3, 1, 2)          Right-right (insert 1, 2, 3)
    3                                    1
   /                                      \
  1     rotate left at 1, then            2     rotate left at 1
   \    right at 3 gives root 2            \    gives root 2
    2                                       3
```

Sorted insertion of 7 keys gave a tree of height 3, and without balancing it would have been a chain of height 7.

### Height after many sorted insertions

```python
big_root: AVLNode | None = None
for key in range(1, 1_001):
    big_root = insert(big_root, key)

assert big_root is not None
print(big_root.height)  # 10
```

A plain BST fed the same input has 1,000 levels. The AVL tree has 10, which is `ceil(log2(1001))`.

### A skip list

```python
import random
from collections.abc import Iterator
from typing import Any, Protocol, cast


class SupportsLessThan(Protocol):
    def __lt__(self, other: Any, /) -> bool: ...


class _SkipNode[K]:
    __slots__ = ("key", "forward")

    def __init__(self, key: K, level: int) -> None:
        self.key = key
        self.forward: list[_SkipNode[K] | None] = [None] * level


class SkipList[K: SupportsLessThan]:
    """A sorted set. Search, add, and remove take expected O(log n) time."""

    MAX_LEVEL = 16

    def __init__(self, rng: random.Random | None = None) -> None:
        self._rng = rng or random.Random()
        self._head: _SkipNode[K] = _SkipNode(cast(K, None), self.MAX_LEVEL)
        self._level = 1  # how many lanes are currently in use
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def _random_level(self) -> int:
        level = 1
        while level < self.MAX_LEVEL and self._rng.random() < 0.5:
            level += 1  # promote with probability 1/2
        return level

    def _search(self, key: K) -> tuple[list[_SkipNode[K]], _SkipNode[K]]:
        """Return the last node before `key` on every lane, and on the bottom lane."""
        update: list[_SkipNode[K]] = [self._head] * self.MAX_LEVEL
        node = self._head
        for level in range(self._level - 1, -1, -1):  # start at the top lane
            following = node.forward[level]
            while following is not None and following.key < key:
                node = following  # move right while the next key is still smaller
                following = node.forward[level]
            update[level] = node  # then drop down a lane
        return update, node

    def __contains__(self, key: K) -> bool:
        _, node = self._search(key)
        candidate = node.forward[0]
        return candidate is not None and not key < candidate.key

    def add(self, key: K) -> bool:
        update, node = self._search(key)
        existing = node.forward[0]
        if existing is not None and not key < existing.key:
            return False  # already present
        level = self._random_level()
        self._level = max(self._level, level)
        new = _SkipNode(key, level)
        for lane in range(level):
            new.forward[lane] = update[lane].forward[lane]
            update[lane].forward[lane] = new
        self._size += 1
        return True

    def remove(self, key: K) -> bool:
        update, node = self._search(key)
        target = node.forward[0]
        if target is None or key < target.key:
            return False
        for lane in range(self._level):
            if update[lane].forward[lane] is not target:
                break  # the node does not reach this lane or any above it
            update[lane].forward[lane] = target.forward[lane]
        while self._level > 1 and self._head.forward[self._level - 1] is None:
            self._level -= 1  # drop empty top lanes
        self._size -= 1
        return True

    def __iter__(self) -> Iterator[K]:
        node = self._head.forward[0]
        while node is not None:
            yield node.key
            node = node.forward[0]


skip = SkipList[int](random.Random(1))
for value in (30, 10, 50, 20, 40, 10):
    skip.add(value)
print(list(skip), len(skip))  # [10, 20, 30, 40, 50] 5
print(20 in skip, 25 in skip)  # True False
skip.remove(30)
print(list(skip))  # [10, 20, 40, 50]
```


### Searching a B-tree node by node

A real B-tree handles insertion by splitting full nodes, which takes a few dozen lines. Searching is short and shows why the structure is shallow: each step does a binary search inside one node and then follows one child.

```python
from bisect import bisect_left
from dataclasses import dataclass, field


@dataclass(slots=True)
class BTreeNode:
    keys: list[int]
    children: list["BTreeNode"] = field(default_factory=list)  # empty for a leaf


def btree_search(root: BTreeNode, key: int) -> bool:
    node = root
    while True:
        index = bisect_left(node.keys, key)  # binary search inside the node
        if index < len(node.keys) and node.keys[index] == key:
            return True
        if not node.children:  # a leaf: nowhere left to look
            return False
        node = node.children[index]  # keys before position `index` are smaller


#            [10, 20]
#           /    |    \
#     [3, 5]  [12, 15]  [25, 30]
btree = BTreeNode(
    [10, 20],
    [BTreeNode([3, 5]), BTreeNode([12, 15]), BTreeNode([25, 30])],
)
print(btree_search(btree, 15), btree_search(btree, 16))  # True False
```

How tall can a B-tree be? With minimum degree `t` (every node except the root has at least `t - 1` keys), the height is at most `log_t((n + 1) / 2)`.

```python
import math


def btree_height_bound(keys: int, min_degree: int) -> int:
    """Upper bound on the number of levels below the root."""
    return math.ceil(math.log((keys + 1) / 2, min_degree))


print(btree_height_bound(1_000_000, 2))  # 19, a small fanout is like a binary tree
print(btree_height_bound(1_000_000, 100))  # 3
print(btree_height_bound(1_000_000_000, 500))  # 4
```

A billion keys fit in a tree about four levels deep, so a database can find any row after reading about four pages, and the top levels are usually cached in memory.

### sortedcontainers

```python
from sortedcontainers import SortedDict, SortedList

latencies = SortedList([30, 10, 20])
latencies.add(15)
print(list(latencies))  # [10, 15, 20, 30]
print(latencies[0], latencies[-1])  # 10 30
print(latencies.bisect_left(20))  # 2
print(list(latencies.irange(12, 25)))  # [15, 20]
latencies.remove(15)
print(latencies.pop(0))  # 10

prices = SortedDict({100: "low", 300: "high", 200: "mid"})
print(prices.peekitem(0), prices.peekitem(-1))  # (100, 'low') (300, 'high')
print(list(prices.irange(150, 300)))  # [200, 300]
index = prices.bisect_right(250)
print(prices.peekitem(index - 1))  # (200, 'mid'), the floor of 250
```

If your type checker cannot find types for the package, add an override to `pyproject.toml`:

```toml
[[tool.mypy.overrides]]
module = "sortedcontainers"
ignore_missing_imports = true
```

### A sorted list with bisect

For data that is read much more often than it changes, the standard library is enough.

```python
from bisect import bisect_left, bisect_right, insort

prices_sorted = [100, 200, 300]
insort(prices_sorted, 250)  # O(log n) to find the place, O(n) to shift
print(prices_sorted)  # [100, 200, 250, 300]

position = bisect_right(prices_sorted, 260)
print(prices_sorted[position - 1])  # 250, the floor of 260
position = bisect_left(prices_sorted, 260)
print(prices_sorted[position])  # 300, the ceiling of 260
```

## Deeper behavior

### What a rotation preserves

A rotation only changes which node is the parent. The inorder sequence is identical before and after, so every key is still between the same neighbors and the BST property holds. After `rotate_right(y)`, the keys in order are still `x.left`, `x`, `x.right` (now `y.left`), `y`, `y.right`. This is why the repair is always safe, and why rebalancing needs no knowledge of the keys, only of heights.

### Why AVL insertion needs at most one rotation

After inserting, walk up the path recomputing heights. The first node (lowest on the path) whose balance factor becomes `2` or `-2` is the only place that needs repair. A single or double rotation there restores that subtree to the height it had before the insertion, so every ancestor above it is balanced again and no further rotations are needed. Deletion is different: a rotation can shorten the subtree, which may unbalance an ancestor, so repairs can cascade up to the root.

### The height guarantee

The most unbalanced AVL tree of a given height has as few nodes as possible when its subtrees differ by one at every level, which gives the Fibonacci recurrence `N(h) = N(h - 1) + N(h - 2) + 1`. Solving it gives a height of at most about `1.44 * log2(n + 2)`. That bound is a worst case. In practice, AVL trees built from random keys have a height close to `log2(n) + 0.25`.

### AVL versus red-black

- AVL trees are more strictly balanced, so lookups are somewhat faster. Updates do more rebalancing work.
- Red-black trees allow a height up to `2 * log2(n)`, so lookups can be somewhat slower, but updates do less work and the rebalancing logic changes fewer pointers.
- Both give `O(log n)` for every operation in the worst case. For a read-heavy workload, AVL trees tend to win. For an update-heavy workload, red-black trees tend to win. The difference is a small constant factor.

Other balanced designs: treaps give each node a random priority and keep the tree a heap on those priorities, which makes a random shape with expected `O(log n)` height; splay trees move each accessed node to the root, which gives amortized `O(log n)` and fast repeated access to the same keys; scapegoat trees rebuild a subtree when it gets too lopsided (the guard used in the previous note's snippet is a crude form of this idea).

### B-tree details

- Node capacity: with minimum degree `t`, every node except the root holds between `t - 1` and `2t - 1` keys.
- Insertion: find the leaf and insert the key. If the leaf is full, split it into two nodes around its median key and push the median up into the parent, which may split in turn. A new root is created only when the old root splits, which is the only way the tree gets taller, so all leaves stay at the same depth.
- Deletion: remove the key, and if a node becomes too small, borrow a key from a sibling or merge with it.
- Cost: every operation reads and writes `O(log_t n)` nodes. When nodes are disk pages, that is the number of I/O operations, which is what matters.
- B+ tree: internal nodes hold only keys for routing, and all records live in the leaves, which are linked in order. A range query finds the first key in `O(log n)` and then follows the leaf links with sequential reads. This is the structure behind most relational database indexes.

### Why indexes make reads fast and writes slower

An index is a second B-tree keyed by the indexed column that points back to the rows. A query that filters on that column walks the tree instead of scanning the table. Every insert, update, or delete of an indexed value must also update each index, so a table with many indexes writes more slowly. The decision of which columns to index is a trade between read speed and write cost.

### Skip lists versus trees

Skip lists need no rotations and are easier to write correctly, and their updates touch only a few nearby pointers, which makes lock-free and fine-grained-locking versions practical. They use more memory than a tree (an average of two pointers per node) and give only expected, not worst-case, `O(log n)`, and a sequential walk visits scattered nodes. The `rng` parameter above makes the structure reproducible for tests.

### How sortedcontainers stays fast

The key design choice is to avoid pointers. `SortedList` stores items in a list of lists with a "load factor" of about 1,000 items per sublist.

- `add`: binary search on the list of sublist maximums to find the right sublist, then `bisect.insort` into that sublist. Inserting into a list of 1,000 items shifts memory in C and takes a few microseconds. If a sublist grows beyond twice the load factor, it is split in two.
- `remove`: the same search, then delete from one sublist, merging small neighbors.
- Index access, such as `latencies[500]`, uses an auxiliary positional index, a tree over the sublist sizes, so it is also `O(log n)`.
- The operations are described as `O(log n)` in practice: the sublist length is bounded by a constant, so the shifting cost is bounded, and it is cache friendly and runs mostly in the C code of the interpreter.

The result is usually faster than a hand-written tree in Python, because a tree spends its time in interpreted pointer-following code. Its trade-offs: it is a third-party dependency, it has no per-node access, and keys must be orderable and must not change while stored. In the capstone projects of this roadmap, third-party libraries are not allowed, so you implement the structure yourself there.

### Choosing in practice

- Lookup by key only: `dict`.
- Sorted data that is built once or changed rarely: a list with `bisect`, or `sorted()`.
- Ordered updates and queries in memory, such as floor, ceiling, and ranges: `sortedcontainers`.
- Data in a database: let the database index (a B-tree) do the work.
- Learning, interviews, or an environment where you cannot add packages: write an AVL or a skip list.
- An ordered structure shared between threads: protect it with a lock, or use a database or a purpose-built store.

## Industry-level snippet

A meeting-room service keeps bookings for one room as non-overlapping time ranges. The core operation is the check "does the requested range clash with an existing booking?" which needs the booking that starts at or before the request and the one that starts right after. In note 2.7 this was done with a sorted list and `bisect`, where finding the neighbors was fast but inserting a booking shifted the list. With thousands of bookings and frequent changes, a balanced tree gives `O(log n)` for finding the neighbors and for the insertion.

The code is built from an AVL tree with `floor_item`, `ceiling_item`, and a bounded range iterator, which are the ordered-map queries a `dict` cannot answer. The booking index keeps the database as the source of truth: it writes the booking in a transaction first and updates the in-memory tree after the commit, so a failed write cannot leave the two out of step. The tree has an `is_valid` method that checks the BST order, the stored heights, and the balance rule, which the tests call after random sequences of insertions and deletions.

```python
# avl_tree.py
from collections.abc import Iterator
from dataclasses import dataclass
from typing import Any, Protocol


class SupportsLessThan(Protocol):
    def __lt__(self, other: Any, /) -> bool: ...


@dataclass(slots=True)
class _Node[K, V]:
    key: K
    value: V
    height: int = 1
    left: "_Node[K, V] | None" = None
    right: "_Node[K, V] | None" = None


def _height[K, V](node: _Node[K, V] | None) -> int:
    return node.height if node is not None else 0


def _update[K, V](node: _Node[K, V]) -> None:
    node.height = 1 + max(_height(node.left), _height(node.right))


def _balance[K, V](node: _Node[K, V]) -> int:
    return _height(node.left) - _height(node.right)


def _rotate_right[K, V](y: _Node[K, V]) -> _Node[K, V]:
    x = y.left
    assert x is not None
    y.left = x.right
    x.right = y
    _update(y)
    _update(x)
    return x


def _rotate_left[K, V](x: _Node[K, V]) -> _Node[K, V]:
    y = x.right
    assert y is not None
    x.right = y.left
    y.left = x
    _update(x)
    _update(y)
    return y


def _rebalance[K, V](node: _Node[K, V]) -> _Node[K, V]:
    _update(node)
    balance = _balance(node)
    if balance > 1:
        assert node.left is not None
        if _balance(node.left) < 0:
            node.left = _rotate_left(node.left)
        return _rotate_right(node)
    if balance < -1:
        assert node.right is not None
        if _balance(node.right) > 0:
            node.right = _rotate_right(node.right)
        return _rotate_left(node)
    return node


class AVLTree[K: SupportsLessThan, V]:
    """An ordered map with O(log n) operations in the worst case."""

    def __init__(self) -> None:
        self._root: _Node[K, V] | None = None
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def height(self) -> int:
        """Edges on the longest path. An empty tree has height -1."""
        return _height(self._root) - 1

    def get(self, key: K) -> V | None:
        node = self._root
        while node is not None:
            if key < node.key:
                node = node.left
            elif node.key < key:
                node = node.right
            else:
                return node.value
        return None

    def put(self, key: K, value: V) -> None:
        self._root, added = self._insert(self._root, key, value)
        if added:
            self._size += 1

    def _insert(
        self, node: _Node[K, V] | None, key: K, value: V
    ) -> tuple[_Node[K, V], bool]:
        if node is None:
            return _Node(key, value), True
        if key < node.key:
            node.left, added = self._insert(node.left, key, value)
        elif node.key < key:
            node.right, added = self._insert(node.right, key, value)
        else:
            node.value = value
            return node, False
        return _rebalance(node), added

    def delete(self, key: K) -> bool:
        self._root, removed = self._delete(self._root, key)
        if removed:
            self._size -= 1
        return removed

    def _delete(
        self, node: _Node[K, V] | None, key: K
    ) -> tuple[_Node[K, V] | None, bool]:
        if node is None:
            return None, False
        if key < node.key:
            node.left, removed = self._delete(node.left, key)
        elif node.key < key:
            node.right, removed = self._delete(node.right, key)
        else:
            if node.left is None:
                return node.right, True
            if node.right is None:
                return node.left, True
            successor = node.right  # two children: copy the successor, delete it below
            while successor.left is not None:
                successor = successor.left
            node.key, node.value = successor.key, successor.value
            node.right, _ = self._delete(node.right, successor.key)
            removed = True
        return _rebalance(node), removed

    def floor_item(self, key: K) -> tuple[K, V] | None:
        """The item with the largest key that is less than or equal to `key`."""
        best: _Node[K, V] | None = None
        node = self._root
        while node is not None:
            if key < node.key:
                node = node.left
            else:
                best = node
                node = node.right
        return (best.key, best.value) if best is not None else None

    def ceiling_item(self, key: K) -> tuple[K, V] | None:
        """The item with the smallest key that is greater than or equal to `key`."""
        best: _Node[K, V] | None = None
        node = self._root
        while node is not None:
            if node.key < key:
                node = node.right
            else:
                best = node
                node = node.left
        return (best.key, best.value) if best is not None else None

    def range_items(self, low: K, high: K) -> Iterator[tuple[K, V]]:
        """Items with low <= key < high in order, skipping out-of-range subtrees."""
        stack: list[_Node[K, V]] = []
        node = self._root
        while node is not None or stack:
            while node is not None:
                if node.key < low:
                    node = node.right  # this node and its left subtree are too small
                else:
                    stack.append(node)
                    node = node.left
            if not stack:
                return
            node = stack.pop()
            if not node.key < high:
                return  # everything from here on is past the end of the range
            yield node.key, node.value
            node = node.right

    def is_valid(self) -> bool:
        """Check the BST order, the stored heights, and the AVL balance rule."""

        def check(
            node: _Node[K, V] | None, low: K | None, high: K | None
        ) -> tuple[bool, int]:
            if node is None:
                return True, 0
            if low is not None and not low < node.key:
                return False, 0
            if high is not None and not node.key < high:
                return False, 0
            left_ok, left_height = check(node.left, low, node.key)
            right_ok, right_height = check(node.right, node.key, high)
            height = 1 + max(left_height, right_height)
            balanced = abs(left_height - right_height) <= 1
            return left_ok and right_ok and balanced and node.height == height, height

        return check(self._root, None, None)[0]
```

```python
# booking_index.py
import logging
import sqlite3
from dataclasses import dataclass

from avl_tree import AVLTree

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS booking (
    start_time INTEGER PRIMARY KEY,
    end_time INTEGER NOT NULL,
    label TEXT NOT NULL,
    CHECK (start_time < end_time)
)
"""


INSERT_SQL = "INSERT INTO booking (start_time, end_time, label) VALUES (?, ?, ?)"


class InvalidBookingError(ValueError):
    """Raised for an empty or reversed time range, or a missing label."""


class BookingConflictError(Exception):
    """Raised when a booking overlaps an existing one."""


@dataclass(frozen=True, slots=True)
class Booking:
    start: int
    end: int
    label: str


class BookingIndex:
    """Non-overlapping bookings for one room, indexed by start time.

    Half-open ranges [start, end): a booking that ends at 10 does not block one
    that starts at 10. Assumes one writer process for the database.
    """

    def __init__(self, connection: sqlite3.Connection) -> None:
        self._connection = connection
        self._tree: AVLTree[int, tuple[int, str]] = AVLTree()
        rows = connection.execute("SELECT start_time, end_time, label FROM booking")
        for start, end, label in rows:
            self._tree.put(start, (end, label))

    def __len__(self) -> int:
        return len(self._tree)

    def book(self, start: int, end: int, label: str) -> Booking:
        if start >= end:
            raise InvalidBookingError("start must be before end")
        if not label:
            raise InvalidBookingError("label must not be empty")
        before = self._tree.floor_item(start)  # starts at or before the request
        if before is not None and before[1][0] > start:
            raise BookingConflictError(f"overlaps the booking starting at {before[0]}")
        after = self._tree.ceiling_item(start)  # the next booking to start
        if after is not None and after[0] < end:
            raise BookingConflictError(f"overlaps the booking starting at {after[0]}")
        try:
            with self._connection:
                self._connection.execute(INSERT_SQL, (start, end, label))
        except sqlite3.Error:
            logger.exception("could not store the booking at %d", start)
            raise
        self._tree.put(start, (end, label))  # only after the commit succeeded
        return Booking(start, end, label)

    def cancel(self, start: int) -> bool:
        try:
            with self._connection:
                cursor = self._connection.execute(
                    "DELETE FROM booking WHERE start_time = ?", (start,)
                )
        except sqlite3.Error:
            logger.exception("could not cancel the booking at %d", start)
            raise
        if cursor.rowcount == 0:
            return False
        self._tree.delete(start)
        return True

    def overlapping(self, first: int, last: int) -> list[Booking]:
        """Bookings that overlap the half-open range [first, last). Integer times."""
        if first >= last:
            raise InvalidBookingError("first must be before last")
        found: list[Booking] = []
        earlier = self._tree.floor_item(first - 1)  # one that began before `first`
        if earlier is not None and earlier[1][0] > first:
            found.append(Booking(earlier[0], earlier[1][0], earlier[1][1]))
        for start, (end, label) in self._tree.range_items(first, last):
            found.append(Booking(start, end, label))
        return found

    def height(self) -> int:
        return self._tree.height()
```

```python
# test_booking_index.py
import math
import random
import sqlite3
from collections.abc import Iterator

import pytest

from avl_tree import AVLTree
from booking_index import (
    SCHEMA_SQL,
    BookingConflictError,
    BookingIndex,
    InvalidBookingError,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA_SQL)
    yield conn
    conn.close()


def test_avl_tree_stays_balanced_for_sorted_insertions() -> None:
    tree = AVLTree[int, int]()
    for key in range(1, 2_001):
        tree.put(key, key)
    assert tree.is_valid()
    assert tree.height() <= 1.45 * math.log2(len(tree) + 2)


def test_avl_tree_matches_a_dictionary_on_random_operations() -> None:
    rng = random.Random(17)
    tree = AVLTree[int, int]()
    model: dict[int, int] = {}
    for step in range(3_000):
        key = rng.randint(0, 200)
        if rng.random() < 0.4:
            assert tree.delete(key) is (key in model)
            model.pop(key, None)
        else:
            tree.put(key, step)
            model[key] = step
        if step % 50 == 0:
            assert tree.is_valid()
        floors = [k for k in model if k <= key]
        ceilings = [k for k in model if k >= key]
        floor = tree.floor_item(key)
        ceiling = tree.ceiling_item(key)
        assert (floor[0] if floor else None) == (max(floors) if floors else None)
        expected_ceiling = min(ceilings) if ceilings else None
        assert (ceiling[0] if ceiling else None) == expected_ceiling
    assert len(tree) == len(model)
    assert [k for k, _ in tree.range_items(50, 150)] == sorted(
        k for k in model if 50 <= k < 150
    )


def test_bookings_may_touch_but_not_overlap(connection: sqlite3.Connection) -> None:
    index = BookingIndex(connection)
    index.book(100, 200, "standup")
    index.book(200, 300, "review")  # starts exactly when the first ends
    index.book(50, 100, "setup")  # ends exactly when the first starts
    for start, end in ((150, 250), (90, 110), (120, 180), (0, 1_000), (299, 301)):
        with pytest.raises(BookingConflictError):
            index.book(start, end, "clash")
    assert len(index) == 3


def test_invalid_bookings_are_rejected(connection: sqlite3.Connection) -> None:
    index = BookingIndex(connection)
    with pytest.raises(InvalidBookingError):
        index.book(10, 10, "empty")
    with pytest.raises(InvalidBookingError):
        index.book(10, 20, "")


def test_cancel_frees_the_slot(connection: sqlite3.Connection) -> None:
    index = BookingIndex(connection)
    index.book(100, 200, "a")
    assert index.cancel(100) is True
    assert index.cancel(100) is False
    index.book(150, 250, "b")
    assert len(index) == 1


def test_the_index_reloads_from_the_database(connection: sqlite3.Connection) -> None:
    BookingIndex(connection).book(100, 200, "a")
    reloaded = BookingIndex(connection)
    assert len(reloaded) == 1
    with pytest.raises(BookingConflictError):
        reloaded.book(150, 160, "b")


def test_overlapping_query(connection: sqlite3.Connection) -> None:
    index = BookingIndex(connection)
    for start, end in ((0, 10), (10, 20), (30, 40), (50, 60)):
        index.book(start, end, f"b{start}")
    assert [b.start for b in index.overlapping(5, 35)] == [0, 10, 30]
    assert [b.start for b in index.overlapping(20, 30)] == []
    assert [b.start for b in index.overlapping(15, 16)] == [10]


def test_random_requests_match_a_brute_force_check(
    connection: sqlite3.Connection,
) -> None:
    rng = random.Random(23)
    index = BookingIndex(connection)
    accepted: list[tuple[int, int]] = []
    for _ in range(400):
        start = rng.randint(0, 500)
        end = start + rng.randint(1, 15)
        clashes = any(s < end and start < e for s, e in accepted)
        if clashes:
            with pytest.raises(BookingConflictError):
                index.book(start, end, "x")
        else:
            index.book(start, end, "x")
            accepted.append((start, end))
    assert len(index) == len(accepted)
    window = (100, 300)
    expected = sorted(a for a in accepted if a[0] < window[1] and window[0] < a[1])
    found = [(b.start, b.end) for b in index.overlapping(*window)]
    assert found == expected


def test_many_ordered_bookings_keep_the_tree_shallow(
    connection: sqlite3.Connection,
) -> None:
    index = BookingIndex(connection)
    for number in range(1_000):
        index.book(number * 10, number * 10 + 5, "slot")
    assert index.height() <= 1.45 * math.log2(len(index) + 2)


def test_failed_write_leaves_the_index_unchanged(
    connection: sqlite3.Connection,
) -> None:
    index = BookingIndex(connection)
    index.book(100, 200, "a")
    connection.execute("DROP TABLE booking")
    with pytest.raises(sqlite3.OperationalError):
        index.book(300, 400, "b")
    assert len(index) == 1
```

The test with 1,000 bookings in increasing order is the same worst case that would turn a plain BST into a chain. Here the height stays below `1.45 * log2(n + 2)`, about 14 levels.

## Common mistakes

- Using a plain BST for ordered or time-based keys and getting a chain. Use a balanced structure, or a sorted list with `bisect` for read-mostly data.
- Updating heights in the wrong order after a rotation. The lower node must be updated first, then the new subtree root.
- Forgetting to reattach the new subtree root after a rotation or rebalance, so the rotation is lost.
- Applying only a single rotation in a left-right or right-left case. The child must be rotated first.
- Rebalancing only on the way down instead of on the way back up, when the heights are known.
- Assuming AVL deletion needs only one rotation. It can cascade to the root.
- Rolling your own balanced tree when `sortedcontainers`, `bisect`, or a database index would do.
- Choosing a skip list or B-tree implementation without understanding its probabilistic or page-based assumptions.
- Using `bisect.insort` on a huge list inside a hot loop and not noticing the `O(n)` shift.
- Mutating a key that is stored in an ordered structure, which silently breaks the ordering.
- Updating the in-memory index before the database commit, so a failed write leaves them out of step.
- Not testing the structure's invariants after random operations. An `is_valid` method catches rotation bugs quickly.

## Practice exercises

1. Insert `10, 20, 30, 40, 50, 25` into an empty AVL tree by hand, drawing the tree and naming the rotation after each insertion that needs one.
2. Delete the keys `10`, then `40`, from the tree you built, showing the heights and any rotations.
3. Write `is_balanced_avl(root)` for the plain `AVLNode` and test it against trees built by `insert`.
4. Write `build_balanced_bst(sorted_values)` that builds a height-balanced tree from a sorted list, and compare its height with the AVL tree built from the same keys.
5. Add `delete` to the plain `AVLNode` code of the basic section and check it with `is_balanced_avl` after random deletions.
6. Add `__contains__`, `__iter__`, and `min_item` and `max_item` to `AVLTree`, then test them against a sorted list.
7. Extend `AVLTree` to store subtree sizes and add `rank(key)` and `select(k)` in `O(log n)`.
8. Write a test that inserts 100,000 sorted keys into `AVLTree` and measures the height and the time, then does the same for the unbalanced `BST` from the previous note with 5,000 keys. Compare.
9. Implement `SkipList.floor` and `SkipList.ceiling`, and write a model-based test with a sorted list.
10. Measure how many nodes your `SkipList` has on each lane for 10,000 random inserts, and compare with the expected halving.
11. Write `SortedMultiset` using `sortedcontainers.SortedList` with `add`, `remove`, `count`, and `range_count(low, high)` using `bisect_left` and `bisect_right`.
12. Implement B-tree insertion for minimum degree 2 (a 2-3-4 tree): search for the leaf, split full nodes on the way down, and test that all leaves stay at the same depth.
13. Compare `SortedList.add`, `bisect.insort` on a list, and your `AVLTree.put` for 100,000 random keys, and explain the differences.
14. Extend `BookingIndex` with `next_free_slot(duration, after)` that returns the earliest start time at or after `after` where a booking of the given duration fits, using `floor_item`, `ceiling_item`, and `range_items`.

## Summary

A self-balancing tree repairs its shape during updates, so the height stays near `log2(n)` and every operation is `O(log n)` in the worst case, whatever the insertion order. AVL trees track a balance factor at each node and restore the `-1, 0, 1` rule with single or double rotations, which change links but preserve the inorder order. Red-black trees balance more loosely with fewer rotations and appear in many standard libraries. B-trees hold many sorted keys per node, which keeps them extremely shallow and matches the page size of disks, so databases and file systems index with them. Skip lists use random promotion instead of rotations, and `sortedcontainers` uses a list of sorted sublists, which is usually the fastest option for an in-memory ordered container in Python. Reach for `dict`, `bisect`, `sortedcontainers`, or a database index first, and write your own balanced tree to learn, to interview, or when you cannot add dependencies. Always test the invariants after random sequences of updates.
