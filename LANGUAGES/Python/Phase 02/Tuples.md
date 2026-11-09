## Overview

A **tuple** is an ordered collection like a list, with one decisive difference: once created, it cannot be changed. That sounds like a limitation, but it is exactly what makes tuples valuable. A tuple says, "these values belong together and will not change." It is the natural way to represent a point, a database row, a date, a version number, or the several values a function returns at once. Because tuples cannot change, they can also be used as dictionary keys and set members.

By the end of this note you will be able to:

- Create tuples, including the tricky one-item tuple.
- Pack and unpack values, including with the star operator.
- Use tuples as lightweight records and as multiple return values.
- Explain why a tuple can be a dictionary key and a list cannot.
- Decide when a tuple is a better choice than a list.

> [!note] Immutable does not always mean unchangeable inside
> A tuple cannot gain, lose, or swap items. But if one of its items is itself a mutable object, such as a list, that object can still be changed. The tuple holds the references fixed, not the contents of what they point to.

## Core concepts

### Creating tuples

A tuple is created by separating values with commas. The parentheses are usually added for readability, and are required in some places to avoid ambiguity.

| Way | Example | Notes |
| --- | --- | --- |
| Parentheses | `(3, 4)` | The usual style |
| Without parentheses | `3, 4` | The commas make the tuple |
| Empty | `()` | The only tuple with no commas |
| One item | `(5,)` | The trailing comma is **required** |
| From an iterable | `tuple([1, 2])` | Converts lists, strings, ranges, and more |

The one-item case catches everyone once. `(5)` is simply the number `5` in parentheses. It is the comma, not the brackets, that makes a tuple.

### Reading a tuple

Indexing, negative indexes, slicing, `len`, `in`, `+`, `*`, and comparisons all work like they do for lists and strings. Slicing a tuple gives a tuple. Tuples have only two methods:

| Method | Meaning |
| --- | --- |
| `count(x)` | How many items equal `x` |
| `index(x)` | Position of the first item equal to `x` |

There is nothing like `append` or `sort`, because the tuple cannot change.

### Packing and unpacking

**Packing** means putting values into a tuple. **Unpacking** means pulling them back out into separate names. The number of names must match the number of values, unless you use a star.

```python
point = 3, 4         # packing
x, y = point         # unpacking
```

Unpacking works with any iterable, not only tuples: lists, strings, ranges, and dictionary views too.

### Star unpacking

A name prefixed with `*` collects "all the rest" into a list. Only one starred name is allowed per assignment.

```python
first, *middle, last = [1, 2, 3, 4, 5]
```

### Ignoring values

By convention, `_` is a name for a value you do not care about: `_, year, _ = record`.

### Tuples as records

A tuple is the simplest **record**: a fixed group of values in which each position has a meaning, such as `(name, birth_year, city)`. The weakness is that the meaning lives only in your head, because `record[1]` does not say "birth year". When a record has more than two or three fields, prefer a `namedtuple` (Chapter 2.6) or a dataclass (Chapter 6.7), which give each position a name.

### Returning several values

Functions can return several values by returning a tuple. The caller then unpacks the result:

```python
def min_max(values):
    return min(values), max(values)


low, high = min_max([3, 9, 1])
```

### Hashability

An object is **hashable** if Python can compute a fixed number (a hash) from its value that never changes during its lifetime. Immutable objects such as numbers, strings, and tuples of hashable items are hashable. Dictionary keys and set members must be hashable (Chapters 2.3 and 2.4). A tuple is hashable only if every item inside it is hashable, so `(1, 2)` can be a key, but `([1], 2)` cannot.

### Tuple or list?

| Question | Tuple | List |
| --- | --- | --- |
| Can it change after creation? | No | Yes |
| Typical contents | A few values of different meaning (a record) | Many values of the same kind (a collection) |
| Can it be a dictionary key? | Yes, if its items are hashable | No |
| Length known and fixed by design? | Usually | Usually not |
| Example | `(latitude, longitude)` | A list of all customer names |

## Basic syntax examples

### Creating and reading

```text
>>> point = (3, 4)
>>> point[0]
3
>>> point[-1]
4
>>> x, y = point
>>> x, y
(3, 4)
>>> 1, 2, 3
(1, 2, 3)
>>> empty = ()
>>> len(empty)
0
>>> single = (5,)
>>> type(single), type((5))
(<class 'tuple'>, <class 'int'>)
>>> tuple("abc")
('a', 'b', 'c')
>>> tuple([1, 2])
(1, 2)
```

