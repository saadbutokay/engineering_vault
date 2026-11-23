## Overview

A tree is a structure made of nodes connected by parent-child links, with one special node at the top called the root. Unlike a list, where every item has one successor, a node in a tree can have several children, so a tree can describe things that branch: a company org chart, a folder hierarchy, a family tree, the structure of an HTML page, or the parsed form of an arithmetic expression.

A binary tree is a tree in which every node has at most two children, called the left child and the right child. That small restriction makes trees easy to implement and analyze, and it is the starting point for the most important tree structures in programming: binary search trees, balanced trees, heaps, and expression trees.

This note builds the vocabulary and the core techniques you will use for every other tree topic. You will learn the terminology (root, leaf, height, depth, level), how to represent a tree with node objects or with an array, how to visit every node in four standard orders (preorder, inorder, postorder, and level order) using both recursion and an explicit stack or queue, how to compute height and size, how to compare trees for equality and symmetry, and how to rebuild a tree from its traversals. Most tree algorithms are one of these traversals plus a small decision at each node.

## Core concepts

### Terminology

- Node: one element of the tree, holding a value and links to its children.
- Root: the top node. A non-empty tree has exactly one root. It has no parent.
- Parent and child: if node `A` links down to node `B`, then `A` is the parent of `B` and `B` is a child of `A`.
- Siblings: nodes with the same parent.
- Leaf: a node with no children.
- Internal node: a node with at least one child.
- Edge: a link from a parent to a child.
- Subtree: a node together with everything below it. Every child of a node is the root of its own subtree, which is why trees are naturally recursive.
- Depth of a node: the number of edges from the root to that node. The root has depth 0.
- Level: all nodes at the same depth. Level 0 is the root, level 1 its children, and so on.
- Height of a node: the number of edges on the longest downward path from that node to a leaf. A leaf has height 0.
- Height of a tree: the height of its root. This roadmap uses edges, so a single-node tree has height 0 and an empty tree has height `-1`. Some books count nodes instead, so always check which convention is in use.
- Size: the total number of nodes.

### Kinds of binary trees

- Full: every node has either 0 or 2 children.
- Perfect: all internal nodes have 2 children and all leaves are at the same depth. A perfect tree of height `h` has `2^(h + 1) - 1` nodes.
- Complete: every level is full except possibly the last, and the last level is filled from the left. Heaps use this shape.
- Balanced: the heights of the left and right subtrees of every node differ by a small bounded amount, which keeps the height close to `log2(n)`.
- Degenerate (skewed): every node has only one child, so the tree is really a linked list with height `n - 1`.

### Node representation

- Linked nodes: each node is an object with `value`, `left`, and `right` references (`None` when absent). This is flexible and the usual choice.
- Array representation: store the nodes of a complete tree level by level in a list. For the node at index `i`, its children are at `2i + 1` and `2i + 2`, and its parent is at `(i - 1) // 2`. There are no references at all, which saves memory, but a tree with gaps wastes slots. Heaps are stored this way.

### Tree traversal

Traversal means visiting every node exactly once. All four orders take `O(n)` time. The three depth-first orders differ in when the node itself is processed relative to its subtrees.

- Preorder: node, then left subtree, then right subtree. Use it to copy a tree or to serialize it, because the parent comes before its children.
- Inorder: left subtree, then node, then right subtree. In a binary search tree this visits the keys in sorted order.
- Postorder: left subtree, then right subtree, then node. Use it when a node's result depends on its children's results (computing sizes, deleting a tree, evaluating an expression).
- Level order: visit all nodes of level 0, then level 1, and so on. It uses a queue and is a breadth-first traversal.

### Recursive versus iterative traversal

Recursive traversals are short and match the structure of the tree. Their stack depth equals the height of the tree, so they are fine for balanced trees and can exceed Python's recursion limit on a very skewed tree of thousands of nodes. Iterative traversals use an explicit stack on the heap and have no depth limit.

### Height and size

Both are computed from the children's results: `size(node) = 1 + size(left) + size(right)` and `height(node) = 1 + max(height(left), height(right))`, with the base case for an empty subtree. That is a postorder pattern.

