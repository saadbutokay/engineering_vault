## Overview

A data structure is a way of organizing data in memory so that some operations become cheap. An algorithm is a step-by-step procedure that turns input into output. Together they decide whether a feature answers in 5 milliseconds or 5 seconds, whether a nightly job finishes before morning, and whether a service needs one server or fifty.

At work you will rarely write a sorting algorithm or a hash table yourself, because Python already ships good ones. What you will do is choose the right built-in structure, notice when a loop is secretly quadratic, and explain your choice to a teammate. Studying DSA trains exactly that judgment.

This note is about method, not about one specific structure. It teaches the problem-solving loop that every later note relies on, how to trace code by hand, a reusable solution template, a weekly practice routine, how interview problems differ from production problems, and how the notes in this roadmap are organized. The target version is Python 3.12 or newer.

## Core concepts

### Key terms

- Problem: a precise description of an input and the output you must produce.
- Algorithm: a finite list of steps that solves the problem for every valid input.
- Data structure: the container (list, dict, set, heap, tree, graph) the algorithm reads and writes.
- Correctness: the algorithm gives the right answer for every valid input, including the strange ones.
- Efficiency: how time and memory grow as the input grows. The next note measures this properly.

### The problem-solving loop

Use the same eight steps for every problem until they become automatic.

1. Restate. Say the problem in your own words. If you cannot, you do not understand it yet.
2. Examples. Write two or three small examples by hand, including one edge case such as an empty input.
3. Constraints. Ask how large the input can be, whether values can repeat or be negative, and what to return when there is no answer.
4. Brute force. Find any correct solution, however slow. It proves you understand the problem and later becomes a test oracle.
5. Optimize. Ask what work the brute force repeats and which data structure removes that repetition.
6. Code. Translate the chosen idea into clean Python, using the template below.
7. Test. Run your examples, then the edge cases, then a comparison against the brute force.
8. Analyze. State the time and space complexity and explain why.

### Tracing by hand

Tracing means executing the code on paper, one step at a time, writing down every variable after every step. It finds off-by-one errors and wrong assumptions before the interpreter does, and it is the fastest way to understand someone else's solution. A trace is a small table: one row per step, one column per important variable.

### Interview problems and production problems

- Input: interview inputs are clean and described in full. Production inputs are messy, huge, and partly unknown.
- Goal: an interview wants the best complexity in 30 minutes. Production wants correct, readable, maintainable, observable code.
- Questions: interviews let you ask about constraints. Production makes you discover them from logs, metrics, and users.
- Failure: a wrong interview answer costs one question. A wrong production answer can corrupt data or take a service down.
- Shared skill: in both, you choose a structure by what operations you need to be fast.

### How the notes in this roadmap are organized

Every note uses the same nine parts, so you always know where to look.

- Title: the module number and name.
- Overview: what the topic is and why it matters.
- Core concepts: the ideas and vocabulary, with no code required.
- Basic syntax examples: small, readable Python you can type and run.
- Deeper behavior: what actually happens inside CPython and why edge cases behave as they do.
- Industry-level snippet: realistic backend-style code with type hints, error handling, logging, and tests.
- Common mistakes: errors beginners make repeatedly.
- Practice exercises: problems only. Solving them yourself is the point.
- Summary: the short version to review before an interview or exam.

### A weekly practice routine

- Day 1: read one note slowly, then retype every example instead of pasting it. Change a value and predict the output before running.
- Day 2: solve two easy problems with the full loop. Timebox each to 25 minutes.
- Day 3: solve one medium problem. If you are stuck after 25 minutes, read a hint, not the full solution.
- Day 4: finish the note's practice exercises. Write tests for each one.
- Day 5: re-solve one problem from the previous week from scratch, without notes.
- Day 6: review your mistakes log. For each mistake write the lesson in one sentence.
- Day 7: rest. Sleep consolidates learning.

## Basic syntax examples

### Applying the loop to Two Sum

Restate: given a list of integers and a target, return the positions of two different elements whose values add up to the target, or `None` if no such pair exists.

Examples written before any code:

```text
[2, 7, 11, 15], target 9   ->  (0, 1)   because 2 + 7 = 9
[3, 3], target 6           ->  (0, 1)   duplicates are allowed
[1, 2], target 10          ->  None
[], target 5               ->  None     empty input is an edge case
```

Brute force: try every pair.

```python
def two_sum_brute_force(numbers: list[int], target: int) -> tuple[int, int] | None:
    """Try every pair of positions. Time O(n^2), extra space O(1)."""
    for i in range(len(numbers)):
        for j in range(i + 1, len(numbers)):
            if numbers[i] + numbers[j] == target:
                return i, j
    return None
```

Optimize: for each value, the partner we need is `target - value`. Remember the values we have already seen in a dictionary so we can check for the partner in constant time.

```python
def two_sum(numbers: list[int], target: int) -> tuple[int, int] | None:
    """One pass with a dict of value -> index. Time O(n), extra space O(n)."""
    seen: dict[int, int] = {}
    for index, value in enumerate(numbers):
        needed = target - value
        if needed in seen:
            return seen[needed], index
        seen[value] = index
    return None


if __name__ == "__main__":
    assert two_sum([2, 7, 11, 15], 9) == (0, 1)
    assert two_sum([3, 3], 6) == (0, 1)
    assert two_sum([1, 2], 10) is None
    assert two_sum([], 5) is None
    print("all examples pass")
```

### Tracing by hand

Trace `two_sum([2, 7, 11, 15], 9)` before running it.

```text
step  index  value  needed  seen (before)  what happens
1     0      2      7       {}             7 is not in seen, so store seen[2] = 0
2     1      7      2       {2: 0}         2 is in seen, so return (0, 1)
```

Now trace the tricky case `two_sum([3, 3], 6)`.

```text
step  index  value  needed  seen (before)  what happens
1     0      3      3       {}             3 is not in seen yet, store seen[3] = 0
2     1      3      3       {3: 0}         3 is in seen, so return (0, 1)
```

The trace shows why the code checks `seen` before storing the current value: otherwise a lone `3` would pair with itself.

### A reusable solution template

Copy this skeleton for every new problem. The comments are the loop, so you never skip a step.

```python
def solve(values: list[int], target: int) -> int:
    """Restate the problem here in one sentence of your own words.

    Input: what each parameter means and its limits, for example
        0 <= len(values) <= 100_000 and values may contain negatives.
    Output: what is returned, and what is returned when there is no answer.
    """
    # Edge cases first: empty input, single element, impossible target.
    if not values:
        return -1

    # Brute force (kept as a note): try every pair, O(n^2).
    # Optimized idea: one pass with a dict, O(n) time and O(n) space.
    seen: dict[int, int] = {}
    for index, value in enumerate(values):
        if target - value in seen:
            return index
        seen[value] = index
    return -1


if __name__ == "__main__":
    # Your hand-written examples go here, written before the solution.
    assert solve([], 5) == -1
    assert solve([2, 7], 9) == 1
```

## Deeper behavior

### Why brute force comes first

A slow, obviously correct solution is valuable for three reasons. It proves you understand the problem. It gives you something to say in an interview when you are stuck, which is better than silence. And it becomes an oracle: you can run the fast solution and the slow one on thousands of random inputs and compare answers. The testing note builds this habit properly.

### Constraints choose the algorithm

The input size tells you which complexity is allowed. As a planning rule of thumb, pure Python performs tens of millions of very simple operations per second, and real code that does more per step is slower.

- `n` up to about 20: exponential algorithms such as trying every subset can work.
- `n` up to a few thousand: quadratic algorithms are usually acceptable.
- `n` up to about 100,000: you want `n log n` or linear.
- `n` up to about a million or more: you want linear time and careful memory use.

If you skip the constraints question you may spend an hour optimizing code that was already fast enough, or submit a quadratic solution that times out on large input.

### What to do when you are stuck

- Shrink the input to two or three elements and solve it by hand. Then ask what rule you used.
- Draw the data. Arrows between boxes reveal linked structures and graphs immediately.
- Ask which question you keep answering repeatedly. If it is "have I seen this value?", you want a set or dict. If it is "what is the smallest remaining item?", you want a heap. If it is "what is in this range?", you want sorting plus binary search or prefix sums.
- Solve a simpler version first: without duplicates, without negatives, with a sorted input.
- Explain the problem out loud to an imaginary listener. Gaps in understanding become obvious when you try to say them.

