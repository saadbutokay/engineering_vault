## Overview

Python gives you several ways to run code, and each one fits a different purpose. You will use the interactive REPL to experiment, script files to build real programs, and `python -m` to run tools that ship with Python or with packages. You will also learn how to pass information into a program from the command line and how a program tells the shell whether it succeeded.

By the end of this note you will be able to:

- Start and exit the Python REPL on any operating system.
- Write a `.py` file and run it from the terminal.
- Run a one-liner with `-c` and a module with `-m`.
- Read command-line arguments inside a script.
- Explain the difference between `python`, `python3`, and `py`.

> [!note] Command names in this note
> Examples use `python`. On macOS and Linux, type `python3` instead unless you are inside an activated virtual environment (Chapter 0.5), where `python` works everywhere.

## Core concepts

### The REPL

**REPL** stands for Read, Eval, Print, Loop. You type one piece of code, Python reads it, evaluates it, prints the result, and waits for the next. It is the fastest way to test an idea.

Start it by typing `python` with no arguments. You will see a prompt of three greater-than signs:

```text
Python 3.13.5 (main, ...) on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

### Script files

A **script** is a text file ending in `.py` that contains Python code. Running it executes the code from top to bottom and then the process ends. Scripts are what you will write for real work, because they can be saved, edited, versioned, and shared.

### The four ways to start Python

| Form | Meaning | Typical use |
| --- | --- | --- |
| `python` | Start the interactive REPL | Experimenting |
| `python script.py` | Run a file | Real programs |
| `python -c "code"` | Run a string of code once | Quick one-liners and automation |
| `python -m module` | Run a module by name | Tools such as `venv`, `pip`, `http.server` |

### Command-line arguments

Anything you type after the script name is passed to the program as a list of text values. Inside Python this list is `sys.argv`. Item 0 is the script name, item 1 is the first argument, and so on. Every value arrives as text, even if it looks like a number.

### Exiting

| Situation | How to leave |
| --- | --- |
| In the REPL, any system | Type `exit()` and press Enter |
| In the REPL on macOS and Linux | Press Ctrl and D |
| In the REPL on Windows | Press Ctrl and Z, then Enter |
| A running script is stuck or you want to stop it | Press Ctrl and C |

### python, python3, and py

| Command | Where it exists | Notes |
| --- | --- | --- |
| `python` | Windows always. macOS and Linux only inside a virtual environment, or if you set it up | May be missing or old on macOS and Linux |
| `python3` | macOS and Linux | Standard name for Python 3 |
| `py` | Windows | Launcher or install manager, lets you choose a version such as `py -3.12` |

## Basic syntax examples

### Using the REPL

```text
>>> 2 + 3
5
>>> name = "Ada"
>>> "Hello, " + name
'Hello, Ada'
>>> len(name)
3
>>> exit()
```

Notice that the REPL prints the value of an expression automatically. A script does not, so inside a script you must call `print()` to see output.

Useful REPL helpers:

```text
>>> help(len)       # read documentation
>>> dir(str)        # list what a string can do
>>> _               # the previous result
```

> [!tip] A better REPL in newer versions
> Python 3.13 and newer ship an improved interactive shell with colors, multi-line editing, and a paste mode (press F3). If your REPL looks plainer, you are on an older version.

### Writing and running a script

Create a file named `hello.py` in a folder you can find again:

```python
print("Hello from a script")
print(2 + 3)
```

Run it from the same folder:

```bash
python hello.py
```

```text
Hello from a script
5
```

If the file is somewhere else, give a path:

```bash
python projects/hello.py
```

### One-liners with -c

```bash
python -c "print(10 * 7)"
python -c "import sys; print(sys.version_info[:2])"
```

On Windows PowerShell and on macOS and Linux, wrapping the code in double quotes works. If the code itself contains double quotes, use single quotes inside it.

### Running modules with -m

```bash
python -m pip --version
python -m venv .venv
python -m http.server 8000
python -m json.tool data.json
python -m timeit "sum(range(1000))"
```

Each one runs a program that is part of Python or an installed package. `http.server` starts a small web server for the current folder. Press Ctrl and C to stop it.

### Command-line arguments

Create `greet.py`:

```python
import sys

print("Script name:", sys.argv[0])
print("All arguments:", sys.argv[1:])

