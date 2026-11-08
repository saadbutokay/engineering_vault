## Overview

You do not need advanced mathematics to learn DSA, but a small toolkit of ideas appears again and again: summing series to count loop iterations, logarithms to understand halving, modular arithmetic to wrap indexes and bucket data, `gcd` and primes for number problems, fast exponentiation for huge powers, counting formulas to size search spaces, probability to reason about randomized algorithms, and proofs to convince yourself that an algorithm is correct.

This note teaches each idea with a short explanation, Python code, and the places where it shows up in real systems: hash tables, sharding, ring buffers, rate limiters, A/B tests, and cryptographic hashing. Every formula here is something you will reuse.

You will cover summations and series, logarithms and exponents, modular arithmetic, `gcd` and Euclid's algorithm, primes and the Sieve of Eratosthenes, fast exponentiation, permutations and combinations, probability basics for randomized algorithms, proof by induction, loop invariants, and proof by contradiction.

## Core concepts

### Summations and series

A summation adds up a pattern. Four to memorize:

```text
1 + 2 + 3 + ... + n          = n(n + 1) / 2         (arithmetic series, about n^2 / 2)
1 + 4 + 9 + ... + n^2        = n(n + 1)(2n + 1) / 6 (about n^3 / 3)
1 + 2 + 4 + ... + 2^k        = 2^(k + 1) - 1        (geometric series, about the last term times 2)
1 + 1/2 + 1/3 + ... + 1/n    = about ln(n)          (harmonic series)
```

Where they appear:

- A triangular loop (the inner loop runs `i` times for each `i`) does `n(n + 1) / 2` steps, so it is `O(n^2)`.
- Doubling an array's capacity copies `1 + 2 + 4 + ...` elements, less than `2n` in total, so appends are `O(1)` amortized.
- The harmonic series explains why the Sieve of Eratosthenes does about `n log log n` work, and why some `n / 1 + n / 2 + ... + n / n` loops are `O(n log n)`.

### Logarithms and exponents

`log2(n)` is the number of times you can halve `n` before reaching 1. Exponents and logarithms undo each other: `2^10 = 1024` and `log2(1024) = 10`.

- `log(a * b) = log(a) + log(b)` and `log(a^k) = k * log(a)`.
- Changing the base multiplies by a constant, so Big-O writes `O(log n)` for any base.
- Binary search over a million items takes at most about 20 steps, over a billion about 30.
- The number of bits needed to write `n` in binary is `floor(log2(n)) + 1`, which Python exposes as `n.bit_length()`.

### Modular arithmetic

The remainder operation `a % m` gives a number in `0 .. m - 1`. Two numbers are congruent modulo `m` when they have the same remainder. Modular arithmetic is arithmetic on a clock face: after the last position it wraps around.

- `(a + b) % m == ((a % m) + (b % m)) % m`, and the same holds for multiplication. You may reduce at every step, which keeps numbers small.
- `a // m` is the quotient and `a % m` is the remainder, so `a == (a // m) * m + a % m`. `divmod(a, m)` returns both.
- A ring buffer moves to the next position with `(index + 1) % capacity`.
- Hash tables map a hash to a slot with `hash % capacity` (or a bit mask when capacity is a power of two).
- Many problems ask for the answer "modulo `10**9 + 7`", a prime that keeps results inside 64 bits in other languages.

### Greatest common divisor and Euclid's algorithm

The `gcd` of two integers is the largest number that divides both. Euclid's algorithm relies on the fact that `gcd(a, b) = gcd(b, a % b)`, so you replace the pair by smaller numbers until the second one is zero. It takes `O(log min(a, b))` steps. The least common multiple follows from `lcm(a, b) = a // gcd(a, b) * b`.

### Primes and the Sieve of Eratosthenes

A prime is an integer greater than 1 with no divisors besides 1 and itself. To test one number, trial division only needs divisors up to `isqrt(n)`. To find all primes up to `n`, the Sieve of Eratosthenes crosses out multiples of each prime, starting from its square, and runs in about `O(n log log n)` time.

### Fast exponentiation

