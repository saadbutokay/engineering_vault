## Overview

Python's standard library already contains fast, well-tested versions of most structures and helpers you will need: queues, counters, heaps, binary search, combinatorics, caching, topological sorting, and number theory. Knowing what is already there lets you write less code, make fewer mistakes, and run faster, because these tools are implemented in C or heavily optimized.

This note is a guided tour. For each module you will see what it is for, the key functions, the cost of the main operations, and the traps to avoid. You will meet `collections`, `heapq`, `bisect`, `itertools`, `functools`, `graphlib`, `math`, `array`, `dataclasses`, generic classes with the PEP 695 syntax, `sortedcontainers` as the one popular third-party addition, and fast input with `sys.stdin`.

The rule for the rest of the roadmap: implement structures from scratch to learn how they work, then use the standard library in real code. In the capstone projects only `pytest` and Hypothesis are allowed, so `sortedcontainers` is reserved for ordinary practice.

## Core concepts

### collections

- `deque`: a double-ended queue. `append`, `appendleft`, `pop`, `popleft` are `O(1)`. Supports `maxlen` and `rotate`.
- `Counter`: a dict subclass for counting. Missing keys read as 0. `most_common(k)` returns the `k` largest counts.
- `defaultdict`: a dict that creates a default value for a missing key, such as `defaultdict(list)` for grouping.
- `OrderedDict`: a dict with extra order tools, `move_to_end` and `popitem(last=False)`, which make it convenient for least-recently-used caches. Plain dicts already remember insertion order, but they cannot move a key to the end cheaply.

### heapq

A min-heap stored in a plain list. `heappush` and `heappop` are `O(log n)`, `heapify` turns a list into a heap in `O(n)`, and `heap[0]` is always the smallest item. Helpers: `nlargest(k, items)` and `nsmallest(k, items)` in `O(n log k)`, `heappushpop`, `heapreplace`, and `merge` for combining sorted inputs.

### bisect

Binary search on a sorted list. `bisect_left(a, x)` returns the first position where `x` could be inserted, `bisect_right(a, x)` returns the position after any equal items, and `insort` inserts while keeping the list sorted. Searches are `O(log n)`, but `insort` is `O(n)` because later items shift. Since Python 3.10 all of them accept a `key=` function.

### itertools

Lazy iterator tools that build sequences without creating whole lists.

- `accumulate(values)`: running totals, or any running operation.
- `pairwise(values)`: consecutive pairs.
- `batched(values, n)`: tuples of up to `n` items (Python 3.12 and newer).
- `combinations(values, k)`: all subsets of size `k`, order ignored.
- `permutations(values, k)`: all orderings of `k` items.
- `product(a, b)` or `product(a, repeat=k)`: Cartesian product.
- `chain`, `islice`, `groupby`, `zip_longest`, `count`, `cycle`, and `repeat` for assembling and slicing streams.

### functools

- `cache` and `lru_cache(maxsize=...)`: memoize a function so repeated calls with the same arguments reuse the result.
- `cmp_to_key`: converts an old-style comparison function into a `key=` function for sorting.
- `reduce`, `partial`, and `total_ordering` for functional helpers.

### graphlib

`graphlib.TopologicalSorter` orders nodes of a dependency graph so that every node appears after the things it depends on, and raises `graphlib.CycleError` if the dependencies contain a cycle.

### math

`gcd`, `lcm`, `comb`, `perm`, `factorial`, `isqrt`, `prod`, `log2`, `ceil`, `floor`, and the constant `inf`, which is a convenient starting value for a minimum search.

### array

`array.array` stores one machine type compactly, for example `array("i")` for 4-byte signed integers. It uses far less memory than a list of `int` objects and is useful for large numeric data without installing NumPy.

### dataclasses

`@dataclass` generates `__init__`, `__repr__`, and `__eq__` from annotated fields. Options that matter for DSA are `frozen=True` (immutable and hashable), `slots=True` (less memory), and `order=True` (adds comparisons by field order).

### Generic classes with PEP 695 syntax

Python 3.12 introduced a compact syntax for generic classes, functions, and type aliases, so structures such as stacks and queues can be typed once for any item type.

