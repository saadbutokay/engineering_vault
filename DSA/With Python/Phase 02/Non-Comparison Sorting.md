## Overview

The previous note proved that no sort that works only by comparing items can beat `n log n`. That proof has an escape hatch: it applies to algorithms that learn about the data only through comparisons. If you know something extra about the keys, for example that they are integers in a small range, or fixed-width codes, you can sort without comparing items to each other at all and reach `O(n)` time.

Three classic algorithms use this idea. Counting sort tallies how many times each key occurs and then writes the items out in order. Radix sort sorts by one digit or byte at a time using a stable counting sort. Bucket sort scatters items into ranges, sorts each small range, and concatenates the results.

These algorithms are not general replacements for `sorted`. They need the right kind of data, they trade memory for speed, and they are most useful in specific jobs: sorting ages, scores, status codes, IP addresses, fixed-width identifiers, and values spread evenly across a known range. This note covers when each one beats `n log n`, how to implement them, what they cost in memory, and how to apply them to integers, strings, and fixed-width keys.

## Core concepts

### Why they can beat n log n

A comparison sort extracts one bit of information per comparison and needs about `log2(n!)` bits to find the correct order. A non-comparison sort uses the key itself, for example as an index into an array, so one operation gives far more than one bit of information. The lower bound does not apply, because these algorithms are not comparison sorts.

### Counting sort

Counting sort works on integer keys in a known range `0..k`.

1. Count how many items have each key in an array of size `k + 1`.
2. Turn the counts into starting positions with a running total.
3. Place each item at the position for its key and advance that position.

Time is `O(n + k)` and extra space is `O(n + k)`. It is a good choice when `k` is not much larger than `n`. It is stable when the placement pass goes through the input from left to right.

### Radix sort

Radix sort sorts integers (or fixed-width strings) one digit at a time. In least significant digit (LSD) radix sort, you sort by the last digit first and the first digit last. Each pass must be a stable sort, usually counting sort on that digit. Because later passes are stable, they preserve the order established by the earlier ones, and after the final pass the whole key is sorted.

For `n` numbers with `d` digits in base `b`, the cost is `O(d * (n + b))`. For 32-bit integers with base 256, that is four passes.

### Bucket sort

Bucket sort assumes the values are spread roughly evenly over a known range. It divides the range into buckets, puts each item in its bucket, sorts each bucket (usually with insertion sort or the built-in sort), and joins the buckets in order. If the items are uniformly distributed, each bucket holds a few items and the expected time is `O(n)`. If they all land in one bucket, it degrades to the cost of the inner sort.

### When each one wins

- Counting sort: integer keys with a small range compared with `n` (ages, ratings 1 to 5, HTTP status codes, byte values).
- Radix sort: many integers or fixed-width keys with a large range but few digits (IPv4 addresses, 64-bit IDs, zero-padded codes, timestamps).
- Bucket sort: floating-point values or other data that is evenly distributed over a known range.
- Otherwise: use `sorted`.

### Memory trade-offs

Non-comparison sorts buy speed with memory. Counting sort needs an array as large as the key range, which is hopeless for a range of `10^9` with only 1,000 items. Radix sort and counting sort also need an output array the size of the input. Comparison sorts like quicksort run in place.

## Basic syntax examples

### Counting sort for small integers

```python
def counting_sort(values: list[int], max_value: int) -> list[int]:
    """Sort integers in 0..max_value. O(n + k) time, O(k) extra space."""
    if max_value < 0:
        raise ValueError("max_value must not be negative")
    counts = [0] * (max_value + 1)
    for value in values:
        if not 0 <= value <= max_value:
            raise ValueError(f"{value} is outside 0..{max_value}")
        counts[value] += 1
    result: list[int] = []
    for value, count in enumerate(counts):
        result.extend([value] * count)  # write each value as often as it occurred
    return result


print(counting_sort([3, 1, 4, 1, 5, 2, 3, 3], 5))  # [1, 1, 2, 3, 3, 3, 4, 5]
```

