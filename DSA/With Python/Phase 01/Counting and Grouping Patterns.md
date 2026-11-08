## Overview

A large share of real programming problems are really counting problems in disguise. How many times did each error occur? Which two numbers add up to the target? Which words are anagrams of each other? Which customers were charged twice? Each question has the same shape: walk through the data once, and use a hash table to remember what you have seen, how often, or which items belong together.

This note collects the patterns that come from that idea. Frequency maps answer "how many?", the Two Sum family answers "have I already seen the partner of this value?", grouping with a canonical key answers "which items are equivalent?", and the Boyer-Moore voting algorithm answers "is there a majority?" in constant extra space. The pigeonhole principle explains why some of these questions have guaranteed answers.

Learn these patterns well. They turn many problems that look quadratic into single `O(n)` passes.

## Core concepts

### Frequency maps

A frequency map stores how many times each distinct value appears. In Python it is a `dict[value, int]` or, more conveniently, a `collections.Counter`. Building one costs `O(n)` time and `O(k)` space for `k` distinct values.

Frequency maps answer questions such as:

- Which value is most common?
- Does any value appear more than once?
- Are two collections made of the same items (the same multiset)?
- Is there an element whose count satisfies a rule?

### The Two Sum family

Two Sum asks for two items whose values add up to a target. The brute force checks every pair in `O(n^2)`. The hash-table solution walks the list once and, for each value, asks "have I already seen `target - value`?" That is a single lookup, so the whole pass is `O(n)`. The family has many members:

- Return the indexes of the pair.
- Count the number of pairs.
- Count pairs whose difference is `k`.
- Pairs of values divisible by a number, using remainders.

### Grouping with defaultdict

Grouping collects items that share a property into lists. `defaultdict(list)` creates the list the first time a key appears, so the loop body is a single line: `groups[key].append(item)`. The result is a mapping from the property to its members.

### Canonical-form keys

To group items that are "the same in some sense", compute a canonical form: a single representation that every equivalent item shares. Anagram groups use the sorted letters of each word, `"eat"`, `"tea"`, and `"ate"` all become `"aet"`. The canonical form must be hashable and must be identical for equivalent items and different for non-equivalent ones.

### First unique element and duplicate detection

- First unique element: count everything first, then scan in the original order for the first count of 1.
- Duplicate detection: a `set` of seen items, or comparing `len(set(values))` with `len(values)`.

### Majority element and Boyer-Moore voting

A majority element appears more than `n / 2` times. Boyer-Moore finds the only possible candidate with a single counter and no hash table. It treats the list as a vote in which different values cancel each other out. A majority element has more votes than all the others combined, so it survives.

### Pigeonhole reasoning

If you place more than `m` items into `m` boxes, at least one box holds two items. Typical uses: a list of `n + 1` integers drawn from `1..n` must contain a duplicate, and among 13 people two share a birth month. Pigeonhole arguments tell you a duplicate or collision is guaranteed, which often simplifies the search for it.

## Basic syntax examples

### Counting with Counter

```python
from collections import Counter

votes = ["red", "blue", "red", "green", "red", "blue"]
tally = Counter(votes)

print(tally)  # Counter({'red': 3, 'blue': 2, 'green': 1})
print(tally["red"], tally["purple"])  # 3 0, a missing key reads as zero
print(tally.most_common(2))  # [('red', 3), ('blue', 2)]
print(sum(tally.values()), len(tally))  # 6 3
```

### Duplicate detection

```python
def contains_duplicate(values: list[int]) -> bool:
    """O(n) average time, O(n) space."""
    seen: set[int] = set()
    for value in values:
        if value in seen:
            return True
        seen.add(value)
    return False


print(contains_duplicate([1, 2, 3, 1]))  # True
print(contains_duplicate([1, 2, 3, 4]))  # False
```

### Two Sum: indexes and counting pairs

