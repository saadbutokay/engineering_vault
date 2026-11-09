## Overview

Often you do not need a list sorted. You need one item from it: the median salary, the 95th percentile response time, the tenth highest score, the three most expensive orders. The `k`-th smallest item of a collection is called its `k`-th order statistic, and finding it is the selection problem.

The obvious solution is to sort and index, which costs `O(n log n)`. But sorting does far more work than the question requires, because it orders every item, when you only care about the position of one. Selection algorithms find the `k`-th item in `O(n)` expected time by reusing the partitioning step from quicksort and then following only the side that contains the answer. When you need several of the smallest or largest items, a heap does the job in `O(n log k)` time with only `O(k)` memory.

This note covers partitioning, quickselect and its expected complexity, the median of medians algorithm for a guaranteed linear worst case, finding the `k`-th largest with a heap compared with quickselect, `heapq.nlargest` and `nsmallest`, and finding medians and percentiles. Along the way you will see where the real-world pitfalls are: ties, `NaN` values, and the cost of asking for many percentiles.

## Core concepts

### Order statistics

The `k`-th order statistic of a collection is its `k`-th smallest item when counting from 1. In code, counting from 0 is more convenient, so this note uses a zero-based rank: rank 0 is the minimum, rank `n - 1` is the maximum, and the median sits at rank `n // 2` (for an odd count).

- Minimum and maximum: ranks `0` and `n - 1`. Easy in `O(n)`.
- Median: the middle value. For an even count, it is the average of the two middle values.
- Percentile: the value below which a given percentage of the data falls.

### Partitioning

Partitioning rearranges a list around a pivot value so that items smaller than the pivot come first, equal items next, and larger items last. After partitioning, the pivot's group sits exactly where it would end up in the sorted list. You do not know the order inside the smaller and larger groups, and you do not need to.

### Quickselect

Quickselect finds the item of rank `k` like this:

1. Pick a pivot and partition the list.
2. If rank `k` falls inside the group of items equal to the pivot, the pivot is the answer.
3. If `k` is in the smaller group, repeat on that group only. Otherwise repeat on the larger group, adjusting `k`.

Unlike quicksort, it never sorts both sides, so each step discards a constant fraction of the data on average. The total work is `n + n/2 + n/4 + ...`, which is about `2n`.

- Expected time `O(n)` with a random pivot.
- Worst case `O(n^2)` if pivots are consistently terrible, which is astronomically unlikely with random pivots.

### Median of medians

The median of medians algorithm picks a pivot that is guaranteed to be reasonably central: split the items into groups of five, take the median of each group, and use the median of those medians as the pivot. At least 30 percent of the items are smaller and at least 30 percent are larger, so every step discards a fixed fraction, and the worst case is `O(n)`. The constant factor is large, so in practice random pivots are used and median of medians matters mainly for guarantees and for understanding.

### Kth largest with a heap

To find the `k`-th largest of `n` items, keep a min-heap holding the `k` largest items seen so far. For each new item that is larger than the heap's smallest, replace the smallest. At the end the heap's root is the `k`-th largest.

- Time `O(n log k)`, space `O(k)`.
- Works on a stream of data that does not fit in memory.
- Better than quickselect when `k` is small, or when the data arrives one item at a time.

### heapq.nlargest and nsmallest

`heapq.nlargest(k, items)` and `heapq.nsmallest(k, items)` return the `k` largest or smallest items in sorted order, using the heap approach internally. They accept a `key=` function.

### Medians and percentiles

A median is a selection problem. A percentile is as well: with the nearest-rank definition, the `p`-th percentile of `n` values is the value at rank `ceil(p / 100 * n)` counting from 1. Percentiles such as p50, p95, and p99 are the standard way to describe latency, because averages hide slow outliers.

## Basic syntax examples

### Select by sorting: the baseline

```python
def kth_smallest_sorted(values: list[int], k: int) -> int:
    """Rank k counted from 0. O(n log n) because everything gets sorted."""
    if not 0 <= k < len(values):
        raise IndexError("k is out of range")
    return sorted(values)[k]


print(kth_smallest_sorted([7, 2, 9, 4, 5], 2))  # 5
```

