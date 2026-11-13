## Overview

The `functools` module is the standard library's toolbox for working with functions. It collects the ready-made tools that people kept rewriting: fixing some arguments of a function in advance, remembering results so that expensive work is not repeated, folding a sequence into one value, choosing an implementation by the type of an argument, and computing a property only once.

You have already met one member, `functools.wraps`, in Chapter 3.6. This chapter covers the rest of the everyday set, each of which solves a problem you will meet in real programs.

By the end of this note you will be able to:

- Create specialized functions with `partial`.
- Speed up repeated calls with `lru_cache` and `cache`, and know their limits.
- Combine a sequence into a single value with `reduce`, and know when not to.
- Write functions that behave differently by argument type with `singledispatch`.
- Compute an attribute once and keep it with `cached_property`.

> [!note] Why this module matters
> Each tool here replaces a pattern that would otherwise take a closure, a decorator, or a long `if isinstance(...)` chain. Knowing they exist saves you from writing and debugging that code yourself.

## Core concepts

### The toolbox at a glance

| Tool | Solves |
| --- | --- |
| `partial` | Fix some arguments of a function in advance |
| `lru_cache`, `cache` | Remember results of calls with the same arguments |
| `reduce` | Fold a sequence into one value, step by step |
| `singledispatch` | Choose a function implementation by the type of the first argument |
| `cached_property` | Compute an attribute on first use, then keep it |
| `wraps` | Preserve metadata in decorators (Chapter 3.6) |
| `total_ordering` | Generate all comparison methods from `__eq__` and one other (Chapter 6.3) |
| `cmp_to_key` | Convert an old-style comparison function into a `key` function |

### partial

`partial(func, *args, **kwargs)` returns a new callable that behaves like `func` with those arguments already filled in. Calling the new function supplies the remaining arguments. It is a clean alternative to a small lambda or closure, and one of the three fixes for the late-binding bug (Chapter 3.5).

### Memoization with lru_cache and cache

**Memoization** means remembering the result of a call so that a later call with the same arguments returns the stored answer instead of recalculating it.

| Decorator | Behavior |
| --- | --- |
| `@lru_cache(maxsize=128)` | Keeps at most `maxsize` results, discarding the **least recently used** when full |
| `@lru_cache(maxsize=None)` | Unbounded, never discards |
| `@cache` | Shorthand for the unbounded version |

Both require every argument to be **hashable**, because the arguments are used as a dictionary key. Both add helper methods: `cache_info()` reports hits and misses, and `cache_clear()` empties the cache.

### reduce

`reduce(function, iterable, initial)` applies a two-argument function cumulatively: it combines the first two items, then combines that result with the third, and so on, ending with a single value. The optional `initial` value is the starting point, and is also what you get back for an empty iterable.

For the most common jobs, dedicated functions are clearer: `sum`, `max`, `min`, `math.prod`, `any`, `all`, and `"".join`.

### singledispatch

A function decorated with `@singledispatch` becomes a **generic function**: its default behavior is the function you wrote, and you register additional implementations for specific types. When called, Python looks at the type of the **first argument** and runs the best-matching implementation. It replaces long chains of `if isinstance(...)` and keeps each case in its own small function that other modules can extend.

### cached_property

`@cached_property` turns a method into an attribute whose value is computed the first time it is read and then stored on the instance. Later reads return the stored value without calling the method again. It is used inside classes (Chapter 6.2).

### What gets remembered, and for how long

| Tool | Where the result lives | How to discard it |
| --- | --- | --- |
| `lru_cache` or `cache` | In the decorated function | `func.cache_clear()` |
| `cached_property` | In the instance's own `__dict__` | `del instance.attribute` |
| `partial` | Nothing is remembered, only arguments | Not applicable |

## Basic syntax examples

### partial

```text
>>> from functools import partial
>>> int_base2 = partial(int, base=2)
>>> int_base2("1010")
10
>>> power_of_two = partial(pow, 2)
>>> power_of_two(5)
32
>>> int_base2.func is int, int_base2.args, int_base2.keywords
(True, (), {'base': 2})
>>> def greet(greeting, name):
...     return f"{greeting}, {name}!"
...
>>> hello = partial(greet, "Hello")
>>> hello("Ada")
'Hello, Ada!'
>>> hello_ada = partial(greet, name="Ada")
>>> hello_ada("Hi")
'Hi, Ada!'
```

