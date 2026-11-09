## Overview

The elementary sorts take `O(n^2)` time, which is far too slow beyond a few thousand items. Efficient comparison sorts take `O(n log n)`: for a million items that is about 20 million steps instead of a trillion. Two ideas make the difference, both based on divide and conquer. Merge sort splits the list in half, sorts each half, and merges the results. Quicksort picks a pivot, partitions the list into smaller and larger items, and sorts each side.

You will meet these algorithms again and again, because merging and partitioning appear in databases, search engines, and many other algorithms. Python's own `sorted` is a third algorithm, Timsort, a hybrid of merge sort and insertion sort that exploits order already present in real data. Understanding why it is stable and adaptive explains how to use it well, including sorting by several keys with `key=` and by custom rules with `cmp_to_key`.

This note covers merge sort, quicksort (Lomuto and Hoare partitioning, pivot strategies, randomized pivots, and the worst case), a preview of heap sort, why no comparison sort can beat `n log n`, how Timsort works, and how to use `sorted` and `list.sort` effectively.

## Core concepts

### Merge sort

Merge sort splits the list into two halves, sorts each half recursively, and merges the two sorted halves into one sorted list.

- Time: `O(n log n)` in every case. There are `log2(n)` levels of splitting, and each level does `O(n)` merging work.
- Space: `O(n)` extra for the merged lists.
- Stable, because the merge takes from the left half when items are equal.
- Predictable: it does not depend on the input order.

### Quicksort

Quicksort chooses a pivot item and partitions the list so that every item smaller than the pivot comes before it and every larger item comes after it. The pivot is then in its final position. Sort the two sides recursively.

- Average time `O(n log n)`, worst case `O(n^2)` when pivots are consistently bad.
- Space: `O(log n)` stack on average, in place otherwise.
- Not stable.
- Very fast in practice because it works in place with good memory locality.

### Partitioning schemes

- Lomuto partition: use the last item as the pivot and keep a boundary of the "smaller than pivot" region while scanning left to right. Simple to write and to prove correct. It does more swaps.
- Hoare partition: two pointers start at opposite ends and move toward each other, swapping pairs that are on the wrong side. It does fewer swaps and is faster, but its boundary conditions are harder to get right.

### Pivot strategies and the worst case

The pivot decides the balance of the split. A perfectly central pivot gives `n log n`. A pivot that is always the smallest or largest item gives splits of size `0` and `n - 1`, and the total work is `n + (n - 1) + ... = O(n^2)`.

- First or last item: terrible on sorted or reverse-sorted input, which is common.
- Middle item: fine on sorted input, but adversarial input can still hurt.
- Random item: no input is reliably bad. Expected cost is `O(n log n)` for every input.
- Median of three (first, middle, last): a cheap improvement used in production code.

### Many equal items

A simple partition that sends equal items to one side degrades to quadratic when all items are equal. Three-way partitioning splits the list into smaller, equal, and larger, and the equal part needs no further sorting.

### Heap sort preview

Heap sort builds a heap from the items and repeatedly removes the smallest. It runs in `O(n log n)` in the worst case with `O(1)` extra space when done in place, but it is not stable and it jumps around in memory, so it is usually slower than quicksort. It matters as a safety net: introsort starts with quicksort and switches to heap sort if recursion gets too deep. The heap itself is covered in the trees phase.

### The n log n lower bound

Any comparison sort can be drawn as a decision tree. Each internal node is one comparison, and each leaf is one possible ordering of the input. With `n` distinct items there are `n!` orderings, so the tree needs at least `n!` leaves. A binary tree with that many leaves has height at least `log2(n!)`, which is about `n log2(n) - 1.44 n`. So every comparison sort needs `Ω(n log n)` comparisons in the worst case. Merge sort and heap sort are therefore optimal.

### How Timsort works

Python's `sorted` and `list.sort` use Timsort, designed for real-world data that often contains runs that are already ordered.

