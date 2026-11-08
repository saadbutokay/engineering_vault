## Overview

Complexity analysis answers one question: when the input gets bigger, how fast does the cost grow? The cost can be time (how many basic steps run) or space (how much extra memory is used). It does not tell you how many seconds a program takes. It tells you what happens to those seconds when the input doubles, grows tenfold, or grows a thousandfold.

This matters because a program that feels instant on 100 test rows can freeze on 10 million production rows. Complexity analysis lets you predict that before you deploy. It also gives engineers a shared vocabulary: saying "that lookup is linear" or "that loop is quadratic" is faster and more precise than describing timings.

This note covers input size, Big-O, Big-Omega, and Big-Theta, best, average, and worst case, the common growth rates, how to analyze loops and recursion, space complexity including the call stack, amortized analysis, when constants matter, and how to measure growth empirically.

## Core concepts

### Input size

The input size, usually written `n`, is the quantity that makes the problem bigger. Choose it carefully.

- For a list or string, `n` is the number of elements or characters.
- For a graph, there are two sizes: `V` vertices and `E` edges.
- For a number, `n` can be its value, but the honest measure is its number of digits, which is about `log n`.
- When there are two independent inputs, keep two variables, such as `n` and `m`. Do not merge them.

### Cost model

Count basic operations: an arithmetic step, a comparison, an assignment, an index access. Treat each as one unit of cost. This is a simplification, but it is accurate enough to compare algorithms by how their cost grows.

### Big-O, Big-Omega, Big-Theta

- Big-O (written `O`) is an upper bound. "The cost grows no faster than this."
- Big-Omega (written `Ω`) is a lower bound. "The cost grows at least this fast."
- Big-Theta (written `Θ`) is a tight bound. "The cost grows exactly at this rate, up to constant factors."

In everyday engineering talk, people say "Big-O" and mean the tight worst-case bound. This roadmap does the same unless it says otherwise.

### Best, average, and worst case

The same algorithm can behave differently on different inputs of the same size.

- Best case: the luckiest input. Linear search finds the target at index 0.
- Worst case: the unluckiest input. Linear search scans everything and finds nothing.
- Average case: the expected cost over typical inputs, which needs an assumption about what "typical" means.

Worst case is the default for safety because it is a guarantee. Average case matters when the worst case is rare, as with hash tables and quicksort.

### Common growth rates

From slowest-growing (best) to fastest-growing (worst):

- Constant, `O(1)`: cost does not depend on `n`. Example: indexing a list.
- Logarithmic, `O(log n)`: each step cuts the problem down by a constant fraction. Example: binary search.
- Linear, `O(n)`: touch every element a fixed number of times. Example: summing a list.
- Linearithmic, `O(n log n)`: a linear amount of work at each of `log n` levels. Example: merge sort, Python's `sorted`.
- Quadratic, `O(n^2)`: work for every pair. Example: comparing every element with every other element.
- Cubic, `O(n^3)`: work for every triple. Example: naive matrix multiplication.
- Exponential, `O(2^n)`: work doubles with each extra element. Example: enumerating all subsets.
- Factorial, `O(n!)`: work for every ordering. Example: trying every permutation.

Approximate step counts when `n = 1,000`:

| Growth | Approximate steps |
| --- | --- |
| Constant | 1 |
| Logarithmic | 10 |
| Linear | 1,000 |
| Linearithmic | 10,000 |
| Quadratic | 1,000,000 |
| Cubic | 1,000,000,000 |
| Exponential | about 10^301 |
| Factorial | about 10^2,567 |

The last two rows are not a typo. Exponential and factorial algorithms are only usable for very small `n`.

### Simplification rules

- Drop constant factors: `O(3n)` is `O(n)`.
- Drop lower-order terms: `O(n^2 + n)` is `O(n^2)`.
- Sequential steps add: a loop of `n` followed by a loop of `n^2` costs `O(n + n^2)`, which is `O(n^2)`.
- Nested steps multiply: a loop of `n` containing a loop of `m` costs `O(n * m)`.
- Different inputs keep different letters: looping over `a` and then over `b` is `O(a + b)`, not `O(n)`.

