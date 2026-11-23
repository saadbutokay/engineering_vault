## Overview

A binary search tree (BST) is a binary tree with an ordering rule that makes searching fast. For every node, all keys in its left subtree are smaller than the node's key, and all keys in its right subtree are larger. To find a key you start at the root and compare: if the key you want is smaller, go left; if larger, go right; if equal, you have found it. Each comparison discards an entire subtree, just as binary search discards half of a sorted array.

A sorted array gives you the same fast search but makes insertion and deletion slow, because items must shift. A BST keeps the data sorted and supports search, insertion, and deletion that each cost time proportional to the height of the tree. If the tree is reasonably balanced, that height is about `log2(n)`, so all three are `O(log n)`. If the tree is badly shaped, the height can be `n`, and the BST degenerates into a linked list with `O(n)` operations. Handling that danger is the subject of the next note on balanced trees.

This note covers the BST property, search, insertion, deletion with its three cases, finding the successor and predecessor of a key, validating whether a tree is a BST, finding the `k`-th smallest key, how a BST degenerates, inorder iterators, and why Python has no built-in BST. Python programmers rarely need to write one, but understanding the structure is what makes ordered containers, database indexes, and balanced trees make sense.

## Core concepts

### The BST property

For every node `x` with key `k`:

- every key in the left subtree of `x` is less than `k`,
- every key in the right subtree of `x` is greater than `k`.

The rule applies to the whole subtree, not just the immediate children. A tree where each node is merely between its two children is not necessarily a BST. This distinction is the root of a classic bug in validation code.

A consequence: an inorder traversal of a BST visits the keys in sorted order.

### Search

Start at the root. If the key equals the current node's key, stop. If it is smaller, move to the left child; otherwise move to the right child. If you fall off the tree (reach `None`), the key is not present. The cost is the number of nodes on the path, which is at most the height `h`, so search is `O(h)`.

### Insertion

Search for the key. When the search falls off the tree, that empty position is exactly where the new node belongs. Attach it there. If the key is already present, update its value. Insertion is `O(h)`, and a new node always becomes a leaf.

### Deletion

Deleting a key has three cases, depending on how many children the node has.

1. Leaf (no children): remove it by clearing the parent's link.
2. One child: replace the node by its child, so the parent links directly to the grandchild.
3. Two children: you cannot simply remove it. Find its inorder successor (the smallest key in the right subtree), copy the successor's key and value into the node, and then delete the successor from the right subtree. The successor has no left child, so deleting it falls under case 1 or 2.

### Minimum, maximum, successor, and predecessor

- The minimum is found by following left links until you can go no further. The maximum follows right links.
- The successor of a key is the smallest key strictly greater than it. The predecessor is the largest key strictly smaller. Both can be found by walking from the root: when you go left, the current node is a candidate successor; when you go right, it is a candidate predecessor. The last candidate is the answer.

### Validating a BST

A tree is a valid BST if every node's key lies strictly between a lower bound and an upper bound inherited from its ancestors. The root has no bounds. Going left, the node's key becomes the new upper bound. Going right, it becomes the new lower bound. Alternatively, a tree is a valid BST exactly when its inorder traversal is strictly increasing.

### The k-th smallest key

Because inorder traversal gives sorted order, the `k`-th key visited is the `k`-th smallest. Stopping the traversal as soon as you reach it costs `O(h + k)`. If each node also stores the size of its subtree, you can jump straight to the answer in `O(h)`, which is how order statistics trees work.

### Degeneration

The shape of a BST depends on the order in which keys are inserted. Random order tends to give a bushy tree with height around two to three times `log2(n)`. Sorted or nearly sorted order gives a tree where every new key goes to the same side, producing a chain of height `n - 1`. Real data is often nearly sorted, such as timestamps and auto-increment IDs, which makes this the main weakness of a plain BST.

### Inorder iterators

An inorder iterator produces keys one at a time, in sorted order, without building a full list. It keeps a stack of the nodes still to visit: push the path of left children, pop to visit a node, then continue with its right subtree. It uses `O(h)` memory, can be stopped early, and is what a Python generator expresses naturally.

### Why Python has no built-in BST

