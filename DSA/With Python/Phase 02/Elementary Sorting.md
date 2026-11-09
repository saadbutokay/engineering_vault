## Overview

Sorting puts items in order, and it is one of the most common operations in software: showing results by date, ranking scores, preparing data for binary search, grouping equal items together, and removing duplicates. Python's built-in `sorted` and `list.sort` are fast, stable, and reliable, so you will almost never write your own sort in production.

You still study sorting algorithms because they teach reasoning that applies everywhere: loop invariants, counting operations, trading time for simplicity, and understanding properties such as stability. This note covers the three elementary sorts, bubble sort, selection sort, and insertion sort. All three take `O(n^2)` time in the worst case, but they behave differently, and insertion sort is genuinely useful: it is the fastest choice for tiny or nearly sorted inputs, and the efficient sorts in the next note call it for small pieces.

You will learn how each algorithm works, what stability, in-place sorting, and adaptive behavior mean, why insertion sort still matters, and how to count comparisons and swaps to compare algorithms with data rather than opinions.

## Core concepts

### What a comparison sort does

A comparison sort orders items using only comparisons: "is `a` before `b`?" It does not need to know anything else about the items. That generality is why one sort works for numbers, strings, and records, but it also sets a speed limit that the next note explains.

### Bubble sort

Repeatedly scan the list and swap any adjacent pair that is out of order. After each full pass the largest remaining item has "bubbled" to the end of the unsorted part. If a pass makes no swaps, the list is sorted and the algorithm can stop early.

### Selection sort

Repeatedly find the smallest item in the unsorted part and swap it to the front of that part. The sorted prefix grows by one item per pass. It always makes about `n^2 / 2` comparisons, but at most `n - 1` swaps.

### Insertion sort

Treat the front of the list as a sorted section. Take the next item and slide it left past every larger item until it fits, like arranging playing cards in your hand. The sorted section grows by one item per step.

### Stability

A sort is stable if items with equal keys keep their original relative order. Stability matters whenever you sort by more than one property. If you sort customers by city and the sort is stable, customers in the same city stay in whatever order they were in before, for example sorted by name from an earlier pass.

- Bubble sort (swapping only when strictly greater): stable.
- Insertion sort (shifting only when strictly greater): stable.
- Selection sort (swapping the minimum into place): not stable.

### In-place sorting

An in-place sort rearranges the list itself using `O(1)` extra memory. All three elementary sorts are in place. `list.sort()` in Python also sorts in place, while `sorted()` builds a new list.

### Adaptive behavior

A sort is adaptive if it runs faster when the input is already nearly sorted.

- Insertion sort is adaptive: its cost is `O(n + inversions)`, where an inversion is a pair of items that are out of order. A sorted list has zero inversions.
- Bubble sort with early exit is adaptive: a sorted list needs one pass.
- Selection sort is not adaptive: it does the same comparisons on any input.

### Counting comparisons and swaps

Time on a real machine is noisy. Counting comparisons and moves is a clean, repeatable way to compare algorithms and to check that your analysis matches what the code actually does.

### Why insertion sort still matters

For small inputs, the constant factors dominate, and insertion sort has very small ones: simple loops, no recursion, good memory locality. For nearly sorted input it is close to linear. That is why production sorts use it: Python's Timsort sorts short runs with insertion sort, and many `n log n` sort implementations switch to insertion sort when a piece shrinks below a small threshold.

## Basic syntax examples

### Counting operations

All three sorts below sort a list of integers in place and return a small record of how much work they did. For bubble and selection sort a "move" is a swap. For insertion sort a "move" is a shift of one item one slot to the right.

```python
from dataclasses import dataclass


@dataclass(slots=True)
class SortStats:
    comparisons: int = 0
    moves: int = 0
```

### Bubble sort

```python
def bubble_sort(values: list[int]) -> SortStats:
    """Swap adjacent out-of-order pairs. Stable, in place, adaptive with early exit."""
    stats = SortStats()
    for end in range(len(values) - 1, 0, -1):
        swapped = False
        for i in range(end):
            stats.comparisons += 1
            if values[i] > values[i + 1]:
                values[i], values[i + 1] = values[i + 1], values[i]
                stats.moves += 1
                swapped = True
        if not swapped:  # a pass with no swaps means the list is sorted
            break
    return stats
```

Trace of one pass on `[5, 1, 4, 2]`:

```text
compare 5 and 1: swap  -> [1, 5, 4, 2]
compare 5 and 4: swap  -> [1, 4, 5, 2]
compare 5 and 2: swap  -> [1, 4, 2, 5]   the largest item is now in its final place
```

### Selection sort

```python
def selection_sort(values: list[int]) -> SortStats:
    """Move the smallest remaining item to the front. At most n - 1 swaps."""
    stats = SortStats()
    n = len(values)
    for start in range(n - 1):
        smallest = start
        for i in range(start + 1, n):
            stats.comparisons += 1
            if values[i] < values[smallest]:
                smallest = i
        if smallest != start:
            values[start], values[smallest] = values[smallest], values[start]
            stats.moves += 1
    return stats
```

### Insertion sort

```python
def insertion_sort(values: list[int]) -> SortStats:
    """Grow a sorted prefix by sliding each new item left into place."""
    stats = SortStats()
    for i in range(1, len(values)):
        current = values[i]
        j = i - 1
        while j >= 0:
            stats.comparisons += 1
            if values[j] > current:
                values[j + 1] = values[j]  # shift the larger item right
                stats.moves += 1
                j -= 1
            else:
                break
        values[j + 1] = current  # drop the item into the gap
    return stats
```

Trace of insertion sort on `[4, 3, 2]`:

```text
start:        [4, 3, 2]     sorted prefix is [4]
i=1, item 3:  shift 4 right -> [4, 4, 2], place 3 -> [3, 4, 2]
i=2, item 2:  shift 4, shift 3 -> [3, 3, 4], place 2 -> [2, 3, 4]
```

### Comparing them with real counts

```python
for name, sort in (
    ("bubble", bubble_sort),
    ("selection", selection_sort),
    ("insertion", insertion_sort),
):
    for label, data in (
        ("sorted", list(range(10))),
        ("reversed", list(range(9, -1, -1))),
    ):
        stats = sort(data)
        counts = f"comparisons={stats.comparisons:<4}moves={stats.moves}"
        print(f"{name:<10}{label:<10}{counts}")
```

```text
bubble    sorted    comparisons=9   moves=0
bubble    reversed  comparisons=45  moves=45
selection sorted    comparisons=45  moves=0
selection reversed  comparisons=45  moves=5
insertion sorted    comparisons=9   moves=0
insertion reversed  comparisons=45  moves=45
```

Read the table carefully. On sorted input, bubble sort and insertion sort need only `n - 1 = 9` comparisons while selection sort still does 45. On reversed input all three need 45 comparisons, but selection sort makes only 5 swaps while the others make 45 moves.

### Sorting records by a key

```python
from collections.abc import Callable


def insertion_sort_by[T](items: list[T], key: Callable[[T], int]) -> None:
    """Stable: an item moves left only past items with a strictly larger key."""
    for i in range(1, len(items)):
        current = items[i]
        current_key = key(current)
        j = i - 1
        while j >= 0 and key(items[j]) > current_key:
            items[j + 1] = items[j]
            j -= 1
        items[j + 1] = current


def selection_sort_by[T](items: list[T], key: Callable[[T], int]) -> None:
    """Not stable: swapping across a distance can jump over equal items."""
    for start in range(len(items) - 1):
        smallest = start
        for i in range(start + 1, len(items)):
            if key(items[i]) < key(items[smallest]):
                smallest = i
        items[start], items[smallest] = items[smallest], items[start]


stable = [("b", 2), ("a", 1), ("c", 2), ("d", 1)]
insertion_sort_by(stable, key=lambda pair: pair[1])
print(stable)  # [('a', 1), ('d', 1), ('b', 2), ('c', 2)], equal keys keep their order

unstable = [("a", 2), ("b", 2), ("c", 1)]
selection_sort_by(unstable, key=lambda pair: pair[1])
print(unstable)  # [('c', 1), ('b', 2), ('a', 2)], the two 2s traded places
```

## Deeper behavior

### The loop invariants

Each algorithm maintains a statement that is true after every pass.

- Bubble sort: after pass `k`, the last `k` items are the `k` largest items, in sorted order and in their final positions.
- Selection sort: after pass `k`, the first `k` items are the `k` smallest items in sorted order and in their final positions.
- Insertion sort: after step `i`, the first `i + 1` items are the original first `i + 1` items in sorted order. They are not necessarily in their final positions.

