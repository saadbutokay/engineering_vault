## Overview

A function is recursive when it calls itself to solve a smaller version of the same problem. Recursion is a way of thinking: instead of describing how to do all the work in a loop, you describe how to do a little work and trust the function to handle the rest. Trees, nested folders, nested JSON, expression parsers, and many sorting and searching algorithms are naturally recursive, because the data itself is made of smaller copies of the same shape.

Recursion has a cost that loops do not. Every call needs memory to remember where it was, and Python limits how deep calls may nest. So an engineer needs two skills: writing recursive solutions clearly, and knowing when and how to convert them into loops with an explicit stack.

This note covers thinking recursively, base and recursive cases, the call stack and stack frames, recursion trees, deriving complexity from recurrences, the Master theorem, the Python recursion limit, converting recursion to iteration, and an introduction to memoization.

## Core concepts

### Thinking recursively

Solve a problem recursively by answering three questions.

1. What is the smallest version of the problem that I can answer directly? That is the base case.
2. How can I reduce a bigger problem to one or more smaller ones? That is the recursive case.
3. How do I combine the answers to the smaller problems into the answer to the bigger one?

Then trust the recursive call. Do not try to trace the whole chain of calls in your head. If the base case is right, each step makes progress toward it, and the combination step is right, the whole function is right. This is proof by induction in code form.

### Base case and recursive case

- Base case: returns without calling itself. Without one, recursion never stops.
- Recursive case: calls itself with an argument that is closer to a base case (a smaller number, a shorter list, a node closer to a leaf).

Two bugs cause most recursion failures: a missing or unreachable base case, and a recursive call whose argument does not get smaller.

### The call stack and stack frames

Each function call creates a stack frame: a block of memory holding the call's arguments, local variables, and the place to return to. Frames are stored in a stack. A new call pushes a frame. When a call returns, its frame is popped and execution continues in the caller. Recursive calls stack up until the base case is reached, then return one by one in reverse order.

### Recursion trees

Draw each call as a node and each recursive call as a child. For a function that makes one recursive call, the tree is a single chain. For one that makes two calls, such as the naive Fibonacci, it is a branching tree, and the number of nodes shows the total work. The height of the tree is the maximum depth of the call stack.

### Recurrences and complexity

The running time of a recursive function can be written as a recurrence: an equation for the cost on size `n` in terms of the cost on smaller sizes.

- `T(n) = T(n - 1) + O(1)`: one call on size `n - 1` plus constant work. Solves to `O(n)`.
- `T(n) = T(n / 2) + O(1)`: halve each time. Solves to `O(log n)`.
- `T(n) = 2T(n / 2) + O(n)`: two halves plus linear merge work. Solves to `O(n log n)`.
- `T(n) = 2T(n - 1) + O(1)`: two calls on a slightly smaller input. Solves to `O(2^n)`.

### The Master theorem

For recurrences of the form `T(n) = a * T(n / b) + f(n)`, where a problem is split into `a` subproblems of size `n / b` and `f(n)` is the work to split and combine, compare `f(n)` with `n^c` where `c = log_b(a)`:

- If `f(n)` grows slower than `n^c`, the leaves dominate: `T(n) = Θ(n^c)`.
- If `f(n)` grows at the same rate as `n^c`, every level costs about the same: `T(n) = Θ(n^c log n)`.
- If `f(n)` grows faster than `n^c` (and is regular enough), the top level dominates: `T(n) = Θ(f(n))`.

### The recursion limit in Python

CPython stops a runaway recursion by raising `RecursionError` when the depth exceeds a limit, 1,000 by default. Python does not optimize tail calls, so even a recursive function that does nothing after the call still uses a new frame each time.

### Converting recursion to iteration

Any recursion can be turned into a loop by managing the stack yourself: keep a list of pending work, pop an item, process it, and push the smaller pieces. This removes the depth limit and is how production code traverses deep or untrusted structures.

### Memoization introduced

When a recursive function solves the same subproblem many times, store each answer the first time and reuse it. This is memoization. In Python, `functools.cache` does it with one line. It turns the exponential naive Fibonacci into a linear computation, and it is the first step toward dynamic programming.

## Basic syntax examples

### Factorial, with the call stack