1. Scan for natural runs: stretches that are already ascending, or strictly descending (which are reversed in place).
2. Extend short runs to a minimum length (between 32 and 64) using binary insertion sort.
3. Push runs onto a stack and merge neighbors according to rules that keep the merges balanced.
4. While merging, switch to galloping mode (exponential search) when one run keeps winning, so merging a long run with a short one is cheap.

Timsort is stable, takes `O(n)` time on sorted or reverse-sorted input, `O(n log n)` in the worst case, and uses up to `O(n)` extra memory.

### Sorting with key= and cmp_to_key

- `key=` takes a function that maps each item to a sort key. It is called once per item, and the keys are compared instead of the items. Keys can be tuples to sort by several fields.
- `reverse=True` sorts in descending order and still keeps equal items in their original order.
- `functools.cmp_to_key` wraps an old-style comparison function `compare(a, b)` that returns a negative number, zero, or a positive number, for sorting rules that cannot be expressed as a key.

## Basic syntax examples

### Merge sort

```python
def merge(left: list[int], right: list[int]) -> list[int]:
    """Merge two sorted lists. Takes from the left on ties, so the sort is stable."""
    merged: list[int] = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            merged.append(left[i])
            i += 1
        else:
            merged.append(right[j])
            j += 1
    merged.extend(left[i:])
    merged.extend(right[j:])
    return merged


def merge_sort(values: list[int]) -> list[int]:
    """Return a new sorted list. O(n log n) time, O(n) extra space."""
    if len(values) <= 1:
        return values[:]
    middle = len(values) // 2
    return merge(merge_sort(values[:middle]), merge_sort(values[middle:]))


print(merge_sort([38, 27, 43, 3, 9, 82, 10]))  # [3, 9, 10, 27, 38, 43, 82]
```

The splitting, as a tree for `[38, 27, 43, 3]`:

```text
[38, 27, 43, 3]
   split into [38, 27] and [43, 3]
   [38, 27] -> [38] and [27] -> merge -> [27, 38]
   [43, 3]  -> [43] and [3]  -> merge -> [3, 43]
   merge [27, 38] and [3, 43] -> [3, 27, 38, 43]
```

### Quicksort with Lomuto partitioning and a random pivot

```python
import random


def lomuto_partition(values: list[int], low: int, high: int) -> int:
    """Partition values[low..high] around values[high]. Returns the pivot's index.

    Invariant: values[low:boundary] are smaller than the pivot.
    """
    pivot = values[high]
    boundary = low
    for i in range(low, high):
        if values[i] < pivot:
            values[boundary], values[i] = values[i], values[boundary]
            boundary += 1
    values[boundary], values[high] = values[high], values[boundary]
    return boundary


def quicksort(values: list[int]) -> None:
    """Sort in place. Expected O(n log n), stack depth O(log n)."""
    _quicksort(values, 0, len(values) - 1)


def _quicksort(values: list[int], low: int, high: int) -> None:
    while low < high:
        pivot_index = random.randint(low, high)  # a random pivot defeats bad inputs
        values[pivot_index], values[high] = values[high], values[pivot_index]
        split = lomuto_partition(values, low, high)
        # Recurse into the smaller side and loop on the larger one,
        # which keeps the recursion depth at O(log n) even for bad splits.
        if split - low < high - split:
            _quicksort(values, low, split - 1)
            low = split + 1
        else:
            _quicksort(values, split + 1, high)
            high = split - 1


data = [9, 4, 7, 1, 8, 2, 8]
quicksort(data)
print(data)  # [1, 2, 4, 7, 8, 8, 9]
```

One partition of `[7, 2, 9, 4, 5]` around pivot `5` (the last item):

```text
boundary = 0
scan 7: not smaller than 5
scan 2: smaller, swap with index 0 -> [2, 7, 9, 4, 5], boundary = 1
scan 9: not smaller
scan 4: smaller, swap with index 1 -> [2, 4, 9, 7, 5], boundary = 2
finally swap the pivot into the boundary -> [2, 4, 5, 7, 9], pivot index 2
```

### Hoare partitioning