Computing `base ** exponent` by multiplying `exponent` times costs `O(exponent)` multiplications. Square-and-multiply uses the binary digits of the exponent and needs only `O(log exponent)` multiplications. With a modulus, you reduce after every multiplication so the numbers never grow. Python's built-in `pow(base, exponent, modulus)` does exactly this.

### Counting

- Rule of product: independent choices multiply. A 4-digit PIN has `10 ** 4` possibilities.
- A set of `n` items has `2 ** n` subsets.
- Orderings of `n` distinct items: `n!` (`math.factorial`).
- Ordered selections of `k` from `n`: `n! / (n - k)!` (`math.perm`).
- Unordered selections of `k` from `n`: `n! / (k! * (n - k)!)` (`math.comb`).
- Pigeonhole principle: if you place more than `m` items into `m` boxes, some box holds at least two.

### Probability basics

- Probability is a number from 0 to 1. Independent events multiply: two fair coin flips both land heads with probability `1/2 * 1/2 = 1/4`.
- Expected value is the probability-weighted average outcome. By linearity of expectation, the expected value of a sum is the sum of the expected values, even when the parts are not independent.
- Randomized quicksort picks a random pivot, so no particular input is bad: its expected number of comparisons is about `1.39 * n * log2(n)`.
- The birthday paradox: with `k` items in `m` slots the probability of no collision is about `exp(-k^2 / (2m))`. Collisions begin after roughly `sqrt(m)` insertions, much earlier than intuition suggests.
- A Las Vegas algorithm always returns the correct answer and its running time is random. A Monte Carlo algorithm has a fixed running time and a small probability of a wrong answer.

### Proof techniques

- Induction: prove a base case, then prove that if the claim holds for `k` then it holds for `k + 1`. All larger cases follow.
- Loop invariant: a statement that holds before the loop, is preserved by each iteration, and together with the exit condition implies correctness.
- Contradiction: assume the claim is false and derive an impossibility.

## Basic syntax examples

### Arithmetic operators and helpers

```python
import math

print(17 // 5)  # 3, floor division
print(17 % 5)  # 2, remainder
print(divmod(17, 5))  # (3, 2)
print(2**10)  # 1024
print(pow(2, 10, 1000))  # 24, which is 1024 % 1000
print(math.gcd(12, 18), math.lcm(4, 6))  # 6 12
print(math.factorial(5), math.perm(5, 2), math.comb(5, 2))  # 120 20 10
print(math.isqrt(50))  # 7, an exact integer square root
print(math.log2(1024))  # 10.0
print((1000).bit_length())  # 10, the number of bits in 1000
```

### Series in code

```python
def sum_to(n: int) -> int:
    """1 + 2 + ... + n in O(1) with the arithmetic series formula."""
    return n * (n + 1) // 2


def sum_to_loop(n: int) -> int:
    """The same result in O(n), useful for checking the formula."""
    return sum(range(1, n + 1))


assert all(sum_to(n) == sum_to_loop(n) for n in range(100))
```

### Euclid's algorithm

```python
def gcd(a: int, b: int) -> int:
    while b:
        a, b = b, a % b
    return a


print(gcd(48, 18))  # 6
```

Trace of `gcd(48, 18)`:

```text
a=48, b=18   ->  a=18, b=12   (48 % 18 = 12)
a=18, b=12   ->  a=12, b=6    (18 % 12 = 6)
a=12, b=6    ->  a=6,  b=0    (12 % 6  = 0)
b is 0, so the answer is 6
```

### Primes

```python
import math


def is_prime(n: int) -> bool:
    """Trial division up to the square root: O(sqrt(n))."""
    if n < 2:
        return False
    for divisor in range(2, math.isqrt(n) + 1):
        if n % divisor == 0:
            return False
    return True


def primes_up_to(limit: int) -> list[int]:
    """Sieve of Eratosthenes: O(n log log n) time, O(n) space."""
    if limit < 2:
        return []
    flags = bytearray([1]) * (limit + 1)
    flags[0] = flags[1] = 0
    for candidate in range(2, math.isqrt(limit) + 1):
        if flags[candidate]:
            flags[candidate * candidate :: candidate] = bytes(
                len(range(candidate * candidate, limit + 1, candidate))
            )
    return [number for number, flag in enumerate(flags) if flag]


print(primes_up_to(30))  # [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

### Fast modular exponentiation

```python
def power_mod(base: int, exponent: int, modulus: int) -> int:
    """Square-and-multiply: O(log exponent) multiplications."""
    if exponent < 0 or modulus < 1:
        raise ValueError("exponent must be >= 0 and modulus must be >= 1")
    result = 1 % modulus
    base %= modulus
    while exponent > 0:
        if exponent & 1:  # the lowest binary digit is 1
            result = result * base % modulus
        base = base * base % modulus
        exponent >>= 1
    return result


