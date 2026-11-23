## Overview

Some collections are not read in arrival order or key order. They are read in priority order: the most urgent job first, the cheapest route first, the earliest timer first. A priority queue is a collection that always hands you the item with the highest priority (by convention, the smallest value) and lets you add new items at any time. The standard way to build one is a heap.

A heap is a binary tree with one simple rule: every node is at most as large as its children. That rule puts the smallest item at the root, where you can read it in constant time, and it is far weaker than the ordering rule of a binary search tree, which is exactly why it is cheap to maintain. A heap is stored in a plain array without any node objects, and inserting an item or removing the smallest costs `O(log n)`. Building a heap from `n` items costs only `O(n)`.

Heaps power an enormous range of algorithms and systems: task schedulers, event simulations, Dijkstra's shortest path algorithm, finding the top `K` of a large stream, merging many sorted files, and keeping a running median. Python provides the `heapq` module, so you will rarely write a heap yourself, but you need to understand how it works to use it correctly. This note covers the heap layout and the sift operations, `heapify`, `heapq` with its quirks (min-heap only, negation for a max-heap, tuples with tie-breakers), top-`K`, merging `K` sorted lists, two heaps for a running median, lazy deletion, and schedulers.

## Core concepts

### The heap property and the array layout

A min-heap is a complete binary tree in which every node's value is less than or equal to the values of its children. Because the tree is complete (every level is full except possibly the last, which fills from the left), it is stored in an array with no gaps, using the layout from the binary tree note: the children of index `i` are at `2i + 1` and `2i + 2`, and the parent is at `(i - 1) // 2`.

```text
index:   0   1   2   3   4   5   6
value:   1   3   2   7   4   8   9

              1
           /     \
         3         2
        / \       / \
       7   4     8   9
```

The root is always the minimum. There is no order between siblings, or between a node and its cousins, so a heap is not sorted. Only the parent-to-child relationship is guaranteed.

### Heap versus binary search tree

- A BST orders all keys: left is smaller, right is larger. That supports search, ordered traversal, and floor and ceiling queries, but needs a balancing scheme to stay efficient.
- A heap orders only parent to child. It supports "give me the minimum" and "add an item" in `O(log n)`, but finding an arbitrary item is `O(n)`.
- A heap is always balanced by construction, because its shape is a complete tree packed into an array.

### Sift up and sift down

The two operations that maintain the heap property.

- Sift up (push): append the new item at the end of the array, then repeatedly swap it with its parent while it is smaller than the parent. It climbs at most `log2(n)` levels.
- Sift down (pop): take the root as the result, move the last item of the array to the root, then repeatedly swap it with its smaller child while it is larger than that child. It sinks at most `log2(n)` levels.

### Heapify in linear time

To turn an arbitrary list into a heap, you could push the items one by one in `O(n log n)`. Faster: leave the array as it is and sift down every non-leaf node, starting from the last one and moving backwards to the root. Half the nodes are leaves and need no work, a quarter are one level above the leaves and sink at most one step, an eighth sink at most two steps, and so on. The total work is a series that sums to `O(n)`.

### heapq

Python's `heapq` module implements a min-heap on an ordinary list.

- `heappush(heap, item)` and `heappop(heap)`: `O(log n)`.
- `heapify(items)`: in place, `O(n)`.
- `heap[0]`: the smallest item, in `O(1)`.
- `heappushpop(heap, item)`: push, then pop the smallest, in one step that is faster than the two calls.
- `heapreplace(heap, item)`: pop the smallest, then push, in one step.
- `nlargest(k, items)` and `nsmallest(k, items)`: the `k` largest or smallest items, in `O(n log k)`.
- `merge(*sorted_iterables)`: lazily merges sorted inputs.

There is no max-heap. A common workaround is to store negated values.

### Tuples and tie-breakers

Store `(priority, item)` tuples to attach data to a priority. Tuples compare element by element, so when two priorities are equal, Python compares the items, which fails with `TypeError` if the items cannot be ordered. The standard fix is a unique tie-breaker between them: `(priority, sequence_number, item)`. A counter also makes items with equal priority come out in the order they were added.

### Top-K

