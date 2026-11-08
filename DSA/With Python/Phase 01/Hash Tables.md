## Overview

A hash table is a structure that finds a value from a key in constant time on average, without scanning. It is the machinery behind Python's `dict` and `set`, and it is probably the single most useful data structure in everyday programming: caches, indexes, counters, deduplication, lookups by ID, and routing tables are all hash tables in disguise.

The idea is simple. A hash function turns a key into a number, and the number decides which slot of an array to look in. Two different keys can land in the same slot, which is called a collision, so every hash table needs a strategy for handling collisions and a rule for growing before it gets crowded. When those two pieces work well, lookups, inserts, and deletes cost `O(1)` on average. When they fail, for example because many keys collide, operations slow down to `O(n)`.

This note explains hash functions, collisions, chaining and open addressing, load factor and resizing, how to build a hash map from scratch, how CPython's `dict` and `set` are laid out, the contract between `__hash__` and `__eq__`, why mutable keys break, hash randomization, and what a hash-flooding attack is.

## Core concepts

### Hash functions

A hash function maps a key to an integer. In Python, `hash(key)` does it. A good hash function is:

- Deterministic: the same key always gives the same hash within one program run.
- Fast to compute.
- Well spread: different keys should give very different numbers, so they do not pile up in the same slot.

To pick a slot, the table reduces the hash to a valid index, usually `hash(key) % capacity`.

### Collisions

A collision happens when two different keys map to the same slot. Collisions are unavoidable: there are far more possible keys than slots. By the pigeonhole principle, once you store more keys than slots, some slot holds two. A hash table is defined as much by how it handles collisions as by its hash function.

### Chaining

In separate chaining, each slot holds a small collection (a bucket), often a list, of all the key-value pairs that hash there. To look up a key, hash it, go to its bucket, and compare against the few entries inside. Deleting is simple: remove the entry from its bucket.

### Open addressing

In open addressing, every entry lives directly in the array. If the slot for a key is taken, the table probes other slots in a fixed pattern (linear probing tries the next slot, then the next) until it finds the key or an empty slot. It uses less memory per entry and is cache friendly. Deleting needs care: an emptied slot would break the probe chain, so deleted slots are marked with a tombstone that lookups skip over and inserts may reuse.

### Load factor and resizing

The load factor is `size / capacity`, the fraction of slots in use. As it grows, collisions become more likely and operations slow down. A hash table keeps the load factor below a limit (around 0.66 to 0.75 in common implementations). When the limit is crossed, the table allocates a bigger array and re-inserts every entry, because slot positions depend on the capacity. Like a dynamic array, this makes inserts `O(1)` amortized.

### Python's dict and set

- `dict` maps hashable keys to values, keeps insertion order, and offers average `O(1)` lookup, insert, and delete.
- `set` is the same hash table without values: average `O(1)` membership, add, and discard.
- Keys and set members must be hashable. Immutable built-ins such as `int`, `str`, `tuple` (of hashables), and `frozenset` are. `list`, `dict`, and `set` are not.

### The `__hash__` and `__eq__` contract

For an object to work as a dictionary key or set member, two rules must hold.

- If `a == b`, then `hash(a) == hash(b)`.
- The hash of an object must not change while it is stored in a table.

Equal objects must hash equally because the table first uses the hash to find the slot and only then compares with `==`. The reverse is not required: different objects may share a hash.

### Mutable keys

If a key's hash depends on a field and that field changes after insertion, the entry is stored in a slot that no longer matches the key's hash. Lookups look in the wrong place, and the entry becomes unreachable. This is why Python makes lists unhashable, and why user-defined keys should be immutable (a frozen dataclass or a tuple).

### Hash randomization and hash flooding

CPython randomizes the hash of `str` and `bytes` at startup using a per-process secret seed. An attacker who could predict string hashes could send many keys that all land in the same slot and force every lookup to scan a long chain, turning `O(1)` operations into `O(n)` and slowing a server to a crawl. This is a hash-flooding attack. Randomization defends against it for strings, but the hash of numbers is not randomized, so a table keyed by attacker-controlled integers can still be attacked.