```python
def hoare_partition(values: list[int], low: int, high: int) -> int:
    """Returns j such that values[low..j] <= pivot <= values[j+1..high]."""
    pivot = values[(low + high) // 2]
    i, j = low - 1, high + 1
    while True:
        i += 1
        while values[i] < pivot:
            i += 1
        j -= 1
        while values[j] > pivot:
            j -= 1
        if i >= j:
            return j
        values[i], values[j] = values[j], values[i]


def quicksort_hoare(values: list[int], low: int = 0, high: int | None = None) -> None:
    if high is None:
        high = len(values) - 1
    if low < high:
        split = hoare_partition(values, low, high)
        quicksort_hoare(values, low, split)  # note: split, not split - 1
        quicksort_hoare(values, split + 1, high)


data = [9, 4, 7, 1, 8, 2, 8]
quicksort_hoare(data)
print(data)  # [1, 2, 4, 7, 8, 8, 9]
```

### Heap sort preview

```python
import heapq


def heap_sort(values: list[int]) -> list[int]:
    """O(n log n) worst case. Uses a heap from the standard library."""
    heap = values[:]
    heapq.heapify(heap)  # O(n)
    return [heapq.heappop(heap) for _ in range(len(heap))]  # n pops, O(log n) each


print(heap_sort([5, 3, 8, 1, 9, 2]))  # [1, 2, 3, 5, 8, 9]
```

### The lower bound in numbers

```python
import math

# Sorting 10 distinct items needs at least ceil(log2(10!)) comparisons.
print(math.ceil(math.log2(math.factorial(10))))  # 22
```

### sorted, key, and reverse

```python
words = ["banana", "apple", "Cherry", "Apple"]
print(sorted(words))  # ['Apple', 'Cherry', 'apple', 'banana'], uppercase sorts first
print(sorted(words, key=str.casefold))  # ['apple', 'Apple', 'banana', 'Cherry']
print(sorted(words, key=len))  # ['apple', 'Apple', 'banana', 'Cherry']

players = [("zoe", 90), ("adam", 85), ("bella", 90)]
by_score_then_name = sorted(players, key=lambda player: (-player[1], player[0]))
print(by_score_then_name)  # [('bella', 90), ('zoe', 90), ('adam', 85)]

numbers = [3, 1, 2]
numbers.sort(reverse=True)  # in place, returns None
print(numbers)  # [3, 2, 1]
```

### Custom order with cmp_to_key

The "largest number" problem: arrange numbers so that their digits, joined together, form the largest possible number. No single key captures the rule, but a comparison does: `a` should come before `b` if `a + b` is larger than `b + a` as text.

```python
from functools import cmp_to_key


def largest_number(numbers: list[int]) -> str:
    if not numbers:
        raise ValueError("numbers must not be empty")

    def compare(first: str, second: str) -> int:
        if first + second > second + first:
            return -1  # first goes before second
        if first + second < second + first:
            return 1
        return 0

    parts = sorted((str(number) for number in numbers), key=cmp_to_key(compare))
    result = "".join(parts)
    return "0" if result[0] == "0" else result


print(largest_number([3, 30, 34, 5, 9]))  # 9534330
```

## Deeper behavior

### Merge sort in detail

The merge step does at most `len(left) + len(right) - 1` comparisons and touches every item once, so each level of the recursion costs `O(n)` and there are `log2(n)` levels. The recurrence is `T(n) = 2T(n / 2) + O(n)`, which solves to `O(n log n)` by the Master theorem. The recursion depth is only `log2(n)`, so merge sort never hits Python's recursion limit.

The implementation above is simple but allocates many small lists by slicing. A more careful version sorts index ranges and merges into one preallocated buffer of size `n`. A bottom-up version avoids recursion entirely: merge runs of size 1 into runs of size 2, then 4, then 8, and so on. Merge sort is also the natural choice for data that does not fit in memory, because it reads and writes sequentially. External sorting and sort-merge joins in databases are built on it.

### Why quicksort is fast and why it is not stable

Quicksort sorts in place and scans memory sequentially, so it makes good use of the processor cache. But partitioning swaps items across long distances, so equal items can cross each other and lose their original order. If you need stability, use merge sort or Timsort.

### The expected cost of a random pivot