When the loop ends, the invariant covers the whole list, so the list is sorted. Writing the invariant in a comment is the best way to avoid off-by-one errors.

### Inversions explain insertion sort

An inversion is a pair of positions `i < j` with `values[i] > values[j]`. Each shift in insertion sort removes exactly one inversion, so the number of moves equals the number of inversions in the input. A sorted list has none, a reversed list of `n` items has `n(n - 1) / 2`, and a random list has about half of that, `n(n - 1) / 4`, on average. That is why insertion sort costs `O(n + k)` for a list with `k` inversions, and why it is excellent for data that is already almost sorted.

```python
def count_inversions(values: list[int]) -> int:
    """Brute force: O(n^2). A faster method appears with divide and conquer."""
    return sum(
        1
        for i in range(len(values))
        for j in range(i + 1, len(values))
        if values[i] > values[j]
    )


print(count_inversions([3, 1, 2, 5, 4]))  # 3
print(insertion_sort([3, 1, 2, 5, 4]))  # SortStats(comparisons=6, moves=3)
```

The three moves equal the three inversions: `(3, 1)`, `(3, 2)`, and `(5, 4)`.

### Complexity summary

- Bubble sort: best `O(n)` (with the early-exit check), average and worst `O(n^2)`, space `O(1)`, stable.
- Selection sort: best, average, and worst `O(n^2)` comparisons, at most `n - 1` swaps, space `O(1)`, not stable.
- Insertion sort: best `O(n)`, average and worst `O(n^2)`, space `O(1)`, stable.

Without the early-exit flag, bubble sort does `O(n^2)` comparisons on every input, and the flag is the only thing that makes it adaptive.

### Why selection sort is not stable

The swap moves the minimum across many items in one step. In `[("a", 2), ("b", 2), ("c", 1)]`, the first pass swaps `("c", 1)` with `("a", 2)`, which moves `a` to the far end, behind `b`. The relative order of the two equal items has changed. Selection sort can be made stable by shifting items instead of swapping, but then it makes `O(n)` writes per pass, and it is simpler to use insertion sort.

### When fewer moves matter

When writing an item is much more expensive than comparing two items, for example sorting records on flash storage or large objects that must be physically moved, selection sort's `n - 1` swaps is a real advantage. Python lists only move references, so writes are cheap, and this rarely matters in practice.

### Binary insertion sort

Insertion sort finds the right position by scanning. Because the prefix is already sorted, binary search can find it with `O(log n)` comparisons. The shifting still costs `O(n)`, so the total is still `O(n^2)` moves. Timsort uses this idea for short runs, because comparisons of Python objects can be expensive while memory moves are cheap.

### Hybrid sorts use insertion sort for small pieces

Efficient sorts such as merge sort and quicksort recurse down to very small subarrays. Below a threshold (often 16 to 64 items), the recursion overhead costs more than the quadratic work saved, so they finish with insertion sort. For tiny inputs the "inefficient" algorithm is the fastest one.

### Online sorting

Insertion sort can accept items one at a time and keep the list sorted after each, which makes it suitable for streams. Each new item costs `O(k)` where `k` is how far it must travel. If items arrive nearly in order, `k` is small.

### Never use these on large inputs

For `n = 100,000`, a quadratic sort does about ten billion comparisons. In Python that takes minutes, while `sorted` finishes in a fraction of a second. Learn these algorithms to reason about sorting. Use `sorted` and `list.sort` in real code.

## Industry-level snippet

A service collects events from several machines whose clocks and network delays differ slightly. Events usually arrive in timestamp order, but a few arrive a little late. The database wants them stored in timestamp order, with events that share a timestamp kept in arrival order.

Because the data is nearly sorted, insertion sort is a good fit: it costs `O(n + inversions)`, so a batch with only a few late events is processed in nearly linear time, and it is stable. The sort also doubles as a data quality check. An event that has to move many positions is not slightly late, it is suspicious, and the batch is rejected with a clear error. The limit also bounds the work: no event can cause more than `max_shift` shifts.

The ordered batch is inserted in one transaction, so a failure leaves no partial batch in the database.

