## Overview

An array is a row of items stored one after another in memory. Because the items are side by side and the same size, the computer can jump straight to item number `i` with one multiplication and one addition, no matter how long the array is. That single property, constant-time indexing, is why arrays are the most used data structure in programming.

Python's everyday array is the `list`. It is a dynamic array: it can grow and shrink while the program runs, even though the memory underneath has to be reserved in advance. This note shows how that works, what each operation costs, and how to write algorithms that modify an array in place without allocating a second one. You will implement a small dynamic array yourself, then meet the compact alternatives `array` and `bytearray`.

Nearly every later topic in the roadmap builds on arrays: strings, hash tables, heaps, dynamic programming tables, and grids all sit on top of them.

## Core concepts

### Contiguous memory and constant-time indexing

If an array starts at memory address `base` and each item takes `size` bytes, then item `i` lives at `base + i * size`. The computer does not search for it. That is why reading or writing `values[i]` is `O(1)` regardless of `n`.

### How Python lists store references

A Python `list` does not store the numbers or strings themselves. It stores references (8-byte pointers on a 64-bit machine) to objects that live elsewhere. The array of pointers is contiguous, so indexing is still `O(1)`. Because every slot has the same size, a list can hold an `int`, a `str`, and another list side by side.

- Good: flexible, and sorting or swapping moves only pointers.
- Cost: every number is a separate object, so a list of a million integers uses far more memory than a million raw numbers.

### Dynamic arrays: size and capacity

A dynamic array tracks two numbers.

- Size: how many items are stored.
- Capacity: how many items fit in the memory currently reserved.

Appending is cheap while `size < capacity`. When the array is full, it allocates a larger block, copies every item across, and frees the old block. If the new capacity is the old capacity times a constant growth factor, the occasional expensive copy is spread over many cheap appends, so `append` is `O(1)` amortized.

### Cost of insertion and deletion

- Append or pop at the end: `O(1)` (amortized for append).
- Insert or delete at index `i`: every item after `i` must shift by one slot, so the cost is `O(n - i)`. At the front it is `O(n)`.
- Reading or overwriting an existing slot: `O(1)`.

### In-place versus extra-space algorithms

- An in-place algorithm changes the input array using only a constant amount of extra memory, `O(1)` auxiliary space.
- An extra-space algorithm builds a new array of size proportional to the input, `O(n)` auxiliary space.

In-place is not automatically better. It saves memory but it destroys the original data, and it often makes the code harder to read. Choose it when memory matters or the problem demands it.

### Common in-place techniques

- Reversing: swap the two ends and walk inward.
- Rotating: three reversals rotate an array by `k` positions in `O(n)` time and `O(1)` space.
- Removing duplicates from a sorted array: one pointer reads, one pointer writes.
- Filtering (for example moving zeros to the end): a write pointer marks where the next kept item goes.

### Compact arrays: array and bytearray

- `array.array` stores raw numbers of one fixed type (for example 4-byte integers or 8-byte floats) without a Python object per item.
- `bytearray` is a mutable sequence of bytes, each in the range 0 to 255. It is the mutable cousin of `bytes`.

## Basic syntax examples

### Everyday list operations

```python
numbers: list[int] = [10, 20, 30, 40, 50]

print(numbers[0], numbers[-1])  # 10 50, negative indexes count from the end
print(numbers[1:4])  # [20, 30, 40], slices copy and exclude the stop index
print(numbers[::-1])  # [50, 40, 30, 20, 10], a reversed copy

numbers.append(60)  # O(1) amortized
numbers.insert(2, 25)  # O(n): shifts 30, 40, 50, 60 right
last = numbers.pop()  # O(1)
second = numbers.pop(1)  # O(n): shifts everything after index 1 left
print(numbers, last, second)  # [10, 25, 30, 40, 50] 60 20
```

### Reversing in place

```python
def reverse_in_place(values: list[int]) -> None:
    """O(n) time, O(1) extra space."""
    left, right = 0, len(values) - 1
    while left < right:
        values[left], values[right] = values[right], values[left]
        left += 1
        right -= 1


data = [1, 2, 3, 4, 5]
reverse_in_place(data)
print(data)  # [5, 4, 3, 2, 1]
```

### Rotating with three reversals

Rotating `[1, 2, 3, 4, 5, 6, 7]` right by 3 gives `[5, 6, 7, 1, 2, 3, 4]`. Reverse everything, then reverse the first `k` items, then reverse the rest.

