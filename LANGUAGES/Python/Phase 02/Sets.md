## Overview

A **set** is an unordered collection of **unique** items. It answers two kinds of questions extremely well: "is this item in the collection?" and "how do these collections overlap?" Removing duplicates, checking permissions, comparing two lists of tags, finding which users appear in both reports: these are all set problems, and a set solves each of them in a line or two.

Sets are built on the same hashing technique as dictionaries (Chapter 2.3), which gives them very fast membership tests.

By the end of this note you will be able to:

- Create sets and add, remove, and test items.
- Combine sets with union, intersection, difference, and symmetric difference.
- Compare sets with subset, superset, and disjoint tests.
- Use `frozenset` when you need an immutable, hashable set.
- Explain why `item in some_set` is dramatically faster than `item in some_list`.

> [!note] A set has no positions
> A set has no first or last item and no index. You cannot write `s[0]`. If you need order, use a list. If you need uniqueness **and** order, use `dict.fromkeys(items)`, which this note shows.

## Core concepts

### What a set is

A set stores each distinct value at most once. Adding something that is already present changes nothing. The items must be **hashable**: numbers, strings, tuples of hashable items, and `frozenset` are fine, while lists, dictionaries, and ordinary sets are not.

Sets are **mutable**: you can add and remove items. The type is `set`. Its immutable sibling is `frozenset`.

### Creating sets

| Way | Example | Notes |
| --- | --- | --- |
| Literal | `{1, 2, 3}` | Duplicates are dropped |
| Empty set | `set()` | `{}` would be an empty **dictionary** |
| From an iterable | `set([1, 2, 2, 3])` | Removes duplicates |
| From a string | `set("hello")` | A set of its distinct characters |
| Comprehension | `{n % 3 for n in range(10)}` | Like list comprehensions |
| Immutable | `frozenset([1, 2, 3])` | Cannot change, can be a dictionary key |

### Changing a set

| Method | What it does | If the item is absent |
| --- | --- | --- |
| `add(x)` | Add one item | |
| `update(iterable)` | Add every item of one or more iterables | |
| `remove(x)` | Remove an item | Raises `KeyError` |
| `discard(x)` | Remove an item | Does nothing |
| `pop()` | Remove and return an arbitrary item | Raises `KeyError` if the set is empty |
| `clear()` | Remove everything | |
| `copy()` | A shallow copy | |

Use `discard` when absence is fine, and `remove` when absence would indicate a bug.

### Set operations

Sets support the operations of school mathematics. Each has an operator form, which needs sets on both sides, and a method form, which accepts any iterable.

| Operation | Method | Result contains |
| --- | --- | --- |
| Union | `a.union(b)` | Items in either set |
| Intersection | `a.intersection(b)` | Items in both sets |
| Difference | `a.difference(b)` | Items in `a` that are not in `b` |
| Symmetric difference | `a.symmetric_difference(b)` | Items in exactly one of the two |

The operators:

- `a | b` is the union.
- `a & b` is the intersection.
- `a - b` is the difference.
- `a ^ b` is the symmetric difference.
- `a |= b`, `a &= b`, `a -= b`, `a ^= b` update `a` in place.

### Comparing sets

| Question | Method | Operator form |
| --- | --- | --- |
| Is every item of `a` also in `b`? | `a.issubset(b)` | `a <= b` |
| Is every item of `b` also in `a`? | `a.issuperset(b)` | `a >= b` |
| Is `a` a subset but not equal? | | `a < b` |
| Do they share nothing? | `a.isdisjoint(b)` | |
| Same items? | | `a == b` |

### Membership

`x in some_set` is the signature operation. On average it takes constant time, so a set of a million items answers as fast as a set of ten.

### frozenset

A `frozenset` has every non-changing feature of a set, including the operations, but no `add` or `remove`. Because it cannot change, it is **hashable**, so it can be a dictionary key or a member of another set.

## Basic syntax examples

### Creating

```text
>>> s = {3, 1, 2, 3, 1}
>>> s
{1, 2, 3}
>>> len(s)
3
>>> set()
set()
>>> type({}), type(set())
(<class 'dict'>, <class 'set'>)
>>> sorted(set("hello"))
['e', 'h', 'l', 'o']
>>> {n % 3 for n in range(10)}
{0, 1, 2}
>>> {len(word) for word in ["a", "bb", "cc"]}
{1, 2}
>>> {(1, 2), (1, 2), (2, 1)} == {(1, 2), (2, 1)}
True
```

