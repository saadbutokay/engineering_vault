## Overview

The earlier tree notes gave you the vocabulary, the traversals, and the main search structures. This note puts those tools to work on the problems that come up again and again: finding where two nodes meet, measuring the longest path through a tree, finding the best path by value, checking and counting paths that add up to a target, turning a tree into a string and back, handling trees with any number of children, and solving optimization problems by combining answers from subtrees.

Almost all of these problems have the same shape. You process a node by first asking each child for a small piece of information about its subtree, such as its height, the best downward path, or the best score with and without the node itself, and then you combine those pieces into your own answer and pass it up. The only real design question is what to compute at each node and what to return to the parent. That is a postorder computation, and with a few exceptions the code is a dozen lines once you know what to return.

Trees are also everywhere in real systems: file systems, organization charts, the structure of a web page, the abstract syntax trees that compilers and linters build, nested comment threads, and category hierarchies. This note covers lowest common ancestor, diameter, maximum path sum, path sum variants, serialization and deserialization, `N`-ary trees, tree dynamic programming, and how to handle tree-shaped data from a database safely.

## Core concepts

### Lowest common ancestor

The lowest common ancestor (LCA) of two nodes `p` and `q` is the deepest node that has both of them in its subtree, where a node counts as its own descendant. It answers questions such as "who is the nearest manager above both of these employees?" and "which folder contains both of these files?"

- General binary tree: search both subtrees. If one side finds `p` and the other finds `q`, the current node is the LCA. If only one side finds anything, pass that result up.
- Binary search tree: use the ordering. If both values are smaller than the node, go left. If both are larger, go right. Otherwise the current node is the split point, which is the LCA.
- With parent pointers: collect the ancestors of one node, then walk up from the other until you hit one of them.

### Diameter

The diameter of a tree is the number of edges on the longest path between any two nodes. That path need not pass through the root. For each node, the longest path that bends at that node has length `height(left) + height(right) + 2` (counting edges), so compute heights bottom-up and keep the best bend seen. It takes `O(n)` in one pass.

### Maximum path sum

Each node has a value, possibly negative. Find the path (any sequence of connected nodes, going up and then down, without revisiting) with the largest sum. For each node, compute the best downward path starting at that node, ignoring any child path whose sum is negative. The best path that bends at the node is its value plus the two best downward paths from its children. Keep the best of those over all nodes.

### Path sum variants

- Does any root-to-leaf path add up to the target?
- List all root-to-leaf paths with the target sum (a backtracking problem: add a node to the current path on the way down and remove it on the way back).
- Count all downward paths, starting and ending anywhere, with the target sum. The trick is the prefix-sum technique from arrays: keep a dictionary of how many times each running sum has occurred on the path from the root to the current node.

### Serialization and deserialization

Serialization converts a tree into a string so it can be stored or sent. Deserialization rebuilds the tree. A preorder traversal with an explicit marker for missing children (for example `#`) identifies the tree uniquely, because the markers record exactly where each subtree ends. For example `1,2,#,#,3,#,#` is a root `1` with a leaf `2` on the left and a leaf `3` on the right. Level-order with markers is the format many problem statements use.

### N-ary trees

Real hierarchies rarely have exactly two children per node. An `N`-ary tree node holds a list of children. All the traversal ideas carry over: preorder and postorder visit a node before or after all its children, and level order uses a queue. Depth-first versions use recursion or a stack, and the height of a node is `1 + max` over its children.

### Tree dynamic programming

Many optimization problems on trees can be solved by computing, for each node, a small table of answers for its subtree under different conditions, and combining the children's tables. A classic example is the house robber problem on a tree: you cannot rob two directly connected houses. For each node compute the best total if the node is robbed and the best total if it is skipped. If it is robbed, both children must be skipped. If it is skipped, each child can be in either state, whichever is better.

### Tree-shaped data in real systems

- File systems: directory size is the sum of its files plus the sizes of its subdirectories (postorder).
- Organization charts: headcount under a manager, chain of command, and the lowest common manager of two employees.
- The DOM and XML: nested elements, searched by depth-first traversal.
- Abstract syntax trees: compilers, linters, and formatters walk the tree of a program, and Python's own `ast` module exposes it.
- Comment threads and category trees: nested replies and parent-child categories, usually stored as rows with a `parent_id` column.

