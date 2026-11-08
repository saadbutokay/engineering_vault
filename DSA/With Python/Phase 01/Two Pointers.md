## Overview

Many array and string problems look like they need two nested loops: compare every pair, check every sub-range, test every combination. The two-pointers technique replaces the nested loops with two index variables that move through the data in a controlled way. Because each pointer only moves forward (or only inward), the total number of steps is linear.

The technique has three common shapes. Opposite-end pointers start at both ends and move toward each other, which suits sorted data and symmetric problems such as palindromes. Same-direction pointers, often called a slow pointer and a fast pointer, or a read pointer and a write pointer, move left to right at different speeds, which suits filtering and removing items in place. Partitioning pointers divide an array into regions as they sweep.

This note covers opposite-end pointers, same-direction pointers, partitioning arrays, merging sorted arrays, Three Sum, container with most water, in-place filtering, and how to tell when the technique applies. The key habit is to ask, for every pointer move, "why is it safe to discard everything I skip?"

## Core concepts

### Opposite-end pointers

Place `left` at the start and `right` at the end. At each step, look at the two items, decide which pointer to move, and shrink the search range. The loop ends when the pointers meet. This works when the data has an order (sorted values) or a symmetry (a palindrome), because the order lets you prove that one side can be discarded.

### Same-direction pointers

Both pointers start near the beginning and move forward.

- Read and write pointers: the read pointer scans every item, and the write pointer marks where the next kept item goes. Used for removing duplicates and filtering in place.
- Slow and fast pointers: the fast pointer moves further per step than the slow one, which finds middles and detects cycles. This pattern returns with linked lists.

### Partitioning

Partitioning rearranges an array so that items satisfying a rule come first. One pointer marks the boundary of the finished region, and another scans. A three-way partition keeps three regions (for example 0s, 1s, and 2s), which is the Dutch national flag problem and the core of quicksort partitioning.

### Merging sorted arrays

To merge two sorted arrays, keep one pointer in each and repeatedly take the smaller front item. Each item is taken once, so merging takes `O(n + m)`. This is the merge step of merge sort and of many reconciliation tasks.

### Three Sum

Find all unique triples that add up to zero. Sort the array, fix one item, and run an opposite-end search for the other two. It turns `O(n^3)` into `O(n^2)`.

### Container with most water

Given vertical lines of various heights, find the two lines that together with the x-axis hold the most water. Start with the widest container and move the pointer at the shorter line inward.

### When the technique applies

- The input is sorted, or you are willing to sort it first.
- The condition you test changes in a predictable direction when a pointer moves (a sum gets larger as the left pointer advances in a sorted array).
- You need to modify the array in place with `O(1)` extra space.
- The structure is symmetric, as in palindromes.

When none of these hold, such as finding a pair in an unsorted array without sorting, a hash table is usually the better tool.

## Basic syntax examples

### Two Sum in a sorted array

```python
def two_sum_sorted(numbers: list[int], target: int) -> tuple[int, int] | None:
    """Return indexes of two items adding up to target. O(n) time, O(1) space."""
    left, right = 0, len(numbers) - 1
    while left < right:
        total = numbers[left] + numbers[right]
        if total == target:
            return left, right
        if total < target:
            left += 1  # we need a bigger sum, so move to a bigger left item
        else:
            right -= 1  # we need a smaller sum, so move to a smaller right item
    return None


print(two_sum_sorted([1, 3, 4, 6, 8, 11], 10))  # (2, 3), which is 4 + 6
```

Trace of `two_sum_sorted([1, 3, 4, 6, 8, 11], 10)`:

```text
left=0 (1), right=5 (11): total 12 > 10, move right left
left=0 (1), right=4 (8):  total 9  < 10, move left right
left=1 (3), right=4 (8):  total 11 > 10, move right left
left=1 (3), right=3 (6):  total 9  < 10, move left right
left=2 (4), right=3 (6):  total 10 == 10, found (2, 3)
```

### Removing duplicates with read and write pointers