```python
def factorial(n: int) -> int:
    """n! = n * (n - 1)!. Time O(n), stack depth O(n)."""
    if n < 0:
        raise ValueError("n must not be negative")
    if n == 0:  # base case
        return 1
    return n * factorial(n - 1)  # recursive case


print(factorial(5))  # 120
```

What the stack looks like for `factorial(3)`:

```text
factorial(3)
  factorial(2)
    factorial(1)
      factorial(0)      returns 1   (base case)
    returns 1 * 1 = 1
  returns 2 * 1 = 2
returns 3 * 2 = 6
```

### Summing a list without slicing

```python
def sum_list(values: list[int], index: int = 0) -> int:
    """Pass an index instead of slicing, so each call is O(1)."""
    if index == len(values):  # base case: nothing left
        return 0
    return values[index] + sum_list(values, index + 1)


print(sum_list([4, 8, 15, 16, 23, 42]))  # 108
```

### Reversing a string

```python
def reverse(text: str) -> str:
    if len(text) <= 1:
        return text
    return reverse(text[1:]) + text[0]  # slicing and joining copy: O(n^2) overall


print(reverse("recursion"))  # noisrucer
```

This is a good way to see recursion, and a poor way to reverse a string in real code, because each level copies. `text[::-1]` is `O(n)`.

### Fast power: halving the problem

```python
def power(base: float, exponent: int) -> float:
    """base ** exponent in O(log exponent) multiplications."""
    if exponent < 0:
        return 1 / power(base, -exponent)
    if exponent == 0:
        return 1.0
    half = power(base, exponent // 2)  # one call on half the size
    if exponent % 2 == 0:
        return half * half
    return half * half * base


print(power(2, 10))  # 1024.0
print(power(2, -2))  # 0.25
```

### Naive Fibonacci and its recursion tree

```python
def fib(n: int) -> int:
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)


def fib_calls(n: int) -> tuple[int, int]:
    """Return (fib(n), how many calls were made)."""
    if n < 2:
        return n, 1
    first, first_calls = fib_calls(n - 1)
    second, second_calls = fib_calls(n - 2)
    return first + second, first_calls + second_calls + 1


print(fib_calls(10))  # (55, 177)
print(fib_calls(20))  # (6765, 21891)
```

The number of calls grows by about 1.6 times each time `n` increases by one. The tree for `fib(5)`:

```text
                      fib(5)
               /                \
          fib(4)                fib(3)
         /      \              /      \
     fib(3)    fib(2)      fib(2)    fib(1)
     /   \      /   \       /   \
 fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
  /   \
fib(1) fib(0)
```

`fib(3)` is computed twice and `fib(2)` three times. The tree repeats the same subproblems, which is exactly what memoization removes.

### Memoization with functools.cache

```python
from functools import cache


@cache
def fib_cached(n: int) -> int:
    if n < 2:
        return n
    return fib_cached(n - 1) + fib_cached(n - 2)


print(fib_cached(80))  # 23416728348467685, instant
print(fib_cached.cache_info().currsize)  # 81 stored results
```

### Towers of Hanoi: recursion with two calls

```python
type Move = tuple[str, str]


def hanoi(n: int, source: str, target: str, spare: str, moves: list[Move]) -> None:
    """Move n discs from source to target. Makes 2**n - 1 moves."""
    if n == 0:
        return
    hanoi(n - 1, source, spare, target, moves)  # clear the way
    moves.append((source, target))  # move the biggest disc
    hanoi(n - 1, spare, target, source, moves)  # put the rest back on top


moves: list[Move] = []
hanoi(3, "A", "C", "B", moves)
print(len(moves), moves[:3])  # 7 [('A', 'C'), ('A', 'B'), ('C', 'B')]
```

### Recursion over nested data

```python
type Nested = int | list[Nested]


def flatten(items: list[Nested]) -> list[int]:
    """Flatten arbitrarily nested lists of integers."""
    result: list[int] = []
    for item in items:
        if isinstance(item, list):
            result.extend(flatten(item))  # the nested list is a smaller instance
        else:
            result.append(item)
    return result


print(flatten([1, [2, [3, 4], 5], [[6]]]))  # [1, 2, 3, 4, 5, 6]
```