## Basic syntax examples

### Node and a helper to build trees

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
```

### Lowest common ancestor in a binary tree

```python
def lowest_common_ancestor(
    root: TreeNode[int] | None, first: int, second: int
) -> TreeNode[int] | None:
    """Assumes both values are in the tree and are distinct. O(n)."""
    if root is None:
        return None
    if root.value == first or root.value == second:
        return root  # this node is one of the two, so it covers the other below it
    left = lowest_common_ancestor(root.left, first, second)
    right = lowest_common_ancestor(root.right, first, second)
    if left is not None and right is not None:
        return root  # one value was found on each side: this is where they split
    return left if left is not None else right


tree = build_tree([3, 5, 1, 6, 2, 0, 8, None, None, 7, 4])
lca = lowest_common_ancestor(tree, 5, 1)
print(lca.value if lca else None)  # 3
lca = lowest_common_ancestor(tree, 5, 4)
print(lca.value if lca else None)  # 5, a node is its own ancestor
lca = lowest_common_ancestor(tree, 7, 4)
print(lca.value if lca else None)  # 2
```

The tree used above:

```text
            3
          /   \
         5     1
        / \   / \
       6   2 0   8
          / \
         7   4
```

### Lowest common ancestor in a binary search tree

```python
def lca_in_bst(root: TreeNode[int], first: int, second: int) -> TreeNode[int]:
    """Use the ordering to follow one path. O(h) time, O(1) space."""
    node = root
    low, high = min(first, second), max(first, second)
    while True:
        if high < node.value and node.left is not None:
            node = node.left  # both values are in the left subtree
        elif low > node.value and node.right is not None:
            node = node.right  # both values are in the right subtree
        else:
            return node  # the values split here, or this node is one of them


bst = build_tree([6, 2, 8, 0, 4, 7, 9, None, None, 3, 5])
print(lca_in_bst(bst, 2, 8).value, lca_in_bst(bst, 3, 5).value)  # 6 4
```

### Diameter in one pass

```python
def diameter[T](root: TreeNode[T] | None) -> int:
    """Edges on the longest path between any two nodes. O(n)."""
    best = 0

    def height(node: TreeNode[T] | None) -> int:  # edges; an empty subtree is -1
        nonlocal best
        if node is None:
            return -1
        left = height(node.left)
        right = height(node.right)
        best = max(best, left + right + 2)  # the longest path that bends at this node
        return 1 + max(left, right)

    height(root)
    return best


print(diameter(build_tree([1, 2, 3, 4, 5])))  # 3, the path 4 - 2 - 1 - 3
print(diameter(build_tree([1, 2])))  # 1
print(diameter(None))  # 0
```

### Maximum path sum

```python
def max_path_sum(root: TreeNode[int]) -> int:
    """Best sum of any path through connected nodes. Values may be negative."""
    best = root.value

    def downward_gain(node: TreeNode[int] | None) -> int:
        """Best sum of a path that starts at this node and goes down one branch."""
        nonlocal best
        if node is None:
            return 0
        left = max(downward_gain(node.left), 0)  # never take a negative branch
        right = max(downward_gain(node.right), 0)
        best = max(best, node.value + left + right)  # a path that bends at this node
        return node.value + max(left, right)  # the parent can extend only one branch

    downward_gain(root)
    return best


tree_sum = build_tree([-10, 9, 20, None, None, 15, 7])
assert tree_sum is not None
print(max_path_sum(tree_sum))  # 42, the path 15 - 20 - 7
```

### Path sum variants

```python
def has_path_sum(root: TreeNode[int] | None, target: int) -> bool:
    """True if some root-to-leaf path adds up to the target."""
    if root is None:
        return False
    if root.left is None and root.right is None:
        return root.value == target
    remaining = target - root.value
    return has_path_sum(root.left, remaining) or has_path_sum(root.right, remaining)