### Space complexity

Space complexity counts the extra memory an algorithm needs beyond the input itself, called auxiliary space. Space includes the call stack: a recursive function that goes `n` levels deep uses `O(n)` stack space even if it allocates nothing else.

### Amortized analysis

Amortized cost is the average cost per operation over a worst-case sequence of operations. A Python list append is occasionally expensive, when the list must be copied to a bigger block, but that cost is spread across many cheap appends, so each append is `O(1)` amortized.

## Basic syntax examples

One small function for each growth rate. The comments give the complexity.

```python
from itertools import pairwise


def first_item(values: list[int]) -> int:
    """O(1): one index operation, however long the list is."""
    return values[0]


def total(values: list[int]) -> int:
    """O(n): touches every element once."""
    result = 0
    for value in values:
        result += value
    return result


def halvings(n: int) -> int:
    """O(log n): how many times can n be halved before it reaches 1?"""
    steps = 0
    while n > 1:
        n //= 2
        steps += 1
    return steps


def largest_gap(values: list[int]) -> int:
    """O(n log n): the sort dominates the O(n) scan that follows."""
    ordered = sorted(values)
    best = 0
    for previous, current in pairwise(ordered):
        best = max(best, current - previous)
    return best


def has_duplicate_quadratic(values: list[int]) -> bool:
    """O(n^2) time, O(1) space: compares every pair."""
    for i in range(len(values)):
        for j in range(i + 1, len(values)):
            if values[i] == values[j]:
                return True
    return False


def has_duplicate_linear(values: list[int]) -> bool:
    """O(n) average time, O(n) space: trades memory for speed."""
    seen: set[int] = set()
    for value in values:
        if value in seen:
            return True
        seen.add(value)
    return False


def all_subsets(items: list[int]) -> list[list[int]]:
    """O(2^n) subsets, so at least O(2^n) work (copying makes it more)."""
    subsets: list[list[int]] = [[]]
    for item in items:
        subsets += [subset + [item] for subset in subsets]
    return subsets
```

### Counting operations

Counting steps makes growth visible. A function that does nothing but count shows the pattern clearly.

```python
def count_linear_steps(n: int) -> int:
    steps = 0
    for _ in range(n):
        steps += 1
    return steps


def count_quadratic_steps(n: int) -> int:
    steps = 0
    for _ in range(n):
        for _ in range(n):
            steps += 1
    return steps


for size in (10, 100, 1_000):
    print(size, count_linear_steps(size), count_quadratic_steps(size))
```

```text
10 10 100
100 100 10000
1000 1000 1000000
```

When the input grows tenfold, the linear count grows tenfold and the quadratic count grows a hundredfold.

### Reading loops

```python
def triangular_steps(n: int) -> int:
    """The inner loop runs 0, 1, 2, ..., n - 1 times."""
    steps = 0
    for i in range(n):
        for _ in range(i):
            steps += 1
    return steps  # n * (n - 1) / 2, which is O(n^2)


def doubling_steps(n: int) -> int:
    """The loop variable doubles each time, so the loop runs about log2(n) times."""
    steps = 0
    i = 1
    while i < n:
        i *= 2
        steps += 1
    return steps  # O(log n)
```

## Deeper behavior

### The formal definition

```text
f(n) is O(g(n)) if there are constants c > 0 and n0 such that
f(n) <= c * g(n) for every n >= n0.
```

Big-O does not care about small inputs, only about the behavior once `n` is large enough. Example: show that `3n^2 + 5n + 2` is `O(n^2)`.

```text
For n >= 1:  5n <= 5n^2  and  2 <= 2n^2
So  3n^2 + 5n + 2  <=  3n^2 + 5n^2 + 2n^2  =  10 n^2
Choose c = 10 and n0 = 1.  The definition is satisfied.
```

