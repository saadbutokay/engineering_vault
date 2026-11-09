## Overview

An interval is a stretch of a line with a start and an end: a meeting from 9:00 to 10:00, a server that was down from 02:14 to 02:31, a hotel room booked from the 3rd to the 7th, a software license valid for one year. Questions about intervals are everywhere in real systems. Do two bookings clash? How many rooms do I need? When is everyone free? Which maintenance windows overlap? What is the total time covered by all these jobs?

Almost every interval problem becomes easy after one step: sort the intervals by their start time. Once they are in order, overlaps can only occur between neighbors, and a single left-to-right pass finds them. A second powerful technique is the sweep line: turn each interval into two events (a start and an end), sort all the events by time, and sweep across them while keeping a running count of how many intervals are currently active.

This note covers merging intervals, inserting an interval, overlap detection, meeting rooms and the minimum number of rooms, event-based sweep line, calendar booking, and interval scheduling. The most important habit is to decide, before writing any code, whether the endpoints are inclusive or exclusive, because that decision changes every comparison.

## Core concepts

### Closed and half-open intervals

- A closed interval `[start, end]` includes both endpoints. `[1, 3]` and `[3, 5]` share the point 3.
- A half-open interval `[start, end)` includes the start and excludes the end. `[1, 3)` and `[3, 5)` do not overlap. They touch.

For time bookings the half-open convention is almost always right: a meeting from 9:00 to 10:00 and another from 10:00 to 11:00 can both use the same room. This note uses half-open intervals unless it says otherwise, so two intervals overlap exactly when each one starts before the other ends: `a.start < b.end and b.start < a.end`.

### Sorting by start

Once intervals are sorted by start time, an interval can overlap only with those that follow it before the next gap. You never need to compare an interval with all the others, so the pass after the sort is `O(n)` and the total is `O(n log n)`.

### Merging intervals

To merge overlapping intervals, sort by start, then walk through them while keeping the last merged interval. If the next interval starts at or before the end of the last one, extend the last one's end to the larger of the two ends. Otherwise start a new merged interval. The result is a sorted list of disjoint intervals that covers exactly the same points.

### Inserting an interval

Given a sorted list of disjoint intervals and a new interval, the result has three parts: the intervals that end before the new one starts, the merged group that overlaps (or touches) it, and the intervals that start after it ends. It needs one pass, `O(n)`, and no sorting.

### Overlap detection

Sort by start and check each interval against the previous one. If an interval starts before the previous one ends, there is a conflict. If no neighbor pair conflicts, no pair conflicts at all.

### Meeting rooms and the minimum number of rooms

The minimum number of rooms equals the maximum number of meetings happening at the same moment, the peak concurrency. Two standard ways to find it:

- Sweep line: create `+1` events at starts and `-1` events at ends, sort them, and track the running total.
- Min-heap of end times: process meetings by start time and keep the earliest finishing room at the top. If the new meeting starts after that room frees up, reuse it; otherwise open another room.

### Sweep line

A sweep line imagines a vertical line moving left to right across time. Events change the state when the line passes them: an interval starts or ends. By sorting events and processing them in order, you maintain a running state such as "how many intervals are active", and the answer to many questions falls out: peak load, total covered time, or the moments when a resource is overused.

### Calendar booking

A calendar must accept a new booking only if it does not overlap any existing one. If the bookings are kept sorted by start time, a new interval can only conflict with its immediate neighbors: the booking just before it and the booking just after it. Binary search finds that position in `O(log n)`.

### Interval scheduling

Interval scheduling asks for the largest set of non-overlapping intervals (for example the maximum number of meetings one person can attend). The greedy rule is to always take the interval that ends earliest among those compatible with what you have already taken. Earliest finish leaves the most room for the rest.

## Basic syntax examples

### Merging intervals

```python
from collections.abc import Iterable

type Interval = tuple[int, int]


def merge_intervals(intervals: Iterable[Interval]) -> list[Interval]:
    """Merge overlapping or touching intervals. O(n log n) for the sort."""
    merged: list[Interval] = []
    for start, end in sorted(intervals):
        if merged and start <= merged[-1][1]:  # overlaps or touches the last one
            last_start, last_end = merged[-1]
            merged[-1] = (last_start, max(last_end, end))  # max handles nesting
        else:
            merged.append((start, end))
    return merged


print(merge_intervals([(8, 10), (1, 3), (2, 6), (15, 18), (17, 20)]))
# [(1, 6), (8, 10), (15, 20)]
print(merge_intervals([(1, 10), (2, 4), (5, 6)]))  # [(1, 10)], the others are nested
```

