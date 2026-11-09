## Overview

Binary search finds an item in a sorted sequence by repeatedly cutting the search range in half. Looking up a name in a printed phone book works the same way: open in the middle, see whether the name comes before or after, and discard the half that cannot contain it. A list of a million sorted items needs at most 20 steps, and a list of a billion needs at most 30.

The idea is simple and famously easy to get wrong. The classic bugs are off-by-one errors, infinite loops, and returning the wrong position when there are duplicates. Most of this note is about writing binary search so that it is right every time: choosing a clear invariant, deciding between a closed and a half-open range, and treating "find the first position where something becomes true" as the general problem.

Binary search is also much more than looking up values in a list. It can search any problem where an answer can be tested and the test flips from false to true exactly once: the smallest capacity that works, the first bad version, the square root of a number to ten decimal places. This note covers the classic form and its traps, lower and upper bound, rotated and bitonic arrays, binary search on the answer space, floating-point binary search, the `bisect` module with its `key` parameter, and searching sorted logs by timestamp.

## Core concepts

### The requirement

Binary search needs data that is sorted, or more generally a yes-or-no question whose answers are monotonic: false for every position up to some point and true from then on. Without that structure, discarding half the range is not justified.

### The classic form

Keep a range `[low, high]` that must contain the target if the target exists. Look at the middle item. If it equals the target, you are done. If it is smaller than the target, everything at or before the middle is too small, so continue in the right half. Otherwise continue in the left half. Stop when the range is empty.

- Time: `O(log n)` comparisons.
- Space: `O(1)` for the iterative version.

### Lower bound and upper bound

With duplicates, "find the target" is ambiguous. Two precise questions have clear answers.

- Lower bound: the first index whose value is greater than or equal to the target. Also the position where the target should be inserted to keep the list sorted, before any equal items.
- Upper bound: the first index whose value is strictly greater than the target. Inserting here places the new item after any equal items.

The number of occurrences of a value is `upper_bound - lower_bound`.

### The general form: first position where a condition is true

Most variants are one idea: find the smallest `x` in a range for which a predicate is true, given that the predicate is false, false, ..., false, true, true, ..., true. Lower bound uses the predicate `values[i] >= target`. Upper bound uses `values[i] > target`.

### Rotated sorted arrays

A sorted array rotated at some pivot, such as `[4, 5, 6, 7, 0, 1, 2]`, is not sorted overall, but each half you cut it into has at least one sorted side. You can tell which side by comparing the middle with an end, and then decide where the target can be.

### Bitonic arrays

A bitonic (mountain) array increases and then decreases. The peak is where the sequence stops increasing. Comparing the middle item with its right neighbor tells you whether you are on the rising or the falling slope, so binary search finds the peak.

### Binary search on the answer space

Sometimes the thing you search is not a list but a range of possible answers. If you can check quickly whether a candidate answer is large enough (feasible), and feasibility is monotonic (once feasible, every larger answer is feasible), binary search finds the smallest feasible answer. Examples: the least ship capacity that delivers all packages in `D` days, the smallest speed that finishes a task in time, the smallest `x` with `x * x > n`.

### Floating-point binary search

For real-valued answers there is no "last index". Instead you stop after a fixed number of iterations or when the range is narrower than a tolerance. Each iteration halves the range, so 50 to 100 iterations are enough to reach the limits of floating-point precision.

### The bisect module

Python's `bisect` module provides `bisect_left` (lower bound), `bisect_right` (upper bound), and `insort_left` and `insort_right`, which find the position and insert. Since Python 3.10 all of them take a `key=` function that is applied to the items of the list, not to the value you search for.

## Basic syntax examples

### Classic binary search

```python
from collections.abc import Sequence


def binary_search(values: Sequence[int], target: int) -> int:
    """Return an index of target in the sorted sequence, or -1. O(log n)."""
    low, high = 0, len(values) - 1
    while low <= high:
        # Invariant: if target is present, its index is in [low, high].
        middle = low + (high - low) // 2
        if values[middle] == target:
            return middle
        if values[middle] < target:
            low = middle + 1
        else:
            high = middle - 1
    return -1


print(binary_search([2, 5, 8, 12, 16, 23, 38], 23))  # 5
print(binary_search([2, 5, 8, 12, 16, 23, 38], 7))  # -1
```

