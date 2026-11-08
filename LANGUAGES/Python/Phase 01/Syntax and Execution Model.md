## Overview

Every programming language has **syntax**, the rules for how code must be written, and an **execution model**, the rules for how that code is actually run. Python's syntax is deliberately small and readable. Its execution model is simple once you see it: Python reads your file, checks it, translates it into a lower-level form, and then carries it out one statement at a time, from top to bottom.

This chapter gives you the vocabulary and mental model that every later chapter builds on.

By the end of this note you will be able to:

- Use indentation correctly to group code into blocks.
- Write comments and tell them apart from docstrings.
- Explain the difference between a statement and an expression.
- Split long lines of code safely.
- Describe the path your code takes from source text to running program.

> [!note] If you are new to programming
> Do not try to memorize every term at once. Run the examples, change them, break them, and read the error messages. Understanding grows from experiments.

## Core concepts

### Python is case-sensitive

`name`, `Name`, and `NAME` are three different things. Keywords such as `if`, `for`, and `True` must be written exactly as shown. `true` is not a Python keyword, and `If` is not either.

### Indentation creates blocks

Many languages use braces to group code. Python uses **indentation**, the whitespace at the start of a line. A line that ends with a colon (`:`) introduces a **block**, and every line in that block must be indented by the same amount.

| Construct | Introduces a block |
| --- | --- |
| `if`, `elif`, `else` | Conditional code |
| `for`, `while` | Loop bodies |
| `def` | A function body |
| `class` | A class body |
| `try`, `except`, `finally` | Error handling |
| `with` | A managed resource |
| `match`, `case` | Pattern matching |

The standard is **4 spaces per level**. Never mix tabs and spaces in one file. Configure your editor to insert spaces when you press Tab (Chapter 0.4 and Chapter 0.7).

### Comments

A **comment** is text that Python ignores. It starts with `#` and runs to the end of the line. Python has no special syntax for multi-line comments, you simply start each line with `#`.

A **docstring** looks like a comment but is different: it is a string literal placed as the first statement of a module, function, or class. Python keeps it and tools can read it. Comments are for people reading the source. Docstrings are for people using the code.

### Statements and expressions

This distinction explains many beginner errors.

| Idea | Expression | Statement |
| --- | --- | --- |
| What it does | Produces a value | Performs an action |
| Can it appear inside `print(...)`? | Yes | No |
| Examples | `3 + 4`, `len(name)`, `age >= 18`, `"a" if ok else "b"` | `x = 5`, `if ...:`, `for ...:`, `import os`, `return x`, `def f():`, `pass` |

A quick test: if you can put it inside the parentheses of `print()` and it makes sense, it is an expression. A function call on its own line, such as `print("hi")`, is an expression used as a statement.

Assignment (`x = 5`) is a statement, so it does not produce a value. The one exception is the **walrus operator** `:=`, an assignment expression introduced in Chapter 1.6.

### Line continuation

A statement normally ends at the end of the line. When a statement is too long, you have two options.

1. **Implicit continuation.** Inside `()`, `[]`, or `{}` you may break the line anywhere. This is the recommended way.
2. **Explicit continuation.** A backslash `\` at the very end of a line joins it to the next. It is fragile, because a trailing space after the backslash breaks it, so prefer brackets.

You can also put several statements on one line with a semicolon, but this is discouraged. One statement per line is easier to read and debug.

### How Python runs code

Python executes a file **from top to bottom**, one statement at a time. Two details surprise beginners:

- A `def` statement does not run the function. It creates a function object and attaches it to a name. The body runs only when the function is called.
- A name must be defined by the time the line using it runs. Order in the file matters for code that runs immediately.

Before any line runs, Python does preparation work. The standard implementation, **CPython**, performs these stages:

```text
Source code (.py text)
   -> tokens        the text is split into words, numbers, symbols
   -> syntax tree   the tokens are checked against the grammar (SyntaxError happens here)
   -> bytecode      the tree is translated into compact instructions
   -> execution     the Python virtual machine runs the bytecode, line by line
```

Because the first three stages happen before execution begins, a syntax error anywhere in a file stops the entire program before it prints anything.

## Basic syntax examples

### Indentation and blocks

```python
temperature = 31

if temperature >= 30:
    print("It is hot")
    print("Drink water")
else:
    print("It is comfortable")