### Big-O is an upper bound, not a promise of tightness

A linear algorithm is technically also `O(n^2)` and `O(n^3)`, because those are also upper bounds. When you want to say "exactly linear", use Big-Theta. Interviews and everyday talk usually mean the tightest Big-O.

### Best, average, and worst case in practice

- Linear search: best `Θ(1)`, worst `Θ(n)`, average about `n / 2` comparisons, which is `Θ(n)`.
- Dictionary lookup: average `O(1)`, worst `O(n)` if many keys collide.
- Quicksort: average `O(n log n)`, worst `O(n^2)` on unlucky pivots.
- Python's `sorted`: worst case `O(n log n)`, best case `O(n)` on already-sorted input.

Notice that best, average, and worst describe the input, while Big-O, Big-Omega, and Big-Theta describe the bound. They are separate ideas and can be combined: "the worst case is `Θ(n)`".

### Logarithms: why the base does not matter

A logarithm counts how many times you can divide by a constant before reaching 1. `log2(1,000,000)` is about 20, so binary search on a million items needs at most about 20 steps. On a billion items it needs about 30. Changing the base only multiplies by a constant (`log2(n) = log10(n) / log10(2)`), so Big-O ignores the base and writes `O(log n)`.

### Analyzing recursion with recurrences

A recurrence writes the cost of a problem of size `n` in terms of smaller problems.

```text
T(n) = T(n - 1) + O(1)      one call, size n - 1       ->  O(n)       (sum of a list recursively)
T(n) = T(n / 2) + O(1)      one call, half the size    ->  O(log n)   (binary search)
T(n) = 2T(n / 2) + O(n)     two halves plus a merge    ->  O(n log n) (merge sort)
T(n) = 2T(n - 1) + O(1)     two calls, size n - 1      ->  O(2^n)     (naive Fibonacci style)
```

The recursion-tree method makes these easy to see. For `T(n) = 2T(n / 2) + n`: level 0 does `n` work, level 1 has two subproblems of size `n / 2` and again does `n` work in total, level 2 does `n` work in total, and so on. There are `log2(n)` levels, each doing `n` work, so the total is `n log n`. The Master theorem, covered with recursion later in the roadmap, turns this reasoning into a formula.

### Space complexity and the call stack

```python
import sys


def sum_recursive(values: list[int], index: int = 0) -> int:
    """Time O(n). Space O(n) because every pending call occupies a stack frame."""
    if index == len(values):
        return 0
    return values[index] + sum_recursive(values, index + 1)


def sum_iterative(values: list[int]) -> int:
    """Time O(n). Space O(1): a single accumulator."""
    result = 0
    for value in values:
        result += value
    return result


print(sys.getrecursionlimit())  # 1000 by default
```

Calling `sum_recursive` on a list of 5,000 numbers raises `RecursionError`, even though the logic is correct. Python deliberately limits recursion depth to protect the interpreter. This is one reason Python programmers prefer loops for deep iteration.

### Amortized analysis

When a list is full and you append, CPython allocates a larger block and copies every reference. Doubling is the textbook example: after copying `1 + 2 + 4 + ... + n/2` elements, total copy work is less than `2n`, so `n` appends cost `O(n)` in total, which is `O(1)` per append. CPython actually grows by about 12.5 percent plus a small constant, not by doubling, and the argument still holds: growth by any constant factor gives `O(1)` amortized appends.

Amortized is different from average-case. Average-case needs an assumption about the input distribution. Amortized is a guarantee about any sequence of operations, with no probability involved.

### Hidden loops

Many one-line Python expressions contain a loop. They count toward your complexity.

- `x in some_list`, `some_list.index(x)`, `some_list.count(x)`: `O(n)`.
- `sum(values)`, `max(values)`, `min(values)`, `len(list(iterable))`: `O(n)`.
- `sorted(values)`: `O(n log n)`.
- `values[a:b]`: `O(b - a)`, because it copies.
- `"".join(parts)`: `O(total length)`.