### Reading solutions without wasting them

Timebox your own attempt, then read the solution once. Close it, wait a day, and re-solve the problem from a blank file. Only a solution you can reproduce from scratch counts as learned. Keep a mistakes log with one line per mistake, for example "forgot the empty list" or "off by one in the loop bound". Patterns in that log show you what to practice.

### "Accepted" is not "understood"

A solution that passes a judge can still be wrong for inputs the judge did not include, and you may not know why it works. Before moving on, state the complexity, name the invariant that makes the algorithm correct, and list the edge cases you tested.

### Interview thinking versus production thinking

In production the same structure choice appears as a different question. Instead of "return the first duplicate", the requirement is "ignore webhook events we have already processed". The algorithm is the same, a set of seen identifiers, but now you also ask about persistence, concurrency, memory growth, and what happens when the process restarts. The industry snippet below shows that translation.

## Industry-level snippet

A payment provider can deliver the same webhook event more than once. The service must store each event exactly once, reject malformed events, and never leave a batch half written. The problem-solving loop applies directly: the brute force compares every event with every other event, which is `O(n^2)`, and a set of seen identifiers makes it `O(n)`.

```python
# webhook_events.py
import logging
import sqlite3
from collections.abc import Iterable
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS webhook_event (
    event_id TEXT PRIMARY KEY,
    payload TEXT NOT NULL CHECK (length(payload) <= 100)
)
"""

INSERT_SQL = (
    "INSERT INTO webhook_event (event_id, payload) VALUES (?, ?) "
    "ON CONFLICT (event_id) DO NOTHING"
)


class InvalidEventError(ValueError):
    """Raised when an incoming webhook event is malformed."""


@dataclass(frozen=True, slots=True)
class WebhookEvent:
    event_id: str
    payload: str


def dedupe_events(events: Iterable[WebhookEvent]) -> list[WebhookEvent]:
    """Keep the first occurrence of each event_id, preserving arrival order.

    Comparing every event with every kept event would be O(n^2). A set of seen
    identifiers makes each check O(1) on average, so the whole pass is O(n).
    """
    seen: set[str] = set()
    unique: list[WebhookEvent] = []
    for event in events:
        if not event.event_id:
            raise InvalidEventError("event_id must not be empty")
        if event.event_id in seen:
            continue
        seen.add(event.event_id)
        unique.append(event)
    return unique


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def store_events(connection: sqlite3.Connection, events: Iterable[WebhookEvent]) -> int:
    """Store a batch atomically and return how many events were new."""
    unique = dedupe_events(events)
    rows = [(event.event_id, event.payload) for event in unique]
    try:
        with connection:  # commits on success, rolls back if an exception escapes
            before = connection.total_changes
            connection.executemany(INSERT_SQL, rows)
            inserted = connection.total_changes - before
    except sqlite3.Error:
        logger.exception("failed to store a batch of %d webhook events", len(rows))
        raise
    logger.info("stored %d new events out of %d unique", inserted, len(rows))
    return inserted
```

```python
# test_webhook_events.py
import sqlite3
from collections.abc import Iterator

import pytest

from webhook_events import (
    InvalidEventError,
    WebhookEvent,
    create_schema,
    dedupe_events,
    store_events,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def count_rows(connection: sqlite3.Connection) -> int:
    row = connection.execute("SELECT COUNT(*) FROM webhook_event").fetchone()
    return int(row[0])


def test_dedupe_keeps_first_occurrence_in_order() -> None:
    events = [WebhookEvent("a", "1"), WebhookEvent("b", "2"), WebhookEvent("a", "3")]
    assert dedupe_events(events) == [WebhookEvent("a", "1"), WebhookEvent("b", "2")]


def test_dedupe_rejects_empty_event_id() -> None:
    with pytest.raises(InvalidEventError):
        dedupe_events([WebhookEvent("", "payload")])


def test_store_events_is_idempotent(connection: sqlite3.Connection) -> None:
    batch = [WebhookEvent("a", "1"), WebhookEvent("b", "2"), WebhookEvent("a", "1")]
    assert store_events(connection, batch) == 2
    assert store_events(connection, batch) == 0
    assert count_rows(connection) == 2


def test_failed_batch_is_rolled_back_completely(
    connection: sqlite3.Connection,
) -> None:
    batch = [WebhookEvent("a", "ok"), WebhookEvent("b", "x" * 101)]
    with pytest.raises(sqlite3.IntegrityError):
        store_events(connection, batch)
    assert count_rows(connection) == 0
```

