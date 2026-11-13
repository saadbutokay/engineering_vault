## Overview

A **list** is Python's everyday container: an ordered, changeable collection of items. Shopping carts, rows from a database, lines from a file, the users returned by an API: almost every program keeps its data in lists at some point. Mastering them means knowing how to create them, change them, sort them, build them from other data with comprehensions, and understand what happens when lists are nested or shared.

By the end of this note you will be able to:

- Create lists in several ways and read them with indexing and slicing.
- Add, remove, and replace items using the list methods.
- Choose correctly between `list.sort()` and `sorted()`, and sort with a `key`.
- Build new lists with list comprehensions.
- Work with nested lists without falling into the classic aliasing trap.
- Describe which list operations are fast and which are slow.

> [!note] Lists are mutable
> Unlike strings and tuples, a list can be changed in place after it is created. That power is why lists are so useful, and also why Chapter 1.2's warnings about shared objects matter so much here.

## Core concepts

### What a list is

A list holds items in a defined order, each at a numbered position starting from `0`. The items can be of any type, including other lists, and they do not have to match. In practice, good lists hold items of one kind (a list of prices, a list of names) so that code can treat every item the same way.

Under the hood, a CPython list is a **dynamic array**: a block of memory holding references to the items. This makes reading by position very fast and makes adding to the end cheap. Adding or removing near the front is slower, because everything after it has to shift.

### Creating lists

| Way | Example | Result |
| --- | --- | --- |
| Literal | `[1, 2, 3]` | `[1, 2, 3]` |
| Empty list | `[]` or `list()` | `[]` |
| From any iterable | `list("abc")` | `['a', 'b', 'c']` |
| From a range | `list(range(3))` | `[0, 1, 2]` |
| Repetition | `[0] * 3` | `[0, 0, 0]` |
| From text | `"a,b".split(",")` | `['a', 'b']` |
| Comprehension | `[n * n for n in range(3)]` | `[0, 1, 4]` |

### Indexing and slicing

Everything you learned about string indexing and slicing in Chapter 1.4 works the same way, with one difference: because lists are mutable, you can also **assign** to a position or a slice, or **delete** one.

```text
 items:    "a"  "b"  "c"  "d"  "e"
 index:     0    1    2    3    4
 negative: -5   -4   -3   -2   -1
```

### List methods

| Method | What it does | Returns |
| --- | --- | --- |
| `append(x)` | Add one item to the end | `None` |
| `extend(iterable)` | Add every item of an iterable to the end | `None` |
| `insert(i, x)` | Insert an item before position `i` | `None` |
| `remove(x)` | Remove the first item equal to `x` | `None` |
| `pop()` / `pop(i)` | Remove and return the last item, or the item at `i` | the item |
| `clear()` | Remove everything | `None` |
| `index(x)` | Position of the first item equal to `x` | an `int` |
| `count(x)` | How many items equal `x` | an `int` |
| `sort()` | Sort in place | `None` |
| `reverse()` | Reverse in place | `None` |
| `copy()` | A shallow copy | a new list |

Notice how many return `None`. A method that changes a list in place conventionally returns `None`, so that you do not mistake it for something that produced a new list.

### Useful built-in functions

| Function | Purpose |
| --- | --- |
| `len(items)` | Number of items |
| `sum(items)` | Total of numbers |
| `min(items)`, `max(items)` | Smallest, largest |
| `sorted(items)` | A new sorted list |
| `reversed(items)` | An iterator over the items in reverse |
| `any(...)`, `all(...)` | Is any, or every, condition true |
| `enumerate(items)` | Pairs of `(position, item)` |
| `zip(a, b)` | Pairs of items from two lists |

### sort versus sorted

| | `items.sort()` | `sorted(items)` |
| --- | --- | --- |
| Changes the original | Yes | No |
| Returns | `None` | A new list |
| Works on | Lists only | Any iterable |

Both accept `reverse=True` and a `key=` function. The **key function** is called once per item and returns the value to compare. Sorting is **stable**: items that compare equal keep their original relative order.

### List comprehensions

A **list comprehension** builds a list from another iterable in one expression:

```python
[expression for item in iterable if condition]
```