```python
def remove_duplicates(values: list[int]) -> int:
    """Remove duplicates from a sorted list in place. Returns the new length."""
    if not values:
        return 0
    write = 1
    for read in range(1, len(values)):
        if values[read] != values[write - 1]:
            values[write] = values[read]
            write += 1
    return write


data = [0, 0, 1, 1, 1, 2, 2, 3]
length = remove_duplicates(data)
print(data[:length])  # [0, 1, 2, 3]
```

### Removing a value in place

```python
def remove_value(values: list[int], target: int) -> int:
    """Remove every occurrence of target in place. Returns the new length."""
    write = 0
    for read in range(len(values)):
        if values[read] != target:
            values[write] = values[read]
            write += 1
    return write


data = [3, 2, 2, 3, 4]
length = remove_value(data, 3)
print(data[:length])  # [2, 2, 4]
```

### Merging two sorted lists

```python
def merge_sorted(first: list[int], second: list[int]) -> list[int]:
    """Merge two sorted lists into a new sorted list. O(n + m) time and space."""
    merged: list[int] = []
    i = j = 0
    while i < len(first) and j < len(second):
        if first[i] <= second[j]:  # <= keeps the merge stable
            merged.append(first[i])
            i += 1
        else:
            merged.append(second[j])
            j += 1
    merged.extend(first[i:])  # one of these two is empty
    merged.extend(second[j:])
    return merged


print(merge_sorted([1, 4, 7], [2, 3, 9, 10]))  # [1, 2, 3, 4, 7, 9, 10]
```

### Palindrome check

```python
def is_palindrome(text: str) -> bool:
    left, right = 0, len(text) - 1
    while left < right:
        if text[left] != text[right]:
            return False
        left += 1
        right -= 1
    return True


print(is_palindrome("racecar"), is_palindrome("python"))  # True False
```

### Three Sum

```python
def three_sum(numbers: list[int]) -> list[tuple[int, int, int]]:
    """All unique triples that sum to zero. O(n^2) time."""
    ordered = sorted(numbers)
    triples: list[tuple[int, int, int]] = []
    for i in range(len(ordered) - 2):
        if ordered[i] > 0:
            break  # all remaining numbers are positive, no triple can sum to zero
        if i > 0 and ordered[i] == ordered[i - 1]:
            continue  # skip duplicate first items
        left, right = i + 1, len(ordered) - 1
        while left < right:
            total = ordered[i] + ordered[left] + ordered[right]
            if total < 0:
                left += 1
            elif total > 0:
                right -= 1
            else:
                triples.append((ordered[i], ordered[left], ordered[right]))
                left += 1
                right -= 1
                while left < right and ordered[left] == ordered[left - 1]:
                    left += 1  # skip duplicate second items
                while left < right and ordered[right] == ordered[right + 1]:
                    right -= 1  # skip duplicate third items
    return triples


print(three_sum([-1, 0, 1, 2, -1, -4]))  # [(-1, -1, 2), (-1, 0, 1)]
```

### Container with most water

```python
def max_water(heights: list[int]) -> int:
    """Largest area between two lines. O(n) time, O(1) space."""
    left, right = 0, len(heights) - 1
    best = 0
    while left < right:
        width = right - left
        best = max(best, width * min(heights[left], heights[right]))
        if heights[left] < heights[right]:
            left += 1  # the shorter line limits the area, so try a different one
        else:
            right -= 1
    return best


print(max_water([1, 8, 6, 2, 5, 4, 8, 3, 7]))  # 49
```

### Partitioning: the Dutch national flag

```python
def sort_colors(colors: list[int]) -> None:
    """Sort a list containing only 0, 1, and 2 in one pass, in place."""
    low, mid, high = 0, 0, len(colors) - 1
    while mid <= high:
        if colors[mid] == 0:
            colors[low], colors[mid] = colors[mid], colors[low]
            low += 1
            mid += 1
        elif colors[mid] == 1:
            mid += 1
        else:
            colors[mid], colors[high] = colors[high], colors[mid]
            high -= 1  # do not advance mid: the swapped-in item is unchecked


data = [2, 0, 2, 1, 1, 0]
sort_colors(data)
print(data)  # [0, 0, 1, 1, 2, 2]
```

## Deeper behavior

### Why moving a pointer is safe

Every two-pointers algorithm rests on a proof that the skipped candidates cannot be part of an answer.