Trace of the counting step for `[3, 1, 4, 1, 3]` with `max_value = 4`:

```text
counts = [0, 0, 0, 0, 0]
read 3 -> [0, 0, 0, 1, 0]
read 1 -> [0, 1, 0, 1, 0]
read 4 -> [0, 1, 0, 1, 1]
read 1 -> [0, 2, 0, 1, 1]
read 3 -> [0, 2, 0, 2, 1]
output: 1 twice, 3 twice, 4 once -> [1, 1, 3, 3, 4]
```

### Stable counting sort for records

When you sort records by a small integer key, you must move the records, not just count the keys, and the sort must be stable. The version with running totals does both.

```python
from collections.abc import Callable, Sequence
from typing import cast


def counting_sort_by[T](
    items: Sequence[T], key: Callable[[T], int], max_key: int
) -> list[T]:
    """Stable sort of items by an integer key in 0..max_key. O(n + k)."""
    counts = [0] * (max_key + 2)
    for item in items:
        item_key = key(item)
        if not 0 <= item_key <= max_key:
            raise ValueError(f"key {item_key} is outside 0..{max_key}")
        counts[item_key + 1] += 1
    for i in range(1, len(counts)):
        counts[i] += counts[i - 1]  # counts[k] is now where key k starts in the output
    output: list[T | None] = [None] * len(items)
    for item in items:  # left to right keeps equal keys in their original order
        item_key = key(item)
        output[counts[item_key]] = item
        counts[item_key] += 1
    return cast(list[T], output)


students = [("ana", 3), ("bo", 1), ("cy", 3), ("di", 2), ("ed", 1)]
print(counting_sort_by(students, key=lambda student: student[1], max_key=3))
# [('bo', 1), ('ed', 1), ('di', 2), ('ana', 3), ('cy', 3)]
```

### Radix sort for non-negative integers

```python
def radix_sort(values: Sequence[int], base: int = 10) -> list[int]:
    """LSD radix sort. O(d * (n + base)) for numbers with d digits in this base."""
    if base < 2:
        raise ValueError("base must be at least 2")
    if any(value < 0 for value in values):
        raise ValueError("values must not be negative")
    result = list(values)
    if not result:
        return result
    largest = max(result)
    place = 1
    while largest // place > 0:
        result = counting_sort_by(
            result, key=lambda value: (value // place) % base, max_key=base - 1
        )
        place *= base
    return result


print(radix_sort([170, 45, 75, 90, 802, 24, 2, 66]))
# [2, 24, 45, 66, 75, 90, 170, 802]
```

How the passes work on `[170, 45, 75, 90, 802, 24, 2, 66]` with base 10:

```text
start:         170 045 075 090 802 024 002 066
by ones digit: 170 090 802 002 024 045 075 066
by tens digit: 802 002 024 045 066 170 075 090
by hundreds:   002 024 045 066 075 090 170 802
```

After the second pass the numbers are ordered by their last two digits. The hundreds pass is stable, so numbers with the same hundreds digit keep that order.

### Sorting fixed-width strings

The same idea sorts equal-length strings by looking at one character position per pass, from the last position to the first.

```python
def sort_fixed_width_codes(codes: list[str]) -> list[str]:
    """Sort equal-length ASCII strings with one stable pass per character."""
    if not codes:
        return []
    width = len(codes[0])
    if any(len(code) != width or not code.isascii() for code in codes):
        raise ValueError("codes must be ASCII strings of equal length")
    result = list(codes)
    for position in range(width - 1, -1, -1):
        result = counting_sort_by(
            result, key=lambda code: ord(code[position]), max_key=127
        )
    return result


print(sort_fixed_width_codes(["B2X", "A9Z", "B1X", "A9A"]))
# ['A9A', 'A9Z', 'B1X', 'B2X']
```

### Bucket sort for floats in [0, 1)

