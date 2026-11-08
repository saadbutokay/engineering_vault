## Overview

A program that cannot receive data or show results is not very useful. **Output** means sending information out of your program: to the screen, to a file, to another program. **Input** means bringing information in: from the keyboard, from a file, from another program's output.

This chapter covers `print()` and `input()` in depth, the three standard streams every program has, and the skill of making a script work well both when a person types at it and when another program feeds data into it.

By the end of this note you will be able to:

- Control the output of `print()` with its parameters.
- Read typed input with `input()` and convert it to the right type.
- Format neat tables and reports.
- Read from standard input in several ways.
- Send normal output and error messages to different places.
- Explain why `print()` is for program output and `logging` is for diagnostics.

> [!note] A mental model
> Every running program is connected to three channels: **standard input** (where data comes in, usually the keyboard), **standard output** (where results go, usually the screen), and **standard error** (where problems are reported, also usually the screen). The shell can redirect each one independently.

## Core concepts

### print()

`print()` converts each of its arguments to text with `str()`, joins them, and writes the result. Its full signature is:

```python
print(*objects, sep=" ", end="\n", file=sys.stdout, flush=False)
```

| Parameter | Meaning | Default |
| --- | --- | --- |
| `*objects` | Any number of values to print | none |
| `sep` | Text placed between values | a space |
| `end` | Text placed after the last value | a new line |
| `file` | Where to write | standard output |
| `flush` | Force the text out immediately | `False` |

`print()` always returns `None`.

### input()

`input(prompt)` shows the prompt, waits for the user to type a line and press Enter, and returns what was typed as a **string**, without the trailing newline. It is always a string, even if the user typed digits.

### Type conversion

Because `input()` returns text, convert it before doing arithmetic:

| Conversion | Result |
| --- | --- |
| `int("42")` | `42` |
| `float("3.5")` | `3.5` |
| `int("abc")` | `ValueError` |
| `int("3.5")` | `ValueError` |

Remember `.strip()` removes accidental spaces around what the user typed. (`int` and `float` already ignore surrounding whitespace, but other code might not.)

### Formatted output

Use f-strings with width and alignment specifications (Chapter 1.4) to build tidy columns, and use the `repr()` of a value when you need to see its exact contents, including hidden whitespace.

### Standard streams

| Stream | Python object | Purpose |
| --- | --- | --- |
| Standard input | `sys.stdin` | Data coming in |
| Standard output | `sys.stdout` | The program's results |
| Standard error | `sys.stderr` | Warnings and errors |

Keeping results on standard output and problems on standard error is a long-standing convention. It lets someone save the results to a file while still seeing the problems on screen.

### Redirection and pipes

The shell can reconnect the streams:

- `command > file.txt` sends standard output to a file, replacing it.
- `command >> file.txt` appends to the file.
- `command < input.txt` reads standard input from a file (bash and zsh only).
- `command 2> errors.txt` sends standard error to a file.
- `command1 | command2` sends the output of the first command into the input of the second. This is called a **pipe**.

> [!warning] PowerShell is different
> PowerShell does not support `<`. To feed a file into a program, use `Get-Content input.txt | python script.py`. PowerShell also feeds each item of a list as one line, so `1,2,3 | python script.py` sends three lines.

### Reading from standard input

| Technique | Use it when |
| --- | --- |
| `input()` | You want one line, with an optional prompt |
| `sys.stdin.readline()` | You want one line, with the newline kept |
| `for line in sys.stdin:` | You want to process every line as it arrives |
| `sys.stdin.read()` | You want the entire input as one string |

When there is no more input, `input()` raises `EOFError`, and `for line in sys.stdin` simply ends. A person signals the end of input from the keyboard with Ctrl and D (macOS, Linux) or Ctrl and Z then Enter (Windows).

## Basic syntax examples

### Printing

```python
print("Hello")
print("a", "b", "c")
print("a", "b", "c", sep="-")
print("loading", end="...")
print("done")
print()
print("Total:", 42)
print(*[1, 2, 3], sep=", ")
```

```text
Hello
a b c
a-b-c
loading...done

Total: 42
1, 2, 3
```

`print()` with no arguments prints an empty line. Putting `*` before a list passes its items as separate arguments.

### print converts values to text

```python
print(3.0, True, None, [1, "a"])
```

```text
3.0 True None [1, 'a']
```

### Printing to standard error and to a file

```python
import sys

print("This is a normal result")
print("This is a warning", file=sys.stderr)

with open("report.txt", "w", encoding="utf-8") as report:
    print("Report", file=report)
    print("rows:", 3, file=report)
```

On screen, both lines appear. If you run `python script.py > out.txt`, only the normal result goes into `out.txt` and the warning still shows on screen. The `with` statement closes the file automatically (Chapter 5.3).