assert power_mod(2, 10, 1000) == 24
assert power_mod(7, 1_000_000, 13) == pow(7, 1_000_000, 13)
```

### Counting in code

```python
import math
from itertools import combinations

print(math.comb(52, 5))  # 2598960 possible poker hands
print(len(list(combinations(range(6), 3))), math.comb(6, 3))  # 20 20
print(2**20)  # 1048576 subsets of a 20-element set
```

## Deeper behavior

### Python's division and modulo for negative numbers

In Python, `//` rounds toward negative infinity and `%` always takes the sign of the divisor, so `a == (a // b) * b + a % b` always holds. In C, Java, and many other languages `%` follows the sign of the dividend, so ports of algorithms can behave differently.

```python
print(-7 // 2)  # -4, not -3
print(-7 % 3)  # 2, never negative when the divisor is positive
print(int(-7 / 2))  # -3, this truncates toward zero instead
```

The non-negative result is convenient for wrapping indexes, since `(index - 1) % capacity` always lands inside the buffer even when `index` is 0.

### Prefer integer math to floats

Floats have 53 bits of precision, so they silently lose accuracy on large integers. `math.isqrt` is exact for any size and `n ** 0.5` is not:

```python
import math

n = 4_503_599_761_588_224  # 67108865 squared, minus 1
print(int(n**0.5))  # 67108865, wrong
print(math.isqrt(n))  # 67108864, correct
```

To compute `floor(log2(n))` for a positive integer, use `n.bit_length() - 1`. It is exact, while `math.log2(n)` can round up near large powers of two.

### Overflow does not happen, but cost does

Python integers grow without limit, so there is no silent overflow like in C or Java. A loop that multiplies without a modulus will keep producing bigger and bigger numbers, and each multiplication costs more than the last. This is the reason to reduce modulo `m` after every step in modular problems. When you port a solution to a language with fixed-size integers, remember that the overflow behavior changes.

### Why Euclid's algorithm is fast

After two steps of `(a, b) -> (b, a % b)`, the larger number is at most half of what it was. So the number of steps is `O(log min(a, b))`, even for 100-digit numbers. The worst case is consecutive Fibonacci numbers.

### Sieve details

The inner loop starts at `candidate * candidate`, because smaller multiples were already crossed out by smaller primes. The outer loop stops at `isqrt(limit)`, because any composite number up to `limit` has a divisor no larger than its square root. The slice assignment replaces the Python-level inner loop with a single C-level operation, which makes the sieve far faster than a naive implementation. A `bytearray` uses one byte per number, so a sieve to 10 million needs about 10 MB.

### Fast exponentiation and the binary digits

The binary form of the exponent drives the algorithm. For `exponent = 13`, which is `1101` in binary, the result is `base^8 * base^4 * base^1`. The loop squares the base each step to produce `base^1, base^2, base^4, base^8`, and multiplies in the ones that match a 1 digit. Modular inverses use the same built-in: `pow(a, -1, m)` returns the inverse of `a` modulo `m` when it exists (Python 3.8 and newer).

### Hash functions, buckets, and modulo

`hash(x) % capacity` maps any hash to a slot. CPython's string hash is randomized per process (`PYTHONHASHSEED`), so it is a good table hash and a terrible stable identifier: the same string hashes differently after a restart. For anything that must be stable across processes or machines, such as shard routing or experiment buckets, use a fixed algorithm from `hashlib`. Hashing a value and taking it modulo a bucket count has a tiny bias when the hash range is not a multiple of the bucket count. With a 64-bit hash and 10,000 buckets the bias is about one part in 10^16, which is irrelevant in practice.

### Expected value, randomization, and worst cases

