## Overview

Each basic data structure is good at one thing. A dictionary finds an item by key in constant time but has no useful order. A linked list can move or remove a node in constant time but cannot find one. A list gives instant access by position but cannot remove from the middle cheaply. Real requirements rarely fit a single structure, for example: "look up by key in `O(1)`, and also know which item was used least recently, and evict it in `O(1)`". No single structure does both.

The solution is to combine structures so that each one covers the other's weakness. A dictionary mapping keys to nodes of a doubly linked list gives you an LRU cache. A list plus a dictionary gives you a set with `O(1)` random selection. A pair of stacks gives you browser history with back and forward. These designs appear in production caches, schedulers, game servers, and databases, and in many interview questions.

This note covers an LRU cache built from a hash map and a doubly linked list, `OrderedDict` and `functools.lru_cache` and how they work inside, an LFU cache, ring-buffer log storage, browser history, a set with insert, delete, and random pick in constant time, and a method for designing a data structure from a list of requirements. The recurring difficulty is keeping the combined structures consistent with each other, so most of the care goes into making every change update all of them together.

## Core concepts

### Why combine structures

A combined structure keeps the same data in two or more forms, each organized for a different question.

- A dictionary answers "where is the item with this key?" in `O(1)`.
- A linked list or ordered container answers "which item is oldest, newest, or next?" and supports moving an item in `O(1)`.
- A list answers "give me the item at position `i`" in `O(1)`.
- A heap answers "which item is smallest?" in `O(1)`.

The combined structure is correct only if its parts always agree. Every operation must update all of them, and the invariant that ties them together should be written down.

### LRU cache

A cache stores recent results so they need not be recomputed. It has a limited capacity, so when it is full it must discard something. A least recently used (LRU) cache discards the item that has gone unused for the longest time, on the theory that recent use predicts future use.

Required operations, each in `O(1)`:

- `get(key)`: return the value and mark the item as most recently used.
- `put(key, value)`: insert or update, mark as most recently used, and evict the least recently used item if the capacity is exceeded.

Design: a dictionary from key to node, and a doubly linked list ordered from most recent to least recent. The dictionary finds the node. The list moves it to the front and finds the oldest one at the other end. Because the dictionary gives you the node directly, unlinking it from the doubly linked list is `O(1)`, which is exactly the case where a linked list beats an array.

### OrderedDict and functools.lru_cache

- `collections.OrderedDict` is a dictionary that remembers insertion order and supports `move_to_end(key)` and `popitem(last=False)` in `O(1)`. It is a ready-made hash map plus linked list, so an LRU cache takes a dozen lines.
- `functools.lru_cache` is a decorator that caches a function's results by its arguments with an LRU policy. Its C implementation uses a dictionary and a circular doubly linked list, the same design.

### LFU cache

A least frequently used (LFU) cache evicts the item with the fewest accesses, with ties broken by recency. It protects items that are popular over time, where LRU can be flushed by one large scan. An `O(1)` design keeps a dictionary from key to value, a dictionary from key to its count, and a dictionary from count to an ordered group of keys with that count, plus a variable tracking the minimum count.

### Ring-buffer log storage

A ring buffer keeps only the most recent `n` records in a fixed amount of memory by overwriting the oldest. A `deque` with `maxlen` is a ring buffer. Combined with a counter of total records ever written, it supports "show the last 100 lines" and "how many lines have there been in total".

### Browser history

Back and forward navigation can use two stacks or a list with a pointer. Visiting a new page truncates everything after the current position (the forward pages), because the old future no longer applies.

### Insert, delete, and random in constant time

To pick a random element from a set in `O(1)` you need array indexing, and to find and delete an element in `O(1)` you need a hash table. Keep both: a list of items and a dictionary from item to its index in the list. To delete, swap the item with the last one in the list, update the dictionary, and pop the end.

### Designing a structure from requirements

A reliable method:

1. List the operations the structure must support.
2. Write the time complexity each one must achieve.
3. For each operation, ask which structure answers it at that cost.
4. Combine the structures, with each one storing exactly what is needed to answer its questions.
5. State the invariant that links them.
6. Check that every mutating operation preserves the invariant, and test with a simple model (an obviously correct, slow version).

## Basic syntax examples

### An LRU cache from a dictionary and a doubly linked list

```python
from typing import cast


class _Node[K, V]:
    __slots__ = ("key", "value", "prev", "next")

    def __init__(self, key: K, value: V) -> None:
        self.key = key
        self.value = value
        self.prev: _Node[K, V] | None = None
        self.next: _Node[K, V] | None = None


class LRUCache[K, V]:
    """get and put are O(1). The list runs from most recent (head) to oldest (tail)."""

    def __init__(self, capacity: int) -> None:
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        self._capacity = capacity
        self._nodes: dict[K, _Node[K, V]] = {}
        # Sentinels mean no operation has to check for an empty list.
        self._head: _Node[K, V] = _Node(cast(K, None), cast(V, None))
        self._tail: _Node[K, V] = _Node(cast(K, None), cast(V, None))
        self._head.next = self._tail
        self._tail.prev = self._head

    def __len__(self) -> int:
        return len(self._nodes)

    def _unlink(self, node: _Node[K, V]) -> None:
        before, after = node.prev, node.next
        assert before is not None and after is not None
        before.next = after
        after.prev = before

    def _push_front(self, node: _Node[K, V]) -> None:
        first = self._head.next
        assert first is not None
        node.prev, node.next = self._head, first
        self._head.next = node
        first.prev = node

    def get(self, key: K) -> V | None:
        node = self._nodes.get(key)
        if node is None:
            return None
        self._unlink(node)  # O(1) because the dictionary gave us the node itself
        self._push_front(node)  # it is now the most recently used
        return node.value

    def put(self, key: K, value: V) -> None:
        node = self._nodes.get(key)
        if node is not None:
            node.value = value
            self._unlink(node)
            self._push_front(node)
            return
        if len(self._nodes) >= self._capacity:
            oldest = self._tail.prev
            assert oldest is not None and oldest is not self._head
            self._unlink(oldest)
            del self._nodes[oldest.key]  # keep the dictionary and the list in step
        new = _Node(key, value)
        self._nodes[key] = new
        self._push_front(new)

    def keys_by_recency(self) -> list[K]:
        keys: list[K] = []
        node = self._head.next
        while node is not None and node is not self._tail:
            keys.append(node.key)
            node = node.next
        return keys


cache = LRUCache[str, int](2)
cache.put("a", 1)
cache.put("b", 2)
print(cache.get("a"))  # 1, and "a" becomes the most recent
cache.put("c", 3)  # the cache is full, so the least recent key "b" is evicted
print(cache.get("b"))  # None
print(cache.keys_by_recency())  # ['c', 'a']
```

State after each step (most recent first):

```text
put a:        [a]
put b:        [b, a]
get a:        [a, b]
put c:        evict b (the tail), result [c, a]
```

### The same idea with OrderedDict

```python
from collections import OrderedDict


class OrderedLRU[K, V]:
    def __init__(self, capacity: int) -> None:
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        self._capacity = capacity
        self._items: OrderedDict[K, V] = OrderedDict()

    def get(self, key: K) -> V | None:
        if key not in self._items:
            return None
        self._items.move_to_end(key)  # the right end is the most recent
        return self._items[key]

    def put(self, key: K, value: V) -> None:
        self._items[key] = value
        self._items.move_to_end(key)
        if len(self._items) > self._capacity:
            self._items.popitem(last=False)  # drop the oldest item


lru = OrderedLRU[str, int](2)
lru.put("a", 1)
lru.put("b", 2)
lru.get("a")
lru.put("c", 3)
print(lru.get("b"), lru.get("a"))  # None 1
```

### functools.lru_cache

