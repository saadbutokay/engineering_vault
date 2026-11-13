## Overview

A queue is a collection where the first item added is the first item removed, a rule called first in, first out (FIFO). It is the line at a ticket counter: people join at the back and are served from the front. Queues appear wherever work must be handled in arrival order: print jobs, web requests waiting for a worker, messages between services, tasks in a scheduler, and the frontier of a breadth-first search.

A deque (double-ended queue, pronounced "deck") lets you add and remove at both ends in constant time. It can behave as a queue, as a stack, or as a sliding window, and in Python it is the standard tool for all three. The most common mistake with queues in Python is to use a `list` and call `pop(0)`, which shifts every remaining item and turns a linear algorithm into a quadratic one.

Queues also matter in system design, not only in algorithms. A queue between a fast producer and a slow consumer smooths out bursts, but an unbounded queue hides overload until memory runs out. Bounded queues, which make the producer wait or reject work when the queue is full, implement back-pressure. This note covers FIFO behavior, `collections.deque` and why `list.pop(0)` is wrong, circular buffers, a queue built from two stacks, a monotonic deque for sliding window maximum, `queue.Queue` and `queue.PriorityQueue` for threads, and bounded queues with back-pressure.

## Core concepts

### FIFO and the two operations

- Enqueue: add an item at the back.
- Dequeue: remove and return the item at the front.
- Peek: look at the front item without removing it.

All three should be `O(1)`. A good queue makes them so.

### The deque

`collections.deque` supports `append` and `pop` at the right end and `appendleft` and `popleft` at the left end, all in `O(1)`. Internally it is a doubly linked list of fixed-size blocks, so growing and shrinking at either end never shifts items. Indexing in the middle is `O(n)`, because you must walk the blocks. A deque can also have a maximum length: once full, adding at one end silently drops an item from the other.

### Why `list.pop(0)` is wrong

A list is an array. Removing the first item shifts all the others one position left, which costs `O(n)`. A loop that dequeues `n` items from a list does `n + (n - 1) + ... + 1` shifts, or `O(n^2)` work. With `deque.popleft()` the same loop is `O(n)`. For a queue of 100,000 items the difference is milliseconds versus many seconds.

### Circular buffers

A circular buffer (ring buffer) is a fixed-size array used as a queue. Two indexes track where the next item is read (`head`) and how many items are stored. New items are written at `(head + size) % capacity`, and the index arithmetic wraps around the end of the array back to the beginning. It uses constant memory, never shifts items, and every operation is `O(1)`. It is the basis of audio buffers, network drivers, and bounded log storage.

### A queue from two stacks

Two stacks can implement a FIFO queue. New items are pushed onto an input stack. When you need to dequeue and the output stack is empty, pour every item from the input stack into the output stack, which reverses their order so the oldest item ends up on top. Each item is moved at most once, so the cost per operation is `O(1)` amortized.

### Monotonic deque and sliding window maximum

To find the maximum of every window of `k` consecutive items, keep a deque of indexes whose values are in decreasing order. When a new item arrives, drop from the back every index with a value not larger than it, because those can never be a maximum while the new item is in the window. Drop from the front any index that has left the window. The front of the deque is always the maximum of the current window. Each index is added and removed at most once, so the total is `O(n)`.

### queue.Queue and queue.PriorityQueue

The `queue` module provides queues for threads. `queue.Queue` is a thread-safe FIFO queue with blocking `put` and `get`, an optional size limit, and `task_done` and `join` to wait until all work is finished. `queue.PriorityQueue` always returns the smallest item first and is built on a heap. `queue.LifoQueue` is the thread-safe stack. Do not use these inside single-threaded code, because the locking makes them slower than `deque`.

### Bounded queues and back-pressure

An unbounded queue accepts work forever. If producers are faster than consumers, it grows until the process runs out of memory, and every item waits longer and longer. A bounded queue has a maximum size. When it is full, the producer must wait (block), fail fast (raise an error or return a "try again later" response), or drop something. This pushback toward the producer is called back-pressure, and it keeps a system stable under overload.

## Basic syntax examples

### deque operations