### Hitting the recursion limit

```python
import sys


def depth(n: int) -> int:
    return 0 if n == 0 else 1 + depth(n - 1)


print(sys.getrecursionlimit())  # 1000
try:
    depth(5_000)
except RecursionError as error:
    print(type(error).__name__)  # RecursionError
```

### The same flatten with an explicit stack

```python
from collections.abc import Iterator


def flatten_iterative(items: list[Nested]) -> list[int]:
    """No recursion: a stack of iterators remembers where each list left off."""
    result: list[int] = []
    stack: list[Iterator[Nested]] = [iter(items)]
    while stack:
        for item in stack[-1]:
            if isinstance(item, list):
                stack.append(iter(item))  # go deeper; resume the outer list later
                break
            result.append(item)
        else:  # the inner iterator is exhausted
            stack.pop()
    return result


deep: Nested = 0
for _ in range(10_000):
    deep = [deep]  # nested 10,000 levels deep
print(flatten_iterative([deep]))  # [0], where the recursive version would fail
```

## Deeper behavior

### What a stack frame costs

Every call allocates a frame that holds the arguments, the local variables, and a reference to the code. Frames take memory, and creating them takes time, so a recursive solution is slower than an equivalent loop by a constant factor and uses `O(depth)` stack memory. In the recursive `sum_list`, time is `O(n)` but stack space is `O(n)` too, while the loop uses `O(1)` space. When you analyze a recursive function, always report the space used by the stack, which is the depth of the recursion tree.

### Why Python has a recursion limit

A deep recursion could exhaust memory or crash the interpreter. The limit, found with `sys.getrecursionlimit()`, turns that into a catchable `RecursionError`. You can raise it with `sys.setrecursionlimit(n)`, but a deeper limit only moves the cliff. Very deep recursion can still crash the process when the underlying C stack runs out, and a crash cannot be caught. Use a higher limit only for a known, bounded depth. Otherwise convert the recursion.

Python does not eliminate tail calls. A function written in tail-recursive style, where the recursive call is the last thing it does, still consumes one frame per call, so the style that is free in some languages is not free here. Write a loop instead.

### Reading a recursion error

A `RecursionError` traceback repeats the same line many times. The repeated line is almost always where the base case is missing or unreachable, or where the argument does not shrink. Look at the argument passed to the recursive call and check that it moves toward the base case in every branch.

### Solving recurrences with the recursion tree

For `T(n) = 2T(n / 2) + n`, sum the work level by level.

```text
level 0:  1 problem of size n        -> n work
level 1:  2 problems of size n/2     -> n work in total
level 2:  4 problems of size n/4     -> n work in total
...
level k:  2^k problems of size n/2^k -> n work in total
The tree has log2(n) levels, each doing n work, so T(n) = n log n.
```

For `T(n) = T(n - 1) + 1`, the tree is a chain of `n` nodes each doing constant work, so `T(n) = n`. For `T(n) = T(n - 1) + n`, the work is `n + (n - 1) + ... + 1`, which is `n(n + 1) / 2`, so `T(n) = Θ(n^2)`.

### Applying the Master theorem

```text
Binary search:    T(n) = T(n/2) + 1      a=1, b=2, c=0, f(n)=1=n^0        -> Theta(log n)
Merge sort:       T(n) = 2T(n/2) + n     a=2, b=2, c=1, f(n)=n=n^1        -> Theta(n log n)
Karatsuba:        T(n) = 3T(n/2) + n     a=3, b=2, c=1.58, f(n) smaller   -> Theta(n^1.58)
Tree traversal:   T(n) = 2T(n/2) + 1     a=2, b=2, c=1, f(n)=1 smaller    -> Theta(n)
Quickselect avg:  T(n) = T(n/2) + n      a=1, b=2, c=0, f(n)=n larger     -> Theta(n)
```

The theorem needs equal-sized subproblems that shrink by a constant factor. It does not apply to `T(n) = T(n - 1) + f(n)` or to `T(n) = 2T(n - 1) + 1`. Use the recursion tree for those.

### Converting recursion to iteration

There are three common patterns.