```python
from functools import lru_cache


@lru_cache(maxsize=128)
def square(n: int) -> int:
    return n * n


square(4)
square(4)
square(5)
print(square.cache_info())  # CacheInfo(hits=1, misses=2, maxsize=128, currsize=2)
square.cache_clear()
print(square.cache_info().currsize)  # 0
```

### An LFU cache with O(1) operations

```python
from collections import OrderedDict, defaultdict


class LFUCache[K, V]:
    """Evicts the least frequently used key. Ties go to the least recently used."""

    def __init__(self, capacity: int) -> None:
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        self._capacity = capacity
        self._values: dict[K, V] = {}
        self._counts: dict[K, int] = {}
        # count -> keys with that count, oldest first
        self._buckets: defaultdict[int, OrderedDict[K, None]] = defaultdict(OrderedDict)
        self._min_count = 0

    def _touch(self, key: K) -> None:
        count = self._counts[key]
        del self._buckets[count][key]
        if not self._buckets[count]:
            del self._buckets[count]
            if self._min_count == count:
                self._min_count += 1  # the lowest bucket emptied: minimum moves up
        self._counts[key] = count + 1
        self._buckets[count + 1][key] = None

    def get(self, key: K) -> V | None:
        if key not in self._values:
            return None
        self._touch(key)
        return self._values[key]

    def put(self, key: K, value: V) -> None:
        if key in self._values:
            self._values[key] = value
            self._touch(key)
            return
        if len(self._values) >= self._capacity:
            evicted, _ = self._buckets[self._min_count].popitem(last=False)
            if not self._buckets[self._min_count]:
                del self._buckets[self._min_count]
            del self._values[evicted]
            del self._counts[evicted]
        self._values[key] = value
        self._counts[key] = 1
        self._buckets[1][key] = None
        self._min_count = 1  # a new key always has the lowest possible count


lfu = LFUCache[str, int](2)
lfu.put("a", 1)
lfu.put("b", 2)
lfu.get("a")  # a has been used twice, b once
lfu.put("c", 3)  # evicts b, the least frequently used
print(lfu.get("b"), lfu.get("a"), lfu.get("c"))  # None 1 3
```

### A set with O(1) insert, delete, and random choice

```python
import random


class RandomizedSet[T]:
    """add, remove, and random_item are all O(1)."""

    def __init__(self) -> None:
        self._items: list[T] = []
        self._index: dict[T, int] = {}  # item -> its position in the list

    def __len__(self) -> int:
        return len(self._items)

    def add(self, item: T) -> bool:
        if item in self._index:
            return False
        self._index[item] = len(self._items)
        self._items.append(item)
        return True

    def remove(self, item: T) -> bool:
        if item not in self._index:
            return False
        position = self._index.pop(item)
        last = self._items.pop()  # always remove from the end of the list: O(1)
        if position < len(self._items):  # the removed item was not the last one
            self._items[position] = last  # move the last item into the gap
            self._index[last] = position
        return True

    def random_item(self, rng: random.Random | None = None) -> T:
        if not self._items:
            raise IndexError("random_item from an empty set")
        return (rng or random.Random()).choice(self._items)


items = RandomizedSet[str]()
for name in ("a", "b", "c", "d"):
    items.add(name)
items.remove("b")  # "d" moves into b's slot
print(len(items), items.random_item(random.Random(1)) in {"a", "c", "d"})  # 3 True
```

### Browser history

```python
class BrowserHistory:
    """A list of pages and a pointer to the current one."""

    def __init__(self, home: str) -> None:
        self._pages = [home]
        self._current = 0

    def visit(self, url: str) -> None:
        del self._pages[self._current + 1 :]  # visiting clears the forward pages
        self._pages.append(url)
        self._current += 1

    def back(self, steps: int = 1) -> str:
        self._current = max(0, self._current - steps)
        return self._pages[self._current]

    def forward(self, steps: int = 1) -> str:
        self._current = min(len(self._pages) - 1, self._current + steps)
        return self._pages[self._current]


history = BrowserHistory("home")
history.visit("news")
history.visit("mail")
print(history.back(), history.back())  # news home
print(history.forward())  # news
history.visit("shop")  # the "mail" page is gone from the forward history
print(history.forward())  # shop, there is nothing further to go to
```