Run the tests with `pytest -q`. The last test proves the transaction works: the first row was valid, yet it does not survive because the second row violated the `CHECK` constraint.

## Common mistakes

- Coding before understanding. Ten minutes of examples and constraints saves an hour of debugging the wrong idea.
- Skipping the edge cases. Empty input, a single element, and all-equal values break more solutions than any clever input does.
- Never asking about constraints. Without `n` you cannot know whether `O(n^2)` is acceptable.
- Jumping straight to the optimal trick. If you cannot explain why it works, you have memorized it, not learned it.
- Not tracing. Reading code and believing it works is not the same as running it by hand.
- Reading solutions too early. Struggle for a fixed time first, because the struggle is where the learning happens.
- Never re-solving. A problem you solved once and never revisited is mostly forgotten within weeks.
- Treating a passing sample as proof. Samples are tiny. Your own edge cases and a brute-force comparison are the real check.
- Forgetting to state complexity. An answer without time and space cost is incomplete.
- Comparing your pace to other people's. Progress is measured against your own mistakes log.

## Practice exercises

1. Restate in one sentence, write three examples (including an empty input), and list your clarifying questions for: "Given a list of user emails, report whether any email appears twice."
2. Write a brute-force function and an optimized function for "return the first value that appears twice in a list, or `None`". State the complexity of each.
3. Trace `two_sum([3, 2, 4], 6)` by hand in a table and write down the contents of `seen` after every step. Then explain why the answer is `(1, 2)` and not `(0, 0)`.
4. Trace the following function on `[5, 3, 8, 1]` and write the value of `best` after each loop iteration. Then describe in one sentence what the function computes.

   ```python
   def mystery(values: list[int]) -> int:
       best = values[0]
       for value in values[1:]:
           if value > best:
               best = value
       return best
   ```

5. For each input size, say which complexity classes are realistic in Python and justify briefly: `n = 15`, `n = 3_000`, `n = 200_000`, `n = 5_000_000`.
6. Write the restate, examples, and constraints steps (no code) for "return the longest word in a sentence". List at least three questions you would ask, such as how ties and punctuation are handled.
7. Write only steps 1 to 4 of the loop (restate, examples, constraints, brute force) for "check whether a string of brackets is balanced". Do not write the optimized solution yet.
8. Copy the solution template into a file and fill it in for "count how many numbers in a list are greater than the average". Include at least four assert-based examples.
9. A product team asks for "the 10 most viewed pages from yesterday's access log". Write five clarifying questions a production engineer should ask that an interview version of the problem would not include.
10. Modify `store_events` so it returns the list of event identifiers that were actually new instead of a count. Update the tests accordingly.
11. Design your own four-week practice schedule using the weekly routine above. Decide which days you study, how long each session lasts, and when you review mistakes.
12. Pick one problem you have solved in the past and re-solve it from a blank file without looking at any code. Record how long it took and which step of the loop you skipped.

## Summary

DSA is the skill of choosing structures and procedures that keep programs fast, correct, and understandable as data grows. The eight-step loop of restating, examples, constraints, brute force, optimize, code, test, and analyze turns an intimidating problem into a sequence of small decisions. Tracing by hand finds mistakes before the interpreter does, and a brute-force solution doubles as a test oracle. Constraints tell you which complexity is acceptable. Practice works best when it is regular, timeboxed, reviewed, and repeated. Interview and production problems share the same core ideas, but production adds persistence, concurrency, logging, and failure handling.