```python
from collections import deque

queue: deque[str] = deque()
queue.append("a")  # enqueue at the right
queue.append("b")
queue.append("c")
print(queue[0])  # a, peek at the front
print(queue.popleft())  # a, dequeue from the left in O(1)
print(list(queue))  # ['b', 'c']

queue.appendleft("z")  # a deque also works at the left end
print(queue.pop())  # c, remove from the right end
print(list(queue))  # ['z', 'b']

window: deque[int] = deque(maxlen=3)
for value in range(6):
    window.append(value)  # the oldest item is dropped automatically when full
print(list(window))  # [3, 4, 5]
```

### Why list.pop(0) is slow

```python
import timeit
from collections import deque


def drain_list(n: int) -> None:
    items = list(range(n))
    while items:
        items.pop(0)  # O(n) each time


def drain_deque(n: int) -> None:
    items = deque(range(n))
    while items:
        items.popleft()  # O(1) each time


size = 50_000
list_time = min(timeit.repeat(lambda: drain_list(size), number=1, repeat=3))
deque_time = min(timeit.repeat(lambda: drain_deque(size), number=1, repeat=3))
print(f"list: {list_time:.3f}s, deque: {deque_time:.3f}s")
```

### Breadth-first order with a queue

```python
from collections import deque


def bfs_order(graph: dict[str, list[str]], start: str) -> list[str]:
    """Visit nodes level by level. Each node is queued once."""
    seen = {start}
    order: list[str] = []
    queue = deque([start])
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph.get(node, []):
            if neighbor not in seen:
                seen.add(neighbor)  # mark when queued, not when dequeued
                queue.append(neighbor)
    return order


graph = {"a": ["b", "c"], "b": ["d"], "c": ["d"], "d": []}
print(bfs_order(graph, "a"))  # ['a', 'b', 'c', 'd']
```

### A circular buffer

```python
class CircularBuffer[T]:
    """A fixed-capacity FIFO queue on top of a preallocated list."""

    def __init__(self, capacity: int) -> None:
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        self._slots: list[T | None] = [None] * capacity
        self._head = 0  # index of the oldest item
        self._size = 0

    def __len__(self) -> int:
        return self._size

    @property
    def is_full(self) -> bool:
        return self._size == len(self._slots)

    def enqueue(self, item: T) -> None:
        if self.is_full:
            raise OverflowError("buffer is full")
        tail = (self._head + self._size) % len(self._slots)  # wrap around the end
        self._slots[tail] = item
        self._size += 1

    def dequeue(self) -> T:
        if self._size == 0:
            raise IndexError("dequeue from empty buffer")
        item = self._slots[self._head]
        self._slots[self._head] = None  # release the reference
        self._head = (self._head + 1) % len(self._slots)
        self._size -= 1
        assert item is not None
        return item


buffer = CircularBuffer[int](3)
for value in (1, 2, 3):
    buffer.enqueue(value)
print(buffer.dequeue(), buffer.dequeue())  # 1 2
buffer.enqueue(4)  # reuses the first slot
buffer.enqueue(5)
print(buffer.dequeue(), buffer.dequeue(), buffer.dequeue())  # 3 4 5
```

State of the buffer (capacity 3) during the run:

```text
after 1, 2, 3 enqueued:  slots [1, 2, 3]   head 0, size 3
after two dequeues:      slots [-, -, 3]   head 2, size 1
enqueue 4:               tail = (2 + 1) % 3 = 0    slots [4, -, 3]
enqueue 5:               tail = (2 + 2) % 3 = 1    slots [4, 5, 3]
three dequeues give 3, 4, 5 in order
```

### A queue from two stacks

```python
class QueueFromStacks[T]:
    """FIFO queue built from two stacks. Amortized O(1) per operation."""

    def __init__(self) -> None:
        self._inbox: list[T] = []
        self._outbox: list[T] = []

    def enqueue(self, item: T) -> None:
        self._inbox.append(item)

    def dequeue(self) -> T:
        if not self._outbox:
            while self._inbox:  # reverse the order by pouring one stack into the other
                self._outbox.append(self._inbox.pop())
        if not self._outbox:
            raise IndexError("dequeue from empty queue")
        return self._outbox.pop()


queue_from_stacks = QueueFromStacks[int]()
for value in (1, 2, 3):
    queue_from_stacks.enqueue(value)
print(queue_from_stacks.dequeue())  # 1
queue_from_stacks.enqueue(4)
print(queue_from_stacks.dequeue(), queue_from_stacks.dequeue())  # 2 3
```

