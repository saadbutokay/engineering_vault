## Overview

The built-in list, tuple, dictionary, and set cover most needs, but a few patterns come up so often that Python's standard library provides specialized containers for them. The `collections` module offers five that every working Python developer should know:

| Container | Solves |
| --- | --- |
| `defaultdict` | A dictionary that creates missing values automatically |
| `Counter` | Counting things |
| `deque` | A fast queue or buffer, with cheap additions and removals at both ends |
| `namedtuple` | A tuple whose positions have names |
| `ChainMap` | Several dictionaries searched as if they were one |

By the end of this note you will be able to:

- Group and count data in one line, without checking whether keys exist.
- Build queues, sliding windows, and fixed-size buffers with `deque`.
- Create lightweight records with named fields.
- Layer settings from several sources with `ChainMap`.
- Pick the right container for a problem.

> [!note] Import style
> Each container is imported by name: `from collections import defaultdict, Counter, deque, namedtuple, ChainMap`. They behave like the built-in types they extend, so the skills you already have carry over.

## Core concepts

### defaultdict

A `defaultdict` is a dictionary with a **factory**: a function it calls to create a value whenever you read a key that does not exist. You give the factory when you create it.

| Factory | Missing key starts as | Typical use |
| --- | --- | --- |
| `list` | `[]` | Grouping items |
| `int` | `0` | Counting |
| `set` | `set()` | Collecting unique items per key |
| `lambda: ...` | Anything you choose | Custom defaults |

Compare it with the plain dictionary approach of Chapter 2.3: no `get`, no `setdefault`, no `if key not in`.

### Counter

A `Counter` is a dictionary specialized for counting hashable items. You can build it from any iterable, and a missing item counts as zero instead of raising `KeyError`.

| Feature | Meaning |
| --- | --- |
| `Counter(iterable)` | Count each distinct item |
| `c[item]` | The count, `0` if absent |
| `c.most_common(n)` | The `n` most frequent items, as `(item, count)` pairs |
| `c.update(iterable)` | Add counts from more data |
| `c.subtract(iterable)` | Subtract counts |
| `c.elements()` | Repeat each item as often as its count |
| `c.total()` | Sum of all counts |
| `c1 + c2`, `c1 - c2` | Combine counts, keeping only positive results |

### deque

A `deque` (pronounced "deck", short for double-ended queue) is a sequence that supports **fast additions and removals at both ends**. A list is slow at the front because every item must shift. A deque is not.

| Operation | Meaning |
| --- | --- |
| `append(x)`, `appendleft(x)` | Add to the right or left end |
| `pop()`, `popleft()` | Remove from the right or left end |
| `extend(it)`, `extendleft(it)` | Add many items at one end |
| `rotate(n)` | Shift items by `n` places, wrapping around |
| `deque(maxlen=n)` | A bounded buffer that drops items from the opposite end when full |

### namedtuple

`namedtuple` creates a new tuple type whose positions also have names. It is still a tuple, so it can be unpacked, indexed, compared, hashed, and is immutable. It just reads better: `point.x` instead of `point[0]`.

| Feature | Meaning |
| --- | --- |
| `Point = namedtuple("Point", ["x", "y"])` | Define the type |
| `Point(3, 4)` | Create an instance |
| `p.x`, `p[0]` | Read by name or position |
| `p._replace(x=10)` | A new instance with a changed field |
| `p._asdict()` | The fields as a dictionary |
| `Point._fields` | The tuple of field names |

### ChainMap

A `ChainMap` links several dictionaries so they act as one. A lookup tries the first dictionary, then the next, and so on. **Writes and deletes go to the first dictionary only.** The underlying dictionaries are not copied, so changes to them show through.

It is perfect for layered configuration, where command-line options beat environment variables, which beat defaults.

### OrderedDict (briefly)

Since ordinary dictionaries now remember insertion order, `OrderedDict` is rarely needed. It still offers `move_to_end()` and `popitem(last=False)`, which make it convenient for building a least-recently-used cache.

## Basic syntax examples

### defaultdict: grouping

