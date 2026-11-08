## Overview

Many problems ask about a contiguous stretch of a list or string: the best run of seven days, the longest substring with no repeated character, the shortest stretch that contains everything you need. The obvious solution examines every possible stretch, which is `O(n^2)` or worse. The sliding window technique examines each stretch incrementally. It keeps a window, a range `[left, right]`, and moves it along the data while updating a summary of what is inside, so each element is added once and removed once.

There are two forms. A fixed-size window has a known length `k`: slide it one step by adding the new element on the right and removing the old one on the left. A variable-size window grows by moving `right` and shrinks by moving `left` whenever a condition is violated, and it looks for the longest or shortest window that satisfies a rule.

This note covers fixed windows, variable windows, windows with a hash map, the longest substring without repeats, the minimum window substring, at most `K` distinct elements, shrink conditions, and the difference between sliding windows and prefix sums. The central idea is amortized cost: although there is a loop inside a loop, both pointers only move forward, so the total work is `O(n)`.

## Core concepts

### The window and its summary

A window is a contiguous range of the data. To update it cheaply, keep a summary that you can adjust in `O(1)` when an element enters or leaves: a running sum, a count of each character, the number of distinct values, a count of how many requirements are still missing.

### Fixed-size windows

The window length `k` never changes. Compute the summary for the first window, then for each step add the incoming element and subtract the outgoing one. Typical problems: the maximum sum of `k` consecutive items, the average of every `k`-length window, and counting windows that satisfy a condition.

### Variable-size windows

The right pointer expands the window one element at a time. After each expansion, the window may violate a rule. While it does, the left pointer shrinks the window. After shrinking, the window is valid again and you record a result.

- Longest window satisfying a rule: record the length after restoring validity.
- Shortest window satisfying a rule: record the length while the window is valid, then shrink as far as it stays valid.

### Windows with a hash map

When the rule involves characters or values (distinct count, repeats, required letters), the summary is a dictionary from value to count. Entering increments a count, leaving decrements it, and a count of zero is removed so that `len(counts)` means the number of distinct values.

### Longest substring without repeating characters

Slide a window over the string while keeping every character in it unique. When the new character is already in the window, move `left` just past the earlier occurrence.

### Minimum window substring

Given a text and a pattern, find the shortest part of the text that contains every character of the pattern, including repeats. Expand until the window covers the pattern, then shrink from the left while it still does.

### At most K distinct elements

Keep a dictionary of counts. If `len(counts)` exceeds `K`, shrink from the left until it does not. This is the template for many "at most" problems.

### Shrink conditions

Every variable-window problem needs a precise shrink condition. Write it as a sentence before coding: "while the window has more than `K` distinct characters, remove the leftmost one." The `while` loop (not an `if`) is essential, because one new element can require removing several old ones.

### Sliding window versus prefix sums

Both handle contiguous ranges, but they need different properties.

- Sliding window works when extending the window moves a property in one direction and shrinking moves it the other way (for example, sums of non-negative numbers only grow when you extend).
- Prefix sums with a hash map work for any numbers, including negative ones, because they rely only on an equation between two prefix values.

## Basic syntax examples

### Fixed window: maximum sum of k consecutive items

```python
def max_sum_window(values: list[int], k: int) -> int:
    """Largest sum of any k consecutive items. O(n) time, O(1) space."""
    if not 1 <= k <= len(values):
        raise ValueError("k must be between 1 and len(values)")
    window = sum(values[:k])
    best = window
    for right in range(k, len(values)):
        window += values[right] - values[right - k]  # add the new item, drop the old
        best = max(best, window)
    return best


print(max_sum_window([2, 1, 5, 1, 3, 2], 3))  # 9, from 5 + 1 + 3
```

Trace of `max_sum_window([2, 1, 5, 1, 3, 2], 3)`:

```text
first window [2, 1, 5]  sum 8
slide: + 1 - 2 = 7   window [1, 5, 1]
slide: + 3 - 1 = 9   window [5, 1, 3]   best 9
slide: + 2 - 5 = 6   window [1, 3, 2]
answer 9
```