### Equality and symmetry

Two trees are the same if their roots hold equal values and their left subtrees are the same and their right subtrees are the same. A tree is symmetric (a mirror image of itself) if its left and right subtrees are mirrors of each other: the left child of one matches the right child of the other and vice versa.

### Constructing a tree from traversals

A single traversal does not identify a tree, but two of them often do. If all values are distinct, preorder plus inorder determine the tree uniquely. Preorder tells you the root (the first item). Inorder tells you which values lie to the left of the root and which to the right. Repeat for each subtree.

## Basic syntax examples

### A node class and a helper to build trees

The helper reads a level-order list, the same format many problem statements use, where `None` marks a missing child.

```python
from collections import deque
from collections.abc import Sequence
from dataclasses import dataclass


@dataclass(slots=True)
class TreeNode[T]:
    value: T
    left: "TreeNode[T] | None" = None
    right: "TreeNode[T] | None" = None


def build_tree(values: Sequence[int | None]) -> TreeNode[int] | None:
    """Build a tree from a level-order list, where None marks a missing child."""
    if not values:
        return None
    first = values[0]
    if first is None:
        return None
    root = TreeNode(first)
    queue = deque([root])
    index = 1
    while queue and index < len(values):
        node = queue.popleft()
        left_value = values[index]
        index += 1
        if left_value is not None:
            node.left = TreeNode(left_value)
            queue.append(node.left)
        if index < len(values):
            right_value = values[index]
            index += 1
            if right_value is not None:
                node.right = TreeNode(right_value)
                queue.append(node.right)
    return root


tree = build_tree([1, 2, 3, 4, 5, None, 7])
```

The tree built by the last line:

```text
        1
       / \
      2   3
     / \   \
    4   5   7
```

Its root is `1`, its leaves are `4`, `5`, and `7`, its height is 2 (two edges from the root to a leaf), and its size is 6.

### Recursive traversals

```python
def preorder[T](root: TreeNode[T] | None) -> list[T]:
    result: list[T] = []

    def visit(node: TreeNode[T] | None) -> None:
        if node is None:  # base case: an empty subtree
            return
        result.append(node.value)  # node first
        visit(node.left)
        visit(node.right)

    visit(root)
    return result


def inorder[T](root: TreeNode[T] | None) -> list[T]:
    result: list[T] = []

    def visit(node: TreeNode[T] | None) -> None:
        if node is None:
            return
        visit(node.left)
        result.append(node.value)  # node between its subtrees
        visit(node.right)

    visit(root)
    return result


def postorder[T](root: TreeNode[T] | None) -> list[T]:
    result: list[T] = []

    def visit(node: TreeNode[T] | None) -> None:
        if node is None:
            return
        visit(node.left)
        visit(node.right)
        result.append(node.value)  # node last

    visit(root)
    return result


print(preorder(tree))  # [1, 2, 4, 5, 3, 7]
print(inorder(tree))  # [4, 2, 5, 1, 3, 7]
print(postorder(tree))  # [4, 5, 2, 7, 3, 1]
```

Order of visits for the three traversals on the tree above:

```text
preorder:   1  2  4  5  3  7       (root, left subtree, right subtree)
inorder:    4  2  5  1  3  7       (left subtree, root, right subtree)
postorder:  4  5  2  7  3  1       (left subtree, right subtree, root)
```

### Iterative traversals