### Three-way partitioning

```python
def partition_three_way(
    items: list[float], low: int, high: int, pivot: float
) -> tuple[int, int]:
    """Rearrange items[low..high] into smaller, equal, and larger than the pivot.

    Returns (first_equal, last_equal): the index range of the equal items.
    """
    smaller, current, larger = low, low, high
    while current <= larger:
        if items[current] < pivot:
            items[smaller], items[current] = items[current], items[smaller]
            smaller += 1
            current += 1
        elif items[current] > pivot:
            items[current], items[larger] = items[larger], items[current]
            larger -= 1  # the swapped-in item has not been inspected yet
        else:
            current += 1
    return smaller, larger


data = [5.0, 2.0, 9.0, 5.0, 1.0, 7.0, 5.0]
print(partition_three_way(data, 0, len(data) - 1, 5.0))  # (2, 4)
print(data)  # [1.0, 2.0, 5.0, 5.0, 5.0, 7.0, 9.0]
```

### Quickselect

```python
import random
from collections.abc import Sequence


def quickselect(
    values: Sequence[float], k: int, rng: random.Random | None = None
) -> float:
    """The item of rank k (counting from 0). Expected O(n). Does not modify `values`."""
    if not 0 <= k < len(values):
        raise IndexError("k is out of range")
    rng = rng or random.Random()
    items = list(values)  # work on a copy
    low, high = 0, len(items) - 1
    while True:
        pivot = items[rng.randint(low, high)]
        first_equal, last_equal = partition_three_way(items, low, high, pivot)
        if k < first_equal:
            high = first_equal - 1  # the answer is among the smaller items
        elif k > last_equal:
            low = last_equal + 1  # the answer is among the larger items
        else:
            return pivot  # rank k lies in the block of items equal to the pivot


print(quickselect([7, 2, 9, 4, 5], 2))  # 5
print(quickselect([3, 3, 3, 3], 1))  # 3
```

Trace of `quickselect([7, 2, 9, 4, 5], 2)` with pivot `5` on the first round:

```text
partition around 5 -> [2, 4, 5, 7, 9]   smaller: [2, 4], equal: [5], larger: [7, 9]
first_equal = 2, last_equal = 2
k = 2 lies in [2, 2], so the answer is 5
```

Only one partition pass over the data was needed, and no sorting.

### Kth largest with a heap

```python
import heapq


def kth_largest(values: Sequence[float], k: int) -> float:
    """The k-th largest, counting from 1. O(n log k) time, O(k) space."""
    if not 1 <= k <= len(values):
        raise ValueError("k must be between 1 and len(values)")
    heap = list(values[:k])
    heapq.heapify(heap)  # a min-heap of the k largest items so far
    for value in values[k:]:
        if value > heap[0]:
            heapq.heapreplace(heap, value)  # pop the smallest, push the new one
    return heap[0]


print(kth_largest([3, 2, 1, 5, 6, 4], 2))  # 5
```

### heapq.nlargest and nsmallest

```python
import heapq

scores = [88, 92, 79, 95, 85, 91, 70]
print(heapq.nlargest(3, scores))  # [95, 92, 91]
print(heapq.nsmallest(2, scores))  # [70, 79]

orders = [("a", 120), ("b", 80), ("c", 300), ("d", 150)]
print(heapq.nlargest(2, orders, key=lambda order: order[1]))  # [('c', 300), ('d', 150)]
```

### Median

```python
def median(values: Sequence[float]) -> float:
    """Median in expected O(n). The average of the two middle values if n is even."""
    n = len(values)
    if n == 0:
        raise ValueError("median of an empty sequence")
    middle = n // 2
    if n % 2 == 1:
        return quickselect(values, middle)
    return (quickselect(values, middle - 1) + quickselect(values, middle)) / 2


print(median([5, 1, 9, 3, 7]))  # 5
print(median([4, 1, 3, 2]))  # 2.5
```

### Percentiles with the nearest-rank rule