With a random pivot, every item is equally likely to be the pivot, so the split is balanced often enough that the recursion depth is `O(log n)` with high probability. The expected number of comparisons is about `1.39 n log2(n)`, only 39 percent more than the information-theoretic minimum. A randomized algorithm shifts the question from "which input is bad?" to "was I unlucky?", and the probability of a quadratic run on a large input is astronomically small.

### Keeping the stack small

A recursion that always recurses on both sides can reach depth `n` with consistently bad splits. Recursing into the smaller side and looping on the larger one guarantees that each recursive call handles at most half of the items, so the depth is at most `log2(n)`. The version of `_quicksort` above does exactly this.

### Equal items and three-way partitioning

With Lomuto partitioning and `<`, a list of `n` equal items puts nothing on the left, so each partition removes only the pivot, and the total work is `O(n^2)`. Three-way partitioning fixes this:

```python
def quicksort_three_way(values: list[int]) -> list[int]:
    """Returns a new sorted list. Equal items are placed once and never revisited."""
    if len(values) <= 1:
        return values[:]
    pivot = random.choice(values)
    smaller = [value for value in values if value < pivot]
    equal = [value for value in values if value == pivot]
    larger = [value for value in values if value > pivot]
    return quicksort_three_way(smaller) + equal + quicksort_three_way(larger)


print(quicksort_three_way([4, 4, 4, 4, 4]))  # [4, 4, 4, 4, 4], one pass
```

This version is simple but not in place. In-place versions use the Dutch national flag scheme from the two-pointers topic.

### Introsort

Many standard libraries use introsort: run quicksort, but track the recursion depth, and if it exceeds about `2 log2(n)`, switch to heap sort for that part. It keeps quicksort's speed and guarantees `O(n log n)` in the worst case. For small pieces it finishes with insertion sort.

### Why Timsort suits real data

Real data is rarely random. Logs are appended in time order, database results often arrive partly sorted, and a list that is "sorted plus a few new items" is common. Timsort detects those runs and merges them, so it costs close to `O(n)` on such data. You can see this by counting comparisons with a class that counts every `<`:

```python
class Counted:
    comparisons = 0

    def __init__(self, value: int) -> None:
        self.value = value

    def __lt__(self, other: "Counted") -> bool:
        Counted.comparisons += 1
        return self.value < other.value


def comparisons_for(values: list[int]) -> int:
    Counted.comparisons = 0
    sorted(Counted(value) for value in values)
    return Counted.comparisons


ordered = list(range(1_000))
print(comparisons_for(ordered))  # 999
print(comparisons_for(ordered[::-1]))  # 999, a descending run is detected and reversed
```

For 1,000 randomly shuffled items the count is roughly 8,600, a little below `n log2(n)`, which is about 9,966.

### Stability and multi-key sorting

Because `sorted` is stable, you can sort by several keys in separate passes: sort by the least important key first, then by the next, and finish with the most important. Each later pass keeps the earlier order among ties. A single pass with a tuple key does the same thing more efficiently, as long as every field can be compared in the same direction. For a mix of ascending and descending fields on non-numeric data, use two stable passes:

```python
people = [("lee", "sales"), ("ann", "dev"), ("bob", "sales"), ("cy", "dev")]
people.sort(key=lambda person: person[0])  # name ascending
people.sort(key=lambda person: person[1], reverse=True)  # then team descending
print(people)  # [('bob', 'sales'), ('lee', 'sales'), ('ann', 'dev'), ('cy', 'dev')]
```

`reverse=True` does not simply reverse the result: equal items still keep their original order, which is why it composes correctly with earlier passes.

### Properties of key=

- The key function is called exactly once per item, so an expensive key costs `n` calls, not `n log n`.
- Keys must be comparable with each other. Mixing `None` and numbers, or numbers and strings, raises `TypeError`.
- Negating a key reverses a numeric order. It does not work for strings.
- `float("nan")` breaks ordering, because every comparison with it is false. Clean or reject NaN values before sorting.

### Choosing a sort

