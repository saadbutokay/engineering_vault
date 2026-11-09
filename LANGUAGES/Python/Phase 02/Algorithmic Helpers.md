## Overview

Two programs can produce the same answer, yet one finishes in a blink and the other takes minutes once the data grows. The difference is rarely clever tricks. It is usually the choice of **data structure and algorithm**, and the vocabulary for talking about it is **Big-O notation**. This chapter gives you that vocabulary, a table of what the built-in operations cost, and two standard-library helpers, `heapq` and `bisect`, that solve "find the smallest items" and "search sorted data" efficiently.

By the end of this note you will be able to:

- Explain Big-O notation and read the common complexity classes.
- State the time complexity of the main operations on lists, dictionaries, sets, and deques.
- Use `heapq` to maintain a priority queue and to find the largest or smallest items.
- Use `bisect` to search and insert into sorted lists in logarithmic time.
- Choose a data structure by thinking about how the data will be used.

> [!note] Measure before you optimize
> Big-O tells you how a cost **grows**, not how big it is today. A simple list is often the right choice for small data. Use this chapter to avoid obviously bad choices, and use measurement (Chapter 11.5) to find the real bottlenecks.

## Core concepts

### What Big-O notation says

**Big-O notation** describes how the running time (or memory) of an operation grows as the size of its input, usually called **n**, grows. It ignores constant factors and small terms, and keeps only the dominant behavior. If doubling the input doubles the time, the operation is **O(n)**, read "order n" or "linear".

### The common classes

| Class | Name | If n doubles, time... | Example |
| --- | --- | --- | --- |
| O(1) | Constant | stays the same | `d[key]`, `items[3]`, `items.append(x)` |
| O(log n) | Logarithmic | grows by one small step | Binary search with `bisect` |
| O(n) | Linear | doubles | `x in items`, `sum(items)`, `items.insert(0, x)` |
| O(n log n) | Linearithmic | slightly more than doubles | `sorted(items)` |
| O(n squared) | Quadratic | quadruples | Comparing every item with every other item |
| O(2 to the n) | Exponential | explodes | Trying every subset |

Logarithms grow very slowly. A million items need about 20 halving steps. A billion need about 30. That is why logarithmic algorithms scale so well.

### Average, worst, and amortized

- **Worst case** is the slowest the operation can ever be. **Average case** is the typical cost. For hash tables the worst case is rare and effectively avoided by Python's hashing.
- **Amortized** cost averages over many operations. `list.append` is O(1) amortized: occasionally the array must grow and copy everything, but spread over many appends the average cost per append is constant.

### Counting nested work

Loops multiply. A loop over n items that does a linear search inside it is O(n) times O(n), which is O(n squared). Hidden linear work, such as `x in some_list` inside a loop, is the most common cause of code that is fast on a test file and unusable in production.

### Time complexity of built-in operations

| Structure and operation | Cost |
| --- | --- |
| list: index `items[i]`, `len`, `append`, `pop()` | O(1) |
| list: `insert(i, x)`, `pop(i)`, `remove(x)`, `x in items`, `index(x)`, `count(x)` | O(n) |
| list: `sort()`, `sorted(items)` | O(n log n) |
| list: slice `items[a:b]` | O(size of slice) |
| list: `min`, `max`, `sum` | O(n) |
| dict: `d[key]`, `d[key] = v`, `del d[key]`, `key in d` | O(1) average |
| dict: iterate all items | O(n) |
| set: `add`, `remove`, `discard`, `x in s` | O(1) average |
| set: union, intersection, difference | About O(len of the sets involved) |
| deque: `append`, `appendleft`, `pop`, `popleft` | O(1) |
| deque: index in the middle | O(n) |
| str: `x in text`, `text.find(x)` | O(n) |
| str: concatenation `a + b` | O(len(a) + len(b)) |
| heap (`heapq`): `heappush`, `heappop` | O(log n) |
| heap: `heapify(items)` | O(n) |
| bisect: `bisect_left`, `bisect_right` | O(log n) |
| bisect: `insort` | O(n), because the insertion shifts items |

### heapq: the priority queue

A **heap** is a structure that always gives you the smallest item first, quickly. Python's `heapq` module implements a **min-heap** on an ordinary list. After `heapify`, the smallest item is always at `heap[0]`, and pushing or popping takes O(log n). The rest of the list is only partly ordered, so printing a heap does not show a sorted list.

