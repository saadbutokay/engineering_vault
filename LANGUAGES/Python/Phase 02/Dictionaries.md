## Overview

A **dictionary** stores **key-value pairs**. Instead of asking "what is at position 3?", you ask "what is the value for the key `"email"`?" Dictionaries are the workhorse of Python: they represent records, settings, JSON documents, lookup tables, counters, caches, and indexes. Nearly every web API you will ever write reads and produces dictionaries.

Dictionaries are also fast. Looking up a key takes about the same time whether the dictionary holds ten items or ten million, because of a technique called hashing.

By the end of this note you will be able to:

- Create dictionaries and read, add, update, and delete entries.
- Choose between `d[key]`, `d.get(key)`, `setdefault`, and `in`.
- Loop over keys, values, and pairs.
- Merge dictionaries and build them with comprehensions.
- Explain why dictionary keys must be hashable and why lookups are fast.
- Rely on the guaranteed insertion order of dictionaries.

> [!note] Dictionary order
> Since Python 3.7, a dictionary remembers the order in which keys were first inserted, and that is part of the language, not an accident. Two dictionaries with the same pairs in a different order are still equal.

## Core concepts

### Keys and values

Each **key** is unique within a dictionary and is associated with one **value**. Keys must be **hashable** (immutable in practice): strings, numbers, booleans, `None`, and tuples of hashable items all qualify. Lists, dictionaries, and sets do not. Values can be anything, including other dictionaries and lists.

### Creating dictionaries

| Way | Example |
| --- | --- |
| Literal | `{"name": "Ada", "age": 36}` |
| Empty | `{}` or `dict()` |
| Keyword arguments | `dict(name="Ada", age=36)` |
| From pairs | `dict([("x", 1), ("y", 2)])` |
| From two lists | `dict(zip(keys, values))` |
| Same value for many keys | `dict.fromkeys(["a", "b"], 0)` |
| Comprehension | `{n: n * n for n in range(4)}` |

Note that `{}` makes an empty **dictionary**, not an empty set. An empty set is `set()` (Chapter 2.4).

### Reading and changing entries

| Task | Code | If the key is missing |
| --- | --- | --- |
| Read | `d[key]` | Raises `KeyError` |
| Read safely | `d.get(key)` or `d.get(key, default)` | Returns `None` or the default |
| Test for a key | `key in d` | Returns `False` |
| Add or update | `d[key] = value` | Adds the key |
| Delete | `del d[key]` | Raises `KeyError` |
| Delete and return | `d.pop(key)` or `d.pop(key, default)` | Raises `KeyError` unless a default is given |

The `in` operator tests **keys**, not values.

### Dictionary methods

| Method | What it does |
| --- | --- |
| `get(key, default=None)` | Read without raising |
| `setdefault(key, default)` | Return the value if the key exists, otherwise insert the default and return it |
| `pop(key[, default])` | Remove a key and return its value |
| `popitem()` | Remove and return the most recently inserted pair |
| `update(other)` | Add all pairs from another mapping, overwriting duplicates |
| `keys()`, `values()`, `items()` | Views of the keys, values, and `(key, value)` pairs |
| `copy()` | A shallow copy |
| `clear()` | Remove everything |

### Iteration

Looping over a dictionary directly gives its **keys**. To get values or pairs, use `values()` or `items()`:

```python
for key in d: ...
for value in d.values(): ...
for key, value in d.items(): ...
```

### Merging

Two modern ways to combine dictionaries, and the older method:

- `a | b` creates a new dictionary containing the pairs of both.
- `a |= b` updates `a` in place with the pairs of `b`.
- `{**a, **b}` unpacks both into a new dictionary.
- `a.update(b)` is the older in-place way.

In every case, when both have the same key, the **right-hand** value wins.

### Dictionary comprehensions

Like list comprehensions, but producing key-value pairs: `{key_expr: value_expr for item in iterable if condition}`.

### Hashing in one paragraph