### Sliding window maximum with a monotonic deque

```python
from collections import deque


def sliding_window_max(values: list[int], k: int) -> list[int]:
    """Maximum of every window of k consecutive items. O(n) time."""
    if not 1 <= k <= len(values):
        raise ValueError("k must be between 1 and len(values)")
    candidates: deque[int] = deque()  # indexes; values decrease front to back
    result: list[int] = []
    for index, value in enumerate(values):
        while candidates and values[candidates[-1]] <= value:
            candidates.pop()  # smaller items can never be the maximum again
        candidates.append(index)
        if candidates[0] <= index - k:
            candidates.popleft()  # the front item left the window
        if index >= k - 1:
            result.append(values[candidates[0]])
    return result


print(sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3))  # [3, 3, 5, 5, 6, 7]
```

### Thread-safe queues from the queue module

```python
import queue

jobs: queue.Queue[str] = queue.Queue(maxsize=2)
jobs.put("first")
jobs.put("second")
try:
    jobs.put("third", block=False)  # the queue is full
except queue.Full:
    print("queue full")  # queue full

print(jobs.get(), jobs.qsize())  # first 1
jobs.task_done()

priorities: queue.PriorityQueue[tuple[int, str]] = queue.PriorityQueue()
priorities.put((2, "send newsletter"))
priorities.put((1, "charge card"))
priorities.put((3, "clean up"))
print(priorities.get())  # (1, 'charge card')
```

## Deeper behavior

### What a deque costs

A deque is built from blocks of 64 slots linked in a chain. Adding at either end touches only the end block and sometimes allocates a new one, so it is `O(1)` and never copies existing items. Walking from the ends to index `i` visits about `i / 64` blocks, so indexing close to either end is fast, and indexing the middle of a large deque is `O(n)`. If you need fast random access, use a list. If you need fast operations at both ends, use a deque.

`deque(maxlen=n)` is the standard way to keep "the last `n` items": appending to a full deque discards the item at the opposite end. It is how you keep recent log lines, the last `k` samples for a moving average, or an undo history of bounded size.

### Amortized cost of the two-stack queue

An item is pushed onto the inbox once, moved to the outbox once, and popped from the outbox once. That is three constant-time operations per item over its lifetime. A single `dequeue` may move many items at once, but those moves are paid for by earlier `enqueue` calls, so any sequence of `n` operations costs `O(n)` in total. The worst case of one operation is `O(n)`, so this is amortized, not worst-case, `O(1)`.

### Circular buffer details

- Full versus empty: if you track only `head` and `tail`, an empty buffer and a full one both have `head == tail`. Keeping a `size` counter, as above, removes the ambiguity.
- Overwrite mode: for logs and audio, a full buffer often overwrites the oldest item instead of failing. That is `deque(maxlen=...)` behavior.
- Modulo arithmetic: Python's `%` is always non-negative, so `(index - 1) % capacity` works for stepping backwards.
- Cache behavior: a ring buffer reuses the same block of memory, which is why it is common in performance-critical code.

### The monotonic deque, why it is right

The deque holds indexes in increasing order and their values in decreasing order. Its front is the maximum of the current window. When a new value arrives, every earlier item with a smaller or equal value is both older and smaller, so it will leave the window earlier and can never again be the answer. Removing them keeps the deque sorted by value. Removing the front when it falls out of the window keeps all indexes inside the window. Each index is appended once and removed once, so the loop body does `O(1)` amortized work.

### queue.Queue under the hood

`queue.Queue` wraps a `deque` with a lock and two condition variables, one that signals "not empty" to consumers and one that signals "not full" to producers. `put` blocks while the queue is full, and `get` blocks while it is empty. Both accept a `timeout`. `task_done()` and `join()` let a producer wait until every item that was put has been fully processed, which is how you wait for a batch of work to finish. Each `get()` should be matched by exactly one `task_done()`, or `join()` will wait forever.