Read it as: "the expression, for each item in the iterable, but only if the condition holds." The `if` part is optional. You can also put a conditional expression inside, and use several `for` clauses (the leftmost is the outer loop).

### Nested lists

A list can contain lists, which is the simplest way to model a table or a grid. `grid[row][column]` first picks the row, then the column.

## Basic syntax examples

### Changing a list

```text
>>> nums = [3, 1, 4]
>>> nums.append(1)
>>> nums
[3, 1, 4, 1]
>>> nums.extend([5, 9])
>>> nums
[3, 1, 4, 1, 5, 9]
>>> nums.insert(0, 99)
>>> nums
[99, 3, 1, 4, 1, 5, 9]
>>> nums.remove(1)
>>> nums
[99, 3, 4, 1, 5, 9]
>>> nums.pop()
9
>>> nums.pop(0)
99
>>> nums
[3, 4, 1, 5]
>>> nums.index(4)
1
>>> nums.count(1)
1
>>> nums.reverse()
>>> nums
[5, 1, 4, 3]
```

`remove(1)` removed only the first `1`. `pop()` returns the removed item, which is useful when you need it.

### append versus extend

```text
>>> a = [1, 2]
>>> a.append([3, 4])
>>> a
[1, 2, [3, 4]]
>>> b = [1, 2]
>>> b.extend([3, 4])
>>> b
[1, 2, 3, 4]
>>> b += [5]
>>> b
[1, 2, 3, 4, 5]
>>> [1, 2] + [3]
[1, 2, 3]
>>> [0] * 3
[0, 0, 0]
```

`append` adds **one** item (here, a list). `extend` adds **each item** of an iterable. `+=` on a list behaves like `extend`. The `+` operator builds a new list.

### Slicing and slice assignment

```text
>>> letters = ["a", "b", "c", "d", "e"]
>>> letters[1:3]
['b', 'c']
>>> letters[-2:]
['d', 'e']
>>> letters[::2]
['a', 'c', 'e']
>>> letters[1:3] = ["X", "Y", "Z"]
>>> letters
['a', 'X', 'Y', 'Z', 'd', 'e']
>>> del letters[0]
>>> letters
['X', 'Y', 'Z', 'd', 'e']
>>> letters[10]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: list index out of range
```

A slice assignment may replace a slice with a different number of items, and the list grows or shrinks to fit.

### Built-in helpers

```text
>>> nums = [4, 8, 15]
>>> len(nums), sum(nums), min(nums), max(nums)
(3, 27, 4, 15)
>>> any(n > 10 for n in nums), all(n > 10 for n in nums)
(True, False)
>>> 8 in nums
True
>>> list("abc")
['a', 'b', 'c']
>>> list(range(3))
[0, 1, 2]
>>> [1, 2, 3] < [1, 2, 4]
True
```

Lists compare item by item from the left, like words in a dictionary.

### Looping

```python
prices = [4.5, 1.25, 39.9]

for price in prices:
    print(price)

for position, price in enumerate(prices, start=1):
    print(f"{position}: {price:.2f}")
```

```text
4.5
1.25
39.9
1: 4.50
2: 1.25
3: 39.90
```

### Sorting

```text
>>> nums = [3, 1, 2]
>>> nums.sort()
>>> nums
[1, 2, 3]
>>> result = nums.sort()
>>> print(result)
None
>>> original = [3, 1, 2]
>>> sorted(original)
[1, 2, 3]
>>> original
[3, 1, 2]
>>> sorted(original, reverse=True)
[3, 2, 1]
```

The line `result = nums.sort()` is a classic bug: `sort()` returns `None`, so `result` is useless.

### Sorting with key

```text
>>> words = ["banana", "Apple", "cherry", "fig"]
>>> sorted(words)
['Apple', 'banana', 'cherry', 'fig']
>>> sorted(words, key=str.lower)
['Apple', 'banana', 'cherry', 'fig']
>>> sorted(words, key=len)
['fig', 'Apple', 'banana', 'cherry']
>>> sorted(words, key=len, reverse=True)
['banana', 'cherry', 'Apple', 'fig']
>>> people = [("Ada", 36), ("Grace", 45), ("Linus", 28)]
>>> sorted(people, key=lambda person: person[1])
[('Linus', 28), ('Ada', 36), ('Grace', 45)]
>>> from operator import itemgetter
>>> sorted(people, key=itemgetter(1), reverse=True)
[('Grace', 45), ('Ada', 36), ('Linus', 28)]
```