```python
def preorder_iterative[T](root: TreeNode[T] | None) -> list[T]:
    result: list[T] = []
    stack: list[TreeNode[T]] = [root] if root is not None else []
    while stack:
        node = stack.pop()
        result.append(node.value)
        if node.right is not None:  # push right first so left is popped first
            stack.append(node.right)
        if node.left is not None:
            stack.append(node.left)
    return result


def inorder_iterative[T](root: TreeNode[T] | None) -> list[T]:
    result: list[T] = []
    stack: list[TreeNode[T]] = []
    current = root
    while current is not None or stack:
        while current is not None:  # go as far left as possible
            stack.append(current)
            current = current.left
        current = stack.pop()
        result.append(current.value)
        current = current.right  # then handle the right subtree
    return result


def postorder_iterative[T](root: TreeNode[T] | None) -> list[T]:
    """Visit root, right, left, then reverse: that gives left, right, root."""
    result: list[T] = []
    stack: list[TreeNode[T]] = [root] if root is not None else []
    while stack:
        node = stack.pop()
        result.append(node.value)
        if node.left is not None:
            stack.append(node.left)
        if node.right is not None:
            stack.append(node.right)
    result.reverse()
    return result


def level_order[T](root: TreeNode[T] | None) -> list[list[T]]:
    """One inner list per level. Uses a queue, so it is breadth first."""
    levels: list[list[T]] = []
    queue: deque[TreeNode[T]] = deque([root] if root is not None else [])
    while queue:
        level: list[T] = []
        for _ in range(len(queue)):  # exactly the nodes of the current level
            node = queue.popleft()
            level.append(node.value)
            if node.left is not None:
                queue.append(node.left)
            if node.right is not None:
                queue.append(node.right)
        levels.append(level)
    return levels


print(preorder_iterative(tree))  # [1, 2, 4, 5, 3, 7]
print(inorder_iterative(tree))  # [4, 2, 5, 1, 3, 7]
print(postorder_iterative(tree))  # [4, 5, 2, 7, 3, 1]
print(level_order(tree))  # [[1], [2, 3], [4, 5, 7]]
```

### Height, size, and leaves

```python
def height[T](root: TreeNode[T] | None) -> int:
    """Edges on the longest root-to-leaf path. An empty tree has height -1."""
    if root is None:
        return -1
    return 1 + max(height(root.left), height(root.right))


def size[T](root: TreeNode[T] | None) -> int:
    if root is None:
        return 0
    return 1 + size(root.left) + size(root.right)


def count_leaves[T](root: TreeNode[T] | None) -> int:
    if root is None:
        return 0
    if root.left is None and root.right is None:
        return 1
    return count_leaves(root.left) + count_leaves(root.right)


print(height(tree), size(tree), count_leaves(tree))  # 2 6 3
```

### Equality, symmetry, and inversion

```python
def is_same_tree[T](first: TreeNode[T] | None, second: TreeNode[T] | None) -> bool:
    if first is None or second is None:
        return first is second  # both empty is a match, one empty is not
    return (
        first.value == second.value
        and is_same_tree(first.left, second.left)
        and is_same_tree(first.right, second.right)
    )


def is_symmetric[T](root: TreeNode[T] | None) -> bool:
    def mirrors(left: TreeNode[T] | None, right: TreeNode[T] | None) -> bool:
        if left is None or right is None:
            return left is right
        return (
            left.value == right.value
            and mirrors(left.left, right.right)  # outer pair
            and mirrors(left.right, right.left)  # inner pair
        )

    return root is None or mirrors(root.left, root.right)


def invert[T](root: TreeNode[T] | None) -> TreeNode[T] | None:
    """Swap the children of every node, producing the mirror image."""
    if root is not None:
        root.left, root.right = invert(root.right), invert(root.left)
    return root


print(is_same_tree(build_tree([1, 2, 3]), build_tree([1, 2, 3])))  # True
print(is_symmetric(build_tree([1, 2, 2, 3, 4, 4, 3])))  # True
print(is_symmetric(build_tree([1, 2, 2, None, 3, None, 3])))  # False
```

### Array representation

```python
def left_child(index: int) -> int:
    return 2 * index + 1


def right_child(index: int) -> int:
    return 2 * index + 2


def parent(index: int) -> int:
    return (index - 1) // 2


heap_like = [10, 20, 30, 40, 50, 60]
print(heap_like[left_child(1)], heap_like[right_child(1)])  # 40 50
print(heap_like[parent(5)])  # 30
```

Index map for the six items above:

```text
index:   0    1    2    3    4    5
value:  10   20   30   40   50   60

              10 (0)
            /        \
        20 (1)       30 (2)
        /    \       /
    40 (3)  50 (4) 60 (5)
```