```text
>>> from collections import defaultdict
>>> groups = defaultdict(list)
>>> groups["a"].append(1)
>>> groups["a"].append(2)
>>> groups["b"].append(3)
>>> dict(groups)
{'a': [1, 2], 'b': [3]}
```

### defaultdict: counting

```text
>>> from collections import defaultdict
>>> counts = defaultdict(int)
>>> for char in "hello":
...     counts[char] += 1
...
>>> dict(counts)
{'h': 1, 'e': 1, 'l': 2, 'o': 1}
```

### defaultdict: grouping words by first letter

```python
from collections import defaultdict

words = ["apple", "avocado", "banana", "blueberry", "cherry"]
by_letter: defaultdict[str, list[str]] = defaultdict(list)
for word in words:
    by_letter[word[0]].append(word)

print(dict(by_letter))
```

```text
{'a': ['apple', 'avocado'], 'b': ['banana', 'blueberry'], 'c': ['cherry']}
```

### Counter

```text
>>> from collections import Counter
>>> c = Counter("mississippi")
>>> c
Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
>>> c["s"]
4
>>> c["z"]
0
>>> c.most_common(2)
[('i', 4), ('s', 4)]
>>> c.update("ss")
>>> c["s"]
6
>>> c.total()
13
>>> Counter(["a", "b", "a"]) + Counter(["a"])
Counter({'a': 3, 'b': 1})
>>> Counter(a=3, b=1) - Counter(a=1, b=5)
Counter({'a': 2})
>>> list(Counter("aab").elements())
['a', 'a', 'b']
```

A Counter built from a list of words counts the words, so the classic word-frequency program is two lines:

```python
from collections import Counter

text = "to be or not to be that is the question"
print(Counter(text.split()).most_common(3))
```

```text
[('to', 2), ('be', 2), ('or', 1)]
```

### deque

```text
>>> from collections import deque
>>> d = deque([1, 2, 3])
>>> d.append(4)
>>> d.appendleft(0)
>>> d
deque([0, 1, 2, 3, 4])
>>> d.pop()
4
>>> d.popleft()
0
>>> d.rotate(1)
>>> d
deque([3, 1, 2])
>>> d.extendleft([8, 9])
>>> d
deque([9, 8, 3, 1, 2])
```

Notice that `extendleft` adds the items one by one at the left end, so they end up in **reverse** order.

### deque as a bounded buffer

```text
>>> from collections import deque
>>> recent = deque(maxlen=3)
>>> for n in range(5):
...     recent.append(n)
...
>>> recent
deque([2, 3, 4], maxlen=3)
```

The buffer keeps only the three most recent items, silently discarding the oldest. This is ideal for "the last N log lines" or a moving average.

### deque as a queue

```python
from collections import deque

jobs = deque(["resize", "email", "report"])
jobs.append("backup")

while jobs:
    print("processing", jobs.popleft())
```

```text
processing resize
processing email
processing report
processing backup
```

First in, first out, with fast operations at both ends.

### namedtuple

```text
>>> from collections import namedtuple
>>> Point = namedtuple("Point", ["x", "y"])
>>> p = Point(3, 4)
>>> p
Point(x=3, y=4)
>>> p.x, p[1]
(3, 4)
>>> x, y = p
>>> p._replace(x=10)
Point(x=10, y=4)
>>> p._asdict()
{'x': 3, 'y': 4}
>>> Point._fields
('x', 'y')
>>> Point._make([1, 2])
Point(x=1, y=2)
>>> p.x = 5
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: can't set attribute
```

A `namedtuple` instance behaves like a tuple (it can be unpacked, compared, and used as a dictionary key) but cannot be changed.

### namedtuple with defaults

```text
>>> from collections import namedtuple
>>> Point3 = namedtuple("Point3", "x y z", defaults=[0])
>>> Point3(1, 2)
Point3(x=1, y=2, z=0)
```

The field names may be given as a list or as one space-separated string, and `defaults` fill in the **rightmost** fields.

### ChainMap

```text
>>> from collections import ChainMap
>>> defaults = {"color": "red", "user": "guest"}
>>> overrides = {"user": "ada"}
>>> config = ChainMap(overrides, defaults)
>>> config["user"], config["color"]
('ada', 'red')
>>> config["theme"] = "dark"
>>> overrides
{'user': 'ada', 'theme': 'dark'}
>>> defaults
{'color': 'red', 'user': 'guest'}
>>> sorted(config)
['color', 'theme', 'user']
>>> len(config.maps)
2
```