### Immutability

```text
>>> point = (3, 4)
>>> point[0] = 10
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
```

To "change" a tuple, build a new one:

```text
>>> point = (3, 4)
>>> point = (10,) + point[1:]
>>> point
(10, 4)
```

### Operations

```text
>>> (1, 2) + (3,)
(1, 2, 3)
>>> (1, 2) * 2
(1, 2, 1, 2)
>>> (3, 1, 2).count(1)
1
>>> (3, 1, 2).index(2)
2
>>> 2 in (1, 2, 3)
True
>>> (1, 2, 3)[1:]
(2, 3)
>>> sorted((3, 1, 2))
[1, 2, 3]
>>> tuple(sorted((3, 1, 2)))
(1, 2, 3)
```

Note that `sorted` always returns a list. Convert it back with `tuple(...)` if you need a tuple.

### Comparing tuples

```text
>>> (1, 2, 3) < (1, 2, 4)
True
>>> (1, 5) < (2, 0)
True
>>> (1, 5) == (1, 5)
True
```

Tuples compare item by item, left to right, stopping at the first difference. That makes them perfect for multi-level sorting and for version numbers.

### Unpacking

```text
>>> a, b = 1, 2
>>> a, b = b, a
>>> a, b
(2, 1)
>>> a, (b, c), d = 1, (2, 3), 4
>>> a, b, c, d
(1, 2, 3, 4)
>>> _, year, _ = ("Ada", 1815, "London")
>>> year
1815
```

The swap works because the right side is packed into a tuple first, then unpacked.

### Star unpacking

```text
>>> first, *middle, last = [1, 2, 3, 4, 5]
>>> first, middle, last
(1, [2, 3, 4], 5)
>>> head, *tail = "abc"
>>> head, tail
('a', ['b', 'c'])
>>> first, *rest = [10]
>>> first, rest
(10, [])
```

The starred name always receives a **list**, which may be empty.

### Wrong number of values

```text
>>> x, y = 1, 2, 3
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: too many values to unpack (expected 2)
>>> x, y, z = 1, 2
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: not enough values to unpack (expected 3, got 2)
```

### Unpacking in loops

```python
people = [("Ada", 36), ("Grace", 45), ("Linus", 28)]

for name, age in people:
    print(f"{name} is {age}")

for position, (name, age) in enumerate(people, start=1):
    print(position, name)
```

```text
Ada is 36
Grace is 45
Linus is 28
1 Ada
2 Grace
3 Linus
```

You have already been unpacking tuples: `enumerate`, `zip`, and `dict.items()` all produce them.

### Returning several values

```python
def min_max(values):
    return min(values), max(values)


low, high = min_max([3, 9, 1])
print(low, high)
print(min_max([3, 9, 1]))
```

```text
1 9
(1, 9)
```

### Tuples as dictionary keys

```text
>>> locations = {(0, 0): "origin", (1, 2): "treasure"}
>>> locations[(1, 2)]
'treasure'
>>> {[1, 2]: "x"}
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

### A tuple can hold a changeable item

```text
>>> mixed = ([1, 2], 3)
>>> mixed[0].append(99)
>>> mixed
([1, 2, 99], 3)
>>> hash((1, 2)) == hash((1, 2))
True
>>> hash(([1], 2))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

The tuple itself did not change, because it still holds the same list. But the list inside it did, and the tuple with a list inside is not hashable.

### Passing tuples to calls

```python
point = (3, 4)
print(*point)                            # unpack into separate arguments
print("report.csv".endswith((".csv", ".tsv")))   # a tuple of alternatives
print(isinstance(3.5, (int, float)))
```

```text
3 4
True
True
```

Several built-in functions accept a tuple to mean "any of these".

### Constants

```python
ALLOWED_METHODS = ("GET", "POST", "DELETE")
```

A tuple is ideal for a fixed group of constants, since nobody can accidentally modify it.

## Deeper behavior

### Shallow immutability

A tuple guarantees that its **references** never change. It says nothing about the objects they point to. When a tuple contains only immutable items (numbers, strings, other such tuples), the whole structure is effectively frozen. If it contains a list or a dictionary, treat it as partly mutable and do not use it as a key.

### `+=` on a tuple creates a new tuple

```python
t = (1, 2)
u = t
t += (3,)
print(t, u)
```

