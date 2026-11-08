## Overview

Algorithm bugs are quiet. A binary search that is off by one still returns the right answer on most inputs. A sliding window that forgets the empty list works for every example in the textbook. Nothing crashes, and the wrong answer looks plausible. The only defense is to test deliberately: pick inputs that expose mistakes, compare against a version that is obviously correct, and let the computer generate thousands of inputs you would never think to write.

This note teaches the testing tools used throughout the roadmap. You will use `pytest` to write and run tests, an edge-case checklist to choose inputs, a brute-force oracle to check fast solutions, randomized stress tests, property-based testing with Hypothesis, loop invariants and `assert` to reason about correctness, debugging by tracing, and a small benchmark harness.

Tests do not prove an algorithm correct. They find bugs. Reasoning (invariants, proofs) shows why the algorithm works. Strong engineers use both.

## Core concepts

### Tests as small programs

A test is a function that runs your code on a chosen input and uses `assert` to state what must be true. `pytest` finds every function named `test_*` in files named `test_*.py`, runs them, and reports failures with the values involved.

### The edge-case checklist

Run through this list for every function you write.

- Empty input: `[]`, `""`, `{}`.
- A single element.
- Two elements, the smallest case where order and pairs matter.
- All elements equal.
- Duplicates mixed with distinct values.
- Negative numbers and zero.
- Already sorted input and reverse-sorted input.
- The smallest and largest values the problem allows.
- Very large input, to look for slowness and recursion depth problems.
- The target or answer at the very first position, the very last position, and missing entirely.
- Boundary values: for a range check, test the value just below, exactly at, and just above each boundary.

### Brute-force oracle testing

An oracle is a trusted answer source. For most algorithm problems you can write a slow, simple version that is obviously correct: try every subarray, every pair, every ordering. Run the fast version and the slow version on the same input and assert that they agree. Where they disagree, one of them is wrong, and the input that exposes the difference is a ready-made bug report.

### Randomized stress tests

A stress test runs the fast and slow versions on many randomly generated small inputs. Small sizes and small value ranges are important, because they make duplicates, empty inputs, and ties likely. Use a fixed random seed so a failure can be reproduced.

### Property-based testing

Instead of writing examples, you describe properties that must hold for every valid input, and a library generates inputs for you. When a test fails, the library shrinks the input to the smallest example that still fails. Hypothesis is the standard Python library for this. Typical properties:

- Agreement with an oracle: the fast and slow versions return the same answer.
- Structural properties: a sorted result is in order and contains the same elements as the input.
- Round trips: decoding what you encoded returns the original.
- Idempotence: applying the operation twice gives the same result as applying it once.

### Loop invariants and assert

A loop invariant is a statement that is true before the loop starts and stays true after every iteration. When the loop ends, the invariant plus the stop condition proves the result. Writing the invariant as a comment, and sometimes as an `assert`, forces you to understand what the loop is really maintaining.

### Debugging by tracing

Tracing means printing or inspecting the state at each step. `print` with f-strings is the quick tool, `breakpoint()` opens the interactive debugger, and `logging` is the permanent tool for production code. The point is to compare what you expect with what actually happens at each step and find the first place they differ.

### A benchmark harness

A benchmark harness runs several implementations at several input sizes and prints their times, so you can see both the winner and the growth rate.

## Basic syntax examples

### Setting up

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install pytest hypothesis
pytest -q
```

On Windows PowerShell the commands are:

```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install pytest hypothesis
pytest -q
```

### A first test file

```python
# second_largest.py
def second_largest(values: list[int]) -> int | None:
    """Return the second largest distinct value, or None if there is none."""
    distinct = sorted(set(values), reverse=True)
    if len(distinct) < 2:
        return None
    return distinct[1]
```

```python
# test_second_largest.py
import pytest

from second_largest import second_largest


@pytest.mark.parametrize(
    ("values", "expected"),
    [
        ([], None),
        ([5], None),
        ([5, 5, 5], None),
        ([1, 2], 1),
        ([2, 1], 1),
        ([3, 3, 2], 2),
        ([-1, -2, -3], -2),
    ],
)
def test_second_largest(values: list[int], expected: int | None) -> None:
    assert second_largest(values) == expected


def test_does_not_modify_its_input() -> None:
    values = [3, 1, 2]
    snapshot = list(values)
    second_largest(values)
    assert values == snapshot
```

`@pytest.mark.parametrize` runs the same test body once per row, and each row is reported separately. To check that a function raises an error, use `pytest.raises`:

```python
import pytest


def first_item(values: list[int]) -> int:
    if not values:
        raise ValueError("values must not be empty")
    return values[0]