### sortedcontainers

`sortedcontainers` is a third-party library (`pip install sortedcontainers`) with `SortedList`, `SortedDict`, and `SortedSet`. They keep items in sorted order with `O(log n)`-like operations in practice and fill the gap left by the missing balanced tree in the standard library.

### Fast input with sys.stdin

When a program reads large input from standard input, `sys.stdin.readline` and `sys.stdin.read` are much faster than repeated `input()` calls. This is common in contests, and rare in backend services, which parse requests with proper libraries.

## Basic syntax examples

### collections

```python
from collections import Counter, OrderedDict, defaultdict, deque

queue: deque[int] = deque([1, 2, 3])
queue.append(4)
queue.appendleft(0)
print(queue.popleft(), queue.pop())  # 0 4

counts = Counter("mississippi")
print(counts.most_common(2))  # [('i', 4), ('s', 4)]
print(counts["z"])  # 0, a missing key reads as zero and is not created

groups: defaultdict[int, list[str]] = defaultdict(list)
for word in ["tree", "pear", "kiwi", "fig"]:
    groups[len(word)].append(word)
print(dict(groups))  # {4: ['tree', 'pear', 'kiwi'], 3: ['fig']}

recent: OrderedDict[str, int] = OrderedDict(a=1, b=2, c=3)
recent.move_to_end("a")
recent.popitem(last=False)  # removes the oldest item, "b"
print(list(recent))  # ['c', 'a']
```

### heapq

```python
import heapq

heap: list[int] = []
for value in [5, 1, 8, 3]:
    heapq.heappush(heap, value)
print(heap[0])  # 1, the smallest item is always first
print(heapq.heappop(heap))  # 1

numbers = [5, 1, 8, 3, 9, 2]
heapq.heapify(numbers)  # O(n), in place
print(heapq.nlargest(2, numbers))  # [9, 8]
print(heapq.nsmallest(2, numbers))  # [1, 2]
```

### bisect

```python
from bisect import bisect_left, bisect_right, insort

scores = [10, 20, 20, 30]
print(bisect_left(scores, 20))  # 1, the first position that could hold 20
print(bisect_right(scores, 20))  # 3, the position after the last 20
insort(scores, 25)
print(scores)  # [10, 20, 20, 25, 30]
```

### itertools

```python
from itertools import accumulate, batched, combinations, pairwise, permutations, product

print(list(accumulate([1, 2, 3, 4])))  # [1, 3, 6, 10]
print(list(pairwise("abcd")))  # [('a', 'b'), ('b', 'c'), ('c', 'd')]
print(list(batched(range(7), 3)))  # [(0, 1, 2), (3, 4, 5), (6,)]
print(list(combinations([1, 2, 3], 2)))  # [(1, 2), (1, 3), (2, 3)]
print(len(list(permutations([1, 2, 3]))))  # 6
print(list(product("ab", repeat=2)))  # [('a', 'a'), ('a', 'b'), ('b', 'a'), ('b', 'b')]
```

### functools

```python
from functools import cache, cmp_to_key


@cache
def fibonacci(n: int) -> int:
    return n if n < 2 else fibonacci(n - 1) + fibonacci(n - 2)


def by_length_then_alphabet(first: str, second: str) -> int:
    if len(first) != len(second):
        return len(first) - len(second)
    return (first > second) - (first < second)


print(fibonacci(80))  # 23416728348467685, instant instead of astronomically slow
words = ["pear", "fig", "apple", "kiwi"]
ordered = sorted(words, key=cmp_to_key(by_length_then_alphabet))
print(ordered)  # ['fig', 'kiwi', 'pear', 'apple']
```

### graphlib

```python
from graphlib import TopologicalSorter

dependencies: dict[str, set[str]] = {
    "app": {"db", "cache"},
    "db": {"config"},
    "cache": {"config"},
    "config": set(),
}
order = list(TopologicalSorter(dependencies).static_order())
print(order)  # ['config', 'db', 'cache', 'app']: config first, app last
```

### math