### A ring-buffer log

```python
from collections import deque


class RecentLog:
    """Keeps the newest `capacity` lines and counts everything ever written."""

    def __init__(self, capacity: int) -> None:
        self._lines: deque[str] = deque(maxlen=capacity)
        self.total_written = 0

    def write(self, line: str) -> None:
        self._lines.append(line)  # the oldest line drops out when the deque is full
        self.total_written += 1

    def latest(self, count: int) -> list[str]:
        if count <= 0:
            return []  # note: lines[-0:] would return everything
        return list(self._lines)[-count:]


log = RecentLog(3)
for number in range(1, 6):
    log.write(f"line {number}")
print(log.latest(2), log.total_written)  # ['line 4', 'line 5'] 5
```

## Deeper behavior

### The invariant that holds the parts together

In the LRU cache, the invariant is: the set of keys in the dictionary equals the set of keys in the linked list, and each dictionary value is the node holding that key. Every operation must keep it true.

- `put` for a new key adds to the dictionary and the list.
- Eviction removes the oldest node from the list and its key from the dictionary. Forgetting the dictionary removal leaves a node that is no longer in the list, which later causes an `AttributeError` or an incorrect hit.
- `get` and `put` for an existing key only move the node.

In `RandomizedSet`, the invariant is: `index[item] == position` for every item at that position in the list. The swap-with-last trick in `remove` is exactly what restores it after a deletion. Writing the invariant down, and testing it after random operations, catches most bugs in combined structures.

### Why OrderedDict is O(1) for move_to_end

CPython's `OrderedDict` is implemented in C as a dictionary plus a doubly linked list of nodes. The dictionary maps keys to nodes, so `move_to_end` finds the node in `O(1)` and relinks its neighbors in `O(1)`. It is the same design as the hand-written LRU cache above, and it is why a few lines of `OrderedDict` code replace the whole class. A plain `dict` also remembers insertion order, but it has no way to move a key to the end without deleting and re-inserting it, and it cannot pop from the front cheaply.

### Inside functools.lru_cache

The decorator wraps the function and stores results in a dictionary keyed by the call's arguments, with each entry linked into a circular doubly linked list ordered by use. A hit moves the entry to the most recent position. When the cache is full, the entry at the oldest position is removed. In practice:

- Arguments must be hashable. Passing a list raises `TypeError`.
- `f(1, 2)` and `f(a=1, b=2)` are separate cache entries, because the key includes how the call was written.
- `maxsize=None` disables eviction (the same as `functools.cache`), which is fastest but grows without bound.
- `typed=True` caches `f(1)` and `f(1.0)` separately.
- `cache_info()` reports hits, misses, size, and capacity. `cache_clear()` empties the cache.
- Decorating a method caches on `self` too, so instances stay alive as long as their entries are cached. Cache module-level functions, or manage a per-instance cache yourself.
- The cached function must be pure. A function that reads the clock or a database would return stale data forever.

### LFU details

The frequency buckets form a second index: counts to groups of keys. Increasing a key's count moves it from one bucket to the next higher one. The minimum count changes only in two ways: a new key resets it to 1, and when an access empties the minimum bucket, the minimum moves up by exactly one, because the key that was touched now sits in the next bucket. Eviction takes the oldest key in the minimum bucket, which breaks ties by recency. Every step touches a constant number of dictionary and `OrderedDict` operations, so everything is `O(1)`.

LFU has a weakness: an item that was very popular in the past keeps a high count and may never be evicted even if nobody asks for it anymore. Production variants add aging, so that counts decay over time.

### LRU versus LFU

- LRU is simple and adapts quickly to shifting access patterns. A one-time scan through many items can flush the useful ones.
- LFU keeps items that are accessed often over time, so a scan does not push them out. It adapts slowly and needs aging.
- Neither is universally better. Measure the hit rate on a sample of real access logs.