A **hash** is a number computed from an object's value, and equal objects always have equal hashes. A dictionary uses the hash of a key to jump almost directly to the place where its value is stored, instead of checking every key. That is why lookups are fast on average regardless of size, and why keys must be immutable: if a key could change, its hash would change, and the dictionary could no longer find it.

## Basic syntax examples

### Reading and changing

```text
>>> user = {"name": "Ada", "age": 36}
>>> user["name"]
'Ada'
>>> user["email"]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'email'
>>> user.get("email")
>>> user.get("email", "unknown")
'unknown'
>>> "age" in user
True
>>> user["age"] = 37
>>> user["city"] = "London"
>>> user
{'name': 'Ada', 'age': 37, 'city': 'London'}
>>> del user["city"]
>>> user.pop("age")
37
>>> user.pop("missing", None)
>>> user
{'name': 'Ada'}
```

`user.get("email")` returned `None`, so the REPL showed nothing. Assigning to an existing key replaces the value, and assigning to a new key adds it.

### Creating

```text
>>> dict(a=1, b=2)
{'a': 1, 'b': 2}
>>> dict([("x", 1), ("y", 2)])
{'x': 1, 'y': 2}
>>> dict(zip(["a", "b"], [1, 2]))
{'a': 1, 'b': 2}
>>> dict.fromkeys(["a", "b"], 0)
{'a': 0, 'b': 0}
>>> {n: n * n for n in range(4)}
{0: 0, 1: 1, 2: 4, 3: 9}
>>> {name: len(name) for name in ["ada", "grace"]}
{'ada': 3, 'grace': 5}
>>> {k: v for k, v in {"a": 1, "b": -2, "c": 3}.items() if v > 0}
{'a': 1, 'c': 3}
```

### Views and loops

```text
>>> stock = {"apple": 3, "pear": 0, "plum": 7}
>>> list(stock)
['apple', 'pear', 'plum']
>>> list(stock.values())
[3, 0, 7]
>>> list(stock.items())
[('apple', 3), ('pear', 0), ('plum', 7)]
>>> len(stock)
3
>>> sum(stock.values())
10
```

```python
stock = {"apple": 3, "pear": 0, "plum": 7}

for fruit, count in stock.items():
    print(f"{fruit}: {count}")
```

```text
apple: 3
pear: 0
plum: 7
```

### Counting with get

```python
text = "to be or not to be"
counts = {}
for word in text.split():
    counts[word] = counts.get(word, 0) + 1
print(counts)
```

```text
{'to': 2, 'be': 2, 'or': 1, 'not': 1}
```

`counts.get(word, 0)` supplies `0` the first time a word appears. Chapter 2.6 shows `Counter`, which does this for you.

### Grouping with setdefault

```text
>>> groups = {}
>>> groups.setdefault("a", []).append(1)
>>> groups.setdefault("a", []).append(2)
>>> groups.setdefault("b", []).append(3)
>>> groups
{'a': [1, 2], 'b': [3]}
```

`setdefault` returns the existing list if the key is there, or inserts a new empty list and returns that. Either way you can append immediately.

### Merging

```text
>>> defaults = {"color": "red", "size": "M"}
>>> overrides = {"size": "L"}
>>> defaults | overrides
{'color': 'red', 'size': 'L'}
>>> {**defaults, **overrides, "extra": 1}
{'color': 'red', 'size': 'L', 'extra': 1}
>>> defaults
{'color': 'red', 'size': 'M'}
>>> defaults |= overrides
>>> defaults
{'color': 'red', 'size': 'L'}
>>> defaults.update({"size": "XL"})
>>> defaults
{'color': 'red', 'size': 'XL'}
```

The first two lines left `defaults` untouched, while `|=` and `update` changed it.

### Nested dictionaries

```text
>>> config = {"db": {"host": "localhost", "port": 5432}}
>>> config["db"]["port"]
5432
>>> config.get("cache", {}).get("ttl", 60)
60
```

The second form is a safe way to read through several levels when any level might be missing.

### Sorting a dictionary