name = sys.argv[1]
print(f"Hello, {name}!")
```

Run it:

```bash
python greet.py Ada
```

```text
Script name: greet.py
All arguments: ['Ada']
Hello, Ada!
```

Now run it without an argument and read the error:

```bash
python greet.py
```

```text
Traceback (most recent call last):
  File "greet.py", line 6, in <module>
    name = sys.argv[1]
           ~~~~~~~~^^^
IndexError: list index out of range
```

That error is correct: there is no item at position 1 because you passed nothing. The industry snippet below shows how to handle this properly.

### Running a script and then staying in the REPL

```bash
python -i hello.py
```

The `-i` flag runs the file and then drops you into the REPL with all its variables still available. This is excellent for inspecting results.

## Deeper behavior

### Why `python -m pip` is better than `pip`

When you type `pip`, the shell runs whichever `pip` program comes first on `PATH`, which may belong to a different Python than the one you think. Writing `python -m pip` runs pip as a module of exactly the interpreter you named. The same reasoning applies to `python -m venv` and `python -m pytest`.

### What `-m` does differently from a file path

`python script.py` runs a file. `python -m package.module` finds a module using Python's import system, then runs it. It also adds your current folder to the module search path. This matters for projects with packages, where running files directly breaks imports.

### The `__name__ == "__main__"` guard

Python sets a special variable `__name__` for every file. When a file is run directly, `__name__` equals `"__main__"`. When another file imports it, `__name__` is the module's name instead. This lets one file serve as both a script and a reusable module:

```python
def main() -> None:
    print("running as a program")


if __name__ == "__main__":
    main()
```

The code inside the `if` only executes when you run the file, not when someone imports it. Chapter 4.1 covers this in depth.

### Exit codes from scripts

A script finishes with exit code `0` by default. You can choose another code:

```python
import sys

sys.exit(2)
```

If your script crashes with an unhandled exception, Python exits with code `1`. Automation tools read these numbers to decide what to do next.

### Shebang lines on macOS and Linux

The first line of a script can name the interpreter:

```python
#!/usr/bin/env python3
print("I can be run directly")
```

After running `chmod +x script.py`, you can start it with `./script.py`. This does not work the same way on Windows, where the `py` launcher reads the line to pick a version.

### Working directory again

Running `python projects/hello.py` from your home folder sets the working directory to your home folder, not to `projects`. Any relative file path inside the script is resolved from where you typed the command.

### Interpreter output buffering

When Python's output is sent to a file or a pipe instead of a terminal, it is buffered, meaning it may appear later than expected. Use `python -u script.py` or `print(..., flush=True)` when you need output immediately, such as in logs of long-running services.

## Industry-level snippet

A backend team keeps small command-line tools for operations: checking a service, clearing a cache, rotating logs. Here is a realistic health checker that reads its arguments from `sys.argv`, validates them, handles network errors, logs what it does, and returns exit codes that a deployment system can act on. It uses only the standard library.

```python
"""Check whether an HTTP service is healthy.

Usage:
    python healthcheck.py URL [TIMEOUT_SECONDS]

Exit codes:
    0  the service answered with a 2xx status
    1  the service is unhealthy or unreachable
    2  the command-line arguments were invalid
"""

import logging
import sys
import urllib.error
import urllib.request

EXIT_OK = 0
EXIT_UNHEALTHY = 1
EXIT_USAGE = 2
DEFAULT_TIMEOUT_SECONDS = 5.0

logger = logging.getLogger("healthcheck")


def parse_args(argv: list[str]) -> tuple[str, float]:
    """Return (url, timeout) from the raw argument list, or raise ValueError."""
    if len(argv) not in (2, 3):
        raise ValueError("expected a URL and an optional timeout")

    url = argv[1]
    if not url.startswith(("http://", "https://")):
        raise ValueError(f"URL must start with http:// or https://, got {url!r}")

    timeout = float(argv[2]) if len(argv) == 3 else DEFAULT_TIMEOUT_SECONDS
    if timeout <= 0:
        raise ValueError("timeout must be greater than zero")
    return url, timeout