### Time to live and expiry

Many caches also expire entries after a time to live (TTL). Two approaches:

- Lazy expiry: check the entry's expiry time when you read it, and delete it then if it has expired. No background work, but expired entries occupy space until they are read or evicted by the LRU policy.
- Active expiry: a background task removes expired entries. This frees memory sooner but needs a timer or a heap ordered by expiry time.

Combining a TTL with an LRU policy is still one structure: the `OrderedDict` handles recency and capacity, and each entry stores its expiry time.

### Cache invalidation

A cache is a second copy of data, and two copies can disagree. When the source of truth changes, the cached copy must be removed or updated. The simple and common strategy is invalidate on write: change the database first, and when the change has committed, delete the cached entry so that the next read reloads it. The order matters. If you delete the cache first and then write, a concurrent reader can reload the old value in between and cache it. Even with the right order a narrow race remains: a reader that loaded the old value just before the commit may store it just after the invalidation. A TTL bounds how long such stale data can survive.

### Thread safety

A cache shared between threads needs a lock around every compound operation. Looking up an entry, checking its expiry, and moving it to the end must happen as one step, or two threads can interleave and corrupt the order. Holding a lock while loading a missing value serializes all misses behind a slow load. Release the lock before calling the loader, accept that two threads may both load the same key, and consider a per-key lock if loading is expensive. This duplicate loading is the cache stampede problem.

### Choosing structures from requirements

- Look up by key: dictionary.
- Remove or reorder a known item in `O(1)`: doubly linked list with a dictionary to find nodes (or `OrderedDict`).
- Smallest or largest first: heap, with lazy deletion if removals are needed.
- Random element: list plus dictionary of positions.
- Ordered iteration and range queries: sorted list with `bisect`, or a balanced tree.
- Last `n` items: `deque(maxlen=n)`.
- Duplicates not allowed: set or dictionary keys.
- Undo and redo, history: stacks, or a list with a pointer.

### Testing combined structures

Compare against a trivially correct model. For an LRU cache, the model is a plain list of keys ordered by recency, searched linearly. Run thousands of random operations on both and check that their observable behavior (results of `get`, set of keys) is identical after every step. This catches the subtle bugs, such as a stale dictionary entry, that unit tests with a few examples miss.

## Industry-level snippet

A mobile app calls a profile service for every screen, and the same few thousand profiles account for most requests. A database read for each request is wasteful, so the service keeps a cache in front of the database. The cache combines the structures from this note: an `OrderedDict` for recency and capacity, an expiry timestamp in each entry for the time-to-live, and a lock so that many threads can share it. It tracks hits, misses, evictions, and expirations so that operators can see whether it is working.

The repository around the cache follows the cache invalidation rules above. Reads use `get_or_load`, and the loader runs outside the cache's lock. Writes update the database in a transaction, and the cache entry is invalidated only after the transaction commits. If the update fails, the cached value remains valid because nothing changed. If the loader raises, as when a profile does not exist, nothing is cached, so the error is not remembered.

The clock is injected as a function, so tests can move time forward without sleeping.

