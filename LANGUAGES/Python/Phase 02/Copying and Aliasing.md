## Overview

Chapter 1.2 introduced the idea that variables are names pointing at objects, and that assignment never copies. This chapter follows that idea to its practical conclusion. Whenever you have a list, dictionary, or set, there are two very different things you might mean by "make another one": give the **same** object a second name, or create a **new** object with the same contents. Confusing the two is behind a large share of real-world bugs, from configuration that mysteriously changes to caches that return corrupted data.

By the end of this note you will be able to:

- Recognize when two names refer to the same object (aliasing).
- Make shallow copies and explain exactly what they do and do not copy.
- Make deep copies, and know what they cost.
- Predict the behavior of nested structures after copying.
- Avoid the most common aliasing bugs: shared grids, mutable defaults, and mutated arguments.
- Choose designs that make these bugs impossible.

> [!note] The one sentence to remember
> A shallow copy duplicates the **container** but shares the **contents**. A deep copy duplicates everything, all the way down.

## Core concepts

### Aliasing

**Aliasing** means that two or more names refer to the same object. It happens through plain assignment, through function arguments, through items stored in containers, and through default values.

```python
a = [1, 2, 3]
b = a            # b is an alias: one list, two names
```

Aliasing is not a bug. It is how Python avoids copying large data when it does not have to. It becomes a bug when you change the object through one name and forget that the other name sees the change.

### Identity versus equality

`a == b` asks whether two objects have equal contents. `a is b` asks whether they are the same object. After copying, you expect `==` to be true and `is` to be false.

### Shallow copy

A **shallow copy** creates a new container and fills it with references to the same items as the original. For a flat list of numbers, strings, or other immutable items, that is a complete, independent copy. For a container that holds mutable items, the new container and the old one **share** those items.

| Type | Ways to make a shallow copy |
| --- | --- |
| list | `items.copy()`, `items[:]`, `list(items)`, `copy.copy(items)` |
| dict | `d.copy()`, `dict(d)`, `{**d}`, `copy.copy(d)` |
| set | `s.copy()`, `set(s)`, `copy.copy(s)` |

### Deep copy

A **deep copy**, `copy.deepcopy(x)`, recursively copies the container and every container inside it, so the result shares nothing mutable with the original. It is the right tool when you need a truly independent copy of a nested structure.

### Summary of the options

| Statement | Creates a new container? | Copies the items? |
| --- | --- | --- |
| `b = a` | No | No |
| `b = a.copy()` | Yes | No (shares them) |
| `b = copy.deepcopy(a)` | Yes | Yes, recursively |

### Immutable objects need no copying

Numbers, strings, tuples of immutable items, and `frozenset` cannot change, so sharing them is always safe. Python's `copy.copy` simply returns the same object for them.

## Basic syntax examples

### Alias versus copy

```text
>>> a = [1, 2, 3]
>>> b = a
>>> b is a
True
>>> c = a.copy()
>>> c is a, c == a
(False, True)
>>> c.append(4)
>>> a
[1, 2, 3]
>>> b.append(99)
>>> a
[1, 2, 3, 99]
```

`c` is independent, so changing it did not affect `a`. `b` is an alias, so changing it did.

### Other ways to copy a list

```text
>>> x = [1, 2]
>>> y = x[:]
>>> z = list(x)
>>> y is x, z is x
(False, False)
>>> y == x == z
True
```

### The shallow copy trap

```text
>>> nested = [[1, 2], [3, 4]]
>>> shallow = nested.copy()
>>> shallow[0].append(99)
>>> nested
[[1, 2, 99], [3, 4]]
>>> shallow[0] is nested[0]
True
```

The outer lists are different objects, but both contain the **same inner lists**. Modifying an inner list is visible through both.

Replacing an entire item, on the other hand, affects only the copy:

```text
>>> shallow[1] = ["x"]
>>> nested[1]
[3, 4]
```

The distinction: **changing an object that two containers share** is visible to both, **rebinding a slot in one container** is not.

### Deep copy

```text
>>> import copy
>>> nested = [[1, 2], [3, 4]]
>>> deep = copy.deepcopy(nested)
>>> deep[0].append(100)
>>> nested
[[1, 2], [3, 4]]
>>> deep
[[1, 2, 100], [3, 4]]
>>> deep[0] is nested[0]
False
```