To find the `k` largest items of a large collection or a stream, keep a min-heap of size `k`. The root is the smallest of the `k` best so far. A new item enters only if it beats the root, replacing it. The time is `O(n log k)` and the memory is `O(k)`, and the stream never has to fit in memory.

### Merging K sorted lists

Keep a heap holding the current front item of each list. Pop the smallest, output it, and push the next item from the same list. Each output item costs `O(log k)`, so merging `N` items in total takes `O(N log k)`. This is the heart of external merge sort and of merging search results from several shards.

### Two heaps for a running median

Keep the smaller half of the numbers in a max-heap and the larger half in a min-heap, with sizes that differ by at most one. The median is the root of the larger heap, or the average of the two roots when the sizes are equal. Each new number goes into one heap, and one item may move across to rebalance, so every insertion costs `O(log n)` and the median is read in `O(1)`.

### Lazy deletion

Removing an arbitrary item from a heap, or changing an item's priority, is not supported efficiently: finding the item takes `O(n)`. The standard workaround is lazy deletion. Do not remove the entry. Mark it as invalid, and when it reaches the top, discard it and pop again. Changing a priority is done by adding a new entry and invalidating the old one.

### Schedulers

A scheduler keeps jobs in a heap keyed by the time they should run. The next job is always at the root. A loop sleeps until that time, pops the job, runs it, and repeats. Delayed job queues, timers, retry schedulers, and discrete-event simulations all use this pattern.

## Basic syntax examples

### A min-heap from scratch

```python
from collections.abc import Iterable
from typing import Any, Protocol


class SupportsLessThan(Protocol):
    def __lt__(self, other: Any, /) -> bool: ...


class MinHeap[T: SupportsLessThan]:
    """A binary min-heap stored in a list. push and pop are O(log n)."""

    def __init__(self, items: Iterable[T] = ()) -> None:
        self._items: list[T] = list(items)
        for index in range(len(self._items) // 2 - 1, -1, -1):
            self._sift_down(index)  # heapify in O(n): last non-leaf back to the root

    def __len__(self) -> int:
        return len(self._items)

    def peek(self) -> T:
        if not self._items:
            raise IndexError("peek at an empty heap")
        return self._items[0]

    def push(self, item: T) -> None:
        self._items.append(item)
        self._sift_up(len(self._items) - 1)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("pop from an empty heap")
        smallest = self._items[0]
        last = self._items.pop()
        if self._items:
            self._items[0] = last  # move the last item to the root, then let it sink
            self._sift_down(0)
        return smallest

    def _sift_up(self, index: int) -> None:
        items = self._items
        item = items[index]
        while index > 0:
            parent = (index - 1) // 2
            if not item < items[parent]:
                break
            items[index] = items[parent]  # pull the parent down instead of swapping
            index = parent
        items[index] = item

    def _sift_down(self, index: int) -> None:
        items = self._items
        size = len(items)
        item = items[index]
        while True:
            child = 2 * index + 1
            if child >= size:
                break
            if child + 1 < size and items[child + 1] < items[child]:
                child += 1  # pick the smaller child
            if not items[child] < item:
                break
            items[index] = items[child]
            index = child
        items[index] = item


heap = MinHeap([5, 3, 8, 1, 9, 2])
heap.push(0)
print(heap.peek(), len(heap))  # 0 7
print([heap.pop() for _ in range(len(heap))])  # [0, 1, 2, 3, 5, 8, 9]
```

Trace of `push(1)` into the heap `[3, 5, 4, 8, 9]`:

```text
append 1:         [3, 5, 4, 8, 9, 1]      index 5, parent is index 2 (value 4)
1 < 4, move up:   [3, 5, 1, 8, 9, 4]      index 2, parent is index 0 (value 3)
1 < 3, move up:   [1, 5, 3, 8, 9, 4]      index 0 is the root, stop
```

Trace of `pop()` on `[1, 5, 3, 8, 9, 4]`:

```text
result is the root, 1; move the last item 4 to the root:   [4, 5, 3, 8, 9]
children 5 and 3: the smaller is 3, and 3 < 4, so swap:    [3, 5, 4, 8, 9]
index 2 has no children left in the array, so stop
```