Sorted Two Sum: suppose `numbers[left] + numbers[right] > target`. Then `numbers[right]` plus any item at or to the right of `left` is at least as large as this sum, because the array is sorted, so `numbers[right]` cannot pair with anything still in range. It is safe to discard `right`. The symmetric argument discards `left` when the sum is too small.

Container with most water: the area between two lines is limited by the shorter one. Moving the taller pointer inward can only decrease the width while the height stays limited by the same short line, so the area cannot improve. Only moving the shorter pointer might find a taller line. That is why the shorter pointer moves.

A solution is correct only if you can state this argument. If you cannot explain why the skipped pairs are useless, the technique may not apply.

### The loop invariants

In the Dutch national flag partition, the regions are always:

```text
colors[0:low]       all 0
colors[low:mid]     all 1
colors[mid:high+1]  not yet examined
colors[high+1:]     all 2
```

Each branch keeps those regions valid. When `colors[mid]` is 2 and it is swapped with `colors[high]`, the item that arrives at `mid` has not been examined, so `mid` stays where it is. When `colors[mid]` is 0 it is swapped with `colors[low]`, which is known to be a 1 (or the same slot), so both `low` and `mid` advance safely.

For read and write pointers the invariant is: `values[:write]` holds the kept items seen so far, in order, and `read >= write` always. Because the write pointer never passes the read pointer, writing never overwrites an item that has not been read.

### Complexity

Each pointer moves at most `n` times in total, so the pointer phase is `O(n)`. Techniques that need sorted input add `O(n log n)` for the sort. Three Sum is `O(n log n + n^2)`, which is `O(n^2)`. The extra space is `O(1)` apart from the output and any sorting copy.

### Duplicates and uniqueness

Three Sum shows the usual treatment of duplicates. After sorting, equal values sit next to each other. Skip a repeated first item, and after recording a triple, skip repeated second and third items. The sort is what makes this duplicate skipping a cheap local check instead of a global set lookup.

### Stability and order

A read-write filter keeps the relative order of the kept items (it is stable). The swap-based partition used by quicksort is not stable: equal items can change places. The Dutch national flag algorithm above is also not stable. If order among equal items matters, use the read-write method or a stable sort.

### Strings and pointers

Python strings are immutable, so pointer techniques that modify in place work on a list of characters, `list(text)`. Reading with two pointers needs no conversion, as in the palindrome check.

### Fast and slow pointers on arrays

Two pointers moving at different speeds over a sequence find the middle (when the fast pointer reaches the end, the slow pointer is at the middle) and detect cycles in a linked structure. On a plain list the read and write pointers are the more common form. The cycle-detection form appears again with linked lists.

### When not to use it

- Unsorted data where order cannot be exploited and sorting would destroy the indexes you must return. Use a hash map.
- Problems where the condition is not monotonic in the pointer positions, so moving a pointer cannot be justified.
- Subarray problems with negative numbers where a window sum does not change in one direction. Use prefix sums with a hash map.

## Industry-level snippet

A finance system must reconcile its own ledger against a bank statement. Both sides provide a sorted list of transaction IDs for a day. The reconciliation must find the IDs the ledger has but the bank does not, the IDs the bank has but the ledger does not, and the IDs that match. Doing this with a nested loop costs `O(n * m)`. Because both lists are sorted, one pass with two pointers is `O(n + m)` with no hash table and no extra memory beyond the results.

Production concerns shape the code. The two-pointer logic is only valid on strictly increasing input, so the function checks that and raises a clear error instead of returning a wrong report. The result is saved under a run identifier in one transaction, replacing any earlier result for the same run so a rerun is safe.