| Function | What it does |
| --- | --- |
| `heapq.heappush(heap, item)` | Add an item |
| `heapq.heappop(heap)` | Remove and return the smallest item |
| `heapq.heapify(items)` | Turn a list into a heap in place |
| `heapq.heappushpop(heap, item)` | Push, then pop the smallest, more efficiently than two calls |
| `heapq.nsmallest(n, iterable, key=None)` | The `n` smallest items, sorted |
| `heapq.nlargest(n, iterable, key=None)` | The `n` largest items, sorted |
| `heapq.merge(*sorted_iterables)` | Merge already-sorted inputs lazily |

There is no max-heap. To get the largest first, push the **negated** value, or use `nlargest`.

### Priority queues with tuples

Push `(priority, item)` tuples. Tuples compare item by item (Chapter 2.2), so the smallest priority comes out first. If two priorities are equal, Python compares the items, which fails if they cannot be ordered. A common solution is to insert a running counter between them: `(priority, counter, item)`. It also makes equal-priority items come out in first-in, first-out order.

### bisect: binary search on sorted lists

**Binary search** finds a position in a **sorted** list by repeatedly halving the range. The `bisect` module provides it:

| Function | What it does |
| --- | --- |
| `bisect.bisect_left(items, x)` | The position where `x` would go, before any equal items |
| `bisect.bisect_right(items, x)` | The same, after any equal items (also named `bisect`) |
| `bisect.insort(items, x)` | Insert `x` and keep the list sorted |

Both search functions and `insort` accept a `key=` argument in Python 3.10 and later. The list **must already be sorted**, or the results are meaningless.

## Basic syntax examples

### heapq basics

```text
>>> import heapq
>>> heap = []
>>> for n in [5, 1, 8, 3]:
...     heapq.heappush(heap, n)
...
>>> heap[0]
1
>>> heapq.heappop(heap)
1
>>> heapq.heappop(heap)
3
>>> heap[0]
5
>>> len(heap)
2
```

### Turning a list into a heap

```text
>>> import heapq
>>> data = [9, 4, 7, 1]
>>> heapq.heapify(data)
>>> data[0]
1
>>> [heapq.heappop(data) for _ in range(len(data))]
[1, 4, 7, 9]
```

Popping everything from a heap returns the items in sorted order (this is the idea behind "heap sort").

### nsmallest, nlargest, merge

```text
>>> import heapq
>>> heapq.nsmallest(2, [9, 4, 7, 1])
[1, 4]
>>> heapq.nlargest(2, [9, 4, 7, 1])
[9, 7]
>>> words = ["pear", "fig", "banana", "kiwi"]
>>> heapq.nlargest(2, words, key=len)
['banana', 'pear']
>>> list(heapq.merge([1, 4, 7], [2, 3, 9]))
[1, 2, 3, 4, 7, 9]
```

### A priority queue

```text
>>> import heapq
>>> tasks = []
>>> heapq.heappush(tasks, (2, "write report"))
>>> heapq.heappush(tasks, (1, "fix outage"))
>>> heapq.heappush(tasks, (2, "answer email"))
>>> heapq.heappop(tasks)
(1, 'fix outage')
>>> heapq.heappop(tasks)
(2, 'answer email')
```

The two priority-2 tasks were compared as strings to break the tie, so `"answer email"` came before `"write report"`. That is not the order they were added. The industry-level snippet below fixes this with a counter.

### A max-heap by negation

```text
>>> import heapq
>>> heap = []
>>> for n in [5, 1, 8, 3]:
...     heapq.heappush(heap, -n)
...
>>> -heapq.heappop(heap)
8
```

### bisect basics

```text
>>> import bisect
>>> scores = [10, 20, 30, 40]
>>> bisect.bisect_left(scores, 30)
2
>>> bisect.bisect_right(scores, 30)
3
>>> bisect.bisect_left(scores, 25)
2
>>> bisect.insort(scores, 25)
>>> scores
[10, 20, 25, 30, 40]
```

For a value that is already present, `bisect_left` returns its position and `bisect_right` returns the position just after it. For a missing value they agree.

### Grade lookup with bisect

```python
import bisect

BREAKPOINTS = [60, 70, 80, 90]
LETTERS = "FDCBA"


def letter_grade(score: int) -> str:
    return LETTERS[bisect.bisect_right(BREAKPOINTS, score)]


print([letter_grade(score) for score in [59, 60, 89, 90, 100]])
```

```text
['F', 'D', 'B', 'A', 'A']
```