Trace of the first example after sorting to `[(1, 3), (2, 6), (8, 10), (15, 18), (17, 20)]`:

```text
(1, 3):   merged is empty, start a new one            -> [(1, 3)]
(2, 6):   2 <= 3 overlaps, extend to max(3, 6) = 6    -> [(1, 6)]
(8, 10):  8 > 6 no overlap, start a new one           -> [(1, 6), (8, 10)]
(15, 18): 15 > 10, start a new one                    -> [..., (15, 18)]
(17, 20): 17 <= 18 overlaps, extend to 20             -> [..., (15, 20)]
```

### Inserting an interval

```python
def insert_interval(intervals: list[Interval], new: Interval) -> list[Interval]:
    """Insert into a sorted list of disjoint intervals. O(n), no sorting needed."""
    result: list[Interval] = []
    start, end = new
    i = 0
    while i < len(intervals) and intervals[i][1] < start:  # entirely before new
        result.append(intervals[i])
        i += 1
    while i < len(intervals) and intervals[i][0] <= end:  # overlaps or touches new
        start = min(start, intervals[i][0])
        end = max(end, intervals[i][1])
        i += 1
    result.append((start, end))
    result.extend(intervals[i:])  # entirely after new
    return result


print(insert_interval([(1, 3), (6, 9)], (2, 5)))  # [(1, 5), (6, 9)]
print(insert_interval([(1, 2), (3, 5), (6, 7), (8, 10)], (4, 8)))  # [(1, 2), (3, 10)]
```

### Detecting conflicts

```python
def has_conflict(intervals: Iterable[Interval]) -> bool:
    """True if any two half-open intervals overlap."""
    ordered = sorted(intervals)
    for (_, previous_end), (start, _) in zip(ordered, ordered[1:]):
        if start < previous_end:
            return True
    return False


print(has_conflict([(0, 30), (5, 10), (15, 20)]))  # True
print(has_conflict([(7, 10), (2, 4)]))  # False
print(has_conflict([(1, 3), (3, 5)]))  # False, touching is allowed
```

### Minimum number of rooms with a sweep line

```python
def min_rooms(meetings: Iterable[Interval]) -> int:
    """Peak number of simultaneous meetings. O(n log n)."""
    events: list[tuple[int, int]] = []
    for start, end in meetings:
        events.append((start, 1))  # a meeting starts: one more room in use
        events.append((end, -1))  # a meeting ends: one room becomes free
    events.sort()  # at equal times -1 sorts first, so back-to-back meetings share
    in_use = peak = 0
    for _, change in events:
        in_use += change
        peak = max(peak, in_use)
    return peak


print(min_rooms([(0, 30), (5, 10), (15, 20)]))  # 2
print(min_rooms([(7, 10), (2, 4)]))  # 1
print(min_rooms([(1, 3), (3, 5)]))  # 1
```

Trace of the events for `[(0, 30), (5, 10), (15, 20)]`:

```text
time  event  in use
0     +1     1
5     +1     2   <- peak
10    -1     1
15    +1     2
20    -1     1
30    -1     0
```

### Minimum number of rooms with a heap

```python
import heapq


def min_rooms_heap(meetings: Iterable[Interval]) -> int:
    """Process meetings by start time. The heap holds the end time of each busy room."""
    busy_until: list[int] = []
    for start, end in sorted(meetings):
        if busy_until and busy_until[0] <= start:
            heapq.heapreplace(busy_until, end)  # reuse the room that frees up first
        else:
            heapq.heappush(busy_until, end)  # all rooms are busy: open a new one
    return len(busy_until)


print(min_rooms_heap([(0, 30), (5, 10), (15, 20)]))  # 2
```

### Calendar booking with binary search

