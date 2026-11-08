## Overview

Suppose you have a list of a million daily sales figures and a dashboard that asks, thousands of times a second, "what was the total between day 120 and day 480?" Adding up the numbers on every request costs `O(n)` each time. A prefix sum does the adding once, in advance, and then answers any range question in `O(1)` by subtracting two stored numbers.

A prefix sum precomputes running totals. Its mirror image, the difference array, makes the opposite trade: it makes range updates cheap ("add 5 to every day from day 10 to day 99") by recording only where the change starts and where it stops, and rebuilding the full list at the end with one prefix sum.

This note covers one-dimensional prefix sums, range-sum queries, the subarray-sum-equals-`K` pattern with a hash map, two-dimensional prefix sums, difference arrays, prefix XOR and product, and how these ideas show up as running totals in analytics code. The central skill is precomputation: spend `O(n)` once so that every later question costs `O(1)`.

## Core concepts

### One-dimensional prefix sums

Given `values = [v0, v1, ..., v(n-1)]`, the prefix array has one extra slot at the front:

```text
prefix[0] = 0
prefix[1] = v0
prefix[2] = v0 + v1
prefix[i] = v0 + v1 + ... + v(i-1)       (the sum of the first i items)
```

So `prefix[i]` is the sum of `values[:i]`, and `prefix` has length `n + 1`. The leading zero is what makes ranges starting at index 0 work without special cases.

### Range-sum queries

The sum of `values[left:right]` (left included, right excluded) is:

```text
range_sum(left, right) = prefix[right] - prefix[left]
```

The first `right` items minus the first `left` items leaves exactly the items from `left` to `right - 1`. Building the prefix array costs `O(n)` time and `O(n)` space, and every query costs `O(1)`.

### Subarray sum equals K

A subarray is a contiguous slice. To count subarrays whose sum is `K`, note that the sum of the subarray ending at position `j` and starting after position `i` is `prefix[j] - prefix[i]`. It equals `K` when `prefix[i] == prefix[j] - K`. So while scanning, keep a hash map that counts how many times each prefix value has occurred, and for each new running total add the number of earlier prefixes equal to `running - K`. This works even when the numbers can be negative, and it runs in `O(n)`.

### Two-dimensional prefix sums

For a grid, let `prefix[r][c]` be the sum of the rectangle from the top-left corner to just before row `r` and column `c`. A query for any sub-rectangle then uses inclusion and exclusion on four stored values and costs `O(1)`.

### Difference arrays

The difference array stores the change between neighbors: `diff[i] = values[i] - values[i - 1]`. Taking its prefix sum rebuilds the original. To add `delta` to every item from `left` to `right` (inclusive), you change only two entries:

```text
diff[left]      += delta       the increase starts here
diff[right + 1] -= delta       and is cancelled just after the range
```

Each update is `O(1)`. After all updates, one prefix sum rebuilds the final array in `O(n)`. Prefix sums and difference arrays are inverse operations.

### Prefix XOR and prefix product

The same trick works for any operation that can be undone.

- XOR undoes itself, so `prefix_xor[right] ^ prefix_xor[left]` is the XOR of a range.
- Products can be undone by division, but division fails when a value is zero. Problems that ask for the product of everything except one item are better solved with products of the items to the left and to the right of each position.

### Running totals in analytics code

Cumulative sums are everywhere in reporting: year-to-date revenue, cumulative signups, running balance of an account, rolling counts. A prefix array is the data structure behind those reports.

## Basic syntax examples

### Building a prefix array and answering queries

```python
from itertools import accumulate


def build_prefix(values: list[int]) -> list[int]:
    """prefix[i] is the sum of values[:i]. Length n + 1, O(n) time."""
    return [0, *accumulate(values)]


def range_sum(prefix: list[int], left: int, right: int) -> int:
    """Sum of values[left:right], with right excluded. O(1)."""
    return prefix[right] - prefix[left]


sales = [3, 1, 4, 1, 5, 9, 2, 6]
prefix = build_prefix(sales)
print(prefix)  # [0, 3, 4, 8, 9, 14, 23, 25, 31]
print(range_sum(prefix, 2, 6))  # 19, which is 4 + 1 + 5 + 9
print(range_sum(prefix, 0, len(sales)))  # 31, the whole list
```

Trace of `range_sum(prefix, 2, 6)`:

```text
prefix[6] = sum of the first 6 items = 3 + 1 + 4 + 1 + 5 + 9 = 23
prefix[2] = sum of the first 2 items = 3 + 1 = 4
23 - 4 = 19, the sum of items at indexes 2, 3, 4, 5
```