def paths_with_sum(root: TreeNode[int] | None, target: int) -> list[list[int]]:
    """All root-to-leaf paths with the target sum, by backtracking."""
    found: list[list[int]] = []
    path: list[int] = []

    def visit(node: TreeNode[int] | None, remaining: int) -> None:
        if node is None:
            return
        path.append(node.value)  # extend the path on the way down
        if node.left is None and node.right is None and node.value == remaining:
            found.append(path[:])  # copy it: the path list keeps changing
        visit(node.left, remaining - node.value)
        visit(node.right, remaining - node.value)
        path.pop()  # undo on the way back up

    visit(root, target)
    return found


def count_paths_with_sum(root: TreeNode[int] | None, target: int) -> int:
    """Count downward paths (start and end anywhere) with the target sum. O(n)."""
    seen: dict[int, int] = {0: 1}  # running sums on the current root-to-node path
    total = 0

    def visit(node: TreeNode[int] | None, running: int) -> None:
        nonlocal total
        if node is None:
            return
        running += node.value
        total += seen.get(running - target, 0)  # earlier prefixes that complete a path
        seen[running] = seen.get(running, 0) + 1
        visit(node.left, running)
        visit(node.right, running)
        seen[running] -= 1  # leaving this node: remove its prefix sum

    visit(root, 0)
    return total


sums = build_tree([5, 4, 8, 11, None, 13, 4, 7, 2, None, None, 5, 1])
print(has_path_sum(sums, 22))  # True
print(paths_with_sum(sums, 22))  # [[5, 4, 11, 2], [5, 8, 4, 5]]
counting = build_tree([10, 5, -3, 3, 2, None, 11, 3, -2, None, 1])
print(count_paths_with_sum(counting, 8))  # 3
```

### Serialization and deserialization

```python
def serialize(root: TreeNode[int] | None) -> str:
    """Preorder with '#' for missing children, using an explicit stack."""
    parts: list[str] = []
    stack: list[TreeNode[int] | None] = [root]
    while stack:
        node = stack.pop()
        if node is None:
            parts.append("#")
            continue
        parts.append(str(node.value))
        stack.append(node.right)  # pushed first, so it is popped after the left side
        stack.append(node.left)
    return ",".join(parts)


def deserialize(data: str) -> TreeNode[int] | None:
    """Rebuild the tree from the output of serialize, rejecting malformed input."""
    tokens = data.split(",")
    if tokens[0] == "#":
        if len(tokens) != 1:
            raise ValueError("unused data after an empty tree")
        return None
    root = TreeNode(int(tokens[0]))
    stack: list[tuple[TreeNode[int], int]] = [(root, 0)]  # (node, 0 left / 1 right)
    position = 1
    while stack:
        if position >= len(tokens):
            raise ValueError("the data ended before the tree was complete")
        node, side = stack.pop()
        token = tokens[position]
        position += 1
        child = None if token == "#" else TreeNode(int(token))
        if side == 0:
            node.left = child
            stack.append((node, 1))  # come back for the right child
        else:
            node.right = child
        if child is not None:
            stack.append((child, 0))  # finish the child's whole subtree first
    if position != len(tokens):
        raise ValueError("unused data at the end")
    return root


original = build_tree([1, 2, 3, None, None, 4, 5])
text = serialize(original)
print(text)  # 1,2,#,#,3,4,#,#,5,#,#
print(serialize(deserialize(text)) == text)  # True
```

### N-ary trees

```python
from dataclasses import field


@dataclass(slots=True)
class NaryNode:
    value: str
    children: list["NaryNode"] = field(default_factory=list)


def nary_max_depth(root: NaryNode | None) -> int:
    """Number of levels, using an explicit queue so depth is not a problem."""
    if root is None:
        return 0
    depth = 0
    level = [root]
    while level:
        depth += 1
        level = [child for node in level for child in node.children]
    return depth


def nary_preorder(root: NaryNode | None) -> list[str]:
    if root is None:
        return []
    result: list[str] = []
    stack = [root]
    while stack:
        node = stack.pop()
        result.append(node.value)
        stack.extend(reversed(node.children))  # reversed keeps left-to-right order
    return result