### Reading typed input

```python
name = input("Your name: ")
age = int(input("Your age: "))
print(f"Hello, {name}! Next year you will be {age + 1}.")
```

A session looks like this (the text after each colon is typed by the user):

```text
Your name: Ada
Your age: 36
Hello, Ada! Next year you will be 37.
```

### The most common input bug

```python
age = input("Your age: ")
print(age + 1)
```

```text
Traceback (most recent call last):
  File "/home/learner/app.py", line 2, in <module>
    print(age + 1)
          ~~~~^~~
TypeError: can only concatenate str (not "int") to str
```

`age` is the string `"36"`. Convert it with `int(age)`.

### Invalid text

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

### Asking again until the answer is valid

```python
while True:
    text = input("Enter a whole number: ")
    try:
        number = int(text)
    except ValueError:
        print("That is not a whole number, try again.")
    else:
        break

print(f"You entered {number}")
```

Here `try`, `except`, and `else` catch the error instead of crashing (Chapter 5.1 teaches them properly), and `while True` repeats until `break` runs (Chapter 1.6).

### Formatted tables

```python
items = [("Notebook", 3, 4.5), ("Pen", 12, 1.25), ("Backpack", 1, 39.9)]

print(f"{'Item':<12}{'Qty':>5}{'Price':>10}")
print("-" * 27)
for name, qty, price in items:
    print(f"{name:<12}{qty:>5}{price:>10.2f}")
```

```text
Item          Qty     Price
---------------------------
Notebook        3      4.50
Pen            12      1.25
Backpack        1     39.90
```

Text is aligned left (`<`), numbers right (`>`), and the width reserves space for each column.

### Reading a stream of lines

Save this as `sum_numbers.py`:

```python
import sys

total = 0
for line in sys.stdin:
    total += int(line)
print(total)
```

Feed it data from a pipe or a file:

```bash
printf "1\n2\n3\n" | python3 sum_numbers.py
python3 sum_numbers.py < numbers.txt
```

```powershell
1,2,3 | python sum_numbers.py
Get-Content numbers.txt | python sum_numbers.py
```

```text
6
```

### Writing without print

```python
import sys

sys.stdout.write("no newline is added")
sys.stdout.write("\n")
```

`sys.stdout.write` accepts only strings and does not add a newline.

### Hidden characters

```python
text = input("Type something and press Enter: ")
print(text)
print(repr(text))
```

If you type two trailing spaces, `print(text)` hides them, while `repr` reveals them inside quotes.

### Passwords

```python
from getpass import getpass

password = getpass("Password: ")
```

`getpass` does not show the typed characters. Never print a password or write it to a log.

## Deeper behavior

### Buffering

Writing to the screen character by character would be slow, so Python collects output in a **buffer** and writes it in chunks. When output goes to a terminal, the buffer is emptied at every newline. When output goes to a file or a pipe, it is emptied only when the buffer fills up or the program ends. As a result, output from a long-running script may seem to appear late.

Ways to control this:

- `print(..., flush=True)` forces one call out immediately.
- `python -u script.py` runs the whole script unbuffered.
- The environment variable `PYTHONUNBUFFERED=1` does the same, and is common in containers.

Standard error is flushed promptly so that messages about problems appear in order.

### Results and diagnostics are different things

A well-behaved command-line tool writes **only its results** to standard output, and everything else (progress notes, warnings, errors) to standard error. That makes the tool composable: you can pipe its output into another program without having to filter messages out.

### `print` versus `logging`

`print` is fine for the output a program exists to produce. For diagnostic messages in anything that will run unattended, use the `logging` module (Chapter 5.7). Logging adds levels (`DEBUG`, `INFO`, `WARNING`, `ERROR`), timestamps, destinations, and the ability to turn detail up or down without editing code. By default, `logging` writes to standard error, which matches the convention above.

### Programs that adapt to their environment

When someone runs your script by hand, prompting is friendly. When a program or scheduler runs it, a prompt would hang forever. You can detect the difference:

```python
import sys

if sys.stdin.isatty():
    name = input("Your name: ")
else:
    name = sys.stdin.readline().strip()
```

`isatty()` is true when input comes from an interactive terminal.

### End of input and interruption

```python
try:
    answer = input("Continue? ")
except EOFError:
    print("No input available")
except KeyboardInterrupt:
    print("\nCancelled by the user")
```

`EOFError` occurs when input is closed. `KeyboardInterrupt` occurs when the user presses Ctrl and C. Well-written tools handle both and exit cleanly.

### Text encoding of output

