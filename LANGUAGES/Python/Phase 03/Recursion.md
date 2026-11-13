## Overview

**Recursion** is when a function calls itself. It sounds like a trick that must go on forever, yet it is one of the most natural ways to solve certain problems: anything whose structure contains smaller copies of itself. A folder contains folders. A JSON document contains documents. A family tree contains family trees. A sorted list can be searched by searching half of it. For such problems, a recursive function mirrors the shape of the data and is often much shorter than the alternative.

Recursion also has costs. Every call takes memory, Python limits how deep calls can go, and careless recursion can be astonishingly slow. This chapter teaches you to write recursive functions correctly, to recognize when they blow up, and to choose iteration when it is the better tool.

By the end of this note you will be able to:

- Write a recursive function with a correct base case and recursive case.
- Trace a recursive call by hand and picture the call stack.
- Explain the recursion limit and the `RecursionError`.
- Fix exponential slowness with memoization.
- Convert a recursion into a loop with an explicit stack.
- Decide when recursion is the right tool.

> [!note] Recursion is thinking, not syntax
> To solve a problem recursively, say: "If I already had the answer to a slightly smaller version of this problem, how would I get the answer to the whole thing?" Then also say: "What is the smallest version, which I can answer directly?"

## Core concepts

### The two parts

Every correct recursive function has two parts:

| Part | Meaning |
| --- | --- |
| **Base case** | A situation simple enough to answer directly, without another call. It stops the recursion |
| **Recursive case** | The function calls itself on a **smaller** version of the problem, then uses that answer to build its own |

Without a base case, the calls never stop. Without progress toward the base case, the same thing happens.

```python
def factorial(n):
    if n <= 1:              # base case
        return 1
    return n * factorial(n - 1)    # recursive case: a smaller problem
```

### The call stack

Each call to a function gets its own **frame** holding its parameters and local variables (Chapter 3.1). Frames are stacked: a function that calls another waits while the new frame runs on top. A recursive call stacks one more frame, so a recursion of depth 1000 holds 1000 frames at once.

```text
factorial(4)
= 4 * factorial(3)
= 4 * (3 * factorial(2))
= 4 * (3 * (2 * factorial(1)))
= 4 * (3 * (2 * 1))
= 24
```

The calls go down until the base case is reached, then the answers come back up, each multiplied into the pending result.

### The recursion limit

Python caps the depth of nested calls, by default at about **1000**, to protect the interpreter from running out of memory. Exceeding it raises `RecursionError`. You can read the limit with `sys.getrecursionlimit()`.

### Common recursive patterns

| Pattern | Idea | Example |
| --- | --- | --- |
| Reduce by one | Solve for n using the answer for n minus 1 | Factorial |
| Divide and conquer | Split into halves, solve each, combine | Binary search, merge sort |
| Traverse a nested structure | Handle one level, recurse into each child | Directory trees, JSON, menus |
| Backtracking | Try a choice, recurse, undo it | Puzzles, permutations, search |

### Memoization

Some recursive functions solve the same smaller problem many times. **Memoization** stores each answer the first time so that later calls reuse it (Chapter 3.7). It can turn an impossibly slow function into a fast one.

### Recursion versus iteration

Anything recursive can be written with a loop, usually with an explicit **stack** (a list used as one) to remember pending work. The choice is about clarity, depth, and speed.

| | Recursion | Iteration |
| --- | --- | --- |
| Best for | Tree-shaped or nested data, divide and conquer | Simple repetition, very large depth |
| Readability | Often shorter and closer to the problem | Straightforward for linear tasks |
| Memory | One frame per level | Constant, or an explicit stack |
| Depth limit | About 1000 in Python | None |
| Speed | Function calls cost a little | Usually a little faster |

## Basic syntax examples

### Factorial

```text
>>> def factorial(n):
...     if n < 0:
...         raise ValueError("n must not be negative")
...     if n <= 1:
...         return 1
...     return n * factorial(n - 1)
...
>>> factorial(5)
120
>>> factorial(0)
1
>>> factorial(-1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: n must not be negative
```

Notice the guard against negative input. Without it, `factorial(-1)` would never reach the base case.

### Watching the calls