```python
# reconciliation.py
import logging
import sqlite3
from collections.abc import Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS reconciliation_item (
    run_id TEXT NOT NULL,
    transaction_id INTEGER NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('matched', 'missing_in_bank', 'missing_in_ledger')),
    PRIMARY KEY (run_id, transaction_id)
)
"""

INSERT_SQL = (
    "INSERT INTO reconciliation_item (run_id, transaction_id, status) "
    "VALUES (?, ?, ?)"
)


class UnsortedInputError(ValueError):
    """Raised when an input list is not strictly increasing."""


@dataclass(frozen=True, slots=True)
class Reconciliation:
    matched: list[int]
    missing_in_bank: list[int]
    missing_in_ledger: list[int]


def require_strictly_increasing(values: Sequence[int], name: str) -> None:
    for index in range(1, len(values)):
        if values[index - 1] >= values[index]:
            raise UnsortedInputError(
                f"{name} must be sorted with no duplicates (problem at index {index})"
            )


def reconcile(ledger_ids: Sequence[int], bank_ids: Sequence[int]) -> Reconciliation:
    """Compare two sorted, duplicate-free lists in O(n + m) time with two pointers."""
    require_strictly_increasing(ledger_ids, "ledger_ids")
    require_strictly_increasing(bank_ids, "bank_ids")
    matched: list[int] = []
    missing_in_bank: list[int] = []
    missing_in_ledger: list[int] = []
    i = j = 0
    while i < len(ledger_ids) and j < len(bank_ids):
        ledger_id, bank_id = ledger_ids[i], bank_ids[j]
        if ledger_id == bank_id:
            matched.append(ledger_id)
            i += 1
            j += 1
        elif ledger_id < bank_id:
            missing_in_bank.append(ledger_id)  # the bank has already passed this id
            i += 1
        else:
            missing_in_ledger.append(bank_id)
            j += 1
    missing_in_bank.extend(ledger_ids[i:])  # whatever remains has no partner
    missing_in_ledger.extend(bank_ids[j:])
    return Reconciliation(matched, missing_in_bank, missing_in_ledger)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_reconciliation(
    connection: sqlite3.Connection, run_id: str, result: Reconciliation
) -> int:
    """Store the result for a run, replacing any earlier result for that run."""
    if not run_id:
        raise ValueError("run_id must not be empty")
    rows = (
        [(run_id, tid, "matched") for tid in result.matched]
        + [(run_id, tid, "missing_in_bank") for tid in result.missing_in_bank]
        + [(run_id, tid, "missing_in_ledger") for tid in result.missing_in_ledger]
    )
    try:
        with connection:
            connection.execute(
                "DELETE FROM reconciliation_item WHERE run_id = ?", (run_id,)
            )
            connection.executemany(INSERT_SQL, rows)
    except sqlite3.Error:
        logger.exception("could not save reconciliation run %s", run_id)
        raise
    logger.info("saved %d reconciliation rows for run %s", len(rows), run_id)
    return len(rows)
```

```python
# test_reconciliation.py
import random
import sqlite3
from collections.abc import Iterator

import pytest

from reconciliation import (
    Reconciliation,
    UnsortedInputError,
    create_schema,
    reconcile,
    save_reconciliation,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def test_basic_reconciliation() -> None:
    result = reconcile([1, 2, 4, 7], [2, 3, 4, 8, 9])
    assert result.matched == [2, 4]
    assert result.missing_in_bank == [1, 7]
    assert result.missing_in_ledger == [3, 8, 9]


@pytest.mark.parametrize(
    ("ledger", "bank"),
    [([], []), ([1, 2], []), ([], [1, 2]), ([1, 2, 3], [1, 2, 3])],
)
def test_empty_and_identical_inputs(ledger: list[int], bank: list[int]) -> None:
    result = reconcile(ledger, bank)
    assert result.matched == sorted(set(ledger) & set(bank))
    assert result.missing_in_bank == sorted(set(ledger) - set(bank))
    assert result.missing_in_ledger == sorted(set(bank) - set(ledger))


@pytest.mark.parametrize("bad", [[2, 1], [1, 1], [1, 3, 2]])
def test_input_must_be_strictly_increasing(bad: list[int]) -> None:
    with pytest.raises(UnsortedInputError):
        reconcile(bad, [1, 2, 3])
    with pytest.raises(UnsortedInputError):
        reconcile([1, 2, 3], bad)


def test_random_inputs_match_a_set_based_oracle() -> None:
    rng = random.Random(7)
    for _ in range(500):
        ledger = sorted(rng.sample(range(40), rng.randint(0, 20)))
        bank = sorted(rng.sample(range(40), rng.randint(0, 20)))
        result = reconcile(ledger, bank)
        assert result.matched == sorted(set(ledger) & set(bank))
        assert result.missing_in_bank == sorted(set(ledger) - set(bank))
        assert result.missing_in_ledger == sorted(set(bank) - set(ledger))


def test_saving_twice_replaces_the_run(connection: sqlite3.Connection) -> None:
    save_reconciliation(connection, "2026-11-08", Reconciliation([1], [2], [3]))
    written = save_reconciliation(connection, "2026-11-08", Reconciliation([5], [], []))
    assert written == 1
    rows = connection.execute(
        "SELECT transaction_id, status FROM reconciliation_item"
    ).fetchall()
    assert rows == [(5, "matched")]


def test_empty_run_id_is_rejected(connection: sqlite3.Connection) -> None:
    with pytest.raises(ValueError):
        save_reconciliation(connection, "", Reconciliation([], [], []))
```