print("This line always runs")
```

```text
It is hot
Drink water
This line always runs
```

The last `print` is not indented, so it is outside the `if` statement.

### Nested blocks

```python
for number in range(1, 4):
    if number % 2 == 0:
        print(number, "is even")
    else:
        print(number, "is odd")
```

```text
1 is odd
2 is even
3 is odd
```

Each new block adds four more spaces.

### What indentation errors look like

Missing indentation after a colon:

```python
def greet():
print("hi")
```

```text
  File "/home/learner/app.py", line 2
    print("hi")
    ^
IndentationError: expected an indented block after function definition on line 1
```

Indentation where none is expected:

```python
x = 1
    y = 2
```

```text
  File "/home/learner/app.py", line 2
    y = 2
IndentationError: unexpected indent
```

### Comments

```python
# This is a full-line comment
price = 19.99  # an inline comment, with two spaces before the #

# There is no multi-line comment syntax.
# Use one # per line instead.
```

### Docstring versus comment

```python
def area(width, height):
    """Return the area of a rectangle."""
    # Multiply the sides. (A comment, ignored by Python.)
    return width * height


print(area.__doc__)
```

```text
Return the area of a rectangle.
```

### Expressions and statements

```text
>>> 3 + 4
7
>>> len("hello")
5
>>> 10 > 3
True
>>> "yes" if 10 > 3 else "no"
'yes'
```

Each line above is an expression, so the REPL shows its value. Assignment is a statement, so nothing is shown:

```text
>>> x = 5
>>> x
5
```

Because assignment is a statement, you cannot nest it inside another expression. Writing `y = (x = 5)` is a `SyntaxError`.

### Splitting a long line

Preferred, using brackets:

```python
total_price = (
    base_price
    + shipping_cost
    + handling_fee
    - discount
)

message = (
    "Your order has been received. "
    "We will email you when it ships."
)

colors = [
    "red",
    "green",
    "blue",
]
```

Adjacent string literals inside brackets are joined automatically, which is handy for long messages.

Explicit backslash (works, but avoid it):

```python
total = 1 + 2 + \
    3 + 4
```

### Execution order

```python
print("1. starting")


def say_hello():
    print("3. inside the function")


print("2. function defined, not yet called")
say_hello()
print("4. done")
```

```text
1. starting
2. function defined, not yet called
3. inside the function
4. done
```

Using a function before its `def` line has run fails:

```python
print(square(3))


def square(n):
    return n * n
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 1, in <module>
    print(square(3))
          ^^^^^^
