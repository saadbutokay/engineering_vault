## Overview

Big-O describes algorithms in the abstract. A cost model tells you what each real operation costs in a particular language. In Python, one line can hide a loop, a copy, or a memory allocation, and two lines that look equally simple can differ by a factor of ten thousand. This note maps the common built-in operations to their real costs, so you can look at a piece of Python and estimate its complexity correctly.

Everything here refers to CPython, the standard interpreter you get from python.org. Other implementations such as PyPy behave differently in details, but the complexity of the main built-in operations is the same.

You will learn how CPython builds lists, dicts, sets, strings, and `deque`, the cost of every common built-in operation, the hidden costs that cause most accidental slowdowns, how large integers behave, how much memory Python objects use, why the recursion limit exists, and why built-in functions usually beat hand-written loops.

## Core concepts

### Everything is an object

Every value in Python is an object with a header that stores a type and a reference count. A list does not store numbers directly. It stores references (pointers) to number objects. That is why Python containers use more memory than the same data in C, and why a list can hold values of different types.

### Lists are over-allocated dynamic arrays

A `list` is an array of references stored in one contiguous block of memory. Because the references are contiguous, jumping to index `i` is one calculation, so indexing is `O(1)`. When the block is full, CPython allocates a larger block, copies the references over, and frees the old one. The new block has spare room (over-allocation), which makes `append` amortized `O(1)`.

- `values[i]`, `values[i] = x`: `O(1)`.
- `values.append(x)`: amortized `O(1)`.
- `values.pop()`: `O(1)` from the end.
- `values.pop(0)`, `values.insert(0, x)`, `del values[0]`: `O(n)`, because every other element shifts.
- `values.insert(i, x)`, `values.pop(i)`: `O(n - i)` shifts.
- `x in values`, `values.index(x)`, `values.count(x)`, `values.remove(x)`: `O(n)`.
- `values[a:b]`: `O(b - a)`, because a new list is built.
- `values + other`: `O(n + m)`; `values.extend(other)`: `O(m)`.
- `values.sort()`, `sorted(values)`: `O(n log n)` worst case, `O(n)` on already-sorted data.
- `len(values)`: `O(1)`.

### Dicts are compact, insertion-ordered hash tables

A `dict` turns each key into an integer with `hash(key)`, uses that integer to find a slot, and stores the key and value. Since Python 3.7, dicts keep insertion order. Lookups, inserts, and deletes are `O(1)` on average because collisions are rare when the table has room to spare, and the table is resized before it gets too full. In the worst case, when many keys collide, an operation can degrade to `O(n)`.

- `d[key]`, `d[key] = v`, `key in d`, `del d[key]`, `d.get(key)`: average `O(1)`.
- Iterating keys, values, or items: `O(n)`.
- `d.copy()`, `dict(d)`: `O(n)`.
- Keys must be hashable. Strings, numbers, and tuples of hashables are. Lists, dicts, and sets are not.

### Sets are hash tables without values

A `set` is a hash table that stores only keys.

- `x in s`, `s.add(x)`, `s.remove(x)`, `s.discard(x)`: average `O(1)`.
- `a.union(b)`: `O(len(a) + len(b))`.
- `a.intersection(b)`: about `O(min(len(a), len(b)))` on average.
- `a.difference(b)`: `O(len(a))`.
- Sets have no order. Never rely on the order of elements.

### Strings are immutable sequences

A `str` is an immutable sequence of Unicode characters. Immutable means you can never change one in place. Every "modification" builds a new string.

- `s[i]`: `O(1)`.
- `s[a:b]`: `O(b - a)`, a copy.
- `s + t`: `O(len(s) + len(t))`, a new string.
- `"".join(parts)`: `O(total length)`, one allocation.
- `t in s`, `s.find(t)`: treat as `O(len(s))` for typical input. The exact algorithm is covered in the string algorithms phase.
- `len(s)`: `O(1)`.

### deque is a queue with fast ends

`collections.deque` is a doubly linked list of fixed-size blocks. It is built for adding and removing at both ends.

- `append`, `appendleft`, `pop`, `popleft`: `O(1)`.
- Indexing near either end is fast; indexing the middle is `O(n)`.
- `deque(maxlen=k)` automatically discards from the opposite end when full.

### Complexity of the common operations at a glance