site = NaryNode(
    "home",
    [
        NaryNode("products", [NaryNode("phones"), NaryNode("laptops")]),
        NaryNode("about"),
        NaryNode("support", [NaryNode("faq")]),
    ],
)
print(nary_max_depth(site))  # 3
order = nary_preorder(site)
print(order[:4], len(order))  # ['home', 'products', 'phones', 'laptops'] 7
```

### Tree dynamic programming: house robber on a tree

```python
def rob_tree(root: TreeNode[int] | None) -> int:
    """Maximum total when no two directly connected nodes may both be taken."""

    def solve(node: TreeNode[int] | None) -> tuple[int, int]:
        """Return (best if this node is taken, best if it is skipped)."""
        if node is None:
            return 0, 0
        left_taken, left_skipped = solve(node.left)
        right_taken, right_skipped = solve(node.right)
        taken = node.value + left_skipped + right_skipped  # children must be skipped
        skipped = max(left_taken, left_skipped) + max(right_taken, right_skipped)
        return taken, skipped

    return max(solve(root))


print(rob_tree(build_tree([3, 2, 3, None, 3, None, 1])))  # 7, which is 3 + 3 + 1
print(rob_tree(build_tree([3, 4, 5, 1, 3, None, 1])))  # 9, which is 4 + 5
```

## Deeper behavior

### The shape of a tree algorithm

Almost every function above is the same idea written four ways. Decide three things.

1. What does the function return for a subtree? (A height, a best gain, a pair of scores, or a found node.)
2. How do you combine the answers from the two children and the node's own value into the answer for the whole subtree?
3. What do you record on the side because it is not what the parent needs? (The diameter and the best path sum are not what the parent receives. The parent needs a height or a downward gain.)

Separating "what I return to my parent" from "what I record as a candidate for the global answer" is the key idea in diameter and maximum path sum. The value returned upward is a path that can still be extended, which can bend at most zero times. The candidate recorded is a path that bends at this node, and cannot be extended.

### Why the LCA algorithm is correct

The function returns the node itself if it is `first` or `second`. Otherwise it returns whatever its subtrees find. Consider the deepest node that has `first` in one subtree and `second` in the other, or is one of the two values with the other below it. Above that node, only one of its two subtrees reports a result, so the same node is passed up unchanged. Below it, at most one of the two values is in any subtree, so no node can see both. So the first node that sees a result from both sides, or that is itself a target and has the other below it, is exactly the LCA. If one of the values is missing, the function returns the other as if it were the LCA, which is why the precondition matters.

### LCA with many queries

For one query, the `O(n)` recursion is fine. For many queries on a large static tree, preprocess. Store each node's parent and depth in one traversal, and then answer each query by lifting the deeper node up to the same depth and then moving both up together, which costs `O(depth)`. Binary lifting stores, for each node, its ancestors at distances 1, 2, 4, and so on, which answers each query in `O(log n)` after `O(n log n)` preprocessing. Another approach reduces LCA to a range minimum query over an Euler tour, giving constant-time queries.

### The diameter is not height times two

The longest path can lie entirely inside one subtree and never touch the root. That is why the code records a candidate at every node. A solution that only combines the heights of the root's two subtrees is wrong for trees where one side is a long chain that bends lower down.

### Maximum path sum: handling negatives

If every value is negative, the best path is the single largest value, not an empty path. The code initializes `best` with the root's value and takes `max(gain, 0)` only for the children, so a node always contributes its own value. A negative branch is dropped by using zero instead of extending into it. A common bug is returning `node.value + left + right` upward, which would let the parent extend a path that already bends.

### Backtracking versus returning values

`paths_with_sum` shares one mutable `path` list across calls. It appends on entry and pops on exit, so the list always equals the path from the root to the current node. When a result is found, it must be copied with `path[:]`, because the same list keeps changing afterwards. The `count_paths_with_sum` function uses the same undo discipline for a dictionary. Forgetting the undo step makes a sibling subtree see prefix sums that do not belong to its path.

### The prefix-sum trick on trees

A downward path from an ancestor `a` to a descendant `d` has sum `prefix(d) - prefix(parent of a)`, where `prefix(x)` is the sum from the root to `x`. So at each node, you count how many ancestors have a prefix equal to `prefix(node) - target`. The dictionary holds prefix sums for exactly the current root-to-node path, so each query is `O(1)` and the whole count is `O(n)`, instead of the `O(n * h)` of starting a new search at every node.

### Serialization formats

- Preorder with markers, as above: compact, easy to parse, and the standard approach. Any traversal order works if you include the markers for missing children.
- Level order with markers: matches the way trees are usually drawn and is what many examples use, but needs a queue to rebuild.
- Parent-id rows: a list of `(id, parent_id, value)` records, which is how trees live in databases, and which handles N-ary trees directly.
- JSON: a nested object with a `children` list is readable, but a parser that follows nesting recursively can be attacked with deeply nested input, so limit depth or use an iterative parser.

Validate the input of any deserializer. The code above rejects data that ends early, data with leftover tokens, and non-numeric tokens (through `int`), because a deserializer that trusts its input can be driven into huge allocations or infinite loops.

### Recursion and tree depth

All of the recursive functions in this note use `O(h)` stack space. For trees you build from controlled data, that is a few dozen frames. For hierarchies that come from users or databases, depth is not under your control. Either enforce a maximum depth when the data is loaded, or write the algorithm with an explicit stack, as the serialization functions and the org chart code in the next section do.

### Processing children before parents without recursion

To compute a bottom-up result on a deep tree without recursion, take a preorder list of the nodes (parents before children) and process it in reverse. In the reversed order, every child comes before its parent, so you can add each node's result into its parent's total as you go. This is the iterative form of postorder, and it is how the headcount below is computed.

### Tree DP, more generally

The house robber structure repeats in other problems: each node has a small number of states, and the state of a node restricts the states of its children. Maximum independent set (the largest set of nodes with no two adjacent) is exactly house robber on a general tree. Minimum vertex cover and minimum dominating set have the same shape with different transitions. Rerooting techniques compute the answer for every possible root with two passes, a downward pass and an upward pass.

## Industry-level snippet

An HR system stores the reporting structure as rows: each employee has an id, a name, and the id of their manager. The company needs headcount under every manager (including indirect reports), the chain of command above any employee, and the lowest common manager of two people for approval routing. This is tree-shaped data stored as parent pointers, and the algorithms are exactly the ones in this note.

Data in a table can break the tree property. A manager id can point to an employee who does not exist. A bad edit can make two employees manage each other, or an employee their own manager, which creates a cycle, and then no traversal from the top would ever reach them, while a loop that follows manager pointers upward would never end. The code therefore validates the whole chart before it trusts it: every manager must exist, and a traversal from the top must reach every employee, otherwise the unreached ones are in a cycle.

The headcount uses the iterative technique described above: one preorder pass with an explicit stack, then a reversed pass that adds each employee's size to their manager. This has no recursion, so a chain of thousands of levels is safe. Moving an employee to a new manager is checked against creating a cycle: the new manager must not be the employee or one of their reports, which is found by walking up the new manager's chain. The update runs in a transaction that takes the write lock before reading the chart, so two simultaneous moves cannot combine into a cycle.

```python
# org_chart.py
import logging
import sqlite3
from collections import defaultdict
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS employee (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id INTEGER REFERENCES employee (id),
    reports INTEGER NOT NULL DEFAULT 0
)
"""


class OrgChartError(ValueError):
    """Raised when the reporting structure is not a valid tree."""


class EmployeeNotFoundError(LookupError):
    """Raised when an employee id does not exist."""


@dataclass(frozen=True, slots=True)
class Employee:
    employee_id: int
    name: str
    manager_id: int | None


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def load_employees(connection: sqlite3.Connection) -> dict[int, Employee]:
    rows = connection.execute("SELECT id, name, manager_id FROM employee ORDER BY id")
    return {row[0]: Employee(row[0], row[1], row[2]) for row in rows}


def validate_chart(employees: dict[int, Employee]) -> list[int]:
    """Check the tree property and return all ids in preorder (managers first).

    Iterative, so the depth of the chart cannot cause a RecursionError.
    """
    children: defaultdict[int | None, list[int]] = defaultdict(list)
    for employee in employees.values():
        manager = employee.manager_id
        if manager is not None and manager not in employees:
            raise OrgChartError(
                f"employee {employee.employee_id} reports to unknown manager {manager}"
            )
        children[manager].append(employee.employee_id)
    order: list[int] = []
    stack = list(reversed(children[None]))  # the top-level employees
    while stack:
        current = stack.pop()
        order.append(current)
        stack.extend(reversed(children.get(current, [])))
    if len(order) != len(employees):
        reached = set(order)
        stuck = sorted(set(employees) - reached)
        raise OrgChartError(f"employees in a reporting cycle: {stuck[:5]}")
    return order


def count_reports(employees: dict[int, Employee]) -> dict[int, int]:
    """Direct and indirect reports of everyone. O(n), no recursion."""
    order = validate_chart(employees)
    sizes = {employee_id: 1 for employee_id in order}  # each subtree includes its root
    for employee_id in reversed(order):  # in reverse, reports come before managers
        manager = employees[employee_id].manager_id
        if manager is not None:
            sizes[manager] += sizes[employee_id]
    return {employee_id: size - 1 for employee_id, size in sizes.items()}


def chain_of_command(employees: dict[int, Employee], employee_id: int) -> list[int]:
    """The employee, their manager, and so on up to the top."""
    if employee_id not in employees:
        raise EmployeeNotFoundError(f"employee {employee_id} does not exist")
    chain = [employee_id]
    manager = employees[employee_id].manager_id
    while manager is not None:
        if len(chain) > len(employees):
            raise OrgChartError("the reporting chain never reaches the top")
        chain.append(manager)
        manager = employees[manager].manager_id
    return chain


def lowest_common_manager(
    employees: dict[int, Employee], first: int, second: int
) -> int:
    """The nearest person above both, where a person counts as above themselves."""
    above_first = set(chain_of_command(employees, first))
    for candidate in chain_of_command(employees, second):  # nearest first
        if candidate in above_first:
            return candidate
    raise OrgChartError(f"employees {first} and {second} have no common manager")


def refresh_report_counts(connection: sqlite3.Connection) -> int:
    """Store everyone's report count. Nothing is written if the chart is invalid."""
    employees = load_employees(connection)
    counts = count_reports(employees)
    try:
        with connection:
            connection.executemany(
                "UPDATE employee SET reports = ? WHERE id = ?",
                [(count, employee_id) for employee_id, count in counts.items()],
            )
    except sqlite3.Error:
        logger.exception("could not store report counts")
        raise
    logger.info("updated report counts for %d employees", len(counts))
    return len(counts)