The cost of this thread safety is a lock acquisition for every operation, so use `collections.deque` directly in single-threaded code. Individual `deque.append` and `deque.popleft` calls are thread-safe in CPython, but a sequence like "check if empty, then pop" is not.

### PriorityQueue and ties

`PriorityQueue` pops the smallest item first and compares whole items, so tuples sort by their first element and then by the next. If two items have equal priorities and the second element cannot be compared (a custom object), Python raises `TypeError`. Add a unique tie-breaker such as an increasing counter: `(priority, next(counter), item)`. A priority queue is also not FIFO among equal priorities unless you add that counter.

### Back-pressure strategies

When a bounded queue is full, a system must choose a policy.

- Block the producer: `put()` waits. This slows the producer to the speed of the consumer. It is the simplest and works well inside one process.
- Reject new work: return an error such as HTTP 429 (too many requests) or `503`. The caller decides whether to retry.
- Drop the oldest item: appropriate for telemetry where fresh data matters more than old (`deque(maxlen=...)`).
- Drop the newest item: appropriate when earlier work must not be lost.
- Shed load by priority: keep important work and drop the rest.

The wrong choice is usually "do nothing", an unbounded queue. It looks like it works until a traffic spike or a slow downstream service makes the queue grow without limit, latency explodes, and the process is killed.

### Queues and Little's law