```python
def reverse_range(values: list[int], left: int, right: int) -> None:
    while left < right:
        values[left], values[right] = values[right], values[left]
        left += 1
        right -= 1


def rotate_right(values: list[int], k: int) -> None:
    """Rotate in place by k steps to the right. O(n) time, O(1) extra space."""
    n = len(values)
    if n == 0:
        return
    k %= n  # rotating by n gives the same array, so only k mod n matters
    reverse_range(values, 0, n - 1)
    reverse_range(values, 0, k - 1)
    reverse_range(values, k, n - 1)


data = [1, 2, 3, 4, 5, 6, 7]
rotate_right(data, 3)
print(data)  # [5, 6, 7, 1, 2, 3, 4]
```

Trace of the three steps:

```text
start:                     [1, 2, 3, 4, 5, 6, 7]
reverse everything:        [7, 6, 5, 4, 3, 2, 1]
reverse first k = 3 items: [5, 6, 7, 4, 3, 2, 1]
reverse the remaining 4:   [5, 6, 7, 1, 2, 3, 4]
```

### Removing duplicates from a sorted array in place

```python
def remove_duplicates_sorted(values: list[int]) -> int:
    """Keep one copy of each value. Returns the new length.

    After the call, values[:length] holds the unique values in order.
    One pointer reads every item, another marks where the next unique item goes.
    """
    if not values:
        return 0
    write = 1
    for read in range(1, len(values)):
        if values[read] != values[write - 1]:
            values[write] = values[read]
            write += 1
    return write


data = [1, 1, 2, 2, 2, 3, 5, 5]
length = remove_duplicates_sorted(data)
print(data[:length])  # [1, 2, 3, 5]
```

### Moving zeros to the end

```python
def move_zeros_to_end(values: list[int]) -> None:
    """Keep the order of non-zero items. O(n) time, O(1) extra space."""
    write = 0
    for read in range(len(values)):
        if values[read] != 0:
            values[write], values[read] = values[read], values[write]
            write += 1


data = [0, 1, 0, 3, 12]
move_zeros_to_end(data)
print(data)  # [1, 3, 12, 0, 0]
```

### array and bytearray

```python
from array import array

temperatures = array("d", [21.5, 22.0, 19.75])  # "d" means 8-byte float
temperatures.append(23.25)
print(len(temperatures), temperatures.itemsize)  # 4 8
print(temperatures[1])  # 22.0

buffer = bytearray(b"abc")
buffer[0] = ord("X")  # bytearray items are integers from 0 to 255
buffer.extend(b"!")
print(buffer)  # bytearray(b'Xbc!')
```

## Deeper behavior

### Implementing a dynamic array

The class below behaves like a small version of `list`. Python lists already do all of this in C, so the class exists to show the mechanism. It uses a plain list filled with `None` as the fixed block of memory, and it never lets the list grow by itself. It also counts element copies, so you can verify the amortized cost.

```python
from collections.abc import Iterator
from typing import cast


class DynamicArray[T]:
    """A teaching version of a dynamic array with an explicit capacity."""

    def __init__(self, growth_factor: int = 2) -> None:
        if growth_factor < 2:
            raise ValueError("growth_factor must be at least 2")
        self._growth_factor = growth_factor
        self._capacity = 4
        self._size = 0
        self._slots: list[T | None] = [None] * self._capacity
        self.copies = 0  # total number of items copied during resizes

    def __len__(self) -> int:
        return self._size

    @property
    def capacity(self) -> int:
        return self._capacity

    def _normalize(self, index: int) -> int:
        """Turn a negative index into a positive one and check the bounds."""
        if index < 0:
            index += self._size
        if not 0 <= index < self._size:
            raise IndexError("index out of range")
        return index

    def __getitem__(self, index: int) -> T:
        return cast(T, self._slots[self._normalize(index)])

    def __setitem__(self, index: int, value: T) -> None:
        self._slots[self._normalize(index)] = value

    def append(self, value: T) -> None:
        if self._size == self._capacity:
            self._resize(self._capacity * self._growth_factor)
        self._slots[self._size] = value
        self._size += 1

    def insert(self, index: int, value: T) -> None:
        if not 0 <= index <= self._size:
            raise IndexError("insert position out of range")
        if self._size == self._capacity:
            self._resize(self._capacity * self._growth_factor)
        for i in range(self._size, index, -1):  # shift right: O(n - index)
            self._slots[i] = self._slots[i - 1]
        self._slots[index] = value
        self._size += 1

    def pop(self) -> T:
        if self._size == 0:
            raise IndexError("pop from empty array")
        self._size -= 1
        value = cast(T, self._slots[self._size])
        self._slots[self._size] = None  # release the reference
        if self._capacity > 4 and self._size <= self._capacity // 4:
            self._resize(self._capacity // 2)  # shrink, but not too eagerly
        return value

    def _resize(self, new_capacity: int) -> None:
        new_slots: list[T | None] = [None] * new_capacity
        for i in range(self._size):
            new_slots[i] = self._slots[i]
            self.copies += 1
        self._slots = new_slots
        self._capacity = new_capacity

    def __iter__(self) -> Iterator[T]:
        for i in range(self._size):
            yield cast(T, self._slots[i])


array_of_numbers = DynamicArray[int]()
for number in range(1_000):
    array_of_numbers.append(number)
print(len(array_of_numbers), array_of_numbers.capacity, array_of_numbers.copies)
# 1000 1024 1020
```