This replaces a long `if/elif` chain with a table, and works the same way for tax brackets, shipping tiers, and pricing steps.

### Seeing Big-O in practice

```python
import timeit

for size in (10_000, 20_000, 40_000, 80_000):
    items = list(range(size))
    seconds = timeit.timeit(lambda: -1 in items, number=300) / 300
    print(f"{size:>6} items: {seconds * 1e6:7.1f} microseconds")
```

```text
 10000 items:    51.5 microseconds
 20000 items:    92.5 microseconds
 40000 items:   178.8 microseconds
 80000 items:   385.2 microseconds
```

These numbers come from one machine and yours will differ, but each doubling of the list roughly doubles the time. That is O(n) made visible. The same experiment with a set would show a flat line.

### Front insertion: list versus deque

```python
import timeit
from collections import deque

for size in (10_000, 20_000, 40_000):
    items = list(range(size))
    queue = deque(range(size))
    list_time = timeit.timeit(lambda: items.insert(0, 1), number=2000) / 2000
    deque_time = timeit.timeit(lambda: queue.appendleft(1), number=2000) / 2000
    print(f"{size:>6}: list {list_time * 1e6:5.2f} us, deque {deque_time * 1e6:5.3f} us")
```

```text
 10000: list  2.06 us, deque 0.034 us
 20000: list  3.83 us, deque 0.031 us
 40000: list  7.32 us, deque 0.033 us
```

The list gets slower as it grows. The deque stays constant.

### Top-k without sorting everything

```python
import heapq
import random

values = [random.random() for _ in range(200_000)]

top_five = heapq.nlargest(5, values)
also_top_five = sorted(values, reverse=True)[:5]
print(top_five == also_top_five)
```

```text
True
```

Both give the same answer. On one test machine, sorting 200,000 values to keep five took about 27 milliseconds, while `nlargest` took about 1.6. Sorting does far more work than the question needs.

## Deeper behavior

### Why `list.append` is O(1) amortized

A Python list reserves extra room beyond its current length. Most appends simply use some of that room. When it runs out, Python allocates a bigger block (growing by a proportion of the current size) and copies the items over. Because each expensive copy is followed by many cheap appends, the average cost per append stays constant. Inserting at the front has no such trick: every item has to shift.

### Hash tables and the worst case

A dictionary or set lookup is O(1) on average. In theory, if many keys land in the same slot, lookups degrade toward O(n). Python's randomized string hashing (Chapter 2.3) makes it practically impossible for an attacker to craft such keys. For your own classes with a poor `__hash__`, the problem can appear for real (Chapter 6.3).

### Constants matter at small sizes

Big-O ignores constant factors. A linear scan of ten items can beat a hash lookup because there is less overhead. Do not convert small collections into sets "for speed". Convert when the collection is large or when you search it repeatedly.

### Sorting and stability

`sorted` and `list.sort` use Timsort, which is O(n log n) in general, and approaches O(n) on data that is already nearly sorted. It is **stable**: equal items keep their relative order. A heap is **not** a sorted container and not stable, which is why it is used for "give me the next smallest" rather than "give me everything in order".

### When to use `nlargest` or `sorted`

| Situation | Better choice |
| --- | --- |
| A few extremes from a large collection | `heapq.nlargest` or `nsmallest` |
| Only the single extreme | `max` or `min` |
| Most of the collection | `sorted(...)[:n]` |
| A continuous stream where you keep the best n | A heap of size n |

### A heap of fixed size keeps the top k of a stream

To track the k largest items in a stream without storing the entire stream, keep a **min-heap of size k**. The smallest of your current top k sits at `heap[0]`. When a new item arrives that is bigger, replace it with `heappushpop`. Each item costs O(log k), and memory stays at k items.

### `insort` is not O(log n)

Finding the position is logarithmic, but inserting into a list shifts everything after it, so `insort` is O(n) overall. It is excellent for occasional inserts into a list that you also search often. For heavy insert traffic on large data, a heap, a balanced tree from a third-party library, or a database is a better fit.

### Binary search needs sorted data

`bisect` does not check that the list is sorted. If it is not, you get an answer anyway, and it is wrong. Sort once, then search many times. If the data changes often and must stay sorted, use `insort` or rethink the structure.

### Space complexity

Memory cost is described with the same notation. `set(items)` takes O(n) extra space. A generator expression (Chapter 7.2) takes O(1). Faster algorithms often trade memory for time, as a set does when it speeds up membership tests.