### Counting subarrays with sum K

```python
def count_subarrays_with_sum(values: list[int], k: int) -> int:
    """Count contiguous subarrays whose sum equals k. O(n) time, O(n) space."""
    seen: dict[int, int] = {0: 1}  # the empty prefix has sum 0
    running = 0
    count = 0
    for value in values:
        running += value
        count += seen.get(running - k, 0)  # earlier prefixes that complete a subarray
        seen[running] = seen.get(running, 0) + 1
    return count


print(count_subarrays_with_sum([1, 2, 3], 3))  # 2, the subarrays [1, 2] and [3]
print(count_subarrays_with_sum([1, -1, 0], 0))  # 3
```

### Two-dimensional prefix sums

```python
def build_prefix_2d(grid: list[list[int]]) -> list[list[int]]:
    """prefix[r][c] is the sum of grid[:r][:c]. Size (rows + 1) by (cols + 1)."""
    rows = len(grid)
    cols = len(grid[0]) if grid else 0
    prefix = [[0] * (cols + 1) for _ in range(rows + 1)]
    for r in range(rows):
        for c in range(cols):
            prefix[r + 1][c + 1] = (
                grid[r][c] + prefix[r][c + 1] + prefix[r + 1][c] - prefix[r][c]
            )
    return prefix


def rectangle_sum(
    prefix: list[list[int]], top: int, left: int, bottom: int, right: int
) -> int:
    """Sum of grid[top:bottom][left:right], bottom and right excluded. O(1)."""
    return (
        prefix[bottom][right]
        - prefix[top][right]
        - prefix[bottom][left]
        + prefix[top][left]
    )


grid = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
]
prefix_2d = build_prefix_2d(grid)
print(rectangle_sum(prefix_2d, 1, 1, 3, 3))  # 28, which is 5 + 6 + 8 + 9
```

The four-term formula is inclusion and exclusion. Take the big rectangle from the origin to `(bottom, right)`, subtract the strip above the target, subtract the strip to its left, and add back the corner that was subtracted twice.

### Difference array for range updates

```python
from itertools import accumulate


def apply_range_updates(length: int, updates: list[tuple[int, int, int]]) -> list[int]:
    """Start from zeros and add delta to values[left..right] inclusive for each update.

    O(1) per update and O(n) to rebuild, instead of O(n) per update.
    """
    diff = [0] * (length + 1)  # one extra slot so right + 1 is always valid
    for left, right, delta in updates:
        diff[left] += delta
        diff[right + 1] -= delta
    return list(accumulate(diff[:length]))


result = apply_range_updates(6, [(1, 3, 5), (2, 5, 2)])
print(result)  # [0, 5, 7, 7, 2, 2]
```

Trace of the two updates:

```text
start:                diff = [0, 0, 0, 0, 0, 0, 0]
add 5 to indexes 1-3: diff = [0, 5, 0, 0, -5, 0, 0]
add 2 to indexes 2-5: diff = [0, 5, 2, 0, -5, 0, -2]
prefix sum:           [0, 5, 7, 7, 2, 2]  (the last -2 lands outside and is dropped)
```

### Prefix XOR and product except self

```python
def range_xor(values: list[int], left: int, right: int) -> int:
    """XOR of values[left:right]. Built here for one query; reuse the prefix for many."""
    prefix = [0]
    for value in values:
        prefix.append(prefix[-1] ^ value)
    return prefix[right] ^ prefix[left]


def product_except_self(values: list[int]) -> list[int]:
    """For each position, the product of all other items. No division, O(n)."""
    n = len(values)
    result = [1] * n
    left_product = 1
    for i in range(n):
        result[i] = left_product  # product of everything to the left of i
        left_product *= values[i]
    right_product = 1
    for i in range(n - 1, -1, -1):
        result[i] *= right_product  # multiply in everything to the right of i
        right_product *= values[i]
    return result


print(range_xor([5, 1, 7, 2], 1, 4))  # 4, which is 1 ^ 7 ^ 2
print(product_except_self([1, 2, 3, 4]))  # [24, 12, 8, 6]
print(product_except_self([2, 0, 3]))  # [0, 6, 0], zeros handled without division
```

## Deeper behavior

### Choosing a convention and sticking to it

Most prefix-sum bugs are off-by-one errors. Choose one convention and use it everywhere.

- Half-open ranges: `prefix` has length `n + 1`, and `range_sum(left, right)` covers indexes `left` up to `right - 1`. It matches Python slices, so `sum(values[left:right]) == range_sum(prefix, left, right)`.
- Inclusive ranges: the sum from `left` to `right` inclusive is `prefix[right + 1] - prefix[left]`. Many problem statements use this form, so convert carefully.