`partial` objects expose `func`, `args`, and `keywords`, so you can see what they wrap. They also work well as callbacks and as arguments to `map`:

```python
from functools import partial

print(list(map(partial(int, base=16), ["ff", "10", "1a"])))
```

```text
[255, 16, 26]
```

### lru_cache

```python
from functools import lru_cache


@lru_cache(maxsize=128)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)


print(fib(50))
print(fib.cache_info())
fib.cache_clear()
print(fib.cache_info())
```

```text
12586269025
CacheInfo(hits=48, misses=51, maxsize=128, currsize=51)
CacheInfo(hits=0, misses=0, maxsize=128, currsize=0)
```

Without the cache, this recursive Fibonacci would need over twenty billion calls to compute `fib(50)`. With it, the function body runs just 51 times (the misses), and the 48 hits are free. Chapter 3.8 explains why.

### cache

```text
>>> from functools import cache
>>> calls = []
>>> @cache
... def square(n):
...     calls.append(n)
...     return n * n
...
>>> square(4), square(4), square(5)
(16, 16, 25)
>>> calls
[4, 5]
```

The function body ran only twice, once for `4` and once for `5`.

### Arguments must be hashable

```text
>>> from functools import cache
>>> @cache
... def total(items):
...     return sum(items)
...
>>> total([1, 2])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
>>> total((1, 2))
3
```

### reduce

```text
>>> from functools import reduce
>>> import operator
>>> reduce(operator.add, [1, 2, 3, 4])
10
>>> reduce(lambda acc, n: acc * n, [1, 2, 3, 4], 1)
24
>>> reduce(operator.add, [], 0)
0
>>> reduce(operator.add, [])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: reduce() of empty iterable with no initial value
>>> reduce(lambda total, word: total + len(word), ["a", "bb", "ccc"], 0)
6
>>> reduce(lambda a, b: a if a > b else b, [3, 9, 4])
9
>>> import math
>>> math.prod([1, 2, 3, 4])
24
```

Here is how the first call proceeds: `1 + 2` is `3`, then `3 + 3` is `6`, then `6 + 4` is `10`. Always supply an initial value when the input might be empty. The last lines show the simpler tools you should prefer when they exist.

### A reduce that has no simpler replacement

```python
from functools import reduce


def merge(settings, layer):
    return {**settings, **layer}


layers = [{"color": "red", "size": "M"}, {"size": "L"}, {"debug": True}]
print(reduce(merge, layers, {}))
```

```text
{'color': 'red', 'size': 'L', 'debug': True}
```

Folding a list of dictionaries into one, later layers winning, is a good fit for `reduce`.

### singledispatch

```text
>>> from functools import singledispatch
>>> @singledispatch
... def describe(value):
...     return f"object {value!r}"
...
>>> @describe.register
... def _(value: int):
...     return f"integer {value}"
...
>>> @describe.register
... def _(value: list):
...     return f"list of {len(value)} items"
...
>>> @describe.register(float)
... def _(value):
...     return f"float {value:.1f}"
...
>>> describe(3), describe([1, 2]), describe(2.5), describe("x")
('integer 3', 'list of 2 items', 'float 2.5', "object 'x'")
>>> describe(True)
'integer True'
```

`describe("x")` found no registered string version, so it used the default. `describe(True)` used the `int` version because `bool` is a subclass of `int`. When several registered types match, the most specific one wins. A registration can name the type through the argument's annotation, as above, or explicitly, as in `register(float)`. The function name `_` is a convention for "I never call this directly".

### cached_property

```python
from functools import cached_property


class Report:
    def __init__(self, rows):
        self.rows = rows

    @cached_property
    def total(self):
        print("computing total")
        return sum(self.rows)


report = Report([1, 2, 3])
print(report.total)
print(report.total)
report.rows.append(10)
print(report.total)
del report.total
print(report.total)
```

```text
computing total
6
6
6
computing total
16
```

The work ran once. After the rows changed, the stored value was **stale** until `del report.total` forced a recomputation. Cached values do not update themselves.

## Deeper behavior

### How `lru_cache` decides what to keep

With a size limit, the cache keeps the most recently **used** entries. Reading an entry marks it as fresh, and when the cache is full the entry that has gone longest without use is thrown away. Choose a `maxsize` that matches how many distinct argument combinations you expect to see in active use. A power of two is customary but not required.