Trace of `binary_search([2, 5, 8, 12, 16, 23, 38], 23)`:

```text
low=0, high=6, middle=3 (12): 12 < 23, move low to 4
low=4, high=6, middle=5 (23): found, return 5
```

### The first-true template

This is the form worth memorizing. The range is half-open, `[low, high)`, and the answer is always in `[low, high]`, where `high` means "none".

```python
from collections.abc import Callable


def first_true(low: int, high: int, predicate: Callable[[int], bool]) -> int:
    """Smallest x in [low, high) with predicate(x) True, or high if there is none.

    Requires the predicate to be False for small x and True from some point on.
    """
    while low < high:
        middle = low + (high - low) // 2
        if predicate(middle):
            high = middle  # the answer is middle or something before it
        else:
            low = middle + 1  # middle is False, so the answer is after it
    return low
```

### Lower and upper bound

```python
def lower_bound(values: Sequence[int], target: int) -> int:
    """First index with values[i] >= target."""
    return first_true(0, len(values), lambda i: values[i] >= target)


def upper_bound(values: Sequence[int], target: int) -> int:
    """First index with values[i] > target."""
    return first_true(0, len(values), lambda i: values[i] > target)


def count_occurrences(values: Sequence[int], target: int) -> int:
    return upper_bound(values, target) - lower_bound(values, target)


data = [1, 2, 2, 2, 3, 5]
print(lower_bound(data, 2), upper_bound(data, 2))  # 1 4
print(count_occurrences(data, 2))  # 3
print(lower_bound(data, 4))  # 5, where 4 would be inserted
print(lower_bound(data, 9))  # 6, past the end
```

### Searching a rotated sorted array

```python
def search_rotated(values: Sequence[int], target: int) -> int:
    """Distinct values, sorted then rotated. Returns the index or -1. O(log n)."""
    low, high = 0, len(values) - 1
    while low <= high:
        middle = low + (high - low) // 2
        if values[middle] == target:
            return middle
        if values[low] <= values[middle]:  # the left half is sorted
            if values[low] <= target < values[middle]:
                high = middle - 1
            else:
                low = middle + 1
        else:  # the right half is sorted
            if values[middle] < target <= values[high]:
                low = middle + 1
            else:
                high = middle - 1
    return -1


def find_min_rotated(values: Sequence[int]) -> int:
    """Index of the smallest item in a rotated sorted array of distinct values."""
    low, high = 0, len(values) - 1
    while low < high:
        middle = low + (high - low) // 2
        if values[middle] > values[high]:
            low = middle + 1  # the minimum is to the right of the middle
        else:
            high = middle  # the middle might be the minimum
    return low


rotated = [4, 5, 6, 7, 0, 1, 2]
print(search_rotated(rotated, 0), search_rotated(rotated, 3))  # 4 -1
print(find_min_rotated(rotated))  # 4
```

### Finding a peak in a mountain array

```python
def find_peak(values: Sequence[int]) -> int:
    """Index of the peak of an array that strictly rises and then strictly falls."""
    low, high = 0, len(values) - 1
    while low < high:
        middle = low + (high - low) // 2
        if values[middle] < values[middle + 1]:
            low = middle + 1  # still rising, the peak is to the right
        else:
            high = middle  # falling or at the peak, the peak is here or to the left
    return low


print(find_peak([1, 3, 8, 12, 9, 4, 2]))  # 3
```

### Binary search on the answer: ship capacity

Packages must be shipped in the given order. Find the smallest ship capacity that delivers all of them within `days` days.

```python
def can_ship(weights: Sequence[int], days: int, capacity: int) -> bool:
    """Greedy check: can everything ship within `days` days with this capacity?"""
    needed_days = 1
    load = 0
    for weight in weights:
        if load + weight > capacity:
            needed_days += 1
            load = 0
        load += weight
    return needed_days <= days


def min_ship_capacity(weights: Sequence[int], days: int) -> int:
    """Smallest feasible capacity. The search range is [max weight, total weight]."""
    low = max(weights)  # a smaller ship could not carry the heaviest package
    high = sum(weights) + 1  # exclusive: sum(weights) is always feasible
    return low + first_true(
        0, high - low, lambda offset: can_ship(weights, days, low + offset)
    )


print(min_ship_capacity([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 5))  # 15
```