Python's `dict` covers lookup by key, `list` with `bisect` covers sorted data with occasional changes, and `sorted` covers one-off ordering. A pointer-based tree written in Python is slow because every step follows a reference through an interpreter. When programmers need an ordered collection with frequent inserts and deletes, they use the third-party `sortedcontainers` package, a database index, or a balanced tree designed for the job.

## Basic syntax examples

### A BST with search, insertion, and ordered queries

```python
from collections.abc import Iterator
from dataclasses import dataclass
from itertools import islice
from typing import Any, Protocol


class SupportsLessThan(Protocol):
    def __lt__(self, other: Any, /) -> bool: ...


@dataclass(slots=True)
class _Node[K, V]:
    key: K
    value: V
    left: "_Node[K, V] | None" = None
    right: "_Node[K, V] | None" = None


class BST[K: SupportsLessThan, V]:
    """An unbalanced binary search tree. Operations are O(h) for height h."""

    def __init__(self) -> None:
        self._root: _Node[K, V] | None = None
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def _find(self, key: K) -> _Node[K, V] | None:
        node = self._root
        while node is not None:
            if key < node.key:
                node = node.left
            elif node.key < key:
                node = node.right
            else:  # neither is smaller, so the keys are equal
                return node
        return None

    def __contains__(self, key: K) -> bool:
        return self._find(key) is not None

    def get(self, key: K, default: V | None = None) -> V | None:
        node = self._find(key)
        return node.value if node is not None else default

    def put(self, key: K, value: V) -> None:
        if self._root is None:
            self._root = _Node(key, value)
            self._size = 1
            return
        node = self._root
        while True:
            if key < node.key:
                if node.left is None:
                    node.left = _Node(key, value)  # the empty spot we were looking for
                    break
                node = node.left
            elif node.key < key:
                if node.right is None:
                    node.right = _Node(key, value)
                    break
                node = node.right
            else:
                node.value = value  # the key exists: update, do not grow
                return
        self._size += 1

    def min_key(self) -> K:
        if self._root is None:
            raise KeyError("the tree is empty")
        node = self._root
        while node.left is not None:
            node = node.left
        return node.key

    def max_key(self) -> K:
        if self._root is None:
            raise KeyError("the tree is empty")
        node = self._root
        while node.right is not None:
            node = node.right
        return node.key

    def successor(self, key: K) -> K | None:
        """Smallest key strictly greater than `key`, or None."""
        candidate: K | None = None
        node = self._root
        while node is not None:
            if key < node.key:
                candidate = node.key  # a possible answer: look for something closer
                node = node.left
            else:
                node = node.right
        return candidate

    def predecessor(self, key: K) -> K | None:
        """Largest key strictly smaller than `key`, or None."""
        candidate: K | None = None
        node = self._root
        while node is not None:
            if node.key < key:
                candidate = node.key
                node = node.right
            else:
                node = node.left
        return candidate

    def items(self) -> Iterator[tuple[K, V]]:
        """Inorder iterator with an explicit stack: sorted order, O(h) memory."""
        stack: list[_Node[K, V]] = []
        node = self._root
        while node is not None or stack:
            while node is not None:
                stack.append(node)
                node = node.left
            node = stack.pop()
            yield node.key, node.value
            node = node.right

    def __iter__(self) -> Iterator[K]:
        for key, _ in self.items():
            yield key

    def kth_smallest(self, k: int) -> K:
        """The k-th smallest key, counting from 1. O(h + k)."""
        if not 1 <= k <= self._size:
            raise IndexError("k is out of range")
        return next(islice(self, k - 1, None))

    def height(self) -> int:
        """Edges on the longest path. An empty tree has height -1."""
        best = -1
        stack: list[tuple[_Node[K, V], int]] = []
        if self._root is not None:
            stack.append((self._root, 0))
        while stack:
            node, depth = stack.pop()
            best = max(best, depth)
            if node.left is not None:
                stack.append((node.left, depth + 1))
            if node.right is not None:
                stack.append((node.right, depth + 1))
        return best


tree = BST[int, str]()
for key in (50, 30, 70, 20, 40, 60, 80):
    tree.put(key, f"v{key}")

print(list(tree))  # [20, 30, 40, 50, 60, 70, 80]
print(tree.get(60), tree.get(65))  # v60 None
print(tree.min_key(), tree.max_key())  # 20 80
print(tree.successor(50), tree.predecessor(50))  # 60 40
print(tree.successor(80), tree.successor(65))  # None 70
print(tree.kth_smallest(3), len(tree), tree.height())  # 40 7 2
```