The extra leading zero is not optional. Without it, a query that starts at index 0 needs a special case.

### Why the hash-map version handles negative numbers

A sliding window needs the sum to grow when you extend the window and shrink when you shrink it, so it fails when values can be negative. The prefix-sum pattern does not use any such monotonic behavior. It only uses the identity `prefix[j] - prefix[i] == K`, which holds for any numbers. That is why "subarray sum equals `K`" with negative values is a prefix-sum problem and not a sliding-window problem.

The map starts with `{0: 1}` because a subarray that begins at index 0 uses the empty prefix, whose sum is 0. Forgetting it misses every subarray that starts at the beginning.

### The remainder variant

To count subarrays whose sum is divisible by `m`, store prefix sums modulo `m` instead of the sums themselves. Two prefixes with the same remainder differ by a multiple of `m`. Python's `%` returns a result in `0 .. m - 1` even for negative sums, so no special handling is needed, which differs from C or Java.

```python
def subarrays_divisible_by(values: list[int], m: int) -> int:
    counts: dict[int, int] = {0: 1}
    remainder = 0
    total = 0
    for value in values:
        remainder = (remainder + value) % m
        total += counts.get(remainder, 0)
        counts[remainder] = counts.get(remainder, 0) + 1
    return total


print(subarrays_divisible_by([4, 5, 0, -2, -3, 1], 5))  # 7
```

### Why the two-dimensional formula works

`prefix[r][c]` equals the cell `grid[r - 1][c - 1]` plus the prefix above it plus the prefix to its left, minus the overlap that both of those include, `prefix[r - 1][c - 1]`. The query formula is the same four-corner inclusion and exclusion applied to any rectangle. Building costs `O(rows * cols)`, and each query costs `O(1)`.

### Difference arrays and prefix sums are inverses

Taking a prefix sum of a difference array gives the original values. Taking the difference of a prefix array gives the original values back. Seen this way, a range update on the original array is two point updates on the difference array, and a range query on the original array is a point query on its prefix array. When you need both range updates and range queries interleaved, neither array alone is enough. A structure such as a Fenwick tree or a segment tree (covered in the advanced data structures phase) handles that case. Use a difference array only when all updates come before all reads.

### Static data and the cost of changes

A prefix array answers queries on data that does not change. If one value changes, every later prefix entry changes, so a single point update costs `O(n)`. If your data changes often between queries, a plain prefix array is the wrong structure.

### Products and zeros

A prefix product breaks when a value is zero, because later prefixes become zero and cannot be divided out. The left-and-right products technique in `product_except_self` avoids division entirely, so zeros need no special case.

### Integer sizes

Python integers never overflow, so prefix sums of large values just become large integers. In languages with fixed-width integers, prefix sums can overflow even when each value fits. That matters when you port a solution.

## Industry-level snippet

A finance team keeps daily revenue totals, in cents, for each day of a quarter. Analysts ask for the total between any two dates thousands of times per report, and occasionally the team applies adjustments such as refunds or promotional credits to a date range. Both operations map directly onto these structures: range totals use a prefix array, and applying many date-range adjustments uses a difference array.

The data lives in a database table. The service loads a date window into a `DailyTotals` object, answers queries in `O(1)`, applies adjustments, and writes the changed days back inside a single transaction. `BEGIN IMMEDIATE` makes the read-modify-write atomic, so two workers cannot lose each other's adjustments. If any adjustment falls outside the loaded window, the whole operation fails and nothing is written.