Randomization turns a bad worst case into a bad but unlikely one. Randomized quicksort has worst-case `O(n^2)`, but because the pivot is random, the chance of repeatedly picking bad pivots is astronomically small, and the expected cost is `O(n log n)` for every input. Hash tables use the same idea with randomized hash seeds. When you test randomized code, seed the generator (`random.Random(seed)`) so failures can be reproduced.

### An induction proof, step by step

Claim: `1 + 2 + ... + n = n(n + 1) / 2` for every `n >= 1`.

```text
Base case: n = 1.   Left side = 1.  Right side = 1 * 2 / 2 = 1.  They match.

Step: assume the claim is true for n = k, that is
        1 + 2 + ... + k = k(k + 1) / 2.
      Add (k + 1) to both sides:
        1 + 2 + ... + k + (k + 1) = k(k + 1) / 2 + (k + 1)
                                  = (k + 1)(k / 2 + 1)
                                  = (k + 1)(k + 2) / 2.
      That is the claim for n = k + 1.

By induction the claim is true for every n >= 1.
```

### A loop invariant, step by step

```python
import math


def gcd_with_invariant(a: int, b: int) -> int:
    """Euclid's algorithm with its invariant checked by assert (debugging only)."""
    expected = math.gcd(a, b)  # used only to check the invariant, never to compute
    while b:
        assert math.gcd(a, b) == expected  # invariant: the gcd never changes
        a, b = b, a % b
    return a
```

The mathematical content is in three steps:

- Initialization: before the loop, the invariant holds trivially, because `(a, b)` is the original pair.
- Maintenance: `gcd(a, b) = gcd(b, a % b)`, so replacing `(a, b)` by `(b, a % b)` preserves the greatest common divisor.
- Termination: `b` strictly decreases and stays non-negative, so it reaches 0. Then `gcd(a, 0) = a`, which is the answer.

### A proof by contradiction

Claim: if `n > 1` is composite, it has a divisor `d` with `1 < d <= sqrt(n)`.

```text
Suppose n = a * b with 1 < a <= b, and suppose (for contradiction)
that every divisor of n is larger than sqrt(n).
Then a > sqrt(n) and b > sqrt(n), so a * b > sqrt(n) * sqrt(n) = n.
But a * b = n.  Contradiction.
So the smaller factor a satisfies a <= sqrt(n).
```

This is exactly why trial division and the sieve stop at the square root.

## Industry-level snippet

A product team runs A/B tests. Each user must be assigned to a variant deterministically: the same user always sees the same variant for a given experiment, assignments across experiments should look independent, and the split must match the configured percentages. The mathematics is a handful of the ideas above:

- A cryptographic hash (`sha256`) of `experiment:user_id` gives a stable, well-mixed number. The built-in `hash()` is not usable because it changes between processes.
- Taking that number modulo `10_000` gives a bucket from 0 to 9,999, so weights can be expressed in basis points (hundredths of a percent).
- `itertools.accumulate` turns weights into cumulative boundaries, and `bisect_right` finds the variant in `O(log v)` for `v` variants.
- A single transaction records assignments exactly once per user and experiment, even if the job reruns.

