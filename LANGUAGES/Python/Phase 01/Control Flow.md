## Overview

So far your programs have run straight from the first line to the last. **Control flow** is how you change that: choosing between actions, repeating actions, and jumping out of repetitions early. Every real program is built from these few ideas. A web server decides what to do with each request, retries failed operations, and loops over thousands of records.

By the end of this note you will be able to:

- Make decisions with `if`, `elif`, and `else`, and with the one-line conditional expression.
- Match values and shapes of data with `match` and `case`.
- Repeat work with `while` and `for`, and generate sequences with `range`.
- Control loops with `break`, `continue`, and the loop `else` clause.
- Use the walrus operator `:=` where it makes code clearer.

> [!note] Conditions are expressions that become true or false
> Any value can be used as a condition. Chapter 1.3 listed which values are falsy. Everything else is truthy. Remember this chapter's rule of thumb: if the condition reads naturally aloud, it is probably well written.

## Core concepts

### Conditions

A **condition** is an expression whose truthiness decides what happens next. Typical building blocks:

| Test | Example |
| --- | --- |
| Comparison | `age >= 18`, `name == "Ada"` |
| Membership | `"admin" in roles`, `code not in allowed` |
| Identity | `value is None`, `value is not None` |
| Combination | `a and b`, `a or b`, `not a` |
| Truthiness | `if items:`, `if not name:` |

### if, elif, else

```python
if condition_one:
    ...
elif condition_two:
    ...
else:
    ...
```

Python checks the conditions from top to bottom and runs **only the first block whose condition is true**. The `elif` and `else` parts are optional. Because order matters, put the most specific test first.

### Conditional expression

A one-line version that produces a value:

```python
label = "adult" if age >= 18 else "minor"
```

Read it as: the value is `"adult"` if the condition holds, otherwise `"minor"`. It is an expression, so it can be used inside `print()` or an f-string. Do not nest several, because they quickly become unreadable.

### match and case

`match` (Python 3.10 and newer) compares a value against a series of **patterns** and runs the block of the first pattern that fits. It can look inside the data, not just compare it.

| Pattern | Example | Matches |
| --- | --- | --- |
| Literal | `case 404:` | Exactly that value |
| Capture | `case ["go", direction]:` | Binds a name to part of the data |
| Wildcard | `case _:` | Anything, used as the default |
| Guard | `case n if n > 100:` | Pattern plus an extra condition |
| Sequence | `case [x, y]:` | A list or tuple of that shape |
| Mapping | `case {"type": "click"}:` | A dictionary containing those keys |
| Class | `case Point(x=0, y=0):` | An object of that class with those attributes |

An **or-pattern** lists alternatives separated by a vertical bar, as in `case 201 | 202:`, and matches if any one of them fits.

### while

`while condition:` repeats its block as long as the condition stays true. Use it when you do not know in advance how many repetitions you need.

### for and range

`for name in iterable:` visits each item of a sequence, such as a list, a string, or a `range`. `range` generates whole numbers:

| Call | Produces |
| --- | --- |
| `range(5)` | 0, 1, 2, 3, 4 |
| `range(2, 6)` | 2, 3, 4, 5 |
| `range(10, 0, -3)` | 10, 7, 4, 1 |

The stop value is **never included**. The helpers `enumerate` (gives a counter alongside each item) and `zip` (walks several sequences at once) make loops cleaner.

### break, continue, pass

| Keyword | Effect |
| --- | --- |
| `break` | Leave the innermost loop immediately |
| `continue` | Skip to the next iteration of the innermost loop |
| `pass` | Do nothing. A placeholder where a block is required |

### The loop else clause

A `for` or `while` loop may have an `else` block. It runs **only if the loop finished without hitting `break`**. It is the natural way to say "if the search found nothing".

### The walrus operator

`name := expression` assigns a value and also gives back that value, so you can use it in the same line. It is named for its sideways face. It saves repeating a calculation inside a condition.

## Basic syntax examples

### if, elif, else

```python
score = 76

if score >= 90:
    grade = "A"
elif score >= 75:
    grade = "B"
elif score >= 60:
    grade = "C"
else:
    grade = "F"

print(grade)
```