```text
(1, 2, 3) (1, 2)
```

This is the same behavior as Chapter 1.2: with immutable objects, an "update" rebinds the name to a new object.

### Tuples are smaller and slightly faster

Because they never change size, tuples need less memory than lists holding the same items. On CPython 3.12, a three-item tuple takes 64 bytes and a three-item list takes 88. Constant tuples in your source code are even built once, at compile time. The difference is real but small, so do not pick tuples for speed. Pick them for meaning.

### Tuple comparison powers multi-level sorting

Return a tuple from the sort key to sort by several criteria. Negating a number reverses its order within the tuple:

```python
employees = [("Ada", "Eng", 120), ("Grace", "Eng", 150), ("Linus", "Ops", 90)]

ordered = sorted(employees, key=lambda e: (e[1], -e[2]))
print(ordered)
```

```text
[('Grace', 'Eng', 150), ('Ada', 'Eng', 120), ('Linus', 'Ops', 90)]
```

The result is sorted by department, then by salary from highest to lowest.

### The trailing comma trap

```python
name = "Ada",          # a tuple! the trailing comma made it one
print(type(name))
```

```text
<class 'tuple'>
```

A stray comma at the end of a line is a classic source of confusing bugs.

### Unpacking works with any iterable

```python
a, b, c = "xyz"
first, second = range(2)
key, value = {"only": 1}.popitem()
print(a, b, c, first, second, key, value)
```

```text
x y z 0 1 only 1
```

### Records: position versus name

```python
employee = ("Ada", "Engineering", 120_000)
print(employee[2])
```

What is index `2`? You have to remember. A `namedtuple` (Chapter 2.6) or dataclass (Chapter 6.7) lets you write `employee.salary` instead. Use bare tuples for short, obvious groupings such as coordinates and key-value pairs.

### Tuples in type hints

Type hints distinguish a fixed-shape tuple, `tuple[int, str]`, from a variable-length one, `tuple[int, ...]`. Chapter 8.1 covers the details.

## Industry-level snippet

Software versions are a perfect real-world tuple: `(major, minor, patch)`. Comparing them as text gives wrong answers (`"1.10.0"` sorts before `"1.9.0"`), but comparing tuples of integers works naturally. This module parses version strings, checks compatibility ranges, and picks the newest acceptable release from a list, ignoring bad entries with a log message instead of crashing.

```python
"""Version parsing and compatibility checks using tuples."""

import logging

logger = logging.getLogger(__name__)

Version = tuple[int, int, int]


def parse_version(text: str) -> Version:
    """Convert text such as "v1.4.2" into the tuple (1, 4, 2).

    Raises:
        ValueError: If the text is not three non-negative whole numbers
            separated by dots.
    """
    parts = text.strip().removeprefix("v").split(".")
    if len(parts) != 3:
        raise ValueError(f"expected MAJOR.MINOR.PATCH, got {text!r}")
    try:
        major, minor, patch = (int(part) for part in parts)
    except ValueError as exc:
        raise ValueError(f"version parts must be whole numbers, got {text!r}") from exc
    if min(major, minor, patch) < 0:
        raise ValueError(f"version parts must not be negative, got {text!r}")
    return major, minor, patch


def format_version(version: Version) -> str:
    """Convert (1, 4, 2) back into "1.4.2"."""
    major, minor, patch = version
    return f"{major}.{minor}.{patch}"


def is_compatible(version: Version, minimum: Version, below: Version) -> bool:
    """Return True if minimum <= version < below."""
    return minimum <= version < below


def newest_compatible(candidates: list[str], minimum: str, below: str) -> str | None:
    """Pick the highest candidate in the half-open range [minimum, below).

    Candidates that cannot be parsed are logged and skipped.

    Returns:
        The newest acceptable version as text, or None if there is none.
    """
    low, high = parse_version(minimum), parse_version(below)
    best: Version | None = None
    for text in candidates:
        try:
            version = parse_version(text)
        except ValueError as exc:
            logger.warning("Ignoring candidate %r: %s", text, exc)
            continue
        if is_compatible(version, low, high) and (best is None or version > best):
            best = version
    return None if best is None else format_version(best)
```

Try it:

```python
print("1.10.0" < "1.9.0")                       # comparing as text: wrong
print(parse_version("1.10.0") < parse_version("1.9.0"))   # comparing as tuples: right

releases = ["1.8.3", "1.10.0", "2.0.0", "oops", "1.9.9", "v1.2.0"]
print(newest_compatible(releases, minimum="1.5.0", below="2.0.0"))
print(newest_compatible(releases, minimum="3.0.0", below="4.0.0"))
```

```text
True
False
1.10.0
None
```

The `oops` entry is skipped with a warning, so one bad value cannot break the whole selection.

Points to notice:

- The type alias `Version` documents what the tuple means. A reader does not have to guess what `(1, 4, 2)` represents.
- A generator expression `(int(part) for part in parts)` is unpacked straight into three names.
- `minimum <= version < below` chains tuple comparisons, which compare item by item.
- The function returns `None` to say "nothing suitable", and the caller decides what to do (Chapter 1.2).
- Parsing errors keep their original cause attached with `from exc`.

## Common mistakes

1. **Forgetting the comma in a one-item tuple.** `(5)` is just `5`. Write `(5,)`.
2. **Adding a stray trailing comma** after a value, which silently creates a tuple.
3. **Trying to modify a tuple.** Build a new one, or use a list if the data must change.
4. **Unpacking into the wrong number of names**, which raises `ValueError`. Use a starred name to absorb the extra items.
5. **Assuming a tuple is deeply frozen.** A list inside a tuple can still change.
6. **Using a tuple that contains a list as a dictionary key or set member.** It is unhashable.
7. **Using a tuple when positions are not obvious**, such as a six-field record. Use a `namedtuple` or dataclass.
8. **Comparing version numbers as strings.** Compare tuples of integers.
9. **Expecting `sorted(some_tuple)` to return a tuple.** It returns a list.
10. **Forgetting that the starred name in unpacking is a list**, not a tuple.
11. **Using a list for fixed constants** such as allowed methods, which anyone can modify by accident. Use a tuple.

## Practice exercises

1. Create a tuple with your name, age, and city. Print each item using indexing, then unpack it into three variables and print them again.
2. Show the difference between `(5)` and `(5,)` with `type()`, and write a function `ensure_tuple(value)` that returns the value unchanged if it is a tuple and wraps it in a one-item tuple otherwise.
3. Swap three variables in a rotation using tuple unpacking in a single line.
4. Given `data = (10, 20, 30, 40, 50)`, use star unpacking to get the first value, the last value, and the middle ones as a list. Do it again for a tuple of length 2 and length 1 and describe what happens to the middle.
5. Write `divide(a, b)` that returns `(quotient, remainder)` using `divmod`, and call it with unpacking. Handle a zero divisor by returning `None`, then explain why that makes callers' unpacking risky.
6. Given the list `[("Ada", 36), ("Grace", 45), ("Linus", 28)]`, produce two separate lists, one of names and one of ages, using `zip(*pairs)` and unpacking. Explain what the star does.
7. Store points on a grid in a dictionary keyed by `(row, column)` tuples, and write code that finds all points in a given row.
8. Create a tuple containing a list. Modify the list, and show the tuple changing. Then try to use the tuple as a dictionary key and read the error.
9. Sort a list of `(name, department, salary)` records by department ascending and salary descending, using a tuple key.
10. Write `parse_point(text)` that turns `"3,4"` into the tuple `(3, 4)` of integers, rejecting anything with the wrong number of parts or non-numeric parts with a clear error.
11. Explain why `("a", "b")` is better than `["a", "b"]` for a constant that lists allowed values, and give a case where it would still be wrong to use a tuple.
12. Extend the version module with a function `bump(version, part)` that returns a new version tuple with the major, minor, or patch part increased and the lower parts reset to zero. Reject an unknown part name.

## Summary

- A tuple is an ordered, immutable sequence. The comma creates it, so a single item needs a trailing comma: `(5,)`.
- Tuples support indexing, slicing, concatenation, repetition, comparison, and the two methods `count` and `index`.
- Packing builds a tuple, unpacking splits it into names. A starred name collects the remaining items into a list, and `_` marks a value you ignore.
- Tuples are the natural way to return several values from a function and to represent small fixed records. Use a `namedtuple` or dataclass when the fields need names.
- Tuples compare item by item, which makes them ideal for multi-level sorting and version numbers.
- A tuple is hashable only if all its items are, so tuples of immutable values can be dictionary keys and set members.
- Immutability is shallow: a mutable object stored inside a tuple can still change.
- Use a tuple for fixed groups of different values, and a list for growing collections of similar values.