```python
# experiment_buckets.py
import hashlib
import logging
import sqlite3
from bisect import bisect_right
from collections.abc import Iterable, Sequence
from dataclasses import dataclass
from itertools import accumulate

logger = logging.getLogger(__name__)

BUCKET_COUNT = 10_000

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS assignment (
    experiment TEXT NOT NULL,
    user_id INTEGER NOT NULL,
    variant TEXT NOT NULL,
    PRIMARY KEY (experiment, user_id)
)
"""

INSERT_SQL = (
    "INSERT INTO assignment (experiment, user_id, variant) VALUES (?, ?, ?) "
    "ON CONFLICT (experiment, user_id) DO NOTHING"
)


class InvalidExperimentError(ValueError):
    """Raised when an experiment definition is inconsistent."""


@dataclass(frozen=True, slots=True)
class Variant:
    name: str
    weight: int  # share of BUCKET_COUNT: 5_000 means 50 percent


class Experiment:
    def __init__(self, name: str, variants: Sequence[Variant]) -> None:
        if not name:
            raise InvalidExperimentError("experiment name must not be empty")
        if not variants:
            raise InvalidExperimentError("at least one variant is required")
        if any(variant.weight <= 0 for variant in variants):
            raise InvalidExperimentError("variant weights must be positive")
        total = sum(variant.weight for variant in variants)
        if total != BUCKET_COUNT:
            raise InvalidExperimentError(
                f"weights must add up to {BUCKET_COUNT}, got {total}"
            )
        self.name = name
        self._names = [variant.name for variant in variants]
        self._boundaries = list(accumulate(variant.weight for variant in variants))

    def bucket_for(self, user_id: int) -> int:
        """Stable bucket in range(BUCKET_COUNT): same inputs, same answer, always."""
        digest = hashlib.sha256(f"{self.name}:{user_id}".encode()).digest()
        return int.from_bytes(digest[:8], "big") % BUCKET_COUNT

    def assign(self, user_id: int) -> str:
        """Find the variant whose weight range contains the user's bucket."""
        bucket = self.bucket_for(user_id)
        return self._names[bisect_right(self._boundaries, bucket)]


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def record_assignments(
    connection: sqlite3.Connection,
    experiment: Experiment,
    user_ids: Iterable[int],
) -> int:
    """Persist assignments in one transaction and return how many were new."""
    rows = [(experiment.name, uid, experiment.assign(uid)) for uid in user_ids]
    try:
        with connection:
            before = connection.total_changes
            connection.executemany(INSERT_SQL, rows)
            inserted = connection.total_changes - before
    except sqlite3.Error:
        logger.exception(
            "failed to record %d assignments for %s", len(rows), experiment.name
        )
        raise
    logger.info(
        "recorded %d new assignments (%d submitted) for %s",
        inserted,
        len(rows),
        experiment.name,
    )
    return inserted
```

```python
# test_experiment_buckets.py
import sqlite3
from collections.abc import Iterator

import pytest

from experiment_buckets import (
    BUCKET_COUNT,
    Experiment,
    InvalidExperimentError,
    Variant,
    create_schema,
    record_assignments,
)

HALF_AND_HALF = [Variant("control", 5_000), Variant("new", 5_000)]


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def test_bucket_is_stable_and_in_range() -> None:
    experiment = Experiment("checkout", HALF_AND_HALF)
    bucket = experiment.bucket_for(42)
    assert bucket == experiment.bucket_for(42)
    assert 0 <= bucket < BUCKET_COUNT


def test_assignment_is_stable() -> None:
    experiment = Experiment("checkout", HALF_AND_HALF)
    assert experiment.assign(7) == Experiment("checkout", HALF_AND_HALF).assign(7)


def test_split_is_close_to_the_configured_weights() -> None:
    experiment = Experiment(
        "checkout", [Variant("control", 7_000), Variant("new", 3_000)]
    )
    assigned = [experiment.assign(user_id) for user_id in range(20_000)]
    assert abs(assigned.count("new") - 6_000) < 300


def test_experiments_assign_independently() -> None:
    first = Experiment("checkout", HALF_AND_HALF)
    second = Experiment("search", HALF_AND_HALF)
    differing = [u for u in range(1_000) if first.assign(u) != second.assign(u)]
    assert len(differing) > 100


@pytest.mark.parametrize(
    "variants",
    [
        [],
        [Variant("only", 9_999)],
        [Variant("a", 10_001)],
        [Variant("a", 10_000), Variant("b", 0)],
        [Variant("a", 12_000), Variant("b", -2_000)],
    ],
)
def test_invalid_weights_are_rejected(variants: list[Variant]) -> None:
    with pytest.raises(InvalidExperimentError):
        Experiment("checkout", variants)


def test_empty_name_is_rejected() -> None:
    with pytest.raises(InvalidExperimentError):
        Experiment("", HALF_AND_HALF)


def test_recording_is_idempotent(connection: sqlite3.Connection) -> None:
    experiment = Experiment("checkout", HALF_AND_HALF)
    assert record_assignments(connection, experiment, [1, 2, 3]) == 3
    assert record_assignments(connection, experiment, [2, 3, 4]) == 1
    count = connection.execute("SELECT COUNT(*) FROM assignment").fetchone()[0]
    assert count == 4
```