### Integer square root

```python
def integer_sqrt(n: int) -> int:
    """Largest x with x * x <= n, found as (first x with x * x > n) - 1."""
    if n < 0:
        raise ValueError("n must not be negative")
    return first_true(0, n + 2, lambda x: x * x > n) - 1


print(integer_sqrt(50), integer_sqrt(49), integer_sqrt(0))  # 7 7 0
```

### Floating-point binary search

```python
def float_sqrt(value: float) -> float:
    """Square root of a non-negative number by bisection on the real line."""
    if value < 0:
        raise ValueError("value must not be negative")
    low, high = 0.0, max(1.0, value)  # for value < 1 the root is larger than value
    for _ in range(100):  # each iteration halves the range
        middle = (low + high) / 2
        if middle * middle < value:
            low = middle
        else:
            high = middle
    return (low + high) / 2


print(round(float_sqrt(2), 9))  # 1.414213562
```

### The bisect module

```python
from bisect import bisect_left, bisect_right, insort

scores = [10, 20, 20, 30]
print(bisect_left(scores, 20), bisect_right(scores, 20))  # 1 3
insort(scores, 25)
print(scores)  # [10, 20, 20, 25, 30]

# With key=, the function is applied to the items of the list, not to the target.
log = [(100, "start"), (150, "login"), (150, "click"), (210, "logout")]
first_at_150 = bisect_left(log, 150, key=lambda entry: entry[0])
after_150 = bisect_right(log, 150, key=lambda entry: entry[0])
print(first_at_150, after_150)  # 1 3
print(log[first_at_150:after_150])  # [(150, 'login'), (150, 'click')]
```

## Deeper behavior

### Choose one convention and one invariant

Almost every binary search bug comes from mixing conventions. There are two clean ones.

Closed range `[low, high]`, loop `while low <= high`:

- Both ends are candidates.
- After testing `middle`, move to `middle + 1` or `middle - 1`, because `middle` itself has been ruled out.
- The loop ends when the range is empty (`low > high`).

Half-open range `[low, high)`, loop `while low < high`:

- `high` is one past the last candidate, so it can represent "not found" or "past the end".
- Move to `middle + 1` or set `high = middle`.
- The loop ends when `low == high`, and that single remaining position is the answer.

Pick one per function and write the invariant as a comment: "if the target exists, it is in `[low, high]`", or "the answer is in `[low, high]` and everything before `low` fails the predicate".

### The classic traps

- Infinite loop: setting `low = middle` (instead of `middle + 1`) in a loop with `low < high` and a floor-division middle. When `high == low + 1`, `middle == low`, nothing changes, and the loop never ends. If you need `low = middle`, compute the middle with a round-up (`(low + high + 1) // 2`).
- Off-by-one on the loop condition: `while low < high` with a closed range misses the case where `low == high` and the last candidate is never examined.
- Wrong upper limit: starting `high` at `len(values)` with a closed-range loop reads past the end.
- Overflow: `(low + high) // 2` can overflow in languages with fixed-size integers. Python integers do not overflow, but writing `low + (high - low) // 2` is a habit that carries over safely to other languages.

### Duplicates

Classic binary search returns any matching index, which may be neither the first nor the last. If you need a specific one, use lower bound for the first occurrence, and `upper_bound - 1` for the last. To check that a value exists with lower bound, verify `index < len(values) and values[index] == target`.

### Why the first-true template covers so much

Every "find the boundary" problem reduces to a monotonic predicate. For the insertion point, the predicate is `values[i] >= target`. For "first bad version", it is `is_bad(i)`. For the ship capacity, it is `can_ship(capacity)`. Once you write the predicate, the template finds the boundary and you never touch the indexes again, which is why bugs disappear.

### Binary search on the answer: setting the bounds