## Basic syntax examples

### Dictionaries and sets

```python
ages: dict[str, int] = {"Asha": 31, "Bilal": 27}
ages["Chen"] = 45  # insert, O(1) average
ages["Asha"] = 32  # update
print(ages.get("Dina", -1))  # -1, a default instead of KeyError
print("Bilal" in ages)  # True, O(1) average
del ages["Bilal"]
for name, age in ages.items():  # insertion order is preserved
    print(name, age)

visited: set[int] = set()
visited.add(7)
visited.add(7)  # duplicates are ignored
print(len(visited), 7 in visited)  # 1 True
```

### What hash() returns

```python
print(hash(42))  # 42, small integers hash to themselves
print(hash((1, 2)) == hash((1, 2)))  # True, equal tuples hash equally
print(hash(1) == hash(1.0) == hash(True))  # True, because 1 == 1.0 == True
print({1: "a", True: "b", 1.0: "c"})  # {1: 'c'}, all three are the same key

try:
    hash([1, 2])
except TypeError as error:
    print(error)  # unhashable type: 'list'
```

### A hashable custom key

A frozen dataclass generates `__eq__` and `__hash__` from its fields, and frozen means the fields cannot change, so the contract holds.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Coordinate:
    row: int
    col: int


distances: dict[Coordinate, int] = {Coordinate(0, 0): 0, Coordinate(0, 1): 1}
print(distances[Coordinate(0, 1)])  # 1, equal fields mean an equal key
```

### A hash map from scratch with chaining

This version uses a fixed number of buckets. Each bucket is a list of `(key, value)` pairs.

```python
class SimpleHashMap[K, V]:
    """Separate chaining with a fixed number of buckets. Never resizes."""

    def __init__(self, bucket_count: int = 8) -> None:
        if bucket_count < 1:
            raise ValueError("bucket_count must be at least 1")
        self._buckets: list[list[tuple[K, V]]] = [[] for _ in range(bucket_count)]
        self._size = 0

    def __len__(self) -> int:
        return self._size

    @property
    def bucket_count(self) -> int:
        return len(self._buckets)

    def _bucket_for(self, key: K) -> list[tuple[K, V]]:
        return self._buckets[hash(key) % len(self._buckets)]

    def put(self, key: K, value: V) -> None:
        bucket = self._bucket_for(key)
        for index, (existing_key, _) in enumerate(bucket):
            if existing_key == key:
                bucket[index] = (key, value)  # update an existing key
                return
        bucket.append((key, value))
        self._size += 1

    def get(self, key: K, default: V | None = None) -> V | None:
        for existing_key, value in self._bucket_for(key):
            if existing_key == key:
                return value
        return default

    def remove(self, key: K) -> V:
        bucket = self._bucket_for(key)
        for index, (existing_key, value) in enumerate(bucket):
            if existing_key == key:
                del bucket[index]
                self._size -= 1
                return value
        raise KeyError(key)

    def __contains__(self, key: K) -> bool:
        return any(existing_key == key for existing_key, _ in self._bucket_for(key))


table = SimpleHashMap[str, int]()
table.put("apple", 3)
table.put("pear", 5)
table.put("apple", 4)
print(table.get("apple"), len(table), "pear" in table)  # 4 2 True
```

Trace of `put("apple", 3)` into 8 buckets:

```text
hash("apple") is some large integer, for example 5791...
bucket index = hash % 8, for example 3
bucket 3 is empty, so append ("apple", 3)
get("apple") repeats the same hash and index, then scans bucket 3
```

## Deeper behavior

### Resizing keeps buckets short

With a fixed number of buckets, adding a million keys to 8 buckets leaves each bucket with about 125,000 entries, and every lookup scans them all. Resizing fixes this. When the load factor crosses a limit, double the bucket count and re-insert every entry. Each entry's bucket is recomputed with the new capacity.

```python
class ResizingHashMap[K, V](SimpleHashMap[K, V]):
    """Adds automatic resizing to keep the load factor below the limit."""

    MAX_LOAD_FACTOR = 0.75

    @property
    def load_factor(self) -> float:
        return len(self) / self.bucket_count

    def put(self, key: K, value: V) -> None:
        super().put(key, value)
        if self.load_factor > self.MAX_LOAD_FACTOR:
            self._resize(self.bucket_count * 2)

    def _resize(self, new_bucket_count: int) -> None:
        old_buckets = self._buckets
        self._buckets = [[] for _ in range(new_bucket_count)]
        for bucket in old_buckets:
            for key, value in bucket:
                self._buckets[hash(key) % new_bucket_count].append((key, value))