def test_first_item_rejects_empty_list() -> None:
    with pytest.raises(ValueError, match="must not be empty"):
        first_item([])
```

### Useful pytest commands

```bash
pytest -q                    # quiet summary
pytest -x                    # stop at the first failure
pytest -k second_largest     # run only tests whose name matches
pytest -s                    # show print output
pytest --pdb                 # drop into the debugger at a failure
```

## Deeper behavior

### An oracle and a stress test

Kadane's algorithm finds the best sum of a contiguous subarray in one pass. It is short and easy to get subtly wrong, which makes it a perfect candidate for oracle testing.

```python
import random


def max_subarray_fast(values: list[int]) -> int:
    """Kadane's algorithm: O(n) time, O(1) space. Requires a non-empty list."""
    best = current = values[0]
    for value in values[1:]:
        current = max(value, current + value)
        best = max(best, current)
    return best


def max_subarray_slow(values: list[int]) -> int:
    """Oracle: try every subarray, O(n^2). Obviously correct."""
    best = values[0]
    for start in range(len(values)):
        running = 0
        for end in range(start, len(values)):
            running += values[end]
            best = max(best, running)
    return best


def test_matches_oracle_on_random_inputs() -> None:
    rng = random.Random(12345)  # fixed seed: failures are reproducible
    for _ in range(2_000):
        size = rng.randint(1, 12)
        values = [rng.randint(-5, 5) for _ in range(size)]
        expected = max_subarray_slow(values)
        assert max_subarray_fast(values) == expected, values
```

Two design choices matter here. The values are small (`-5` to `5`) so negative-only lists and ties appear constantly. And the failing message includes `values`, so a failure tells you the exact input.

### Hypothesis

Hypothesis replaces the hand-written random loop. You describe the inputs with strategies, and Hypothesis generates them, runs your test, and shrinks any failure.

```python
# Same file as the stress test above: max_subarray_fast and max_subarray_slow
# are defined there.
from collections import Counter
from itertools import pairwise

from hypothesis import example, given, settings
from hypothesis import strategies as st

small_ints = st.integers(min_value=-5, max_value=5)


@given(st.lists(small_ints, min_size=1, max_size=12))
@example([-1])  # pin a case you never want to regress
def test_kadane_matches_oracle(values: list[int]) -> None:
    assert max_subarray_fast(values) == max_subarray_slow(values)


@given(st.lists(st.integers()))
@settings(max_examples=300)
def test_sorted_is_an_ordered_permutation(values: list[int]) -> None:
    result = sorted(values)
    assert all(left <= right for left, right in pairwise(result))
    assert Counter(result) == Counter(values)
```

Key facts about Hypothesis:

- Strategies are building blocks: `st.integers()`, `st.lists(...)`, `st.text()`, `st.booleans()`, `st.tuples(...)`, `st.sampled_from(...)`. You can bound them with arguments such as `min_value` and `max_size`.
- When a property fails, Hypothesis searches for a smaller failing input and reports that. A failing list of 40 numbers usually shrinks to two or three.
- `@example(...)` always runs a specific input in addition to the generated ones.
- Hypothesis stores failures in a `.hypothesis/` directory so that failing cases are retried first. Add that directory to `.gitignore`.
- `assume(condition)` discards inputs that do not meet a precondition, but use it sparingly. If most inputs are discarded, build a better strategy instead.

### Loop invariants in code

Binary search with its invariant written down:

```python
def binary_search(values: list[int], target: int) -> int:
    """Return an index of target in the sorted list `values`, or -1."""
    low, high = 0, len(values) - 1
    while low <= high:
        # Invariant: if target is anywhere in values, it is at an index
        # between low and high, inclusive.
        assert target not in values[:low]
        assert target not in values[high + 1 :]
        middle = (low + high) // 2
        if values[middle] == target:
            return middle
        if values[middle] < target:
            low = middle + 1
        else:
            high = middle - 1
    return -1
```

Reading the loop through its invariant:

- Initialization: before the first iteration `low = 0` and `high = len(values) - 1`, so the range covers the whole list and the invariant is trivially true.
- Maintenance: if `values[middle] < target`, everything up to `middle` is too small, so moving `low` to `middle + 1` keeps the invariant. The other branch is symmetric.
- Termination: each iteration makes `high - low` strictly smaller. When the loop ends, `low > high`, the range is empty, and the invariant says the target is nowhere in the list, so returning `-1` is correct.

The asserts above are `O(n)` each, so they make the search linear. They are a debugging aid, and you would remove them or run Python with `-O` (which strips every `assert`) once you trust the code.

### assert is not input validation

`assert` statements disappear when Python runs with `-O`. Use `assert` to state internal facts that must be true if your code is correct. Use `if` plus `raise ValueError(...)` to validate input from callers, files, or network requests. Mixing them up produces code that silently accepts bad data in optimized mode.

### Tracing with f-strings, breakpoint, and logging

```python
def find_peak(values: list[int]) -> int:
    """Return the index of a peak. Requires a non-empty list."""
    low, high = 0, len(values) - 1
    while low < high:
        middle = (low + high) // 2
        print(f"{low=} {high=} {middle=}")  # temporary trace
        if values[middle] < values[middle + 1]:
            low = middle + 1
        else:
            high = middle
    return low