The tree built by those insertions:

```text
          50
        /    \
      30      70
     /  \    /  \
   20   40  60   80
```

Trace of `get(60)`: compare with 50 (60 is larger, go right), compare with 70 (60 is smaller, go left), compare with 60, found. Searching for 65 goes 50, 70, 60, then right of 60, which is empty, so the key is absent.

### Deletion with its three cases

```python
def delete[K: SupportsLessThan, V](tree: BST[K, V], key: K) -> bool:
    """Remove a key. Returns False if it was not present."""
    parent: _Node[K, V] | None = None
    node = tree._root
    while node is not None:
        if key < node.key:
            parent, node = node, node.left
        elif node.key < key:
            parent, node = node, node.right
        else:
            break
    if node is None:
        return False
    if node.left is not None and node.right is not None:
        # Case 3: two children. Copy the successor into this node, then remove it.
        successor_parent = node
        successor = node.right
        while successor.left is not None:
            successor_parent, successor = successor, successor.left
        node.key, node.value = successor.key, successor.value
        parent, node = successor_parent, successor  # now delete the successor
    # Cases 1 and 2: the node has at most one child.
    child = node.left if node.left is not None else node.right
    if parent is None:
        tree._root = child
    elif parent.left is node:
        parent.left = child
    else:
        parent.right = child
    tree._size -= 1
    return True


print(delete(tree, 20))  # True, a leaf
print(delete(tree, 30))  # True, one child (40) takes its place
print(delete(tree, 50))  # True, two children: the successor 60 moves up
print(delete(tree, 99))  # False
print(list(tree), tree.height())  # [40, 60, 70, 80] 2
```

How the three deletions change the tree:

```text
delete 20 (leaf)             delete 30 (one child)        delete 50 (two children)
        50                           50                           60
      /    \                       /    \                       /    \
    30      70                   40      70                   40      70
      \    /  \                         /  \                           \
      40  60   80                      60   80                          80

For 50: its successor is 60, the smallest key in the right subtree.
The key 60 is copied into the root, then the old node for 60 (a leaf) is removed.
```

### Validating a BST

```python
@dataclass(slots=True)
class TreeNode:
    value: int
    left: "TreeNode | None" = None
    right: "TreeNode | None" = None


def is_valid_bst(root: TreeNode | None) -> bool:
    """Every value must lie strictly between the bounds inherited from ancestors."""
    stack: list[tuple[TreeNode, int | None, int | None]] = []
    if root is not None:
        stack.append((root, None, None))
    while stack:
        node, low, high = stack.pop()
        if low is not None and node.value <= low:
            return False
        if high is not None and node.value >= high:
            return False
        if node.left is not None:
            stack.append((node.left, low, node.value))  # left: bounded above by node
        if node.right is not None:
            stack.append((node.right, node.value, high))  # right: bounded below
    return True


good = TreeNode(5, TreeNode(3), TreeNode(8))
print(is_valid_bst(good))  # True

# Each node is correct relative to its children, but 6 is in the left subtree of 5.
tricky = TreeNode(5, TreeNode(3, None, TreeNode(6)), TreeNode(8))
print(is_valid_bst(tricky))  # False
```

### A degenerate tree

```python
import random

keys = list(range(1, 1_001))

ordered = BST[int, int]()
for key in keys:
    ordered.put(key, key)  # sorted order: every new key goes to the right

shuffled_keys = keys[:]
random.Random(1).shuffle(shuffled_keys)
shuffled = BST[int, int]()
for key in shuffled_keys:
    shuffled.put(key, key)

print(ordered.height(), shuffled.height())  # 999 19
```

Both trees hold the same 1,000 keys. The tree built from sorted input is a chain of height 999, so a search may take 1,000 comparisons. The tree built from shuffled input has height 19, close to `log2(1000)`, which is about 10.

## Deeper behavior

### Cost is the height

Every BST operation follows one path from the root, so its cost is at most the height `h`. The tree's shape therefore decides everything.