`banana` and `cherry` have the same length, and they keep their original order in both directions because the sort is stable. The `lambda person: person[1]` is a tiny nameless function (Chapter 3.4). `itemgetter(1)` is an equivalent ready-made one.

Mixed types cannot be ordered:

```text
>>> sorted([3, "a", 1])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: '<' not supported between instances of 'str' and 'int'
```

### List comprehensions

```text
>>> [n * n for n in range(5)]
[0, 1, 4, 9, 16]
>>> [n for n in range(10) if n % 3 == 0]
[0, 3, 6, 9]
>>> [word.upper() for word in ["a", "b"]]
['A', 'B']
>>> ["even" if n % 2 == 0 else "odd" for n in range(4)]
['even', 'odd', 'even', 'odd']
>>> [(x, y) for x in range(2) for y in range(2)]
[(0, 0), (0, 1), (1, 0), (1, 1)]
```

The same work written as a loop:

```python
squares = []
for n in range(5):
    squares.append(n * n)
```

Prefer the comprehension when it fits on one readable line. It is shorter and usually faster.

### Nested lists

```text
>>> matrix = [[1, 2, 3], [4, 5, 6]]
>>> matrix[1][2]
6
>>> [value for row in matrix for value in row]
[1, 2, 3, 4, 5, 6]
>>> [[row[i] for row in matrix] for i in range(3)]
[[1, 4], [2, 5], [3, 6]]
```

The second comprehension flattens the table, the third **transposes** it (turns rows into columns).

### Creating a grid correctly

```text
>>> grid = [[0] * 3 for _ in range(2)]
>>> grid[0][1] = 5
>>> grid
[[0, 5, 0], [0, 0, 0]]
>>> bad = [[0] * 3] * 2
>>> bad[0][1] = 5
>>> bad
[[0, 5, 0], [0, 5, 0]]
```

`[[0] * 3] * 2` repeats the **same inner list** twice, so both rows are one object. The comprehension builds a fresh inner list for each row. The underscore `_` is a conventional name for a value you do not use.

## Deeper behavior

### Lists store references

A list holds references to objects, not the objects themselves. Two lists can refer to the same objects, and `b = a` makes two names for one list (Chapter 1.2). Copying a list copies the references, not the objects they point to. Chapter 2.5 shows exactly how this creates shallow-copy bugs with nested lists.

### Slicing makes a shallow copy

`items[:]` creates a new list with the same references. It is a common idiom for copying, as is `items.copy()` and `list(items)`.

### How fast are list operations?

| Operation | Cost | Why |
| --- | --- | --- |
| `items[i]`, `len(items)` | Constant | Direct position lookup |
| `items.append(x)` | Constant on average | The array grows in occasional large steps |
| `items.pop()` | Constant | Removes from the end |
| `items.insert(0, x)`, `items.pop(0)` | Proportional to length | Every other item must shift |
| `x in items`, `items.index(x)`, `items.remove(x)` | Proportional to length | May check every item |
| `items.sort()` | About n times log n | Python's Timsort |
| `items[a:b]` | Proportional to the slice size | Copies the references |

"Proportional to length" means that doubling the list roughly doubles the time. Chapter 2.7 explains this notation properly. The practical lesson: a list is excellent as a **stack** (add and remove at the end), poor as a **queue** (remove from the front), and poor for repeated "is this in there?" questions on large data. Sets (Chapter 2.4) and `deque` (Chapter 2.6) fix those cases.

### Do not change a list while looping over it

Chapter 1.6 showed that removing items during iteration skips elements. Loop over a copy, or build a new list:

```python
numbers = [2, 4, 6, 8]
kept = [n for n in numbers if n > 4]
print(kept)
```

```text
[6, 8]
```

### Sorting details

- `sort` and `sorted` use **Timsort**, which is very fast on data that is already partly ordered.
- The `key` function runs once per item, not once per comparison.
- To sort by several criteria, return a tuple from the key: `key=lambda p: (p.department, -p.salary)`. Tuples compare item by item (Chapter 2.2).
- Sorting **strings** compares Unicode code points, so uppercase sorts before lowercase. Use `key=str.casefold` for human-friendly ordering.