| Operation | List | Dict | Set | Deque |
| --- | --- | --- | --- | --- |
| Get by index or key | O(1) | O(1) average | not available | O(1) near ends |
| Add at end | O(1) amortized | O(1) average | O(1) average | O(1) |
| Add at front | O(n) | not applicable | not applicable | O(1) |
| Remove from end | O(1) | O(1) average | O(1) average | O(1) |
| Remove from front | O(n) | not applicable | not applicable | O(1) |
| Membership test | O(n) | O(1) average | O(1) average | O(n) |

### Hidden costs

These are the operations behind most accidental slowdowns.

- Slicing copies. `values[1:]` in a recursive function copies the list at every level.
- `x in some_list` scans the list. Inside a loop it makes the loop quadratic.
- `insert(0, x)` and `pop(0)` shift every element.
- Building a string with `+=` in a loop can create a new string each time.
- `list.index` and `list.count` scan the list.
- `values = values + [x]` builds a brand new list every time, unlike `values.append(x)`.
- `sorted(...)` inside a loop re-sorts everything on every iteration.

### Integers have arbitrary precision

Python integers never overflow. A value like `2 ** 10_000` is a normal `int` with thousands of bits. The cost is that arithmetic is no longer a single machine instruction. Adding two numbers with `d` digits costs `O(d)`, and multiplying them costs more than `O(d)` (large multiplications use Karatsuba, about `O(d^1.585)`). For ordinary values below 2^63 the cost is negligible, but for numbers with thousands of digits it matters. Converting a huge `int` to a decimal string is quadratic in the number of digits, and since Python 3.11 the interpreter refuses to convert integers longer than 4,300 digits by default.

### Object overhead and sys.getsizeof

`sys.getsizeof(obj)` returns the size of the object itself in bytes, not of the objects it refers to. For a list it counts the array of references, not the elements. Useful sizes on a 64-bit CPython 3.12 build:

- `0` and other small `int` values: 28 bytes each.
- An empty `str`: 41 bytes; each extra ASCII character adds 1 byte.
- An empty `tuple`: 40 bytes.
- An empty `list`: 56 bytes, and each element reference adds 8 bytes plus any over-allocation.
- An empty `dict`: 64 bytes.
- An empty `set`: 216 bytes.

These numbers differ slightly between Python versions and platforms. The shape of the numbers matters more than the exact values.

### The recursion limit

CPython caps the depth of nested Python function calls, 1,000 by default, so runaway recursion raises a clean `RecursionError` instead of crashing the interpreter. `sys.getrecursionlimit()` reads the limit and `sys.setrecursionlimit()` changes it. Raising it can turn a clean error into a hard crash when the C stack runs out, so prefer iteration or an explicit stack for deep problems.

## Basic syntax examples

### Lists, with the cost of each line

```python
numbers: list[int] = []
numbers.append(10)  # O(1) amortized
numbers.extend([20, 30])  # O(k) for k new items
last = numbers.pop()  # O(1)
numbers.insert(0, 5)  # O(n): shifts everything to the right
first = numbers.pop(0)  # O(n): shifts everything to the left
middle = numbers[1]  # O(1)
found = 20 in numbers  # O(n): scans the list
print(numbers, last, first, middle, found)
```

### Dicts and sets

```python
ages: dict[str, int] = {"Asha": 31, "Bilal": 27}
ages["Chen"] = 45  # O(1) average
print(ages.get("Dina", -1))  # O(1) average, returns -1 when missing
print("Asha" in ages)  # O(1) average

unique_ids: set[int] = set()
unique_ids.add(7)  # O(1) average
unique_ids.add(7)  # a duplicate is ignored
print(len(unique_ids))  # 1
```

### Strings and join

```python
parts: list[str] = []
for number in range(5):
    parts.append(str(number))  # O(1) amortized each
line = ",".join(parts)  # O(total length), a single allocation
print(line)  # 0,1,2,3,4
```

### deque as a queue

```python
from collections import deque

queue: deque[str] = deque(["a", "b", "c"])
queue.append("d")  # O(1)
first = queue.popleft()  # O(1), unlike list.pop(0)
print(first, list(queue))  # a ['b', 'c', 'd']

recent: deque[int] = deque(maxlen=3)
for value in range(6):
    recent.append(value)  # the oldest item drops out automatically
print(list(recent))  # [3, 4, 5]
```

### Watching a list over-allocate

```python
import sys

items: list[int] = []
last_size = sys.getsizeof(items)
print(f"len=0   size={last_size}")
for value in range(40):
    items.append(value)
    size = sys.getsizeof(items)
    if size != last_size:
        print(f"len={len(items):<3} size={size}")
        last_size = size
```

Output on CPython 3.12 (64-bit):