### Rebuilding a tree from preorder and inorder

```python
def build_from_traversals(
    preorder_values: list[int], inorder_values: list[int]
) -> TreeNode[int] | None:
    """Rebuild a tree whose values are all distinct. O(n) with a position map."""
    if len(preorder_values) != len(inorder_values):
        raise ValueError("the traversals must have the same length")
    if len(set(preorder_values)) != len(preorder_values):
        raise ValueError("values must be distinct")
    if set(preorder_values) != set(inorder_values):
        raise ValueError("the traversals must contain the same values")
    position = {value: index for index, value in enumerate(inorder_values)}
    next_root = 0  # index into the preorder list of the next subtree root

    def build(low: int, high: int) -> TreeNode[int] | None:
        nonlocal next_root
        if low > high:
            return None
        value = preorder_values[next_root]
        next_root += 1
        node = TreeNode(value)
        middle = position[value]  # inorder splits the range around the root
        node.left = build(low, middle - 1)  # left must be built before right
        node.right = build(middle + 1, high)
        return node

    return build(0, len(inorder_values) - 1)


rebuilt = build_from_traversals([1, 2, 4, 5, 3, 7], [4, 2, 5, 1, 3, 7])
print(level_order(rebuilt))  # [[1], [2, 3], [4, 5, 7]]
```

## Deeper behavior

### Why recursion fits trees

A tree is defined recursively: a tree is either empty or a node with two subtrees that are trees. A function that handles the empty case and combines the results from the two subtrees follows that definition exactly, and its correctness proof is a short induction. Almost all of the tree code in this roadmap has one of three shapes:

- Compute something from the children first, then combine (postorder): height, size, sum, balance checks.
- Do something with the node, then pass information down to the children (preorder): depth tracking, path accumulation, copying.
- Process the left, the node, and the right in sorted order (inorder): binary search tree operations.

### The depth problem

The recursion depth of a traversal equals the height of the tree. For a balanced tree with a million nodes, the height is about 20, so recursion is fine. For a skewed tree it can be as large as the node count.

```python
def make_chain(length: int) -> TreeNode[int] | None:
    """A degenerate tree: every node has only a right child."""
    root: TreeNode[int] | None = None
    for value in range(length, 0, -1):
        root = TreeNode(value, None, root)
    return root


chain = make_chain(5_000)
try:
    height(chain)
except RecursionError:
    print("recursive height failed")  # recursive height failed

print(len(level_order(chain)) - 1)  # 4999, the iterative version has no depth limit
```

Production code that handles trees built from outside data (parsed JSON, user-defined hierarchies) should use iterative traversals or enforce a maximum depth, as the industry snippet below does.

### Space and time of each traversal

- All depth-first traversals take `O(n)` time. Their extra space is `O(h)`, the stack depth, where `h` is the height: `O(log n)` for balanced trees and `O(n)` for skewed ones.
- Level order uses `O(w)` extra space, where `w` is the maximum width of any level. For a perfect tree, the last level holds about half of all the nodes, so level order needs `O(n)` space there.
- Morris traversal visits nodes in order with `O(1)` extra space by temporarily rewiring links and restoring them. It is mostly of interest as a technique, and it is hard to use safely when a tree is shared.

### Facts worth knowing about the shapes

- A binary tree with `n` nodes has `n - 1` edges.
- A perfect binary tree of height `h` has `2^(h + 1) - 1` nodes and `2^h` leaves, so about half of the nodes are leaves.
- In a full binary tree, the number of leaves is the number of internal nodes plus one.
- The height of a binary tree with `n` nodes is at least `floor(log2(n))` and at most `n - 1`. That range between logarithmic and linear is the difference between fast and useless search trees.
- The number of different binary tree shapes with `n` nodes is the Catalan number, which grows roughly like `4^n`.

### Which traversals identify a tree

If the values are distinct, preorder plus inorder identify the tree, and so do postorder plus inorder. Preorder plus postorder do not in general: a root with only a left child and a root with only a right child give the same two sequences. They do identify a full binary tree. Inorder is the key: it separates left from right. Without it you must record the missing children explicitly, which is what serialization does (covered with tree algorithms).