- Best case (perfectly balanced): `h = floor(log2(n))`, so operations are `O(log n)`.
- Typical case (random insertion order): `h` is around two to three times `log2(n)`, still `O(log n)`. The average depth of a node in a randomly built BST is about `1.39 * log2(n)`, the same constant that appears for quicksort, and for the same reason: a random insertion order is equivalent to choosing random pivots.
- Worst case (sorted input): `h = n - 1`, so every operation is `O(n)` and building the tree takes `O(n^2)`.

### Why deletion with two children works

The successor of a node with a right subtree is the leftmost node of that subtree. It is larger than everything in the left subtree and smaller than everything else in the right subtree, so placing its key at the deleted node's position keeps the BST property intact. The successor has no left child (otherwise it would not be leftmost), so removing it is the easy one-child case. Using the predecessor instead (the rightmost node of the left subtree) is equally correct. Always choosing the successor can make a tree lopsided over many deletions, so some implementations alternate or choose randomly.

### Duplicates

A BST needs a policy for equal keys. Common choices are to treat keys as unique and update the value (as above), to store a count in each node, or to send equal keys consistently to one side (for example to the right). The side-based policy breaks the strict property and complicates deletion and successor logic, so storing a count or a list of values in the node is usually cleaner.

### Comparisons and ordering rules

The tree relies on the keys having a consistent total order. The code above uses only `<`, and decides equality by "neither is smaller", which is the standard way ordered containers work. Problems appear with keys whose comparison is not a proper order. `float("nan")` is not smaller than, larger than, or equal to anything, so inserting it corrupts the tree. Mixed types such as `int` and `str` raise `TypeError`. Keys must also not change after insertion, because a changed key is in the wrong place.

### Recursion versus iteration

Recursive versions of search, insert, and delete are short, but their stack depth is the tree height, which is `n` for a degenerate tree. The iterative versions above have no such limit, and they are also faster in Python because they avoid call overhead. This is the practical reason to prefer loops for tree operations that follow a single path.

### Iterator semantics

The `items` generator walks the tree lazily. If you modify the tree while iterating over it, the stack may hold nodes that no longer belong to the tree, so the behavior is undefined. Copy the keys first if you need to modify during iteration, or finish the iteration before you change the tree.

### Order statistics and augmented nodes

Storing a subtree size in each node lets you compute the rank of a key (how many keys are smaller) and select the `k`-th smallest in `O(h)` without walking the nodes in between. You must update the sizes along the path on every insertion and deletion. This technique is the model for many augmented trees, such as interval trees, that attach extra information to nodes to answer more questions.

### BST versus the alternatives

- Need only lookup by key: use a `dict`, which is `O(1)` on average.
- Sorted data that rarely changes: a sorted `list` with `bisect` gives `O(log n)` search and is far faster in practice than a pointer-based tree.
- Ordered data with frequent inserts and deletes, plus queries such as floor, ceiling, and ranges: a balanced tree, `sortedcontainers`, or a database index.
- A hand-written, unbalanced BST: teaching, or a structure where you control the insertion order and can guard against degeneration.

## Industry-level snippet

A billing service prices products by quantity tiers: 1 to 9 units cost 100 cents each, 10 to 49 units cost 90 cents each, 50 or more cost 80 cents each. Each tier is stored as its minimum quantity and its unit price, and finding the right tier for a quantity means finding the largest minimum quantity that is less than or equal to it. That is a floor query, which a BST answers by walking one path.

This is a case where it pays to be honest about the design. If the tiers changed rarely, a sorted list with `bisect` would be simpler and faster in Python, and you should use it. The BST is shown for the case where tiers are edited live and reads and writes interleave, and it comes with the guard that every real BST needs. Tier thresholds are often added in increasing order, which is exactly the input that turns a plain BST into a linked list. So every insertion reports its depth, and when the depth exceeds a bound based on the number of tiers, the table rebuilds itself into a perfectly balanced tree from its sorted contents. The check is cheap, and the rebuild is `O(n)` but rare.

Edits are written to the database in a transaction first, and the in-memory tree changes only after the commit succeeds, so the memory and the database cannot disagree.