### The cache key is built from the arguments

Two calls can hit the same entry when their arguments are **equal and hash alike**. That has some surprising consequences:

```text
>>> from functools import cache, lru_cache
>>> @cache
... def pair(a, b):
...     return (a, b)
...
>>> pair(1, 2)
(1, 2)
>>> pair(1.0, 2.0)
(1, 2)
>>> pair.cache_info().misses
1
>>> @lru_cache
... def add(a=0, b=0):
...     return a + b
...
>>> add(a=1, b=2), add(b=2, a=1)
(3, 3)
>>> add.cache_info().misses
2
```

The second call passed floats, but because `1 == 1.0` and `2 == 2.0`, the cache returned the stored result for the integers. If the type of an argument matters, use `@lru_cache(typed=True)`, which keeps the types in the key. Exactly which equal values share an entry is partly an implementation detail (for example, a lone plain `int` or `str` argument is keyed in a special fast way), so do not rely on either behavior. Calls that pass the same keyword arguments in a different order count as **different** calls, so they use separate entries, as the last two lines show.

### Cached results are shared

A cache returns **the same object** every time (Chapter 2.5). If the function returns a list or dictionary and a caller modifies it, the cache is corrupted for everyone. Cache only functions that return immutable values (numbers, strings, tuples, frozen dataclasses), or return copies.

### Only cache pure functions

Memoization is correct only if the same arguments **always** give the same result and the function has no side effects you rely on. A function that reads the clock, a file, or a database may return different answers on different days, and the cache would hide that. Caching a function that prints will print only on the first call.

### Caches never expire on their own

`lru_cache` has no time limit. A cached "current price" would stay stale forever. For time-based expiry, call `cache_clear()` on a schedule, include a time bucket in the arguments (such as the current minute), or use a third-party library built for the purpose.

### Unbounded caches grow without limit

`@cache` and `maxsize=None` keep every result for the life of the process. If the arguments come from users or requests, an attacker or a busy day can make the cache consume all memory. Prefer a bounded `lru_cache` in services.

### Do not put `lru_cache` on instance methods carelessly

A cached method uses `self` as part of the key, so the cache holds a reference to every instance it has ever seen, preventing them from being freed. For per-instance results, use `cached_property`, or cache a module-level function that takes the needed values rather than `self`.

### Thread safety

The cache itself is safe to use from several threads, but the decorated function may run more than once for the same arguments if two threads miss at the same moment. If the work must happen exactly once, add a lock (Chapter 10.2).

### `reduce` is rarely the best choice

Readers must mentally unroll a `reduce` to see what it does. Prefer `sum`, `math.prod`, `max`, `min`, `any`, `all`, `"".join`, or a plain loop with a named accumulator. Use `reduce` when the combining step is a real function you already have, as in the dictionary-merging example.

### `singledispatch` details

- It dispatches only on the **first argument**. Methods in classes use `singledispatchmethod`.
- Implementations can be registered from other modules, so third parties can extend a function you wrote. That makes it a lightweight plug-in mechanism.
- It dispatches on the **type** (and its parents), not on values. For decisions about the shape of data, use `match` (Chapter 1.6).
- A union type such as `int | float` can be used in an annotation to register one function for both.

### `cached_property` details

It stores the value in the instance's `__dict__` under the same name, which is why later reads are as fast as a normal attribute. It needs the instance to have a `__dict__`, so it does not work with classes that declare `__slots__` without one. It does not lock, so two threads may compute it at the same time. Use it for values that are expensive and do not change for the life of the object.

### `partial` versus `lambda`

| | `partial(func, 2)` | `lambda x: func(2, x)` |
| --- | --- | --- |
| Shows what it wraps | Yes: `.func`, `.args`, `.keywords` | No |
| Can be pickled (sent to another process) | Yes | No |
| Evaluates arguments | Immediately, so no late-binding trap | When called |
| Can do extra work or rearrange arguments | No | Yes |

### Other members

`total_ordering` generates `<`, `<=`, `>`, `>=` from `__eq__` and one other comparison (Chapter 6.3). `cmp_to_key` adapts a legacy "compare two items and return negative, zero, or positive" function for use as a `key`. `reduce` lives here, rather than among the built-ins, because it is considered a specialist tool.

## Industry-level snippet