```python
def factorial(n, depth=0):
    indent = "  " * depth
    print(f"{indent}factorial({n})")
    result = 1 if n <= 1 else n * factorial(n - 1, depth + 1)
    print(f"{indent}-> {result}")
    return result


factorial(3)
```

```text
factorial(3)
  factorial(2)
    factorial(1)
    -> 1
  -> 2
-> 6
```

The indentation shows the depth. Each line starting with an arrow is an answer returning back up the stack.

### Summing a list

```python
def total(numbers):
    if not numbers:                         # base case: an empty list sums to 0
        return 0
    return numbers[0] + total(numbers[1:])  # first item plus the sum of the rest


print(total([4, 8, 15, 16]))
```

```text
43
```

This is clear but inefficient: each `numbers[1:]` copies the rest of the list. In real code, use `sum()`. The example is here to show the shape of the idea.

### Power by repeated halving

```python
def power(base, exponent):
    if exponent == 0:
        return 1
    half = power(base, exponent // 2)
    if exponent % 2 == 0:
        return half * half
    return half * half * base


print(power(2, 10), power(3, 5), power(2, 100) == 2 ** 100)
```

```text
1024 243 True
```

Each call halves the exponent, so computing `base ** n` takes only about log n multiplications instead of n. This is **divide and conquer**.

### Binary search

```python
def binary_search(items, target, low=0, high=None):
    """Return the index of target in the sorted list items, or -1."""
    if high is None:
        high = len(items) - 1
    if low > high:                  # base case: nothing left to search
        return -1
    middle = (low + high) // 2
    if items[middle] == target:
        return middle
    if items[middle] < target:
        return binary_search(items, target, middle + 1, high)
    return binary_search(items, target, low, middle - 1)


numbers = [1, 3, 5, 7, 9, 11]
print(binary_search(numbers, 7), binary_search(numbers, 4))
```

```text
3 -1
```

Each call discards half of the remaining range. Chapter 2.7's `bisect` module provides the ready-made version.

### Nested data: flattening

```python
def flatten(items):
    flat = []
    for item in items:
        if isinstance(item, list):
            flat.extend(flatten(item))      # recurse into nested lists
        else:
            flat.append(item)               # base case: a plain value
    return flat


print(flatten([1, [2, [3, 4]], 5, [[6]]]))
```

```text
[1, 2, 3, 4, 5, 6]
```

The function does not need to know how deeply the lists are nested. It handles one level and trusts itself for the rest.

### Walking a tree

```python
org = {
    "name": "Ada",
    "reports": [
        {"name": "Grace", "reports": [{"name": "Linus", "reports": []}]},
        {"name": "Ken", "reports": []},
    ],
}


def print_tree(node, depth=0):
    print("  " * depth + node["name"])
    for child in node["reports"]:
        print_tree(child, depth + 1)


print_tree(org)
```

```text
Ada
  Grace
    Linus
  Ken
```

A node with no reports is the base case: its loop does nothing, so no further calls are made.

### A missing base case

```python
def count_down(n):
    return count_down(n - 1)


count_down(5)
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 4, in <module>
    count_down(5)
  File "/home/learner/app.py", line 2, in count_down
    return count_down(n - 1)
           ^^^^^^^^^^^^^^^^^
  File "/home/learner/app.py", line 2, in count_down
    return count_down(n - 1)
           ^^^^^^^^^^^^^^^^^
  File "/home/learner/app.py", line 2, in count_down
    return count_down(n - 1)
           ^^^^^^^^^^^^^^^^^
  [Previous line repeated 996 more times]
RecursionError: maximum recursion depth exceeded
```

Python collapses the repeated frames into one line. A `RecursionError` almost always means a missing or unreachable base case, or input too deep to process recursively.

```text
>>> import sys
>>> sys.getrecursionlimit()
1000
```

### The Fibonacci trap

```python
calls = 0


def slow_fib(n):
    global calls
    calls += 1
    return n if n < 2 else slow_fib(n - 1) + slow_fib(n - 2)


print(slow_fib(25), calls)
```

```text
75025 242785
```

Computing the 25th Fibonacci number took **242,785 calls**, because the same values are recomputed over and over. The number of calls roughly multiplies by 1.6 for each increase in `n`, so `slow_fib(50)` would need tens of billions.

### Memoization to the rescue