`bisect_right` never runs past the end of the list: the largest bucket is `9_999`, and the last boundary is `10_000`, so the index is always a valid position. That is the kind of small argument worth writing down as a comment when you rely on it.

## Common mistakes

- Using `/` when you mean `//`. In Python 3, `/` always returns a float.
- Using floats for integer problems. `int(n ** 0.5)` and `math.log2` can be off by one for large values; use `math.isqrt` and `bit_length`.
- Assuming `%` behaves like C for negative numbers. Python's result takes the sign of the divisor.
- Forgetting to reduce modulo `m` at each step of a computation that is supposed to stay small.
- Writing `range(n)` when the problem needs the value `n` included, as in a sieve up to `n`, which needs `range(n + 1)`.
- Starting a sieve's inner loop at `2 * candidate` instead of `candidate * candidate`, which does extra work.
- Computing `base ** exponent % modulus` for huge exponents. It builds the giant number first. Use `pow(base, exponent, modulus)`.
- Using the built-in `hash()` for anything that must be stable across restarts or machines.
- Dividing by zero in a modulus or `gcd` helper without checking the input.
- Confusing combinations (order does not matter) with permutations (order matters).
- Treating a few passing examples as a proof. Examples can only disprove a claim.
- Forgetting the base case in an induction argument or in a recursive function.

## Practice exercises

1. Without running code, find the number of steps of `for i in range(n): for j in range(i, n): ...` and express it as a formula. Then run it for `n = 10` and check.
2. Prove by induction that `1 + 3 + 5 + ... + (2n - 1) = n^2` for every `n >= 1`.
3. Write `is_power_of_two(n)` using only integer arithmetic and `%`, and again using `bit_length`. State which one you prefer and why.
4. How many times can you halve 1,000,000 before reaching 1? Compute it with a loop, with `math.log2`, and with `bit_length`, and compare the three answers.
5. Write `mod_index(i, size)` that moves an index backwards in a ring buffer, `(i - 1) % size`, and confirm it behaves correctly at index 0. Explain why a C-style `%` would need a special case.
6. Write `lcm_many(values)` that returns the least common multiple of a list using `math.gcd`, and test it on at least five lists, including one with a single element.
7. Implement `gcd` recursively and iteratively. Count the number of steps each takes for `gcd(10**18, 10**18 - 1)` and for two consecutive Fibonacci numbers of your choice.
8. Write `count_primes(n)` using the sieve. Check your answer for `n = 100` (25 primes) and `n = 1_000_000` (78,498 primes), and time it.
9. Write a trial-division `is_prime` and check it against the sieve for every number below 10,000. Report any disagreement.
10. Implement `power_mod` iteratively and recursively. Verify them against `pow` on 1,000 random inputs.
11. How many different 8-character passwords can be formed from 26 lowercase letters and 10 digits, with repetition allowed? How many if no character may repeat? Use Python to compute both.
12. Using the birthday formula, estimate how many keys you can insert into a hash table with 1,000,000 slots before a collision is more likely than not. Then simulate it with `random` and a seeded generator.
13. A fair six-sided die is rolled twice. What is the expected value of the sum? What is the probability that the sum is 7? Verify both by enumeration with `itertools.product`.
14. Prove by contradiction that there are infinitely many primes. Write the proof in your own words in five lines or fewer.
15. Extend `Experiment` so that it also accepts weights as percentages with two decimal places (for example `33.33`), converting to basis points, and add tests for rounding.

## Summary

A few mathematical ideas power most of DSA. Series formulas count loop iterations and explain amortized costs. Logarithms measure halving and tie directly to binary search and tree height. Modular arithmetic wraps indexes, maps hashes to buckets, and keeps numbers small. Euclid's algorithm computes `gcd` in logarithmic time, the sieve finds primes quickly, and square-and-multiply, available as `pow(base, exponent, modulus)`, makes huge modular powers cheap. Counting formulas size search spaces, and probability explains randomized algorithms and hash collisions. Induction, loop invariants, and contradiction turn "it seems to work" into "it must work". In Python, prefer integer tools over floats, remember that `%` and `//` round toward negative infinity, and use `hashlib` rather than `hash()` when a value must be stable.