A single `if x in some_list` inside a loop over `n` items turns the program quadratic without any visible nested `for`.

### Constants still matter

Big-O hides constant factors. An `O(n)` algorithm that does 500 operations per item can be slower than an `O(n log n)` algorithm that does 5 for any `n` you will ever see. For very small inputs, simple quadratic algorithms such as insertion sort beat sophisticated ones, which is why Python's `sorted` uses insertion sort on short runs. Use complexity to choose between algorithms that are far apart, and measure when they are close.

### Measuring growth empirically with timeit

`timeit` runs code many times and reports a reliable time. A doubling test runs the same function at `n`, `2n`, `4n`, and so on, and looks at the ratio between consecutive times.

```python
import timeit
from collections.abc import Callable


def best_time(func: Callable[[], object], repeat: int = 5) -> float:
    """Best of several runs, which reduces noise from other processes."""
    return min(timeit.repeat(func, number=1, repeat=repeat))


def doubling_test(
    work: Callable[[list[int]], object],
    start: int = 1_000,
    rounds: int = 5,
) -> None:
    previous = 0.0
    n = start
    for _ in range(rounds):
        data = list(range(n))
        elapsed = best_time(lambda: work(data))
        ratio = elapsed / previous if previous else 0.0
        print(f"n={n:>8}  time={elapsed:.6f}s  ratio={ratio:.2f}")
        previous = elapsed
        n *= 2
```

How to read the ratio column:

- Ratio near 1: constant or logarithmic.
- Ratio near 2: linear.
- Ratio slightly above 2: linearithmic (`n log n`).
- Ratio near 4: quadratic.
- Ratio near 8: cubic.

Timing is noisy for small `n`. Start large enough that each run takes at least a few milliseconds.

## Industry-level snippet

Performance regressions are expensive to find in production. This module estimates how fast a function's cost grows and fails a test when growth is steeper than allowed. It fits a straight line to `log(cost)` against `log(size)`, and the slope is the exponent `k` in `cost ~ size^k`: a slope near 1 means linear, near 2 means quadratic.

Notice the design: the code that decides whether growth is acceptable takes a plain `cost_of(size)` function. Production code passes a timer. Unit tests pass exact mathematical functions, so the tests are deterministic and never flaky. Real wall-clock checks belong in a separate benchmark job, not in the unit test suite.