Output is turned into bytes using an encoding. Modern macOS and Linux terminals use UTF-8. Some older Windows consoles and redirected output may use another encoding, which leads to an error naming `charmap` or `cp1252` when you print characters it cannot represent. Fixes include setting `PYTHONUTF8=1`, running `python -X utf8 script.py`, or calling `sys.stdout.reconfigure(encoding="utf-8")` at the start of your program. Be aware that Windows PowerShell 5.1 writes redirected output (`>`) in UTF-16 by default, which can confuse other programs. PowerShell 7 uses UTF-8.

### `print` returns None

```python
result = print("hi")
print(result)
```

```text
hi
None
```

A frequent beginner bug is writing `total = print(a + b)` and then wondering why `total` is empty. `print` shows a value, it does not return it.

### Every `input()` call is text, even in loops

Parse at the boundary. Convert and validate user text once, as soon as it arrives, so the rest of your program works with proper types.

### Services rarely use input()

A web service or background worker has no keyboard. Real backend programs receive data from command-line arguments, environment variables, files, network requests, and message queues. The skills in this chapter matter for scripts, tools, data pipelines, and for understanding how every program talks to its surroundings.

## Industry-level snippet

Operations teams write small filters that read data from standard input and print a clean report, so they can be chained in a shell pipeline. This tool reads `user,amount` lines, skips and reports malformed ones on standard error, and prints a sorted summary to standard output. Its exit code tells automation whether any lines were rejected.

```python
"""Summarize a stream of "user,amount" lines read from standard input.

Usage:
    cat payments.csv | python ledger_report.py

Exit codes:
    0  every line was valid
    1  some lines were skipped because they were malformed
    2  no valid lines were found
"""

import logging
import sys
from collections.abc import Iterable
from decimal import Decimal, InvalidOperation

logger = logging.getLogger("ledger_report")

ZERO = Decimal("0")

EXIT_OK = 0
EXIT_SKIPPED_LINES = 1
EXIT_NO_DATA = 2


def parse_line(line: str) -> tuple[str, Decimal]:
    """Split one input line into a user name and an amount.

    Raises:
        ValueError: If the line is not in the form "user,amount".
    """
    user, separator, amount_text = line.partition(",")
    user = user.strip()
    amount_text = amount_text.strip()
    if not separator or not user:
        raise ValueError("expected the form 'user,amount'")
    try:
        amount = Decimal(amount_text)
    except InvalidOperation as exc:
        raise ValueError(f"amount is not a number: {amount_text!r}") from exc
    if not amount.is_finite():
        raise ValueError(f"amount must be finite: {amount_text!r}")
    return user, amount


def summarize(lines: Iterable[str]) -> tuple[dict[str, Decimal], int]:
    """Add up amounts per user and count the lines that had to be skipped."""
    totals: dict[str, Decimal] = {}
    skipped = 0
    for number, raw_line in enumerate(lines, start=1):
        line = raw_line.strip()
        if not line or line.startswith("#"):
            continue  # ignore blank lines and comments
        try:
            user, amount = parse_line(line)
        except ValueError as exc:
            logger.warning("line %d skipped: %s", number, exc)
            skipped += 1
            continue
        totals[user] = totals.get(user, ZERO) + amount
    return totals, skipped


def print_report(totals: dict[str, Decimal]) -> None:
    """Print a table of totals, largest first, to standard output."""
    width = max(len(user) for user in [*totals, "USER"])
    print(f"{'USER':<{width}}  {'TOTAL':>12}")
    print("-" * (width + 14))
    for user, total in sorted(totals.items(), key=lambda item: item[1], reverse=True):
        print(f"{user:<{width}}  {total:>12,.2f}")
    grand_total = sum(totals.values(), ZERO)
    print(f"{'ALL':<{width}}  {grand_total:>12,.2f}")


def main() -> int:
    """Read standard input, print the report, and return an exit code."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    try:
        totals, skipped = summarize(sys.stdin)
    except KeyboardInterrupt:
        logger.error("Interrupted")
        return 130
    if not totals:
        logger.error("no valid input lines")
        return EXIT_NO_DATA
    print_report(totals)
    return EXIT_SKIPPED_LINES if skipped else EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Try it with this sample, saved as `payments.csv`:

```text
# user,amount
ada,120.50
grace,99.99
ada,30.25
broken line
linus,abc
grace,1000
```

```bash
python3 ledger_report.py < payments.csv
```

```text
WARNING line 5 skipped: expected the form 'user,amount'
WARNING line 6 skipped: amount is not a number: 'abc'
USER          TOTAL
-------------------
grace      1,099.99
ada          150.75
ALL        1,250.74
```

The exit code of this run is `1`, because two lines were skipped.

Notice what happened on screen: the report came from standard output, the two warnings from standard error. Run it again with `> report.txt` and only the report goes into the file.

An interactive helper for tools that do talk to a person follows the same discipline: validate, limit retries, and treat a closed input stream as a clean failure rather than a crash.

```python
import sys