```python
import math

print(math.gcd(12, 18), math.lcm(4, 6))  # 6 12
print(math.comb(5, 2), math.perm(5, 2))  # 10 20
print(math.isqrt(17))  # 4
shortest = math.inf
for length in [7, 3, 9]:
    shortest = min(shortest, length)
print(shortest)  # 3
```

### array and dataclasses

```python
from array import array
from dataclasses import dataclass, field

compact = array("i", [1, 2, 3])
compact.append(4)
print(compact, compact.itemsize)  # array('i', [1, 2, 3, 4]) 4


@dataclass(order=True, frozen=True, slots=True)
class Task:
    priority: int
    name: str = field(compare=False)


print(Task(1, "email") < Task(2, "backup"))  # True, compared by priority only
```

### Generic classes with PEP 695 syntax

```python
type Edge = tuple[int, int, float]  # a type alias for readability


class Stack[T]:
    """A last-in, first-out stack of any item type."""

    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()

    def __len__(self) -> int:
        return len(self._items)


def first[T](items: list[T]) -> T:
    return items[0]


numbers = Stack[int]()
numbers.push(1)
print(numbers.pop())  # 1
```

### sortedcontainers

```python
from sortedcontainers import SortedList

latencies = SortedList([30, 10, 20])
latencies.add(15)
print(list(latencies))  # [10, 15, 20, 30]
print(latencies.bisect_left(20))  # 2
print(latencies.pop(0))  # 10, removes the smallest
```

This is the only third-party import in this note. If your type checker complains that it cannot find types for it, add an override to `pyproject.toml`:

```toml
[[tool.mypy.overrides]]
module = "sortedcontainers"
ignore_missing_imports = true
```

### Fast input

```python
import sys


def read_ints() -> list[int]:
    """Read every whitespace-separated integer from standard input at once."""
    return [int(token) for token in sys.stdin.read().split()]


read_line = sys.stdin.readline  # one line at a time, including the newline
```

## Deeper behavior

### heapq is a min-heap and a plain list

The heap is an ordinary list arranged so that each item is no larger than its two children. Only `heap[0]` is guaranteed to be the smallest. The rest of the list is not sorted, so do not print or index it expecting order. For a max-heap, push negated values: `heappush(heap, -value)` and negate again on the way out.

Heaps compare their items. For tasks with priorities, push tuples of `(priority, tie_breaker, item)`. If two priorities are equal, Python compares the next tuple element, and comparing arbitrary objects raises `TypeError`. A counter makes every tuple unique:

```python
import heapq
from itertools import count

tie_breaker = count()
heap: list[tuple[int, int, str]] = []
for priority, name in [(2, "backup"), (1, "email"), (2, "report")]:
    heapq.heappush(heap, (priority, next(tie_breaker), name))

while heap:
    priority, _, name = heapq.heappop(heap)
    print(priority, name)  # 1 email, 2 backup, 2 report (first in, first out for ties)
```

`nlargest(k, items)` costs `O(n log k)`, so it beats a full sort when `k` is much smaller than `n`. When `k` is close to `n`, `sorted` is faster.

### bisect needs sorted data and has a costly insort

`bisect` assumes the list is sorted and gives meaningless answers if it is not. `insort` finds the position in `O(log n)` but inserting into a list shifts later items, which costs `O(n)`. If you insert constantly into a large sorted collection, a heap or `sortedcontainers` is a better fit. The `key=` parameter transforms the items in the list, not the value you search for: `bisect_left(records, 20, key=lambda r: r.age)`.

### Counter and defaultdict surprises

- Reading a missing key from a `Counter` returns 0 without storing it. Reading a missing key from a `defaultdict` creates it. Use `.get(key)` or `in` when you do not want to create entries.
- `Counter` arithmetic drops zero and negative results. `Counter(a=3) - Counter(a=5)` is empty.
- `Counter.update(iterable)` adds counts, while `dict.update` replaces values.
- `most_common(k)` is implemented with a heap when `k` is given, so it costs `O(n log k)`.

### functools.cache pitfalls