### Variable window: longest subarray with sum at most a limit

This version requires non-negative numbers, so removing items never increases the sum.

```python
def longest_subarray_at_most(values: list[int], limit: int) -> int:
    """Length of the longest subarray whose sum is at most limit.

    Requires non-negative values. O(n) time, O(1) space.
    """
    if any(value < 0 for value in values):
        raise ValueError("values must be non-negative")
    left = 0
    total = 0
    best = 0
    for right, value in enumerate(values):
        total += value
        while total > limit:  # shrink until the window is valid again
            total -= values[left]
            left += 1
        best = max(best, right - left + 1)
    return best


print(longest_subarray_at_most([1, 2, 1, 0, 1, 1, 0], 4))  # 5
```

### Longest substring without repeating characters

```python
def longest_unique_substring(text: str) -> int:
    """Length of the longest substring with all distinct characters. O(n)."""
    last_seen: dict[str, int] = {}
    start = 0  # the left edge of the current window
    best = 0
    for index, char in enumerate(text):
        if char in last_seen and last_seen[char] >= start:
            start = last_seen[char] + 1  # jump just past the earlier copy
        last_seen[char] = index
        best = max(best, index - start + 1)
    return best


print(longest_unique_substring("abcabcbb"))  # 3, the substring "abc"
print(longest_unique_substring("pwwkew"))  # 3, the substring "wke"
```

### At most K distinct characters

```python
def longest_with_at_most_k_distinct(text: str, k: int) -> int:
    """Longest substring containing at most k distinct characters. O(n)."""
    if k < 0:
        raise ValueError("k must not be negative")
    counts: dict[str, int] = {}
    left = 0
    best = 0
    for right, char in enumerate(text):
        counts[char] = counts.get(char, 0) + 1
        while len(counts) > k:
            leaving = text[left]
            counts[leaving] -= 1
            if counts[leaving] == 0:
                del counts[leaving]  # so len(counts) is the distinct count
            left += 1
        best = max(best, right - left + 1)
    return best


print(longest_with_at_most_k_distinct("eceba", 2))  # 3, the substring "ece"
```

### Minimum window substring

```python
from collections import Counter


def min_window(text: str, pattern: str) -> str:
    """Shortest part of text containing every character of pattern. O(n + m)."""
    if not pattern or len(pattern) > len(text):
        return ""
    need = Counter(pattern)  # how many of each character are still required
    missing = len(pattern)  # how many required characters are still missing
    best_start = 0
    best_length = len(text) + 1
    left = 0
    for right, char in enumerate(text):
        if need[char] > 0:
            missing -= 1  # this character fills a real requirement
        need[char] -= 1  # may go negative for characters we have in surplus
        while missing == 0:  # the window is valid: try to shrink it
            if right - left + 1 < best_length:
                best_start, best_length = left, right - left + 1
            need[text[left]] += 1
            if need[text[left]] > 0:
                missing += 1  # we just removed a required character
            left += 1
    if best_length > len(text):
        return ""
    return text[best_start : best_start + best_length]


print(min_window("ADOBECODEBANC", "ABC"))  # BANC
```

## Deeper behavior

### Why the total work is linear

The variable-window code has a `while` inside a `for`, which looks quadratic. It is not. The `right` pointer moves `n` times in total. The `left` pointer only ever moves forward and never passes `right`, so it moves at most `n` times in total. The `while` loop body therefore runs at most `n` times across the whole run, not `n` times per step. Total work: at most `2n` pointer moves, which is `O(n)`. This is amortized analysis: some iterations do a lot of shrinking, but the shrinking is paid for by the earlier expansions.

### The invariant that makes it correct

Write down what is true after the shrink loop, every time:

- Longest-window problems: after the shrink loop, the window `[left, right]` is valid. It is the longest valid window that ends at `right`, because `left` was moved only as far as needed.
- Shortest-window problems: inside the loop, the window is valid, and the loop keeps shrinking while it stays valid, so when it exits the window is as small as it can be for this `right`.

Checking that the code maintains such a statement is how you find off-by-one mistakes.

### Why shrinking needs monotonicity

Shrinking is justified only when "removing an element from the left makes the window closer to valid" holds in a predictable direction. With non-negative numbers and a "sum at most `limit`" rule, removing an element can only lower the sum. With negative numbers, removing an element could raise the sum, so shrinking might not restore validity and expanding might not make the window worse. Sliding windows then fail, and the prefix-sum approach from the previous topic is needed.

### Counting windows instead of finding one

To count the subarrays with at most `k` distinct values, every time the window is valid for a given `right`, all windows that end at `right` and start between `left` and `right` are valid, so add `right - left + 1`.

```python
def count_at_most_k_distinct(values: list[int], k: int) -> int:
    """Number of subarrays containing at most k distinct values."""
    counts: dict[int, int] = {}
    left = 0
    total = 0
    for right, value in enumerate(values):
        counts[value] = counts.get(value, 0) + 1
        while len(counts) > k:
            leaving = values[left]
            counts[leaving] -= 1
            if counts[leaving] == 0:
                del counts[leaving]
            left += 1
        total += right - left + 1  # every start from left to right is valid
    return total


def count_exactly_k_distinct(values: list[int], k: int) -> int:
    """Exactly k is 'at most k' minus 'at most k - 1'."""
    return count_at_most_k_distinct(values, k) - count_at_most_k_distinct(values, k - 1)


print(count_exactly_k_distinct([1, 2, 1, 2, 3], 2))  # 7
```

Counting "exactly `k`" directly with a window is awkward because the window is not monotonic for it, but the difference of two "at most" counts is easy and still `O(n)`.

### Minimum window bookkeeping

The `need` counter in `min_window` can go negative. A value of `-2` for a character means the window holds two more copies of it than the pattern requires. Only a positive count represents an unmet requirement, which is why `missing` changes only when the count crosses between zero and positive. This lets a single integer, `missing`, answer "does the window cover the pattern?" in `O(1)` instead of comparing two dictionaries each step.

### Choosing the summary

- A running sum for sums.
- A dictionary of counts for character and value problems.
- A deque of indexes for window maximum and minimum, which keeps candidates in order. That monotonic deque technique appears with queues later in the roadmap.
- A set when only uniqueness matters and the window never has duplicates.

### Time and space

All the variable-window examples run in `O(n)` time. Space is `O(1)` for sums, and `O(k)` or `O(alphabet)` for dictionaries of counts, where `k` is the number of distinct values allowed in the window.

### Slicing inside a window is a hidden cost

Do not compute `sum(values[left : right + 1])` inside the loop. It turns an `O(n)` algorithm into `O(n^2)` again. Maintain the summary incrementally.

## Industry-level snippet

A login service wants to flag possible password-guessing attacks: any account that accumulates more than a set number of failed logins inside a short window of time. Scanning every pair of failures is quadratic. A sliding window over each user's sorted failure timestamps flags the same accounts in one linear pass: the right pointer takes in each failure, and the left pointer drops the failures that are too old.

The detector accepts failures in any order, groups them by user, sorts each user's timestamps, and slides the window. It reports the first window that violates the limit for each user. Alerts are saved in a transaction with a unique constraint, so reprocessing the same logs never creates duplicate alerts.