```python
# growth_guard.py
import logging
import math
import time
from collections.abc import Callable, Sequence

logger = logging.getLogger(__name__)


class GrowthMeasurementError(ValueError):
    """Raised when sizes or costs cannot be used to estimate growth."""


class PerformanceRegressionError(AssertionError):
    """Raised when measured growth is steeper than allowed."""


def estimate_exponent(sizes: Sequence[int], costs: Sequence[float]) -> float:
    """Estimate k in cost ~ size**k with a least-squares fit on log-log data."""
    if len(sizes) != len(costs):
        raise GrowthMeasurementError("sizes and costs must have the same length")
    if len(sizes) < 2:
        raise GrowthMeasurementError("at least two measurements are required")
    if any(size <= 0 for size in sizes) or any(cost <= 0 for cost in costs):
        raise GrowthMeasurementError("sizes and costs must be positive")

    xs = [math.log(size) for size in sizes]
    ys = [math.log(cost) for cost in costs]
    mean_x = sum(xs) / len(xs)
    mean_y = sum(ys) / len(ys)
    spread = sum((x - mean_x) ** 2 for x in xs)
    if spread == 0:
        raise GrowthMeasurementError("sizes must not all be equal")
    pairs = zip(xs, ys, strict=True)
    covariance = sum((x - mean_x) * (y - mean_y) for x, y in pairs)
    return covariance / spread


def describe_growth(exponent: float) -> str:
    if exponent < 0.3:
        return "constant or logarithmic"
    if exponent < 1.25:
        return "linear or n log n"
    if exponent < 1.75:
        return "between n log n and quadratic"
    if exponent < 2.4:
        return "quadratic"
    return "cubic or worse"


def timed[T](
    work: Callable[[T], object],
    make_input: Callable[[int], T],
    repeat: int = 5,
) -> Callable[[int], float]:
    """Return a cost function measuring the best-of-N wall time of `work`.

    `work` must not modify its input, because the same input is reused for
    every repetition. Building the input is not part of the measured time.
    """
    if repeat < 1:
        raise GrowthMeasurementError("repeat must be at least 1")

    def cost_of(size: int) -> float:
        data = make_input(size)
        best = math.inf
        for _ in range(repeat):
            started = time.perf_counter()
            work(data)
            best = min(best, time.perf_counter() - started)
        return best

    return cost_of


def check_growth(
    cost_of: Callable[[int], float],
    sizes: Sequence[int],
    max_exponent: float,
) -> float:
    """Measure growth and raise if it is steeper than `max_exponent`."""
    costs = [cost_of(size) for size in sizes]
    exponent = estimate_exponent(sizes, costs)
    logger.info(
        "growth exponent %.2f (%s) for sizes %s",
        exponent,
        describe_growth(exponent),
        list(sizes),
    )
    if exponent > max_exponent:
        raise PerformanceRegressionError(
            f"exponent {exponent:.2f} exceeds the allowed {max_exponent:.2f}"
        )
    return exponent
```

```python
# test_growth_guard.py
import pytest

from growth_guard import (
    GrowthMeasurementError,
    PerformanceRegressionError,
    check_growth,
    describe_growth,
    estimate_exponent,
)

SIZES = [1_000, 2_000, 4_000, 8_000]


def test_linear_cost_has_exponent_one() -> None:
    costs = [float(size) for size in SIZES]
    assert estimate_exponent(SIZES, costs) == pytest.approx(1.0)


def test_quadratic_cost_has_exponent_two() -> None:
    costs = [float(size * size) for size in SIZES]
    assert estimate_exponent(SIZES, costs) == pytest.approx(2.0)


@pytest.mark.parametrize(
    ("sizes", "costs"),
    [
        ([1], [1.0]),
        ([1, 2], [1.0]),
        ([0, 2], [1.0, 2.0]),
        ([1, 2], [1.0, -2.0]),
        ([5, 5], [1.0, 2.0]),
    ],
)
def test_bad_measurements_are_rejected(sizes: list[int], costs: list[float]) -> None:
    with pytest.raises(GrowthMeasurementError):
        estimate_exponent(sizes, costs)


def test_check_growth_accepts_linear_work() -> None:
    exponent = check_growth(lambda size: float(size), SIZES, max_exponent=1.25)
    assert exponent == pytest.approx(1.0)


def test_check_growth_flags_quadratic_work() -> None:
    with pytest.raises(PerformanceRegressionError):
        check_growth(lambda size: float(size * size), SIZES, max_exponent=1.25)


@pytest.mark.parametrize(
    ("exponent", "label"),
    [
        (0.0, "constant or logarithmic"),
        (1.0, "linear or n log n"),
        (2.0, "quadratic"),
        (3.0, "cubic or worse"),
    ],
)
def test_describe_growth(exponent: float, label: str) -> None:
    assert describe_growth(exponent) == label
```

In a real project you would wire `timed(my_function, build_sample_input)` into a nightly benchmark job and keep `check_growth` with synthetic cost functions in the fast unit tests.

## Common mistakes

