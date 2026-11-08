## Overview

Every programmer, from a first-week student to a veteran engineer, spends a large part of every day looking at error messages. The difference between a frustrated beginner and an effective developer is mostly one skill: **reading the error calmly and extracting what it says**. Python's error messages are some of the best in any language, and they tell you what went wrong, where, and often how to fix it.

By the end of this note you will be able to:

- Read a traceback from the bottom up and find the line that matters.
- Recognize the six most common error types and the typical cause of each.
- Tell syntax errors, runtime errors, and logic errors apart.
- Follow a systematic process to debug any failing script.
- Write code that reports problems in a clear and useful way.

> [!note] Errors are information, not insults
> An error message is the interpreter pointing at the exact place where your instructions stopped making sense. Beginners often read it last. Experienced developers read it first.

## Core concepts

### Three kinds of problems

| Kind | When it appears | Example | What you see |
| --- | --- | --- | --- |
| Syntax error | Before the program runs | Missing colon or bracket | `SyntaxError` or `IndentationError`, nothing runs |
| Runtime error (exception) | While the program runs | Dividing by zero, missing key | A traceback, and the program stops at that line |
| Logic error | The program runs to the end | Using `+` instead of `-` | No message, only a wrong result |

Syntax and runtime errors announce themselves. Logic errors do not, and finding them takes testing and careful thinking.

### Anatomy of a traceback

When an exception is not handled, Python prints a **traceback**: the chain of function calls that led to the problem. Here is a small program with a bug:

```python
def load_prices():
    return {"apple": "1.20", "pear": "abc"}


def total_cost(items):
    prices = load_prices()
    total = 0.0
    for item in items:
        total += float(prices[item])
    return total


def main():
    print(total_cost(["apple", "pear"]))


main()
```

And what Python prints:

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 17, in <module>
    main()
  File "/home/learner/app.py", line 14, in main
    print(total_cost(["apple", "pear"]))
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/learner/app.py", line 9, in total_cost
    total += float(prices[item])
             ^^^^^^^^^^^^^^^^^^^
ValueError: could not convert string to float: 'abc'
```

Reading it:

| Part | Meaning |
| --- | --- |
| `Traceback (most recent call last):` | The header. Calls are listed oldest first, newest last |
| `File "...", line 17, in <module>` | The file, the line number, and which function that line was in. `<module>` means the top level of the file |
| The indented code line | The source text of that line |
| The `^^^^` markers | The exact part of the line that was being evaluated |
| `ValueError: could not convert...` | **The error type and message. This is the most important line** |

Each frame shows one function call. The program started at line 17, called `main`, which called `total_cost`, where the failure happened.

### How to read it

1. **Start at the last line.** The error type tells you the category of problem, and the message gives the details.
2. **Look at the last frame in your own code** (the one just above the error line). That is usually where to look first.
3. **Read upward** through the frames to understand how the program got there.

When a library is involved, the frames closest to the error may belong to the library. Scan upward until you find a file that is yours.

### The common errors

| Error | Typical message | Usual cause | First thing to try |
| --- | --- | --- | --- |
| `SyntaxError` | `expected ':'`, `'(' was never closed` | Missing colon, quote, or bracket | Look at the reported line and the one before it |
| `IndentationError` | `expected an indented block` | Wrong or missing indentation | Align the block with 4 spaces |
| `NameError` | `name 'x' is not defined` | Typo, or used before assignment or import | Check spelling and order of lines |
| `TypeError` | `can only concatenate str (not "int") to str` | Wrong type for an operation | Print `type(value)` for the values involved |
| `ValueError` | `invalid literal for int()` | Right type, unacceptable value | Print `repr(value)` and validate input |
| `IndexError` | `list index out of range` | Position beyond the end | Check `len()` and loop bounds |
| `KeyError` | `'email'` | Dictionary key does not exist | Use `.get()` or check with `in` |
| `AttributeError` | `'NoneType' object has no attribute 'strip'` | Attribute does not exist, often because a value is `None` | Find where the `None` came from |
| `ZeroDivisionError` | `division by zero` | Divisor was zero | Check the denominator before dividing |
| `ModuleNotFoundError` | `No module named 'requestz'` | Typo, or package not installed in this environment | Check spelling and the active virtual environment |
| `FileNotFoundError` | `No such file or directory` | Wrong path or wrong working directory | Print `Path.cwd()` and the full path |
| `UnboundLocalError` | `cannot access local variable` | Assigning to a name inside a function before it has a value | Check `global` and scope (Chapter 3.3) |

### A systematic way to debug

1. **Read the last line** of the traceback aloud. What kind of error is it, and what does the message say?
2. **Go to the line** it points at, in your own code.
3. **Inspect the actual values.** Print them with `repr()`, or use the debugger (Chapter 0.4). Do not assume you know what they are.
4. **Reduce the problem.** Remove code until you have the smallest example that still fails. This is called a **minimal reproducible example**.
5. **Form one hypothesis** about the cause and change **one thing** to test it.
6. **Verify the fix**, and run the program again from the start.
7. **Understand why** it happened, and add a test or a check so it cannot return.

## Basic syntax examples

All tracebacks below were produced by Python 3.12. Newer versions may color the output or word a message slightly differently, but the structure is identical.

### SyntaxError: missing colon

```python
age = 20
if age >= 18
    print("adult")