def move_employee(
    connection: sqlite3.Connection, employee_id: int, new_manager_id: int | None
) -> None:
    """Change an employee's manager, refusing to create a cycle."""
    try:
        with connection:
            connection.execute("BEGIN IMMEDIATE")  # no one else can edit the chart now
            employees = load_employees(connection)
            if employee_id not in employees:
                raise EmployeeNotFoundError(f"employee {employee_id} does not exist")
            if new_manager_id is not None:
                above_new_manager = chain_of_command(employees, new_manager_id)
                if employee_id in above_new_manager:
                    raise OrgChartError(
                        f"{new_manager_id} is the employee or one of their reports"
                    )
            connection.execute(
                "UPDATE employee SET manager_id = ? WHERE id = ?",
                (new_manager_id, employee_id),
            )
    except sqlite3.Error:
        logger.exception("could not move employee %d", employee_id)
        raise
    logger.info("moved employee %d under %s", employee_id, new_manager_id)
```

```python
# test_org_chart.py
import sqlite3
from collections.abc import Iterator

import pytest

from org_chart import (
    EmployeeNotFoundError,
    OrgChartError,
    chain_of_command,
    count_reports,
    create_schema,
    load_employees,
    lowest_common_manager,
    move_employee,
    refresh_report_counts,
)