### heapq basics

```python
import heapq

numbers = [5, 3, 8, 1, 9, 2]
heapq.heapify(numbers)  # in place, O(n)
print(numbers[0])  # 1, the minimum is always first
heapq.heappush(numbers, 0)
print(heapq.heappop(numbers))  # 0
print(heapq.heappushpop(numbers, 7))  # 1, push 7 then pop the smallest
print(heapq.heapreplace(numbers, 4))  # 2, pop the smallest then push 4

scores = [88, 92, 79, 95, 85, 91, 70]
print(heapq.nlargest(3, scores))  # [95, 92, 91]
print(heapq.nsmallest(2, scores))  # [70, 79]
```

### A max-heap with negation

```python
import heapq

max_heap: list[int] = []
for value in (5, 3, 8, 1):
    heapq.heappush(max_heap, -value)  # store negatives so the largest comes first
print(-heapq.heappop(max_heap))  # 8
print(-max_heap[0])  # 5, the next largest
```

### Tuples and tie-breakers

```python
import heapq
from dataclasses import dataclass
from itertools import count


@dataclass
class Job:
    name: str  # a Job cannot be ordered: there is no __lt__


tie_breaker = count()
queue: list[tuple[int, int, Job]] = []
for priority, name in ((2, "backup"), (1, "email"), (2, "report"), (1, "billing")):
    heapq.heappush(queue, (priority, next(tie_breaker), Job(name)))

order = [heapq.heappop(queue)[2].name for _ in range(len(queue))]
print(order)  # ['email', 'billing', 'backup', 'report']
```

Without the counter, two entries with equal priority would force Python to compare two `Job` objects and raise `TypeError`. With it, equal priorities come out in the order they were added.

### Top-K from a stream

```python
import heapq
from collections.abc import Iterable


def top_k_largest(stream: Iterable[int], k: int) -> list[int]:
    """The k largest items, largest first. O(n log k) time, O(k) memory."""
    if k < 1:
        raise ValueError("k must be at least 1")
    best: list[int] = []  # a min-heap holding the k largest items seen so far
    for value in stream:
        if len(best) < k:
            heapq.heappush(best, value)
        elif value > best[0]:
            heapq.heapreplace(best, value)  # drop the smallest, keep the new one
    return sorted(best, reverse=True)


print(top_k_largest([4, 1, 7, 3, 9, 2, 8], 3))  # [9, 8, 7]
```

### Merging K sorted lists

```python
import heapq


def merge_sorted_lists(lists: list[list[int]]) -> list[int]:
    """Merge k sorted lists in O(N log k) for N items in total."""
    heap: list[tuple[int, int, int]] = []  # (value, list index, position in that list)
    for list_index, values in enumerate(lists):
        if values:
            heapq.heappush(heap, (values[0], list_index, 0))
    merged: list[int] = []
    while heap:
        value, list_index, position = heapq.heappop(heap)
        merged.append(value)
        if position + 1 < len(lists[list_index]):
            next_value = lists[list_index][position + 1]
            heapq.heappush(heap, (next_value, list_index, position + 1))
    return merged


inputs = [[1, 4, 7], [2, 5], [3, 6, 8, 9]]
print(merge_sorted_lists(inputs))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
print(list(heapq.merge(*inputs)))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### A running median with two heaps

```python
import heapq


class RunningMedian:
    """add is O(log n) and median is O(1)."""

    def __init__(self) -> None:
        self._low: list[float] = []  # max-heap (negated) holding the smaller half
        self._high: list[float] = []  # min-heap holding the larger half

    def add(self, value: float) -> None:
        if not self._low or value <= -self._low[0]:
            heapq.heappush(self._low, -value)
        else:
            heapq.heappush(self._high, value)
        # Keep the sizes equal, or with one extra item in the low half.
        if len(self._low) > len(self._high) + 1:
            heapq.heappush(self._high, -heapq.heappop(self._low))
        elif len(self._high) > len(self._low):
            heapq.heappush(self._low, -heapq.heappop(self._high))

    def median(self) -> float:
        if not self._low:
            raise ValueError("no values have been added")
        if len(self._low) > len(self._high):
            return -self._low[0]
        return (-self._low[0] + self._high[0]) / 2