- Confusing complexity with speed. An `O(n)` algorithm with a huge constant can lose to an `O(n log n)` one on realistic inputs.
- Using one variable for two inputs. Two lists of lengths `a` and `b` give `O(a + b)` or `O(a * b)`, not `O(n)`.
- Missing hidden loops, especially `in` on a list, slicing, `sum`, and `max` inside another loop.
- Forgetting space. Memory is a resource just like time, and recursion uses stack space.
- Calling every expensive operation `O(n^2)`. Count the work carefully: a triangular loop is quadratic, but a loop whose inner bound halves each time is `O(n)` overall.
- Treating amortized as average. Amortized is a guarantee over a sequence of operations, not a probability statement.
- Keeping constants and lower-order terms in the final answer. Write `O(n^2)`, not `O(3n^2 + 7n)`.
- Confusing best case with Big-Omega, or worst case with Big-O. They are different dimensions.
- Benchmarking with one tiny input. One data point cannot reveal growth. Use at least four sizes.
- Writing unit tests that depend on wall-clock timing. They fail randomly on busy machines. Separate measurement from decision logic, as the snippet above does.

## Practice exercises

1. State the time and space complexity of each function and justify your answer in one sentence.

   ```python
   def a(values: list[int]) -> int:
       return values[len(values) // 2]


   def b(values: list[int]) -> int:
       return max(values) - min(values)


   def c(n: int) -> int:
       total = 0
       for i in range(n):
           for j in range(n):
               total += i * j
       return total


   def d(n: int) -> int:
       total = 0
       i = n
       while i > 0:
           total += i
           i //= 2
       return total
   ```

2. How many times does the innermost statement run, as a function of `n`? Give the exact count and the Big-O.

   ```python
   def count(n: int) -> int:
       steps = 0
       for i in range(n):
           for j in range(i, n):
               steps += 1
       return steps
   ```

3. Show using the formal definition that `7n + 12` is `O(n)`. Give explicit values for `c` and `n0`.
4. A function runs a loop of `n` steps and then a nested loop of `m` by `m` steps. What is its complexity in terms of `n` and `m`? What changes if `m` is always less than `n`?
5. Write the recurrence and the complexity of this function.

   ```python
   def halve(values: list[int]) -> int:
       if len(values) <= 1:
           return len(values)
       middle = len(values) // 2
       return halve(values[:middle]) + 1
   ```

   Then rewrite it using indices instead of slices and say how the complexity changes.
6. Sketch the recursion tree for `T(n) = 2T(n / 2) + n` for `n = 16`. Count the work at each level and the number of levels.
7. List three expressions from this note that hide a loop, and write one example where each makes a program quadratic.
8. Explain in your own words why `n` appends to a Python list cost `O(n)` in total even though one append occasionally copies the whole list.
9. A function has best case `O(1)`, average case `O(log n)`, and worst case `O(n)`. Give a realistic example of an algorithm or data structure operation with that profile and explain each case.
10. Run `doubling_test` on `sorted`, on `sum`, and on your own `has_duplicate_quadratic`. Record the ratio column for each and decide which complexity each one suggests.
11. Predict, without running it, what happens to the runtime of `has_duplicate_quadratic` and `has_duplicate_linear` on a list of 100,000 unique numbers. Then run both and compare with your prediction.
12. Write a function `count_pairs_with_sum(values, target)` in two ways: a quadratic way and a linear-time way using a dictionary. State the time and space complexity of both and the trade-off between them.
13. Extend `estimate_exponent` so it also returns the largest relative error between the fitted line and the measured points, and add tests for it.

## Summary

Complexity analysis describes how the cost of an algorithm grows with input size, ignoring constants and lower-order terms. Big-O is an upper bound, Big-Omega a lower bound, and Big-Theta a tight bound, while best, average, and worst case describe different inputs. The growth rates to know are constant, logarithmic, linear, linearithmic, quadratic, cubic, exponential, and factorial. Loops multiply when nested and add when sequential, and recursion is analyzed with recurrences and recursion trees. Space includes the call stack, amortized analysis spreads rare expensive steps across many cheap ones, and hidden loops in built-ins count. Constants matter in practice, so confirm your reasoning by measuring with `timeit` and doubling tests.