type Row = tuple[int, str, int | None]

ROWS: list[Row] = [
    (1, "Ceo", None),
    (2, "Vp one", 1),
    (3, "Vp two", 1),
    (4, "Lead one", 2),
    (5, "Lead two", 2),
    (6, "Engineer", 4),
]


def seed(connection: sqlite3.Connection, rows: list[Row]) -> None:
    connection.executemany(
        "INSERT INTO employee (id, name, manager_id) VALUES (?, ?, ?)", rows
    )
    connection.commit()


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    seed(conn, ROWS)
    yield conn
    conn.close()


def test_report_counts_include_indirect_reports(connection: sqlite3.Connection) -> None:
    counts = count_reports(load_employees(connection))
    assert counts == {1: 5, 2: 3, 3: 0, 4: 1, 5: 0, 6: 0}


def test_chain_of_command(connection: sqlite3.Connection) -> None:
    employees = load_employees(connection)
    assert chain_of_command(employees, 6) == [6, 4, 2, 1]
    assert chain_of_command(employees, 1) == [1]
    with pytest.raises(EmployeeNotFoundError):
        chain_of_command(employees, 99)


@pytest.mark.parametrize(
    ("first", "second", "expected"),
    [(6, 5, 2), (6, 3, 1), (6, 4, 4), (6, 6, 6), (3, 1, 1)],
)
def test_lowest_common_manager(
    connection: sqlite3.Connection, first: int, second: int, expected: int
) -> None:
    employees = load_employees(connection)
    assert lowest_common_manager(employees, first, second) == expected