The write went to the first dictionary (`overrides`), and `defaults` was left alone.

### ChainMap scopes

```text
>>> from collections import ChainMap
>>> scope = ChainMap({"x": 1})
>>> inner = scope.new_child({"y": 2})
>>> inner["x"], inner["y"]
(1, 2)
>>> inner.parents["x"]
1
```

`new_child` adds a new empty (or provided) dictionary in front, which is how interpreters model nested variable scopes.

### OrderedDict

```text
>>> from collections import OrderedDict
>>> cache = OrderedDict(a=1, b=2, c=3)
>>> cache.move_to_end("a")
>>> list(cache)
['b', 'c', 'a']
>>> cache.popitem(last=False)
('b', 2)
```

## Deeper behavior

### defaultdict creates keys when you only read

The factory runs on `d[key]`, and **the key is then stored**. That is convenient for building but it means a typo silently creates an entry:

```text
>>> from collections import defaultdict
>>> counts = defaultdict(int)
>>> counts["typo"]
0
>>> "typo" in counts
True
>>> counts.get("other")
>>> "other" in counts
False
```

`get` and `in` do not trigger the factory. Convert to a plain `dict` with `dict(d)` when you finish building, so later reads of a missing key raise `KeyError` as usual.

### Nested defaultdicts

```python
from collections import defaultdict

visits: defaultdict[str, defaultdict[str, int]] = defaultdict(lambda: defaultdict(int))
visits["ada"]["/home"] += 1
visits["ada"]["/home"] += 1
visits["grace"]["/docs"] += 1
print({user: dict(pages) for user, pages in visits.items()})
```

```text
{'ada': {'/home': 2}, 'grace': {'/docs': 1}}
```

### Counter details

- A `Counter` is a dictionary subclass, so you can loop over it, sort it, and use every dictionary method.
- `c[item]` for a missing item returns `0` but, unlike `defaultdict`, does **not** insert the key.
- Counts can be zero or negative. The `+` and `-` operators drop non-positive results, while `update` and `subtract` keep them.
- `most_common()` with no argument returns everything, sorted by count. It uses a heap for small `n`, so it is efficient for "top few" questions.

### Why a list is a poor queue

`list.pop(0)` must shift every remaining item one place to the left, so its cost grows with the length of the list. `deque.popleft()` takes constant time. On one test machine, inserting at the front of a list of 40,000 items took about 7 microseconds, while `deque.appendleft` stayed at about 0.03 microseconds. Chapter 2.7 explains the notation behind such statements.

### deque trade-offs

A deque is excellent at both ends and **slow in the middle**: indexing `d[5000]` walks from the nearer end. Do not use it when you need random access. It is also designed for safe appends and pops from several threads, which is why it appears in producer-consumer code (Chapter 10.2).

### maxlen is a feature, not a limit

A deque with `maxlen` never raises when full. It just drops from the other side. If you need to know that something was dropped, check `len(d) == d.maxlen` before appending.

### namedtuple versus alternatives

| Need | Choice |
| --- | --- |
| A tiny, immutable, tuple-compatible record | `namedtuple` |
| The same with type hints and class syntax | `typing.NamedTuple` (Chapter 8.5) |
| A mutable record, defaults, methods, validation | A dataclass (Chapter 6.7) |
| A plain dictionary of data | `dict` or `TypedDict` (Chapter 8.5) |

A `namedtuple` uses less memory than a regular class instance and unpacks like a tuple, so it suits records from a database row or CSV file.

### ChainMap does not copy

The dictionaries you pass in remain the real storage. That has two consequences: changes to them appear in the `ChainMap` immediately, and a write through the `ChainMap` modifies the first dictionary you passed. If you do not want your inputs modified, pass copies, or put a fresh empty dictionary first.

### ChainMap lookups cost one step per map

A lookup tries each map in turn, so a long chain is slower than a flat dictionary. For a handful of layers this does not matter. If you read a merged configuration millions of times, build a flat dictionary once with `dict(chain_map)`.