```python
def bucket_sort(values: Sequence[float], bucket_count: int = 10) -> list[float]:
    """Expected O(n) for uniformly distributed values in [0, 1)."""
    if bucket_count < 1:
        raise ValueError("bucket_count must be at least 1")
    buckets: list[list[float]] = [[] for _ in range(bucket_count)]
    for value in values:
        if not 0.0 <= value < 1.0:
            raise ValueError(f"{value} is outside [0, 1)")
        buckets[min(int(value * bucket_count), bucket_count - 1)].append(value)
    result: list[float] = []
    for bucket in buckets:
        bucket.sort()  # each bucket is small when the data is spread evenly
        result.extend(bucket)
    return result


print(bucket_sort([0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12]))
# [0.12, 0.17, 0.21, 0.26, 0.39, 0.72, 0.78, 0.94]
```

## Deeper behavior

### Why the prefix-sum placement works

After the counting pass, `counts[k + 1]` holds how many items have key `k`. The running total turns `counts[k]` into "how many items have a key smaller than `k`", which is exactly the index where the first item with key `k` belongs. Each time an item with key `k` is placed, the pointer for `k` advances, so the next one goes right after it. Processing the input left to right places equal keys in input order, which makes the sort stable.

If you run the placement pass from right to left, you must also decrement positions from the end of each key's range to stay stable. Mixing the two directions is a common source of unstable or off-by-one results.

### Stability is what makes radix sort work

Suppose you sort by ones, then tens. After the tens pass, two numbers with the same tens digit must still be in ones-digit order. That is guaranteed only if the tens pass is stable. An unstable pass would destroy the work of the earlier passes. This is why radix sort is built on counting sort and not on a quicksort-style pass.

MSD (most significant digit first) radix sort goes the other way: it splits by the first digit and recursively sorts each group. It can stop early for variable-length data such as strings with different lengths, but it needs recursion and bucket bookkeeping.

### Choosing the base

Using base 10 on 32-bit numbers needs 10 passes. Base 256 needs only 4, each with a 256-slot count array that fits comfortably in memory. Larger bases mean fewer passes but a bigger count array. The best base is usually around the size of the data, balancing passes against table size. Powers of two let you replace division with bit shifts and masks: `(value >> shift) & 0xFF`.

### Negative numbers and offsets

Counting sort needs keys starting at 0. For a range `low..high`, subtract `low` from every key and add it back at the end, so the table has `high - low + 1` slots. For radix sort with negative numbers, sort the negatives and non-negatives separately, or add an offset that makes everything non-negative.

### The cost of a large range

Counting sort costs `O(n + k)` memory and time. Sorting 1,000 user IDs that range up to `10^9` would need a table of a billion slots. That is `O(k)` with a huge `k`, and any comparison sort is far better. Radix sort fixes this by sorting one digit at a time, so the table is the size of the base, not the size of the range.

### Bucket sort needs a distribution assumption

Bucket sort is fast only if the values are spread evenly. If most values fall in one bucket, the expected linear time becomes the cost of the inner sort on that bucket. Choose the bucket boundaries from knowledge of the data. Latencies, for example, are heavily skewed toward small values, so equal-width buckets are a poor choice there.

### Python reality check

Python's `sorted` is written in C, so it carries a large constant-factor advantage over any loop written in Python. Counting sort still wins when its work is dominated by one tight loop and the range is small. In one test of 1,000,000 integers in the range 0 to 999, a simple counting sort was about three to four times faster than `sorted`. With a range of 100,000,000 it was not an option, because the table alone would need 100 million slots. Radix sort in pure Python, with several passes of Python-level loops, often loses to `sorted` unless the data is very large. Measure before you choose, and choose a non-comparison sort because the data fits it, not because it sounds faster.

A counting sort over a `Counter` is a compact way to get the same effect for sparse keys: `Counter` counts, then `sorted(counter)` orders the distinct keys only.

```python
from collections import Counter

counts = Counter([5, 3, 5, 1, 3, 5])
ordered = [value for value in sorted(counts) for _ in range(counts[value])]
print(ordered)  # [1, 3, 3, 5, 5, 5]
```

The cost is `O(n + d log d)` for `d` distinct values, which beats `n log n` when there are few distinct values.