Backends convert Python values to JSON constantly, and the standard `json.dumps` cannot handle decimals, timestamps, sets, identifiers, or dataclasses. This module builds a JSON encoder from three `functools` tools: `singledispatch` to choose a conversion by type (each case a small, separate function), `partial` to pre-configure `json.dumps` once, and `lru_cache` to avoid reparsing the same duration strings. The cached function returns an `int`, so sharing the cached result is safe.

```python
"""JSON encoding and duration parsing, built from functools tools."""

import json
import logging
from collections.abc import Set
from dataclasses import asdict, is_dataclass
from datetime import date, datetime
from decimal import Decimal
from functools import lru_cache, partial, singledispatch
from pathlib import PurePath
from typing import Any
from uuid import UUID

logger = logging.getLogger(__name__)

UNIT_SECONDS = {"d": 86_400, "h": 3_600, "m": 60, "s": 1}


@singledispatch
def to_jsonable(value: object) -> object:
    """Convert a value that json.dumps cannot handle into one it can.

    This default handles dataclasses. Other types are registered below.

    Raises:
        TypeError: If the type is not supported.
    """
    if is_dataclass(value) and not isinstance(value, type):
        return asdict(value)
    raise TypeError(f"cannot serialize {type(value).__name__} to JSON")


@to_jsonable.register
def _(value: Decimal) -> str:
    return str(value)  # strings keep every digit exact


@to_jsonable.register
def _(value: date) -> str:
    return value.isoformat()


@to_jsonable.register
def _(value: datetime) -> str:
    if value.tzinfo is None:
        raise ValueError("naive datetimes are not allowed; attach a time zone")
    return value.isoformat()


@to_jsonable.register
def _(value: UUID) -> str:
    return str(value)


@to_jsonable.register
def _(value: PurePath) -> str:
    return value.as_posix()


@to_jsonable.register(set)
@to_jsonable.register(frozenset)
def _(value: Set[Any]) -> list[Any]:
    return sorted(value, key=str)  # sets have no order, so sort for stable output


to_json = partial(
    json.dumps,
    default=to_jsonable,
    sort_keys=True,
    separators=(",", ":"),
    ensure_ascii=False,
)


@lru_cache(maxsize=256)
def parse_duration(text: str) -> int:
    """Convert text such as "1h30m" or "45s" into a positive number of seconds.

    Raises:
        ValueError: If the text is not made of number-and-unit pairs.
    """
    total = 0
    digits = ""
    for char in text.strip().lower():
        if char.isdigit():
            digits += char
        elif char in UNIT_SECONDS and digits:
            total += int(digits) * UNIT_SECONDS[char]
            digits = ""
        else:
            raise ValueError(f"invalid duration: {text!r}")
    if digits or total == 0:
        raise ValueError(f"invalid duration: {text!r}")
    return total
```

Try it:

```python
from dataclasses import dataclass
from datetime import timezone


@dataclass
class Invoice:
    number: str
    total: Decimal
    issued: datetime
    tags: set[str]


invoice = Invoice(
    "A-1", Decimal("19.99"), datetime(2026, 3, 1, 12, 0, tzinfo=timezone.utc), {"vip", "eu"}
)
print(to_json(invoice))

try:
    to_json({"when": datetime(2026, 3, 1, 12, 0)})
except ValueError as exc:
    print("ValueError:", exc)

try:
    to_json({"thing": object()})
except TypeError as exc:
    print("TypeError:", exc)

print(parse_duration("1h30m"), parse_duration("1h30m"), parse_duration("45s"))
print(parse_duration.cache_info())
```

```text
{"issued":"2026-03-01T12:00:00+00:00","number":"A-1","tags":["eu","vip"],"total":"19.99"}
ValueError: naive datetimes are not allowed; attach a time zone
TypeError: cannot serialize object to JSON
5400 5400 45
CacheInfo(hits=1, misses=2, maxsize=256, currsize=2)
```

Why it is written this way:

- Each type has its **own small function**. Supporting a new type means adding one more registered function, even from another module, without touching existing code. This is the open-for-extension idea behind many design patterns (Chapter 13.4).
- `datetime` is a subclass of `date`, and `singledispatch` correctly picks the more specific `datetime` function. It also refuses datetimes without a time zone, which are a classic source of bugs (Chapter 12.1).
- `json.dumps` calls the `default` function only for objects it cannot handle itself, and calls it again for anything unusual in the result, so a dataclass holding a `Decimal` and a `set` is converted level by level.
- `partial` fixes the encoder's settings once, so every call site just writes `to_json(value)`. The sorted keys and compact separators give stable, byte-for-byte reproducible output.
- `parse_duration` is pure and returns an immutable `int`, so caching it is safe. Failed calls raise exceptions, and exceptions are never cached.
- The cache is bounded to 256 entries, so unusual user input cannot exhaust memory.

## Common mistakes

1. **Caching a function that returns a mutable object** and letting callers modify it. Return tuples or copies.
2. **Caching a function that depends on the clock, files, a database, or randomness.** The cache will serve stale or wrong answers.
3. **Using `@cache` or `maxsize=None` in a server with user-controlled arguments,** creating a memory leak.
4. **Passing unhashable arguments** such as lists or dictionaries to a cached function. Convert to tuples.
5. **Putting `lru_cache` on an instance method** and keeping every instance alive.
6. **Expecting the cache to know about time.** It never expires entries on its own.
7. **Forgetting that equal values such as `1` and `1.0` can share a cache entry** unless you use `typed=True`.
8. **Using `reduce` without an initial value** on input that might be empty.
9. **Using `reduce` for sums, products, and maximums,** where `sum`, `math.prod`, and `max` read better.
10. **Expecting `singledispatch` to dispatch on the second argument or on the value.** It uses the type of the first argument only.
11. **Treating a `cached_property` as if it updates automatically.** It stays stale until deleted.
12. **Writing a lambda to fix arguments when `partial` is clearer,** or the reverse, using `partial` when arguments need rearranging.

## Practice exercises

1. Use `partial` to create `to_int_hex` from `int` with `base=16`, and use it with `map` on a list of hexadecimal strings.
2. Create `log_error = partial(print, "ERROR:", sep=" ")` and use it. Print `log_error.func`, `.args`, and `.keywords`.
3. Write a slow function (use `time.sleep(0.2)`) that returns the square of a number. Time three identical calls with and without `@cache`.
4. Apply `@lru_cache(maxsize=2)` to a function, call it with three different arguments, and use `cache_info()` and the order of calls to demonstrate which entry was evicted.
5. Explain why `@cache` on a function taking a list fails, and write a wrapper that accepts a list by converting it to a tuple.
6. Demonstrate that a cached function returning a list can be corrupted by a caller. Fix it two ways.
7. Use `reduce` to compute the greatest common divisor of a list of numbers with `math.gcd`, and then to flatten a list of lists. Compare the flattening with a comprehension.
8. Write `describe(value)` with `singledispatch` for `int`, `str`, `list`, `dict`, and a default. Add a registration for `bool` and explain why it needs one.
9. Use `singledispatch` to write `size_in_bytes(value)` that handles `str` (UTF-8 length), `bytes`, `list` (sum of item sizes), and a default that raises a clear error.
10. Write a class with an expensive `cached_property`, show that it is computed once, show that it becomes stale, and refresh it with `del`.
11. Write a two-argument cached function and show that calling it with `(1, 2)` and then `(1.0, 2.0)` returns the stored result for the first call. Then fix it with `typed=True` and show that the second call is now computed separately.
12. Extend the JSON module with support for `bytes` (as a base64 string) and for `Enum` members (as their value). Write tests that include a nested structure containing every supported type.

## Summary

- `functools` collects the standard tools for working with functions.
- `partial(func, *args, **kwargs)` makes a new function with some arguments pre-filled. It is clearer than a lambda for that job, and avoids the late-binding trap.
- `lru_cache(maxsize=...)` and `cache` memoize results by arguments. Arguments must be hashable. Use `cache_info()` and `cache_clear()` to inspect and reset.
- Cache only pure functions that return immutable values, bound the cache size in servers, and remember that caches never expire on their own.
- `reduce(function, iterable, initial)` folds a sequence into one value. Prefer `sum`, `max`, `math.prod`, and loops when they are clearer, and give an initial value for possibly empty input.
- `singledispatch` picks a function implementation by the type of the first argument, with the default as a fallback and the most specific registered type winning.
- `cached_property` computes an attribute once per instance and stores it. It does not refresh itself.
- Together these tools replace a lot of hand-written closures, decorators, and `isinstance` chains.