```python
from collections import Counter


def two_sum(numbers: list[int], target: int) -> tuple[int, int] | None:
    """Return indexes of two different items that add up to target."""
    seen: dict[int, int] = {}  # value -> index
    for index, value in enumerate(numbers):
        partner = target - value
        if partner in seen:
            return seen[partner], index
        seen[value] = index
    return None


def count_pairs_with_sum(numbers: list[int], target: int) -> int:
    """Count pairs i < j with numbers[i] + numbers[j] == target."""
    seen: Counter[int] = Counter()
    pairs = 0
    for value in numbers:
        pairs += seen[target - value]  # every earlier partner forms a pair
        seen[value] += 1
    return pairs


print(two_sum([2, 7, 11, 15], 9))  # (0, 1)
print(count_pairs_with_sum([1, 5, 3, 3, 3], 6))  # 4
```

The second result is 4 because `1 + 5` is one pair and each of the three `3` values pairs with each other `3`, which gives three more pairs.

### Grouping anagrams

```python
from collections import defaultdict


def group_anagrams(words: list[str]) -> list[list[str]]:
    """Group words that are rearrangements of each other.

    O(n * k log k) for n words of length up to k, because each word is sorted.
    """
    groups: defaultdict[str, list[str]] = defaultdict(list)
    for word in words:
        canonical = "".join(sorted(word))  # "eat", "tea", "ate" all become "aet"
        groups[canonical].append(word)
    return list(groups.values())


print(group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"]))
# [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']]
```

### First unique element

```python
from collections import Counter


def first_unique(values: list[int]) -> int | None:
    """Return the first value that appears exactly once. Two passes, O(n)."""
    counts = Counter(values)
    for value in values:  # scan in the original order, not in dictionary order
        if counts[value] == 1:
            return value
    return None


print(first_unique([4, 5, 4, 6, 5, 7]))  # 6
```

### Majority element with Boyer-Moore voting

```python
def majority_element(values: list[int]) -> int | None:
    """Return the value that appears more than len(values) / 2 times, or None.

    O(n) time and O(1) extra space.
    """
    candidate: int | None = None
    votes = 0
    for value in values:
        if votes == 0:
            candidate = value  # nobody is ahead, adopt the current value
        votes += 1 if value == candidate else -1
    if candidate is not None and values.count(candidate) > len(values) // 2:
        return candidate  # the verification pass confirms it is a real majority
    return None


print(majority_element([2, 2, 1, 1, 2, 2, 3, 2]))  # 2
print(majority_element([1, 2, 3]))  # None
```

Trace of `majority_element([2, 2, 1, 1, 2])`:

```text
value 2: votes is 0, candidate = 2, votes = 1
value 2: same as candidate,        votes = 2
value 1: different,                votes = 1
value 1: different,                votes = 0
value 2: votes is 0, candidate = 2, votes = 1
candidate 2 appears 3 times out of 5, which is more than 2, so 2 is the majority
```

### Canonical keys

```python
from collections import Counter


def anagram_key(word: str) -> tuple[tuple[str, int], ...]:
    """A canonical key from letter counts: O(k) instead of sorting in O(k log k)."""
    return tuple(sorted(Counter(word).items()))


print(anagram_key("aab") == anagram_key("aba"))  # True
print(anagram_key("aab") == anagram_key("abb"))  # False
```

### Pigeonhole in code

```python
def find_duplicate(values: list[int]) -> int:
    """The list has n + 1 integers drawn from 1..n, so a duplicate must exist."""
    seen: set[int] = set()
    for value in values:
        if value in seen:
            return value
        seen.add(value)
    raise ValueError("input does not satisfy the pigeonhole precondition")


print(find_duplicate([1, 3, 4, 2, 2]))  # 2
```

## Deeper behavior

### Why the Two Sum pass is correct

When the loop reaches position `j`, the dictionary holds every value from positions before `j`. If a pair `(i, j)` with `i < j` adds up to the target, then at step `j` the partner `target - numbers[j]` was already stored at step `i`, so it is found. Because the lookup happens before the current value is inserted, an item is never paired with itself. If it were inserted first, `two_sum([3], 6)` would wrongly pair the single `3` with itself.

For counting pairs, the same reasoning shows that each pair is counted exactly once: at the step of its later element.