```

The `{name=}` form prints the name and the value, which saves typing. Common debugger commands after `breakpoint()` or `pytest --pdb`: `n` runs the next line, `s` steps into a call, `c` continues, `p expression` prints a value, `l` lists the surrounding code, and `q` quits. Delete temporary `print` statements when you are done. If a message is worth keeping, convert it to a `logging.debug(...)` call.

### Floating-point comparisons and order

Never compare floats with `==` in a test. Use `pytest.approx`. Never assert on the iteration order of a `set`, because it is not defined. Sort the result or compare sets.

### A small benchmark harness

```python
import timeit
from collections.abc import Callable, Iterable


def benchmark(
    candidates: dict[str, Callable[[list[int]], object]],
    sizes: Iterable[int],
    repeat: int = 5,
) -> None:
    """Print the best time of each candidate at each input size."""
    for size in sizes:
        data = list(range(size))
        for name, func in candidates.items():
            best = min(timeit.repeat(lambda: func(data), number=1, repeat=repeat))
            print(f"{name:<12} n={size:>8}  best={best * 1000:9.3f} ms")


if __name__ == "__main__":
    benchmark({"sum": sum, "sorted": sorted}, sizes=[10_000, 100_000, 1_000_000])
```

Benchmarks should be run on a quiet machine, should use the best of several runs rather than the average, and should always check that the compared functions return the same answer first.

## Industry-level snippet

An audit-log service stores event timestamps in an append-only, sorted list. An endpoint such as `GET /events/count?start=...&end=...` returns how many events fall in a half-open time range, meaning `start` is included and `end` is excluded. Two binary searches answer the query in `O(log n)` instead of scanning the whole log.

The test file shows the full toolkit: parametrized edge cases, a seeded stress test, and Hypothesis properties. It uses a different oracle (a plain filter) and a structural property (adjacent ranges add up) that no example list would think of.

```python
# event_range.py
import logging
from bisect import bisect_left
from collections.abc import Sequence

logger = logging.getLogger(__name__)


class InvalidRangeError(ValueError):
    """Raised when the start of a range is after its end."""


def count_events_in_range(timestamps: Sequence[int], start: int, end: int) -> int:
    """Count timestamps t with start <= t < end.

    Precondition: `timestamps` is sorted in ascending order, as an append-only
    audit log is. Two binary searches make this O(log n) instead of an O(n) scan.
    """
    if start > end:
        raise InvalidRangeError(f"start {start} is after end {end}")
    first = bisect_left(timestamps, start)
    last = bisect_left(timestamps, end)
    logger.debug("range [%s, %s) covers positions %s to %s", start, end, first, last)
    return last - first
```

```python
# test_event_range.py
import random

import pytest
from hypothesis import given
from hypothesis import strategies as st

from event_range import InvalidRangeError, count_events_in_range


def slow_count(timestamps: list[int], start: int, end: int) -> int:
    """Oracle: an obviously correct O(n) filter."""
    return sum(1 for timestamp in timestamps if start <= timestamp < end)


@pytest.mark.parametrize(
    ("timestamps", "start", "end", "expected"),
    [
        ([], 0, 10, 0),
        ([5], 0, 10, 1),
        ([5], 6, 10, 0),
        ([5], 5, 5, 0),
        ([5], 5, 6, 1),
        ([5], 4, 5, 0),
        ([1, 1, 1, 2], 1, 2, 3),
        ([1, 2, 3, 4], -100, 100, 4),
    ],
)
def test_edge_cases(
    timestamps: list[int], start: int, end: int, expected: int
) -> None:
    assert count_events_in_range(timestamps, start, end) == expected


def test_start_after_end_is_rejected() -> None:
    with pytest.raises(InvalidRangeError):
        count_events_in_range([1, 2, 3], 5, 1)