- Default: `sorted` or `list.sort`. They are stable, adaptive, fast, and tested.
- Need a stable sort of records: the built-in already is.
- Need only the smallest `k` items: `heapq.nsmallest` or quickselect, covered in a later topic.
- Integers in a small range, or fixed-width keys: the non-comparison sorts in the next note can beat `n log n`.
- Data larger than memory: external merge sort.

## Industry-level snippet

A search service splits its index across several shards. For a query, every shard returns its own best hits already sorted by relevance: highest score first, ties broken by document id. The coordinator must merge the shard lists into one global top-`N`, and a document that appears in more than one shard (for example because of replication) must appear only once with its best score.

The merge step of merge sort is exactly the right tool. `heapq.merge` merges any number of sorted inputs lazily with a heap, so it produces the global order without sorting everything and stops as soon as the top `N` are collected. A second function sorts a product listing with `key=` and a deterministic tie-break, so pages are stable between requests.

Production concerns shape the code. The merge is only correct if each shard's list is really sorted, so it validates the input and raises a specific error naming the bad shard. `NaN` scores would silently corrupt every comparison, so they are rejected up front.

```python
# search_results.py
import heapq
import logging
import math
from collections.abc import Callable, Sequence
from dataclasses import dataclass
from itertools import pairwise

logger = logging.getLogger(__name__)

type ProductKey = tuple[float, str, str]


class UnsortedShardError(ValueError):
    """Raised when a shard returned hits that are not in the agreed order."""


class InvalidScoreError(ValueError):
    """Raised for a score that cannot be ordered, such as NaN."""


class UnknownSortModeError(ValueError):
    """Raised when a listing is requested with an unsupported sort mode."""


@dataclass(frozen=True, slots=True)
class Hit:
    doc_id: int
    score: float
    shard: int


@dataclass(frozen=True, slots=True)
class Product:
    sku: str
    name: str
    price_cents: int
    rating: float


def hit_order(hit: Hit) -> tuple[float, int]:
    """Best first: higher score, then lower document id."""
    return (-hit.score, hit.doc_id)


def validate_shard(index: int, hits: Sequence[Hit]) -> None:
    if any(math.isnan(hit.score) for hit in hits):
        raise InvalidScoreError(f"shard {index} returned a NaN score")
    keys = [hit_order(hit) for hit in hits]
    if any(earlier > later for earlier, later in pairwise(keys)):
        raise UnsortedShardError(f"shard {index} is not sorted by score and id")


def merge_shard_hits(shards: Sequence[Sequence[Hit]], limit: int) -> list[Hit]:
    """Merge sorted shard results into the global top `limit`, without duplicates.

    heapq.merge keeps one candidate per shard in a heap, so the cost is
    O(limit * log(number of shards)) plus the O(total hits) validation.
    """
    if limit < 1:
        raise ValueError("limit must be at least 1")
    for index, hits in enumerate(shards):
        validate_shard(index, hits)
    results: list[Hit] = []
    seen: set[int] = set()
    for hit in heapq.merge(*shards, key=hit_order):
        if hit.doc_id in seen:
            continue  # a better-scored copy of this document came out first
        seen.add(hit.doc_id)
        results.append(hit)
        if len(results) == limit:
            break
    logger.debug("merged %d shards into %d hits", len(shards), len(results))
    return results


def price_asc(product: Product) -> ProductKey:
    return (float(product.price_cents), product.name.casefold(), product.sku)


def price_desc(product: Product) -> ProductKey:
    return (-float(product.price_cents), product.name.casefold(), product.sku)


def best_rated(product: Product) -> ProductKey:
    return (-product.rating, product.name.casefold(), product.sku)


SORT_MODES: dict[str, Callable[[Product], ProductKey]] = {
    "price_asc": price_asc,
    "price_desc": price_desc,
    "rating": best_rated,
}


def sort_products(products: Sequence[Product], mode: str) -> list[Product]:
    """Sort a listing. The sku in every key makes the order fully deterministic."""
    key = SORT_MODES.get(mode)
    if key is None:
        raise UnknownSortModeError(f"unknown sort mode {mode!r}")
    return sorted(products, key=key)
```