```text
B
```

Even though `76 >= 60` is also true, the `elif` for 60 is never reached because the earlier test already matched.

### Order matters

```python
score = 95

if score >= 60:
    print("pass")
elif score >= 90:
    print("excellent")    # never runs
```

```text
pass
```

Put the strictest condition first.

### Truthiness in conditions

```python
items = []
name = ""

if not items:
    print("no items")
if not name:
    print("no name")
if "admin" in ["user", "admin"]:
    print("is admin")
```

```text
no items
no name
is admin
```

Prefer `if items:` over `if len(items) > 0:`. It is shorter and works for every container.

### Conditional expression

```python
age = 20
status = "adult" if age >= 18 else "minor"
print(f"Status: {status}")
print(f"Ticket: {'child' if age < 12 else 'standard'}")
```

```text
Status: adult
Ticket: standard
```

### Matching commands

```python
def handle(command: str) -> str:
    match command.split():
        case ["quit"]:
            return "Goodbye"
        case ["go", direction]:
            return f"Going {direction}"
        case ["take", *items] if items:
            return f"Taking {', '.join(items)}"
        case []:
            return "Type a command"
        case _:
            return f"Unknown command: {command!r}"


for text in ["quit", "go north", "take sword shield", "take", "", "dance"]:
    print(handle(text))
```

```text
Goodbye
Going north
Taking sword, shield
Unknown command: 'take'
Type a command
Unknown command: 'dance'
```

The pattern `["go", direction]` matches a two-item list whose first item is `"go"`, and captures the second into `direction`. The pattern `*items` captures the remaining words. The guard `if items` rejects an empty capture, which is why plain `take` falls through to the default.

### Matching values with or-patterns and guards

```python
def describe_status(code: int) -> str:
    match code:
        case 200:
            return "OK"
        case 201 | 202:
            return "Accepted"
        case 404:
            return "Not found"
        case c if 500 <= c < 600:
            return "Server error"
        case _:
            return "Other"


print(describe_status(202), describe_status(503), describe_status(301))
```

```text
Accepted Server error Other
```

### Matching dictionaries

```python
event = {"type": "click", "x": 10, "y": 20, "button": "left"}

match event:
    case {"type": "click", "x": x, "y": y}:
        print(f"click at {x},{y}")
    case {"type": "key", "key": key}:
        print(f"key {key}")
    case _:
        print("unknown event")
```

```text
click at 10,20
```

A mapping pattern ignores extra keys, so the `button` entry does not prevent the match.

### while

```python
count = 3
while count > 0:
    print(count)
    count -= 1
print("Liftoff")
```

```text
3
2
1
Liftoff
```

### for and range

```text
>>> list(range(5))
[0, 1, 2, 3, 4]
>>> list(range(2, 6))
[2, 3, 4, 5]
>>> list(range(10, 0, -3))
[10, 7, 4, 1]
>>> list(range(0))
[]
```

```python
total = 0
for number in range(1, 6):
    total += number
print(total)
```

```text
15
```

### enumerate and zip

```python
names = ["Ada", "Grace", "Linus"]
scores = [90, 85, 70]

for position, name in enumerate(names, start=1):
    print(position, name)

for name, score in zip(names, scores):
    print(f"{name}: {score}")
```

```text
1 Ada
2 Grace
3 Linus
Ada: 90
Grace: 85
Linus: 70
```

### break and continue

```python
for n in range(1, 10):
    if n % 2 == 0:
        continue          # skip even numbers
    if n > 7:
        break             # stop completely
    print(n)
```

```text
1
3
5
7
```

### Loop else

```python
numbers = [3, 5, 8, 9]

for n in numbers:
    if n % 2 == 0:
        print(f"first even number: {n}")
        break
else:
    print("no even numbers")
```

```text
first even number: 8
```

If the list had contained only odd numbers, the loop would end naturally and the `else` block would print "no even numbers". A classic use is listing prime numbers:

```python
for n in range(2, 20):
    for divisor in range(2, n):
        if n % divisor == 0:
            break
    else:
        print(n, end=" ")
print()
```