### Dictionaries

```text
>>> original = {"tags": ["a"], "n": 1}
>>> alias = original
>>> shallow = original.copy()
>>> shallow["n"] = 2
>>> original["n"]
1
>>> shallow["tags"].append("b")
>>> original["tags"]
['a', 'b']
>>> alias is original
True
```

Changing the integer under `"n"` replaced a slot in the copy only. Appending to the list under `"tags"` changed an object that both dictionaries hold.

### Immutable objects

```text
>>> import copy
>>> t = (1, 2)
>>> copy.copy(t) is t
True
>>> s = "abc"
>>> copy.copy(s) is s
True
```

### The repetition trap, again

```text
>>> rows = [[0] * 2] * 3
>>> rows[0].append(9)
>>> rows
[[0, 0, 9], [0, 0, 9], [0, 0, 9]]
>>> safe = [[0] * 2 for _ in range(3)]
>>> safe[0].append(9)
>>> safe
[[0, 0, 9], [0, 0], [0, 0]]
```

Multiplying a list repeats **references**, not copies. All three rows are one list.

### A function that changes its argument

```python
def add_default_role(roles):
    roles.append("viewer")        # changes the caller's list


def with_default_role(roles):
    return [*roles, "viewer"]     # builds a new list


mine = ["editor"]
print(with_default_role(mine), mine)
add_default_role(mine)
print(mine)
```

```text
['editor', 'viewer'] ['editor']
['editor', 'viewer']
```

The first function is safe for the caller. The second changed the caller's data, which may or may not be what they expected.

### The mutable default argument

```python
def add_item(item, items=[]):
    items.append(item)
    return items


print(add_item("a"))
print(add_item("b"))
```

```text
['a']
['a', 'b']
```

The default list `[]` is created **once**, when the function is defined, and every call that omits `items` shares it. The standard fix uses `None`:

```python
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

Chapter 3.2 covers this in depth.

### Cycles and deepcopy

```text
>>> import copy
>>> node = {"name": "a"}
>>> node["self"] = node
>>> clone = copy.deepcopy(node)
>>> clone["self"] is clone
True
>>> clone is node
False
```

`deepcopy` keeps track of what it has already copied, so even a structure that refers to itself is copied correctly.

## Deeper behavior

### What a shallow copy really costs and shares

A shallow copy is cheap: it copies one reference per item, so the time is proportional to the number of items, but it never duplicates the items themselves. That is why it is the default for `copy()`. A deep copy is much more expensive: it visits every object reachable from the container, and on a big structure it can be far slower than a shallow copy and uses much more memory.

### Copying objects of your own classes

`copy.copy` and `copy.deepcopy` also work on instances of your own classes. A shallow copy of an instance shares its attributes:

```python
import copy
from dataclasses import dataclass, field


@dataclass
class Cart:
    items: list[str] = field(default_factory=list)


original = Cart(["apple"])
shallow = copy.copy(original)
deep = copy.deepcopy(original)

shallow.items.append("pear")
print(original.items)
print(deep.items)
```

```text
['apple', 'pear']
['apple']
```

### The cache that hands out the same list

Caching a function result is a common optimization (Chapter 3.7). But a cache returns the **same object** every time, so a caller who modifies it corrupts the cache for everyone:

```python
from functools import lru_cache


@lru_cache(maxsize=None)
def default_roles() -> list[str]:
    return ["viewer"]