```python
# test_search_results.py
import math
import random

import pytest

from search_results import (
    Hit,
    InvalidScoreError,
    Product,
    UnknownSortModeError,
    UnsortedShardError,
    hit_order,
    merge_shard_hits,
    sort_products,
)


def hit(doc_id: int, score: float, shard: int = 0) -> Hit:
    return Hit(doc_id, score, shard)


def test_merges_in_global_order() -> None:
    shard_a = [hit(1, 0.9, 0), hit(4, 0.5, 0)]
    shard_b = [hit(2, 0.8, 1), hit(3, 0.7, 1)]
    merged = merge_shard_hits([shard_a, shard_b], limit=10)
    assert [h.doc_id for h in merged] == [1, 2, 3, 4]


def test_duplicate_documents_keep_the_best_score() -> None:
    shard_a = [hit(7, 0.9, 0), hit(8, 0.4, 0)]
    shard_b = [hit(7, 0.6, 1), hit(9, 0.5, 1)]
    merged = merge_shard_hits([shard_a, shard_b], limit=10)
    assert [(h.doc_id, h.score) for h in merged] == [(7, 0.9), (9, 0.5), (8, 0.4)]


def test_ties_are_broken_by_document_id() -> None:
    merged = merge_shard_hits([[hit(5, 1.0)], [hit(2, 1.0)]], limit=2)
    assert [h.doc_id for h in merged] == [2, 5]


def test_limit_stops_the_merge() -> None:
    shard = [hit(i, 1.0 - i / 100) for i in range(10)]
    assert len(merge_shard_hits([shard], limit=3)) == 3


def test_unsorted_shard_is_rejected() -> None:
    with pytest.raises(UnsortedShardError):
        merge_shard_hits([[hit(1, 0.1), hit(2, 0.9)]], limit=5)


def test_nan_score_is_rejected() -> None:
    with pytest.raises(InvalidScoreError):
        merge_shard_hits([[hit(1, math.nan)]], limit=5)


def test_limit_must_be_positive() -> None:
    with pytest.raises(ValueError):
        merge_shard_hits([], limit=0)


def test_random_shards_match_a_sort_everything_oracle() -> None:
    rng = random.Random(3)
    for _ in range(300):
        shards = []
        for shard_index in range(rng.randint(0, 4)):
            hits = [
                hit(rng.randint(0, 15), rng.randint(0, 5) / 5, shard_index)
                for _ in range(rng.randint(0, 8))
            ]
            shards.append(sorted(hits, key=hit_order))
        limit = rng.randint(1, 10)
        everything = sorted((h for shard in shards for h in shard), key=hit_order)
        expected: list[Hit] = []
        seen: set[int] = set()
        for candidate in everything:
            if candidate.doc_id not in seen:
                seen.add(candidate.doc_id)
                expected.append(candidate)
        merged = merge_shard_hits(shards, limit)
        assert [h.doc_id for h in merged] == [h.doc_id for h in expected[:limit]]


PRODUCTS = [
    Product("s3", "Mouse", 2_500, 4.5),
    Product("s1", "keyboard", 7_900, 4.8),
    Product("s2", "Monitor", 19_900, 4.5),
    Product("s4", "Cable", 2_500, 3.9),
]


@pytest.mark.parametrize(
    ("mode", "expected_skus"),
    [
        ("price_asc", ["s4", "s3", "s1", "s2"]),
        ("price_desc", ["s2", "s1", "s4", "s3"]),
        ("rating", ["s1", "s2", "s3", "s4"]),
    ],
)
def test_sort_modes(mode: str, expected_skus: list[str]) -> None:
    assert [p.sku for p in sort_products(PRODUCTS, mode)] == expected_skus


def test_unknown_mode_is_rejected() -> None:
    with pytest.raises(UnknownSortModeError):
        sort_products(PRODUCTS, "popularity")


def test_sorting_does_not_modify_the_input() -> None:
    snapshot = list(PRODUCTS)
    sort_products(PRODUCTS, "price_asc")
    assert PRODUCTS == snapshot
```