### OrderedDict and ordinary dictionaries

Equality between two `OrderedDict` objects **does** consider order, unlike ordinary dictionaries. This is the main behavioral difference apart from the extra methods.

### Choosing the right container

| Problem | Container |
| --- | --- |
| Group items by a key | `defaultdict(list)` |
| Count occurrences | `Counter` |
| First in, first out | `deque` |
| Last N items | `deque(maxlen=N)` |
| Record with named fields | `namedtuple` or dataclass |
| Layered settings | `ChainMap` |
| Least recently used cache | `OrderedDict` (or `functools.lru_cache`) |

## Industry-level snippet

Three of these containers appear again and again in backend code. This module provides a **sliding-window rate limiter** (a `deque` of timestamps per client inside a `defaultdict`), a **status-code summary** using `Counter`, and **layered settings** built with `ChainMap`. The limiter takes its clock as a parameter, so tests can control time. (The `class` syntax is covered in Phase 6. For now, read `__init__` as "how to create one" and `allow` as "what you can ask it".)

```python
"""Operational helpers built on the collections module."""

import logging
import time
from collections import ChainMap, Counter, defaultdict, deque
from collections.abc import Callable, Iterable, Mapping

logger = logging.getLogger(__name__)


class SlidingWindowLimiter:
    """Allow at most max_requests per client within a rolling time window."""

    def __init__(
        self,
        max_requests: int,
        window_seconds: float,
        clock: Callable[[], float] = time.monotonic,
    ) -> None:
        if max_requests < 1:
            raise ValueError("max_requests must be at least 1")
        if window_seconds <= 0:
            raise ValueError("window_seconds must be greater than 0")
        self._max_requests = max_requests
        self._window = window_seconds
        self._clock = clock
        self._hits: defaultdict[str, deque[float]] = defaultdict(deque)

    def allow(self, client_id: str) -> bool:
        """Record a request and return True if it is within the limit."""
        now = self._clock()
        hits = self._hits[client_id]
        cutoff = now - self._window
        while hits and hits[0] <= cutoff:  # forget requests outside the window
            hits.popleft()
        if len(hits) >= self._max_requests:
            logger.info("Rate limit reached for client %s", client_id)
            return False
        hits.append(now)
        return True


def summarize_status_codes(codes: Iterable[int]) -> Counter[str]:
    """Count HTTP responses by class, such as "2xx" or "5xx"."""
    return Counter(f"{code // 100}xx" for code in codes)


def error_rate(summary: Counter[str]) -> float:
    """Return the fraction of responses that were client or server errors."""
    total = summary.total()
    if total == 0:
        return 0.0
    return (summary["4xx"] + summary["5xx"]) / total


def build_settings(
    defaults: Mapping[str, str],
    environment: Mapping[str, str],
    command_line: Mapping[str, str],
) -> ChainMap[str, str]:
    """Layer settings: command line beats environment, which beats defaults.

    The inputs are copied, so writing to the result never changes them.
    """
    return ChainMap({}, dict(command_line), dict(environment), dict(defaults))
```

Try it:

```python
now = [0.0]
limiter = SlidingWindowLimiter(max_requests=2, window_seconds=10, clock=lambda: now[0])
print([limiter.allow("ada") for _ in range(3)])
print(limiter.allow("grace"))
now[0] = 10.5
print(limiter.allow("ada"))

summary = summarize_status_codes([200, 200, 404, 500, 200, 301, 503])
print(summary.most_common())
print(f"{error_rate(summary):.1%}")

settings = build_settings(
    defaults={"port": "8000", "log_level": "INFO"},
    environment={"log_level": "DEBUG"},
    command_line={"port": "9000"},
)
print(settings["port"], settings["log_level"])
```

```text
[True, True, False]
True
True
[('2xx', 3), ('5xx', 2), ('4xx', 1), ('3xx', 1)]
42.9%
9000 DEBUG
```

Why it is written this way:

- Each client has its own deque of timestamps. Expired entries are removed from the **left** in constant time, which is exactly what a list cannot do cheaply.
- A `defaultdict(deque)` means the first request from a new client needs no special case.
- The clock is injected, so the demonstration (and any test) moves time by assigning a number instead of sleeping.
- `Counter` turns a log summary into one expression, and `total()` gives the denominator for the error rate.
- `ChainMap` expresses the precedence rule directly in the order of its arguments. The empty first dictionary absorbs any writes.
- A production limiter would also remove clients whose deques have become empty, so that memory does not grow with the number of distinct clients ever seen, and would need a lock if shared between threads.

## Common mistakes

1. **Using a plain dictionary and writing `if key not in d` everywhere** when `defaultdict` or `Counter` would do.
2. **Reading a `defaultdict` with a mistyped key,** which creates a new entry. Convert to `dict` when building is finished.
3. **Passing a call instead of a factory.** `defaultdict(list())` is wrong. Pass `list`, with no brackets.
4. **Using a list as a queue** with `pop(0)`. Use `deque.popleft()`.
5. **Indexing into the middle of a large deque** in a loop. It is slow. Use a list.
6. **Expecting `extendleft([1, 2, 3])` to preserve order.** It reverses it.
7. **Forgetting that `Counter` subtraction with `-` drops zero and negative counts.** Use `subtract` to keep them.
8. **Trying to change a `namedtuple` field.** Use `_replace` to create an updated copy, or choose a dataclass.
9. **Writing to a `ChainMap` and being surprised that the first dictionary changed.** Put a fresh `{}` first if you want to keep inputs intact.
10. **Using `OrderedDict` out of habit.** An ordinary dictionary keeps insertion order already.
11. **Expecting `Counter.most_common` to break ties in any particular way** other than first-seen order.

## Practice exercises

1. Count the words in a paragraph of your own with `Counter`, and print the five most common, ignoring case and punctuation.
2. Group a list of names by length using `defaultdict(list)`, then print each group in order of length.
3. Rewrite the character-counting program from Chapter 2.3 (which used `get`) three ways: with `defaultdict(int)`, with `Counter`, and with a plain dictionary and `setdefault`. Compare readability.
4. Use `deque(maxlen=5)` to keep the last five lines read from a file. Print them at the end, like the `tail` command.
5. Simulate a queue at a help desk: customers arrive (append) and are served (popleft). Print the order of service and the queue length after each event.
6. Use `rotate` to implement a function `rotate_left(items, k)` that works on a list by converting to a `deque`, and compare its code to the slicing version from Chapter 2.1.
7. Define a `namedtuple` called `Employee` with `name`, `department`, and `salary`. Create three employees, sort them by salary, and use `_replace` to give one a raise.
8. Compare two `Counter` objects built from two lists of purchases to find which items were bought more in the second list than in the first.
9. Build a configuration lookup with `ChainMap` that checks a `session` dictionary, then a `user` dictionary, then `defaults`. Show a write and a deletion and explain which dictionary each one affected.
10. Use `OrderedDict` and `move_to_end` to build a small least-recently-used cache with a maximum size of three. Show an item being evicted.
11. Measure the time to pop from the front of a list and of a deque, each holding 50,000 items, 10,000 times. Explain the difference.
12. Extend `SlidingWindowLimiter` with a method `cleanup()` that removes clients whose deques are empty after pruning, and a method `retry_after(client_id)` that returns how many seconds until the oldest request leaves the window. Write tests with a fake clock.

## Summary

- `defaultdict(factory)` creates missing values automatically, so grouping and counting need no existence checks. Pass the factory itself (`list`, `int`, `set`), not a call.
- `Counter` counts hashable items, returns `0` for missing ones, and offers `most_common`, `update`, `subtract`, `elements`, `total`, and arithmetic.
- `deque` supports constant-time operations at both ends, `rotate`, and bounded buffers with `maxlen`. It is slow in the middle.
- `namedtuple` makes immutable, tuple-compatible records with named fields, plus `_replace`, `_asdict`, `_fields`, and `_make`.
- `ChainMap` searches several dictionaries as one, writes only to the first, and does not copy its inputs. It models layered configuration and scopes.
- `OrderedDict` is mostly historical now, but its `move_to_end` is handy for caches.
- Choose the container that matches the problem, and your code shrinks and becomes clearer.