fast = ResizingHashMap[int, int]()
for number in range(1_000):
    fast.put(number, number * number)
print(fast.bucket_count, round(fast.load_factor, 2))  # 2048 0.49
```

Resizing costs `O(n)` but happens after exponentially more inserts, so inserts are `O(1)` amortized, the same argument as for dynamic arrays.

### Open addressing and tombstones

A lookup with linear probing looks like this:

```python
def probe_index(slots: list[object], capacity: int, start: int) -> int | None:
    """Return the first empty slot from `start`, wrapping around, or None if full."""
    for offset in range(capacity):
        index = (start + offset) % capacity
        if slots[index] is None:
            return index
    return None
```

Trace with capacity 5 and keys whose hashes map to slots as shown:

```text
insert A  (hash slot 2)  -> slot 2 is free, store A at 2
insert B  (hash slot 2)  -> slot 2 taken, try 3, store B at 3
insert C  (hash slot 3)  -> slot 3 taken, try 4, store C at 4
delete A                 -> mark slot 2 as a tombstone, not as empty
lookup B  (hash slot 2)  -> slot 2 is a tombstone, keep probing, find B at 3
```

If deleting A had set slot 2 to empty, the lookup for B would stop at slot 2 and wrongly report "not found". Tombstones preserve the probe chain. Too many tombstones slow lookups, so tables clean them up when they resize.

### How CPython stores a dict

A modern CPython `dict` keeps two arrays. A small sparse array of indexes is addressed by hash and uses open addressing with a probing sequence that mixes in more bits of the hash after each collision (the "perturbation"). A dense array of entries holds `(hash, key, value)` in insertion order. Lookup finds the index in the sparse array and then reads the entry. This design saves memory compared with older layouts and is why dictionaries remember insertion order. The table resizes when it is about two-thirds full. A deleted entry leaves a hole in the dense array that is reclaimed at the next resize. Sets use a similar open-addressing table without values.

### Hash values you can predict and ones you cannot

- Small integers hash to themselves, so `hash(5) == 5`. The exception is `-1`, which hashes to `-2` because `-1` is reserved internally as an error signal.
- Floats that equal an integer hash like that integer, which is why `1`, `1.0`, and `True` are the same dictionary key.
- Strings and bytes hash to a value that changes from run to run. You can see this by running a short command twice:

```bash
python3 -c "print(hash('apple'))"
python3 -c "print(hash('apple'))"
```

The two numbers differ. Setting the environment variable `PYTHONHASHSEED=0` makes them repeat, which is useful for reproducing a bug but should never be used in production.

Because of randomization, never store `hash(text)` in a database, use it as a stable shard number, or compare it across processes. For anything that must be stable, use a fixed algorithm from `hashlib`.

### Mutable keys lose their entries

```python
class Point:
    """A deliberately bad key: its hash depends on fields that can change."""

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

    def __eq__(self, other: object) -> bool:
        return isinstance(other, Point) and (self.x, self.y) == (other.x, other.y)

    def __hash__(self) -> int:
        return hash((self.x, self.y))