```

```text
  File "/home/learner/app.py", line 2
    if age >= 18
                ^
SyntaxError: expected ':'
```

Python shows the line and points at the place where it noticed the problem. Syntax errors have no "traceback" header because the program never started.

### SyntaxError: bracket never closed

```python
total = (1 + 2
print(total)
```

```text
  File "/home/learner/app.py", line 1
    total = (1 + 2
            ^
SyntaxError: '(' was never closed
```

Modern Python points back to the unclosed bracket itself, even though the parser only discovered the problem later.

### IndentationError

```python
print("a")
if True:
print("b")
```

```text
  File "/home/learner/app.py", line 3
    print("b")
    ^
IndentationError: expected an indented block after 'if' statement on line 2
```

### NameError

```python
message = "hi"
print(mesage)
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(mesage)
          ^^^^^^
NameError: name 'mesage' is not defined. Did you mean: 'message'?
```

Python noticed a similar name and suggested it. Use such hints.

### TypeError: wrong types

```python
age = 30
print("Age: " + age)
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print("Age: " + age)
          ~~~~~~~~^~~~~
TypeError: can only concatenate str (not "int") to str
```

The `~` markers show the left operand and the `^` marks the operator. Convert with `str(age)` or use an f-string.

### TypeError: wrong number of arguments

```python
def add(a, b):
    return a + b


add(1)
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 5, in <module>
    add(1)
TypeError: add() missing 1 required positional argument: 'b'
```

### TypeError: calling something that is not a function

```python
sum = 10
print(sum([1, 2]))
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(sum([1, 2]))
          ^^^^^^^^^^^
TypeError: 'int' object is not callable
```

The earlier line replaced the built-in `sum` with the number `10`. This is why you should never reuse built-in names.

### TypeError: cannot subscript

```python
x = 5
print(x[0])
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(x[0])
          ~^^^
TypeError: 'int' object is not subscriptable
```

### ValueError

```python
quantity = int("twelve")
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 1, in <module>
    quantity = int("twelve")
               ^^^^^^^^^^^^^
ValueError: invalid literal for int() with base 10: 'twelve'
```

The type (`str`) is fine for `int()`, but this particular value cannot be converted.

### IndexError

```python
colors = ["red", "green"]
print(colors[2])
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(colors[2])
          ~~~~~~^^^
IndexError: list index out of range
```

A list of two items has positions 0 and 1.

### KeyError

```python
user = {"name": "Ada"}
print(user["email"])
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(user["email"])
          ~~~~^^^^^^^^^
KeyError: 'email'
```

The message is just the missing key. Use `user.get("email")` when the key may be absent.

### AttributeError

```python
name = None
print(name.strip())
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(name.strip())
          ^^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'strip'
```

This is one of the most common real-world errors. The real question is always: **why is the value `None`?** Look at where it was assigned.

### ZeroDivisionError

```python
items = 0
print(10 / items)
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(10 / items)
          ~~~^~~~~~~
ZeroDivisionError: division by zero
```

### UnboundLocalError

```python
counter = 0


def bump():
    counter += 1


bump()
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 6, in <module>
    bump()
  File "/home/learner/app.py", line 4, in bump
    counter += 1
    ^^^^^^^
UnboundLocalError: cannot access local variable 'counter' where it is not associated with a value
```

Because the function assigns to `counter`, Python treats it as a local name, which has no value yet. Chapter 3.3 explains scope.

### ModuleNotFoundError and FileNotFoundError

```python
import requestz
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 1, in <module>
    import requestz
ModuleNotFoundError: No module named 'requestz'
```

```python
open("missing.txt")
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 1, in <module>
    open("missing.txt")
FileNotFoundError: [Errno 2] No such file or directory: 'missing.txt'
```

For a missing module, check the spelling and whether you installed the package in the **active** virtual environment (Chapter 0.5). For a missing file, check the working directory (Chapter 0.1).

### Walk-through: debugging the price calculator

Return to the traceback in the Core concepts section.

1. **Last line:** `ValueError: could not convert string to float: 'abc'`. So `float()` was given a string that is not a number.
2. **Last frame in our code:** line 9 of `total_cost`: `total += float(prices[item])`.
3. **Inspect values.** Temporarily add a line before it:

   ```python
   print(repr(item), repr(prices[item]))
   ```

   It prints `'pear' 'abc'` right before the crash.
4. **Cause:** the data returned by `load_prices()` contains a bad price for `pear`.
5. **Fix:** decide what the program should do. Validate data where it enters the program, report which item is wrong, and do not let a bad value travel deep into the calculation.

The traceback told us everything except one thing it could not know: what the correct behavior should be.

## Deeper behavior

### Where syntax errors point

The parser reports the place where it **noticed** the problem, which is not always where you made it. A missing closing bracket on line 10 may be reported on line 11 or 12. When the reported line looks fine, look at the line above. For an unclosed bracket, modern Python usually points back at the opening one, as shown earlier.

### Chained exceptions

Sometimes one error causes another, and Python shows both:

```python
def read_setting(config, key):
    try:
        return config[key]
    except KeyError as exc:
        raise ValueError(f"missing setting: {key}") from exc


read_setting({}, "port")
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 3, in read_setting
    return config[key]
           ~~~~~~^^^^^
KeyError: 'port'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/learner/app.py", line 8, in <module>
    read_setting({}, "port")
  File "/home/learner/app.py", line 5, in read_setting
    raise ValueError(f"missing setting: {key}") from exc
ValueError: missing setting: port
```

Read the **last** traceback first (the final error is the one your program ended with), then read upward to the original cause. If you see "During handling of the above exception, another exception occurred", an error happened inside an error handler, which often points to a bug in the handler itself.

### The exception family tree

Exceptions form a hierarchy, and handlers can catch a whole family at once.

```text
BaseException
  KeyboardInterrupt
  SystemExit
  Exception
    ArithmeticError
      ZeroDivisionError
    AttributeError
    ImportError
      ModuleNotFoundError
    LookupError
      IndexError
      KeyError
    NameError
      UnboundLocalError
    OSError
      FileNotFoundError
      ConnectionError
    SyntaxError
      IndentationError
    TypeError
    ValueError
      UnicodeError
```

Your own programs normally handle subclasses of `Exception`. `KeyboardInterrupt` (Ctrl and C) and `SystemExit` deliberately sit outside it. Chapter 5.1 covers handling in detail.

### The error line is a message to a person, not just a label

Messages often include the exact values involved: the missing key, the invalid text, the unexpected type. When you write your own errors (Chapter 5.2), do the same.

### Warnings are not errors

Python also prints **warnings**, for example `SyntaxWarning` or `DeprecationWarning`. The program continues. Do not ignore them. They are early notice that something will break in a future version or is probably a mistake.

### Tools for looking inside a failing program

| Tool | Use |
| --- | --- |
| `print(repr(value))` | Fast, reveals hidden characters and types |
| `type(value)` | Confirm what kind of object you really have |
| `breakpoint()` | Pause execution and open the debugger at that line |
| The editor debugger | Breakpoints, stepping, and a variables panel (Chapter 0.4) |
| `python -i script.py` | Run the script, then stay in the REPL with all variables available |
| `python -m pdb script.py` | Run under the command-line debugger (Chapter 11.4) |

```python
def total_cost(items):
    prices = load_prices()
    breakpoint()          # execution pauses here and you can inspect prices
    ...
```

### Handling errors still needs the traceback

A program that catches an exception and prints only "something went wrong" throws away the best clue. When you handle an unexpected error, record the traceback:

```python
import logging
import traceback

logger = logging.getLogger(__name__)

try:
    int("x")
except ValueError:
    traceback.print_exc()                        # print it
    logger.exception("could not convert input")  # or log it, with the traceback
```

### Logic errors need different tools

When the program runs but gives the wrong answer, there is no traceback. Reproduce the problem with a small, known input. Work out by hand what the right answer should be. Then find the first place where the program's intermediate values differ from yours. Automated tests (Chapter 11.1) capture these checks permanently.

### Search effectively

Copy the last line of the error, remove anything specific to your files or values, and search for it. Add the library name if the traceback runs through one. Official documentation and well-voted answers usually explain the cause. Be careful when pasting your own code or data into public sites, because it may contain secrets.

### Explain it aloud

"Rubber duck debugging" means describing the problem line by line to an object or a colleague. The act of explaining often exposes the wrong assumption before anyone answers.

## Industry-level snippet

A backend service receives untrusted data and must reject bad input without crashing, while still telling the difference between a bad request (the client's problem) and a bug (your problem). This module validates an order payload, raises errors whose messages are safe and specific, and uses `logger.exception` to record a full traceback only for the unexpected failures.

```python
"""Validate incoming order payloads and report problems clearly."""

import json
import logging
import sys

logger = logging.getLogger("order_intake")

REQUIRED_FIELDS = ("order_id", "customer_email", "quantity")

EXIT_OK = 0
EXIT_REJECTED = 1
EXIT_UNEXPECTED = 70


class PayloadError(ValueError):
    """Raised when an order payload is malformed.

    The message is written so that it is safe to show to the client.
    """


def parse_order(raw: str) -> dict[str, object]:
    """Parse and validate a JSON order.

    Args:
        raw: The request body as text.

    Returns:
        The order as a dictionary with all required fields present.

    Raises:
        PayloadError: If the body is not JSON, not an object, is missing a
            required field, or has an invalid quantity.
    """
    try:
        data = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise PayloadError(
            f"body is not valid JSON: {exc.msg} (line {exc.lineno}, column {exc.colno})"
        ) from exc

    if not isinstance(data, dict):
        raise PayloadError(f"body must be a JSON object, got {type(data).__name__}")

    missing = [name for name in REQUIRED_FIELDS if name not in data]
    if missing:
        raise PayloadError(f"missing required fields: {', '.join(missing)}")

    quantity = data["quantity"]
    # bool is a subtype of int in Python, so exclude it explicitly.
    if isinstance(quantity, bool) or not isinstance(quantity, int) or quantity < 1:
        raise PayloadError(
            f"quantity must be a positive whole number, got {quantity!r}"
        )
    return data


def process(raw: str) -> int:
    """Validate one payload and return a process exit code."""
    try:
        order = parse_order(raw)
    except PayloadError as exc:
        # An expected problem with the input: a short message is enough.
        logger.error("Rejected payload: %s", exc)
        return EXIT_REJECTED
    except Exception:
        # Anything else is a bug in our own code. Keep the full traceback.
        logger.exception("Unexpected error while parsing an order")
        return EXIT_UNEXPECTED

    logger.info(
        "Accepted order %s with quantity %s", order["order_id"], order["quantity"]
    )
    return EXIT_OK


def main() -> int:
    """Read one payload from standard input and process it."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    return process(sys.stdin.read())


if __name__ == "__main__":
    sys.exit(main())
```

Feed it some payloads and read the messages. Each one is a different failure, and each message names the problem precisely:

```python
payloads = [
    '{"order_id": "A1", "customer_email": "ada@example.com", "quantity": 2}',
    '{"order_id": "A1", "quantity": 0}',
    '{"order_id": "A1", "customer_email": "ada@example.com", "quantity": "two"}',
    "not json",
    "[1, 2]",
]
for payload in payloads:
    process(payload)
```

```text
INFO Accepted order A1 with quantity 2
ERROR Rejected payload: missing required fields: customer_email
ERROR Rejected payload: quantity must be a positive whole number, got 'two'
ERROR Rejected payload: body is not valid JSON: Expecting value (line 1, column 1)
ERROR Rejected payload: body must be a JSON object, got list
```

Why it is written this way:

- Validation happens at the boundary, so the rest of the system can trust the data.
- Error messages state **what** was wrong and **which** value caused it, which saves everyone time.
- `raise ... from exc` keeps the original parsing error attached for later diagnosis.
- Expected errors (`PayloadError`) get one clear log line. Unexpected errors get `logger.exception`, which writes the complete traceback. A broad `except Exception` is acceptable only at a boundary like this one, where the program must report the problem and exit cleanly.
- Distinct exit codes let automation tell a rejected input from a crash. `70` is the conventional code for an internal software error.
- The messages never include the whole payload, because request bodies may contain personal data.

## Common mistakes

1. **Reading the first line of the traceback instead of the last.** The error type and message are at the bottom.
2. **Panicking at the length of a traceback.** Most of it is the call chain. Find the last frame in your own code.
3. **Trusting the reported line number blindly** for syntax errors. Check the line above it.
4. **Changing several things at once** while debugging. You will not know which change mattered.
5. **Guessing instead of looking.** Print the real values with `repr()`, or use a breakpoint.
6. **Catching an error and hiding it**, for example `except: pass`. The problem still exists, and you have removed the evidence.
7. **Printing "an error occurred" without the details.** Always include the message and, for unexpected errors, the traceback.
8. **Ignoring warnings.** They usually become errors later.
9. **Fixing the symptom, not the cause.** For `AttributeError: 'NoneType'...`, do not wrap the call in a check. Find out why the value was `None`.
10. **Reusing built-in names** such as `sum`, `list`, or `input`, then getting `'int' object is not callable`.
11. **Debugging in the wrong environment.** A `ModuleNotFoundError` often means the package is installed in a different virtual environment than the one running.
12. **Pasting secrets into public forums** when asking for help. Remove tokens, passwords, and personal data first.

## Practice exercises

1. For each message, say which kind of error it is, what is probably wrong, and what you would check first: `NameError: name 'totl' is not defined`, `IndexError: list index out of range`, `KeyError: 'id'`, `TypeError: 'NoneType' object is not subscriptable`.
2. Write a three-line program that raises each of these on purpose, one at a time: `ZeroDivisionError`, `ValueError`, `IndexError`, `KeyError`, `TypeError`. For each, read the traceback and identify the exact part of the line that is marked.
3. Diagnose and fix this program, and write down your reasoning in the seven steps from this note:

   ```python
   def average(numbers):
       total = 0
       for n in numbers:
           total += n
       return total / len(numbers)

   print(average([]))
   ```

4. Predict what this program prints when the user types `10`. It does not crash. Explain why, and then fix it so it calculates the real total:

   ```python
   price = input("Price: ")
   quantity = 3
   print("Total:", price * quantity)
   ```

5. The following code has a syntax error. Find it by reading only the error message, then fix it:

   ```python
   def greet(name):
       message = "Hello, " + name
       print(message
   greet("Ada")
   ```

6. Fix the `UnboundLocalError` in the `bump` example from this note in two different ways. Explain which one you prefer and why.
7. Write a function that reads a number from a string. Use `try` and `except ValueError` to print a friendly message that includes the bad text, and return `None`. Then explain why returning `None` may cause an `AttributeError` or `TypeError` later in the program.
8. Create a program with three functions that call each other, where the deepest one raises an error. Read the traceback and list the functions in the order they were called.
9. Reproduce a chained exception: catch a `KeyError` and raise a `ValueError` from it. Then do it again without `from exc`, and compare the two tracebacks.
10. Use `breakpoint()` inside a loop in a small program. At the pause, print three variables, step forward twice, and continue.
11. Write down the minimal reproducible example for this bug report: "my report script crashes when a customer has no phone number." Include the smallest data and code that fail.
12. Extend the order validator so that it also rejects an email address without an `@`, and an `order_id` that is not a non-empty string. Write five payloads that exercise every rule, and keep the error messages specific.

## Summary

- Python reports three kinds of problems: syntax errors (before running), exceptions (while running), and logic errors (silent wrong answers).
- A traceback lists the call chain from oldest to newest. **Read the last line first**, then the last frame that belongs to your code.
- The markers `^` and `~` under a line show which part of it failed.
- Learn the common errors: `SyntaxError`, `IndentationError`, `NameError`, `TypeError`, `ValueError`, `IndexError`, `KeyError`, `AttributeError`, `ZeroDivisionError`, `ModuleNotFoundError`, `FileNotFoundError`, and `UnboundLocalError`.
- Debug systematically: read the error, go to the line, inspect real values, reduce the problem, change one thing, verify, and prevent a repeat.
- Syntax errors point to where the parser noticed the problem, so check the previous line too.
- Chained tracebacks show a cause and an effect. Read the final error first, then the original cause above it.
- Write code that fails informatively: validate at the boundary, use specific messages, preserve the original cause, and log full tracebacks for unexpected errors.
- Never silence errors, and never ignore warnings.