```text
len=0   size=56
len=1   size=88
len=5   size=120
len=9   size=184
len=17  size=248
len=25  size=312
len=33  size=376
```

The size grows only occasionally, in jumps, because each jump reserves spare room for the next several appends. Between jumps, appends cost nothing extra.

### Timing a hidden cost

```python
import timeit
from collections import deque


def drain_list(n: int) -> None:
    values = list(range(n))
    while values:
        values.pop(0)


def drain_deque(n: int) -> None:
    values = deque(range(n))
    while values:
        values.popleft()


size = 100_000
list_time = min(timeit.repeat(lambda: drain_list(size), number=1, repeat=3))
deque_time = min(timeit.repeat(lambda: drain_deque(size), number=1, repeat=3))
print(f"list.pop(0):   {list_time:.4f} s")
print(f"deque.popleft: {deque_time:.4f} s")
```

On a typical laptop `list.pop(0)` takes hundreds of milliseconds here and `popleft` takes a few milliseconds, a gap of two orders of magnitude that widens as `n` grows.

## Deeper behavior

### How a list grows

When `append` finds the list full, CPython asks for a new capacity of roughly `new_size + new_size / 8 + 6`, rounded down to a multiple of four. That is about 12.5 percent growth, which matches the capacities you can read off the `getsizeof` output above: 4, 8, 16, 24, 32, 40. Growing by a constant factor is what makes the amortized cost constant. `list.pop()` also shrinks the block occasionally, so memory is returned when a list shrinks a lot.

### How a dict stores data

A modern CPython dict has two arrays. A small sparse array of indices is addressed by hash, and a dense array of entries (hash, key, value) stores items in insertion order. That is why dicts are ordered and why they use less memory than older implementations. Deleting a key leaves a hole in the entries array that is reclaimed at the next resize. The table resizes before it is two-thirds full.

Hash values matter. `hash(int)` is the integer itself for small values, but `hash(str)` is randomized per process by default (controlled by `PYTHONHASHSEED`). This blocks attackers from crafting many keys that collide. Because of this randomization, `hash("abc")` gives different values on different runs, so it must never be used for anything that has to be stable across processes.

### Membership tests compare with ==

`x in some_list` calls `==` on each element until one matches, so its cost depends on how expensive each comparison is. For lists of strings or tuples, comparisons are not constant time. `x in some_set` computes `hash(x)` once and then compares with the few keys that share the slot. Strings cache their hash value, so hashing a string repeatedly is cheap after the first time.

### Slicing and recursion

Taking `values[1:]` creates a new list of length `n - 1`. A recursive function that slices at every call therefore does `n + (n - 1) + (n - 2) + ...` work, which is `O(n^2)`, even if the recursion itself looks linear. Pass index parameters instead.

```python
def sum_slices(values: list[int]) -> int:
    """O(n^2) total: every call copies the rest of the list."""
    if not values:
        return 0
    return values[0] + sum_slices(values[1:])


def sum_from(values: list[int], start: int = 0) -> int:
    """O(n) total: no copies, just an index."""
    if start == len(values):
        return 0
    return values[start] + sum_from(values, start + 1)
```

### String concatenation

Because strings are immutable, `result += piece` conceptually builds a brand new string of length `len(result) + len(piece)` every time, which makes a loop of `n` pieces `O(n^2)`. CPython contains an optimization that can extend a string in place when nothing else references it, so a simple loop often runs quickly on CPython. That optimization is an implementation detail. It does not exist on every Python implementation and it silently stops working when the string is referenced elsewhere. Rely on `"".join(parts)` instead, which is guaranteed linear.

### Why built-ins are faster than hand-written loops

A Python `for` loop executes bytecode, and each bytecode step costs tens of nanoseconds. Built-ins such as `sum`, `sorted`, `max`, `min`, `any`, `all`, `str.join`, and `list.sort` run their loops in C, with no per-item bytecode. A hand-written loop that does the same work is usually several times slower. The complexity is identical, but the constant factor is different. This is why experienced Python programmers reach for built-ins, comprehensions, and the standard library first.

```python
import timeit

data = list(range(100_000))


def loop_sum() -> int:
    total = 0
    for value in data:
        total += value
    return total


print("manual loop:", min(timeit.repeat(loop_sum, number=1, repeat=5)))
print("built-in sum:", min(timeit.repeat(lambda: sum(data), number=1, repeat=5)))
```

A typical result is that the built-in is about five times faster.

### Copies: shallow and deep