```python
# tier_table.py
import logging
import sqlite3
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS rate_tier (
    min_quantity INTEGER PRIMARY KEY CHECK (min_quantity >= 0),
    unit_price_cents INTEGER NOT NULL CHECK (unit_price_cents >= 0)
)
"""

UPSERT_SQL = (
    "INSERT INTO rate_tier (min_quantity, unit_price_cents) VALUES (?, ?) "
    "ON CONFLICT (min_quantity) DO UPDATE "
    "SET unit_price_cents = excluded.unit_price_cents"
)


class TierError(ValueError):
    """Raised for invalid quantities or prices."""


class NoTierError(LookupError):
    """Raised when no tier applies to a quantity."""


@dataclass(slots=True)
class _Node:
    key: int
    price: int
    left: "_Node | None" = None
    right: "_Node | None" = None


def _build_balanced(items: list[tuple[int, int]], low: int, high: int) -> _Node | None:
    """Build a perfectly balanced tree from sorted items. Depth is O(log n)."""
    if low > high:
        return None
    middle = (low + high) // 2
    node = _Node(*items[middle])
    node.left = _build_balanced(items, low, middle - 1)
    node.right = _build_balanced(items, middle + 1, high)
    return node


class TierTable:
    """Quantity tiers with floor lookups and a guard against a degenerate tree."""

    def __init__(self, connection: sqlite3.Connection) -> None:
        self._connection = connection
        rows = connection.execute(
            "SELECT min_quantity, unit_price_cents FROM rate_tier ORDER BY min_quantity"
        ).fetchall()
        items = [(row[0], row[1]) for row in rows]
        self._root = _build_balanced(items, 0, len(items) - 1)
        self._size = len(items)
        self.rebuilds = 0

    def __len__(self) -> int:
        return self._size

    def height_limit(self) -> int:
        """Allow about twice the ideal height before rebuilding."""
        return 2 * (self._size + 1).bit_length() + 2

    def price_for(self, quantity: int) -> int:
        """Unit price of the tier with the largest minimum <= quantity."""
        if quantity < 0:
            raise TierError("quantity must not be negative")
        best: _Node | None = None
        node = self._root
        while node is not None:
            if node.key <= quantity:
                best = node  # a candidate: look for a larger minimum on the right
                node = node.right
            else:
                node = node.left
        if best is None:
            raise NoTierError(f"no tier applies to a quantity of {quantity}")
        return best.price

    def tiers(self) -> list[tuple[int, int]]:
        """All (minimum quantity, unit price) pairs in order, by inorder traversal."""
        result: list[tuple[int, int]] = []
        stack: list[_Node] = []
        node = self._root
        while node is not None or stack:
            while node is not None:
                stack.append(node)
                node = node.left
            node = stack.pop()
            result.append((node.key, node.price))
            node = node.right
        return result

    def height(self) -> int:
        best = -1
        stack: list[tuple[_Node, int]] = []
        if self._root is not None:
            stack.append((self._root, 0))
        while stack:
            node, depth = stack.pop()
            best = max(best, depth)
            for child in (node.left, node.right):
                if child is not None:
                    stack.append((child, depth + 1))
        return best

    def add_tier(self, min_quantity: int, unit_price_cents: int) -> None:
        if min_quantity < 0 or unit_price_cents < 0:
            raise TierError("quantity and price must not be negative")
        try:
            with self._connection:
                self._connection.execute(UPSERT_SQL, (min_quantity, unit_price_cents))
        except sqlite3.Error:
            logger.exception("could not save tier %d", min_quantity)
            raise
        depth, created = self._insert(min_quantity, unit_price_cents)  # after commit
        if created:
            self._size += 1
        if depth > self.height_limit():
            self._rebuild()

    def remove_tier(self, min_quantity: int) -> bool:
        try:
            with self._connection:
                cursor = self._connection.execute(
                    "DELETE FROM rate_tier WHERE min_quantity = ?", (min_quantity,)
                )
        except sqlite3.Error:
            logger.exception("could not remove tier %d", min_quantity)
            raise
        if cursor.rowcount == 0:
            return False
        self._remove(min_quantity)
        return True

    def _insert(self, key: int, price: int) -> tuple[int, bool]:
        """Insert or update. Returns the depth of the node and whether it is new."""
        if self._root is None:
            self._root = _Node(key, price)
            return 0, True
        node = self._root
        depth = 0
        while True:
            if key == node.key:
                node.price = price
                return depth, False
            depth += 1
            if key < node.key:
                if node.left is None:
                    node.left = _Node(key, price)
                    return depth, True
                node = node.left
            else:
                if node.right is None:
                    node.right = _Node(key, price)
                    return depth, True
                node = node.right

    def _remove(self, key: int) -> None:
        parent: _Node | None = None
        node = self._root
        while node is not None and node.key != key:
            parent, node = node, (node.left if key < node.key else node.right)
        if node is None:
            return
        if node.left is not None and node.right is not None:
            successor_parent, successor = node, node.right
            while successor.left is not None:
                successor_parent, successor = successor, successor.left
            node.key, node.price = successor.key, successor.price
            parent, node = successor_parent, successor
        child = node.left if node.left is not None else node.right
        if parent is None:
            self._root = child
        elif parent.left is node:
            parent.left = child
        else:
            parent.right = child
        self._size -= 1

    def _rebuild(self) -> None:
        items = self.tiers()
        self._root = _build_balanced(items, 0, len(items) - 1)
        self.rebuilds += 1
        logger.info("rebuilt the tier tree with %d tiers", len(items))
```