For files too large to hold in memory, the same pointer logic works on two iterators that stream from disk, which is how sort-merge joins in databases operate.

## Common mistakes

- Using two pointers on unsorted data where moving a pointer cannot be justified. Sort first, or use a hash map.
- Using `while left <= right` when pairs of distinct items are required, which lets an item pair with itself. Use `left < right`.
- Forgetting to skip duplicates in Three Sum, which produces repeated triples.
- Advancing both pointers when only one should move.
- Not advancing any pointer in some branch, which creates an infinite loop.
- In partitioning, advancing `mid` after swapping with the `high` end. The item swapped in has not been inspected.
- Overwriting items the read pointer has not reached yet, because the write pointer got ahead of it.
- Removing items from a list inside a `for` loop over the same list instead of using a write pointer.
- Mixing up the stop conditions of merging, so leftover items from one list are dropped.
- Modifying a string in place. Strings are immutable, so convert to a list of characters.
- Skipping the explanation of why the pointer move is safe, and shipping an unproven heuristic.

## Practice exercises

1. Write `reverse_string_in_place(chars)` for a list of characters, using opposite-end pointers.
2. Write `is_palindrome_ignoring_punctuation(text)` using two pointers and no extra string.
3. Write `two_sum_sorted_count(numbers, target)` that counts all pairs (not just one) adding up to the target, handling repeated values.
4. Write `squares_of_sorted(numbers)` that takes a sorted list with negative values and returns the sorted squares in `O(n)` using two pointers.
5. Write `move_zeroes(values)` that moves all zeros to the end while keeping the order of the other items, in place, with read and write pointers.
6. Write `merge_into_first(first, m, second, n)` where `first` has `m` real items followed by `n` free slots, merging from the back in place.
7. Write `intersection_of_sorted(first, second)` that returns the common values of two sorted lists without a set, handling duplicates the way you decide and documenting it.
8. Write `partition_by_pivot(values, pivot)` that rearranges a list so that items less than the pivot come first, then equal items, then greater items, in one pass.
9. Write `closest_pair_sum(numbers, target)` that returns the pair sum nearest to `target` in a sorted list.
10. Write `four_sum(numbers, target)` using sorting and two nested fixed positions plus an opposite-end search. State the time complexity.
11. Write `trap_rain_water(heights)` with two pointers and a running maximum on each side, and test it against a brute-force version.
12. Write `is_subsequence(small, large)` with two pointers over two strings and explain why it is `O(n)`.
13. Prove in three sentences why the pointer moves in `two_sum_sorted` never skip a valid pair.
14. Extend `reconcile` to accept iterators instead of lists by reading one item at a time from each, and test it with generators.

## Summary

Two pointers replaces nested loops with two indexes that move in a controlled way, so the pointer phase is `O(n)`. Opposite-end pointers need sorted or symmetric data, because the order proves that the discarded side cannot contain an answer. Read and write pointers filter and deduplicate in place with `O(1)` extra space, partitioning pointers divide an array into regions while sweeping, and merging two sorted sequences takes `O(n + m)`. Three Sum and container with most water show how sorting plus a safe-move argument turn cubic or quadratic brute force into quadratic or linear. Always be able to state why a pointer move is safe, skip duplicates carefully, and use a hash table or prefix sums when the data is unsorted or the condition is not monotonic.