def is_healthy(url: str, timeout: float) -> bool:
    """Return True if the URL answers with a 2xx status within the timeout."""
    request = urllib.request.Request(url, method="GET")
    try:
        with urllib.request.urlopen(request, timeout=timeout) as response:
            status: int = response.status
    except urllib.error.HTTPError as exc:
        logger.error("Server answered with an error status: %s", exc.code)
        return False
    except (urllib.error.URLError, TimeoutError) as exc:
        logger.error("Could not reach %s: %s", url, exc)
        return False

    logger.info("%s responded with status %s", url, status)
    return 200 <= status < 300


def main(argv: list[str]) -> int:
    """Run the health check and return a process exit code."""
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(message)s",
    )
    try:
        url, timeout = parse_args(argv)
    except ValueError as exc:
        logger.error("Bad arguments: %s", exc)
        logger.error("Usage: python healthcheck.py URL [TIMEOUT_SECONDS]")
        return EXIT_USAGE

    return EXIT_OK if is_healthy(url, timeout) else EXIT_UNHEALTHY


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Try it:

```bash
python healthcheck.py https://example.com 3
echo $?
python healthcheck.py not-a-url
echo $?
```

Notice the structure, because you will see it in nearly all professional scripts:

- A docstring at the top documents usage and exit codes.
- Constants hold values that would otherwise be "magic numbers."
- Small functions do one job each, so they are easy to test.
- `main()` receives `argv` as a parameter, which makes it testable without touching the real command line.
- The `if __name__ == "__main__"` guard keeps the file importable.
- `sys.exit(main(sys.argv))` converts the returned number into the shell's exit code.

## Common mistakes

1. **Running `python` inside the REPL.** Typing `python hello.py` at the `>>>` prompt causes a `SyntaxError`. Exit the REPL first, then run the file at the shell prompt.
2. **Typing shell commands in the REPL, or Python code in the shell.** The prompt tells you where you are: `>>>` is Python, anything else is the shell.
3. **Running a script from the wrong folder.** `python: can't open file ... No such file or directory` means the path is wrong relative to your working directory.
4. **Naming your file after a built-in module.** A file called `random.py` or `json.py` can shadow the real module and break imports. Pick distinctive names.
5. **Forgetting that `sys.argv` values are text.** `sys.argv[1] + 1` fails because `"5" + 1` is not valid. Convert with `int()` or `float()`.
6. **Indexing `sys.argv` without checking its length.** This raises `IndexError` when the user forgets an argument.
7. **Using `pip` instead of `python -m pip`.** You may install into the wrong Python.
8. **Closing the terminal instead of exiting.** In the REPL use `exit()`, and for a running script press Ctrl and C.
9. **Using curly quotes copied from a document.** Python only accepts straight quotes `"` and `'`.

## Practice exercises

1. Start the REPL. Calculate `17 * 23`, then use `_` to add 1 to the previous result. Exit correctly.
2. In the REPL, use `help` to read the documentation for `round`, and `dir` to list what a string can do.
3. Create `hello.py` that prints your name and today's weekday as text you type yourself. Run it from two different folders using a relative path in one case and an absolute path in the other.
4. Use `python -c` to print the interpreter's version as a tuple of major and minor numbers.
5. Use `python -m` to start a web server on port 9000 serving your current folder, open it in a browser, and then stop it.
6. Write `add.py` that reads two numbers from the command line and prints their sum. Test it with `3 4` and with `2.5 4`.
7. Extend `add.py` so that if the user passes fewer than two arguments, it prints a usage message and exits with code 2. Check the exit code in your shell.
8. Extend `add.py` again so that non-numeric input prints a clear error and exits with code 1 instead of a traceback.
9. Write a script that works as both a program and an importable module. Show that `import` does not run its main code, while running it directly does.
10. Modify the health checker so that a third argument selects how many times to retry before giving up, and log each attempt.

## Summary

- The REPL (`python`) is for experiments, script files are for real programs, `-c` runs one-liners, and `-m` runs modules.
- Exit the REPL with `exit()` or Ctrl and D (macOS, Linux) or Ctrl and Z then Enter (Windows). Stop a running script with Ctrl and C.
- `sys.argv` is the list of text arguments, with the script name at index 0.
- Use `python3` on macOS and Linux, `python` or `py` on Windows, and `python` inside any activated virtual environment.
- Prefer `python -m pip` over bare `pip` so the right interpreter is used.
- The `__name__ == "__main__"` guard separates "run as a program" from "imported as a module."
- Programs signal success or failure with exit codes, where 0 means success.