### Level-order construction

The helper `build_tree` uses a queue of nodes that still need children, taking two values for each node it dequeues. This mirrors how a level-order list is laid out when the tree is stored with explicit `None` markers, and it is the reverse of `level_order`. For a complete tree, the `None` markers are unnecessary, which is why the array representation works for heaps.

### Equality of values versus identity of nodes

`is_same_tree` compares values with `==`, so two separately built trees with equal shapes and values are the same. It uses `is` only for the empty case. If you compare nodes with `==` in code that uses a `@dataclass`, the generated `__eq__` compares fields recursively, which on a deep tree can itself hit the recursion limit. Decide deliberately which comparison you want.

### Mutating a tree during traversal

Inverting or deleting nodes while traversing is safe if you finish with a node's children before you detach them, which is why `invert` swaps after the recursive calls return their results. When a traversal collects nodes to delete, collect first and delete afterwards.

## Industry-level snippet

A marketing platform lets analysts define audience segments with a small boolean filter, for example "country is BD and (plan is pro or age is at least 30)". The filter is a binary tree: each internal node is `and` or `or` with exactly two children, and each leaf is a condition on one user field. The tree is stored as JSON, validated when saved, and evaluated against user records.

Because analysts, or an attacker, can submit arbitrary JSON, the code must treat it as untrusted input. The parser enforces a maximum depth and a maximum node count, so a pathological filter cannot exhaust the stack or the CPU, and it validates the shape of every part. The tree measurements use iterative traversals. Evaluation is recursive, which is safe only because validation bounded the depth, so the bound is enforced in one place. Type mismatches such as comparing a number with a text value evaluate to false instead of raising an error in the middle of a campaign run.

Segments are saved in a transaction after validation, and loading validates again, so a corrupted row cannot produce an unbounded tree.