```python
# event_reorder.py
import logging
import sqlite3
from collections.abc import Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS event (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    source TEXT NOT NULL,
    payload TEXT NOT NULL
)
"""

INSERT_SQL = "INSERT INTO event (timestamp, source, payload) VALUES (?, ?, ?)"


class TooFarOutOfOrderError(ValueError):
    """Raised when an event arrives later than the allowed number of positions."""


@dataclass(frozen=True, slots=True)
class Event:
    timestamp: int
    source: str
    payload: str


@dataclass(frozen=True, slots=True)
class ReorderStats:
    comparisons: int
    shifts: int
    largest_shift: int


def reorder_events(
    events: Sequence[Event], max_shift: int
) -> tuple[list[Event], ReorderStats]:
    """Stable insertion sort by timestamp.

    Cost is O(n + inversions) and no event moves more than max_shift positions.
    """
    if max_shift < 0:
        raise ValueError("max_shift must not be negative")
    ordered = list(events)  # never modify the caller's sequence
    comparisons = shifts = largest_shift = 0
    for i in range(1, len(ordered)):
        current = ordered[i]
        j = i - 1
        while j >= 0:
            comparisons += 1
            if ordered[j].timestamp <= current.timestamp:  # <= keeps ties in order
                break
            if i - j > max_shift:
                raise TooFarOutOfOrderError(
                    f"event at index {i} would move more than {max_shift} positions"
                )
            ordered[j + 1] = ordered[j]
            j -= 1
        distance = i - (j + 1)
        shifts += distance
        largest_shift = max(largest_shift, distance)
        ordered[j + 1] = current
    return ordered, ReorderStats(comparisons, shifts, largest_shift)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def store_batch(
    connection: sqlite3.Connection, events: Sequence[Event], max_shift: int = 50
) -> ReorderStats:
    """Reorder a batch and store it atomically. Rows get ids in timestamp order."""
    ordered, stats = reorder_events(events, max_shift)
    rows = [(event.timestamp, event.source, event.payload) for event in ordered]
    try:
        with connection:
            connection.executemany(INSERT_SQL, rows)
    except sqlite3.Error:
        logger.exception("could not store a batch of %d events", len(rows))
        raise
    logger.info(
        "stored %d events (%d shifts, largest %d)",
        len(rows),
        stats.shifts,
        stats.largest_shift,
    )
    return stats
```

```python
# test_event_reorder.py
import random
import sqlite3
from collections.abc import Iterator

import pytest

from event_reorder import (
    Event,
    TooFarOutOfOrderError,
    create_schema,
    reorder_events,
    store_batch,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def make(timestamp: int, source: str = "a") -> Event:
    return Event(timestamp, source, f"payload-{timestamp}-{source}")


def test_nearly_sorted_input_is_repaired_cheaply() -> None:
    events = [make(1), make(2), make(4), make(3), make(5), make(6)]
    ordered, stats = reorder_events(events, max_shift=3)
    assert [event.timestamp for event in ordered] == [1, 2, 3, 4, 5, 6]
    assert stats.shifts == 1
    assert stats.largest_shift == 1


def test_sorted_input_needs_one_comparison_per_item() -> None:
    events = [make(stamp) for stamp in range(10)]
    ordered, stats = reorder_events(events, max_shift=0)
    assert ordered == events
    assert stats.comparisons == 9
    assert stats.shifts == 0


def test_equal_timestamps_keep_their_arrival_order() -> None:
    events = [make(5, "first"), make(3, "x"), make(5, "second"), make(5, "third")]
    ordered, _ = reorder_events(events, max_shift=10)
    assert [event.source for event in ordered if event.timestamp == 5] == [
        "first",
        "second",
        "third",
    ]


def test_input_is_not_modified() -> None:
    events = [make(3), make(1), make(2)]
    snapshot = list(events)
    reorder_events(events, max_shift=5)
    assert events == snapshot


def test_events_that_are_too_late_are_rejected() -> None:
    events = [make(10), make(11), make(12), make(13), make(1)]
    with pytest.raises(TooFarOutOfOrderError):
        reorder_events(events, max_shift=3)
    ordered, _ = reorder_events(events, max_shift=4)
    assert ordered[0].timestamp == 1


def test_negative_limit_is_rejected() -> None:
    with pytest.raises(ValueError):
        reorder_events([], max_shift=-1)


def test_random_batches_match_the_stable_builtin_sort() -> None:
    rng = random.Random(5)
    for _ in range(300):
        events = [
            make(rng.randint(0, 8), str(index))
            for index in range(rng.randint(0, 25))
        ]
        ordered, _ = reorder_events(events, max_shift=10**6)
        assert ordered == sorted(events, key=lambda event: event.timestamp)


def test_store_batch_writes_rows_in_timestamp_order(
    connection: sqlite3.Connection,
) -> None:
    store_batch(connection, [make(2), make(1), make(3)])
    rows = connection.execute("SELECT timestamp FROM event ORDER BY id").fetchall()
    assert [row[0] for row in rows] == [1, 2, 3]


def test_rejected_batch_stores_nothing(connection: sqlite3.Connection) -> None:
    events = [make(stamp) for stamp in range(10, 20)] + [make(1)]
    with pytest.raises(TooFarOutOfOrderError):
        store_batch(connection, events, max_shift=3)
    count = connection.execute("SELECT COUNT(*) FROM event").fetchone()[0]
    assert count == 0
```