### Complexity summary

- Counting sort: `O(n + k)` time and space, stable, `k` is the key range.
- Radix sort (LSD): `O(d * (n + b))` time, `O(n + b)` space, stable, `d` digits in base `b`.
- Bucket sort: expected `O(n)` for uniform data, `O(n^2)` worst case with insertion sort in the buckets, stable if the inner sort is stable.
- Comparison sorts: `O(n log n)` best possible in the worst case, any data that can be compared.

## Industry-level snippet

A security team maintains a blocklist of abusive IPv4 addresses, collected from several noisy sources. Addresses arrive as text, with duplicates and mistakes. The blocklist is stored as sorted 32-bit integers so that lookups can use binary search and updates can be bulk loaded in key order, which is fast for the database.

An IPv4 address is exactly four bytes, which is the ideal input for LSD radix sort with base 256: four passes, each a counting sort on one byte with a table of just 256 slots. Parsing is strict. Text such as `"1.2.3"`, `"1.2.3.256"`, or `"01.2.3.4"` is rejected rather than guessed at. All the bad addresses are reported together, so an operator can fix a whole file at once, and nothing is stored if any address is invalid.

```python
# ip_blocklist.py
import logging
import sqlite3
from collections.abc import Callable, Iterable, Sequence
from typing import cast

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS blocked_ip (
    address INTEGER PRIMARY KEY
)
"""

INSERT_SQL = "INSERT INTO blocked_ip (address) VALUES (?) ON CONFLICT DO NOTHING"


class InvalidAddressError(ValueError):
    """Raised when input contains addresses that are not valid dotted IPv4."""


def parse_ipv4(text: str) -> int:
    """Strict dotted-quad parser: four ASCII decimal numbers, no leading zeros."""
    pieces = text.strip().split(".")
    if len(pieces) != 4:
        raise InvalidAddressError(f"{text!r} does not have four parts")
    value = 0
    for piece in pieces:
        if not piece or not all("0" <= char <= "9" for char in piece):
            raise InvalidAddressError(f"{text!r} has a non-numeric part")
        if len(piece) > 1 and piece[0] == "0":
            raise InvalidAddressError(f"{text!r} has a leading zero")
        octet = int(piece)
        if octet > 255:
            raise InvalidAddressError(f"{text!r} has a part above 255")
        value = value * 256 + octet
    return value


def format_ipv4(value: int) -> str:
    if not 0 <= value < 2**32:
        raise ValueError(f"{value} is not a 32-bit address")
    return ".".join(str((value >> shift) & 0xFF) for shift in (24, 16, 8, 0))


def counting_sort_by[T](
    items: Sequence[T], key: Callable[[T], int], max_key: int
) -> list[T]:
    counts = [0] * (max_key + 2)
    for item in items:
        counts[key(item) + 1] += 1
    for i in range(1, len(counts)):
        counts[i] += counts[i - 1]
    output: list[T | None] = [None] * len(items)
    for item in items:  # left to right keeps every pass stable
        item_key = key(item)
        output[counts[item_key]] = item
        counts[item_key] += 1
    return cast(list[T], output)


def radix_sort_32bit(values: Sequence[int]) -> list[int]:
    """Four stable counting-sort passes, one per byte, least significant first."""
    result = list(values)
    for shift in (0, 8, 16, 24):
        result = counting_sort_by(
            result, key=lambda value: (value >> shift) & 0xFF, max_key=255
        )
    return result


def sort_unique_addresses(addresses: Iterable[str]) -> list[int]:
    """Parse, radix sort, and remove duplicates. Reports every bad address at once."""
    parsed: list[int] = []
    bad: list[str] = []
    for text in addresses:
        try:
            parsed.append(parse_ipv4(text))
        except InvalidAddressError:
            bad.append(text)
    if bad:
        shown = ", ".join(repr(text) for text in bad[:3])
        raise InvalidAddressError(f"{len(bad)} invalid addresses, for example {shown}")
    ordered = radix_sort_32bit(parsed)
    return [
        value
        for index, value in enumerate(ordered)
        if index == 0 or value != ordered[index - 1]
    ]


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def add_to_blocklist(connection: sqlite3.Connection, addresses: Iterable[str]) -> int:
    """Insert addresses in sorted order inside one transaction. Returns new rows."""
    ordered = sort_unique_addresses(addresses)  # nothing is stored if parsing fails
    try:
        with connection:
            before = connection.total_changes
            connection.executemany(INSERT_SQL, [(value,) for value in ordered])
            inserted = connection.total_changes - before
    except sqlite3.Error:
        logger.exception("could not store %d blocked addresses", len(ordered))
        raise
    logger.info("blocklist gained %d of %d addresses", inserted, len(ordered))
    return inserted
```