```python
# profile_cache.py
import logging
import sqlite3
import threading
import time
from collections import OrderedDict
from collections.abc import Callable
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS user_profile (
    user_id INTEGER PRIMARY KEY,
    display_name TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
)
"""


class ProfileNotFoundError(LookupError):
    """Raised when a user has no profile."""


@dataclass(slots=True)
class CacheStats:
    hits: int = 0
    misses: int = 0
    evictions: int = 0
    expirations: int = 0


class TTLCache[K, V]:
    """A thread-safe LRU cache whose entries also expire after a time to live.

    Values must not be None, because None is how get() reports a miss.
    """

    def __init__(
        self,
        capacity: int,
        ttl_seconds: float,
        clock: Callable[[], float] = time.monotonic,
    ) -> None:
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        if ttl_seconds <= 0:
            raise ValueError("ttl_seconds must be positive")
        self._capacity = capacity
        self._ttl = ttl_seconds
        self._clock = clock
        self._entries: OrderedDict[K, tuple[V, float]] = OrderedDict()
        self._lock = threading.Lock()
        self.stats = CacheStats()

    def __len__(self) -> int:
        with self._lock:
            return len(self._entries)

    def get(self, key: K) -> V | None:
        with self._lock:
            entry = self._entries.get(key)
            if entry is None:
                self.stats.misses += 1
                return None
            value, expires_at = entry
            if self._clock() >= expires_at:
                del self._entries[key]  # lazy expiry: remove it when we notice
                self.stats.expirations += 1
                self.stats.misses += 1
                return None
            self._entries.move_to_end(key)  # most recently used goes to the end
            self.stats.hits += 1
            return value

    def put(self, key: K, value: V) -> None:
        with self._lock:
            self._entries[key] = (value, self._clock() + self._ttl)
            self._entries.move_to_end(key)
            while len(self._entries) > self._capacity:
                self._entries.popitem(last=False)  # evict the least recently used
                self.stats.evictions += 1

    def invalidate(self, key: K) -> bool:
        with self._lock:
            return self._entries.pop(key, None) is not None

    def get_or_load(self, key: K, loader: Callable[[], V]) -> V:
        """Return the cached value, or call the loader and cache its result.

        The loader runs outside the lock, so a slow load does not block other keys.
        If it raises, nothing is cached.
        """
        value = self.get(key)
        if value is not None:
            return value
        loaded = loader()
        self.put(key, loaded)
        return loaded


@dataclass(frozen=True, slots=True)
class Profile:
    user_id: int
    display_name: str
    version: int


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


class ProfileRepository:
    def __init__(
        self, connection: sqlite3.Connection, cache: TTLCache[int, Profile]
    ) -> None:
        self._connection = connection
        self._cache = cache

    def get(self, user_id: int) -> Profile:
        return self._cache.get_or_load(user_id, lambda: self._load(user_id))

    def _load(self, user_id: int) -> Profile:
        row = self._connection.execute(
            "SELECT user_id, display_name, version FROM user_profile WHERE user_id = ?",
            (user_id,),
        ).fetchone()
        if row is None:
            raise ProfileNotFoundError(f"no profile for user {user_id}")
        return Profile(row[0], row[1], row[2])

    def rename(self, user_id: int, display_name: str) -> Profile:
        """Change the name in a transaction, then invalidate the cached copy."""
        if not display_name.strip():
            raise ValueError("display_name must not be blank")
        try:
            with self._connection:
                cursor = self._connection.execute(
                    "UPDATE user_profile SET display_name = ?, version = version + 1 "
                    "WHERE user_id = ?",
                    (display_name.strip(), user_id),
                )
                if cursor.rowcount == 0:
                    raise ProfileNotFoundError(f"no profile for user {user_id}")
        except sqlite3.Error:
            logger.exception("could not rename user %d", user_id)
            raise
        self._cache.invalidate(user_id)  # only after the commit succeeded
        logger.info("renamed user %d", user_id)
        return self._load(user_id)
```