- All arguments must be hashable. A list argument raises `TypeError`; convert it to a tuple.
- `cache` never forgets. It is unbounded and can consume all memory in a long-running process. Use `lru_cache(maxsize=1024)` for services.
- Caching an instance method keeps `self` alive for as long as the cache holds it.
- Recursion still counts: `fibonacci(5000)` raises `RecursionError` even with the cache, because the first call still recurses 5,000 levels deep.
- `fibonacci.cache_info()` and `fibonacci.cache_clear()` help you measure and reset the cache.

### itertools returns iterators

Most `itertools` functions return one-shot iterators. You can loop over one once, and it is then exhausted. Wrap with `list(...)` if you need to reuse it, but remember that `combinations` and `permutations` grow quickly: `math.comb(40, 20)` is about 138 billion. `groupby` only groups consecutive equal keys, so sort first if you want full groups.

### dataclass details

- `frozen=True` makes instances immutable and, together with `eq=True`, hashable, so they can be dict keys.
- `order=True` compares fields in declaration order. `field(compare=False)` excludes a field from comparisons.
- `slots=True` removes the per-instance dict and reduces memory for objects that you create in large numbers.
- Mutable defaults such as lists need `field(default_factory=list)`.

### OrderedDict versus dict

Plain `dict` keeps insertion order, and for most code that is enough. `OrderedDict` adds `move_to_end` and `popitem(last=False)` in `O(1)`, which is exactly what an LRU cache needs. Equality between two `OrderedDict` instances also takes order into account, unlike plain dicts.

### Standard library versus third party

Standard library modules are available everywhere, are stable, and need no installation, which is why capstone projects restrict you to them. `sortedcontainers` is a pure-Python library with excellent performance, widely used in production. Know both: implement structures yourself to understand them, and import the library to ship.

## Industry-level snippet

An API gateway writes one access-log line per request: `METHOD PATH STATUS DURATION_MS`. A nightly job must find the K slowest endpoints by average latency and save a snapshot that dashboards read. The toolbox does most of the work:

- `re` parses each line, and malformed lines are logged and skipped instead of crashing the job.
- `Counter` and `defaultdict` aggregate counts and totals in one pass.
- `heapq.nlargest` selects the top K in `O(n log k)` without sorting everything.
- `itertools.batched` splits inserts into batches, and a single `sqlite3` transaction replaces the snapshot atomically: readers see either the old snapshot or the new one, never half of each.

```python
# access_log_report.py
import heapq
import logging
import re
import sqlite3
from collections import Counter, defaultdict
from collections.abc import Iterable, Iterator, Sequence
from dataclasses import dataclass
from itertools import batched

logger = logging.getLogger(__name__)

LINE_PATTERN = re.compile(
    r"^(?P<method>[A-Z]+) (?P<path>\S+) (?P<status>\d{3}) (?P<ms>\d+)$"
)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS endpoint_stats (
    path TEXT PRIMARY KEY,
    requests INTEGER NOT NULL,
    total_ms INTEGER NOT NULL
)
"""

INSERT_SQL = "INSERT INTO endpoint_stats (path, requests, total_ms) VALUES (?, ?, ?)"


class LogParseError(ValueError):
    """Raised for a log line that does not match the expected format."""


@dataclass(frozen=True, slots=True)
class RequestRecord:
    method: str
    path: str
    status: int
    duration_ms: int


@dataclass(frozen=True, slots=True)
class EndpointStats:
    path: str
    requests: int
    total_ms: int

    @property
    def average_ms(self) -> float:
        return self.total_ms / self.requests


def parse_line(line: str) -> RequestRecord:
    match = LINE_PATTERN.match(line.strip())
    if match is None:
        raise LogParseError(f"malformed log line: {line!r}")
    return RequestRecord(
        method=match["method"],
        path=match["path"],
        status=int(match["status"]),
        duration_ms=int(match["ms"]),
    )


def parse_lines(lines: Iterable[str]) -> Iterator[RequestRecord]:
    """Yield valid records. Bad lines are logged and skipped, never fatal."""
    skipped = 0
    for line in lines:
        if not line.strip():
            continue
        try:
            yield parse_line(line)
        except LogParseError:
            skipped += 1
            logger.warning("skipping malformed line: %r", line)
    if skipped:
        logger.warning("skipped %d malformed lines in total", skipped)


def summarize(records: Iterable[RequestRecord]) -> list[EndpointStats]:
    """One pass over the records: O(n) time, O(distinct paths) space."""
    requests: Counter[str] = Counter()
    total_ms: defaultdict[str, int] = defaultdict(int)
    for record in records:
        requests[record.path] += 1
        total_ms[record.path] += record.duration_ms
    return [
        EndpointStats(path, count, total_ms[path]) for path, count in requests.items()
    ]


def slowest(stats: Iterable[EndpointStats], k: int) -> list[EndpointStats]:
    """Top K by average latency in O(n log k) time and O(k) extra space."""
    if k < 1:
        raise ValueError(f"k must be at least 1, got {k}")
    return heapq.nlargest(k, stats, key=lambda item: item.average_ms)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_stats(
    connection: sqlite3.Connection,
    stats: Sequence[EndpointStats],
    batch_size: int = 500,
) -> None:
    """Replace the stored snapshot atomically: old rows or new rows, never both."""
    if batch_size < 1:
        raise ValueError(f"batch_size must be at least 1, got {batch_size}")
    rows = ((item.path, item.requests, item.total_ms) for item in stats)
    try:
        with connection:
            connection.execute("DELETE FROM endpoint_stats")
            for batch in batched(rows, batch_size):
                connection.executemany(INSERT_SQL, batch)
    except sqlite3.Error:
        logger.exception("failed to save %d endpoint stats", len(stats))
        raise
    logger.info("saved %d endpoint stats", len(stats))
```