```python
# burst_detector.py
import logging
import sqlite3
from collections import defaultdict
from collections.abc import Iterable, Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS login_alert (
    user_id INTEGER NOT NULL,
    window_start INTEGER NOT NULL,
    window_end INTEGER NOT NULL,
    failures INTEGER NOT NULL,
    PRIMARY KEY (user_id, window_start)
)
"""

INSERT_SQL = (
    "INSERT INTO login_alert (user_id, window_start, window_end, failures) "
    "VALUES (?, ?, ?, ?) ON CONFLICT (user_id, window_start) DO NOTHING"
)


@dataclass(frozen=True, slots=True)
class LoginFailure:
    user_id: int
    timestamp: int  # seconds since the epoch


@dataclass(frozen=True, slots=True)
class Alert:
    user_id: int
    window_start: int
    window_end: int
    failures: int


def detect_bursts(
    failures: Iterable[LoginFailure], window_seconds: int, max_failures: int
) -> list[Alert]:
    """Flag users with more than max_failures failures less than window_seconds apart.

    Each user is handled with a sliding window over sorted timestamps. Overall cost
    is O(n log n) for the sorting plus O(n) for the windows.
    """
    if window_seconds < 1:
        raise ValueError("window_seconds must be at least 1")
    if max_failures < 1:
        raise ValueError("max_failures must be at least 1")

    by_user: defaultdict[int, list[int]] = defaultdict(list)
    for failure in failures:
        by_user[failure.user_id].append(failure.timestamp)

    alerts: list[Alert] = []
    for user_id, timestamps in by_user.items():
        timestamps.sort()
        left = 0
        for right, current in enumerate(timestamps):
            while current - timestamps[left] >= window_seconds:
                left += 1  # drop failures that fell out of the window
            count = right - left + 1
            if count > max_failures:
                alerts.append(Alert(user_id, timestamps[left], current, count))
                break  # one alert per user is enough
    alerts.sort(key=lambda alert: alert.user_id)
    return alerts


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_alerts(connection: sqlite3.Connection, alerts: Sequence[Alert]) -> int:
    """Store alerts in one transaction. Returns how many were new."""
    rows = [(a.user_id, a.window_start, a.window_end, a.failures) for a in alerts]
    try:
        with connection:
            before = connection.total_changes
            connection.executemany(INSERT_SQL, rows)
            inserted = connection.total_changes - before
    except sqlite3.Error:
        logger.exception("could not save %d alerts", len(rows))
        raise
    logger.info("saved %d new alerts out of %d", inserted, len(rows))
    return inserted
```

```python
# test_burst_detector.py
import random
import sqlite3
from collections.abc import Iterator

import pytest

from burst_detector import (
    Alert,
    LoginFailure,
    create_schema,
    detect_bursts,
    save_alerts,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def failures(user_id: int, *timestamps: int) -> list[LoginFailure]:
    return [LoginFailure(user_id, stamp) for stamp in timestamps]


def test_burst_inside_the_window_is_flagged() -> None:
    events = failures(1, 100, 110, 120, 130, 140)
    assert detect_bursts(events, window_seconds=60, max_failures=4) == [
        Alert(1, 100, 140, 5)
    ]


def test_slow_failures_are_not_flagged() -> None:
    events = failures(1, 0, 100, 200, 300, 400, 500)
    assert detect_bursts(events, window_seconds=60, max_failures=2) == []


def test_failures_exactly_window_seconds_apart_are_not_together() -> None:
    events = failures(1, 0, 60)
    assert detect_bursts(events, window_seconds=60, max_failures=1) == []


def test_unsorted_input_and_multiple_users() -> None:
    events = failures(2, 30, 10, 20) + failures(1, 5, 1000)
    alerts = detect_bursts(events, window_seconds=60, max_failures=2)
    assert alerts == [Alert(2, 10, 30, 3)]


@pytest.mark.parametrize(("window", "limit"), [(0, 3), (60, 0)])
def test_invalid_settings_are_rejected(window: int, limit: int) -> None:
    with pytest.raises(ValueError):
        detect_bursts([], window_seconds=window, max_failures=limit)


def test_random_inputs_match_a_brute_force_check() -> None:
    rng = random.Random(11)
    for _ in range(300):
        stamps = [rng.randint(0, 100) for _ in range(rng.randint(0, 15))]
        window, limit = rng.randint(1, 30), rng.randint(1, 4)
        flagged = bool(detect_bursts(failures(1, *stamps), window, limit))
        ordered = sorted(stamps)
        expected = any(
            sum(1 for s in ordered if start <= s < start + window) > limit
            for start in ordered
        )
        assert flagged == expected


def test_saving_the_same_alerts_twice_is_idempotent(
    connection: sqlite3.Connection,
) -> None:
    alerts = [Alert(1, 100, 140, 5), Alert(2, 10, 30, 3)]
    assert save_alerts(connection, alerts) == 2
    assert save_alerts(connection, alerts) == 0
```