```text
>>> scores = {"ada": 90, "grace": 85, "linus": 95}
>>> sorted(scores)
['ada', 'grace', 'linus']
>>> sorted(scores.items(), key=lambda item: item[1], reverse=True)
[('linus', 95), ('ada', 90), ('grace', 85)]
>>> max(scores, key=scores.get)
'linus'
>>> dict(sorted(scores.items(), key=lambda item: item[1]))
{'grace': 85, 'ada': 90, 'linus': 95}
>>> {score: name for name, score in scores.items()}
{90: 'ada', 85: 'grace', 95: 'linus'}
```

A dictionary has no `sort` method. Sort its items (which gives a list of pairs), and rebuild a dictionary if you need one.

### Which keys are allowed

```text
>>> {(1, 2): "ok"}
{(1, 2): 'ok'}
>>> {[1, 2]: "x"}
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
>>> {1: "int", 1.0: "float", True: "bool"}
{1: 'bool'}
```

The last example shows that `1`, `1.0`, and `True` are equal and hash alike, so they are the **same key**. The first key inserted (`1`) is kept, and the last value (`"bool"`) wins.

### Order

```text
>>> d = {"b": 1, "a": 2}
>>> d["c"] = 3
>>> list(d)
['b', 'a', 'c']
>>> {"a": 1, "b": 2} == {"b": 2, "a": 1}
True
```

Keys come out in insertion order, yet equality ignores order.

## Deeper behavior

### How a dictionary finds a key

Picture a large array of slots. To store a key, Python computes `hash(key)`, uses that number to pick a slot, and puts the key and value there. To look it up, it does the same calculation and goes straight to the slot, then confirms the key matches using `==`. If two keys land in the same slot (a **collision**), Python probes a few other slots in a predictable pattern. With a good hash and some spare room, the average lookup, insert, and delete take **constant time**, no matter how big the dictionary is.

```text
hash("apple") -> some large number -> slot 3 -> ("apple", 3)
hash("pear")  -> some large number -> slot 1 -> ("pear", 0)
```

The slot numbers above are only an illustration of the idea, since the real hash values and slot layout are internal details.

Two consequences follow:

- **Keys must be hashable.** An object whose value can change cannot have a stable hash.
- **Equal keys must hash equally.** That is why `1`, `1.0`, and `True` collapse into one key.

### String hashes are randomized

For security reasons, Python gives string hashes a random seed each time the interpreter starts. The result is that a **set** of strings may print in a different order on each run. Dictionaries are not affected, because they track insertion order separately. Chapter 2.4 returns to this.

### `d[key]` versus `d.get(key)`

Use `d[key]` when the key must exist and its absence is a bug, so you want the error. Use `d.get(key, default)` when absence is normal. The `get` method never changes the dictionary. `setdefault` does: it inserts the default when the key is missing.

### The default value is created every time

```python
d = {}
d.setdefault("a", [])      # a new empty list is built even if "a" exists
```

For expensive defaults, check with `in` first, or use `collections.defaultdict` (Chapter 2.6), which calls a factory only when it needs to.

### Dictionary views are live

`d.keys()`, `d.values()`, and `d.items()` return **views**, which reflect later changes to the dictionary:

```python
inventory = {"apple": 3}
names = inventory.keys()
inventory["pear"] = 5
print(list(names))
```

```text
['apple', 'pear']
```

Convert to a list when you need a snapshot.

### Do not change the size while iterating

```text
>>> d = {"a": 1}
>>> for key in d:
...     d["b"] = 2
...
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
RuntimeError: dictionary changed size during iteration
```

Loop over a copy of the keys, `for key in list(d):`, or build a new dictionary with a comprehension. Changing the **value** of an existing key during the loop is allowed.

### Shallow copies

`d.copy()`, `dict(d)`, and `{**d}` all make shallow copies. The new dictionary has the same value objects, so a list stored as a value is shared. Chapter 2.5 explains how to avoid trouble.

### Dictionaries and JSON

JSON objects map directly to dictionaries: `json.loads` returns them, `json.dumps` accepts them (Chapter 5.6). JSON keys are always strings, so a dictionary with integer keys becomes one with string keys after a round trip.

### Merging precedence