- Accumulator loop: when the recursion just reduces `n` by one (`factorial`, `sum_list`), a `for` loop with a running result is simpler and faster.
- Explicit stack: when the recursion branches (tree and graph traversals), keep a list of nodes still to process. Push the children and pop the next one. The order of pushing decides the traversal order. Pushing children in reverse order and popping from the end visits them in the original order.
- Bottom-up: when the recursion has overlapping subproblems, fill a table from the smallest subproblems upward. This is tabulation, covered with dynamic programming.

The explicit-stack version uses the heap, which can hold millions of items, instead of the call stack, which is limited to about a thousand frames.

### Memoization in detail

`functools.cache` stores results in a dictionary keyed by the arguments, so the arguments must be hashable. Lists must be converted to tuples. The cache grows without limit, so use `lru_cache(maxsize=...)` for long-running services. The first call still recurses to full depth, so `fib_cached(5_000)` raises `RecursionError` even though each value is cached. A bottom-up loop avoids both problems:

```python
def fib_iterative(n: int) -> int:
    previous, current = 0, 1
    for _ in range(n):
        previous, current = current, previous + current
    return previous


print(fib_iterative(10))  # 55
```

Memoization works only for pure functions, whose result depends only on their arguments. A function that reads the clock, a database, or global state must not be cached this way.

### Designing for recursion safety

- State the maximum depth you expect and test with a depth well beyond it.
- Never recurse on untrusted input depth, such as nested JSON from users, without a depth guard or an explicit stack. A deeply nested payload can be used to crash a service.
- Prefer passing indexes to slicing, so each call costs `O(1)` instead of copying.
- Mutable arguments (lists, sets) shared across calls can hold state from sibling calls. Copy on the way in, or undo changes on the way out.

## Industry-level snippet

An online shop stores product categories in a table where each row points to its parent: Electronics contains Phones, which contains Android. A category tree is recursive data, and the operations on it are recursion or iteration over it: list all descendants, render a menu, and move a category under a new parent. The dangerous operation is the move. If an admin drags Electronics under Android, the tree would contain a cycle and every traversal would loop forever, so a move must reject any new parent that lies inside the moved category's own subtree.

The design follows the lessons above. Descendants are found with an explicit stack, so a chain of thousands of nested categories (bad data or an attack) cannot crash the service. The recursive renderer is kept because it reads clearly, but it has a depth guard that raises a clear error long before Python's limit. The move runs in one transaction that takes the write lock first, checks for cycles against the current data, and updates the row.

```python
# category_tree.py
import logging
import sqlite3
from collections import defaultdict
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS category (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id INTEGER REFERENCES category (id)
)
"""

MAX_RENDER_DEPTH = 50


class CategoryNotFoundError(LookupError):
    """Raised when a category id does not exist."""


class CategoryCycleError(ValueError):
    """Raised when a move would make a category its own ancestor."""


class TreeTooDeepError(ValueError):
    """Raised when a tree is nested deeper than the renderer allows."""


@dataclass(frozen=True, slots=True)
class Category:
    id: int
    name: str
    parent_id: int | None


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def load_categories(connection: sqlite3.Connection) -> dict[int, Category]:
    rows = connection.execute("SELECT id, name, parent_id FROM category ORDER BY id")
    return {row[0]: Category(row[0], row[1], row[2]) for row in rows}


def build_children(categories: dict[int, Category]) -> dict[int | None, list[int]]:
    children: defaultdict[int | None, list[int]] = defaultdict(list)
    for category in categories.values():
        children[category.parent_id].append(category.id)
    return children


def descendants(children: dict[int | None, list[int]], root_id: int) -> list[int]:
    """All descendants in pre-order, with an explicit stack: safe at any depth."""
    result: list[int] = []
    stack = list(reversed(children.get(root_id, [])))
    while stack:
        current = stack.pop()
        result.append(current)
        stack.extend(reversed(children.get(current, [])))
    return result


def render_tree(
    categories: dict[int, Category],
    children: dict[int | None, list[int]],
    parent_id: int | None = None,
    depth: int = 0,
) -> list[str]:
    """Recursive and readable, with a depth guard far below Python's limit."""
    if depth > MAX_RENDER_DEPTH:
        raise TreeTooDeepError(f"categories are nested deeper than {MAX_RENDER_DEPTH}")
    lines: list[str] = []
    for child_id in children.get(parent_id, []):
        lines.append("  " * depth + categories[child_id].name)
        lines.extend(render_tree(categories, children, child_id, depth + 1))
    return lines


def ancestors(categories: dict[int, Category], category_id: int) -> list[int]:
    """Parents from nearest to the root. Stops if existing data contains a cycle."""
    path: list[int] = []
    current = categories[category_id].parent_id
    while current is not None:
        if current in path:
            raise CategoryCycleError(f"category {current} is part of a cycle")
        path.append(current)
        current = categories[current].parent_id
    return path


def move_category(
    connection: sqlite3.Connection, category_id: int, new_parent_id: int | None
) -> None:
    """Move a category under a new parent, or to the top level with None."""
    try:
        with connection:
            connection.execute("BEGIN IMMEDIATE")  # no one else can change the tree now
            categories = load_categories(connection)
            if category_id not in categories:
                raise CategoryNotFoundError(f"category {category_id} does not exist")
            if new_parent_id is not None:
                if new_parent_id not in categories:
                    raise CategoryNotFoundError(f"category {new_parent_id} not found")
                moving_into_itself = new_parent_id == category_id
                if moving_into_itself or category_id in ancestors(
                    categories, new_parent_id
                ):
                    raise CategoryCycleError(
                        f"cannot move {category_id} under its own descendant"
                    )
            connection.execute(
                "UPDATE category SET parent_id = ? WHERE id = ?",
                (new_parent_id, category_id),
            )
    except sqlite3.Error:
        logger.exception("could not move category %s", category_id)
        raise
    logger.info("moved category %s under %s", category_id, new_parent_id)
```