The search range must contain the answer, and the answer must be feasible at the high end. For ship capacity, the lowest possible capacity is the heaviest package (a smaller ship can never carry it), and the highest useful capacity is the total weight (one day is always enough). Choosing too narrow a range misses the answer. Choosing too wide a range costs only a few extra iterations, since the number of steps grows with the logarithm of the range.

For feasibility checks that are themselves `O(n)`, the total cost is `O(n log range)`.

### Rotated arrays with duplicates

When values can repeat, the comparison `values[low] <= values[middle]` no longer proves that the left half is sorted. A list like `[1, 1, 1, 0, 1]` can hide the rotation point behind equal values. The usual fix shrinks the range by one when the ends are equal, which makes the worst case `O(n)`. The `O(log n)` guarantee holds only for distinct values.

### Floating-point search

There is no integer to land on, so compare with a fixed iteration count rather than testing `low == high`, which may never become true or may take thousands of iterations. After about 52 halvings of a range near 1, you reach the spacing between adjacent floating-point numbers, so further iterations change nothing. When you compare a computed value with a target, allow for rounding with `math.isclose` or an explicit tolerance.

### Complexity, precisely

Binary search on `n` items takes at most `floor(log2(n)) + 1` comparisons. For `n = 1,000,000` that is 20. That is why a sorted list plus binary search is so effective for lookups that happen many times: sort once for `O(n log n)`, then search each time for `O(log n)`.

### bisect details

- `bisect_left(a, x)` and `bisect_right(a, x)` return insertion points. They never fail and never return `-1`.
- `insort` finds the position in `O(log n)` but inserting into a list shifts later items, so the call costs `O(n)`.
- With `key=`, the list must be sorted by that same key, and `x` is compared directly with the key values. Do not pass `key(x)`.
- The module works on any sequence that supports `len` and indexing, so it also works on `range` objects and on `array.array`.

### Testing binary search

Compare against the obvious linear implementation on thousands of random sorted lists, including empty lists, one-item lists, all-equal lists, and targets below the minimum, above the maximum, and between items. Binary search bugs are almost always edge cases.

## Industry-level snippet

A service stores application events in an append-only log. Timestamps never decrease, so the log is sorted by `(timestamp, id)`. Clients call two kinds of endpoints: "give me everything between 10:00 and 10:05", and "give me the next 100 events after the one I last saw" for paging. Scanning the whole log for each request would be `O(n)` every time. Binary search makes both operations `O(log n)` to locate the start, plus the cost of returning the result.

Paging uses a cursor made of the last event's `(timestamp, id)`, not an offset. Offsets shift when new events arrive and cause duplicates or gaps, while a cursor points at an exact place in the order. `bisect_right` with a `key` finds the first event strictly after the cursor.

Appends are validated: a timestamp earlier than the last one would break the sort order that binary search depends on, so it is rejected before anything is written. The row is inserted in a transaction first, and the in-memory list is updated only after the commit succeeds, so memory and database cannot disagree after a failure.