NameError: name 'square' is not defined
```

### A syntax error stops everything

```python
print("start")
print("end"
```

```text
  File "/home/learner/app.py", line 2
    print("end"
         ^
SyntaxError: '(' was never closed
```

Notice that `start` was never printed. Python found the problem while checking the file, before running any line.

## Deeper behavior

### Seeing the syntax tree

Python turns your code into a tree that reflects its structure. You can print it with the `ast` module:

```python
import ast

print(ast.dump(ast.parse("x = 1 + 2")))
```

```text
Module(body=[Assign(targets=[Name(id='x', ctx=Store())], value=BinOp(left=Constant(value=1), op=Add(), right=Constant(value=2)))], type_ignores=[])
```

Read it from the inside out: the numbers 1 and 2 are constants, they are combined by an addition, and the result is assigned to the name `x`.

### Seeing the bytecode

The `dis` module shows the instructions the virtual machine executes:

```python
import dis


def add(a, b):
    return a + b


dis.dis(add)
```

```text
  3           0 RESUME                   0

  4           2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 RETURN_VALUE
```

This output is from Python 3.12. Other versions print slightly different instructions, because bytecode is an internal detail that changes between releases. The idea is what matters: load two values, add them, return the result.

### Why `__pycache__` appears

When one file imports another, Python saves the compiled bytecode in a `__pycache__` folder so the next import can skip compilation.

```text
__pycache__/
    helper.cpython-312.pyc
```

The name records the Python version that produced it. The file you run directly is compiled every time and is not cached. These folders are safe to delete and belong in `.gitignore`.

### Python the language versus CPython

"Python" is a language specification. **CPython** is the standard implementation, written in C, and it is what you installed. Other implementations exist, such as PyPy, which uses a just-in-time compiler. The execution model described here is CPython's.

### Compile-time errors versus run-time errors

| Kind | When it is found | Example | Effect |
| --- | --- | --- | --- |
| Syntax error | Before anything runs | Missing colon, unclosed bracket | Nothing in the file runs |
| Runtime error (exception) | While the program runs | Dividing by zero, undefined name | Program stops at that line, earlier lines already ran |
| Logic error | Never reported automatically | Using `+` instead of `-` | Program runs and gives a wrong answer |

Chapter 1.7 teaches you to read these messages.

### Names are looked up when the line runs

Python does not check that a name exists when it reads your file. It checks when the line executes. A function body that mentions an undefined name is not an error until the function is called.

### Imports run code

`import helper` executes `helper.py` from top to bottom the first time it is imported, then stores the result so later imports reuse it. This is why files that are meant to be imported should keep their top level light. Chapter 4.1 covers the `__name__ == "__main__"` guard that separates reusable code from script behavior.

### Whitespace inside a line does not matter, at the start of a line it does

`x=1+2` and `x = 1 + 2` mean the same thing, but spacing around operators is a style rule (Chapter 0.7). Only leading whitespace changes the meaning.

## Industry-level snippet

Backend teams constantly write scripts that read logs. This one summarizes a web server access log. It shows the structure of a well-organized Python file: a module docstring, imports, constants, small functions, implicit line continuation inside brackets, error handling, logging, and a `main` function guarded so it only runs when the file is executed.

```python
"""Summarize a web server access log.

Each line has five space-separated fields:

    2026-03-01T12:00:00Z GET /api/orders 200 123ms

Usage:
    python log_summary.py ACCESS_LOG_PATH
"""

import logging
import sys
from dataclasses import dataclass, field
from pathlib import Path

logger = logging.getLogger("log_summary")

SLOW_THRESHOLD_MS = 500
FIELD_COUNT = 5

EXIT_OK = 0
EXIT_USAGE = 2
EXIT_IO_ERROR = 3


@dataclass
class Summary:
    """Counters collected while reading the log."""

    total: int = 0
    malformed: int = 0
    by_status_class: dict[str, int] = field(default_factory=dict)
    slow_requests: list[str] = field(default_factory=list)


def parse_line(line: str) -> tuple[str, int, int]:
    """Return (path, status, duration_ms) for one log line.

    Raises:
        ValueError: If the line does not have the expected shape.
    """
    parts = line.split()
    if len(parts) != FIELD_COUNT:
        raise ValueError(f"expected {FIELD_COUNT} fields, found {len(parts)}")
    _timestamp, _method, path, status_text, duration_text = parts
    if not duration_text.endswith("ms"):
        raise ValueError(f"duration must end with 'ms': {duration_text!r}")
    return path, int(status_text), int(duration_text.removesuffix("ms"))


def summarize(lines: list[str]) -> Summary:
    """Count requests by status class and collect slow requests."""
    summary = Summary()
    for number, raw_line in enumerate(lines, start=1):
        line = raw_line.strip()
        if not line:
            continue
        try:
            path, status, duration_ms = parse_line(line)
        except ValueError as exc:
            summary.malformed += 1
            logger.warning("Line %d skipped: %s", number, exc)
            continue

        summary.total += 1
        status_class = f"{status // 100}xx"
        summary.by_status_class[status_class] = (
            summary.by_status_class.get(status_class, 0) + 1
        )
        if duration_ms >= SLOW_THRESHOLD_MS:
            summary.slow_requests.append(f"{path} took {duration_ms} ms")
    return summary


def main(argv: list[str]) -> int:
    """Read the log named on the command line and print a summary."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    if len(argv) != 2:
        logger.error("Usage: python log_summary.py ACCESS_LOG_PATH")
        return EXIT_USAGE

    try:
        lines = Path(argv[1]).read_text(encoding="utf-8").splitlines()
    except (OSError, UnicodeDecodeError) as exc:
        logger.error("Cannot read %s: %s", argv[1], exc)
        return EXIT_IO_ERROR

    summary = summarize(lines)
    print(f"requests: {summary.total}, malformed: {summary.malformed}")
    for status_class in sorted(summary.by_status_class):
        print(f"  {status_class}: {summary.by_status_class[status_class]}")
    for entry in summary.slow_requests:
        print(f"  slow: {entry}")
    return EXIT_OK


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Create a small `access.log` to try it:

```text
2026-03-01T12:00:00Z GET /api/orders 200 123ms
2026-03-01T12:00:01Z POST /api/orders 201 640ms
2026-03-01T12:00:02Z GET /api/users 404 12ms
this line is broken
2026-03-01T12:00:03Z GET /api/orders 503 1500ms
```

```bash
python log_summary.py access.log
```

```text
WARNING Line 4 skipped: expected 5 fields, found 4
requests: 4, malformed: 1
  2xx: 2
  4xx: 1
  5xx: 1
  slow: /api/orders took 640 ms
  slow: /api/orders took 1500 ms
```

Observe the execution model in action: the imports, constants, class, and function definitions all run first and only create objects. Nothing is calculated until the last two lines call `main`. The `if __name__ == "__main__"` guard means importing this file from a test does not start the program.

## Common mistakes

1. **Mixing tabs and spaces.** The code looks aligned but Python disagrees. Use 4 spaces everywhere.
2. **Forgetting the colon** after `if`, `for`, `while`, `def`, `class`, or `else`.
3. **Inconsistent indentation inside a block.** Three spaces on one line and four on the next causes an `IndentationError`.
4. **Using a function before it is defined** in code that runs immediately.
5. **Reading the line number as the exact location of the problem.** For unclosed brackets or missing quotes, the real cause can be on an earlier line than the one reported.
6. **Writing `=` where `==` is meant** inside a condition. Assignment is a statement and is not allowed there.
7. **Ending a line with a backslash followed by a space.** The continuation silently fails. Use brackets instead.
8. **Treating a triple-quoted string as a comment.** It is a real string. It is harmless in some places, but it is not ignored the way `#` comments are.
9. **Writing several statements on one line with semicolons.** It hurts readability and hides mistakes.
10. **Copying code from documents that use curly quotes or non-breaking spaces.** Python rejects characters that look right but are not.

## Practice exercises

1. Write a program with a variable `score`. Print "Pass" and "Well done" if the score is at least 50, otherwise print "Try again". Ensure both messages for the passing case sit inside the same block.
2. Take this code and fix its indentation so it prints the numbers 1 to 3, each followed by whether it is odd or even:

   ```python
   for n in range(1, 4):
   if n % 2 == 0:
   print(n, "even")
   else:
   print(n, "odd")
   ```

3. Write a docstring for a function `square(n)` that returns `n * n`, and print it using `__doc__`. Then add a comment inside the function that explains why, not what.
4. For each of the following, decide whether it is an expression or a statement, and justify your answer: `10 / 4`, `total = 10`, `import math`, `max(3, 9)`, `if x > 3:`, `"a" + "b"`.
5. Rewrite this long line using brackets so that no line is longer than 60 characters:
   `report = "Name: " + name + ", Age: " + str(age) + ", City: " + city + ", Country: " + country`
6. Create a script that prints four numbered lines, defines a function in the middle, and calls it at the end. Predict the output before you run it.
7. Make a file with a syntax error on the last line and a `print("hello")` on the first line. Run it and explain why `hello` does not appear.
8. Use `ast.dump(ast.parse(...))` on the expression `2 * (3 + 4)`. Describe in words how the tree reflects the order of operations.
9. Use `dis.dis` on a function that returns `a * b + c`. List the order of the operations the virtual machine performs.
10. Create two files, `helper.py` and `main.py`, where `main.py` imports `helper`. Run `main.py` and find the `__pycache__` folder. Explain what is inside, and why there is no `.pyc` file for `main.py`.
11. Extend the log summary from this note so it also reports the slowest request overall, and handles a log with no valid lines without crashing.

## Summary

- Python is case-sensitive and uses **indentation**, normally 4 spaces, to define blocks that start after a colon.
- Comments start with `#`. Docstrings are string literals that document modules, functions, and classes.
- An **expression** produces a value, a **statement** performs an action. Assignment is a statement.
- Split long lines inside brackets. Avoid the backslash and avoid semicolons.
- Python runs code top to bottom. A `def` creates a function without running it.
- CPython turns source into a syntax tree, then bytecode, then executes it. Syntax errors are found before any line runs, runtime errors are found while running.
- `__pycache__` holds cached bytecode for imported modules, and is safe to delete and ignore in Git.
- Well-structured files separate definitions from the code that starts the program using `main()` and the `__name__` guard.