In a live system the same logic runs over a stream: keep a deque of recent timestamps per user, drop old ones from the front as each event arrives, and compare the deque length with the limit. That streaming form is the sliding-window rate limiter used in production.

## Common mistakes

- Using `if` instead of `while` for the shrink step. One new element may force several removals.
- Recording the result before the window is valid again.
- Forgetting to delete a key from the counts dictionary when its count reaches zero, so `len(counts)` overstates the number of distinct values.
- Using a sliding window when values can be negative and the rule is about sums.
- Recomputing the window summary with `sum(...)` or `set(...)` on a slice at every step, which makes the algorithm quadratic.
- Off-by-one errors in the window length. The length of `[left, right]` is `right - left + 1`.
- Moving `left` past `right + 1`, or comparing against stale values after shrinking.
- In "longest substring without repeats", moving `left` backwards. Only jump forward if the earlier occurrence is inside the current window, which is why the code checks `last_seen[char] >= start`.
- Forgetting the edge cases: `k` larger than the list, `k = 0`, an empty text, or a pattern longer than the text.
- Trying to count "exactly `k`" directly with one window instead of using "at most `k`" minus "at most `k - 1`".

## Practice exercises

1. Write `average_of_windows(values, k)` that returns the average of every window of length `k`, using a running sum.
2. Write `max_vowels_in_window(text, k)` that returns the largest number of vowels in any substring of length `k`.
3. Write `smallest_subarray_with_sum_at_least(values, target)` for non-negative numbers, returning the shortest length or `0` if none exists.
4. Write `longest_repeating_replacement(text, k)` that returns the length of the longest substring that can be made of one repeated character after at most `k` replacements.
5. Write `find_all_anagrams(text, pattern)` that returns the start indexes of every substring of `text` that is an anagram of `pattern`, using a fixed window and a count table.
6. Write `permutation_in_string(pattern, text)` that tells whether any permutation of `pattern` appears in `text`.
7. Write `max_consecutive_ones_with_flips(bits, k)` that returns the longest run of ones after flipping at most `k` zeros.
8. Write `fruit_baskets(trees)` for a row of trees with a fruit type each, picking the longest run containing at most two types.
9. Write `count_subarrays_with_product_below(values, limit)` for positive integers, using a window and a running product.
10. Write `min_window(text, pattern)` from memory and test it against a brute-force check that tries every substring, on random short strings.
11. Explain with an example why `longest_subarray_at_most` gives wrong answers if negative numbers are allowed, then solve that version with prefix sums.
12. Write `longest_subarray_with_at_most_k_zeros(bits, k)` and explain what the shrink condition is in one sentence.
13. Write `moving_median` for a window size you choose using two heaps or a sorted list, and compare its cost with recomputing the median from a slice.
14. Extend `detect_bursts` so that it can return every violating window for a user, not just the first, and add tests that show how overlapping windows are reported.

## Summary

The sliding window technique keeps a range over contiguous data and updates a summary incrementally as elements enter and leave, so each element is processed once on the way in and once on the way out. Fixed windows add the new element and drop the old one. Variable windows expand on the right and shrink from the left with a `while` loop until a rule holds again, giving `O(n)` total work even though loops are nested, because both pointers only move forward. Hash maps of counts handle distinct-value and character problems, and a missing-requirements counter makes minimum window substring simple. The technique needs a rule that behaves monotonically as the window grows or shrinks. When values can be negative, use prefix sums with a hash map instead.