```python
# segment_filter.py
import json
import logging
import sqlite3
from collections import deque
from collections.abc import Mapping
from dataclasses import dataclass
from typing import Any, Protocol

logger = logging.getLogger(__name__)

MAX_DEPTH = 8
MAX_NODES = 63
COMPARATORS = frozenset({"=", "!=", ">", ">=", "<", "<="})

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS segment (
    name TEXT PRIMARY KEY,
    definition TEXT NOT NULL
)
"""


class SupportsLessThan(Protocol):
    def __lt__(self, other: Any, /) -> bool: ...


class SegmentError(ValueError):
    """Raised when a segment definition is malformed or too large."""


class SegmentNotFoundError(LookupError):
    """Raised when no segment with the given name exists."""


@dataclass(slots=True)
class FilterNode:
    kind: str  # "and", "or", or "cond"
    field: str = ""
    comparator: str = ""
    value: int | str = 0
    left: "FilterNode | None" = None
    right: "FilterNode | None" = None


def parse_filter(definition: object) -> FilterNode:
    """Validate untrusted JSON data and build the tree."""
    budget = [MAX_NODES]
    return _parse(definition, 1, budget)


def _parse(definition: object, depth: int, budget: list[int]) -> FilterNode:
    if depth > MAX_DEPTH:
        raise SegmentError(f"filters may be nested at most {MAX_DEPTH} levels deep")
    budget[0] -= 1
    if budget[0] < 0:
        raise SegmentError(f"filters may have at most {MAX_NODES} parts")
    if not isinstance(definition, dict):
        raise SegmentError("every part of a filter must be a JSON object")
    for kind in ("and", "or"):
        if kind in definition:
            if len(definition) != 1:
                raise SegmentError(f"'{kind}' must be the only key in its object")
            children = definition[kind]
            if not isinstance(children, list) or len(children) != 2:
                raise SegmentError(f"'{kind}' needs exactly two parts")
            node = FilterNode(kind)
            node.left = _parse(children[0], depth + 1, budget)
            node.right = _parse(children[1], depth + 1, budget)
            return node
    field = definition.get("field")
    comparator = definition.get("op")
    value = definition.get("value")
    if not isinstance(field, str) or not field:
        raise SegmentError("a condition needs a non-empty 'field'")
    if not isinstance(comparator, str) or comparator not in COMPARATORS:
        raise SegmentError(f"unknown comparison {comparator!r}")
    if isinstance(value, bool) or not isinstance(value, (int, str)):
        raise SegmentError("a condition value must be a number or a string")
    return FilterNode("cond", field, comparator, value)


def _apply[T: SupportsLessThan](comparator: str, left: T, right: T) -> bool:
    match comparator:
        case "=":
            return bool(left == right)
        case "!=":
            return bool(left != right)
        case "<":
            return left < right
        case "<=":
            return left < right or bool(left == right)
        case ">":
            return right < left
        case _:  # ">="
            return right < left or bool(left == right)


def _compare(actual: object, comparator: str, expected: int | str) -> bool:
    if isinstance(actual, int) and isinstance(expected, int):
        return _apply(comparator, actual, expected)
    if isinstance(actual, str) and isinstance(expected, str):
        return _apply(comparator, actual, expected)
    return False  # a missing field or a number-versus-text mismatch never matches


def evaluate(node: FilterNode, user: Mapping[str, int | str]) -> bool:
    """Recursive, and safe because parse_filter bounded the depth of the tree."""
    if node.kind == "cond":
        return _compare(user.get(node.field), node.comparator, node.value)
    assert node.left is not None and node.right is not None
    if node.kind == "and":
        return evaluate(node.left, user) and evaluate(node.right, user)
    return evaluate(node.left, user) or evaluate(node.right, user)


def node_count(root: FilterNode) -> int:
    count = 0
    stack = [root]
    while stack:
        node = stack.pop()
        count += 1
        stack.extend(child for child in (node.left, node.right) if child is not None)
    return count


def tree_height(root: FilterNode) -> int:
    """Edges on the longest path, by level order traversal."""
    levels = 0
    queue = deque([root])
    while queue:
        levels += 1
        for _ in range(len(queue)):
            node = queue.popleft()
            children = (node.left, node.right)
            queue.extend(child for child in children if child is not None)
    return levels - 1


def fields_used(root: FilterNode) -> list[str]:
    """Field names in preorder, without repeats."""
    names: list[str] = []
    stack = [root]
    while stack:
        node = stack.pop()
        if node.kind == "cond" and node.field not in names:
            names.append(node.field)
        for child in (node.right, node.left):  # push right first, so left pops first
            if child is not None:
                stack.append(child)
    return names


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_segment(connection: sqlite3.Connection, name: str, definition: object) -> None:
    if not name:
        raise ValueError("segment name must not be empty")
    parse_filter(definition)  # validate before anything is written
    text = json.dumps(definition, sort_keys=True)
    try:
        with connection:
            connection.execute(
                "INSERT INTO segment (name, definition) VALUES (?, ?) "
                "ON CONFLICT (name) DO UPDATE SET definition = excluded.definition",
                (name, text),
            )
    except sqlite3.Error:
        logger.exception("could not save segment %r", name)
        raise
    logger.info("saved segment %r", name)


def load_segment(connection: sqlite3.Connection, name: str) -> FilterNode:
    row = connection.execute(
        "SELECT definition FROM segment WHERE name = ?", (name,)
    ).fetchone()
    if row is None:
        raise SegmentNotFoundError(f"no segment named {name!r}")
    try:
        return parse_filter(json.loads(row[0]))  # validate again: data can be damaged
    except json.JSONDecodeError as error:
        raise SegmentError(f"segment {name!r} is not valid JSON") from error
```