```python
# test_ip_blocklist.py
import ipaddress
import random
import sqlite3
from collections.abc import Iterator

import pytest

from ip_blocklist import (
    InvalidAddressError,
    add_to_blocklist,
    create_schema,
    format_ipv4,
    parse_ipv4,
    radix_sort_32bit,
    sort_unique_addresses,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


@pytest.mark.parametrize(
    ("text", "expected"),
    [
        ("0.0.0.0", 0),
        ("0.0.0.1", 1),
        ("1.0.0.0", 2**24),
        ("192.168.1.10", 3_232_235_786),
        ("255.255.255.255", 2**32 - 1),
        (" 10.0.0.1 ", 167_772_161),
    ],
)
def test_parse_valid_addresses(text: str, expected: int) -> None:
    assert parse_ipv4(text) == expected
    assert parse_ipv4(text) == int(ipaddress.IPv4Address(text.strip()))


@pytest.mark.parametrize(
    "text",
    ["", "1.2.3", "1.2.3.4.5", "1.2.3.256", "01.2.3.4", "1.2.3.-4", "a.b.c", "1..3.4"],
)
def test_parse_rejects_bad_addresses(text: str) -> None:
    with pytest.raises(InvalidAddressError):
        parse_ipv4(text)


def test_format_round_trip() -> None:
    for value in (0, 1, 255, 256, 3_232_235_786, 2**32 - 1):
        assert parse_ipv4(format_ipv4(value)) == value


def test_format_rejects_out_of_range_values() -> None:
    with pytest.raises(ValueError):
        format_ipv4(-1)
    with pytest.raises(ValueError):
        format_ipv4(2**32)


def test_radix_sort_matches_sorted_on_random_input() -> None:
    rng = random.Random(9)
    for _ in range(200):
        values = [rng.randrange(2**32) for _ in range(rng.randint(0, 60))]
        assert radix_sort_32bit(values) == sorted(values)


def test_sort_unique_addresses_orders_numerically_not_as_text() -> None:
    addresses = ["10.0.0.2", "9.255.255.255", "10.0.0.2", "10.0.0.10"]
    result = sort_unique_addresses(addresses)
    assert [format_ipv4(value) for value in result] == [
        "9.255.255.255",
        "10.0.0.2",
        "10.0.0.10",
    ]


def test_every_bad_address_is_reported_together() -> None:
    with pytest.raises(InvalidAddressError, match="2 invalid"):
        sort_unique_addresses(["1.2.3.4", "nope", "300.1.1.1"])


def test_blocklist_is_idempotent(connection: sqlite3.Connection) -> None:
    assert add_to_blocklist(connection, ["1.1.1.1", "2.2.2.2", "1.1.1.1"]) == 2
    assert add_to_blocklist(connection, ["2.2.2.2", "3.3.3.3"]) == 1
    rows = connection.execute("SELECT address FROM blocked_ip ORDER BY address")
    assert [format_ipv4(row[0]) for row in rows] == ["1.1.1.1", "2.2.2.2", "3.3.3.3"]


def test_bad_input_stores_nothing(connection: sqlite3.Connection) -> None:
    with pytest.raises(InvalidAddressError):
        add_to_blocklist(connection, ["1.1.1.1", "bad"])
    count = connection.execute("SELECT COUNT(*) FROM blocked_ip").fetchone()[0]
    assert count == 0
```