```python
from functools import lru_cache


@lru_cache(maxsize=None)
def fast_fib(n):
    return n if n < 2 else fast_fib(n - 1) + fast_fib(n - 2)


print(fast_fib(25))
print(fast_fib.cache_info())
```

```text
75025
CacheInfo(hits=23, misses=26, maxsize=None, currsize=26)
```

Now the function body runs only 26 times, once for each value from 0 to 25. The same recursion, with the answers remembered, is **linear** instead of exponential.

### The same problem as a loop

```python
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a


print(fib(25), fib(500) > 10 ** 100)
```

```text
75025 True
```

For simple linear recurrences like this one, the loop is the clearest and cheapest solution.

### Replacing recursion with an explicit stack

```python
def total_nested(items):
    total = 0
    stack = [items]                 # work still to be done
    while stack:
        current = stack.pop()
        for item in current:
            if isinstance(item, list):
                stack.append(item)  # remember to look inside later
            else:
                total += item
    return total


print(total_nested([1, [2, [3, 4]], 5]))
```

```text
15
```

The program's own call stack is replaced by a list we manage ourselves. There is no recursion limit, because there are no recursive calls.

## Deeper behavior

### Why there is a limit

Each frame uses memory. Unbounded recursion would exhaust it and crash the process. The limit turns that crash into a catchable `RecursionError`. You can raise it with `sys.setrecursionlimit(n)`, but a high setting risks overflowing the underlying C stack and crashing the whole interpreter without any error message. If you need more than a few thousand levels, switch to iteration instead.

### Python does not eliminate tail calls

A call is a **tail call** if its result is returned directly, as in `return count_down(n - 1)`. Some languages reuse the current frame for tail calls, making them as cheap as loops. Python deliberately does not, partly to keep tracebacks complete. A tail-recursive function in Python still uses one frame per call.

### Cost of slicing in recursion

`total(numbers[1:])` creates a new list at each level, so processing n items copies about n squared over two items overall. Pass an **index** instead of a slice, as the binary search does with `low` and `high`.

### Divide and conquer and complexity

When a function splits a problem in half and recurses on one half, as in binary search, the number of steps is about log n. When it recurses on both halves and combines the results in linear time, as in merge sort, the total is about n log n. Naive Fibonacci recurses twice on slightly smaller problems and does nothing to share the work, which is why it takes exponential time. Chapter 2.7 introduced these notations.

### Memoization by hand

You can memoize with a dictionary passed through the calls, which shows what `lru_cache` does:

```python
def fib_memo(n, memo=None):
    if memo is None:
        memo = {}
    if n < 2:
        return n
    if n not in memo:
        memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo)
    return memo[n]
```

Note the `None` default, to avoid the shared mutable default trap (Chapter 3.2).

### Mutual recursion

Two functions can call each other. This is rare, but happens naturally in parsers and state machines:

```python
def is_even(n):
    return True if n == 0 else is_odd(n - 1)


def is_odd(n):
    return False if n == 0 else is_even(n - 1)
```

### Recursion and untrusted input

Recursive code that processes data from outside, such as JSON from a client, can be attacked with deeply nested input that exhausts the call stack. Standard library functions like `json.loads` guard against some of this, but your own recursive functions must defend themselves with an explicit **depth limit**, as the industry-level snippet does.

### Recursive generators

When a recursive function would build a huge list, it can instead **yield** values one at a time using `yield from` (Chapter 7.2). This lets you walk a tree lazily without storing every result.

### Debugging recursion

- Write and test the **base case first**.
- Print the arguments (with depth indentation) to watch the calls.
- Check that every recursive call is on a **strictly smaller** problem.
- Test the smallest inputs (empty, one item, zero) and then a few larger ones.
- If the function is slow, count calls. Repeated calls with the same arguments mean memoization will help.

### Choosing between recursion and iteration

Choose **recursion** when the data or the problem is naturally nested (trees, nested documents, backtracking), when each level can be explained in a sentence, and when the depth is modest and bounded. Choose **iteration** for simple counting and accumulating, for very deep structures, and in code that is performance sensitive. If you find yourself maintaining an explicit stack that mimics the call stack, ask whether the recursive version was clearer after all.

### The mindset

Recursive thinking is a skill that grows with practice. Trust the recursive call: assume it correctly handles the smaller problem, and only worry about how to combine its answer, and about the base case. Do not try to trace every level in your head.