```python
# test_category_tree.py
import sqlite3
from collections.abc import Iterator

import pytest

from category_tree import (
    CategoryCycleError,
    CategoryNotFoundError,
    TreeTooDeepError,
    build_children,
    create_schema,
    descendants,
    load_categories,
    move_category,
    render_tree,
)


def add(
    connection: sqlite3.Connection, category_id: int, name: str, parent: int | None
) -> None:
    connection.execute(
        "INSERT INTO category (id, name, parent_id) VALUES (?, ?, ?)",
        (category_id, name, parent),
    )


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    add(conn, 1, "Electronics", None)
    add(conn, 2, "Phones", 1)
    add(conn, 3, "Laptops", 1)
    add(conn, 4, "Android", 2)
    add(conn, 5, "Books", None)
    conn.commit()
    yield conn
    conn.close()


def test_descendants_are_in_pre_order(connection: sqlite3.Connection) -> None:
    children = build_children(load_categories(connection))
    assert descendants(children, 1) == [2, 4, 3]
    assert descendants(children, 5) == []


def test_render_tree_indents_by_depth(connection: sqlite3.Connection) -> None:
    categories = load_categories(connection)
    lines = render_tree(categories, build_children(categories))
    assert lines == ["Electronics", "  Phones", "    Android", "  Laptops", "Books"]


def test_move_updates_the_parent(connection: sqlite3.Connection) -> None:
    move_category(connection, 2, 5)
    assert load_categories(connection)[2].parent_id == 5
    move_category(connection, 2, None)
    assert load_categories(connection)[2].parent_id is None


@pytest.mark.parametrize("new_parent", [1, 2, 4])
def test_moving_into_own_subtree_is_rejected(
    connection: sqlite3.Connection, new_parent: int
) -> None:
    with pytest.raises(CategoryCycleError):
        move_category(connection, 1, new_parent)
    assert load_categories(connection)[1].parent_id is None


def test_unknown_ids_are_rejected(connection: sqlite3.Connection) -> None:
    with pytest.raises(CategoryNotFoundError):
        move_category(connection, 99, 1)
    with pytest.raises(CategoryNotFoundError):
        move_category(connection, 1, 99)


def test_very_deep_trees_do_not_crash_the_iterative_code(
    connection: sqlite3.Connection,
) -> None:
    for category_id in range(100, 5_100):
        parent = None if category_id == 100 else category_id - 1
        add(connection, category_id, f"c{category_id}", parent)
    connection.commit()
    children = build_children(load_categories(connection))
    assert len(descendants(children, 100)) == 4_999


def test_renderer_reports_excess_depth_instead_of_recursion_error() -> None:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    for category_id in range(1, 201):
        parent = None if category_id == 1 else category_id - 1
        add(conn, category_id, f"c{category_id}", parent)
    categories = load_categories(conn)
    with pytest.raises(TreeTooDeepError):
        render_tree(categories, build_children(categories))
    conn.close()
```