```python
# event_log.py
import logging
import sqlite3
from bisect import bisect_left, bisect_right
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS log_entry (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    message TEXT NOT NULL
)
"""

type Cursor = tuple[int, int]


class OutOfOrderError(ValueError):
    """Raised when an entry would break the timestamp order of the log."""


@dataclass(frozen=True, slots=True)
class LogEntry:
    entry_id: int
    timestamp: int
    message: str


def _timestamp(entry: LogEntry) -> int:
    return entry.timestamp


def _position(entry: LogEntry) -> Cursor:
    return (entry.timestamp, entry.entry_id)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


class EventLog:
    """An in-memory index over the log table. Assumes a single writer."""

    def __init__(self, connection: sqlite3.Connection) -> None:
        self._connection = connection
        rows = connection.execute(
            "SELECT id, timestamp, message FROM log_entry ORDER BY timestamp, id"
        )
        self._entries = [LogEntry(row[0], row[1], row[2]) for row in rows]

    def __len__(self) -> int:
        return len(self._entries)

    def append(self, timestamp: int, message: str) -> LogEntry:
        if self._entries and timestamp < self._entries[-1].timestamp:
            raise OutOfOrderError(
                f"timestamp {timestamp} is older than {self._entries[-1].timestamp}"
            )
        try:
            with self._connection:
                cursor = self._connection.execute(
                    "INSERT INTO log_entry (timestamp, message) VALUES (?, ?)",
                    (timestamp, message),
                )
        except sqlite3.Error:
            logger.exception("could not append a log entry at %d", timestamp)
            raise
        if cursor.lastrowid is None:
            raise RuntimeError("database did not return a row id")
        entry = LogEntry(cursor.lastrowid, timestamp, message)
        self._entries.append(entry)  # only after the commit succeeded
        return entry

    def between(self, start: int, end: int) -> list[LogEntry]:
        """Entries with start <= timestamp < end, in O(log n + result size)."""
        if start > end:
            raise ValueError("start must not be after end")
        first = bisect_left(self._entries, start, key=_timestamp)
        last = bisect_left(self._entries, end, key=_timestamp)
        return self._entries[first:last]

    def first_at_or_after(self, timestamp: int) -> LogEntry | None:
        index = bisect_left(self._entries, timestamp, key=_timestamp)
        return self._entries[index] if index < len(self._entries) else None

    def page_after(
        self, cursor: Cursor | None, limit: int
    ) -> tuple[list[LogEntry], Cursor | None]:
        """Keyset pagination: the page after `cursor` plus the next cursor."""
        if limit < 1:
            raise ValueError("limit must be at least 1")
        start = 0
        if cursor is not None:
            start = bisect_right(self._entries, cursor, key=_position)
        page = self._entries[start : start + limit]
        next_cursor = _position(page[-1]) if page else cursor
        return page, next_cursor
```

```python
# test_event_log.py
import sqlite3
from collections.abc import Iterator

import pytest

from event_log import (
    Cursor,
    EventLog,
    LogEntry,
    OutOfOrderError,
    create_schema,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


@pytest.fixture
def log(connection: sqlite3.Connection) -> EventLog:
    event_log = EventLog(connection)
    for timestamp in (100, 150, 150, 200, 300):
        event_log.append(timestamp, f"event at {timestamp}")
    return event_log


def timestamps(entries: list[LogEntry]) -> list[int]:
    return [entry.timestamp for entry in entries]


def test_between_includes_start_and_excludes_end(log: EventLog) -> None:
    assert timestamps(log.between(150, 300)) == [150, 150, 200]
    assert timestamps(log.between(0, 100)) == []
    assert timestamps(log.between(0, 1_000)) == [100, 150, 150, 200, 300]
    assert timestamps(log.between(151, 199)) == []


def test_between_rejects_reversed_ranges(log: EventLog) -> None:
    with pytest.raises(ValueError):
        log.between(200, 100)


def test_first_at_or_after(log: EventLog) -> None:
    first = log.first_at_or_after(151)
    assert first is not None
    assert first.timestamp == 200
    assert log.first_at_or_after(301) is None


def test_pagination_visits_every_entry_once_in_order(log: EventLog) -> None:
    seen: list[int] = []
    cursor: Cursor | None = None
    while True:
        page, cursor = log.page_after(cursor, limit=2)
        if not page:
            break
        seen.extend(entry.entry_id for entry in page)
    assert seen == sorted(seen)
    assert len(seen) == 5
    assert len(set(seen)) == 5


def test_cursor_stays_valid_when_new_entries_arrive(log: EventLog) -> None:
    first_page, cursor = log.page_after(None, limit=3)
    log.append(400, "late arrival")
    second_page, _ = log.page_after(cursor, limit=10)
    assert [entry.timestamp for entry in second_page] == [200, 300, 400]
    assert len(first_page) == 3


def test_out_of_order_append_is_rejected_and_not_stored(
    log: EventLog, connection: sqlite3.Connection
) -> None:
    with pytest.raises(OutOfOrderError):
        log.append(50, "too old")
    count = connection.execute("SELECT COUNT(*) FROM log_entry").fetchone()[0]
    assert count == 5
    assert len(log) == 5


def test_log_reloads_from_the_database(
    log: EventLog, connection: sqlite3.Connection
) -> None:
    reloaded = EventLog(connection)
    assert timestamps(reloaded.between(0, 1_000)) == timestamps(log.between(0, 1_000))


def test_limit_must_be_positive(log: EventLog) -> None:
    with pytest.raises(ValueError):
        log.page_after(None, limit=0)
```