def test_separate_trees_have_no_common_manager(
    connection: sqlite3.Connection,
) -> None:
    seed(connection, [(10, "Other boss", None), (11, "Other report", 10)])
    employees = load_employees(connection)
    with pytest.raises(OrgChartError):
        lowest_common_manager(employees, 6, 11)
    assert count_reports(employees)[10] == 1


def test_missing_manager_is_reported(connection: sqlite3.Connection) -> None:
    seed(connection, [(20, "Orphan", 99)])
    with pytest.raises(OrgChartError, match="unknown manager"):
        count_reports(load_employees(connection))


@pytest.mark.parametrize("bad_manager", [6, 1])
def test_cycles_are_detected(connection: sqlite3.Connection, bad_manager: int) -> None:
    seed(connection, [(30, "Self managed", 30)])
    with pytest.raises(OrgChartError, match="cycle"):
        count_reports(load_employees(connection))
    connection.execute("DELETE FROM employee WHERE id = 30")
    update = "UPDATE employee SET manager_id = ? WHERE id = 1"
    connection.execute(update, (bad_manager,))
    connection.commit()
    with pytest.raises(OrgChartError):
        count_reports(load_employees(connection))


def test_refresh_stores_the_counts(connection: sqlite3.Connection) -> None:
    assert refresh_report_counts(connection) == 6
    rows = connection.execute("SELECT id, reports FROM employee ORDER BY id").fetchall()
    assert rows == [(1, 5), (2, 3), (3, 0), (4, 1), (5, 0), (6, 0)]


def test_refresh_writes_nothing_for_an_invalid_chart(
    connection: sqlite3.Connection,
) -> None:
    connection.execute("UPDATE employee SET manager_id = 6 WHERE id = 1")
    connection.commit()
    with pytest.raises(OrgChartError):
        refresh_report_counts(connection)
    reports = connection.execute("SELECT SUM(reports) FROM employee").fetchone()[0]
    assert reports == 0


def test_moving_an_employee(connection: sqlite3.Connection) -> None:
    move_employee(connection, 4, 3)
    employees = load_employees(connection)
    assert chain_of_command(employees, 6) == [6, 4, 3, 1]
    move_employee(connection, 4, None)
    assert load_employees(connection)[4].manager_id is None


@pytest.mark.parametrize(("employee", "manager"), [(2, 6), (2, 2), (1, 5)])
def test_moves_that_would_create_a_cycle_are_rejected(
    connection: sqlite3.Connection, employee: int, manager: int
) -> None:
    with pytest.raises(OrgChartError):
        move_employee(connection, employee, manager)
    assert count_reports(load_employees(connection))[1] == 5


def test_unknown_ids_in_a_move(connection: sqlite3.Connection) -> None:
    with pytest.raises(EmployeeNotFoundError):
        move_employee(connection, 99, 1)
    with pytest.raises(EmployeeNotFoundError):
        move_employee(connection, 4, 99)


def test_a_very_deep_chart_needs_no_recursion() -> None:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    rows: list[Row] = [(1, "root", None)]
    rows += [(number, f"e{number}", number - 1) for number in range(2, 5_001)]
    seed(conn, rows)
    employees = load_employees(conn)
    assert count_reports(employees)[1] == 4_999
    assert len(chain_of_command(employees, 5_000)) == 5_000
    conn.close()