```python
import math


def percentile(values: Sequence[float], p: float) -> float:
    """Nearest-rank percentile for 0 < p <= 100."""
    if not 0 < p <= 100:
        raise ValueError("p must be in (0, 100]")
    rank = math.ceil(p / 100 * len(values))  # counting from 1
    return quickselect(values, rank - 1)


latencies = [120, 85, 300, 95, 110, 105, 90, 2_000, 100, 115]
print(percentile(latencies, 50))  # 105
print(percentile(latencies, 90))  # 300
print(percentile(latencies, 100))  # 2000
```

## Deeper behavior

### Why quickselect is linear on average

With a random pivot, each partition leaves, on average, a side that is no more than about three quarters of the original. If every step removed a constant fraction, the work would form a geometric series: `n + 3n/4 + (3/4)^2 n + ...`, which sums to `4n`. The exact expected comparison count with a random pivot is about `3.4 n` for finding the median. The recurrence is `T(n) = T(3n/4) + O(n)`, and the Master theorem puts it in `Θ(n)` because the top-level work dominates.

The worst case happens when the pivot is repeatedly the smallest or largest item in the range, so each pass removes one item, giving `n + (n - 1) + ... = O(n^2)`. With a random pivot the chance of anything close to that on a large input is negligible.

### Why three-way partitioning matters here

If the list has many equal values, partitioning into only "smaller" and "not smaller" can keep almost all of them on one side and make every pass slow. With three-way partitioning, a block of equal items equal to the pivot is removed in one pass, and a list of `n` identical values is finished in a single `O(n)` pass.

### Median of medians, concretely

```python
def select_deterministic(values: list[int], k: int) -> int:
    """Worst-case O(n) selection of rank k (from 0). Clear, but slow in practice."""
    if len(values) <= 5:
        return sorted(values)[k]
    groups = [sorted(values[i : i + 5]) for i in range(0, len(values), 5)]
    medians = [group[len(group) // 2] for group in groups]
    pivot = select_deterministic(medians, len(medians) // 2)  # median of the medians
    smaller = [value for value in values if value < pivot]
    equal = [value for value in values if value == pivot]
    if k < len(smaller):
        return select_deterministic(smaller, k)
    if k < len(smaller) + len(equal):
        return pivot
    larger = [value for value in values if value > pivot]
    return select_deterministic(larger, k - len(smaller) - len(equal))


print(select_deterministic([9, 1, 8, 2, 7, 3, 6, 4, 5, 0, 12, 11], 5))  # 5
```

The guarantee comes from the pivot. Half of the group medians are at most the pivot, and each such group contributes at least three items (the median and the two smaller ones) that are at most the pivot, so at least about `3n/10` items are at most the pivot. By symmetry at least `3n/10` are at least the pivot. The recursion then handles at most `7n/10` items, and the recurrence `T(n) = T(n/5) + T(7n/10) + O(n)` solves to `O(n)`. The cost is sorting groups, building lists, and a double recursion, so real programs use random pivots instead.

### Choosing between quickselect and a heap

- One selection on data already in memory: quickselect, expected `O(n)`.
- Several of the smallest or largest items, `k` small: heap, `O(n log k)`, and the result comes out sorted for free.
- A stream, or data too large for memory: heap, because it holds only `k` items.
- `k` close to `n`: sorting is just as good, because `n log k` is about `n log n`.
- Many different ranks (many percentiles): sort once. Each quickselect costs `O(n)`, so after about `log2(n)` selections, one sort is cheaper.

The standard library function `statistics.median` simply sorts the data, which is fine for small inputs but `O(n log n)`. Libraries such as NumPy implement `partition`, a selection algorithm called introselect, which behaves like quickselect with a fallback to guarantee linear time.

### Heap-based selection in detail

The heap holds the `k` largest items, with the smallest of them at the root. A new item is useful only if it beats the root. Each replacement costs `O(log k)`, and most items on random data are rejected with one comparison, so the typical cost is close to `O(n)`. On data that is sorted ascending, every item replaces the root, which is the worst case, `O(n log k)`.

### Ties and ranks