```python
# test_segment_filter.py
import sqlite3
from collections.abc import Iterator

import pytest

from segment_filter import (
    MAX_DEPTH,
    MAX_NODES,
    SegmentError,
    SegmentNotFoundError,
    create_schema,
    evaluate,
    fields_used,
    load_segment,
    node_count,
    parse_filter,
    save_segment,
    tree_height,
)


def cond(field: str, op: str, value: object) -> dict[str, object]:
    return {"field": field, "op": op, "value": value}


SEGMENT: dict[str, object] = {
    "and": [
        cond("country", "=", "BD"),
        {"or": [cond("plan", "=", "pro"), cond("age", ">=", 30)]},
    ]
}


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def test_evaluation_follows_and_or_structure() -> None:
    tree = parse_filter(SEGMENT)
    assert evaluate(tree, {"country": "BD", "plan": "pro", "age": 20})
    assert evaluate(tree, {"country": "BD", "plan": "free", "age": 30})
    assert not evaluate(tree, {"country": "BD", "plan": "free", "age": 29})
    assert not evaluate(tree, {"country": "IN", "plan": "pro", "age": 50})


def test_missing_fields_and_type_mismatches_do_not_match() -> None:
    tree = parse_filter(cond("age", ">", 18))
    assert not evaluate(tree, {})
    assert not evaluate(tree, {"age": "twenty"})
    assert evaluate(tree, {"age": 19})


@pytest.mark.parametrize(
    ("op", "actual", "expected"),
    [
        ("=", 5, True),
        ("=", 6, False),
        ("!=", 6, True),
        ("!=", 5, False),
        ("<", 4, True),
        ("<", 5, False),
        ("<=", 5, True),
        ("<=", 6, False),
        (">", 6, True),
        (">", 5, False),
        (">=", 5, True),
        (">=", 4, False),
    ],
)
def test_comparators(op: str, actual: int, expected: bool) -> None:
    tree = parse_filter(cond("n", op, 5))
    assert evaluate(tree, {"n": actual}) is expected


def test_structure_measurements() -> None:
    tree = parse_filter(SEGMENT)
    assert node_count(tree) == 5
    assert tree_height(tree) == 2
    assert fields_used(tree) == ["country", "plan", "age"]


@pytest.mark.parametrize(
    "bad",
    [
        "text",
        {},
        {"and": [cond("a", "=", 1)]},
        {"and": [cond("a", "=", 1), cond("b", "=", 2)], "extra": 1},
        {"not": cond("a", "=", 1)},
        cond("", "=", 1),
        cond("a", "~", 1),
        cond("a", "=", True),
        cond("a", "=", [1]),
        {"field": "a", "op": "="},
        {"field": "a", "op": ["="], "value": 1},
    ],
)
def test_malformed_definitions_are_rejected(bad: object) -> None:
    with pytest.raises(SegmentError):
        parse_filter(bad)


def test_depth_limit() -> None:
    node: dict[str, object] = cond("a", "=", 1)
    for _ in range(MAX_DEPTH - 1):
        node = {"and": [node, cond("b", "=", 2)]}
    parse_filter(node)  # exactly at the limit
    with pytest.raises(SegmentError):
        parse_filter({"or": [node, cond("c", "=", 3)]})


def balanced(leaves: list[dict[str, object]]) -> dict[str, object]:
    """Combine conditions into a balanced tree of 'or' nodes."""
    if len(leaves) == 1:
        return leaves[0]
    middle = len(leaves) // 2
    return {"or": [balanced(leaves[:middle]), balanced(leaves[middle:])]}


def test_size_limit() -> None:
    at_limit = [cond(f"f{i}", "=", i) for i in range((MAX_NODES + 1) // 2)]
    parse_filter(balanced(at_limit))  # 32 leaves and 31 'or' nodes make 63 parts
    too_many = [cond(f"f{i}", "=", i) for i in range(MAX_NODES)]
    with pytest.raises(SegmentError):
        parse_filter(balanced(too_many))


def test_save_and_load_round_trip(connection: sqlite3.Connection) -> None:
    save_segment(connection, "bd-pro", SEGMENT)
    tree = load_segment(connection, "bd-pro")
    assert evaluate(tree, {"country": "BD", "plan": "pro", "age": 1})


def test_invalid_segments_are_not_stored(connection: sqlite3.Connection) -> None:
    with pytest.raises(SegmentError):
        save_segment(connection, "bad", {"and": []})
    with pytest.raises(SegmentNotFoundError):
        load_segment(connection, "bad")
    with pytest.raises(ValueError):
        save_segment(connection, "", SEGMENT)


def test_damaged_rows_are_rejected_on_load(connection: sqlite3.Connection) -> None:
    connection.execute("INSERT INTO segment (name, definition) VALUES ('x', '{oops')")
    with pytest.raises(SegmentError):
        load_segment(connection, "x")
```