### Recursion and complexity

Recursive algorithms (Chapter 3.8) can hide enormous costs. A naive recursive Fibonacci function is exponential because it recomputes the same values repeatedly. Memoization (Chapter 3.7) turns it into linear time. Whenever you write a recursive function, ask how many calls it will make.

### Choosing a data structure by usage

| You mainly need to... | Reach for |
| --- | --- |
| Access by position | list |
| Look up by key | dict |
| Ask "is it present?" repeatedly | set |
| Add and remove at both ends | deque |
| Repeatedly get the smallest item | heap |
| Search a sorted collection | `bisect` on a list |
| Keep order and uniqueness | `dict.fromkeys` |

## Industry-level snippet

Operations dashboards and job systems constantly answer questions like "which are the five slowest endpoints?", "what fraction of requests were faster than 200 ms?", and "which job runs next?". This module answers each with the right structure: a bounded heap for the slowest endpoints, binary search for latency percentiles, and a priority queue with a first-in, first-out tie-breaker for jobs.

```python
"""Algorithmic helpers for monitoring and scheduling."""

import bisect
import heapq
import itertools
import logging
from collections.abc import Iterable

logger = logging.getLogger(__name__)


def top_k_slowest(
    measurements: Iterable[tuple[str, float]], k: int
) -> list[tuple[str, float]]:
    """Return the k slowest (endpoint, seconds) pairs, slowest first.

    Uses a min-heap of size k, so memory stays at k items however long the
    stream is, and each measurement costs O(log k).

    Raises:
        ValueError: If k is less than 1.
    """
    if k < 1:
        raise ValueError("k must be at least 1")
    heap: list[tuple[float, str]] = []  # (seconds, endpoint): smallest on top
    for endpoint, seconds in measurements:
        if len(heap) < k:
            heapq.heappush(heap, (seconds, endpoint))
        elif seconds > heap[0][0]:
            heapq.heappushpop(heap, (seconds, endpoint))
    return [(endpoint, seconds) for seconds, endpoint in sorted(heap, reverse=True)]


def fraction_below(sorted_latencies: list[float], threshold: float) -> float:
    """Return the fraction of latencies strictly below threshold.

    The list must already be sorted in ascending order. The search is O(log n).
    """
    if not sorted_latencies:
        return 0.0
    return bisect.bisect_left(sorted_latencies, threshold) / len(sorted_latencies)


class TaskQueue:
    """A priority queue where equal priorities run in first-in, first-out order."""

    def __init__(self) -> None:
        self._heap: list[tuple[int, int, str]] = []
        self._counter = itertools.count()  # a unique, increasing tie-breaker

    def push(self, priority: int, task: str) -> None:
        """Add a task. Lower numbers run first."""
        heapq.heappush(self._heap, (priority, next(self._counter), task))

    def pop(self) -> str:
        """Remove and return the next task.

        Raises:
            IndexError: If the queue is empty.
        """
        if not self._heap:
            raise IndexError("pop from an empty TaskQueue")
        priority, _, task = heapq.heappop(self._heap)
        logger.debug("Dequeued %r with priority %d", task, priority)
        return task

    def __len__(self) -> int:
        return len(self._heap)
```

Try it:

```python
stream = [
    ("/api/orders", 0.21), ("/api/users", 0.05), ("/api/search", 1.80),
    ("/api/orders", 0.95), ("/api/health", 0.01), ("/api/reports", 2.40),
]
print(top_k_slowest(stream, k=3))

latencies = sorted(seconds for _, seconds in stream)
print(f"{fraction_below(latencies, 0.5):.0%} of requests took under 0.5 s")

queue = TaskQueue()
queue.push(2, "write report")
queue.push(1, "fix outage")
queue.push(2, "answer email")
print([queue.pop() for _ in range(len(queue))])
```

```text
[('/api/reports', 2.4), ('/api/search', 1.8), ('/api/orders', 0.95)]
50% of requests took under 0.5 s
['fix outage', 'write report', 'answer email']
```

Compare the last line with the earlier experiment: the two priority-2 tasks now come out in the order they were added.

Why it is written this way:

- `top_k_slowest` never holds more than `k` items, so it works on a stream far larger than memory, and its heap stores `(seconds, endpoint)` so that the smallest duration sits on top and can be replaced.
- `fraction_below` replaces an O(n) scan with an O(log n) binary search once the data is sorted. Sorting happens once, outside.
- The `itertools.count()` tie-breaker guarantees that tuples never fall through to comparing the task text, and gives fairness among equal priorities.
- Each function states its complexity and its preconditions, such as "must already be sorted". Preconditions that cannot be checked cheaply belong in the documentation.