```python
# revenue_series.py
import logging
import sqlite3
from collections.abc import Iterable, Iterator, Sequence
from dataclasses import dataclass
from datetime import date, timedelta
from itertools import accumulate
from typing import Self

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS daily_revenue (
    day TEXT PRIMARY KEY,
    cents INTEGER NOT NULL
)
"""

UPSERT_SQL = (
    "INSERT INTO daily_revenue (day, cents) VALUES (?, ?) "
    "ON CONFLICT (day) DO UPDATE SET cents = excluded.cents"
)


class DateRangeError(ValueError):
    """Raised when a date or a range falls outside the loaded series."""


@dataclass(frozen=True, slots=True)
class Adjustment:
    first: date
    last: date
    delta_cents: int


class DailyTotals:
    """Daily totals with O(1) inclusive range queries through a prefix array."""

    def __init__(self, start: date, cents: Sequence[int]) -> None:
        if not cents:
            raise ValueError("a series needs at least one day")
        self._start = start
        self._cents = tuple(cents)
        self._prefix = [0, *accumulate(self._cents)]

    @property
    def start(self) -> date:
        return self._start

    @property
    def end(self) -> date:
        return self._start + timedelta(days=len(self._cents) - 1)

    def _offset(self, day: date) -> int:
        offset = (day - self._start).days
        if not 0 <= offset < len(self._cents):
            raise DateRangeError(f"{day} is outside {self._start} to {self.end}")
        return offset

    def total(self, first: date, last: date) -> int:
        """Total for the inclusive date range, in O(1)."""
        if first > last:
            raise DateRangeError(f"{first} is after {last}")
        return self._prefix[self._offset(last) + 1] - self._prefix[self._offset(first)]

    def items(self) -> Iterator[tuple[date, int]]:
        for offset, amount in enumerate(self._cents):
            yield self._start + timedelta(days=offset), amount

    def with_adjustments(self, adjustments: Iterable[Adjustment]) -> Self:
        """Apply range adjustments with a difference array: O(n + number of updates)."""
        diff = [0] * (len(self._cents) + 1)
        for adjustment in adjustments:
            if adjustment.first > adjustment.last:
                raise DateRangeError(f"{adjustment.first} is after {adjustment.last}")
            diff[self._offset(adjustment.first)] += adjustment.delta_cents
            diff[self._offset(adjustment.last) + 1] -= adjustment.delta_cents
        deltas = accumulate(diff[:-1])
        adjusted = [base + delta for base, delta in zip(self._cents, deltas, strict=True)]
        return type(self)(self._start, adjusted)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def load_series(connection: sqlite3.Connection, start: date, end: date) -> DailyTotals:
    """Load a date window. Days with no row count as zero revenue."""
    if end < start:
        raise DateRangeError(f"{end} is before {start}")
    cents = [0] * ((end - start).days + 1)
    cursor = connection.execute(
        "SELECT day, cents FROM daily_revenue WHERE day BETWEEN ? AND ?",
        (start.isoformat(), end.isoformat()),
    )
    for day_text, amount in cursor:
        cents[(date.fromisoformat(day_text) - start).days] = int(amount)
    return DailyTotals(start, cents)


def apply_adjustments(
    connection: sqlite3.Connection,
    start: date,
    end: date,
    adjustments: Sequence[Adjustment],
) -> DailyTotals:
    """Read, adjust, and write back atomically. Nothing is saved on any error."""
    try:
        with connection:
            connection.execute("BEGIN IMMEDIATE")
            series = load_series(connection, start, end)
            adjusted = series.with_adjustments(adjustments)
            rows = [(day.isoformat(), amount) for day, amount in adjusted.items()]
            connection.executemany(UPSERT_SQL, rows)
    except sqlite3.Error:
        logger.exception("could not apply %d adjustments", len(adjustments))
        raise
    logger.info("applied %d adjustments to %s..%s", len(adjustments), start, end)
    return adjusted
```

```python
# test_revenue_series.py
import sqlite3
from collections.abc import Iterator
from datetime import date

import pytest

from revenue_series import (
    Adjustment,
    DailyTotals,
    DateRangeError,
    apply_adjustments,
    create_schema,
    load_series,
)

START = date(2026, 1, 1)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    conn.executemany(
        "INSERT INTO daily_revenue (day, cents) VALUES (?, ?)",
        [("2026-01-01", 100), ("2026-01-02", 200), ("2026-01-04", 400)],
    )
    conn.commit()
    yield conn
    conn.close()


def day(number: int) -> date:
    return date(2026, 1, number)


def test_totals_use_inclusive_ranges() -> None:
    series = DailyTotals(START, [100, 200, 300, 400])
    assert series.total(day(1), day(4)) == 1_000
    assert series.total(day(2), day(3)) == 500
    assert series.total(day(3), day(3)) == 300


def test_dates_outside_the_series_are_rejected() -> None:
    series = DailyTotals(START, [1, 2, 3])
    with pytest.raises(DateRangeError):
        series.total(day(1), day(4))
    with pytest.raises(DateRangeError):
        series.total(day(3), day(2))


def test_adjustments_apply_to_the_whole_range() -> None:
    series = DailyTotals(START, [10, 10, 10, 10, 10])
    adjusted = series.with_adjustments(
        [Adjustment(day(2), day(4), 5), Adjustment(day(3), day(5), -2)]
    )
    assert [amount for _, amount in adjusted.items()] == [10, 15, 13, 13, 8]


def test_missing_days_load_as_zero(connection: sqlite3.Connection) -> None:
    series = load_series(connection, day(1), day(5))
    assert [amount for _, amount in series.items()] == [100, 200, 0, 400, 0]


def test_apply_adjustments_persists_the_result(
    connection: sqlite3.Connection,
) -> None:
    adjusted = apply_adjustments(
        connection, day(1), day(5), [Adjustment(day(2), day(3), 50)]
    )
    assert adjusted.total(day(1), day(5)) == 800
    reloaded = load_series(connection, day(1), day(5))
    assert [amount for _, amount in reloaded.items()] == [100, 250, 50, 400, 0]


def test_invalid_adjustment_changes_nothing(connection: sqlite3.Connection) -> None:
    with pytest.raises(DateRangeError):
        apply_adjustments(
            connection,
            day(1),
            day(5),
            [Adjustment(day(2), day(3), 50), Adjustment(day(4), day(9), 10)],
        )
    reloaded = load_series(connection, day(1), day(5))
    assert [amount for _, amount in reloaded.items()] == [100, 200, 0, 400, 0]
```