> [!warning] Printed order is not guaranteed
> Small sets of integers often print in sorted order, but that is an accident of how they are stored. Sets of strings can print differently on different runs. This note uses `sorted(...)` wherever order would otherwise matter.

### Adding and removing

```text
>>> s = {1, 2, 3}
>>> s.add(4)
>>> s.add(2)
>>> sorted(s)
[1, 2, 3, 4]
>>> s.update([10, 20], {30})
>>> sorted(s)
[1, 2, 3, 4, 10, 20, 30]
>>> s.discard(99)
>>> s.remove(99)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 99
>>> s.remove(10)
>>> 10 in s, 2 in s
(False, True)
```

### Set operations

```text
>>> a = {1, 2, 3, 4}
>>> b = {3, 4, 5}
>>> a | b
{1, 2, 3, 4, 5}
>>> a & b
{3, 4}
>>> a - b
{1, 2}
>>> b - a
{5}
>>> a ^ b
{1, 2, 5}
>>> a.union(b) == a | b
True
>>> a.intersection([3, 9])
{3}
>>> a | [9]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for |: 'set' and 'list'
```

The operators demand sets on both sides. The methods are more forgiving and accept any iterable, as `a.intersection([3, 9])` shows.

### In-place operations

```text
>>> tags = {"python", "backend"}
>>> tags |= {"api"}
>>> sorted(tags)
['api', 'backend', 'python']
>>> tags -= {"backend"}
>>> sorted(tags)
['api', 'python']
```

### Comparing

```text
>>> a = {1, 2, 3, 4}
>>> {1, 2} <= a
True
>>> {1, 2}.issubset(a)
True
>>> a >= {1, 2}
True
>>> {1, 2, 3, 4} < a
False
>>> a.isdisjoint({9})
True
>>> {1, 2} == {2, 1}
True
```

### Removing duplicates

```text
>>> names = ["ada", "grace", "ada", "linus", "grace"]
>>> sorted(set(names))
['ada', 'grace', 'linus']
>>> list(dict.fromkeys(names))
['ada', 'grace', 'linus']
```

`set(names)` removes duplicates but loses the original order. `dict.fromkeys(names)` removes duplicates **and** keeps the order in which each item first appeared, because dictionary keys are unique and ordered.

### Membership testing

```python
allowed_methods = {"GET", "POST", "PUT"}

request_method = "DELETE"
if request_method not in allowed_methods:
    print(f"{request_method} is not allowed")
```

```text
DELETE is not allowed
```

### frozenset

```text
>>> fs = frozenset([1, 2, 3])
>>> fs
frozenset({1, 2, 3})
>>> fs | {4}
frozenset({1, 2, 3, 4})
>>> fs.add(4)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'frozenset' object has no attribute 'add'
>>> {frozenset({1, 2}): "pair"}
{frozenset({1, 2}): 'pair'}
>>> {[1]}
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

### A practical example: comparing two lists

```python
yesterday = ["ada", "grace", "linus", "barbara"]
today = ["grace", "linus", "ken", "dennis"]

before, after = set(yesterday), set(today)

print("new:", sorted(after - before))
print("gone:", sorted(before - after))
print("both days:", sorted(before & after))
```

```text
new: ['dennis', 'ken']
gone: ['ada', 'barbara']
both days: ['grace', 'linus']
```

## Deeper behavior

### Why membership is so much faster

A list answers `x in items` by checking one element after another, so the work grows with the list. A set uses hashing: it computes the hash of `x` and goes straight to the right place. Here is a measurement on one machine, looking for the last of 100,000 numbers:

```python
import timeit

numbers = list(range(100_000))
as_set = set(numbers)