In `a | b`, `{**a, **b}`, and `a.update(b)`, the right-hand dictionary wins on conflicts. Put the defaults on the left and the overrides on the right.

### Performance summary

| Operation | Average cost |
| --- | --- |
| `d[key]`, `d.get(key)`, `key in d` | Constant |
| `d[key] = value`, `del d[key]` | Constant |
| Iterating over all items | Proportional to the size |
| `value in d.values()` | Proportional to the size |

Searching by **value** is slow. If you often need to find an item by something other than the key, build a second dictionary indexed by that field.

### Reverse iteration and `popitem`

Since Python 3.8 you can write `reversed(d)` to walk keys in reverse insertion order, and `popitem()` removes the most recently added pair, which lets you treat a dictionary as a stack.

## Industry-level snippet

Backend code spends much of its time reorganizing data: grouping events by user, building an index for fast lookups, and layering configuration from several sources. This module provides three reusable tools: `group_by`, `index_by` (which refuses duplicates), and a `deep_merge` that combines nested configuration dictionaries **without modifying or sharing** its inputs.

```python
"""Dictionary helpers for grouping, indexing, and layering configuration."""

import copy
import logging
from collections.abc import Callable, Hashable, Iterable, Mapping
from typing import Any

logger = logging.getLogger(__name__)


class DuplicateKeyError(ValueError):
    """Raised when index_by finds two items that map to the same key."""


def group_by[K: Hashable, V](
    items: Iterable[V], key: Callable[[V], K]
) -> dict[K, list[V]]:
    """Group items into lists keyed by the value that key() returns."""
    groups: dict[K, list[V]] = {}
    for item in items:
        groups.setdefault(key(item), []).append(item)
    return groups


def index_by[K: Hashable, V](
    items: Iterable[V], key: Callable[[V], K]
) -> dict[K, V]:
    """Build a dictionary for constant-time lookups by a unique key.

    Raises:
        DuplicateKeyError: If two items produce the same key.
    """
    index: dict[K, V] = {}
    for item in items:
        item_key = key(item)
        if item_key in index:
            raise DuplicateKeyError(f"duplicate key: {item_key!r}")
        index[item_key] = item
    return index


def deep_merge(
    base: Mapping[str, Any], override: Mapping[str, Any]
) -> dict[str, Any]:
    """Combine two nested configuration mappings into a new dictionary.

    Values in override win. When both sides hold a mapping under the same key,
    the mappings are merged recursively. Neither input is modified, and the
    result shares no mutable objects with them.
    """
    merged: dict[str, Any] = copy.deepcopy(dict(base))
    for key, value in override.items():
        existing = merged.get(key)
        if isinstance(existing, Mapping) and isinstance(value, Mapping):
            merged[key] = deep_merge(existing, value)
        else:
            merged[key] = copy.deepcopy(value)
    logger.debug("Merged configuration keys: %s", sorted(merged))
    return merged
```

Try it:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Event:
    user: str
    action: str


events = [Event("ada", "login"), Event("grace", "login"), Event("ada", "purchase")]

by_user = group_by(events, key=lambda event: event.user)
print({user: [e.action for e in items] for user, items in by_user.items()})

try:
    index_by(events, key=lambda event: event.user)
except DuplicateKeyError as exc:
    print("DuplicateKeyError:", exc)