```python
# test_profile_cache.py
import sqlite3
from collections.abc import Iterator

import pytest

from profile_cache import (
    ProfileNotFoundError,
    ProfileRepository,
    TTLCache,
    create_schema,
)


class FakeClock:
    def __init__(self) -> None:
        self.now = 1_000.0

    def __call__(self) -> float:
        return self.now


@pytest.fixture
def clock() -> FakeClock:
    return FakeClock()


def test_hits_and_misses_are_counted(clock: FakeClock) -> None:
    cache = TTLCache[str, int](2, 60, clock)
    assert cache.get("a") is None
    cache.put("a", 1)
    assert cache.get("a") == 1
    assert (cache.stats.hits, cache.stats.misses) == (1, 1)


def test_least_recently_used_entry_is_evicted(clock: FakeClock) -> None:
    cache = TTLCache[str, int](2, 60, clock)
    cache.put("a", 1)
    cache.put("b", 2)
    cache.get("a")  # "a" is now more recent than "b"
    cache.put("c", 3)
    assert cache.get("b") is None
    assert cache.get("a") == 1
    assert cache.get("c") == 3
    assert cache.stats.evictions == 1
    assert len(cache) == 2


def test_entries_expire_after_the_ttl(clock: FakeClock) -> None:
    cache = TTLCache[str, int](5, 30, clock)
    cache.put("a", 1)
    clock.now += 29
    assert cache.get("a") == 1
    clock.now += 1
    assert cache.get("a") is None
    assert cache.stats.expirations == 1
    assert len(cache) == 0


def test_put_refreshes_the_ttl(clock: FakeClock) -> None:
    cache = TTLCache[str, int](5, 30, clock)
    cache.put("a", 1)
    clock.now += 20
    cache.put("a", 2)
    clock.now += 20
    assert cache.get("a") == 2


def test_get_or_load_calls_the_loader_once(clock: FakeClock) -> None:
    cache = TTLCache[str, int](5, 30, clock)
    calls: list[str] = []

    def loader() -> int:
        calls.append("load")
        return 42

    assert cache.get_or_load("k", loader) == 42
    assert cache.get_or_load("k", loader) == 42
    assert calls == ["load"]


def test_a_failing_loader_caches_nothing(clock: FakeClock) -> None:
    cache = TTLCache[str, int](5, 30, clock)

    def failing() -> int:
        raise RuntimeError("database down")

    with pytest.raises(RuntimeError):
        cache.get_or_load("k", failing)
    assert len(cache) == 0
    assert cache.get_or_load("k", lambda: 7) == 7


def test_invalidate(clock: FakeClock) -> None:
    cache = TTLCache[str, int](5, 30, clock)
    cache.put("a", 1)
    assert cache.invalidate("a") is True
    assert cache.invalidate("a") is False
    assert cache.get("a") is None


@pytest.mark.parametrize(("capacity", "ttl"), [(0, 10.0), (3, 0.0), (3, -1.0)])
def test_invalid_settings_are_rejected(capacity: int, ttl: float) -> None:
    with pytest.raises(ValueError):
        TTLCache[str, int](capacity, ttl)


@pytest.fixture
def repository(clock: FakeClock) -> Iterator[ProfileRepository]:
    connection = sqlite3.connect(":memory:")
    create_schema(connection)
    connection.execute(
        "INSERT INTO user_profile (user_id, display_name) VALUES (1, 'Ana')"
    )
    connection.commit()
    yield ProfileRepository(connection, TTLCache(10, 60, clock))
    connection.close()


def test_second_read_comes_from_the_cache(repository: ProfileRepository) -> None:
    first = repository.get(1)
    second = repository.get(1)
    assert first == second
    assert first.display_name == "Ana"
    assert repository._cache.stats.hits == 1


def test_rename_invalidates_so_readers_see_the_new_name(
    repository: ProfileRepository,
) -> None:
    assert repository.get(1).display_name == "Ana"
    updated = repository.rename(1, "  Ana Maria ")
    assert updated.display_name == "Ana Maria"
    assert updated.version == 2
    assert repository.get(1).display_name == "Ana Maria"


def test_unknown_profile_is_an_error_and_is_not_cached(
    repository: ProfileRepository,
) -> None:
    with pytest.raises(ProfileNotFoundError):
        repository.get(99)
    with pytest.raises(ProfileNotFoundError):
        repository.rename(99, "Nobody")
    assert len(repository._cache) == 0


def test_blank_names_are_rejected_and_the_cache_survives(
    repository: ProfileRepository,
) -> None:
    repository.get(1)
    with pytest.raises(ValueError):
        repository.rename(1, "   ")
    assert repository.get(1).display_name == "Ana"
    assert repository._cache.stats.hits == 1
```

Because `TTLCache` takes the clock as an argument, the expiry tests are instant and exact. A test that called `time.sleep(31)` would be slow, and one that slept for a fraction of a second would be flaky.