`values.copy()`, `values[:]`, and `list(values)` are shallow copies that cost `O(n)` and copy references, not the objects. `copy.deepcopy` recursively duplicates everything and costs time proportional to the total size of the whole object graph, and it is much slower per element. Copy only when you must, and know which one you are paying for.

### Sorting behavior

`list.sort()` and `sorted()` use Timsort, which is stable (equal items keep their relative order) and adaptive (nearly sorted input is much faster, down to `O(n)`). `sorted(values)` allocates a new list, so it uses `O(n)` extra space, while `values.sort()` works in place.

## Industry-level snippet

A product shows each user a "recent activity" panel with their latest actions. The service keeps the newest `limit` actions per user in memory, serves them newest first, and exports them as CSV text. The cost model drives three decisions:

- `deque(maxlen=limit)` evicts the oldest action in `O(1)`. A list with `pop(0)` would shift every element on every write.
- `latest` reads only `count` items from the end, so the work is proportional to what the caller asked for, not to the stored history.
- The CSV export builds the response with one `str.join` instead of repeated `+=`.

```python
# activity_feed.py
import logging
from collections import defaultdict, deque
from dataclasses import dataclass
from itertools import islice

logger = logging.getLogger(__name__)


class InvalidFeedSettingError(ValueError):
    """Raised when the feed is configured or queried with an invalid number."""


@dataclass(frozen=True, slots=True)
class Activity:
    user_id: int
    action: str
    timestamp: float


class RecentActivityFeed:
    """Keep the newest `limit` activities for every user, in memory."""

    def __init__(self, limit: int) -> None:
        if limit < 1:
            raise InvalidFeedSettingError(f"limit must be at least 1, got {limit}")
        self._limit = limit
        self._by_user: defaultdict[int, deque[Activity]] = defaultdict(self._new_feed)

    def _new_feed(self) -> deque[Activity]:
        return deque(maxlen=self._limit)

    def record(self, activity: Activity) -> None:
        """O(1): append to the user's deque, dropping the oldest item if full."""
        feed = self._by_user[activity.user_id]
        if len(feed) == self._limit:
            logger.debug("evicting oldest activity for user %s", activity.user_id)
        feed.append(activity)

    def latest(self, user_id: int, count: int) -> list[Activity]:
        """Return up to `count` activities, newest first, in O(count)."""
        if count < 0:
            raise InvalidFeedSettingError(f"count must not be negative, got {count}")
        feed = self._by_user.get(user_id)  # .get never creates an empty entry
        if feed is None:
            return []
        return list(islice(reversed(feed), count))

    def export_csv(self, user_id: int) -> str:
        """Build the response body with one join, in O(total text length)."""
        feed = self._by_user.get(user_id, ())
        return "\n".join(f"{item.timestamp:.3f},{item.action}" for item in feed)
```

```python
# test_activity_feed.py
import pytest

from activity_feed import Activity, InvalidFeedSettingError, RecentActivityFeed


def make(user_id: int, action: str, timestamp: float) -> Activity:
    return Activity(user_id=user_id, action=action, timestamp=timestamp)


def test_keeps_only_the_newest_items_newest_first() -> None:
    feed = RecentActivityFeed(limit=2)
    for i in range(3):
        feed.record(make(1, f"action-{i}", float(i)))
    actions = [item.action for item in feed.latest(1, 10)]
    assert actions == ["action-2", "action-1"]


def test_users_do_not_share_history() -> None:
    feed = RecentActivityFeed(limit=2)
    feed.record(make(1, "login", 1.0))
    feed.record(make(2, "logout", 2.0))
    assert [item.action for item in feed.latest(1, 5)] == ["login"]
    assert [item.action for item in feed.latest(2, 5)] == ["logout"]


def test_unknown_user_returns_empty_results() -> None:
    feed = RecentActivityFeed(limit=3)
    assert feed.latest(99, 5) == []
    assert feed.export_csv(99) == ""


def test_latest_respects_count() -> None:
    feed = RecentActivityFeed(limit=5)
    for i in range(5):
        feed.record(make(1, f"a{i}", float(i)))
    assert [item.action for item in feed.latest(1, 2)] == ["a4", "a3"]
    assert feed.latest(1, 0) == []


def test_export_csv_is_oldest_first() -> None:
    feed = RecentActivityFeed(limit=3)
    feed.record(make(1, "view", 1.5))
    feed.record(make(1, "buy", 2.25))
    assert feed.export_csv(1) == "1.500,view\n2.250,buy"


def test_invalid_limit_is_rejected() -> None:
    with pytest.raises(InvalidFeedSettingError):
        RecentActivityFeed(limit=0)


def test_negative_count_is_rejected() -> None:
    feed = RecentActivityFeed(limit=1)
    with pytest.raises(InvalidFeedSettingError):
        feed.latest(1, -1)
```