### Why Boyer-Moore works

Pair each occurrence of the majority value with an occurrence of a different value and cancel both. Each cancellation removes one majority item and one non-majority item. Since the majority element has more than half the items, at least one majority item is left unpaired at the end. The counter simulates this cancellation: `votes` is the surplus of the current candidate over the values that have been canceled against it. The algorithm guarantees only that if a majority exists, it is the candidate. It does not guarantee that the candidate is a majority, which is why the verification pass is required. Without it, `[1, 2, 3]` would return `3`.

### Counter details worth knowing

- Reading a missing key returns 0 and does not insert it. A `defaultdict(int)` would insert it.
- `Counter.update(iterable)` adds counts. `Counter.subtract(iterable)` subtracts and may leave zero or negative counts.
- The `-` operator between two Counters keeps only positive results. `Counter("aab") - Counter("ab")` is `Counter({'a': 1})`.
- `Counter.total()` (Python 3.10 and newer) sums the counts.
- `elements()` returns an iterator that repeats each key as often as its count.
- `most_common(k)` costs `O(n log k)` when `k` is given.
- Two counters are equal when they contain the same positive counts. Zero counts may or may not be stored, so compare only counters you built the same way, or call `+counter` first to drop zero and negative entries.

### defaultdict gotchas

`defaultdict` creates the default value on any read of a missing key, including a plain `groups[key]` expression, even if you only meant to check. Use `key in groups` or `groups.get(key)` to test without creating. The factory is called with no arguments: `defaultdict(list)` and `defaultdict(int)` are the usual ones.

### Designing canonical keys

A canonical key must satisfy two properties: equivalent items map to the same key, and non-equivalent items map to different keys. Common mistakes:

- `frozenset("aab")` loses the count of `a`, so `"aab"` and `"abb"` would both give `{'a', 'b'}`. Use a sorted tuple or a count table when multiplicity matters.
- A list is not hashable. Convert counters or lists to tuples.
- Case and spelling variants should be normalized first (`casefold()`, whitespace collapsed) if they should count as equal.

For words made only of lowercase letters, a tuple of 26 counts is a canonical key in `O(k)` time, but each key takes 26 slots. Sorting costs `O(k log k)` and takes `O(k)` space. For short words either is fine.

### Counting with arrays instead of dictionaries

When the values are small integers in a known range, a plain list indexed by value is simpler and faster than a dictionary: `counts = [0] * (max_value + 1)`. This is called counting, and it is the idea behind counting sort. It uses memory proportional to the range, not to the number of items, so it fits only when the range is small.

### Frequency of frequencies

Some problems ask about the counts themselves, for example "can the string become valid by removing one character so that all characters have equal frequency?" Build the frequency map, then build a frequency map of its values: `Counter(Counter(text).values())`.

### Top-K frequent items

`Counter.most_common(k)` handles it with a heap in `O(n log k)`. When `k` is large, bucket the items by count (a list where position `c` holds the items with count `c`) and read from the highest bucket down. That is `O(n)`, because counts are bounded by `n`.

## Industry-level snippet

A payments team needs to find accidental duplicate charges. A customer who double-clicks the pay button may be charged twice for the same amount in the same currency with the same reference, though the reference may differ in letter case or spacing. The detector groups payments by a canonical key and reports every group with more than one member. Support staff then refund the extras.

The solution uses the patterns of this note: a canonical key built from normalized fields, `defaultdict(list)` for grouping, and `Counter` for a quick summary. The report is saved in a transaction so a failed run does not leave a half-written report behind.