list_time = timeit.timeit(lambda: 99_999 in numbers, number=200) / 200
set_time = timeit.timeit(lambda: 99_999 in as_set, number=200) / 200
print(f"list: {list_time * 1e6:.1f} microseconds")
print(f"set:  {set_time * 1e6:.3f} microseconds")
```

```text
list: 725.8 microseconds
set:  0.100 microseconds
```

The exact numbers depend on your machine, but the shape does not: here the set was thousands of times faster, and the gap widens as the data grows. If you test membership more than a handful of times against a large collection, convert it to a set once and test against that.

### Building a set costs time too

Converting a list to a set takes time proportional to its length and uses extra memory. It pays off when you will test membership many times. For a single check against a small list, `in` on the list is fine.

### Hashable elements only

```python
{[1, 2]}               # TypeError: unhashable type: 'list'
{(1, 2)}               # fine
{frozenset({1, 2})}    # fine
```

To store a "set of sets" or use a set as a key, use `frozenset`. To store lists, convert them to tuples first.

### Order is not defined, and strings vary between runs

Python randomizes string hashes at startup (Chapter 2.3). A set of strings can therefore iterate in a different order every time you run your program. This is a feature, not a bug: it protects servers from attacks that exploit predictable hashing. Never write code that depends on set order. Sort the set when you need a predictable output.

### Operators need sets, methods accept any iterable

Use operators when both sides are sets, for readability. Use methods such as `intersection` and `update` when one side is a list or another iterable.

### `|` copies, `|=` modifies

`a | b` creates a new set. `a |= b` changes `a` and is equivalent to `a.update(b)`. As with lists, be careful with names that point to the same set (Chapter 2.5).

### Do not change a set while iterating over it

```text
>>> s = {1, 2, 3}
>>> for item in s:
...     s.add(item + 10)
...
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
RuntimeError: Set changed size during iteration
```

Iterate over a copy, `for item in list(s):`, or build a new set.

### Choosing between set and list

| Need | Choose |
| --- | --- |
| Order or positions | List |
| Duplicates are meaningful | List |
| Fast "is it in there?" | Set |
| Remove duplicates | Set, or `dict.fromkeys` to keep order |
| Overlap, difference, and combination of groups | Set |
| Use the collection as a dictionary key | `frozenset` or tuple |

### Cost of set operations

Union, intersection, and difference take time proportional to the sizes of the sets involved. Intersection loops over the smaller set and tests membership in the larger one, so it is efficient even when one set is huge.

### `pop()` is arbitrary

`set.pop()` removes some item, and you must not assume which. If you need the smallest or largest, use `min`, `max`, or a heap (Chapter 2.7).

## Industry-level snippet

Access control and change tracking are natural set problems. This module normalizes tags, computes exactly what changed between two versions of a tag list, checks whether a user holds all required permissions, and detects duplicates in a batch of identifiers, preserving the order in which they appeared.

```python
"""Set-based helpers for tags, permissions, and duplicate detection."""

import logging
from collections.abc import Hashable, Iterable
from dataclasses import dataclass

logger = logging.getLogger(__name__)


class PermissionDeniedError(Exception):
    """Raised when a user lacks one or more required permissions."""


@dataclass(frozen=True)
class TagChanges:
    """The difference between two versions of a tag collection."""

    added: frozenset[str]
    removed: frozenset[str]
    kept: frozenset[str]


def normalize_tags(raw_tags: Iterable[str]) -> frozenset[str]:
    """Trim, case-fold, and de-duplicate tags, dropping blank ones."""
    return frozenset(tag.strip().casefold() for tag in raw_tags if tag.strip())


def diff_tags(old: Iterable[str], new: Iterable[str]) -> TagChanges:
    """Work out which tags were added, removed, and kept."""
    before, after = normalize_tags(old), normalize_tags(new)
    return TagChanges(added=after - before, removed=before - after, kept=before & after)


def missing_permissions(
    granted: Iterable[str], required: Iterable[str]
) -> frozenset[str]:
    """Return the required permissions that have not been granted."""
    return frozenset(required) - frozenset(granted)


def require_permissions(
    granted: Iterable[str], required: Iterable[str], action: str
) -> None:
    """Raise PermissionDeniedError unless every required permission is granted."""
    missing = missing_permissions(granted, required)
    if missing:
        logger.warning("Denied %r: missing %s", action, sorted(missing))
        raise PermissionDeniedError(f"{action} requires: {', '.join(sorted(missing))}")


def find_duplicates[T: Hashable](items: Iterable[T]) -> list[T]:
    """Return each value that appears more than once, in order of first repeat."""
    seen: set[T] = set()
    repeated: dict[T, None] = {}  # a dict gives us an ordered set
    for item in items:
        if item in seen:
            repeated[item] = None
        else:
            seen.add(item)
    return list(repeated)


def unique_in_order[T: Hashable](items: Iterable[T]) -> list[T]:
    """Remove duplicates while keeping the first occurrence of each item."""
    return list(dict.fromkeys(items))
```

Try it:

```python
changes = diff_tags(
    old=["Python", " api ", "backend"],
    new=["python", "API", "testing", ""],
)
print(sorted(changes.added), sorted(changes.removed), sorted(changes.kept))