`test_sort_unique_addresses_orders_numerically_not_as_text` is the kind of test that justifies the whole design. A text sort would place `"10.0.0.10"` before `"10.0.0.2"` and `"9.255.255.255"` last.

## Common mistakes

- Using counting sort when the key range is huge compared with `n`, and running out of memory.
- Forgetting that counting sort needs non-negative keys, and not applying an offset for negative values.
- Placing items with a pass that is not stable (for example iterating from the right without adjusting positions), which breaks radix sort.
- Sorting radix passes from the most significant digit first with a stable LSD routine. LSD must start at the least significant digit.
- Off-by-one errors in the count array size: it needs `max_key + 1` slots for counting values, and `max_key + 2` for the running-total version.
- Using bucket sort on skewed data with equal-width buckets, so nearly every item lands in one bucket.
- Sorting IP addresses, version numbers, or zero-padded numbers as plain text and getting `"10.0.0.10"` before `"10.0.0.2"`.
- Assuming a non-comparison sort is always faster than `sorted`. Python-level loops are slow, and the built-in is written in C.
- Parsing input leniently (accepting `"01.2.3.4"` or Unicode digits) and storing something different from what was meant.
- Forgetting to validate that the keys are in range before indexing into the count array. An out-of-range key raises `IndexError` or, with a negative key, silently indexes from the end.

## Practice exercises

1. Trace counting sort by hand on `[4, 2, 2, 8, 3, 3, 1]` and write the count array after each step.
2. Write `counting_sort_with_offset(values)` that sorts integers of any range by subtracting the minimum value, and test it with negative numbers.
3. Sort a list of 100,000 exam scores between 0 and 100 with `counting_sort`, then time it against `sorted`. Explain the result.
4. Write `sort_by_age(people)` that stably sorts `(name, age)` records with ages from 0 to 120 using `counting_sort_by`, and test that equal ages keep their input order.
5. Write `radix_sort_negative(values)` that handles negative numbers by sorting negatives and non-negatives separately, and test it on random mixed lists.
6. Implement LSD radix sort with a bit-mask base of 256 using shifts and masks, and compare its speed with base 10 on 200,000 random 32-bit integers.
7. Write `sort_zero_padded_ids(ids)` for strings like `"000417"` and `"002003"`, using a character-by-character radix pass, and explain why this works for equal-length strings but not for strings of different lengths.
8. Implement MSD radix sort for lowercase strings of different lengths by recursing on the character at a given position, treating "end of string" as the smallest value.
9. Implement `bucket_sort` for floats in `[0, 1)` and test its running time on uniform data and on data where every value is below `0.1`. Explain the difference.
10. Write `top_k_ratings(reviews, k)` for ratings from 1 to 5 using counting, and state its complexity compared with sorting the reviews.
11. Show with a counterexample that an unstable pass in radix sort gives wrong results, using two-digit numbers and a deliberately unstable inner sort.
12. Write `sort_http_status_codes(codes)` using counting sort with the range 100 to 599 and an offset, and explain what makes this a good use of the algorithm.
13. Extend `ip_blocklist` with `remove_from_blocklist(connection, addresses)` that also parses strictly and deletes in one transaction, and write tests.

## Summary

Non-comparison sorts get around the `n log n` lower bound by using the keys themselves instead of comparing items. Counting sort tallies keys in a table, turns the tallies into positions with a running total, and places items stably in `O(n + k)` time. LSD radix sort applies a stable counting sort to one digit or byte at a time, so the table size is the base instead of the key range and the cost is `O(d * (n + b))`. Bucket sort scatters values over equal ranges and is expected linear for uniform data. All of them trade memory for speed, need keys with the right structure (small integers, fixed-width keys, evenly spread floats), and must keep each pass stable. In Python, `sorted` is a strong default, and the specialized sorts pay off for data such as small-range integers or IP addresses, where measurement confirms the benefit.