## Industry-level snippet

Backends handle nested JSON payloads everywhere, and two jobs come up constantly: **redacting** secrets before data reaches the logs, and **flattening** nested settings into dotted keys. Both follow the shape of the data, so recursion fits. Because the input comes from outside, both enforce a maximum depth. A third function counts the leaves **iteratively**, with an explicit stack, to show the alternative that has no depth limit.

```python
"""Safe recursive processing of nested payloads: redaction and flattening."""

import logging
from collections.abc import Mapping
from typing import Any

logger = logging.getLogger(__name__)

MAX_DEPTH = 20
REDACTED = "***"
SENSITIVE_KEYS = frozenset({"password", "token", "authorization", "secret", "api_key"})


class DepthError(ValueError):
    """Raised when a payload is nested more deeply than allowed."""


def redact(
    value: Any,
    sensitive_keys: frozenset[str] = SENSITIVE_KEYS,
    _depth: int = 0,
) -> Any:
    """Return a copy of a nested payload with sensitive values hidden.

    Mappings and lists are searched recursively. Keys are matched without
    regard to case. The original payload is never modified.

    Raises:
        DepthError: If the payload is nested more than MAX_DEPTH levels deep.
    """
    if _depth > MAX_DEPTH:
        raise DepthError(f"payload is nested deeper than {MAX_DEPTH} levels")
    if isinstance(value, Mapping):
        return {
            key: REDACTED
            if str(key).casefold() in sensitive_keys
            else redact(item, sensitive_keys, _depth + 1)
            for key, item in value.items()
        }
    if isinstance(value, list | tuple):
        return [redact(item, sensitive_keys, _depth + 1) for item in value]
    return value  # base case: a plain value such as a string or number


def flatten(
    data: Mapping[str, Any], prefix: str = "", _depth: int = 0
) -> dict[str, Any]:
    """Turn nested mappings into one flat dictionary with dotted keys.

    Raises:
        DepthError: If the mappings are nested more than MAX_DEPTH levels deep.
    """
    if _depth > MAX_DEPTH:
        raise DepthError(f"payload is nested deeper than {MAX_DEPTH} levels")
    flat: dict[str, Any] = {}
    for key, value in data.items():
        path = f"{prefix}.{key}" if prefix else str(key)
        if isinstance(value, Mapping) and value:
            flat.update(flatten(value, path, _depth + 1))
        else:
            flat[path] = value
    return flat


def count_leaves(value: Any) -> int:
    """Count the non-container values in a nested payload.

    Uses an explicit stack instead of recursion, so depth is not limited by
    Python's recursion limit.
    """
    count = 0
    stack: list[Any] = [value]
    while stack:
        current = stack.pop()
        if isinstance(current, Mapping):
            stack.extend(current.values())
        elif isinstance(current, list | tuple):
            stack.extend(current)
        else:
            count += 1
    return count
```

Try it:

```python
payload = {
    "user": {
        "name": "ada",
        "password": "hunter2",
        "contacts": [{"email": "ada@example.com", "token": "abc"}],
    },
    "Authorization": "Bearer xyz",
    "retries": 3,
}

print(redact(payload))
print(flatten({"db": {"host": "localhost", "port": 5432}, "debug": False}))
print(count_leaves(payload))

deep = current = {}
for _ in range(50):
    current["next"] = {}
    current = current["next"]
current["value"] = 42

try:
    redact(deep)
except DepthError as exc:
    print("DepthError:", exc)
print(count_leaves(deep))
```

```text
{'user': {'name': 'ada', 'password': '***', 'contacts': [{'email': 'ada@example.com', 'token': '***'}]}, 'Authorization': '***', 'retries': 3}
{'db.host': 'localhost', 'db.port': 5432, 'debug': False}
6
DepthError: payload is nested deeper than 20 levels
1
```

Why it is written this way:

- Both recursive functions have an obvious **base case** (a plain value, or a mapping with nothing to descend into) and a recursive case that handles one level and trusts the call for the rest.
- `_depth` increases on every call, and the guard at the top converts hostile input into a clean, specific `DepthError` instead of a `RecursionError` or a crash.
- `redact` builds **new** dictionaries and lists rather than modifying the payload, so the original data used by the rest of the program is untouched (Chapter 2.5).
- Matching keys case-insensitively catches `Authorization` as well as `authorization`, and the message never echoes the sensitive value.
- `count_leaves` shows the explicit-stack alternative. It handles the same 50-level structure that `redact` refuses, because it makes no recursive calls.
- The leading underscore on `_depth` marks it as an implementation detail that callers should not pass.

## Common mistakes

1. **Forgetting the base case,** causing `RecursionError`.
2. **Writing a base case that is never reached,** for example because the argument does not get smaller, or the input can be negative.
3. **Forgetting to `return` the result of the recursive call.** The function silently returns `None`.
4. **Making the problem larger or not smaller** in the recursive call.
5. **Recomputing the same subproblems,** as in naive Fibonacci, instead of memoizing.
6. **Copying big slices at every level** (`items[1:]`) and turning a linear job into a quadratic one.
7. **Using recursion for plain counting and accumulating,** where a loop is simpler and faster.
8. **Recursing over untrusted, arbitrarily deep input** without a depth limit.
9. **Raising the recursion limit as a first resort.** It can crash the interpreter. Prefer iteration.
10. **Using a mutable default argument as an accumulator,** which is shared between calls and between separate top-level uses.
11. **Mixing up the order of work,** such as printing before the recursive call when you meant after (or the reverse).
12. **Trying to trace every level mentally.** Test the base case and one step, then trust the recursion.

## Practice exercises

1. Write `countdown(n)` that prints the numbers from `n` down to 1 and then `"Liftoff"`, using recursion. Then write `countup(n)` that prints 1 up to `n`, and explain what you changed.
2. Write a recursive `power(base, exponent)` for non-negative integer exponents without using `**`. Add the repeated-halving improvement and compare the number of calls.
3. Write a recursive `reverse(text)` that returns a string reversed, and a recursive `is_palindrome(text)`. State the base case for each.
4. Write a recursive `sum_digits(n)` for a non-negative integer, using `divmod` or `%` and `//`.
5. Write `count_occurrences(items, target)` recursively over a nested list (a list that may contain lists), counting how many times a value appears anywhere.
6. Print the contents of a nested dictionary as an indented outline, with keys as headings and non-dictionary values as leaves.
7. Add a call counter to a naive recursive Fibonacci and show how the call count grows for n = 10, 20, and 30. Then add `@cache` and compare.
8. Write the `hanoi(n, source, target, spare)` function that prints the moves needed to solve the Towers of Hanoi puzzle, and compute how many moves it takes for 1 to 10 disks.
9. Convert your nested-list flattening function into an iterative one using an explicit stack, and test it on a structure 5000 levels deep. Show that the recursive version fails and the iterative one succeeds.
10. Write a function `permutations(items)` that returns all orderings of a list using recursion (choose one item, then permute the rest). Compare the result with `itertools.permutations` for a list of four items.
11. Write a recursive `merge_sort(items)`: split in half, sort each half, merge the sorted halves. Test it on empty, single-item, sorted, reversed, and random lists.
12. Extend `redact` so that it also redacts any string value that looks like a long token (for example, 32 or more letters and digits with no spaces), regardless of its key, and add a `max_depth` parameter. Write tests for both features and for the depth limit.

## Summary

- Recursion is a function calling itself. A correct recursive function has a **base case** that answers directly and a **recursive case** that calls itself on a strictly smaller problem.
- Each call adds a frame to the **call stack**. The answers return back up the stack in reverse order.
- Python limits recursion depth, by default to about 1000. Exceeding it raises `RecursionError`, which usually means a missing base case or input that is too deep.
- Recursion suits nested and tree-shaped data (folders, JSON, menus), divide and conquer (binary search, merge sort), and backtracking.
- Naive recursion can repeat work exponentially, as with Fibonacci. **Memoization** (`functools.cache`) fixes it by remembering results.
- Python does not optimize tail calls, so deep recursion is better written as a loop with an explicit stack.
- Prefer iteration for simple counting, very deep structures, and performance-critical code. Avoid copying slices at every level.
- When processing untrusted nested data, enforce your own maximum depth.
- Write the base case first, make sure every call makes progress toward it, and trust the recursive call to handle the smaller problem.