```python
# test_tier_table.py
import random
import sqlite3
from collections.abc import Iterator

import pytest

from tier_table import SCHEMA_SQL, NoTierError, TierError, TierTable


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA_SQL)
    yield conn
    conn.close()


@pytest.fixture
def table(connection: sqlite3.Connection) -> TierTable:
    result = TierTable(connection)
    result.add_tier(10, 90)
    result.add_tier(0, 100)
    result.add_tier(50, 80)
    return result


@pytest.mark.parametrize(
    ("quantity", "expected"),
    [(0, 100), (9, 100), (10, 90), (49, 90), (50, 80), (10_000, 80)],
)
def test_floor_lookup(table: TierTable, quantity: int, expected: int) -> None:
    assert table.price_for(quantity) == expected


def test_quantity_below_the_lowest_tier(connection: sqlite3.Connection) -> None:
    table = TierTable(connection)
    table.add_tier(5, 50)
    with pytest.raises(NoTierError):
        table.price_for(4)
    with pytest.raises(TierError):
        table.price_for(-1)


def test_updating_and_removing_tiers(table: TierTable) -> None:
    table.add_tier(10, 85)
    assert table.price_for(20) == 85
    assert len(table) == 3
    assert table.remove_tier(10) is True
    assert table.remove_tier(10) is False
    assert table.price_for(20) == 100
    assert table.tiers() == [(0, 100), (50, 80)]


def test_the_table_reloads_from_the_database(
    table: TierTable, connection: sqlite3.Connection
) -> None:
    reloaded = TierTable(connection)
    assert reloaded.tiers() == table.tiers()
    assert reloaded.price_for(60) == 80


def test_sorted_insertions_do_not_degenerate(connection: sqlite3.Connection) -> None:
    table = TierTable(connection)
    for quantity in range(1, 1_001):  # the worst case for a plain BST
        table.add_tier(quantity, 1_000 - quantity)
    assert table.rebuilds > 0
    assert table.height() <= table.height_limit()
    assert table.price_for(500) == 500


def test_random_operations_match_a_dictionary_model(
    connection: sqlite3.Connection,
) -> None:
    rng = random.Random(13)
    table = TierTable(connection)
    model: dict[int, int] = {}
    for _ in range(600):
        quantity = rng.randint(0, 60)
        if rng.random() < 0.3 and model:
            victim = rng.choice(sorted(model))
            assert table.remove_tier(victim) is True
            del model[victim]
        else:
            price = rng.randint(1, 500)
            table.add_tier(quantity, price)
            model[quantity] = price
        probe = rng.randint(0, 70)
        floors = [key for key in model if key <= probe]
        if floors:
            assert table.price_for(probe) == model[max(floors)]
        else:
            with pytest.raises(NoTierError):
                table.price_for(probe)
    assert table.tiers() == sorted(model.items())


def test_invalid_tiers_are_rejected(table: TierTable) -> None:
    with pytest.raises(TierError):
        table.add_tier(-1, 10)
    with pytest.raises(TierError):
        table.add_tier(10, -5)


def test_database_failure_leaves_memory_unchanged(
    table: TierTable, connection: sqlite3.Connection
) -> None:
    before = table.tiers()
    connection.execute("DROP TABLE rate_tier")
    with pytest.raises(sqlite3.OperationalError):
        table.add_tier(5, 95)
    assert table.tiers() == before
```