point = Point(1, 2)
visits = {point: "home"}
point.x = 99  # the stored entry is still in the slot for the old hash
print(point in visits)  # False: the new hash points to a different slot
print(Point(1, 2) in visits)  # False: right slot, but the stored key no longer equals it
print(len(visits))  # 1, the entry exists but can no longer be found
```

### Worst-case behavior

If every key has the same hash, all entries share one chain and every operation scans it, so inserting `n` keys costs `O(n^2)`. This is easy to demonstrate with integers, because `hash(2**61 - 1)` is 0, and so is the hash of every multiple of it.

```python
import time

n = 3_000
colliding = [i * (2**61 - 1) for i in range(n)]  # all of these hash to 0

for label, keys in (("distinct", list(range(n))), ("colliding", colliding)):
    started = time.perf_counter()
    table: dict[int, int] = {}
    for key in keys:
        table[key] = 1
    print(label, f"{time.perf_counter() - started:.4f}s")
```

On a typical machine the colliding keys take well over a hundred times longer, and doubling `n` roughly quadruples the time. Real attacks use the same idea against web servers that put attacker-chosen values (query parameters, JSON field names) into hash tables. Defenses are randomized string hashing (which CPython has used since version 3.3), limits on how many keys a request may contain, and rate limiting.

### Dict operations that cost more than they look

- `list(d)` and `d.items()` loops cost `O(n)`.
- Changing a dict's size while iterating over it raises `RuntimeError`. Iterate over `list(d)` if you must delete while looping.
- `d.pop(key)` is `O(1)` average, but `next(iter(d))` on a dict with many deletions can be slower than expected because iteration skips holes.
- Merging, `d1 | d2`, copies, so it is `O(n + m)`.

## Industry-level snippet

A payment API must be safe to retry. If a client sends the same `POST /charges` twice because of a network timeout, the server must charge once and return the same response both times. The standard solution is an idempotency key: the client sends a unique key with each logical request, and the server remembers the result under that key.

This implementation applies the ideas above:

- Lookups are by key, so a hash-table-style structure fits. An in-memory `dict` fronts the database as a cache, keyed by an immutable, hashable `CacheKey`.
- A SHA-256 fingerprint of the request body detects a key reused with a different body, which is a client bug that must be rejected rather than replayed.
- The database is the source of truth. `BEGIN IMMEDIATE` takes the write lock first, so two concurrent workers cannot both run the handler for the same key. If the handler fails, the transaction rolls back and nothing is recorded, so the client can retry.

```python
# idempotency.py
import hashlib
import logging
import sqlite3
from collections.abc import Callable
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS idempotency_record (
    scope TEXT NOT NULL,
    idempotency_key TEXT NOT NULL,
    fingerprint TEXT NOT NULL,
    response TEXT NOT NULL,
    PRIMARY KEY (scope, idempotency_key)
)
"""

SELECT_SQL = (
    "SELECT fingerprint, response FROM idempotency_record "
    "WHERE scope = ? AND idempotency_key = ?"
)

INSERT_SQL = (
    "INSERT INTO idempotency_record (scope, idempotency_key, fingerprint, response) "
    "VALUES (?, ?, ?, ?)"
)


class IdempotencyConflictError(Exception):
    """The same idempotency key was reused with a different request body."""


@dataclass(frozen=True, slots=True)
class CacheKey:
    scope: str
    key: str


@dataclass(frozen=True, slots=True)
class StoredResponse:
    fingerprint: str
    response: str


def fingerprint_of(body: bytes) -> str:
    """Stable fingerprint of a request body. hash() would change between runs."""
    return hashlib.sha256(body).hexdigest()


class IdempotencyStore:
    """Run a handler at most once per (scope, key) and replay its response."""

    def __init__(self, connection: sqlite3.Connection) -> None:
        self._connection = connection
        self._cache: dict[CacheKey, StoredResponse] = {}

    def execute(
        self,
        scope: str,
        key: str,
        body: bytes,
        handler: Callable[[], str],
    ) -> str:
        if not scope or not key:
            raise ValueError("scope and key must not be empty")
        cache_key = CacheKey(scope, key)
        fingerprint = fingerprint_of(body)

        cached = self._cache.get(cache_key)
        if cached is not None:
            return self._replay(cache_key, cached, fingerprint)

        try:
            with self._connection:
                self._connection.execute("BEGIN IMMEDIATE")  # take the write lock
                row = self._connection.execute(SELECT_SQL, (scope, key)).fetchone()
                if row is None:
                    response = handler()
                    self._connection.execute(
                        INSERT_SQL, (scope, key, fingerprint, response)
                    )
                    stored = StoredResponse(fingerprint, response)
                else:
                    stored = StoredResponse(str(row[0]), str(row[1]))
        except sqlite3.Error:
            logger.exception("idempotency store failed for %s", cache_key)
            raise
        self._cache[cache_key] = stored  # cache only after the commit succeeded
        return self._replay(cache_key, stored, fingerprint)

    def _replay(
        self, cache_key: CacheKey, stored: StoredResponse, fingerprint: str
    ) -> str:
        if stored.fingerprint != fingerprint:
            logger.warning("idempotency key reused with a different body: %s", cache_key)
            raise IdempotencyConflictError(
                f"key {cache_key.key!r} was already used with a different request"
            )
        return stored.response
```

```python
# test_idempotency.py
import sqlite3
from collections.abc import Iterator

import pytest

from idempotency import SCHEMA_SQL, IdempotencyConflictError, IdempotencyStore


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA_SQL)
    yield conn
    conn.close()


class CountingHandler:
    def __init__(self, response: str = "charged") -> None:
        self.calls = 0
        self._response = response

    def __call__(self) -> str:
        self.calls += 1
        return self._response


def test_replay_does_not_run_the_handler_again(
    connection: sqlite3.Connection,
) -> None:
    store = IdempotencyStore(connection)
    handler = CountingHandler()
    assert store.execute("tenant-1", "key-1", b'{"amount": 5}', handler) == "charged"
    assert store.execute("tenant-1", "key-1", b'{"amount": 5}', handler) == "charged"
    assert handler.calls == 1


def test_a_new_store_replays_from_the_database(
    connection: sqlite3.Connection,
) -> None:
    first = CountingHandler("first response")
    IdempotencyStore(connection).execute("tenant-1", "key-1", b"body", first)
    second = CountingHandler("second response")
    result = IdempotencyStore(connection).execute("tenant-1", "key-1", b"body", second)
    assert result == "first response"
    assert second.calls == 0


def test_reusing_a_key_with_a_different_body_is_rejected(
    connection: sqlite3.Connection,
) -> None:
    store = IdempotencyStore(connection)
    store.execute("tenant-1", "key-1", b"body-a", CountingHandler())
    with pytest.raises(IdempotencyConflictError):
        store.execute("tenant-1", "key-1", b"body-b", CountingHandler())


def test_scopes_do_not_share_keys(connection: sqlite3.Connection) -> None:
    store = IdempotencyStore(connection)
    one = CountingHandler("one")
    two = CountingHandler("two")
    assert store.execute("tenant-1", "key", b"x", one) == "one"
    assert store.execute("tenant-2", "key", b"x", two) == "two"


def test_failed_handler_is_not_recorded_so_a_retry_runs_it(
    connection: sqlite3.Connection,
) -> None:
    store = IdempotencyStore(connection)

    def failing() -> str:
        raise RuntimeError("payment provider timeout")

    with pytest.raises(RuntimeError):
        store.execute("tenant-1", "key-1", b"body", failing)

    retry = CountingHandler("charged on retry")
    assert store.execute("tenant-1", "key-1", b"body", retry) == "charged on retry"
    assert retry.calls == 1


def test_empty_scope_or_key_is_rejected(connection: sqlite3.Connection) -> None:
    store = IdempotencyStore(connection)
    with pytest.raises(ValueError):
        store.execute("", "key", b"x", CountingHandler())
    with pytest.raises(ValueError):
        store.execute("tenant", "", b"x", CountingHandler())
```

In a larger system the handler would not run while holding the database write lock. A common refinement records an "in progress" row first, commits, runs the handler, and then updates the row with the response. The structure of the lookup, the fingerprint, and the replay stays the same.

## Common mistakes

- Using a list where a dict or set is needed, then scanning with `in` inside a loop.
- Using a mutable object (list, dict, a class with changing fields) as a key. Convert lists to tuples, or make the class frozen.
- Writing `__eq__` without `__hash__`. Python then sets `__hash__` to `None`, and the objects become unhashable.
- Writing a `__hash__` that includes fields not used in `__eq__`, so equal objects can hash differently and the table loses them.
- Depending on `hash()` of a string across runs, machines, or processes. It is randomized.
- Assuming a dict or set is ordered by value or sorted. Dicts keep insertion order only. Sets have no defined order.
- Modifying a dict while iterating over it. Iterate over a copy of the keys.
- Using a floating-point number as a key and expecting `0.1 + 0.2` to find `0.3`. Floats that look equal may not be equal.
- Treating `O(1)` as free. Hashing a long string or a large tuple is `O(length)` each time, and a poor hash makes everything `O(n)`.
- Letting attackers choose keys without limits, which enables hash flooding and memory exhaustion.
- Forgetting that `d[key]` raises `KeyError` for a missing key. Use `d.get(key)` or `key in d`.

## Practice exercises

1. Use a `dict` to count how many times each word appears in a sentence, then repeat with `Counter`. Compare the code.
2. Write `first_duplicate(values)` that returns the first value whose second occurrence appears earliest. Use a set and state the time and space complexity.
3. Add `__iter__` and `items()` methods to `SimpleHashMap`, and test that every inserted pair is yielded exactly once.
4. Write a test that compares `ResizingHashMap` against a regular `dict` on 10,000 random operations (put, get, remove) generated with a seeded `random.Random`.
5. Implement `LinearProbingHashMap` with open addressing, tombstones, and resizing. Test it by inserting, deleting, and looking up keys that collide.
6. Print the bucket sizes of `SimpleHashMap` after inserting 1,000 random integers with 8, 64, and 1,024 buckets. Describe how the longest bucket changes.
7. Make a class with a `__hash__` that always returns 1 and compare insert time with a normal key class for 2,000 keys. Explain the results in terms of collisions.
8. Show that `{1: "a", 1.0: "b", True: "c"}` has one entry. Predict `len({0, False, 0.0, ""})` before running it.
9. Write a class `Version` with fields `major`, `minor`, and `patch` that is hashable and compares equal when all fields match. Use a frozen dataclass, then rewrite it by hand with `__eq__` and `__hash__`.
10. Write `two_sum(numbers, target)` with a dictionary and explain why storing indexes in the dictionary lets you return positions.
11. Write `group_by_first_letter(words)` using `dict.setdefault`, and again with `defaultdict(list)`. Say which you prefer and why.
12. Show with a short experiment that `hash("apple")` differs between two runs, and that `PYTHONHASHSEED=0` makes it repeat.
13. Extend `IdempotencyStore` with an expiry time per record and a method that deletes expired rows in a transaction, using an injectable clock function so tests do not need `time.sleep`.
14. Explain why an attacker could slow down a server that puts attacker-chosen integer keys in a `dict`, and list two defenses.

## Summary

A hash table maps keys to slots through a hash function and gives average `O(1)` lookup, insert, and delete. Collisions are inevitable, so tables use chaining or open addressing, and they resize when the load factor gets high, which makes inserts `O(1)` amortized. CPython's `dict` stores a sparse index array and a dense entry array, so it preserves insertion order, and `set` is the same idea without values. Keys must be hashable, equal objects must have equal hashes, and keys must not change while stored. String hashes are randomized per process, so use `hashlib` for anything stable. A bad or attacker-controlled hash distribution degrades a table to `O(n)` per operation, which is why limits on untrusted keys matter.