roles = default_roles()
roles.append("admin")
print(default_roles())
```

```text
['viewer', 'admin']
```

The fix is to return an immutable value, such as a tuple, or to return a copy.

### Prefer designs that avoid the problem

The cleanest solution to aliasing bugs is not to defend with copies everywhere, but to design so that sharing is harmless:

- Use **immutable** data (tuples, `frozenset`, frozen dataclasses) for values that should not change. Sharing them is always safe.
- Write functions that **return new objects** instead of changing their arguments.
- Update nested data by **copying only the path you change** and sharing the rest. This is called structural sharing or copy-on-write, and the industry snippet below demonstrates it.
- Copy **at the boundary**: when you store data someone else gave you, or hand out data you keep, copy it once at that point.

### Choosing the right tool

| Situation | Tool |
| --- | --- |
| Flat list or dictionary of immutable items | Shallow copy |
| Nested structure that you will modify independently | `copy.deepcopy` |
| Only one branch of a nested structure changes | Copy the path and share the rest |
| You never modify the data | No copy at all |
| The value is naturally constant | Make it immutable |

### The JSON round-trip trick

You may see `json.loads(json.dumps(data))` used as a deep copy. It works only for data that JSON can represent, converts tuples to lists, turns non-string keys into strings, and is slower than `deepcopy`. Do not use it as a general copying tool.

### Customizing copying

A class can control how it is copied by defining `__copy__` and `__deepcopy__` methods (Chapter 6.3). Objects that wrap resources such as open files, network connections, or locks usually cannot be deep-copied at all and raise an error.

### Shared state and concurrency

Two threads that hold aliases of the same mutable object can change it at the same time, producing hard-to-reproduce bugs (Chapter 10.2). Handing each worker its own copy, or giving it only immutable data, avoids a whole class of races.

### How to investigate

When data changes unexpectedly, check identity. `a is b` tells you if they are the same object, and `id(x)` shows the identity number. Following a suspicious object through your code with `id()` quickly reveals where it is shared.

## Industry-level snippet

Applications often hold a nested configuration and must let different parts of the program read and override it without affecting each other. This module provides a deep `snapshot` for callers who need a private copy to edit, and a `set_in` function that returns an updated configuration by copying **only the dictionaries along the changed path** and sharing every untouched branch. It also demonstrates why sharing is safe as long as nobody mutates it.

```python
"""Safe handling of nested configuration: snapshots and copy-on-write updates."""

import copy
import logging
from collections.abc import Mapping, Sequence
from typing import Any

logger = logging.getLogger(__name__)


class ConfigPathError(ValueError):
    """Raised when a configuration path is empty or cannot be followed."""


def snapshot(config: Mapping[str, Any]) -> dict[str, Any]:
    """Return an independent deep copy that the caller may modify freely."""
    return copy.deepcopy(dict(config))


def get_in(config: Mapping[str, Any], path: Sequence[str], default: Any = None) -> Any:
    """Read a nested value by path, returning default if any step is missing."""
    node: Any = config
    for key in path:
        if not isinstance(node, Mapping) or key not in node:
            return default
        node = node[key]
    return node


def set_in(
    config: Mapping[str, Any], path: Sequence[str], value: Any
) -> dict[str, Any]:
    """Return a new configuration with value stored at path.

    Only the dictionaries along the path are copied. All other branches are
    shared with the original, so treat the result as read-only data.

    Raises:
        ConfigPathError: If the path is empty or runs through a non-mapping.
    """
    if not path:
        raise ConfigPathError("path must contain at least one key")
    key, *rest = path
    updated = dict(config)  # shallow copy of this level only
    if rest:
        child = config.get(key, {})
        if not isinstance(child, Mapping):
            raise ConfigPathError(
                f"cannot descend into {key!r}: it holds a {type(child).__name__}"
            )
        updated[key] = set_in(child, rest, value)
    else:
        updated[key] = value
    logger.debug("Updated configuration path %s", ".".join(path))
    return updated
```

Try it:

```python
base = {
    "db": {"host": "localhost", "port": 5432},
    "cache": {"ttl": 60, "nodes": ["a", "b"]},
}

changed = set_in(base, ["db", "host"], "db.internal")

print(get_in(base, ["db", "host"]))
print(get_in(changed, ["db", "host"]))
print(changed["db"] is base["db"])        # the changed branch was copied
print(changed["cache"] is base["cache"])  # the untouched branch is shared