```text
2 3 5 7 11 13 17 19 
```

### Nested loops

```python
for row in range(1, 4):
    for column in range(1, 4):
        print(f"{row * column:3d}", end="")
    print()
```

```text
  1  2  3
  2  4  6
  3  6  9
```

### The walrus operator

```python
data = [4, 8, 15, 16, 23, 42]

if (count := len(data)) > 5:
    print(f"long list with {count} items")
```

```text
long list with 6 items
```

Without `:=`, you would either call `len` twice or write the assignment on a separate line. It is also handy in loops that read until a sentinel value:

```python
while (command := input("> ")) != "quit":
    print(f"you typed {command}")
```

### pass as a placeholder

```python
def not_implemented_yet():
    pass
```

## Deeper behavior

### Truthiness keeps conditions short

`if items:` works for empty lists, strings, dictionaries, and sets. Avoid `if x == True:` and `if x == None:`. Use `if x:` and `if x is None:` (Chapter 1.2).

### Short-circuiting makes safe guards

```python
if user is not None and user.is_active:
    ...
```

The second test runs only if the first passed, so it can never fail on `None`. The same applies to `or`.

### if/elif versus separate ifs

An `if` followed by `elif` chooses **at most one** block. Two separate `if` statements are independent, and both blocks can run. Pick the one that matches your intention.

### `match` is not just a switch statement

A `match` can take data apart and bind names, which a chain of comparisons cannot do conveniently. But it has a rule that surprises people: a **bare name in a pattern is a capture, not a comparison**.

```python
RED = "red"
color = "blue"

match color:
    case RED:
        print("red")
    case "blue":
        print("blue")
```

```text
  File "/home/learner/app.py", line 5
    case RED:
         ^^^
SyntaxError: name capture 'RED' makes remaining patterns unreachable
```

`case RED` does not mean "equal to the variable `RED`". It means "match anything and call it `RED`". To compare against a named constant, use a dotted name (such as an `Enum` member, `Color.RED`) or a literal.

### Class patterns

`match` can also check the type and attributes of objects:

```python
from dataclasses import dataclass


@dataclass
class Point:
    x: int
    y: int


match Point(0, 5):
    case Point(x=0, y=0):
        print("origin")
    case Point(x=0, y=y):
        print(f"on the y axis at {y}")
```

```text
on the y axis at 5
```

### How `for` works

A `for` loop asks the object for an **iterator** and repeatedly requests the next item until none are left. That is why it works on lists, strings, files, dictionaries, `range`, and your own classes. Chapter 7.1 explains the protocol.

### range is lazy

`range(1_000_000)` does not create a million numbers. It stores the start, stop, and step and computes values on demand. It also supports `len()`, indexing, and membership tests.

### Prefer direct iteration over range(len(...))

```python
colors = ["red", "green", "blue"]

# Avoid
for i in range(len(colors)):
    print(colors[i])

# Prefer
for color in colors:
    print(color)

# When you also need the position
for i, color in enumerate(colors):
    print(i, color)
```

### The loop variable stays alive

After a `for` loop, the loop variable still exists and holds the last value. If the loop body never ran, the name may not exist at all, which can cause a `NameError`.

### Do not change a list while looping over it

```python
numbers = [2, 4, 6, 8]
for n in numbers:
    if n % 2 == 0:
        numbers.remove(n)
print(numbers)
```

```text
[4, 8]
```

Removing items shifts the positions, so the loop skips elements. Build a new list instead, or loop over a copy: `for n in numbers[:]:`.

### `break` leaves only the innermost loop

To stop several nested loops at once, move them into a function and use `return`, or use a flag variable.

### Infinite loops

A `while` whose condition never becomes false runs forever. This is sometimes intended, as in `while True:` with a `break`, and sometimes a bug. Press Ctrl and C to interrupt a runaway loop. Watch for loops that depend on floats:

```python
x = 0.0
while x != 1.0:       # never true: 0.1 added ten times is not exactly 1.0
    x += 0.1
```

Compare with `<` and a tolerance instead.

### Off-by-one errors