```python
# duplicate_payments.py
import logging
import sqlite3
from collections import Counter, defaultdict
from collections.abc import Iterable, Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS duplicate_payment (
    payment_id INTEGER PRIMARY KEY,
    group_id INTEGER NOT NULL
)
"""

type GroupKey = tuple[int, int, str, str]


class InvalidPaymentError(ValueError):
    """Raised when a payment has data that cannot be compared safely."""


@dataclass(frozen=True, slots=True)
class Payment:
    payment_id: int
    customer_id: int
    amount_cents: int
    currency: str
    reference: str


def canonical_key(payment: Payment) -> GroupKey:
    """Two payments are 'the same charge' when this key is equal."""
    if payment.amount_cents <= 0:
        raise InvalidPaymentError(f"payment {payment.payment_id} has a bad amount")
    reference = " ".join(payment.reference.casefold().split())
    return (
        payment.customer_id,
        payment.amount_cents,
        payment.currency.upper(),
        reference,
    )


def find_duplicate_groups(payments: Iterable[Payment]) -> list[list[int]]:
    """Return groups of payment ids that share a canonical key.

    One pass, O(n) average. Ids inside a group and the groups themselves are
    sorted so that the output is deterministic.
    """
    groups: defaultdict[GroupKey, list[int]] = defaultdict(list)
    for payment in payments:
        groups[canonical_key(payment)].append(payment.payment_id)
    duplicates = [sorted(ids) for ids in groups.values() if len(ids) > 1]
    duplicates.sort()
    return duplicates


def count_by_currency(payments: Iterable[Payment]) -> Counter[str]:
    return Counter(payment.currency.upper() for payment in payments)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_duplicate_groups(
    connection: sqlite3.Connection, groups: Sequence[Sequence[int]]
) -> int:
    """Replace the stored report atomically. Returns the number of rows written."""
    rows = [
        (payment_id, group_id)
        for group_id, ids in enumerate(groups, start=1)
        for payment_id in ids
    ]
    try:
        with connection:
            connection.execute("DELETE FROM duplicate_payment")
            connection.executemany(
                "INSERT INTO duplicate_payment (payment_id, group_id) VALUES (?, ?)",
                rows,
            )
    except sqlite3.Error:
        logger.exception("could not save %d duplicate rows", len(rows))
        raise
    logger.info("saved %d duplicate payments in %d groups", len(rows), len(groups))
    return len(rows)
```

```python
# test_duplicate_payments.py
import sqlite3
from collections.abc import Iterator

import pytest

from duplicate_payments import (
    InvalidPaymentError,
    Payment,
    canonical_key,
    count_by_currency,
    create_schema,
    find_duplicate_groups,
    save_duplicate_groups,
)


def pay(
    payment_id: int,
    customer: int = 1,
    amount: int = 5_000,
    currency: str = "usd",
    reference: str = "Order 7",
) -> Payment:
    return Payment(payment_id, customer, amount, currency, reference)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def stored(connection: sqlite3.Connection) -> list[tuple[int, int]]:
    cursor = connection.execute(
        "SELECT payment_id, group_id FROM duplicate_payment ORDER BY payment_id"
    )
    return [(int(p), int(g)) for p, g in cursor.fetchall()]


def test_canonical_key_ignores_case_and_extra_spaces() -> None:
    first = canonical_key(pay(1, reference="Order 7", currency="usd"))
    second = canonical_key(pay(2, reference="  ORDER   7 ", currency="USD"))
    assert first == second


def test_different_amounts_are_not_duplicates() -> None:
    assert find_duplicate_groups([pay(1, amount=100), pay(2, amount=200)]) == []


def test_groups_are_found_and_sorted() -> None:
    payments = [
        pay(5),
        pay(2, customer=9, reference="A"),
        pay(1),
        pay(7, customer=9, reference="a"),
        pay(3, amount=1),
    ]
    assert find_duplicate_groups(payments) == [[1, 5], [2, 7]]


def test_non_positive_amount_is_rejected() -> None:
    with pytest.raises(InvalidPaymentError):
        find_duplicate_groups([pay(1, amount=0)])


def test_count_by_currency() -> None:
    counts = count_by_currency([pay(1), pay(2, currency="EUR"), pay(3, currency="usd")])
    assert counts["USD"] == 2
    assert counts["EUR"] == 1


def test_save_replaces_the_previous_report(connection: sqlite3.Connection) -> None:
    save_duplicate_groups(connection, [[1, 2]])
    written = save_duplicate_groups(connection, [[3, 4], [5, 6, 7]])
    assert written == 5
    assert stored(connection) == [(3, 1), (4, 1), (5, 2), (6, 2), (7, 2)]


def test_failed_save_keeps_the_previous_report(
    connection: sqlite3.Connection,
) -> None:
    save_duplicate_groups(connection, [[1, 2]])
    with pytest.raises(sqlite3.IntegrityError):
        save_duplicate_groups(connection, [[3, 4], [4, 5]])  # payment 4 twice
    assert stored(connection) == [(1, 1), (2, 1)]
```