median = RunningMedian()
for number in (5, 2, 8, 1, 9):
    median.add(number)
    print(median.median(), end=" ")  # 5 3.5 5 3.5 5
print()
```

### Lazy deletion and changing priorities

```python
import heapq
from collections.abc import Hashable
from itertools import count


class TaskQueue[T: Hashable]:
    """A priority queue where tasks can be removed or reprioritized."""

    def __init__(self) -> None:
        self._heap: list[tuple[int, int, T]] = []
        self._live: dict[T, tuple[int, int]] = {}  # task -> (priority, sequence)
        self._counter = count()

    def __len__(self) -> int:
        return len(self._live)

    def add(self, task: T, priority: int) -> None:
        """Add a task, or change its priority if it is already queued."""
        sequence = next(self._counter)
        self._live[task] = (priority, sequence)  # any older entry becomes stale
        heapq.heappush(self._heap, (priority, sequence, task))

    def remove(self, task: T) -> bool:
        return self._live.pop(task, None) is not None  # its heap entry is now stale

    def pop(self) -> T:
        while self._heap:
            priority, sequence, task = heapq.heappop(self._heap)
            if self._live.get(task) == (priority, sequence):  # skip stale entries
                del self._live[task]
                return task
        raise KeyError("pop from an empty queue")