### `sum` on lists of lists is slow

`sum(list_of_lists, [])` builds a new list at every step and takes time proportional to the square of the total length. Use a comprehension for flattening, or `itertools.chain` (Chapter 7.3).

### Comprehension scope and readability

The variable inside a comprehension does not leak into the surrounding code. A comprehension with two `for` clauses and an `if` is about the limit of what is easy to read. If you need more, write a normal loop. A comprehension that builds a list you only loop over once can usually become a **generator expression** (Chapter 7.2) by replacing the brackets with parentheses, which avoids creating the full list in memory.

### Lists as stacks and queues

```python
stack = []
stack.append("a")
stack.append("b")
print(stack.pop())    # last in, first out
```

```text
b
```

For first-in, first-out behavior use `collections.deque` (Chapter 2.6), because `list.pop(0)` is slow on large lists.

### Equality and identity

`[1, 2] == [1, 2]` is `True` (equal contents), `[1, 2] is [1, 2]` is `False` (two objects).

## Industry-level snippet

A listing endpoint in a web backend takes a collection of records and must filter, sort, and return one page at a time. This module does that with lists and comprehensions, validates every input, and chunks work into batches for bulk processing. The generic functions use Python 3.12 syntax (`def paginate[T](...)`), which Chapter 8.3 explains.

```python
"""List-processing helpers for an order listing endpoint."""

import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

MAX_PAGE_SIZE = 100


@dataclass(frozen=True)
class Order:
    """A single order as shown in the listing."""

    order_id: str
    customer: str
    total_cents: int
    status: str


SORT_KEYS = {
    "id": lambda order: order.order_id,
    "customer": lambda order: order.customer.casefold(),
    "total": lambda order: order.total_cents,
}


def filter_orders(
    orders: list[Order],
    status: str | None = None,
    min_total_cents: int = 0,
) -> list[Order]:
    """Return the orders matching the optional status and minimum total."""
    return [
        order
        for order in orders
        if (status is None or order.status == status)
        and order.total_cents >= min_total_cents
    ]


def sort_orders(
    orders: list[Order], sort_by: str = "id", descending: bool = False
) -> list[Order]:
    """Return a new list of orders sorted by id, customer, or total.

    Raises:
        ValueError: If sort_by is not a supported field.
    """
    try:
        key = SORT_KEYS[sort_by]
    except KeyError:
        allowed = ", ".join(sorted(SORT_KEYS))
        raise ValueError(
            f"cannot sort by {sort_by!r}; choose one of: {allowed}"
        ) from None
    return sorted(orders, key=key, reverse=descending)


def paginate[T](items: list[T], page: int, page_size: int) -> tuple[list[T], int]:
    """Return one page of items and the total number of pages.

    Args:
        items: The full, already sorted list.
        page: The page number, starting at 1.
        page_size: Items per page, from 1 to MAX_PAGE_SIZE.

    Raises:
        ValueError: If page or page_size is out of range.
    """
    if not 1 <= page_size <= MAX_PAGE_SIZE:
        raise ValueError(f"page_size must be between 1 and {MAX_PAGE_SIZE}")
    if page < 1:
        raise ValueError("page must be at least 1")
    total_pages = max(1, -(-len(items) // page_size))  # ceiling division
    start = (page - 1) * page_size
    return items[start : start + page_size], total_pages


def chunked[T](items: list[T], size: int) -> list[list[T]]:
    """Split items into consecutive batches of at most size items."""
    if size < 1:
        raise ValueError("size must be at least 1")
    return [items[start : start + size] for start in range(0, len(items), size)]
```

Try it:

```python
orders = [
    Order("A3", "grace", 5_000, "paid"),
    Order("A1", "Ada", 12_500, "paid"),
    Order("A2", "linus", 800, "refunded"),
    Order("A4", "Ada", 9_900, "paid"),
]

paid = filter_orders(orders, status="paid")
by_total = sort_orders(paid, "total", descending=True)
page, pages = paginate(by_total, page=1, page_size=2)

print([order.order_id for order in page], pages)
print([len(batch) for batch in chunked(orders, 3)])
```