## Common mistakes

1. **Searching a list inside a loop** (`if x in big_list:`), which makes the whole program quadratic. Use a set or dictionary.
2. **Using `list.pop(0)` or `insert(0, x)` repeatedly** on large lists. Use a deque.
3. **Calling `bisect` on an unsorted list.** The result is meaningless.
4. **Expecting a heap to be sorted.** Only `heap[0]` is guaranteed to be the smallest. Pop to get sorted order.
5. **Looking for a max-heap in `heapq`.** There is none. Negate the values, or use `nlargest`.
6. **Pushing tuples whose second item cannot be compared.** Add a tie-breaker counter.
7. **Sorting a whole collection to find its smallest or largest few items.** Use `min`, `max`, or `heapq.nsmallest` and `nlargest`.
8. **Building a string with `+=` in a loop** and getting quadratic behavior in large cases. Use `"".join`.
9. **Optimizing without measuring.** Profile first (Chapter 11.5).
10. **Assuming Big-O gives actual seconds.** It describes growth. Constant factors and data size decide the winner on small inputs.
11. **Using `insort` in a hot loop on a huge list.** Each insertion is linear.
12. **Forgetting that "average O(1)" for dictionaries depends on a good hash.** Custom `__hash__` methods must be consistent with equality.

## Practice exercises

1. For each snippet, state the Big-O complexity in terms of the length n of `items`, and explain why: `items[0]`, `if x in items`, `sorted(items)`, `for a in items: for b in items: ...`, `items.append(x)`, `items.insert(0, x)`.
2. A function checks whether a list of n numbers contains duplicates by comparing every pair. What is its complexity? Rewrite it with a set and give the new complexity.
3. Use `timeit` to measure `x in some_list` for lists of 1,000, 10,000, and 100,000 items. Does the time grow as predicted? Repeat for a set.
4. Use `heapq` to implement heap sort: push all items, then pop them all. Compare the result with `sorted`.
5. Find the three most expensive items in a list of `(name, price)` tuples in two ways: with `sorted` and with `heapq.nlargest` using a `key`.
6. Merge three already-sorted lists into one sorted list using `heapq.merge`. Explain why it is better than concatenating and sorting when the inputs are huge.
7. Implement a max-heap by negation and use it to repeatedly extract the largest item from `[7, 2, 9, 4]`.
8. Use `bisect` to write `insert_sorted(items, value)` and check, after several insertions, that the list remains sorted. Explain the complexity of one insertion.
9. Write a shipping-fee function that uses `bisect` and two parallel lists (weight limits and prices) instead of `if/elif`. Test the boundaries carefully, including a weight exactly equal to a limit.
10. Maintain the **running median** of a stream of numbers using two heaps (a max-heap for the lower half and a min-heap for the upper half). Test with a small example you can verify by hand.
11. Write a function that finds whether any two numbers in a list add up to a target, first with two nested loops, then in linear time using a set. Compare timings on 5,000 numbers.
12. Extend `TaskQueue` with a `peek()` method that returns the next task without removing it, a `remove(task)` method that cancels a task lazily (by marking it as removed and skipping it on `pop`), and tests that show FIFO order among equal priorities is preserved.

## Summary

- Big-O notation describes how cost grows with input size n. The key classes are O(1), O(log n), O(n), O(n log n), and O(n squared).
- Average, worst, and amortized costs differ. `list.append` is O(1) amortized, dictionary and set operations are O(1) on average.
- Know the costs of built-ins: list indexing and `append` are fast, but `insert(0, x)`, `pop(0)`, `remove`, and `in` are O(n). Dictionaries and sets give constant-time lookups. Deques are fast at both ends.
- Nested loops multiply costs, and a hidden `in some_list` inside a loop is the most common cause of slow code.
- `heapq` provides a min-heap priority queue on a list: `heappush`, `heappop`, `heapify`, `nsmallest`, `nlargest`, and `merge`. Use a counter in the tuple to break ties and keep first-in, first-out order.
- `bisect` searches and inserts into **sorted** lists in logarithmic time. `insort` still costs O(n) for the shifting.
- Choose structures by how data is used, keep constant factors in mind for small inputs, and measure before optimizing.