With duplicates, several positions share the same value. The item of rank `k` is well defined even with ties, because it is the value at position `k` of the sorted list. Percentile definitions differ between tools: nearest-rank returns an actual data value, while linear interpolation (used by NumPy and `statistics.quantiles`) can return a value between two data points. When comparing results across systems, check which definition each uses.

### NaN and mixed types

`float("nan")` is not equal to or ordered against anything, so every comparison involving it is false. Partitioning with a `NaN` in the data silently produces wrong ranks. Validate input and reject or remove `NaN` before selecting. Comparing numbers with strings or `None` raises `TypeError`.

### Selection does not sort

Quickselect leaves the items it examined partially rearranged. Working on a copy, as the code above does, protects the caller's data at the cost of `O(n)` extra memory. If the caller does not care about order, you can work in place and save the copy.

## Industry-level snippet

A platform team publishes service-level objective reports. Every hour, for each endpoint, they must report the 50th, 95th, and 99th percentile response times and list the five slowest requests, so that engineers can investigate them. A single endpoint can handle millions of requests per hour, so the report should not sort everything when it needs only a few values.

The code uses quickselect for one to a few percentiles and a single sort when many are requested, because that is cheaper beyond a handful of selections. The slowest requests come from `heapq.nlargest`, which needs only `O(k)` memory. Samples are validated first, because one `NaN` or negative duration would silently corrupt every rank. The finished report replaces the previous report for the same time window in a single transaction.

```python
# percentile_report.py
import heapq
import logging
import math
import random
import sqlite3
from collections.abc import Iterable, Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS slo_report (
    window TEXT NOT NULL,
    percentile REAL NOT NULL,
    value_ms REAL NOT NULL,
    PRIMARY KEY (window, percentile)
)
"""

INSERT_SQL = "INSERT INTO slo_report (window, percentile, value_ms) VALUES (?, ?, ?)"

SORT_ONCE_THRESHOLD = 4  # beyond this many percentiles, one sort is cheaper


class InvalidSampleError(ValueError):
    """Raised for a sample that is NaN, infinite, or negative."""


class InvalidPercentileError(ValueError):
    """Raised for a percentile outside the range (0, 100]."""


@dataclass(frozen=True, slots=True)
class Request:
    request_id: int
    duration_ms: float


def validate_samples(samples: Sequence[float]) -> None:
    if not samples:
        raise ValueError("at least one sample is required")
    for sample in samples:
        if not math.isfinite(sample) or sample < 0:
            raise InvalidSampleError(f"invalid duration: {sample!r}")


def nearest_rank_index(count: int, p: float) -> int:
    """Zero-based index of the nearest-rank percentile in a sorted list."""
    if not 0 < p <= 100:
        raise InvalidPercentileError(f"percentile must be in (0, 100], got {p}")
    return max(math.ceil(p / 100 * count), 1) - 1


def partition_three_way(
    items: list[float], low: int, high: int, pivot: float
) -> tuple[int, int]:
    smaller, current, larger = low, low, high
    while current <= larger:
        if items[current] < pivot:
            items[smaller], items[current] = items[current], items[smaller]
            smaller += 1
            current += 1
        elif items[current] > pivot:
            items[current], items[larger] = items[larger], items[current]
            larger -= 1
        else:
            current += 1
    return smaller, larger


def quickselect(values: Sequence[float], k: int, rng: random.Random) -> float:
    items = list(values)
    low, high = 0, len(items) - 1
    while True:
        pivot = items[rng.randint(low, high)]
        first_equal, last_equal = partition_three_way(items, low, high, pivot)
        if k < first_equal:
            high = first_equal - 1
        elif k > last_equal:
            low = last_equal + 1
        else:
            return pivot


def percentiles(
    samples: Sequence[float],
    wanted: Sequence[float],
    rng: random.Random | None = None,
) -> dict[float, float]:
    """Return {percentile: value}. Quickselect for a few, one sort for many."""
    validate_samples(samples)
    indexes = {p: nearest_rank_index(len(samples), p) for p in wanted}
    if len(indexes) > SORT_ONCE_THRESHOLD:
        ordered = sorted(samples)
        return {p: ordered[index] for p, index in indexes.items()}
    rng = rng or random.Random()
    return {p: quickselect(samples, index, rng) for p, index in indexes.items()}


def slowest_requests(requests: Iterable[Request], k: int) -> list[Request]:
    """The k slowest requests in O(n log k) time and O(k) memory.

    Ties on duration are broken by the lower request id so the result is stable.
    """
    if k < 1:
        raise ValueError("k must be at least 1")
    return heapq.nsmallest(k, requests, key=lambda r: (-r.duration_ms, r.request_id))


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_report(
    connection: sqlite3.Connection, window: str, values: dict[float, float]
) -> int:
    """Replace the report for a window atomically. Returns the number of rows."""
    if not window:
        raise ValueError("window must not be empty")
    rows = [(window, p, value) for p, value in sorted(values.items())]
    try:
        with connection:
            connection.execute("DELETE FROM slo_report WHERE window = ?", (window,))
            connection.executemany(INSERT_SQL, rows)
    except sqlite3.Error:
        logger.exception("could not save the report for window %s", window)
        raise
    logger.info("saved %d percentiles for window %s", len(rows), window)
    return len(rows)
```