```text
['A1', 'A4'] 2
[3, 1]
```

Why it is written this way:

- Every function returns a **new** list and never modifies the one it received, so callers cannot be surprised.
- A dictionary of key functions replaces a chain of `if` statements, and the error message lists the valid choices.
- Paging uses slice arithmetic. Slices never raise an error for a page beyond the end, they simply return an empty list.
- The ceiling-division trick `-(-a // b)` rounds up without importing `math`.
- Validation happens at the boundary, with limits stated as a named constant.

## Common mistakes

1. **Writing `result = items.sort()`.** `sort()` returns `None`. Use `sorted()` if you need a new list.
2. **Assuming `b = a` copies a list.** It creates a second name for the same list.
3. **Building a grid with `[[0] * 3] * 2`.** All rows are the same object. Use a comprehension.
4. **Removing or inserting items while looping over the same list.** Loop over a copy or build a new list.
5. **Using `append` when `extend` was intended**, ending up with a list inside the list.
6. **Using `remove(x)` without checking.** It raises `ValueError` if `x` is not present.
7. **Indexing an empty list**, such as `items[0]` or `items.pop()`. Check `if items:` first.
8. **Using a list for repeated membership tests on large data.** Use a set.
9. **Using `pop(0)` or `insert(0, ...)` in a loop on a big list.** Use `deque`.
10. **Cramming too much into a comprehension.** If it needs two lines to understand, write a loop.
11. **Sorting mixed types** such as numbers and strings, which raises `TypeError`.
12. **Using `==` to test for an empty list** such as `items == []`. Use `if not items:`.

## Practice exercises

1. Create a list of the first ten square numbers using a loop, then using a comprehension. Print both and confirm they are equal.
2. Given `values = [10, 20, 30, 40, 50, 60]`, write expressions that produce: the first three items, the last two, every second item, and the list reversed (in two different ways).
3. Starting with `tasks = ["write", "test", "deploy"]`, insert `"design"` at the start, add `"review"` before `"deploy"`, remove `"test"`, and print the final list. Use only list methods.
4. Explain the difference between `append` and `extend` by predicting the output of each of these, then run them: `a = [1]; a.append([2, 3])` and `b = [1]; b.extend([2, 3])`.
5. Sort `["pear", "Apple", "fig", "banana"]` four ways: alphabetically ignoring case, by length, by length descending with ties alphabetical, and in reverse order of the original. Use `sorted` and a `key`.
6. Given a list of `(name, score)` tuples, print the names in descending order of score, and for equal scores, in alphabetical order.
7. Use a comprehension to build a list of all the even numbers from 1 to 50 that are not multiples of 4.
8. Write a function `flatten(matrix)` that turns a list of lists into one list, using a comprehension. Test it on an empty list and on a list containing empty lists.
9. Create a 4 by 4 grid of zeros correctly, set the diagonal to 1, and print it row by row. Then create it incorrectly with `*` and demonstrate the bug.
10. Write code that removes all negative numbers from a list in place, correctly, without skipping any element. Show both an in-place approach and a new-list approach.
11. Write `rotate(items, k)` that returns a new list shifted left by `k` places using slicing, and handles `k` larger than the list length.
12. Extend the order listing module with `search_orders(orders, text)` that finds orders whose customer name contains the text ignoring case, and add an optional `limit` argument with validation.

## Summary

- A list is an ordered, mutable sequence of references, stored as a dynamic array. Create it with literals, `list(...)`, `split`, `range`, repetition, or a comprehension.
- Index and slice like strings, and also assign to positions and slices or delete them.
- Key methods: `append`, `extend`, `insert`, `remove`, `pop`, `index`, `count`, `sort`, `reverse`, `copy`. Methods that change a list in place return `None`.
- `sort()` changes the list and returns `None`. `sorted()` returns a new list from any iterable. Both accept `key=` and `reverse=`, and sorting is stable.
- List comprehensions build lists concisely: `[expression for item in iterable if condition]`.
- Nested lists model tables. Build grids with a comprehension, never with `[[0] * n] * m`.
- Appending and popping at the end are fast, while inserting or removing at the front and searching with `in` get slower as the list grows.
- Never modify a list while looping over it, and remember that `b = a` shares one list between two names.