```python
from bisect import bisect_left


class Calendar:
    """Accepts a booking only if it does not overlap any earlier one."""

    def __init__(self) -> None:
        self._starts: list[int] = []
        self._ends: list[int] = []

    def book(self, start: int, end: int) -> bool:
        if start >= end:
            raise ValueError("start must be before end")
        index = bisect_left(self._starts, start)  # where this booking would go
        if index > 0 and self._ends[index - 1] > start:
            return False  # the previous booking runs into this one
        if index < len(self._starts) and self._starts[index] < end:
            return False  # this booking runs into the next one
        self._starts.insert(index, start)
        self._ends.insert(index, end)
        return True


calendar = Calendar()
print(calendar.book(10, 20))  # True
print(calendar.book(15, 25))  # False, overlaps
print(calendar.book(20, 30))  # True, touching is fine
```

### Interval scheduling: the most meetings one person can attend

```python
def max_non_overlapping(intervals: Iterable[Interval]) -> list[Interval]:
    """Greedy: always take the compatible interval that ends first."""
    chosen: list[Interval] = []
    last_end = float("-inf")
    for start, end in sorted(intervals, key=lambda interval: interval[1]):
        if start >= last_end:
            chosen.append((start, end))
            last_end = end
    return chosen


meetings = [(1, 4), (3, 5), (0, 6), (5, 7), (3, 9), (5, 9), (6, 10), (8, 11)]
print(max_non_overlapping(meetings))  # [(1, 4), (5, 7), (8, 11)]
```

## Deeper behavior

### The sort makes the problem local

After sorting by start, intervals with similar start times are neighbors. If interval `i` does not overlap interval `i + 1` (which starts no earlier), then it cannot overlap any interval after `i + 1` either, because those start even later. That locality is why merging and conflict detection need only one pass. It is also why using `max(last_end, end)` is necessary in merging: an early interval may be long enough to contain several later ones, and the last merged end must keep the largest end seen so far.

### Why both sort orders appear

- Sort by start for merging, inserting, conflict detection, and room counting. The question is "what overlaps what", which is a left-to-right question.
- Sort by end for interval scheduling. The question is "what leaves the most room", which is about getting free as early as possible. Sorting by start would let one very long early interval block everything.

### Tie-breaking in the sweep line

When a meeting ends at the exact time another starts, which event comes first? With half-open intervals, the end must be processed before the start, so back-to-back meetings use the same room. Encoding events as `(time, change)` with `-1` for ends and `+1` for starts gets this for free, because tuples sort by time and then by `change`, and `-1 < 1`. For closed intervals you reverse it: starts first. Getting this wrong produces off-by-one room counts at boundaries, so always test a case where one interval ends exactly when another begins.

### Sweep line beyond counting

The same event-sorting idea answers many questions.

- Total covered length: keep a count of active intervals, and add the gap between consecutive events whenever the count is above zero.
- Moments of overload: report each time range where the active count exceeds a limit.
- Skyline and other geometric problems: events carry heights instead of `+1` and `-1`, and a heap tracks the maximum.

### Complexity

- Sorting: `O(n log n)` for merge, conflict detection, room counting, and scheduling.
- Inserting into a sorted disjoint list: `O(n)`, or `O(log n)` to find the position plus `O(n)` to shift.
- Calendar booking: `O(log n)` to find the neighbors, plus `O(n)` for the list insertion. A balanced tree or `sortedcontainers` brings the insertion cost down as well.
- Room counting with a heap: `O(n log n)` for the sort and `O(log n)` per heap operation.

### Small coordinate ranges and difference arrays

When all times are small integers, for example minutes in a day, a difference array does the same job as the sweep line: add `+1` at each start and `-1` at each end, then take a running total. It costs `O(n + range)` and needs no sorting. A sweep line is the better choice when coordinates are large or sparse, such as timestamps in milliseconds.

### Interval trees

If you must answer "which stored intervals overlap this one?" repeatedly on a changing set, sorting once is not enough. Interval trees, covered with the advanced data structures, answer such queries in `O(log n + k)`.

### Weighted scheduling

If intervals carry values and you want the maximum total value of non-overlapping ones, the greedy rule fails, because a short low-value interval can block a long high-value one. That needs dynamic programming over intervals sorted by end time.

## Industry-level snippet