```python
# test_percentile_report.py
import math
import random
import sqlite3
from collections.abc import Iterator

import pytest

from percentile_report import (
    InvalidPercentileError,
    InvalidSampleError,
    Request,
    create_schema,
    nearest_rank_index,
    percentiles,
    save_report,
    slowest_requests,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def oracle(samples: list[float], p: float) -> float:
    return sorted(samples)[nearest_rank_index(len(samples), p)]


def test_known_values() -> None:
    samples = [120, 85, 300, 95, 110, 105, 90, 2_000, 100, 115]
    result = percentiles(samples, [50, 90, 100])
    assert result == {50: 105, 90: 300, 100: 2_000}


def test_single_sample_is_every_percentile() -> None:
    assert percentiles([42.0], [1, 50, 100]) == {1: 42.0, 50: 42.0, 100: 42.0}


def test_all_equal_samples() -> None:
    assert percentiles([7.0] * 50, [50, 99]) == {50: 7.0, 99: 7.0}


def test_random_samples_match_the_sort_based_oracle() -> None:
    rng = random.Random(21)
    for _ in range(300):
        samples = [float(rng.randint(0, 20)) for _ in range(rng.randint(1, 60))]
        wanted = [rng.choice([1, 25, 50, 75, 90, 95, 99, 100]) for _ in range(3)]
        result = percentiles(samples, wanted, random.Random(1))
        for p in wanted:
            assert result[p] == oracle(samples, p)


def test_both_strategies_agree() -> None:
    rng = random.Random(4)
    samples = [rng.random() * 1_000 for _ in range(500)]
    few = percentiles(samples, [50, 95], random.Random(2))
    many = percentiles(samples, [50, 95, 10, 20, 30, 40], random.Random(2))
    assert few[50] == many[50]
    assert few[95] == many[95]


@pytest.mark.parametrize("bad", [math.nan, math.inf, -1.0])
def test_invalid_samples_are_rejected(bad: float) -> None:
    with pytest.raises(InvalidSampleError):
        percentiles([1.0, bad, 3.0], [50])


def test_empty_samples_are_rejected() -> None:
    with pytest.raises(ValueError):
        percentiles([], [50])


@pytest.mark.parametrize("p", [0, -5, 100.1])
def test_invalid_percentiles_are_rejected(p: float) -> None:
    with pytest.raises(InvalidPercentileError):
        percentiles([1.0, 2.0], [p])


def test_slowest_requests_orders_by_duration_then_id() -> None:
    requests = [
        Request(1, 50.0),
        Request(2, 900.0),
        Request(3, 900.0),
        Request(4, 20.0),
        Request(5, 400.0),
    ]
    top = slowest_requests(requests, 3)
    assert [r.request_id for r in top] == [2, 3, 5]
    with pytest.raises(ValueError):
        slowest_requests(requests, 0)


def test_save_report_replaces_the_window(connection: sqlite3.Connection) -> None:
    save_report(connection, "2026-11-08T10", {50: 100.0, 95: 400.0})
    save_report(connection, "2026-11-08T10", {50: 110.0})
    rows = connection.execute(
        "SELECT percentile, value_ms FROM slo_report ORDER BY percentile"
    ).fetchall()
    assert rows == [(50.0, 110.0)]


def test_empty_window_is_rejected(connection: sqlite3.Connection) -> None:
    with pytest.raises(ValueError):
        save_report(connection, "", {50: 1.0})
```