One thousand appends triggered 1,020 element copies in total, about one copy per append. That is the amortized `O(1)`.

### Why amortized append is O(1)

With a growth factor of 2, resizes happen at sizes 4, 8, 16, and so on up to `n`. The copy work is `4 + 8 + 16 + ... + n/2`, which is less than `n`. Add the `n` ordinary writes and the total is below `2n` operations for `n` appends, so each append costs a constant number of operations on average. The argument works for any constant factor greater than 1. If instead the capacity grew by a fixed amount such as 10 each time, the copies would add up to roughly `n^2 / 20`, and appends would be `O(n)` on average.

CPython grows by about 12.5 percent plus a small constant rather than doubling. That wastes less memory and is still amortized `O(1)`.

The shrink rule in `pop` matters too. If the array shrank as soon as it was half full, alternating `append` and `pop` at the boundary would resize on every call. Shrinking only when the array is a quarter full avoids that.

### Aliasing and copying

Assigning a list to another name does not copy it. Both names point at the same list.

```python
original = [1, 2, 3]
alias = original  # same list
copy_of = original[:]  # new list with the same items (a shallow copy)
alias.append(4)
print(original, copy_of)  # [1, 2, 3, 4] [1, 2, 3]
```

A shallow copy copies the references only. If the items are themselves lists, the inner lists are shared between the copies.

### Slice assignment and deletion

Slices on the left side of an assignment can change the length of a list. The cost is proportional to the number of items moved.

```python
values = [0, 1, 2, 3, 4, 5]
values[1:3] = [9]  # replaces two items by one
print(values)  # [0, 9, 3, 4, 5]
del values[1:3]  # removes two items
print(values)  # [0, 4, 5]
```

### Changing a list while iterating over it

Removing items from a list inside a `for` loop over the same list skips elements, because the later items shift left under the iterator.

```python
data = [1, 2, 2, 3]
for value in data:
    if value == 2:
        data.remove(value)  # skips the second 2
print(data)  # [1, 2, 3]
```

Build a new list with a comprehension, or use a write pointer as in `move_zeros_to_end`.

### Memory: list versus array

A list of one million distinct integers stores 8 million bytes of pointers and, on a typical 64-bit CPython, about 28 million more bytes in the integer objects. An `array("i")` holding the same numbers uses about 4 million bytes. The trade-off is that an `array` holds only one numeric type and cannot hold values outside that type's range:

```python
from array import array

small = array("b", [100])  # "b" is a signed 1-byte integer, range -128 to 127
try:
    small.append(200)
except OverflowError as error:
    print(error)  # signed char is greater than maximum
```

The common type codes are `"b"` and `"B"` (1 byte), `"h"` and `"H"` (2 bytes), `"i"` and `"I"` (usually 4 bytes), `"q"` and `"Q"` (8 bytes), `"f"` (4-byte float), and `"d"` (8-byte float). Indexing an `array` still creates a temporary Python object, so reading from it is not faster than from a list. The saving is memory.

### In-place versus copying built-ins

- `values.reverse()` and `values.sort()` change the list in place and return `None`.
- `reversed(values)` returns a lazy iterator. `sorted(values)` returns a new list.
- A common bug is `values = values.sort()`, which sets `values` to `None`.

## Industry-level snippet