Notice that `test_moving_into_own_subtree_is_rejected` moves category 1 under itself, under its child 2, and under its grandchild 4. Each attempt must leave the tree unchanged.

## Common mistakes

- Forgetting the base case, or writing one that is never reached, which ends in `RecursionError`.
- A recursive call whose argument does not get smaller in every branch.
- Slicing lists or strings in each call, which copies the data and turns a linear idea into a quadratic one.
- Recursing on untrusted, arbitrarily deep input without a limit or an explicit stack.
- Raising `sys.setrecursionlimit` to hide a missing base case instead of fixing it.
- Using naive recursion for overlapping subproblems (Fibonacci, grid paths) and wondering why it takes minutes. Memoize or tabulate.
- Memoizing a function that depends on global state, time, or randomness, or whose arguments are unhashable.
- Mutating a shared list across recursive calls and forgetting to undo the change.
- Forgetting to count the call stack when stating space complexity.
- Believing Python optimizes tail recursion. It does not.
- Using a mutable default argument such as `results=[]`, which is shared between calls. Use `None` and create the list inside.

## Practice exercises

1. Write `count_down(n)` that returns a list from `n` to 1 recursively, then iteratively. Compare their stack usage.
2. Write `digit_sum(n)` for a non-negative integer using `n // 10` and `n % 10`, and trace `digit_sum(493)` by hand.
3. Write `is_palindrome(text)` recursively using indexes `left` and `right`, without slicing, and state its time and space complexity.
4. Write `gcd(a, b)` recursively using Euclid's rule, and draw its call stack for `gcd(48, 18)`.
5. Write `binary_search_recursive(values, target, low, high)` and say what the recurrence and its solution are.
6. Write `generate_parentheses(n)` that returns all balanced strings of `n` pairs, using recursion that tracks how many open and close parentheses remain.
7. Write `subsets(items)` that returns all subsets of a list recursively. Count how many calls are made for `n = 10` and relate it to `2^n`.
8. Write `count_paths(rows, cols)` that counts the ways to move from the top-left to the bottom-right of a grid using only right and down moves, recursively. Then add `@cache` and compare the number of calls for a 15 by 15 grid.
9. Write `max_depth(nested)` for arbitrarily nested lists, once recursively and once with an explicit stack, and test it with a nesting depth of 50,000.
10. Use the Master theorem or a recursion tree to solve `T(n) = 4T(n / 2) + n`, `T(n) = T(n / 2) + n`, and `T(n) = 2T(n / 2) + n^2`. State the case you used.
11. Write `sum_to_n(n)` recursively and find the largest `n` that works on your machine before `RecursionError`. Then rewrite it as a loop.
12. Write a function `walk_directory(path)` over a real folder using `os.scandir`, once recursively and once with an explicit stack, returning all file paths. Compare them on a deeply nested test folder you create.
13. Write `count_descendants(children, root)` for the category tree using memoization so that each node's count is computed once, and explain why this matters when the same subtree is queried repeatedly.
14. Extend `move_category` so that it also refuses to move a category when the resulting depth would exceed `MAX_RENDER_DEPTH`, and add tests.

## Summary

A recursive function solves a problem by solving smaller copies of it, and it is correct when it has a reachable base case, makes progress toward it, and combines the results correctly. Each call uses a stack frame, so recursion costs stack space proportional to its depth, and Python raises `RecursionError` past about a thousand nested calls and does not optimize tail calls. Recursion trees and recurrences give the running time, and the Master theorem solves the common divide-and-conquer shapes. Slicing in recursive calls hides copies, so pass indexes. When depth is unbounded or the input is untrusted, convert the recursion to a loop with an explicit stack. When subproblems repeat, memoize with `functools.cache`, remembering that it keeps results forever and still recurses to full depth on the first call.