The reorder step runs before the transaction begins, so a rejected batch never touches the database at all.

## Common mistakes

- Using `>=` instead of `>` when shifting in insertion sort. It moves equal items past each other and breaks stability.
- Forgetting the early-exit flag in bubble sort, which makes the best case `O(n^2)` instead of `O(n)`.
- Running the inner loop of bubble sort over the whole list each pass instead of shrinking it, which wastes comparisons on the already sorted tail.
- Off-by-one errors in the insertion loop: starting `i` at 0 instead of 1, or writing `j > 0` where `j >= 0` is needed, which leaves the first item unreachable.
- Assigning the result of an in-place sort. `values = values.sort()` sets `values` to `None`.
- Assuming selection sort is stable, then sorting by two keys and getting scrambled ties.
- Calling a quadratic sort on large data in production instead of `sorted`.
- Measuring sorting speed on already sorted or reversed input only. Adaptive algorithms look much better or worse depending on the input, so test several shapes.
- Mixing up comparisons and moves when you compare algorithms, then drawing the wrong conclusion about which one is "faster".
- Sorting a list while iterating over it with a `for` loop.

## Practice exercises

1. Trace bubble sort, selection sort, and insertion sort by hand on `[5, 2, 4, 6, 1, 3]`. Write the list after each pass.
2. Add a `SortStats` result to a sort that works on strings, and compare the three algorithms on 1,000 random words, on sorted words, and on reversed words.
3. Change `bubble_sort` to alternate directions on each pass (cocktail shaker sort). Does it reduce comparisons on `[2, 3, 4, 5, 1]`?
4. Write `is_sorted(values)` and use it in a test that checks all three sorts on 500 random lists generated with a seeded `random.Random`.
5. Write a stable version of selection sort that shifts items instead of swapping them, and test its stability with records that share a key.
6. Write `binary_insertion_sort(values)` using `bisect.bisect_right` to find the insertion point. Compare its number of comparisons with `insertion_sort`.
7. Sort a list of `(name, age)` records by age and then by name, using two passes of a stable sort in the right order. Explain why the order of the passes matters.
8. Write `count_inversions(values)` and verify that `insertion_sort`'s move count equals it on 200 random lists.
9. Predict, then measure, how many comparisons `insertion_sort` needs on a list that is sorted except for one item that is out of place by 5 positions.
10. Write `merge_two_nearly_sorted_streams` that takes two sorted lists with a few late items appended and produces one sorted list using insertion sort. Explain when this is better than `sorted`.
11. Time `insertion_sort` and `sorted` on lists of 10, 100, 1,000, and 10,000 random integers. At what size does `sorted` win, and why might it win even at small sizes?
12. Implement shell sort with a gap sequence of your choice and compare its comparison counts with insertion sort on random lists of 2,000 items.
13. Extend `reorder_events` to accept a `key` function so it can sort any record type, and add tests for stability with a custom key.

## Summary

Bubble, selection, and insertion sort are simple, in-place, quadratic comparison sorts. Bubble sort swaps adjacent out-of-order pairs and can stop early, selection sort picks the minimum each pass and makes at most `n - 1` swaps, and insertion sort slides each item into a sorted prefix. Bubble and insertion sort are stable, and selection sort is not. Insertion sort is adaptive: it costs `O(n + inversions)`, which makes it the best choice for tiny or nearly sorted data, and it is the building block that efficient sorts use for small pieces. Counting comparisons and moves gives clean evidence about how algorithms behave on different inputs. In real code, use `sorted` and `list.sort`.