`range(1, 10)` stops at 9. To include 10, write `range(1, 11)`. Slices follow the same rule (Chapter 1.4). Test the first and last values of every loop.

### The loop else is rarely misread when named well

Many programmers find `else` on a loop confusing. A comment such as `# no break happened: nothing found` makes the intent clear.

## Industry-level snippet

Services that call other services must decide what each response means and whether to try again. This module classifies HTTP status codes with `match`, then retries with exponential backoff: wait a short time after the first failure, twice as long after the second, and so on. The sleeping function is a parameter, so tests can run instantly.

```python
"""Retry policy for an outbound HTTP client."""

import logging
import time
from collections.abc import Callable
from enum import Enum

logger = logging.getLogger(__name__)


class Outcome(Enum):
    """What to do after receiving a response."""

    SUCCESS = "success"
    RETRY = "retry"
    FAIL = "fail"


class RequestFailedError(Exception):
    """Raised when the server rejected the request permanently."""


class RetriesExhaustedError(Exception):
    """Raised when every attempt ended in a retryable failure."""


def classify_status(status: int) -> Outcome:
    """Decide what an HTTP status code means for the caller."""
    match status:
        case 200 | 201 | 202 | 204:
            return Outcome.SUCCESS
        case 408 | 429:
            return Outcome.RETRY
        case code if 500 <= code <= 599:
            return Outcome.RETRY
        case _:
            return Outcome.FAIL


def call_with_retries(
    operation: Callable[[], int],
    max_attempts: int = 4,
    base_delay: float = 0.5,
    sleep: Callable[[float], None] = time.sleep,
) -> int:
    """Call operation until it succeeds, fails permanently, or runs out of attempts.

    Args:
        operation: A function that performs the request and returns a status code.
        max_attempts: Total number of tries, including the first. At least 1.
        base_delay: Seconds to wait after the first failure. Doubles each time.
        sleep: The waiting function. Replace it in tests to avoid real delays.

    Returns:
        The successful status code.

    Raises:
        ValueError: If max_attempts is less than 1.
        RequestFailedError: If the server returns a permanent error status.
        RetriesExhaustedError: If every attempt failed in a retryable way.
    """
    if max_attempts < 1:
        raise ValueError("max_attempts must be at least 1")

    for attempt in range(1, max_attempts + 1):
        try:
            status = operation()
        except ConnectionError as exc:
            logger.warning(
                "Attempt %d/%d could not connect: %s", attempt, max_attempts, exc
            )
        else:
            match classify_status(status):
                case Outcome.SUCCESS:
                    logger.info(
                        "Attempt %d/%d succeeded with %d", attempt, max_attempts, status
                    )
                    return status
                case Outcome.FAIL:
                    raise RequestFailedError(
                        f"server rejected the request with status {status}"
                    )
                case Outcome.RETRY:
                    logger.warning(
                        "Attempt %d/%d got retryable status %d",
                        attempt,
                        max_attempts,
                        status,
                    )

        if attempt < max_attempts:
            delay = base_delay * 2 ** (attempt - 1)
            logger.info("Waiting %.1f seconds before the next attempt", delay)
            sleep(delay)

    raise RetriesExhaustedError(f"gave up after {max_attempts} attempts")
```

A quick demonstration with a fake operation that fails twice and then succeeds:

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")

responses = iter([503, 503, 200])
waits: list[float] = []