## Common mistakes

- Inserting the current value into the "seen" map before checking for its partner, so an item pairs with itself.
- Counting each pair twice in a pair-counting problem, by looking at both orders. Count at the later element only.
- Using `frozenset` or `set` as a canonical key when duplicates matter.
- Returning the Boyer-Moore candidate without the verification pass.
- Scanning the dictionary instead of the original list when the order of first occurrence matters.
- Using `defaultdict` and then testing membership with `groups[key]`, which silently creates empty groups.
- Sorting every word to build a key when a fixed alphabet makes a 26-slot count table cheaper.
- Forgetting that `Counter` equality and subtraction treat zero and negative counts specially.
- Using a list as a dictionary key. Convert it to a tuple.
- Choosing a quadratic nested loop for pair problems when a single hash-table pass exists.
- Ignoring normalization (case, spaces, Unicode forms), so equivalent items end up in different groups.

## Practice exercises

1. Given a list of integers and `k`, return the `k` most frequent values using `Counter`. Then solve it with the bucket-by-count method in `O(n)`.
2. Write `is_anagram(first, second)` that runs in `O(n)` using a `Counter`, and a version using a 26-slot list for lowercase letters.
3. Write `count_pairs_with_difference(numbers, k)` that counts pairs `i < j` with `abs(numbers[i] - numbers[j]) == k`. Handle `k == 0` carefully.
4. Write `four_sum_count(a, b, c, d)` for four lists of equal length that returns the number of tuples `(i, j, k, l)` with `a[i] + b[j] + c[k] + d[l] == 0`. Use a dictionary of pair sums to get `O(n^2)`.
5. Write `longest_consecutive(numbers)` that returns the length of the longest run of consecutive integers (such as 4, 5, 6) in an unsorted list, in `O(n)` using a set.
6. Group a list of words by their length, then by their first letter, using `defaultdict` each time.
7. Write `can_form_ransom_note(note, magazine)` that returns whether the letters of `note` can be taken from `magazine`, using each letter at most once.
8. Write `majority_elements_n_over_3(values)` that returns all values appearing more than `n / 3` times, with `O(1)` extra space. Use two candidates.
9. Prove with the pigeonhole principle that among any 27 English words, two start with the same letter. State the number of boxes and items.
10. Write `subarrays_divisible_by_k(values, k)` that counts contiguous subarrays whose sum is divisible by `k` by counting prefix-sum remainders. Explain why Python's modulo behavior for negative numbers helps.
11. Write a canonical key for "same set of words in a sentence regardless of order and case" and group a list of sentences by it.
12. Write `find_missing_and_duplicate(values)` for a list of `n` numbers from `1..n` where one number is missing and one appears twice. Solve it with a `Counter`, then with the sum formulas in `O(1)` extra space.
13. Extend the duplicate-payment detector with a function that returns the customers with the most duplicate groups, using `Counter.most_common`, and write tests for ties.

## Summary

Counting and grouping problems become simple once you reach for a hash table. A `Counter` or dictionary answers frequency questions in one pass. Two Sum works by remembering what has been seen and asking whether the partner has appeared, checking before inserting so that an item never pairs with itself. `defaultdict(list)` plus a canonical key groups equivalent items, and the key must be hashable and distinguish exactly the equivalence you want. Boyer-Moore voting finds a majority candidate in `O(1)` space but needs a verification pass, and the pigeonhole principle tells you when a duplicate must exist. All of these replace nested loops with a single `O(n)` pass at the cost of `O(k)` memory.