## Common mistakes

- Updating only one of the combined structures. Evicting a node from the list but not the dictionary (or the reverse) leaves the cache corrupt.
- Forgetting to move the node to the front on a `get`, so the cache evicts items that are in constant use.
- Writing an LRU cache with a list and `remove`, which is `O(n)` per hit, when a dictionary plus linked list or `OrderedDict` is `O(1)`.
- Using sentinels incorrectly, for example evicting the head sentinel when the list is empty.
- Caching a method with `functools.lru_cache` and keeping every instance alive.
- Caching functions that are not pure, so the cache returns stale data.
- Passing unhashable arguments (lists, dicts) to an `lru_cache` function.
- Invalidating the cache before the database write instead of after the commit.
- Holding the cache lock while calling a slow loader, which blocks every other thread.
- Letting an error from the loader get cached, so a temporary failure becomes permanent.
- Caching `None` as a value when `None` also means "not found" in the cache interface.
- Using `lines[-count:]` when `count` can be zero, which returns everything instead of nothing.
- Removing from the middle of the list in a random-pick set instead of swapping with the last item.
- Never testing the combined structure against a simple model.

## Practice exercises

1. Write an LRU cache with a plain list of keys and a dictionary of values, and measure how its cost per `get` grows with capacity. Then compare it with `LRUCache`.
2. Add `delete(key)` and `__contains__` to `LRUCache`, and make sure `__contains__` does not change the recency order.
3. Write a model-based test for `LRUCache`: run 10,000 random `get` and `put` operations against both the cache and a simple list-based model, and compare the visible state after each step.
4. Write `OrderedLRU` using `OrderedDict` with a `capacity` that can be changed with `resize(new_capacity)`, evicting as needed.
5. Use `functools.lru_cache` on a recursive function for the number of paths in a grid, and read `cache_info()` to compute the hit rate.
6. Trace `LFUCache` by hand for the operations `put(1)`, `put(2)`, `get(1)`, `put(3)`, `get(2)`, `get(3)`, `put(4)` with capacity 2, writing the buckets and the minimum count after each step.
7. Extend `LFUCache` with aging: every `n` operations, halve all counts. Describe the data structure changes you need.
8. Write `RandomizedMultiset` where duplicates are allowed, using a dictionary from item to a set of positions in the list, and make `remove`, `add`, and `random_item` `O(1)`.
9. Write `AllOne`, a structure with `inc(key)`, `dec(key)`, `get_max_key()`, and `get_min_key()` all in `O(1)`, using a doubly linked list of count buckets.
10. Write `BrowserHistory` with two stacks (back and forward) instead of a list and pointer, and compare the code.
11. Write a `MovingAverage` class using `deque(maxlen=k)` and a running sum, where `next(value)` returns the average of the last `k` values in `O(1)`.
12. Design a rate limiter that allows at most `n` requests per client per minute, using a dictionary from client to a deque of timestamps. State the memory and time per request.
13. Design a data structure for a leaderboard that supports `add_score(player, score)`, `top(k)`, and `rank(player)`. Choose structures, state each operation's complexity, and write the invariant.
14. Extend `TTLCache` with active expiry: a method `purge_expired()` that removes all expired entries efficiently by keeping a heap of expiry times, and explain how it interacts with `get`.

## Summary

Combining data structures lets each one cover the other's weakness. A dictionary from key to node plus a doubly linked list gives an LRU cache with `O(1)` get and put, `OrderedDict` implements the same design in C, and `functools.lru_cache` applies it to function calls. An LFU cache adds a dictionary of frequency buckets and a minimum-count pointer. A list plus a dictionary of positions gives a set with `O(1)` random choice through the swap-with-last trick, a `deque(maxlen=n)` is a ring buffer for recent records, and a list with a pointer models browser history. The hard part is consistency: write down the invariant that ties the parts together, update every part in every operation, and test against a simple model. In real services, add expiry, locking around compound operations, a loader that runs outside the lock and never caches failures, and invalidation after a committed write.