```

The last test builds a chain of 5,000 managers, far deeper than Python's default recursion limit of about 1,000, and it passes because nothing in the algorithm recurses. The cycle tests cover the two ways bad data can break the tree property, and none of them can hang the program.

## Common mistakes

- Forgetting that the diameter or the best path may not pass through the root, and checking only the root.
- Returning the bent path upward in maximum path sum, which lets a parent extend a path that already uses two branches.
- Treating the empty path as allowed in maximum path sum, so an all-negative tree returns zero instead of the largest single value.
- Appending the shared `path` list to the results without copying it.
- Forgetting to undo changes (a path entry, a prefix-sum count) when leaving a node in backtracking.
- Calling the LCA function with a value that is not in the tree, and getting a wrong answer without an error.
- Using the BST shortcut for LCA on a tree that is not a binary search tree.
- Serializing without markers for missing children, which makes different trees produce the same string.
- Deserializing untrusted data without checking for truncation, extra data, or malformed tokens.
- Recursing over trees whose depth comes from user data or a database, instead of using an explicit stack or a depth limit.
- Trusting a `parent_id` column to form a tree, and not checking for missing parents and cycles.
- Checking for a cycle after making a change instead of before, so the bad data is already stored.
- Making two separate decisions on the same data in separate transactions, which lets two concurrent edits combine into a cycle.

## Practice exercises

1. Write `max_depth(root)` and `min_depth(root)` and `is_balanced(root)` using the "return a value, record a side result" pattern, and test them on skewed, balanced, and empty trees.
2. Write `lca_with_parents(node_a, node_b)` for nodes that have parent pointers, by walking up with depth counts and no extra memory.
3. Write `distance_between_nodes(root, a, b)` that returns the number of edges on the path between two values, using the LCA and depth.
4. Write `longest_univalue_path(root)` that returns the length of the longest path in which every node has the same value, using the diameter pattern.
5. Write `binary_tree_cameras(root)` that returns the minimum number of cameras needed to monitor every node, where a camera covers its parent, itself, and its children, using tree DP with three states.
6. Write `max_independent_set(root)` for a general binary tree and return the chosen values, not only the total, by also tracking which choice each subtree made.
7. Write `all_paths_sum_to_target(root, target)` that returns root-to-leaf paths as lists of values, using an explicit stack of `(node, path)` pairs and no recursion.
8. Write `serialize_level_order(root)` and `deserialize_level_order(data)` using the level-order format with `null` markers, and compare their output with the preorder format.
9. Write `serialize_nary(root)` and `deserialize_nary(data)` using a format of `value:child_count` in preorder, and validate the input.
10. Write `directory_sizes(entries)` that takes `(path, size)` pairs for files and returns the total size of every directory, using a dictionary of path prefixes and a bottom-up pass.
11. Write `find_duplicate_subtrees(root)` that returns the roots of every subtree that appears more than once, by serializing each subtree and counting the serialized strings.
12. Write `is_subtree(root, sub_root)` with a serialization-based check and with a direct comparison, and compare their costs.
13. Write `kth_ancestor(employees, employee_id, k)` with binary lifting after preprocessing, and compare its query time with the naive walk on a chain of 100,000 employees.
14. Extend `org_chart` with `add_employee(connection, employee_id, name, manager_id)` that inserts in a transaction, checks that the manager exists, and returns the new chain of command. Add tests for an unknown manager and a duplicate id.

## Summary

Most tree algorithms are one pattern: process a node by combining small summaries returned from its children, and pass a summary up to the parent. The decision is what to return and what to record on the side. The LCA is the node where two targets split, found by recursion in a general tree or by following the ordering in a BST. The diameter and the maximum path sum record a best path that bends at each node while returning only an extendable downward path. Path sum problems use backtracking with an undo step, and counting downward paths uses a dictionary of prefix sums along the current path. Preorder with markers for missing children serializes a tree uniquely, and a deserializer must reject malformed input. `N`-ary trees carry over the same traversals, and tree DP keeps a small table of states per node. For hierarchies stored in databases, validate the tree property (existing parents, no cycles), prefer an explicit stack to recursion for depth you do not control, and make updates that could create a cycle atomic.