private = snapshot(base)
private["cache"]["nodes"].append("c")
print(base["cache"]["nodes"])
print(get_in(base, ["cache", "missing", "deeper"], default="n/a"))
```

```text
localhost
db.internal
False
True
['a', 'b']
n/a
```

Why it is written this way:

- `snapshot` uses `deepcopy` because its contract is "you own this and may edit it". Its cost is acceptable because it is called rarely.
- `set_in` pays only for the path it changes. Updating one setting in a configuration with thousands of keys copies a handful of small dictionaries, not the whole tree.
- Sharing untouched branches is safe here because the functions **never mutate** anything. The discipline is to treat configuration values as read-only, and to take a `snapshot` when you need to edit.
- The `isinstance(..., Mapping)` checks give clear errors instead of confusing `TypeError`s when a path runs into a number or a string.
- The original `base` is untouched by `set_in`, which is easy to verify, and which makes the function safe to use anywhere.

## Common mistakes

1. **Using `b = a` and expecting a copy.** It creates a second name for one object.
2. **Assuming `copy()` or `[:]` copies nested items.** They are shallow.
3. **Building a grid with `[[0] * n] * m`.** All rows are the same list.
4. **Using a mutable default argument** such as `items=[]`. Use `None` and create the list inside.
5. **Modifying a function's argument** without telling the caller. Return a new object, or document the change.
6. **Returning a cached or shared mutable object** that callers might modify.
7. **Reaching for `deepcopy` everywhere "to be safe".** It is slow and hides design problems. Prefer immutable data and copying at boundaries.
8. **Copying a structure that holds resources** such as file handles, sockets, or locks. Deep copy cannot duplicate them.
9. **Using the JSON round-trip** as a general-purpose deep copy.
10. **Comparing with `==` to check that something is a copy.** Equal contents do not mean separate objects. Use `is`.
11. **Mutating a collection that you stored from a caller**, which then changes the caller's data. Copy it when you receive it.
12. **Forgetting that dictionary values that are lists are shared** by `dict.copy()`.

## Practice exercises

1. Create a list `a`, make `b` an alias and `c` a copy. Change `b` and `c` in different ways and explain each result using `is` and `==`.
2. Given `matrix = [[1, 2], [3, 4]]`, make a shallow copy and a deep copy. Modify an inner list through each copy and show which one affects the original.
3. Demonstrate that `[[0] * 3] * 3` and `[[0] * 3 for _ in range(3)]` behave differently, then explain why in terms of object identity.
4. Write a function `remove_duplicates(items)` that **does not** modify the list it receives, and a second version that modifies it in place and returns `None`. Show how a caller would notice the difference.
5. Reproduce the mutable-default bug in a function that collects log messages. Fix it two ways, one with `None` and one by avoiding the default entirely.
6. Make a dictionary that contains a list and another dictionary. Make a shallow copy and show which changes are shared and which are not.
7. Write a function that takes a dictionary of user settings and returns a **new** dictionary with one value changed. Prove with `is` that the original and the new one differ, and that untouched nested objects are shared.
8. Create a structure that contains a reference to itself, and show that `copy.deepcopy` handles it while a hand-written recursive copy function would loop forever.
9. Write a function decorated with `lru_cache` that returns a list, show how a caller can corrupt the cache, and then fix it in two ways.
10. Measure the time of `items.copy()` versus `copy.deepcopy(items)` for a list of 100,000 small lists. Explain the difference.
11. Use a dataclass with a list field. Show the difference between `copy.copy`, `copy.deepcopy`, and `dataclasses.replace` on an instance.
12. Extend `set_in` so that it can create missing intermediate dictionaries, with a parameter `create_missing=False` that, when false, raises `ConfigPathError` instead. Write tests for both settings, and show that the original is never changed.

## Summary

- Assignment never copies. `b = a` makes two names for one object, which is called aliasing.
- A **shallow copy** (`copy()`, `[:]`, `list()`, `dict()`, `copy.copy`) creates a new container but shares the items inside it. For flat structures of immutable items that is a full copy.
- A **deep copy** (`copy.deepcopy`) copies everything recursively and handles cycles, but costs more in time and memory.
- Changing an object shared by two containers is visible in both, while replacing a slot in one container is not.
- Immutable objects (numbers, strings, tuples of immutable items) are safe to share and never need copying.
- The classic aliasing bugs are shared grids from `*`, mutable default arguments, functions that modify their arguments, and caches that return shared mutable objects.
- Prefer designs that make sharing harmless: immutable data, functions that return new objects, copy-on-write updates, and copying at the boundaries where data enters or leaves your code.
- When something changes unexpectedly, check identity with `is` and `id()`.