```python
# test_access_log_report.py
import sqlite3
from collections.abc import Iterator

import pytest

from access_log_report import (
    EndpointStats,
    LogParseError,
    create_schema,
    parse_line,
    parse_lines,
    save_stats,
    slowest,
    summarize,
)

LOG_LINES = [
    "GET /api/users 200 100",
    "GET /api/users 200 300",
    "POST /api/orders 201 500",
    "GET /health 200 5",
]


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def saved_rows(connection: sqlite3.Connection) -> list[tuple[str, int, int]]:
    cursor = connection.execute(
        "SELECT path, requests, total_ms FROM endpoint_stats ORDER BY path"
    )
    return [(str(p), int(r), int(t)) for p, r, t in cursor.fetchall()]


def test_parse_line_reads_every_field() -> None:
    record = parse_line("GET /api/users 200 120")
    assert (record.method, record.path, record.status, record.duration_ms) == (
        "GET",
        "/api/users",
        200,
        120,
    )


@pytest.mark.parametrize("line", ["", "garbage", "GET /x 20 5", "get /x 200 5"])
def test_parse_line_rejects_malformed_input(line: str) -> None:
    with pytest.raises(LogParseError):
        parse_line(line)


def test_parse_lines_skips_blank_and_bad_lines() -> None:
    lines = ["GET /a 200 1", "", "oops", "GET /b 200 2"]
    paths = [record.path for record in parse_lines(lines)]
    assert paths == ["/a", "/b"]


def test_summarize_and_slowest() -> None:
    stats = summarize(parse_lines(LOG_LINES))
    by_path = {item.path: item for item in stats}
    assert by_path["/api/users"].requests == 2
    assert by_path["/api/users"].average_ms == pytest.approx(200.0)
    top_two = [item.path for item in slowest(stats, 2)]
    assert top_two == ["/api/orders", "/api/users"]


def test_slowest_rejects_non_positive_k() -> None:
    with pytest.raises(ValueError):
        slowest([], 0)


def test_save_stats_replaces_the_previous_snapshot(
    connection: sqlite3.Connection,
) -> None:
    save_stats(connection, [EndpointStats("/old", 1, 10)])
    save_stats(connection, [EndpointStats("/a", 2, 30), EndpointStats("/b", 1, 5)])
    assert saved_rows(connection) == [("/a", 2, 30), ("/b", 1, 5)]


def test_failed_save_keeps_the_previous_snapshot(
    connection: sqlite3.Connection,
) -> None:
    save_stats(connection, [EndpointStats("/old", 1, 10)])
    duplicate_paths = [EndpointStats("/a", 1, 1), EndpointStats("/a", 2, 2)]
    with pytest.raises(sqlite3.IntegrityError):
        save_stats(connection, duplicate_paths)
    assert saved_rows(connection) == [("/old", 1, 10)]
```