status = call_with_retries(lambda: next(responses), sleep=waits.append)
print(status, waits)
```

```text
WARNING Attempt 1/4 got retryable status 503
INFO Waiting 0.5 seconds before the next attempt
WARNING Attempt 2/4 got retryable status 503
INFO Waiting 1.0 seconds before the next attempt
INFO Attempt 3/4 succeeded with 200
200 [0.5, 1.0]
```

The status returned is `200`, and the recorded waits are `0.5` and `1.0` seconds, doubling as designed. No real time passed, because the fake `sleep` just recorded the delays.

What to notice:

- `match` with an or-pattern, a guard, and a wildcard turns a policy table into readable code.
- The `match` on `Outcome` uses dotted names (`Outcome.SUCCESS`), which compare by value instead of capturing.
- `for attempt in range(1, max_attempts + 1)` makes the counting explicit and avoids off-by-one errors.
- The loop never sleeps after the final attempt, because waiting would accomplish nothing.
- A production version would add random **jitter** to the delays so that many clients do not retry at the same instant.

## Common mistakes

1. **Using `=` instead of `==` in a condition.** Assignment is a statement and is not allowed there.
2. **Writing `if x == 1 or 2:`.** It is always true, because `2` alone is truthy. Write `if x == 1 or x == 2:` or `if x in (1, 2):`.
3. **Placing `elif` tests in the wrong order**, so a broad test hides a narrower one.
4. **Forgetting the colon** at the end of `if`, `elif`, `else`, `for`, `while`, or `case` lines.
5. **Off-by-one errors with `range`.** The stop value is excluded.
6. **Modifying a list while looping over it.** Build a new list or loop over a copy.
7. **Infinite `while` loops** where the condition never changes, or depends on float equality.
8. **Using a bare name in `case`** and accidentally capturing instead of comparing.
9. **Using `range(len(items))`** when direct iteration or `enumerate` would do.
10. **Nesting conditional expressions** until nobody can read them.
11. **Expecting `break` to leave all nested loops.** It leaves only the innermost.
12. **Misreading the loop `else`.** It runs when there was no `break`, not when the loop did not run.
13. **Forgetting that the loop variable survives** and reusing it by accident later.

## Practice exercises

1. Write a program that reads a temperature and prints "freezing" below 0, "cold" below 15, "warm" below 25, and "hot" otherwise.
2. Write `is_leap_year(year)` using `if` statements, and test it on 1900, 2000, 2024, and 2100.
3. Rewrite exercise 1 so that the label is assigned with a single conditional expression for just two categories, such as "hot" versus "not hot".
4. Print the numbers from 1 to 20, replacing multiples of 3 with `Fizz`, multiples of 5 with `Buzz`, and multiples of both with `FizzBuzz`.
5. Use a `for` loop and `range` to print a right-angled triangle of stars with 5 rows.
6. Use a `while` loop to find the smallest power of 2 that is greater than 1000, and print the exponent.
7. Write a number-guessing game. The computer picks a number between 1 and 20 (use a fixed value for now), the user guesses, and the program prints "higher" or "lower" until the guess is right, counting the attempts.
8. Search a list of names for the first one that starts with `"G"`, using a `for` loop with `break` and an `else` clause that reports when none was found.
9. Print all prime numbers below 50 using nested loops and the loop `else` clause.
10. Write a function using `match` that takes a point as a tuple and returns "origin", "on the x axis", "on the y axis", or "somewhere else", using sequence patterns.
11. Use `match` with a mapping pattern to handle events such as `{"type": "login", "user": "ada"}` and `{"type": "purchase", "user": "ada", "amount": 20}`, with a default for anything else.
12. The loop below skips elements. Explain why, then fix it in two different ways.

    ```python
    numbers = [2, 4, 6, 8]
    for n in numbers:
        numbers.remove(n)
    ```

13. Extend `call_with_retries` so that it also retries when the operation raises `TimeoutError`, and add jitter to the delay using a function you pass in.

## Summary

- `if`, `elif`, and `else` run the first block whose condition is true. Order the tests from most specific to most general.
- Conditions use comparisons, `in`, `is`, logical operators, and truthiness. `if items:` is the idiomatic way to test for non-empty.
- A conditional expression `a if condition else b` produces a value in one line. Keep it simple.
- `match` and `case` compare values and also take apart sequences, dictionaries, and objects, binding names as they go. A bare name in a pattern captures, it does not compare.
- `while` repeats until a condition fails, and `for` visits every item of an iterable. `range(start, stop, step)` excludes the stop value.
- `break` leaves the innermost loop, `continue` skips to the next iteration, `pass` does nothing, and a loop's `else` runs only if no `break` occurred.
- `enumerate` and `zip` make loops clearer than manual index handling.
- The walrus operator `:=` assigns inside an expression.
- Never modify a list while iterating over it, and watch out for infinite loops and off-by-one mistakes.