defaults = {"db": {"host": "localhost", "port": 5432}, "debug": False}
override = {"db": {"host": "db.internal"}, "debug": True}
merged = deep_merge(defaults, override)
print(merged)
print(defaults)
```

```text
{'ada': ['login', 'purchase'], 'grace': ['login']}
DuplicateKeyError: duplicate key: 'ada'
{'db': {'host': 'db.internal', 'port': 5432}, 'debug': True}
{'db': {'host': 'localhost', 'port': 5432}, 'debug': False}
```

Why it is written this way:

- `group_by` and `index_by` take a **key function**, the same idea as `sorted(key=...)`, so they work with any record type.
- `index_by` turns a linear search into a constant-time lookup, and fails loudly on duplicate keys instead of silently overwriting data.
- `deep_merge` copies before it combines, so changing the merged configuration later cannot corrupt the defaults. This is the lesson of Chapter 2.5 in practice.
- The recursion applies only where both sides are mappings. Any other value (a list, a number) is replaced wholesale, which is the least surprising rule.
- The type parameter `K: Hashable` states in the signature the requirement that keys be hashable.

## Common mistakes

1. **Using `d[key]` when the key might be missing.** Use `get`, or check with `in`.
2. **Using a list or dictionary as a key.** Convert to a tuple, or use a `frozenset`.
3. **Changing the size of a dictionary while looping over it.** Loop over `list(d)` or build a new one.
4. **Forgetting that `{}` is an empty dictionary,** not an empty set.
5. **Using `in` to test values.** `"Ada" in user` checks keys. Use `"Ada" in user.values()`, which is slow.
6. **Assuming `copy()` is deep.** Nested lists and dictionaries are shared.
7. **Relying on order for equality.** `{"a": 1, "b": 2} == {"b": 2, "a": 1}` is true.
8. **Believing `setdefault` and `get` are the same.** One inserts, one does not.
9. **Mixing up `1`, `1.0`, and `True` as keys.** They collide.
10. **Searching values in a loop to find a record** when a second index dictionary would make it instant.
11. **Writing `for key in d.keys():` out of habit.** `for key in d:` is equivalent.
12. **Merging in the wrong order,** letting defaults overwrite overrides.
13. **Mutating a dictionary a function received.** Copy it first, or return a new one.

## Practice exercises

1. Create a dictionary describing a book with title, author, and year. Print each value, add a `pages` key, change the year, and delete the author.
2. Write a program that counts how many times each character appears in a sentence, using `get`. Then print the three most frequent characters using `sorted` and a key function.
3. Given two lists `keys = ["a", "b", "c"]` and `values = [1, 2, 3]`, build a dictionary in three different ways.
4. Invert the dictionary `{"ada": 1, "grace": 2, "linus": 1}`. What happens to the duplicate value, and how would you keep all the names instead? Write both versions.
5. Group a list of words by their first letter into a dictionary of lists, using `setdefault`.
6. Given `prices = {"apple": 1.2, "pear": 0.8, "plum": 2.5}`, use a comprehension to produce a new dictionary with every price increased by 10 percent and rounded to two decimals, containing only items that cost more than 1 after the increase.
7. Merge `{"a": 1, "b": 2}` and `{"b": 20, "c": 30}` using `|`, using `{**a, **b}`, and using `update`. Which of the three change an existing dictionary?
8. Write a function `safe_get(data, *path, default=None)` that reads through nested dictionaries, returning the default when any step is missing.
9. Demonstrate the error from changing a dictionary's size during iteration, and then fix it in two different ways.
10. Build a phone book as a dictionary, and a second dictionary that maps phone numbers back to names. Explain why having both helps.
11. Show that `1`, `1.0`, and `True` collide as keys, and explain why it happens in terms of hashing and equality.
12. Extend `deep_merge` with an optional parameter that makes list values **concatenate** instead of replace. Write three test cases, and show that the inputs are unchanged.

## Summary

- A dictionary maps unique, hashable keys to values. Create it with literals, `dict(...)`, `zip`, `fromkeys`, or a comprehension.
- Use `d[key]` when the key must exist, `d.get(key, default)` when it may not, and `key in d` to test for a key. `in` checks keys, not values.
- `setdefault` inserts a default and returns the value, which is ideal for grouping. `pop`, `popitem`, `update`, `keys`, `values`, and `items` complete the toolkit.
- Looping gives keys by default, `items()` gives `(key, value)` pairs, and views are live.
- Merge with `a | b`, `a |= b`, `{**a, **b}`, or `update`. The right-hand side wins on conflicts.
- Dictionaries keep insertion order, and equality ignores it.
- Lookups, inserts, and deletes take constant time on average because keys are hashed. This requires keys to be hashable, and equal keys to have equal hashes.
- Searching by value is slow, so index by whatever field you look up most.
- Never change a dictionary's size while looping over it, and remember that copies are shallow.