For a stable system, the average number of items in a queue equals the arrival rate times the average time each item spends waiting and being processed (Little's law). If items arrive at 100 per second and each spends 2 seconds in the system, about 200 items are in flight. This is a quick way to size a bounded queue: it should hold the burst you want to absorb, not an unlimited amount.

### Asynchronous code

`asyncio.Queue` has the same interface (`put`, `get`, `task_done`, `join`, `maxsize`) but its operations are coroutines that you `await`, so a waiting task does not block the whole thread. Use it in `async` programs and `queue.Queue` with threads.

## Industry-level snippet

A reporting service must process many jobs, such as rendering a PDF or calling a slow partner API. The code reads jobs from a source (a database cursor or a message stream), runs them on a small pool of worker threads, and records every outcome.

The design uses the ideas of this note. A bounded `queue.Queue` sits between the producer and the workers. When the workers are busy and the queue is full, `put` blocks, so the producer cannot read faster than the workers can work, and memory use stays flat however large the job source is. That is back-pressure. Workers stop when they receive a sentinel `None` value, one per worker, which is the standard way to shut down consumer threads cleanly. One failing job must not kill a worker or lose the other jobs, so each exception is caught, logged, and recorded as a failed result. If the producer itself fails, the `finally` block still stops all workers before the error propagates, so no threads are left running.

Results are written to the database from the calling thread in one transaction, since SQLite connections should not be shared between threads without care.

```python
# job_runner.py
import logging
import queue
import sqlite3
import threading
from collections.abc import Callable, Iterable
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS job_result (
    job_id INTEGER PRIMARY KEY,
    ok INTEGER NOT NULL,
    output TEXT NOT NULL
)
"""

UPSERT_SQL = (
    "INSERT INTO job_result (job_id, ok, output) VALUES (?, ?, ?) "
    "ON CONFLICT (job_id) DO UPDATE SET ok = excluded.ok, output = excluded.output"
)


@dataclass(frozen=True, slots=True)
class Job:
    job_id: int
    payload: str


@dataclass(frozen=True, slots=True)
class JobResult:
    job_id: int
    ok: bool
    output: str


def run_jobs(
    jobs: Iterable[Job],
    handler: Callable[[Job], str],
    workers: int = 4,
    max_pending: int = 100,
) -> list[JobResult]:
    """Run jobs on worker threads behind a bounded queue. Results are sorted by id."""
    if workers < 1:
        raise ValueError("workers must be at least 1")
    if max_pending < 1:
        raise ValueError("max_pending must be at least 1")

    pending: queue.Queue[Job | None] = queue.Queue(maxsize=max_pending)
    results: list[JobResult] = []
    results_lock = threading.Lock()

    def worker() -> None:
        while True:
            job = pending.get()
            try:
                if job is None:  # the stop signal
                    return
                try:
                    result = JobResult(job.job_id, True, handler(job))
                except Exception as error:  # one bad job must not stop the worker
                    logger.exception("job %d failed", job.job_id)
                    message = f"{type(error).__name__}: {error}"
                    result = JobResult(job.job_id, False, message)
                with results_lock:
                    results.append(result)
            finally:
                pending.task_done()

    threads = [
        threading.Thread(target=worker, name=f"job-worker-{number}", daemon=True)
        for number in range(workers)
    ]
    for thread in threads:
        thread.start()
    try:
        for job in jobs:
            pending.put(job)  # blocks while the queue is full: back-pressure
    finally:
        for _ in threads:
            pending.put(None)  # one stop signal per worker
        for thread in threads:
            thread.join()
    logger.info("ran %d jobs on %d workers", len(results), workers)
    return sorted(results, key=lambda result: result.job_id)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_results(connection: sqlite3.Connection, results: Iterable[JobResult]) -> int:
    """Store all results atomically. Re-running a job replaces its earlier result."""
    rows = [(r.job_id, int(r.ok), r.output) for r in results]
    try:
        with connection:
            connection.executemany(UPSERT_SQL, rows)
    except sqlite3.Error:
        logger.exception("could not save %d job results", len(rows))
        raise
    logger.info("saved %d job results", len(rows))
    return len(rows)
```

```python
# test_job_runner.py
import sqlite3
import threading
import time
from collections.abc import Iterator

import pytest

from job_runner import Job, JobResult, create_schema, run_jobs, save_results


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def make_jobs(count: int) -> list[Job]:
    return [Job(job_id, f"payload-{job_id}") for job_id in range(count)]


def test_every_job_is_processed_and_results_are_sorted() -> None:
    results = run_jobs(make_jobs(50), lambda job: job.payload.upper(), workers=4)
    assert [r.job_id for r in results] == list(range(50))
    assert all(r.ok for r in results)
    assert results[7].output == "PAYLOAD-7"


def test_a_failing_job_does_not_stop_the_others() -> None:
    def handler(job: Job) -> str:
        if job.job_id % 5 == 0:
            raise ValueError(f"bad job {job.job_id}")
        return "fine"

    results = run_jobs(make_jobs(20), handler, workers=3)
    failed = [r for r in results if not r.ok]
    assert [r.job_id for r in failed] == [0, 5, 10, 15]
    assert failed[0].output == "ValueError: bad job 0"
    assert len(results) == 20


def test_small_queue_still_finishes_everything() -> None:
    def slow(job: Job) -> str:
        time.sleep(0.001)
        return "done"

    results = run_jobs(make_jobs(30), slow, workers=2, max_pending=1)
    assert len(results) == 30


def test_concurrency_never_exceeds_the_worker_count() -> None:
    active = 0
    peak = 0
    lock = threading.Lock()

    def handler(job: Job) -> str:
        nonlocal active, peak
        with lock:
            active += 1
            peak = max(peak, active)
        time.sleep(0.002)
        with lock:
            active -= 1
        return "ok"

    run_jobs(make_jobs(40), handler, workers=3, max_pending=5)
    assert 1 <= peak <= 3


def test_a_failing_job_source_still_stops_the_workers() -> None:
    def broken_source() -> Iterator[Job]:
        yield Job(1, "a")
        raise RuntimeError("source failed")

    before = threading.active_count()
    with pytest.raises(RuntimeError):
        run_jobs(broken_source(), lambda job: "x", workers=3)
    assert threading.active_count() == before


@pytest.mark.parametrize(("workers", "pending"), [(0, 5), (2, 0)])
def test_invalid_settings_are_rejected(workers: int, pending: int) -> None:
    with pytest.raises(ValueError):
        run_jobs([], lambda job: "x", workers=workers, max_pending=pending)


def test_empty_job_list_returns_nothing() -> None:
    assert run_jobs([], lambda job: "x") == []


def test_results_are_saved_and_replaced(connection: sqlite3.Connection) -> None:
    save_results(connection, [JobResult(1, True, "a"), JobResult(2, False, "boom")])
    save_results(connection, [JobResult(2, True, "retry ok")])
    rows = connection.execute(
        "SELECT job_id, ok, output FROM job_result ORDER BY job_id"
    ).fetchall()
    assert rows == [(1, 1, "a"), (2, 1, "retry ok")]
```

Handing the results back to the caller, instead of writing from the worker threads, keeps database access in a single thread. For long-running jobs you would normally persist each result as soon as it is finished, using a result queue read by one writer thread.

## Common mistakes

- Using `list.pop(0)` or `list.insert(0, x)` as a queue. Use `collections.deque`.
- Marking a node as visited when it is dequeued instead of when it is enqueued in breadth-first search, so the same node is queued many times.
- Indexing into the middle of a large deque and expecting `O(1)`.
- Using an unbounded queue between a fast producer and a slow consumer, then running out of memory under load.
- Forgetting `task_done()` after each `get()`, which makes `queue.join()` wait forever.
- Calling `task_done()` more times than items were put, which raises `ValueError`.
- Putting only one stop signal on the queue when several workers read from it. Each worker needs its own.
- Using `queue.Queue` in single-threaded code, paying for locks you do not need.
- Checking `if queue:` and then calling `get()` from threads and assuming nothing changed in between. Use `get_nowait()` and handle `queue.Empty`.
- Putting unorderable objects into a `PriorityQueue` without a tie-breaker, which raises `TypeError` on equal priorities.
- Mixing up the empty and full conditions in a circular buffer that tracks only `head` and `tail`.
- Using a thread-unsafe structure from several threads without a lock.

## Practice exercises

1. Write `Queue` as a class wrapping `deque` with `enqueue`, `dequeue`, `peek`, and `__len__`, and test the empty cases.
2. Write `last_n(lines, n)` that returns the last `n` items of an iterable using `deque(maxlen=n)`, and explain its memory use for an input of a million lines.
3. Time `drain_list` and `drain_deque` for `n = 10,000`, `20,000`, `40,000`, and `80,000`. Compute the ratios and explain what they show.
4. Add an `overwrite` option to `CircularBuffer` so that a full buffer drops the oldest item instead of raising, and add `peek`.
5. Implement `QueueFromStacks.peek()` and prove that every sequence of `n` operations costs `O(n)`.
6. Implement a stack using two queues, and state the cost of `push` and `pop`.
7. Write `moving_average(stream, k)` as a generator that yields the average of the last `k` items using a deque and a running sum.
8. Write `first_negative_in_each_window(values, k)` using a deque of indexes.
9. Write `shortest_path_length(grid, start, end)` using breadth-first search on a grid, and explain why a queue gives shortest paths when every step costs the same.
10. Write `sliding_window_min(values, k)` by adapting the monotonic deque, and test it against `min(values[i:i + k])` on random lists.
11. Write `longest_subarray_with_limit(values, limit)` that returns the length of the longest subarray in which the maximum minus the minimum is at most `limit`, using two monotonic deques.
12. Write `hot_potato(names, k)` that simulates passing a potato `k` steps around a circle of players using `deque.rotate`, returning the last player left.
13. Use `queue.PriorityQueue` to process tasks by priority with a counter tie-breaker so equal priorities run first in, first out, and test the order.
14. Extend `run_jobs` with a `timeout` for each `pending.put` call that raises `queue.Full` as a `BackPressureError`, and add a test that triggers it with a stuck handler.

## Summary

A queue serves items first in, first out. In Python, use `collections.deque` for queues, stacks, and windows, because both ends are `O(1)`, and never use `list.pop(0)`, which is `O(n)`. A circular buffer reuses a fixed array with modular index arithmetic, two stacks can make an amortized `O(1)` queue, and a monotonic deque of indexes gives the maximum of every sliding window in `O(n)`. For threads, `queue.Queue` and `PriorityQueue` add locking, blocking, `task_done`, and `join`, and the size limit on a queue is what creates back-pressure: a full bounded queue makes producers wait, fail, or drop work instead of letting memory grow without limit. Shut down consumers with one stop signal each, isolate failures per job, and keep the policy for a full queue an explicit design decision.