A service records how long each request to an endpoint takes. Storing every sample as a Python `float` object would waste memory, so samples are kept in an `array("d")`, a compact array of raw 8-byte floats. When the buffer fills, the service summarizes the samples (count, minimum, maximum, mean, 95th percentile) and writes one summary row to the database in a transaction.

Notice the error handling. Invalid samples are rejected before they reach the array. If the database write fails, the samples stay in the buffer, so no data is lost and the next flush can retry. The buffer is cleared only after the commit succeeds.

```python
# latency_buffer.py
import logging
import math
import sqlite3
from array import array
from collections.abc import Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS latency_summary (
    endpoint TEXT NOT NULL,
    sample_count INTEGER NOT NULL,
    minimum_ms REAL NOT NULL,
    maximum_ms REAL NOT NULL,
    mean_ms REAL NOT NULL,
    p95_ms REAL NOT NULL
)
"""

INSERT_SQL = (
    "INSERT INTO latency_summary "
    "(endpoint, sample_count, minimum_ms, maximum_ms, mean_ms, p95_ms) "
    "VALUES (?, ?, ?, ?, ?, ?)"
)


class InvalidSampleError(ValueError):
    """Raised for a latency that is negative, infinite, or not a number."""


@dataclass(frozen=True, slots=True)
class Summary:
    count: int
    minimum: float
    maximum: float
    mean: float
    p95: float


def summarize(samples: Sequence[float]) -> Summary:
    """Nearest-rank percentile: sort once, O(n log n)."""
    if not samples:
        raise ValueError("cannot summarize zero samples")
    ordered = sorted(samples)
    rank = math.ceil(0.95 * len(ordered))
    return Summary(
        count=len(ordered),
        minimum=ordered[0],
        maximum=ordered[-1],
        mean=sum(ordered) / len(ordered),
        p95=ordered[rank - 1],
    )


class LatencyBuffer:
    """Collect samples for one endpoint in a compact array and flush summaries."""

    def __init__(
        self, connection: sqlite3.Connection, endpoint: str, capacity: int = 1_000
    ) -> None:
        if not endpoint:
            raise ValueError("endpoint must not be empty")
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        self._connection = connection
        self._endpoint = endpoint
        self._capacity = capacity
        self._samples = array("d")

    def __len__(self) -> int:
        return len(self._samples)

    def record(self, milliseconds: float) -> None:
        if not math.isfinite(milliseconds) or milliseconds < 0:
            raise InvalidSampleError(f"invalid latency: {milliseconds!r}")
        self._samples.append(milliseconds)
        if len(self._samples) >= self._capacity:
            self.flush()

    def flush(self) -> None:
        """Write one summary row. Samples are kept if the write fails."""
        if not self._samples:
            return
        summary = summarize(self._samples)
        try:
            with self._connection:
                self._connection.execute(
                    INSERT_SQL,
                    (
                        self._endpoint,
                        summary.count,
                        summary.minimum,
                        summary.maximum,
                        summary.mean,
                        summary.p95,
                    ),
                )
        except sqlite3.Error:
            logger.exception(
                "could not flush %d samples for %s; keeping them",
                len(self._samples),
                self._endpoint,
            )
            raise
        logger.info("flushed %d samples for %s", summary.count, self._endpoint)
        del self._samples[:]  # clear in place only after the commit succeeded
```

```python
# test_latency_buffer.py
import sqlite3
from collections.abc import Iterator

import pytest

from latency_buffer import (
    SCHEMA_SQL,
    InvalidSampleError,
    LatencyBuffer,
    summarize,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA_SQL)
    yield conn
    conn.close()


def row_count(connection: sqlite3.Connection) -> int:
    row = connection.execute("SELECT COUNT(*) FROM latency_summary").fetchone()
    return int(row[0])


def test_summarize_uses_nearest_rank_percentile() -> None:
    summary = summarize([float(value) for value in range(1, 101)])
    assert summary.count == 100
    assert summary.minimum == 1.0
    assert summary.maximum == 100.0
    assert summary.mean == pytest.approx(50.5)
    assert summary.p95 == 95.0


def test_summarize_rejects_empty_input() -> None:
    with pytest.raises(ValueError):
        summarize([])


def test_buffer_flushes_automatically_when_full(
    connection: sqlite3.Connection,
) -> None:
    buffer = LatencyBuffer(connection, "/api/users", capacity=3)
    buffer.record(10.0)
    buffer.record(20.0)
    assert row_count(connection) == 0
    buffer.record(30.0)
    assert row_count(connection) == 1
    assert len(buffer) == 0


@pytest.mark.parametrize("bad", [-1.0, float("nan"), float("inf")])
def test_invalid_samples_are_rejected(
    connection: sqlite3.Connection, bad: float
) -> None:
    buffer = LatencyBuffer(connection, "/api/users")
    with pytest.raises(InvalidSampleError):
        buffer.record(bad)
    assert len(buffer) == 0


def test_failed_flush_keeps_the_samples() -> None:
    broken = sqlite3.connect(":memory:")  # the table does not exist
    buffer = LatencyBuffer(broken, "/api/users", capacity=10)
    buffer.record(5.0)
    buffer.record(7.0)
    with pytest.raises(sqlite3.OperationalError):
        buffer.flush()
    assert len(buffer) == 2


def test_constructor_validates_arguments(connection: sqlite3.Connection) -> None:
    with pytest.raises(ValueError):
        LatencyBuffer(connection, "")
    with pytest.raises(ValueError):
        LatencyBuffer(connection, "/api/users", capacity=0)
```