A company manages meeting rooms. Employees book a room for a time range in minutes since midnight, and two rules must always hold: no two bookings for the same room may overlap, and a booking that ends at 10:00 must not block one that starts at 10:00. The service also shows free slots for a room and tells facilities the peak number of rooms in use at once.

The interesting part is correctness under concurrency. Two people can try to book the same slot at the same moment. The service therefore checks for a clash and inserts the booking inside one transaction that takes the write lock first (`BEGIN IMMEDIATE`), so the check and the insert cannot be separated by another writer. The overlap test in SQL is exactly the half-open condition from this note: an existing booking clashes if it starts before the new end and ends after the new start. The database also has a `CHECK` constraint as a second line of defense against invalid ranges.

Free slots and peak usage use the in-memory techniques from this note: merge the busy intervals, then collect the gaps; count peak concurrency with a sweep line.

```python
# room_booking.py
import logging
import sqlite3
from collections.abc import Iterable

logger = logging.getLogger(__name__)

type Interval = tuple[int, int]

DAY_MINUTES = 24 * 60

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS booking (
    id INTEGER PRIMARY KEY,
    room TEXT NOT NULL,
    start_minute INTEGER NOT NULL,
    end_minute INTEGER NOT NULL,
    CHECK (start_minute >= 0 AND end_minute <= 1440 AND start_minute < end_minute)
)
"""

CLASH_SQL = (
    "SELECT 1 FROM booking "
    "WHERE room = ? AND start_minute < ? AND end_minute > ? LIMIT 1"
)

INSERT_SQL = "INSERT INTO booking (room, start_minute, end_minute) VALUES (?, ?, ?)"


class InvalidIntervalError(ValueError):
    """Raised for a time range that is empty, reversed, or outside the day."""


class BookingConflictError(Exception):
    """Raised when the requested range overlaps an existing booking."""


def validate_interval(start: int, end: int) -> None:
    if not 0 <= start < end <= DAY_MINUTES:
        raise InvalidIntervalError(
            f"need 0 <= start < end <= {DAY_MINUTES}, got ({start}, {end})"
        )


def merge_intervals(intervals: Iterable[Interval]) -> list[Interval]:
    merged: list[Interval] = []
    for start, end in sorted(intervals):
        if merged and start <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], end))
        else:
            merged.append((start, end))
    return merged


def free_slots(
    busy: Iterable[Interval], day_start: int = 0, day_end: int = DAY_MINUTES
) -> list[Interval]:
    """The gaps between busy intervals inside [day_start, day_end)."""
    slots: list[Interval] = []
    cursor = day_start
    for start, end in merge_intervals(busy):
        if end <= cursor:
            continue  # entirely before the part of the day we still care about
        if start >= day_end:
            break  # entirely after the day
        if start > cursor:
            slots.append((cursor, start))
        cursor = max(cursor, end)
    if cursor < day_end:
        slots.append((cursor, day_end))
    return slots


def peak_concurrency(intervals: Iterable[Interval]) -> int:
    """Largest number of overlapping half-open intervals, with a sweep line."""
    events: list[tuple[int, int]] = []
    for start, end in intervals:
        events.append((start, 1))
        events.append((end, -1))  # sorts before a +1 at the same time
    events.sort()
    active = peak = 0
    for _, change in events:
        active += change
        peak = max(peak, active)
    return peak


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def book_room(connection: sqlite3.Connection, room: str, start: int, end: int) -> int:
    """Book a room atomically. Returns the booking id or raises on a clash."""
    if not room:
        raise ValueError("room must not be empty")
    validate_interval(start, end)
    try:
        with connection:
            connection.execute("BEGIN IMMEDIATE")  # no other writer can interleave
            clash = connection.execute(CLASH_SQL, (room, end, start)).fetchone()
            if clash is not None:
                raise BookingConflictError(
                    f"{room} is already booked between {start} and {end}"
                )
            cursor = connection.execute(INSERT_SQL, (room, start, end))
    except sqlite3.Error:
        logger.exception("could not book %s from %d to %d", room, start, end)
        raise
    if cursor.lastrowid is None:
        raise RuntimeError("database did not return a row id")
    logger.info("booked %s from %d to %d as #%d", room, start, end, cursor.lastrowid)
    return cursor.lastrowid


def busy_intervals(connection: sqlite3.Connection, room: str) -> list[Interval]:
    rows = connection.execute(
        "SELECT start_minute, end_minute FROM booking WHERE room = ? "
        "ORDER BY start_minute",
        (room,),
    )
    return [(row[0], row[1]) for row in rows]


def free_slots_for_room(
    connection: sqlite3.Connection,
    room: str,
    day_start: int = 0,
    day_end: int = DAY_MINUTES,
) -> list[Interval]:
    return free_slots(busy_intervals(connection, room), day_start, day_end)
```