tasks = TaskQueue[str]()
tasks.add("write report", 2)
tasks.add("fix outage", 1)
tasks.add("lunch", 3)
tasks.add("lunch", 0)  # raise the priority of an existing task
tasks.remove("write report")
print([tasks.pop() for _ in range(len(tasks))])  # ['lunch', 'fix outage']
```

## Deeper behavior

### Why the operations cost O(log n)

A complete binary tree with `n` nodes has height `floor(log2(n))`. A sift moves an item along a single root-to-leaf path, doing a constant amount of work per level, so push and pop are `O(log n)`. Reading the minimum is `O(1)` because it is the root. Both operations preserve the shape (still complete) and restore the heap property along one path only, which is why a heap can be cheap to maintain: it enforces much less than a sorted order.

### Why heapify is O(n)

Pushing `n` items one by one costs `O(n log n)`, because late pushes climb a long way. Heapify instead sifts down from the bottom: nodes at height `h` sink at most `h` levels, and there are at most `n / 2^(h + 1)` nodes at height `h`. The total work is at most the sum over `h` of `h * n / 2^(h + 1)`, and that series sums to at most `n`. Most of the nodes are near the bottom, where sifting is cheap. This is why `heapq.heapify` and `MinHeap(items)` are the right way to build a heap from existing data, and why `heapq.nlargest` can use it.

### How heapq implements pop

CPython's C implementation does something a little different from the textbook sift down. It moves the last item to the root, then moves the hole all the way down to a leaf by always promoting the smaller child, without comparing the moved item at each level, and finally sifts the item up from there. On average that does fewer comparisons, which matters when comparing Python objects is slow. The result is the same heap.

### No decrease-key

Algorithms such as Dijkstra's shortest path want to lower the priority value of an item already in the heap. A textbook heap supports this with an index from item to position. `heapq` does not keep such an index, so you use lazy deletion: push a new entry with the better priority, and ignore the older entry when it eventually reaches the top. The cost is that the heap may hold stale entries, up to the number of pushes. If many are stale, rebuild the heap from the live entries with `heapify`, as the industry snippet does.

### Stability and ordering

A heap is not stable: equal priorities can come out in any order. The unique counter in `(priority, counter, item)` turns equal priorities into first in, first out. If your priorities are floats, make sure they are not `NaN`, which breaks every comparison.

### Choosing between heap, sorted list, and tree

- Unsorted list: push `O(1)`, take the minimum `O(n)`.
- Sorted list: push `O(n)` (shifting), take the minimum `O(1)` from the front or `O(1)` from the back.
- Heap: push `O(log n)`, take the minimum `O(log n)`, peek `O(1)`, build from `n` items `O(n)`.
- Balanced tree: every operation `O(log n)`, and it also supports finding or deleting any key, ordered traversal, and range queries.

Choose a heap when you only ever need the extreme item. Choose a balanced tree or `sortedcontainers` when you need more than that.

### Other kinds of heaps

A `d`-ary heap uses `d` children per node, which makes the tree shallower and push cheaper at the cost of a more expensive pop. Fibonacci heaps offer amortized `O(1)` decrease-key, which is better in theory for graph algorithms, but their large constant factors mean binary heaps usually win in practice. A pairing heap is simpler and often fast in practice. For Python, `heapq` with lazy deletion is the standard answer.

### Top-K, precisely

`heapq.nlargest(k, items)` is `O(n log k)`. When `k` is close to `n`, sorting the whole input is faster. When `k` is 1, `max` is faster. When items arrive one at a time and you must always know the current top `k`, keep your own heap, as in `top_k_largest`. For the `k`-th largest only, a quickselect is `O(n)` on average, as covered in the selection note.

### Time-based schedulers

A scheduler heap keyed by timestamp relies on a clock that does not go backwards. Use `time.monotonic()` for in-process timers, because the wall clock can jump when it is adjusted. Stored schedules that survive restarts must use wall-clock timestamps, since a monotonic clock restarts from an arbitrary value, and the code that loads them has to cope with jobs whose time has already passed.

### Memory and stale entries

A heap with lazy deletion can grow if many tasks are cancelled and never reach the top. Track the number of live entries, and when the heap holds far more entries than live tasks, rebuild it from the live ones. Rebuilding costs `O(live)` and is cheap compared with letting memory grow.

## Industry-level snippet

A notification service sends reminders at scheduled times: a payment reminder in 24 hours, a retry in 30 seconds, a trial expiry notice next week. Jobs are stored in a database so they survive restarts, but scanning the table for due jobs every second would be wasteful. The service keeps an in-memory heap keyed by run time, and reads the database only to schedule, cancel, and mark jobs.

The design uses the techniques above. A heap of `(run_at, sequence, job_id)` entries gives the next due job in `O(1)` and removes it in `O(log n)`. Cancelling a job is lazy: the database row is marked cancelled and the heap entry is left in place, to be skipped when it reaches the top. A dictionary of live jobs decides which entries are stale, and when stale entries outnumber live ones, the heap is rebuilt with `heapify`.

Claiming due jobs is the delicate step. The jobs are popped from the heap, and then marked as dispatched in the database in a single transaction. If the transaction fails, the popped entries are pushed back, so a database error never loses a job. Only after the commit succeeds does the queue forget them.

```python
# delayed_jobs.py
import heapq
import logging
import sqlite3
from dataclasses import dataclass
from itertools import count

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS delayed_job (
    job_id INTEGER PRIMARY KEY,
    run_at REAL NOT NULL,
    payload TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'dispatched', 'cancelled'))
)
"""

INSERT_SQL = "INSERT INTO delayed_job (job_id, run_at, payload) VALUES (?, ?, ?)"

MAX_CLAIM = 500


class DuplicateJobError(ValueError):
    """Raised when a job id has already been used."""


@dataclass(frozen=True, slots=True)
class DueJob:
    job_id: int
    run_at: float
    payload: str


class DelayedJobQueue:
    """A heap of pending jobs in front of a table of jobs. One writer process."""

    def __init__(self, connection: sqlite3.Connection) -> None:
        self._connection = connection
        self._heap: list[tuple[float, int, int]] = []  # (run_at, sequence, job_id)
        self._pending: dict[int, tuple[float, int]] = {}  # job_id -> (run_at, sequence)
        self._counter = count()
        rows = connection.execute(
            "SELECT job_id, run_at FROM delayed_job WHERE status = 'pending'"
        )
        for job_id, run_at in rows:
            sequence = next(self._counter)
            self._pending[job_id] = (float(run_at), sequence)
            self._heap.append((float(run_at), sequence, job_id))
        heapq.heapify(self._heap)  # O(n), not n pushes

    @property
    def pending_count(self) -> int:
        return len(self._pending)

    @property
    def heap_size(self) -> int:
        """Entries in the heap, including stale ones that are waiting to be skipped."""
        return len(self._heap)

    def schedule(self, job_id: int, run_at: float, payload: str) -> None:
        if job_id in self._pending:
            raise DuplicateJobError(f"job {job_id} is already scheduled")
        try:
            with self._connection:
                self._connection.execute(INSERT_SQL, (job_id, run_at, payload))
        except sqlite3.IntegrityError as error:
            raise DuplicateJobError(f"job {job_id} has already been used") from error
        except sqlite3.Error:
            logger.exception("could not schedule job %d", job_id)
            raise
        sequence = next(self._counter)
        self._pending[job_id] = (run_at, sequence)
        heapq.heappush(self._heap, (run_at, sequence, job_id))

    def cancel(self, job_id: int) -> bool:
        if job_id not in self._pending:
            return False
        try:
            with self._connection:
                self._connection.execute(
                    "UPDATE delayed_job SET status = 'cancelled' "
                    "WHERE job_id = ? AND status = 'pending'",
                    (job_id,),
                )
        except sqlite3.Error:
            logger.exception("could not cancel job %d", job_id)
            raise
        del self._pending[job_id]  # the heap entry is now stale: lazy deletion
        if len(self._heap) > 2 * len(self._pending) + 16:
            self._compact()
        return True

    def _compact(self) -> None:
        """Rebuild the heap from live entries only, dropping the stale ones."""
        self._heap = [
            (run_at, sequence, job_id)
            for job_id, (run_at, sequence) in self._pending.items()
        ]
        heapq.heapify(self._heap)

    def _is_live(self, entry: tuple[float, int, int]) -> bool:
        run_at, sequence, job_id = entry
        return self._pending.get(job_id) == (run_at, sequence)

    def next_run_at(self) -> float | None:
        """When the next live job is due, or None. Discards stale entries on top."""
        while self._heap and not self._is_live(self._heap[0]):
            heapq.heappop(self._heap)
        return self._heap[0][0] if self._heap else None

    def claim_due(self, now: float, limit: int = 100) -> list[DueJob]:
        """Take up to `limit` jobs whose time has come and mark them dispatched."""
        if not 1 <= limit <= MAX_CLAIM:
            raise ValueError(f"limit must be between 1 and {MAX_CLAIM}")
        taken: list[tuple[float, int, int]] = []
        while self._heap and len(taken) < limit and self._heap[0][0] <= now:
            entry = heapq.heappop(self._heap)
            if self._is_live(entry):
                taken.append(entry)  # stale entries are simply dropped
        if not taken:
            return []
        ids = [job_id for _, _, job_id in taken]
        marks = ", ".join("?" for _ in ids)  # only placeholders go into the SQL text
        select_sql = (
            f"SELECT job_id, payload FROM delayed_job WHERE job_id IN ({marks})"
        )
        try:
            with self._connection:
                rows = self._connection.execute(select_sql, ids).fetchall()
                self._connection.executemany(
                    "UPDATE delayed_job SET status = 'dispatched' "
                    "WHERE job_id = ? AND status = 'pending'",
                    [(job_id,) for job_id in ids],
                )
        except sqlite3.Error:
            for entry in taken:  # nothing was dispatched, so put every job back
                heapq.heappush(self._heap, entry)
            logger.exception("could not claim %d due jobs", len(taken))
            raise
        payloads = {row[0]: row[1] for row in rows}
        for _, _, job_id in taken:
            del self._pending[job_id]  # forget them only after the commit
        return [DueJob(job_id, run_at, payloads[job_id]) for run_at, _, job_id in taken]
```

```python
# test_delayed_jobs.py
import sqlite3
from collections.abc import Iterator

import pytest

from delayed_jobs import SCHEMA_SQL, DelayedJobQueue, DueJob, DuplicateJobError


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA_SQL)
    yield conn
    conn.close()


def ids(jobs: list[DueJob]) -> list[int]:
    return [job.job_id for job in jobs]


def test_jobs_come_out_in_time_order_and_only_when_due(
    connection: sqlite3.Connection,
) -> None:
    queue = DelayedJobQueue(connection)
    queue.schedule(1, 300.0, "c")
    queue.schedule(2, 100.0, "a")
    queue.schedule(3, 200.0, "b")
    assert queue.next_run_at() == 100.0
    assert ids(queue.claim_due(50.0)) == []
    assert [job.payload for job in queue.claim_due(250.0)] == ["a", "b"]
    assert ids(queue.claim_due(250.0)) == []
    assert ids(queue.claim_due(1_000.0)) == [1]
    assert queue.pending_count == 0
    assert queue.next_run_at() is None


def test_equal_times_run_in_scheduling_order(connection: sqlite3.Connection) -> None:
    queue = DelayedJobQueue(connection)
    for job_id in (7, 3, 9):
        queue.schedule(job_id, 10.0, f"job {job_id}")
    assert ids(queue.claim_due(10.0)) == [7, 3, 9]


def test_limit_caps_the_batch(connection: sqlite3.Connection) -> None:
    queue = DelayedJobQueue(connection)
    for job_id in range(10):
        queue.schedule(job_id, float(job_id), "x")
    assert ids(queue.claim_due(100.0, limit=3)) == [0, 1, 2]
    assert ids(queue.claim_due(100.0, limit=100)) == [3, 4, 5, 6, 7, 8, 9]


def test_cancelled_jobs_are_skipped(connection: sqlite3.Connection) -> None:
    queue = DelayedJobQueue(connection)
    queue.schedule(1, 10.0, "a")
    queue.schedule(2, 20.0, "b")
    assert queue.cancel(1) is True
    assert queue.cancel(1) is False
    assert queue.cancel(99) is False
    assert queue.next_run_at() == 20.0
    assert ids(queue.claim_due(100.0)) == [2]
    status = connection.execute("SELECT status FROM delayed_job WHERE job_id = 1")
    assert status.fetchone()[0] == "cancelled"


def test_the_heap_is_compacted_when_most_entries_are_stale(
    connection: sqlite3.Connection,
) -> None:
    queue = DelayedJobQueue(connection)
    for job_id in range(100):
        queue.schedule(job_id, float(job_id), "x")
    for job_id in range(90):
        queue.cancel(job_id)
    assert queue.pending_count == 10
    assert queue.heap_size <= 2 * queue.pending_count + 16
    assert ids(queue.claim_due(1_000.0)) == list(range(90, 100))


def test_job_ids_cannot_be_reused(connection: sqlite3.Connection) -> None:
    queue = DelayedJobQueue(connection)
    queue.schedule(1, 10.0, "a")
    with pytest.raises(DuplicateJobError):
        queue.schedule(1, 20.0, "again")
    queue.claim_due(100.0)
    with pytest.raises(DuplicateJobError):
        queue.schedule(1, 30.0, "after dispatch")


def test_a_new_queue_resumes_from_the_database(connection: sqlite3.Connection) -> None:
    first = DelayedJobQueue(connection)
    first.schedule(1, 10.0, "a")
    first.schedule(2, 20.0, "b")
    first.schedule(3, 30.0, "c")
    first.cancel(2)
    first.claim_due(15.0)
    resumed = DelayedJobQueue(connection)
    assert resumed.pending_count == 1
    assert ids(resumed.claim_due(100.0)) == [3]


def test_a_failed_claim_puts_the_jobs_back(connection: sqlite3.Connection) -> None:
    queue = DelayedJobQueue(connection)
    queue.schedule(1, 10.0, "a")
    queue.schedule(2, 20.0, "b")
    connection.execute("DROP TABLE delayed_job")
    with pytest.raises(sqlite3.OperationalError):
        queue.claim_due(100.0)
    assert queue.pending_count == 2
    assert queue.heap_size == 2
    assert queue.next_run_at() == 10.0


@pytest.mark.parametrize("limit", [0, -1, 501])
def test_invalid_limits_are_rejected(
    connection: sqlite3.Connection, limit: int
) -> None:
    with pytest.raises(ValueError):
        DelayedJobQueue(connection).claim_due(1.0, limit=limit)
```

A stale heap entry costs only a little memory until it is skipped. The compaction rule keeps that cost bounded, so a service that cancels most of its jobs does not slowly leak memory.

## Common mistakes

- Assuming `heapq` is a max-heap. It is a min-heap. Negate the priorities, or use a wrapper, for largest-first.
- Treating a heap list as sorted. Only `heap[0]` is guaranteed to be the smallest, so do not print the list or loop over it expecting order. Pop to get sorted output.
- Pushing items that cannot be compared when priorities tie, which raises `TypeError`. Add a unique counter between the priority and the item.
- Calling `heappush` on a list that was never turned into a heap. The list must be a heap already, or start empty. Call `heapify` on existing data.
- Modifying an item's priority in place inside the heap, which silently breaks the heap property.
- Searching a heap for an item with `in`, which is `O(n)`. Keep a dictionary of items to entries.
- Building a heap with `n` pushes when `heapify` does it in `O(n)`.
- Using `heapq.nlargest` for `k` close to `n`, or in a loop for repeated queries, when a sort or a persistent heap is better.
- Forgetting that lazy deletion leaves stale entries, so the heap grows and `len(heap)` is not the number of live items.
- Using the wall clock for in-process timers, so a clock adjustment makes jobs fire early or late.
- Removing the popped entries from memory before the database commit succeeds, so a failure loses jobs.
- Comparing floats that might be `NaN`, which corrupts the heap order.

## Practice exercises

1. Draw the heap that results from pushing `7, 3, 9, 1, 5` one by one into an empty min-heap, and then the heap `heapify` builds from the list `[7, 3, 9, 1, 5]`. Explain why they differ.
2. Write `is_min_heap(values)` that checks the heap property of a list using the index formulas, and test it on heaps built with `heapq`.
3. Write `heap_sort(values)` that sorts in place using a max-heap built by sifting down, with `O(1)` extra space, and verify it against `sorted`.
4. Write `kth_largest_in_stream` as a class with `add(value)` that returns the current `k`-th largest, using a min-heap of size `k`.
5. Write `merge_k_sorted_files(paths)` that merges several sorted text files line by line using `heapq.merge`, and explain why it uses `O(k)` memory.
6. Write `last_stone_weight(stones)` that repeatedly smashes the two heaviest stones and returns what is left, using a max-heap by negation.
7. Write `task_scheduler_idle_time(tasks, cooldown)` that finds the minimum time to run tasks when the same task must wait for a cooldown, using a heap of counts.
8. Write `reorganize_string(text)` that rearranges letters so that no two adjacent letters are equal, using a heap of letter counts, or reports that it is impossible.
9. Write `find_median_stream` using `RunningMedian`, and test it against `statistics.median` on random inputs.
10. Write `smallest_range_covering_lists(lists)` that finds the smallest interval containing at least one number from each of `k` sorted lists, using a min-heap and a running maximum.
11. Write `dijkstra(graph, source)` for a graph with non-negative weights using `heapq` and lazy deletion, and explain how stale entries are handled.
12. Write `PriorityQueueWithDecreaseKey` that keeps a dictionary of item positions and updates it in `sift_up` and `sift_down`, so priorities can be changed in `O(log n)` without stale entries.
13. Write `sliding_window_median(values, k)` using two heaps with lazy deletion, and describe how you keep the two heaps balanced when you remove an item.
14. Extend `DelayedJobQueue` with `reschedule(job_id, new_run_at)` that updates the database in a transaction, then pushes a new heap entry and makes the old one stale, and add tests for it.

## Summary

A heap is a complete binary tree in which every node is no larger than its children, stored in an array where the children of index `i` are at `2i + 1` and `2i + 2`. The smallest item is always at the root, push and pop cost `O(log n)` using sift up and sift down, and `heapify` builds a heap from `n` items in `O(n)`. Python's `heapq` is a min-heap on a list: negate values for a max-heap, add a unique counter to tuples so ties never compare the items, and remember that only `heap[0]` is ordered. Heaps give top-`K` in `O(n log k)` time and `O(k)` memory, merge `k` sorted streams in `O(N log k)`, track a running median with two heaps, and drive schedulers keyed by time. Because arbitrary removal and priority changes are not supported efficiently, use lazy deletion with a live-entry dictionary, compact the heap when stale entries pile up, and in a service that stores state in a database, put jobs back if the database commit fails.