def test_random_stress_against_oracle() -> None:
    rng = random.Random(2024)
    for _ in range(2_000):
        timestamps = sorted(rng.randint(0, 20) for _ in range(rng.randint(0, 15)))
        start = rng.randint(-2, 22)
        end = rng.randint(start, 22)
        expected = slow_count(timestamps, start, end)
        assert count_events_in_range(timestamps, start, end) == expected, (
            timestamps,
            start,
            end,
        )


numbers = st.integers(min_value=-50, max_value=50)


@given(st.lists(numbers), numbers, numbers)
def test_matches_oracle(raw: list[int], first: int, second: int) -> None:
    timestamps = sorted(raw)
    start, end = min(first, second), max(first, second)
    expected = slow_count(timestamps, start, end)
    assert count_events_in_range(timestamps, start, end) == expected


@given(st.lists(numbers), numbers, numbers, numbers)
def test_adjacent_ranges_add_up(raw: list[int], a: int, b: int, c: int) -> None:
    timestamps = sorted(raw)
    start, middle, end = sorted((a, b, c))
    whole = count_events_in_range(timestamps, start, end)
    left = count_events_in_range(timestamps, start, middle)
    right = count_events_in_range(timestamps, middle, end)
    assert whole == left + right
```

The last test is a property that no hand-written example list would naturally contain: splitting a range at any point must never change the total. It catches off-by-one bugs at the boundaries that simple examples can miss.

## Common mistakes

- Testing only the happy path. Bugs live at empty inputs, boundaries, and duplicates.
- Using the same flawed logic in the oracle and in the implementation. An oracle must be simple enough to be obviously correct, not a copy of the fast code.
- Random tests without a seed. A failure you cannot reproduce is nearly useless.
- Random inputs that are too large or too varied. Small sizes and small value ranges hit edge cases far more often.
- Using `assert` to validate user input. It vanishes under `python -O`.
- Comparing floats with `==`. Use `pytest.approx` or integers.
- Asserting on the order of a `set` or on an arbitrary dict iteration assumption.
- Forgetting to test that the function does not mutate its argument when it should not.
- Writing huge tests that take minutes. Slow tests are skipped, and skipped tests protect nothing.
- Fixing a failing test by editing the test until it passes, without understanding why it failed.
- Leaving `print` debugging statements in committed code.

## Practice exercises

1. Write an edge-case list for each function: `remove_duplicates(values)`, `merge_sorted(first, second)`, `is_palindrome(text)`, and `kth_largest(values, k)`. Include at least eight cases per function.
2. Write `parametrize` tests for `second_largest` that cover every item in the edge-case checklist, then deliberately break the function and see which tests fail.
3. Write a brute-force `two_sum_slow` and a fast `two_sum`. Write a seeded stress test that compares them on 5,000 random small lists, remembering that several valid answers may exist, so compare the sums at the returned indices instead of the indices themselves.
4. Write a Hypothesis test asserting that `sorted(sorted(values)) == sorted(values)`. Name the property type that this represents.
5. Write Hypothesis properties for a function `encode(text)` and `decode(encoded)` of your choice (for example run-length encoding). State the round-trip property first.
6. Add the two invariant `assert` statements from `binary_search` to your own version of the function. Run it on 1,000 random sorted lists and confirm that no assertion fires. Then break the update `low = middle + 1` into `low = middle` and observe what happens.
7. Introduce an off-by-one bug into `count_events_in_range` by replacing `bisect_left(timestamps, end)` with `bisect_right(timestamps, end)`. Which of the tests in this note catch it, and which do not?
8. Write a test that checks `max_subarray_fast` does not modify its input. Then write a Hypothesis test that adds a constant to every element and checks how the answer changes for a one-element list.
9. Use `pytest --pdb` on a deliberately failing test and practice the commands `n`, `s`, `p`, `l`, and `c`. Write down what each did.
10. Extend `benchmark` so that it first checks that all candidates return equal results on the data and raises a clear error if they do not.
11. Write a stress test for `binary_search` against `list.index` (wrapped to return `-1`), including lists with duplicates. Explain why the two may legitimately return different indices.
12. Pick any solution you wrote for the practice exercises of the previous notes and add a complete test file with edge cases, an oracle, and one Hypothesis property.

## Summary

Algorithm bugs hide in edge cases, so testing must be deliberate. `pytest` makes tests small and readable, `parametrize` multiplies examples, and `pytest.raises` checks errors. The edge-case checklist guides input choice, a brute-force oracle supplies trusted answers, seeded random stress tests widen coverage, and Hypothesis explores inputs you would not think of and shrinks failures. Loop invariants explain why a loop is correct, and `assert` documents internal facts but never validates input. Trace with prints, `breakpoint()`, or logging, and compare candidates with a small benchmark harness. Testing finds bugs and invariants explain correctness, and reliable code uses both.