The sorted-insertion test is the important one. Without the rebuild guard, those thousand inserts would produce a chain of 1,000 nodes, and every price lookup would take up to a thousand comparisons.

## Common mistakes

- Validating a BST by checking only that each node is between its two children. Check the bounds inherited from every ancestor.
- Inserting a duplicate key and creating two nodes, which breaks the uniqueness assumption of search and delete.
- Forgetting to reconnect the parent when deleting a node, or forgetting the root case when the deleted node has no parent.
- Handling the two-children deletion by deleting the node and losing one subtree, instead of replacing it with its successor or predecessor.
- Assuming a BST is balanced. Sorted insertions produce a chain.
- Using recursion on a tree that can be deep, and hitting `RecursionError`.
- Using `==` on keys when only `<` is defined, or building a tree whose keys include `NaN` or mixed types.
- Mutating a key after it has been inserted.
- Modifying the tree while an iterator over it is active.
- Updating the in-memory tree before the database change has committed, so a failed write leaves them out of step.
- Writing a BST when a `dict`, a sorted list with `bisect`, or `sortedcontainers` would be simpler and faster.

## Practice exercises

1. Insert the keys `8, 3, 10, 1, 6, 14, 4, 7, 13` into an empty BST by hand and draw the result. Then delete 3, then 8, and draw the tree after each deletion.
2. Write `contains(root, key)`, `insert(root, key)`, and `find_min(root)` recursively for a plain `TreeNode`, then rewrite them iteratively.
3. Write `lowest_common_ancestor(root, p, q)` for a BST using only comparisons, in `O(h)` without recursion.
4. Write `floor_key(tree, key)` and `ceiling_key(tree, key)` as new methods of `BST`: the largest key at most `key`, and the smallest key at least `key`.
5. Write `range_query(tree, low, high)` that returns all keys with `low <= key <= high` in sorted order, visiting only the subtrees that can contain them. State its cost in terms of `h` and the number of results.
6. Write `kth_smallest_augmented(root, k)` for nodes that store a `size` field, and show how `put` must update the sizes along the path.
7. Write `rank(tree, key)` that returns how many keys in the tree are smaller than `key`, using subtree sizes.
8. Write `sorted_array_to_bst(values)` that builds a height-balanced BST from a sorted list by choosing the middle item as the root, and show that its height is `floor(log2(n))`.
9. Write `convert_bst_to_greater_tree(root)` so that each key becomes the sum of all keys greater than or equal to it, using a reverse inorder traversal.
10. Write `recover_bst(root)` for a BST in which exactly two keys were swapped by mistake, finding and fixing them in one inorder pass.
11. Write `two_sum_in_bst(root, target)` that finds two keys adding up to the target using two iterators (one forward, one backward) without storing all the keys.
12. Write an iterator class `BSTIterator` with `next()` and `has_next()` that uses `O(h)` memory, and test that it can be stopped early.
13. Insert 10,000 keys in sorted order, then in random order, into your `BST` and time both builds and 10,000 searches. Explain the results.
14. Add a `delete` method to `BST` that uses the predecessor instead of the successor, and write a model-based test that compares both deletion strategies against a `dict` on random operations.

## Summary

A binary search tree keeps keys ordered so that everything in a node's left subtree is smaller and everything in its right subtree is larger, which makes search, insertion, and deletion cost `O(h)` for height `h`. Search and insert follow one path, deletion has three cases (leaf, one child, two children replaced by the inorder successor), successor and predecessor are found by walking from the root, and an inorder traversal visits keys in sorted order. Validate a BST with bounds from all ancestors, not just the parent. A BST is only as good as its shape: random insertion gives a height near `2 log2(n)`, but sorted insertion produces a chain with `O(n)` operations, which real data such as timestamps often triggers. Prefer iterative implementations in Python, treat comparisons and keys carefully, and reach for `dict`, `bisect`, `sortedcontainers`, or a balanced tree rather than an unguarded BST in real systems.