Run the job against real input with `parse_lines(open("access.log"))`, which streams the file one line at a time. Memory use depends on the number of distinct endpoints, not on the size of the log.

## Common mistakes

- Using a list as a queue when `deque` exists.
- Writing a manual counting loop when `Counter` does it in one line.
- Treating `heap` as a sorted list. Only `heap[0]` is the smallest.
- Pushing objects without a tie-breaker into a heap, then getting `TypeError` when two priorities are equal.
- Calling `bisect` on an unsorted list and trusting the result.
- Expecting `heapq` to provide a max-heap. Negate the values or wrap the priority.
- Using `functools.cache` for an unbounded set of inputs in a long-running service. Use `lru_cache(maxsize=...)`.
- Forgetting that `itertools` results are one-shot iterators that run out after one pass.
- Forgetting that `groupby` requires sorted input to produce complete groups.
- Calling `combinations` or `permutations` on large inputs and running out of time or memory.
- Reading a missing `defaultdict` key to test existence. It creates the key.
- Using `input()` in a loop to read hundreds of thousands of lines when `sys.stdin.read()` is far faster.
- Importing a third-party package in a task that forbids it.

## Practice exercises

1. Use `Counter` to find the three most common words in a paragraph of text, ignoring case and punctuation.
2. Use `defaultdict(list)` to group a list of words into anagram groups, using the sorted letters of each word as the key.
3. Implement a queue of tasks using `deque`. Support `add`, `next`, and `size`, and write tests.
4. Use `heapq` to implement a function `k_smallest(values, k)` without calling `nsmallest`. Compare its speed against `nsmallest` and `sorted(values)[:k]` for several values of `k`.
5. Build a max-heap of `(priority, name)` pairs using negation. Show that two tasks with equal priority come out in a well-defined order thanks to a counter.
6. Using `bisect`, write `count_in_range(sorted_values, low, high)` that counts values with `low <= value < high`.
7. Use `bisect` with the `key=` parameter to search a list of `(age, name)` records sorted by age.
8. Write a generator `sliding_pairs(values)` using `pairwise`, and use `accumulate` to compute running totals and running maximums of a list.
9. Use `itertools.batched` to split a list of 1,000 user IDs into chunks of 100 and process each chunk. Handle a final chunk that is shorter.
10. Count how many distinct committees of 3 people can be chosen from 10 people with `math.comb`, then list them all with `itertools.combinations` and confirm that the counts match.
11. Use `functools.cache` to compute the number of ways to climb 40 stairs taking one or two steps at a time. Use `cache_info()` to report the hits and misses.
12. Use `graphlib.TopologicalSorter` to order the steps of a recipe where each step depends on others, and add a dependency that creates a cycle to see `CycleError`.
13. Define a generic `Queue[T]` class with PEP 695 syntax, backed by `deque`, with `enqueue`, `dequeue`, and `__len__`, and write tests.
14. Read 200,000 integers from standard input two ways: with repeated `input()` and with `sys.stdin.read().split()`. Time both.
15. Extend the access-log job so that `slowest` returns only endpoints with at least 10 requests, and add tests for the cutoff.

## Summary

The Python standard library provides production-quality building blocks for most DSA tasks. Use `deque` for queues, `Counter` and `defaultdict` for counting and grouping, `heapq` for priority queues and top-K, `bisect` for binary search on sorted lists, `itertools` for lazy combinatorics and windows, `functools.cache` for memoization, `graphlib` for dependency ordering, `math` for number theory and counting, `array` for compact numeric data, and `dataclasses` for lightweight records. PEP 695 syntax lets you write generic structures cleanly. Know the costs and traps: heaps are not sorted lists, `bisect` needs sorted data, caches can grow without limit, and iterators run out. `sortedcontainers` is the main third-party addition, and `sys.stdin` is the fast path for large inputs.