The switch at `SORT_ONCE_THRESHOLD` is an example of a tuning constant that should come from measurement on real data. Write down why it exists, and revisit it when the data size changes.

## Common mistakes

- Sorting the whole list when only one rank is needed.
- Mixing zero-based and one-based ranks. The `k`-th smallest "counting from 1" is index `k - 1`.
- Using a two-way partition on data with many duplicates and getting quadratic behavior.
- Using a fixed pivot (such as the first item) on sorted data, which is the worst case for quickselect.
- Forgetting that quickselect rearranges its input. Work on a copy if the caller needs the original order.
- Computing a median of an even-length list by taking only one middle item.
- Calling `heapq.nlargest` with `k` close to `n`, where sorting is faster, or calling it in a loop to get successive ranks.
- Treating `statistics.median` as linear time. It sorts.
- Letting `NaN`, `None`, or negative durations into a percentile computation.
- Reporting an average of latencies when a percentile is what describes the user experience.
- Comparing percentile numbers across tools without checking which interpolation rule each one uses.
- Computing many percentiles by calling quickselect separately for each one on a huge list. Sort once.

## Practice exercises

1. Write `kth_smallest(values, k)` using `sorted`, then with quickselect, and test that they agree on 1,000 random lists.
2. Run `partition_three_way` by hand on `[4, 7, 4, 1, 9, 4, 2]` with pivot `4`, and write the list after each swap.
3. Write `quickselect_in_place(values, k)` that rearranges the input, and compare its memory use with the copying version.
4. Write `kth_largest_distinct(values, k)` that returns the `k`-th largest distinct value, and decide how it differs from `kth_largest`.
5. Write `top_k_frequent(words, k)` using `Counter` and `heapq.nlargest`, and state its complexity.
6. Write `closest_to_median(values, k)` that returns the `k` items nearest to the median, using selection.
7. Write `running_kth_largest(stream, k)` as a generator that yields the current `k`-th largest after each new item, using a min-heap of size `k`.
8. Compare the time of `sorted(values)[k]`, quickselect, and `heapq.nsmallest(k + 1, values)[-1]` for `n = 1,000,000` and `k = 10`, `k = 500,000`. Explain which wins and why.
9. Write `select_deterministic` with groups of seven instead of five and explain how the recurrence changes.
10. Write `weighted_median(values, weights)` that returns the item where the cumulative weight first reaches half of the total, and decide whether sorting is needed.
11. Write `trimmed_mean(values, fraction)` that drops the lowest and highest fraction of the data using selection, not a full sort.
12. Write `p95_per_endpoint(requests)` that groups requests by endpoint with a `defaultdict` and computes the 95th percentile for each group. Think about what happens with endpoints that have very few samples.
13. Explain why the average latency of `[100, 100, 100, 100, 5000]` is misleading and what p50 and p99 reveal.
14. Extend `percentiles` to support linear interpolation as an option, and add tests that compare your results with `statistics.quantiles` using `method="inclusive"`.

## Summary

Selection finds the `k`-th smallest item without sorting. Partition the data around a pivot, keep only the side that contains rank `k`, and repeat: with a random pivot this is quickselect, expected `O(n)`, and with three-way partitioning it handles duplicates. Median of medians chooses a pivot that guarantees a linear worst case, at the cost of a large constant, so it is mostly a theoretical tool. When you need the top `k` items, especially from a stream, a size-`k` heap costs `O(n log k)` time and `O(k)` memory, and `heapq.nlargest` and `nsmallest` wrap it. Medians and percentiles are selection problems, nearest-rank is the simplest definition, and a single sort is cheaper when you need many percentiles. Validate the data first, because `NaN` values and mixed types quietly break ranks.