The tests for `MAX_DEPTH` and `MAX_NODES` show why the limits live in the parser: every later function can trust that the tree it receives is small and shallow.

## Common mistakes

- Forgetting the empty case (`None`) in a recursive function, which raises `AttributeError` when the recursion reaches a missing child.
- Mixing up the height of a node (edges down) with its depth (edges up from the root), or mixing the edge-count and node-count conventions without noticing.
- Pushing the children in the wrong order in an iterative preorder, so the traversal goes right before left.
- Confusing level order with depth-first order. Level order needs a queue, and depth-first order needs a stack or recursion.
- Not processing level by level in a level-order traversal when the per-level grouping matters. Take `len(queue)` at the start of each level.
- Building the right subtree before the left when reconstructing from preorder. The preorder list is consumed left to right, so the left subtree must be built first.
- Trying to rebuild a tree from preorder and postorder alone, or from traversals of a tree with duplicate values.
- Recursing on deep or untrusted trees and hitting `RecursionError`.
- Comparing nodes with `is` when you meant to compare values, or the other way around.
- Modifying a tree while you are traversing it without thinking about which nodes are still to be visited.
- Assuming a binary tree is balanced or sorted. Binary trees in general have no ordering rule.

## Practice exercises

1. Draw the tree for `build_tree([3, 9, 20, None, None, 15, 7])` and write its preorder, inorder, postorder, and level-order traversals by hand. Check them with the code.
2. Write `max_depth(root)` and `min_depth(root)` (the shortest path to a leaf). Explain why `min_depth` is not simply `1 + min(height(left), height(right))`.
3. Write `right_side_view(root)` that returns the values you would see standing to the right of the tree, one per level, using level order.
4. Write `zigzag_level_order(root)` that alternates the direction of each level.
5. Write `is_balanced(root)` that returns whether the heights of the two subtrees of every node differ by at most one, in `O(n)` using a single postorder pass.
6. Write `count_nodes_complete(root)` for a complete binary tree in `O(log^2 n)` by comparing the leftmost and rightmost depths.
7. Write `flatten_to_linked_list(root)` that rearranges the tree in place so each node's right child is the next node in preorder and every left child is `None`.
8. Write `build_from_inorder_postorder(inorder_values, postorder_values)` and test it against the preorder version.
9. Write `is_subtree(root, sub_root)` that tells whether `sub_root` occurs as a subtree of `root`, and state its complexity.
10. Write `sum_of_left_leaves(root)` and `sum_root_to_leaf_numbers(root)` where each root-to-leaf path of digits forms a number.
11. Write `binary_tree_paths(root)` that returns every root-to-leaf path as a string such as `"1->2->4"`, using an explicit stack of `(node, path)` pairs.
12. Write `tree_to_array(root)` that converts a complete tree to its array form, and `array_to_tree(values)` for the reverse, using the index formulas.
13. Implement Morris inorder traversal and explain how it restores the links it temporarily changes.
14. Extend `segment_filter` so that conditions also support an `in` comparator with a list of values, and describe how this changes validation and evaluation.

## Summary

A binary tree is a hierarchy in which each node has at most two children. Know the vocabulary: root, leaf, depth, level, and height, with an agreed convention for counting edges. Trees are stored either as linked node objects or, for complete trees, in an array with children at `2i + 1` and `2i + 2`. The four traversals, preorder, inorder, postorder, and level order, all run in `O(n)`, with depth-first versions using stack space proportional to the height and level order using a queue. Most tree algorithms compute a result from the children and combine it at the node, which is why recursion fits so well, but recursion depth equals tree height, so use iterative versions or a depth limit for skewed or untrusted trees. Preorder or postorder together with inorder rebuild a tree whose values are distinct. When a tree comes from outside your program, validate its depth and size before you operate on it.