## Common mistakes

- Mixing closed-range and half-open conventions in one function, which causes off-by-one errors and infinite loops.
- Using `low = middle` with floor division in a loop that ends when `low < high`, which never terminates on a two-item range.
- Running binary search on data that is not sorted, or sorted in the opposite order.
- Returning any matching index when the first or last occurrence is needed.
- Writing `while low < high` for a closed range and skipping the final candidate.
- Searching a list with the wrong key. With `bisect` and `key=`, the list must be sorted by that key and the target is compared with key values.
- Choosing answer-space bounds that exclude the real answer, or testing a predicate that is not monotonic.
- Looping on a floating-point range until `low == high`, which may never happen. Use a fixed number of iterations.
- Using `insort` in a loop to build a large sorted list, which is `O(n^2)` because of shifting. Collect the items and sort once.
- Using offset pagination on a changing log, so rows appear twice or are skipped. Use a cursor.
- Not testing the edge cases: empty input, one item, target below the minimum, above the maximum, and absent but in range.

## Practice exercises

1. Trace `binary_search` on `[1, 3, 5, 7, 9, 11]` for the targets `1`, `11`, `6`, and write the values of `low`, `high`, and `middle` at each step.
2. Write `first_occurrence(values, target)` and `last_occurrence(values, target)` using `first_true`, and test them on lists with many duplicates.
3. Write `insert_position(values, target)` and confirm that it agrees with `bisect_left` on 1,000 random sorted lists.
4. Write `search_insert_range(values, target)` that returns `(first, last)` indexes of the target or `(-1, -1)`, in `O(log n)`.
5. Write `first_bad_version(n, is_bad)` for versions `1..n`, where every version from some point on is bad, using as few calls to `is_bad` as possible.
6. Write `find_min_rotated_with_duplicates(values)` and explain why its worst case is `O(n)`.
7. Write `search_2d_matrix(matrix, target)` for a matrix where each row is sorted and each row's first item is larger than the previous row's last item, using one binary search on the flattened index.
8. Write `smallest_divisor(numbers, threshold)` that finds the smallest divisor such that the sum of each number divided by it (rounded up) is at most the threshold, using binary search on the answer.
9. Write `split_array_largest_sum(values, parts)` that splits an array of positive numbers into `parts` contiguous pieces to minimize the largest piece sum, using binary search on the answer and a greedy check.
10. Write `koko_eating_speed(piles, hours)` that finds the minimum eating speed in bites per hour so all piles are finished in time, and justify the bounds of your search.
11. Write `cube_root(value, digits)` for real numbers using floating-point bisection, and test it against `value ** (1 / 3)` with `math.isclose`.
12. Write `kth_smallest_in_sorted_matrix(matrix, k)` using binary search on the value range and a counting function, and state its complexity.
13. Write a random stress test that compares your `lower_bound`, `upper_bound`, and `search_rotated` against brute-force versions on many small lists.
14. Extend `EventLog` with `last_before(timestamp)` that returns the newest entry strictly before a timestamp, and `count_between(start, end)` that counts entries without creating a list. Add tests for both.

## Summary

Binary search finds a position in a sorted sequence in `O(log n)` by discarding half of the range at each step. Its power comes from a monotonic yes-or-no question, so lower bound, upper bound, insertion points, rotated arrays, peaks, and searching an answer space all reduce to finding the first position where a predicate becomes true. Bugs come from inconsistent conventions, so pick a closed or half-open range, write the invariant down, and use the first-true template. Duplicates need lower and upper bounds, floating-point searches need a fixed number of iterations, and rotated arrays with duplicates lose the logarithmic guarantee. Python's `bisect` module, including its `key=` parameter, does the standard searches, and in real systems the same idea supports timestamp range queries and cursor-based pagination over sorted logs.