## Common mistakes

- Using `values.insert(0, x)` or `values.pop(0)` in a loop. Each call is `O(n)`, so the loop is quadratic. Use `collections.deque`.
- Writing `values = values.sort()` or `values = values.reverse()`. These methods return `None`.
- Modifying a list while looping over it. Loop over a copy, build a new list, or use a write pointer.
- Believing `b = a` copies a list. It creates a second name for the same list.
- Forgetting that slices copy. `values[1:]` inside a loop or a recursive function turns linear code into quadratic code.
- Confusing a shallow copy with a deep copy when a list contains lists.
- Rotating by `k` without reducing `k` modulo the length. Rotating by 10 a list of 3 should behave like rotating by 1.
- Off-by-one errors with the exclusive stop in `range` and slices. `values[a:b]` contains indexes `a` to `b - 1`.
- Reading a negative index by accident. `values[-1]` is valid, so an index that goes below zero in a loop can silently read the wrong item instead of crashing.
- Choosing `array` expecting faster reads. It saves memory, not time, and it rejects values outside the type's range.

## Practice exercises

1. Write `reverse_in_place` for a list of strings and test it on an empty list, a one-element list, and an even-length list.
2. Rotate a list left by `k` positions in place using three reversals. Verify it against `values[k:] + values[:k]`.
3. Write `remove_element(values, target)` that removes every occurrence of `target` in place and returns the new length. Do not create another list.
4. Given a sorted list, write `remove_duplicates_allow_two(values)` that keeps at most two copies of each value, in place, and returns the new length.
5. Write `merge_into_first(first, m, second, n)` where `first` has `m` real items followed by `n` spare slots, and `second` has `n` items. Both are sorted. Merge `second` into `first` in place by filling from the back.
6. Write `partition_even_odd(values)` that moves all even numbers before all odd numbers in place, using two pointers.
7. Add `__repr__`, `extend`, and `clear` methods to `DynamicArray`, and write tests for them.
8. Change `DynamicArray` to grow by a fixed amount of 10 instead of multiplying. Append 100,000 items and compare `copies` with the doubling version. Explain the result.
9. Add `remove_at(index)` to `DynamicArray` that shifts items left and shrinks when the array is a quarter full. State its time complexity.
10. Use `sys.getsizeof` to compare a list and an `array("i")` of one million integers, remembering that `getsizeof` does not count the objects a list refers to. Estimate the real memory of the list.
11. Write `find_missing(values)` for a list holding `n` distinct numbers from `0` to `n`, with one number missing. Solve it in `O(n)` time and `O(1)` extra space with the sum formula, then again with XOR.
12. Explain why `array("b")` raises `OverflowError` for `200` but `list` does not, and give one situation where each behavior is the better one.
13. Extend `LatencyBuffer` with a `flush_if_older_than(seconds)` method driven by an injectable clock function, and test it without calling `time.sleep`.

## Summary

An array stores equal-sized items contiguously, so indexing is `O(1)`. A Python `list` is a dynamic array of references: appending is amortized `O(1)` because capacity grows by a constant factor, while inserting or deleting away from the end shifts items and costs `O(n)`. In-place techniques such as swapping from both ends, three-reversal rotation, and read-write pointers solve many problems in `O(1)` extra space. Copies, aliasing, slices, and modifying a list during iteration are the common traps. `array` and `bytearray` store raw values compactly, which saves memory but limits the item type and range.