granted = {"orders:read", "orders:write"}
print(sorted(missing_permissions(granted, {"orders:read", "orders:delete"})))

try:
    require_permissions(granted, ["orders:delete", "users:read"], "delete order")
except PermissionDeniedError as exc:
    print("PermissionDeniedError:", exc)

print(find_duplicates(["a3", "a1", "a3", "a2", "a1", "a3"]))
print(unique_in_order(["a3", "a1", "a3", "a2"]))
```

```text
['testing'] ['backend'] ['api', 'python']
['orders:delete']
PermissionDeniedError: delete order requires: orders:delete, users:read
['a3', 'a1']
['a3', 'a1', 'a2']
```

Why it is written this way:

- Each function takes any iterable but works internally with sets, so callers can pass lists, tuples, or sets.
- Returning `frozenset` tells callers that the result is a finished fact, not a collection to edit.
- `after - before` and `before & after` express "added" and "kept" as directly as the English words do.
- Error messages sort the missing permissions so that the output is predictable, even though the underlying set has no order.
- `find_duplicates` uses a set for the fast test and a dictionary to remember order, a practical use of both structures together.

## Common mistakes

1. **Writing `{}` for an empty set.** That is a dictionary. Write `set()`.
2. **Indexing a set.** `s[0]` raises `TypeError`. Convert to a list or sorted list.
3. **Expecting a set to keep order.** Use a list, `dict.fromkeys`, or sort the set.
4. **Putting a list or dictionary inside a set.** Use a tuple or `frozenset`.
5. **Using `remove` when the item might be absent.** Use `discard`.
6. **Using `|` with a list on the other side.** Use the method `union`, or convert the list.
7. **Changing a set while iterating over it.** Loop over a copy.
8. **Using a list for repeated membership tests on large data.** Convert to a set first.
9. **Printing a set and relying on its order in tests or output.** Sort it.
10. **Forgetting that `set("hello")` splits a string into characters.** For a set of words, use `set(["hello"])` or `{"hello"}`.
11. **Confusing `a - b` with `b - a`.** Difference is not symmetric.
12. **Building a set inside a loop on every iteration.** Build it once, outside.

## Practice exercises

1. Create a set from the list `[4, 2, 4, 1, 2, 3]`, print its length, and print it in sorted order.
2. Given two lists of student names, `math` and `physics`, print the students taking both, only math, only physics, and either.
3. Write a function that returns `True` if all the items of a list are unique, using a set in one line.
4. Given a long sentence, print the number of distinct words, ignoring case and trailing punctuation.
5. Remove duplicates from `[5, 3, 5, 1, 3, 2]` in two ways: one that loses the order, and one that keeps it.
6. Write code that finds the first duplicate in a list of numbers, stopping as soon as it is found, using a set of items seen so far.
7. Use set comparison operators to check whether `{"read", "write"}` is a subset of `{"read", "write", "admin"}`, and whether two permission sets share nothing.
8. Measure how long it takes to test membership of a missing item in a list of 200,000 numbers versus a set made from it. Repeat with a list of 20,000 numbers and compare how the ratio changes.
9. Explain why `{[1, 2], [3]}` fails, and show two ways to store the same information in a set.
10. Given two dictionaries, find the keys they share, the keys unique to each, and the keys whose values differ. (Dictionary key views support set operations.)
11. Write a function `anagram_groups(words)` that groups words that are anagrams of each other, using `frozenset` or sorted letters as the key.
12. Extend `diff_tags` with an `unchanged_count` and make `require_permissions` accept a special permission `"*"` that grants everything. Write tests covering both.

## Summary

- A set is an unordered collection of unique, hashable items. Create it with `{...}`, `set(iterable)`, or a comprehension, and use `set()` for an empty one.
- `add`, `update`, `remove` (raises), `discard` (does not), and `pop` (arbitrary) change a set.
- Union, intersection, difference, and symmetric difference have operators (`|`, `&`, `-`, `^`) that need sets and methods that accept any iterable.
- `<=`, `>=`, `<`, `isdisjoint`, and `==` compare sets.
- Membership tests on a set take constant time on average, which can make them thousands of times faster than on a list.
- Sets have no order, and the order of a set of strings can change between runs. Sort when you need predictable output.
- `frozenset` is the immutable, hashable version, usable as a dictionary key or set member.
- Use `dict.fromkeys(items)` to remove duplicates while preserving order.