## Common mistakes

- Building the prefix array without the leading zero, so every query that starts at index 0 needs a special case or breaks.
- Mixing inclusive and exclusive ranges. Write the convention in a comment next to the function.
- Forgetting to seed the hash map with `{0: 1}` in the subarray-sum pattern.
- Updating the map before reading it, which counts the empty subarray or a subarray with itself.
- Using a sliding window on arrays with negative numbers when a prefix-sum and hash-map approach is needed.
- Rebuilding the prefix array after every single update. If values change often, a prefix array is the wrong structure.
- Forgetting the extra slot in a difference array, so `right + 1` falls off the end.
- Dividing by a prefix product that may contain zero.
- In the two-dimensional formula, forgetting to add back the corner `prefix[top][left]`.
- Writing the 2D prefix with the wrong offsets (`prefix[r][c]` versus `prefix[r + 1][c + 1]`).
- Using `//` or `%` on negative sums in other languages and assuming Python's behavior carries over.

## Practice exercises

1. Build the prefix sums of `[2, 4, 6, 8, 10]` by hand and use them to find the sum of the middle three numbers. Then verify with code.
2. Write a class `RangeSum` with `__init__(values)` and `query(left, right)` for inclusive ranges, and test it against `sum(values[left : right + 1])` on random inputs.
3. Write `pivot_index(values)` that returns the first index where the sum to the left equals the sum to the right, or `-1`. Use one prefix pass.
4. Write `max_subarray_of_length_k(values, k)` with prefix sums, then solve it again with a sliding window and compare.
5. Write `count_subarrays_with_sum(values, k)` and test it against a brute-force `O(n^2)` version on random lists with negative numbers.
6. Write `longest_subarray_with_sum(values, k)` that returns the length of the longest subarray summing to `k`. Store the first index at which each prefix sum occurs.
7. A binary array contains zeros and ones. Write `longest_balanced_subarray(bits)` for the longest subarray with equal numbers of zeros and ones by treating zeros as `-1`.
8. Write `build_prefix_2d` and `rectangle_sum` from memory, then test them against a brute-force sum on a random 6 by 6 grid.
9. Write `count_squares_with_sum(grid, target)` for a matrix of non-negative integers, returning how many square sub-matrices sum to `target`. Use the 2D prefix array.
10. Starting with a list of `n` zeros, apply 100,000 random range additions with a difference array. Compare the speed with applying each update in a loop.
11. Write `corporate_flight_bookings(bookings, n)` where each booking `(first, last, seats)` reserves seats on flights `first` through `last`, and return the total seats booked on each flight.
12. Write `subarray_xor_equals_k(values, k)` that counts subarrays whose XOR is `k`, using the prefix-XOR version of the hash-map pattern.
13. Explain in your own words why the sliding-window technique cannot count subarrays summing to `K` in a list with negative numbers but the prefix-sum technique can.
14. Extend `DailyTotals` with a method `best_window(length)` that returns the start date of the window of `length` days with the highest total, using the prefix array.

## Summary

A prefix sum stores running totals so that the sum of any range is the difference of two stored values, giving `O(1)` queries after `O(n)` preparation. The leading zero and a single consistent range convention prevent most off-by-one bugs. Combined with a hash map of earlier prefix values, the same idea counts subarrays with a target sum in `O(n)`, even with negative numbers, and a modulo version counts subarrays divisible by `m`. In two dimensions, inclusion and exclusion over four stored values answers rectangle sums. The difference array is the inverse: it makes range updates `O(1)` by marking where a change starts and stops, and one prefix sum rebuilds the result. Both structures assume the data is static between phases, and they power the cumulative totals used throughout analytics code.