In a real service this class would sit behind a lock or live in a single async task, because a plain `deque` shared between threads needs coordination for compound operations. Thread-safe designs are covered in the concurrency phase.

## Common mistakes

- Using a list as a queue with `pop(0)` or `insert(0, x)`. Use `collections.deque`.
- Testing `x in some_list` inside a loop. Build a `set` once, outside the loop.
- Building a large string with `+=` in a loop. Collect the pieces in a list and call `"".join`.
- Slicing in recursive functions. Pass indexes instead of copying the remainder.
- Writing `values = values + [x]` instead of `values.append(x)`. The first copies the whole list every time.
- Assuming dict and set operations are always `O(1)`. They are `O(1)` on average and degrade when keys have bad hashes or are attacker-controlled.
- Using mutable objects as dict keys or set members. Lists and dicts are unhashable, and mutating an object after inserting it breaks lookups.
- Calling `sorted` or `max` inside a loop to repeatedly get the smallest or largest item. A heap does this efficiently.
- Treating `sys.getsizeof` as the deep size of a data structure. It does not include the objects that are referenced.
- Raising `sys.setrecursionlimit` to hide a deep recursion instead of converting it to a loop.
- Using the built-in `hash()` to assign users to stable buckets. String hashes change between runs.
- Optimizing by guessing. Measure with `timeit` before and after, because the bottleneck is often not where you expect.

## Practice exercises

1. For each line, state its time complexity in terms of `n = len(values)`.

   ```python
   values.append(1)
   values.insert(0, 1)
   values[n // 2]
   values[1:]
   42 in values
   values.pop()
   values.pop(0)
   sorted(values)
   ```

2. Rewrite this function so that it runs in `O(n)` instead of `O(n^2)` and say which cost-model fact you used.

   ```python
   def common_items(first: list[int], second: list[int]) -> list[int]:
       return [item for item in first if item in second]
   ```

3. Run the `getsizeof` loop on your machine. Record at which lengths the size jumps. Do the same for a `set` and a `dict` and compare the shapes of the output.
4. Time `drain_list` and `drain_deque` for `n = 10_000`, `20_000`, `40_000`, and `80_000`. Compute the ratio between consecutive timings and decide what complexity each one shows.
5. Write `join_slow(parts)` using `+=` and `join_fast(parts)` using `str.join`. Time both on 10,000 pieces, 100,000 pieces, and 1,000,000 pieces and describe how the gap changes.
6. Explain why `sum_slices` is `O(n^2)` while `sum_from` is `O(n)`. Then explain why both raise `RecursionError` on a list of 10,000 numbers, and rewrite one of them as a loop that has no such limit.
7. Create a dictionary whose keys are instances of a class with a `__hash__` that always returns `1`. Insert 5,000 keys and time lookups. Compare with the default hash and explain what you observe.
8. Measure `sys.getsizeof` for a list of 1,000 integers, then add up `sys.getsizeof` for the list and for each integer. Explain the difference between the two numbers and why small integers behave differently.
9. Compute `2 ** 100_000` and try `str(2 ** 100_000)`. What happens, and what does `sys.get_int_max_str_digits()` return?
10. Write a function that checks whether a list of 100,000 words contains any duplicate in three ways: sorting first, using a set, and using a nested loop. Estimate the complexity of each and time them.
11. A function must repeatedly remove the smallest item from a collection and add new items. Compare a list with `min` and `remove`, a sorted list with `insert`, and a heap using `heapq`. Which cost-model facts decide the winner?
12. Extend `RecentActivityFeed` with a method `forget_user(user_id)` that deletes a user's history, and write tests that confirm `latest` and `export_csv` return empty results afterward.

## Summary

CPython stores lists as over-allocated arrays of references, dicts and sets as hash tables, strings as immutable sequences, and deques as linked blocks. That layout explains the costs: list indexing and append are fast, but inserting or removing at the front and testing membership are linear; dict and set operations are fast on average; strings are rebuilt on every change; and a deque is the right queue. Hidden costs such as slicing, `in` on a list, and string concatenation create most accidental slowdowns. Integers have arbitrary precision, objects carry overhead that `sys.getsizeof` helps you see, and the recursion limit protects the interpreter. Built-in functions run their loops in C, so use them when you can and measure when you are unsure.