```python
# test_room_booking.py
import random
import sqlite3
from collections.abc import Iterator

import pytest

from room_booking import (
    BookingConflictError,
    InvalidIntervalError,
    book_room,
    busy_intervals,
    create_schema,
    free_slots,
    free_slots_for_room,
    merge_intervals,
    peak_concurrency,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def test_overlapping_booking_is_rejected(connection: sqlite3.Connection) -> None:
    book_room(connection, "A", 540, 600)
    with pytest.raises(BookingConflictError):
        book_room(connection, "A", 570, 630)
    with pytest.raises(BookingConflictError):
        book_room(connection, "A", 500, 541)
    with pytest.raises(BookingConflictError):
        book_room(connection, "A", 550, 560)  # fully inside the existing booking
    assert busy_intervals(connection, "A") == [(540, 600)]


def test_back_to_back_bookings_are_allowed(connection: sqlite3.Connection) -> None:
    book_room(connection, "A", 540, 600)
    book_room(connection, "A", 600, 660)
    book_room(connection, "A", 480, 540)
    assert busy_intervals(connection, "A") == [(480, 540), (540, 600), (600, 660)]


def test_rooms_are_independent(connection: sqlite3.Connection) -> None:
    book_room(connection, "A", 540, 600)
    book_room(connection, "B", 540, 600)
    assert busy_intervals(connection, "B") == [(540, 600)]


@pytest.mark.parametrize(
    ("start", "end"), [(10, 10), (20, 10), (-1, 5), (0, 1441)]
)
def test_invalid_intervals_are_rejected(
    connection: sqlite3.Connection, start: int, end: int
) -> None:
    with pytest.raises(InvalidIntervalError):
        book_room(connection, "A", start, end)


def test_empty_room_name_is_rejected(connection: sqlite3.Connection) -> None:
    with pytest.raises(ValueError):
        book_room(connection, "", 0, 10)


def test_free_slots_for_a_room(connection: sqlite3.Connection) -> None:
    book_room(connection, "A", 540, 600)
    book_room(connection, "A", 720, 780)
    slots = free_slots_for_room(connection, "A", day_start=480, day_end=840)
    assert slots == [(480, 540), (600, 720), (780, 840)]


def test_merge_intervals_basic() -> None:
    assert merge_intervals([(8, 10), (1, 3), (2, 6), (15, 18), (17, 20)]) == [
        (1, 6),
        (8, 10),
        (15, 20),
    ]
    assert merge_intervals([(1, 10), (2, 4), (5, 6)]) == [(1, 10)]
    assert merge_intervals([]) == []


def covered_minutes(intervals: list[tuple[int, int]], size: int) -> list[int]:
    """Oracle: how many intervals cover each minute."""
    counts = [0] * size
    for start, end in intervals:
        for minute in range(start, end):
            counts[minute] += 1
    return counts


def random_intervals(rng: random.Random, size: int) -> list[tuple[int, int]]:
    result = []
    for _ in range(rng.randint(0, 8)):
        start = rng.randint(0, size - 1)
        result.append((start, rng.randint(start + 1, size)))
    return result


def test_random_intervals_match_a_minute_by_minute_oracle() -> None:
    rng = random.Random(8)
    size = 30
    for _ in range(400):
        intervals = random_intervals(rng, size)
        counts = covered_minutes(intervals, size)
        assert peak_concurrency(intervals) == max(counts, default=0)
        merged = merge_intervals(intervals)
        merged_counts = covered_minutes(merged, size)
        assert [c > 0 for c in merged_counts] == [c > 0 for c in counts]
        free = free_slots(intervals, 0, size)
        free_counts = covered_minutes(free, size)
        assert [c > 0 for c in free_counts] == [c == 0 for c in counts]


def test_peak_concurrency_for_back_to_back_meetings() -> None:
    assert peak_concurrency([(1, 3), (3, 5)]) == 1
    assert peak_concurrency([(0, 30), (5, 10), (15, 20)]) == 2
    assert peak_concurrency([]) == 0
```