class InputAbortedError(Exception):
    """Raised when the user cannot or will not provide a valid answer."""


def prompt_int(prompt: str, minimum: int, maximum: int, max_attempts: int = 3) -> int:
    """Ask for a whole number in a range, with a limited number of retries."""
    for attempt in range(1, max_attempts + 1):
        try:
            text = input(prompt)
        except EOFError as exc:
            raise InputAbortedError("no more input is available") from exc
        try:
            value = int(text.strip())
        except ValueError:
            print(
                f"Please enter a whole number ({attempt}/{max_attempts}).",
                file=sys.stderr,
            )
            continue
        if minimum <= value <= maximum:
            return value
        print(f"Please enter a number from {minimum} to {maximum}.", file=sys.stderr)
    raise InputAbortedError(f"no valid answer after {max_attempts} attempts")
```

Points to notice in both programs:

- Parsing and validation are separated from printing, so each function can be tested without a keyboard.
- Messages about problems never mix with the data on standard output.
- A function receives any iterable of lines, so tests can pass a list instead of real standard input.
- Exit codes report the outcome to the shell, and `130` is the conventional code for interruption by Ctrl and C.
- `Decimal` keeps money exact, and `is_finite` blocks the strings `NaN` and `Infinity`.

## Common mistakes

1. **Forgetting that `input()` returns a string.** Convert with `int()` or `float()` before doing arithmetic.
2. **Assuming the user will type valid data.** Always handle `ValueError` and offer another chance.
3. **Using `print` for diagnostics in long-running programs.** Use `logging`.
4. **Printing errors to standard output.** Send them to `sys.stderr` so results stay clean.
5. **Writing `x = print(...)`.** `print` returns `None`.
6. **Forgetting that `input()` blocks.** A script waiting for input inside a scheduled job will hang forever.
7. **Using `<` in PowerShell.** Pipe with `Get-Content file | python script.py` instead.
8. **Expecting output to appear instantly when piped.** It is buffered. Use `flush=True` or `python -u`.
9. **Calling `sys.stdin.read()` on an interactive terminal** and wondering why nothing happens. It waits for the end of input (Ctrl and D, or Ctrl and Z then Enter).
10. **Printing secrets.** Passwords, tokens, and personal data do not belong on screen or in logs.
11. **Not handling `EOFError` and `KeyboardInterrupt`** in interactive tools, so users see ugly tracebacks.
12. **Forgetting `end=""` when printing progress on one line**, or forgetting `flush=True` so the progress never shows.

## Practice exercises

1. Print the numbers 1 to 5 on a single line, separated by commas and ending with an exclamation mark instead of a new line.
2. Write a program that asks for a first name and a last name and prints `Last, First` using an f-string.
3. Ask the user for two numbers and print their sum, difference, product, and quotient with two decimal places. Handle the case where the second number is zero.
4. Write a program that keeps asking for a number between 1 and 10 until the user provides one, and prints how many attempts it took.
5. Print the multiplication table for a number the user chooses, formatted so that all the equals signs line up.
6. Create a receipt printer that takes a list of `(name, quantity, price)` tuples and prints aligned columns plus a total line with a thousands separator.
7. Write `shout.py`, which reads lines from standard input and prints each one in uppercase. Run it with a pipe in your shell and with redirection from a file (or the PowerShell equivalent).
8. Write a script that prints a normal message to standard output and a warning to standard error. Run it with `>` redirection and observe what appears on screen and what goes into the file.
9. Modify `sum_numbers.py` so that non-numeric lines are reported to standard error with their line number and skipped, and the exit code is `1` if anything was skipped.
10. Write a script that behaves differently when run interactively and when its input is piped, using `isatty()`.
11. Run the ledger report with an empty input and with an input made only of malformed lines. Explain the exit codes.
12. Extend the ledger report with a command-line option that limits the table to the top three users, and write a test that calls `summarize` with a plain list of strings.

## Summary

- `print()` accepts any number of values and the options `sep`, `end`, `file`, and `flush`. It returns `None`.
- `input()` returns a string. Convert and validate it as soon as it arrives.
- Use f-strings with width and alignment to produce readable tables, and use `repr()` to reveal hidden characters.
- Every program has standard input, standard output, and standard error. Results go to standard output, problems go to standard error.
- Redirection (`>`, `>>`, `<`, `2>`) and pipes (`|`) connect programs. PowerShell needs `Get-Content ... | command` instead of `<`.
- Read standard input with `input()`, `sys.stdin.readline()`, `for line in sys.stdin`, or `sys.stdin.read()`.
- Output is buffered when it is not going to a terminal. Use `flush=True` or `python -u` when you need it immediately.
- Use `print` for the program's results and `logging` for diagnostics. Handle `EOFError` and `KeyboardInterrupt`, and never print secrets.