The `name.casefold()` in each key makes `keyboard` and `Monitor` compare alphabetically without regard to case, and the `sku` at the end guarantees that two products that tie on everything else always appear in the same order, which keeps page boundaries stable between requests.

## Common mistakes

- Choosing the first or last element as the quicksort pivot on data that may already be sorted, which gives quadratic time and deep recursion.
- Writing a quicksort that recurses on both sides without limiting depth, so bad input causes `RecursionError`.
- Letting a simple partition send all equal items to one side, which makes many duplicates quadratic.
- Off-by-one errors in Hoare partitioning: recursing on `split - 1` instead of `split`, or the wrong loop termination, which loops forever or skips items.
- Using `<` instead of `<=` in the merge step and breaking stability.
- Slicing in merge sort and forgetting that every level copies `n` items, which is fine, but doing the same in a quadratic pattern elsewhere.
- Expecting `values.sort()` to return the list. It returns `None`.
- Sorting with a key function that is expensive and also wrong when called twice, assuming it is called per comparison. It is called once per item.
- Sorting a list that contains `None`, mixed types, or `NaN`, which raises `TypeError` or silently produces a wrong order.
- Negating a string key to sort descending. Use `reverse=True` or a two-pass sort instead.
- Reaching for a hand-written sort in production when `sorted` is faster, stable, and already tested.

## Practice exercises

1. Trace merge sort by hand on `[5, 2, 9, 1, 5, 6]` and draw the tree of splits and merges. Say which item from the two 5s ends up first, and why.
2. Write a bottom-up (iterative) merge sort that merges runs of size 1, 2, 4, and so on using a single auxiliary list.
3. Write `quicksort_median_of_three` that picks the median of the first, middle, and last items as the pivot, and compare its comparison counts with a last-item pivot on sorted input.
4. Run the last-item-pivot quicksort on a sorted list of 5,000 items and observe what happens. Explain the result and fix it with a random pivot.
5. Write an in-place three-way quicksort using the Dutch national flag partition, and test it on lists with many duplicates.
6. Count the exact number of comparisons made by `merge_sort` on random lists of 1,000, 2,000, and 4,000 items. Compare the growth with `n log2(n)`.
7. Write `count_inversions(values)` in `O(n log n)` by counting during the merge step, and check it against the brute-force count.
8. Implement `merge_k_sorted(lists)` using a heap and again with `heapq.merge`. Compare their speed and state the time complexity in terms of the total number of items and the number of lists.
9. Use `sorted` with a tuple key to sort `(name, age, city)` records by city ascending, age descending, and name ascending. Then do it again with three stable passes.
10. Write `sort_by_frequency(text)` that orders characters by frequency, highest first, with ties broken alphabetically, using one `sorted` call.
11. Write a comparison function and use `cmp_to_key` to sort version strings such as `"1.10.2"` and `"1.9"` correctly. Then rewrite it as a key function and say which is simpler.
12. Prove in a few sentences why `n log2(n)` comparisons are needed to sort `n` items, using the decision tree and `n!`.
13. Time `sorted` on an already sorted list, a reversed list, a list with 10 random items appended to a sorted list, and a fully random list of the same size, and explain the differences using the description of Timsort.
14. Extend `merge_shard_hits` to return an additional field telling which shards contributed to the final list, and test it.

## Summary

Efficient sorting uses divide and conquer to reach `O(n log n)`. Merge sort splits, sorts, and merges, is stable, always takes `n log n`, and needs `O(n)` extra space. Quicksort partitions around a pivot, sorts in place, averages `n log n`, and degrades to `n^2` with consistently bad pivots, so use random or median-of-three pivots, recurse into the smaller side to keep the stack shallow, and treat many equal items with three-way partitioning. Heap sort guarantees `n log n` in place but is not stable. No comparison sort can beat `n log n` in the worst case, because a decision tree for `n!` orderings is at least that tall. Python's Timsort is a stable, adaptive hybrid that runs in `O(n)` on sorted data, so use `sorted` and `list.sort` with `key=` for multi-field ordering and `cmp_to_key` for custom comparisons, and remember that the key function runs once per item.