If the `INSERT` happened without the lock-taking `BEGIN IMMEDIATE`, two workers could both pass the clash check and both insert. The transaction turns the check-then-insert sequence into one atomic step.

## Common mistakes

- Not deciding whether endpoints are inclusive or exclusive, so touching intervals are sometimes treated as overlapping and sometimes not.
- Forgetting `max(last_end, end)` when merging, which breaks on nested intervals such as `(1, 10)` followed by `(2, 4)`.
- Merging without sorting first.
- Sorting by end time for merging, or by start time for interval scheduling. Each task has its own order.
- Processing a start event before an end event at the same time with half-open intervals, which overcounts rooms.
- Checking a new booking against all bookings in a loop when sorted neighbors are enough, or checking only one neighbor when both are needed.
- Checking for a clash and then inserting in two separate database transactions, which lets a concurrent request slip in between.
- Using `start <= other_end` in the overlap test for half-open intervals, which flags back-to-back bookings as conflicts.
- Modifying the input list while merging, or relying on the caller to pass sorted data without checking.
- Using a large boolean array indexed by time in milliseconds instead of a sweep line.
- Assuming a greedy rule works for weighted intervals.

## Practice exercises

1. Write `intervals_overlap(a, b)` for half-open intervals and test the cases: disjoint, touching, partial overlap, nested, and identical.
2. Merge `[(1, 4), (4, 5)]` as closed intervals and as half-open intervals. Say whether the answers differ and why you might choose to merge touching intervals anyway.
3. Write `remove_covered_intervals(intervals)` that removes any interval fully contained in another, and returns the number that remain.
4. Write `total_covered_length(intervals)` by merging, then again with a sweep line that adds up the gaps between events with a positive count.
5. Write `interval_intersection(first, second)` for two sorted lists of disjoint intervals, using two pointers, and return their overlaps.
6. Write `min_arrows(balloons)` where each balloon is the horizontal range `[start, end]` and an arrow shot at `x` bursts every balloon covering `x`, using the greedy rule on sorted ends.
7. Write `employee_free_time(schedules)` that takes one sorted list of busy intervals per employee and returns the common free time within `[0, 1440)`.
8. Write `max_overlap_moment(intervals)` that returns a time when the maximum number of intervals are active, not only the count.
9. Write `can_attend_all(meetings)` and `min_rooms(meetings)`, and test them against a brute-force minute-by-minute oracle on random inputs.
10. Write `non_overlapping_to_remove(intervals)` that returns the minimum number of intervals to delete so the rest do not overlap, using the earliest-end greedy rule.
11. Implement `Calendar.book` with `sortedcontainers.SortedList` and compare it with the list-and-`bisect` version on 100,000 bookings.
12. Write `weighted_interval_scheduling(intervals)` with a dynamic programming solution that uses binary search to find the last compatible interval, and show an input where the greedy rule fails.
13. Extend `book_room` with an optional `owner` argument, persist it, and write `cancel_booking(connection, booking_id, owner)` that only the owner can call.
14. Extend `free_slots_for_room` with a `minimum_minutes` argument that drops gaps shorter than the requested length, and add tests.

## Summary

Interval problems become manageable once you fix the endpoint convention (half-open is best for time ranges) and sort by start time, which makes overlaps local to neighbors. Merging walks the sorted list and extends the last interval with `max`, inserting is a three-phase linear scan, and conflict detection compares each interval with the previous one. A sweep line turns intervals into `+1` and `-1` events and tracks the number of active intervals, which gives the minimum number of rooms, peak load, and covered time, as long as end events sort before start events at equal times. A calendar can keep bookings sorted and use binary search to check just the two neighbors. For a maximum set of non-overlapping intervals, sort by end time and take whatever fits. In a real service the clash check and the insert belong in one transaction, so concurrent requests cannot both succeed.
