## Overview

So far every program you have written has lived in one file. That works for experiments, but real software is built from many files that work together. In Python, a file of code that other code can use is called a **module**. Modules let you split a big program into small, focused parts, reuse code across projects, keep names from colliding, and test pieces on their own.

You have been using modules from the start: every time you wrote `import math` or `import json`, you were using a module that someone else wrote. This chapter teaches you to use modules fluently, to write your own, and to make one file work both as a reusable library and as a program you can run.

By the end of this note you will be able to:

- Explain what a module is and what happens when you import one.
- Use `import`, `from ... import`, and aliases, and choose between them.
- Write a module with a docstring, a public interface, and private helpers.
- Use `if __name__ == "__main__":` so one file can be both imported and run.
- Avoid the most common module mistakes, such as shadowing the standard library.

> [!note] A module is just a file
> There is nothing magical about a module. Any file ending in `.py` is a module. Its name is the file name without the extension, so `greetings.py` is the module `greetings`.

## Core concepts

### What a module is

A **module** is a Python file that defines names (functions, classes, constants, variables) and can be loaded by other code. When you import it, Python runs the file and gives you an object, of type `module`, whose **attributes** are the names the file defined.

Every module has its own **namespace**: its own table of names. A variable called `total` in one module does not collide with a variable called `total` in another. This is the "global scope" of Chapter 3.3: global means global **to one module**.

### Where modules come from

| Source | Examples | How it gets on your machine |
| --- | --- | --- |
| Your own files | `greetings.py`, `billing.py` | You write them |
| The standard library | `math`, `json`, `pathlib`, `logging` | Installed with Python |
| Built into the interpreter | `sys`, `builtins`, `time` | Compiled into Python itself |
| Third-party packages | `requests`, `pytest` | Installed with `pip` or `uv` (Chapter 4.4) |

### The import statement

| Form | What it does | How you use the name |
| --- | --- | --- |
| `import greetings` | Loads the module and binds the name `greetings` | `greetings.hello("Ada")` |
| `import greetings as g` | Same, but binds the shorter name `g` | `g.hello("Ada")` |
| `from greetings import hello` | Loads the module, binds only `hello` | `hello("Ada")` |
| `from greetings import hello as say_hello` | Binds `hello` under another name | `say_hello("Ada")` |
| `from greetings import *` | Binds every public name | `hello("Ada")` |
| `import os.path` | Loads a submodule of a package | `os.path.join(...)` |

The first time a module is imported, Python finds the file, runs it from top to bottom to create the module's names, and stores the result. Later imports of the same module reuse the stored result and do not run the file again (Chapter 4.3).

### Which form should you use?

| Prefer | When | Why |
| --- | --- | --- |
| `import module` | Most of the time | The name `module.function` shows where each thing comes from |
| `import module as alias` | Long or conventional names | `import numpy as np` is a community standard |
| `from module import name` | A few names used very often | Shorter code, but the origin is less obvious |
| `from module import *` | Almost never | You cannot tell where names come from, and they may overwrite your own |

### Module attributes

Every module has a few special attributes, with names that start and end with two underscores ("dunder" names):

| Attribute | Meaning |
| --- | --- |
| `__name__` | The module's name, or `"__main__"` if it is the file being run directly |
| `__doc__` | The module's docstring, or `None` |
| `__file__` | The path of the file the module was loaded from |
| `__package__` | The name of the package it belongs to (Chapter 4.2) |
| `__spec__` | Details about how the import system found it (Chapter 4.3) |

### The `__name__` guard

When Python runs a file **directly** (`python greetings.py`), it sets that file's `__name__` to the string `"__main__"`. When the file is **imported**, `__name__` is the module's real name instead. So this line separates "run as a program" from "loaded as a library":

```python
if __name__ == "__main__":
    main()
```

Code under the guard runs only when the file is started directly, never when another module imports it.

### Module docstrings

A string literal that is the first statement in a file is the module's **docstring**. It explains what the module is for, and appears in `help(module)` and in editors. A good module docstring says what the module provides and how to use it, in a few lines.

### Public and private names

By convention, a name that starts with an underscore (`_internal`) is **private**: an implementation detail that other code should not rely on. Everything else is public. A module can also list its public interface explicitly in a variable named `__all__` (Chapter 4.2). Nothing prevents someone from using a private name, it is a signal and a promise, not a lock.

### Naming modules

Module names should be short, lowercase, and use underscores if needed: `file_checksums.py`. A module name must be a valid identifier, so no hyphens or spaces. **Never name a module after a standard library module** (`random.py`, `json.py`, `email.py`), or your file will hide the real one (see Deeper behavior).

## Basic syntax examples

### A first module

Create a file `greetings.py`:

```python
"""Greeting helpers."""

print(f"loading greetings (__name__ = {__name__})")

DEFAULT_GREETING = "Hello"


def hello(name, greeting=DEFAULT_GREETING):
    return f"{greeting}, {name}!"


def _internal():
    return "not part of the public API"


if __name__ == "__main__":
    print(hello("from the command line"))
```

### Importing it

From another file, or from the interactive prompt, in the same folder:

```python
import greetings

print(greetings.hello("Ada"))
import greetings          # a second import does nothing
print("second import done")
```

```text
loading greetings (__name__ = greetings)
Hello, Ada!
second import done
```

The "loading" line appears **once**. The second `import` found the module already loaded and reused it.

### Running it directly

```bash
python greetings.py
```

```text
loading greetings (__name__ = __main__)
Hello, from the command line!
```

This time `__name__` was `"__main__"`, so the guarded block ran and printed the greeting.

### Looking at a module object

```python
import greetings

print(greetings.__name__, "|", greetings.__doc__)
print(greetings.__file__)
print(type(greetings))
print([name for name in dir(greetings) if not name.startswith("__")])
```

```text
greetings | Greeting helpers.
/home/learner/project/greetings.py
<class 'module'>
['DEFAULT_GREETING', '_internal', 'hello']
```

A module is an ordinary object, and `dir()` lists the names it holds.

### The import forms side by side

```python
import greetings as g
from greetings import hello as say_hello
import os.path as osp

print(g.hello("A"), say_hello("B"))
print(osp.basename("/a/b.txt"))
```

```text
Hello, A! Hello, B!
b.txt
```

### Importing from the standard library

```text
>>> import math
>>> math.sqrt(16)
4.0
>>> from math import pi, floor
>>> floor(pi)
3
>>> import os.path
>>> os.path.join("a", "b")
'a/b'
```

On Windows the last result uses a backslash. Chapter 5.5 introduces `pathlib`, which is the better modern tool for paths.

### Import errors

```text
ModuleNotFoundError: No module named 'nonexistent'
```

Python could not find any module with that name: check the spelling, whether the package is installed in the **active** environment (Chapter 0.5), and whether you are running from the right folder.

```text
ImportError: cannot import name 'nothing' from 'math' (unknown location)
ImportError: cannot import name 'nothing' from 'greetings' (/home/learner/project/greetings.py)
```

The module exists, but it does not contain that name. `ModuleNotFoundError` means "no such module", plain `ImportError` means "the module is there, the name is not".

### Interactive help

```python
import greetings

help(greetings)
```

This prints the docstring, the functions, and their signatures in a readable page. Press `q` to leave it.

## Deeper behavior

### A module is executed, and the result is shared

Importing runs the file's top-level code **once**, building the module's namespace. After that, every file that imports the module gets the **same object**. This makes a module a natural place for shared constants and for state that really is shared, but it also means that **changing a module-level variable is visible to everyone** who imported the module. Treat module-level state with the same caution as global variables (Chapter 3.3).

### `from module import name` copies the reference

```python
# config.py contains the single line:  DEBUG = False

from config import DEBUG
import config

config.DEBUG = True
print(DEBUG, config.DEBUG)
```

```text
False True
```

`from config import DEBUG` bound a new name `DEBUG` in your module to the object that existed at that moment. Later rebinding `config.DEBUG` did not change it. If you need to see updates, use `import config` and read `config.DEBUG` each time.

### Why `import *` is discouraged

A star import brings in every public name, which has three problems. A reader cannot tell where a name comes from. Two star imports can silently overwrite each other's names. And a linter cannot tell whether a name is defined. If you must use it, the module's `__all__` controls what it exports. Underscore names are never included.

### Shadowing the standard library

Python looks for modules first in the folder of the script you run, then in the standard library (Chapter 4.3). A file of your own with the same name as a standard module therefore **wins**:

```python
# shadow/random.py contains only:  print("my own random.py")
# shadow/main.py contains:
import random

print(random.randint(1, 6))
```

```text
my own random.py
Traceback (most recent call last):
  File "/home/learner/project/shadow/main.py", line 2, in <module>
    print(random.randint(1, 6))
          ^^^^^^^^^^^^^^
AttributeError: module 'random' has no attribute 'randint'
```

The confusing part is that `import random` succeeded. It simply imported **your** file. Python 3.13 and later add a hint to this error message that names the clashing file, but older versions do not. When you see "module has no attribute" for something that certainly exists, print `module.__file__` to see which file you actually got.

### Keep top-level code light

Everything at a module's top level runs on import. Printing, reading files, opening network connections, or doing heavy computation there slows every program that imports the module and makes testing hard. Put work inside functions, and use the `__name__` guard for the program's entry point.

### Import placement and style

PEP 8 says imports go at the **top** of the file, after the module docstring, one per line, in three groups separated by a blank line: standard library, then third-party packages, then your own modules. Tools such as Ruff sort and check them for you (Chapter 0.9). There are legitimate reasons to import inside a function: to avoid a circular import, or to delay loading a heavy or optional dependency until it is actually needed. Do it deliberately and add a comment explaining why.

### Lazy imports

```python
import sys


def slow_operation():
    import json              # imported only when this function runs
    return json.dumps({"a": 1})


print("json" in sys.modules)
print(slow_operation(), "json" in sys.modules)
```

```text
False
{"a": 1} True
```

(In some environments `json` is already loaded by other code, so the first line may print `True`.)

### Reloading

`importlib.reload(module)` runs a module's code again and updates the existing module object. It is useful when experimenting in a long-running session. Never rely on it in production code, because objects created by the old version still exist and behave differently.

### Importing by name at run time

`importlib.import_module("json")` imports a module whose name you only know as a string, which is how plug-in systems work (Chapter 4.3).

### The main-function pattern

Professional scripts follow a shape you have already seen: constants and functions at the top, a `main()` function that does the work and returns an exit code, and a guarded call at the bottom:

```python
if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Because `main` is an ordinary function, tests can call it directly with a list of arguments and check the returned code. The guard ensures importing the module for a test does not start the program.

### How big should a module be?

Group code that changes together and serves one purpose. A module of a few hundred lines with one clear subject is easy to navigate. A module that mixes unrelated things (parsing, database access, and formatting) is a sign to split. A package (Chapter 4.2) is how you group related modules.

## Industry-level snippet

Operations teams often need to confirm that files have not changed, for example after a transfer or a deployment. This module creates and verifies a **checksum manifest**, in the same format as the standard `sha256sum` tool. It is a model of a well-formed module: a docstring that explains both the library and command-line uses, an explicit `__all__`, small testable functions, a `main()` returning exit codes, the `__name__` guard, and defensive handling of a manifest that points outside its folder.

```python
"""Create and verify SHA-256 checksum manifests for files.

Use it as a library:

    from file_checksums import build_manifest, verify_manifest

or from the command line:

    python file_checksums.py create FILE [FILE ...] > manifest.txt
    python file_checksums.py verify manifest.txt

Exit codes:
    0  success: every file matches
    1  at least one file is missing or has changed
    2  bad usage or unreadable input
"""

import hashlib
import logging
import sys
from collections.abc import Iterable
from dataclasses import dataclass
from pathlib import Path

__all__ = [
    "ChecksumProblem",
    "build_manifest",
    "main",
    "sha256_file",
    "verify_manifest",
]

logger = logging.getLogger(__name__)

CHUNK_SIZE = 1024 * 1024
SEPARATOR = "  "

EXIT_OK = 0
EXIT_PROBLEMS = 1
EXIT_USAGE = 2


@dataclass(frozen=True)
class ChecksumProblem:
    """A file that failed verification."""

    path: str
    reason: str


def sha256_file(path: Path, chunk_size: int = CHUNK_SIZE) -> str:
    """Return the SHA-256 hex digest of a file, reading it in chunks."""
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        while chunk := handle.read(chunk_size):
            digest.update(chunk)
    return digest.hexdigest()


def build_manifest(paths: Iterable[Path], base_dir: Path) -> str:
    """Return manifest text with one "digest  relative/path" line per file.

    Raises:
        ValueError: If a path is not inside base_dir.
        OSError: If a file cannot be read.
    """
    lines = [
        f"{sha256_file(path)}{SEPARATOR}{path.relative_to(base_dir).as_posix()}"
        for path in sorted(paths)
    ]
    return "".join(f"{line}\n" for line in lines)


def verify_manifest(manifest_text: str, base_dir: Path) -> list[ChecksumProblem]:
    """Check every file listed in a manifest and return the problems found.

    Raises:
        ValueError: If a manifest line is malformed or points outside base_dir.
    """
    root = base_dir.resolve()
    problems: list[ChecksumProblem] = []
    for number, line in enumerate(manifest_text.splitlines(), start=1):
        if not line.strip():
            continue
        expected, separator, relative = line.partition(SEPARATOR)
        if not separator or len(expected) != 64:
            raise ValueError(f"line {number}: expected '<sha256>  <path>'")
        target = (root / relative).resolve()
        if root not in target.parents:
            raise ValueError(f"line {number}: path escapes the base directory")
        if not target.is_file():
            problems.append(ChecksumProblem(relative, "missing"))
        elif sha256_file(target) != expected:
            problems.append(ChecksumProblem(relative, "changed"))
    return problems


def main(argv: list[str]) -> int:
    """Run the command-line interface and return a process exit code."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    if len(argv) < 3 or argv[1] not in {"create", "verify"}:
        logger.error("usage: file_checksums.py create FILE... or verify MANIFEST")
        return EXIT_USAGE
    try:
        if argv[1] == "create":
            files = [Path(name).resolve() for name in argv[2:]]
            sys.stdout.write(build_manifest(files, Path.cwd()))
            return EXIT_OK
        manifest = Path(argv[2])
        problems = verify_manifest(
            manifest.read_text(encoding="utf-8"), manifest.resolve().parent
        )
    except (OSError, ValueError) as exc:
        logger.error("%s", exc)
        return EXIT_USAGE
    for problem in problems:
        logger.error("%s: %s", problem.path, problem.reason)
    if problems:
        return EXIT_PROBLEMS
    logger.info("all files match")
    return EXIT_OK


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Use it as a library:

```python
import tempfile
from pathlib import Path

from file_checksums import build_manifest, verify_manifest

with tempfile.TemporaryDirectory() as folder:
    base = Path(folder)
    (base / "a.txt").write_text("alpha\n")
    (base / "b.txt").write_text("beta\n")
    manifest = build_manifest([base / "a.txt", base / "b.txt"], base)
    print(manifest, end="")
    print(verify_manifest(manifest, base))

    (base / "b.txt").write_text("changed\n")
    (base / "a.txt").unlink()
    for problem in verify_manifest(manifest, base):
        print(problem)
```

```text
b6a98d9ce9a2d9149288fa3df42d377c3e42737afdcdaf714e33c0a100b51060  a.txt
f2c82decdd7181cf98945929a62598db7e6b477e11f6e0eb0ae97020eff151ad  b.txt
[]
ChecksumProblem(path='a.txt', reason='missing')
ChecksumProblem(path='b.txt', reason='changed')
```

Or from the command line, in a folder containing `a.txt` and `b.txt`:

```bash
python file_checksums.py create a.txt b.txt > manifest.txt
python file_checksums.py verify manifest.txt
echo $?
```

```text
INFO all files match
0
```

After editing `b.txt` and deleting `a.txt`, the second command reports each problem on standard error and exits with code `1`:

```text
ERROR a.txt: missing
ERROR b.txt: changed
```

Why it is written this way:

- The **module docstring** documents both ways to use the file, so `help(file_checksums)` is useful.
- `__all__` states the public interface. Constants such as `CHUNK_SIZE` and the `EXIT_...` codes remain importable but are not advertised.
- `main()` takes `argv` and returns a code, and the guard at the bottom turns that into the process's exit status. A test can call `main(["x", "verify", path])` directly.
- Reading in chunks keeps memory use constant for files of any size.
- The check `root not in target.parents` stops a hostile manifest from making the tool read files outside its own folder, such as `../../etc/passwd`.
- The logger is named after the module with `logging.getLogger(__name__)`, so log lines can be traced back to their source (Chapter 5.7).

## Common mistakes

1. **Naming your file after a standard library or installed module** (`random.py`, `json.py`, `requests.py`), which hides the real one.
2. **Using a file name that is not a valid identifier,** such as `my-module.py`. It cannot be imported with a normal `import` statement.
3. **Running heavy or noisy code at the top level** of a module, so that merely importing it has side effects.
4. **Forgetting the `__name__` guard** in a file that is meant to be both run and imported.
5. **Using `from module import *`,** which hides where names come from and can overwrite your own.
6. **Expecting `from config import DEBUG` to track later changes** to `config.DEBUG`. It copied the reference.
7. **Importing a module from the wrong folder,** or having several copies of the same file. Print `module.__file__` to check.
8. **Confusing `ModuleNotFoundError` and `ImportError`.** The first means no such module, the second means the module has no such name.
9. **Putting imports in random places in the file,** instead of grouped at the top, unless there is a specific reason.
10. **Creating circular imports** where two modules import each other (Chapter 4.3).
11. **Using `importlib.reload` in production code.**
12. **Treating private underscore names as safe to import from other modules.** They can change without warning.
13. **Making one huge module that mixes unrelated responsibilities.**

## Practice exercises

1. Create `greetings.py` as in this note, import it from a second file, and call its function. Then run `greetings.py` directly. Explain the difference in what you see.
2. Add a `print(__name__)` to the top of a module. Print the value when the module is run directly, when it is imported, and when it is imported twice. Explain each result.
3. Write a module `temperature.py` with functions `celsius_to_fahrenheit` and `fahrenheit_to_celsius`, a module docstring, and a guarded block that prints a small conversion table when run directly.
4. Import `math` in four different ways (plain, with an alias, `from math import sqrt`, `from math import sqrt as root`) and use each. Say which you would choose in real code and why.
5. Create a file named `random.py` in a practice folder, then write a second file that imports the standard `random` module and calls `randint`. Observe the error, and fix it. Explain how you would diagnose it with `__file__`.
6. Demonstrate the `from config import DEBUG` behavior: create `config.py`, import the value both ways, change it through the module, and print both.
7. Use `dir()` on a module you wrote and on `math`. Find out which names are public and which are private by convention.
8. Move a function that does expensive work out of the top level of a module so that importing the module does nothing, and show that importing is now instant.
9. Write a module that exposes only two of its three functions with `__all__`, and show what `from module import *` brings into a second file.
10. Use `importlib.import_module` to import a module chosen by a string typed by the user, and handle the case where the module does not exist with a clear message.
11. Write a script with a `main(argv)` function that returns exit codes, a guard at the bottom, and a second file that imports `main` and calls it with a fake argument list to test it.
12. Extend `file_checksums.py` so that `create` can read a list of file names from standard input when no arguments are given (one per line), and add a `--quiet` option that suppresses the "all files match" message. Write a test for each new behavior.

## Summary

- A **module** is a Python file. Importing it runs its top-level code once, creates a module object with its own namespace, and caches the result for later imports.
- Use `import module` by default, `import module as alias` for long or conventional names, and `from module import name` for a few frequently used names. Avoid `from module import *`.
- `ModuleNotFoundError` means there is no such module. `ImportError` with "cannot import name" means the module exists but lacks that name.
- Every module has `__name__`, `__doc__`, `__file__`, and other attributes. The first statement, if it is a string, is the module docstring.
- `if __name__ == "__main__":` runs code only when the file is executed directly, so one file can be both a library and a program.
- Names that start with an underscore are private by convention. `__all__` declares the public interface.
- `from module import name` copies a reference at import time. Later changes to the module's variable are not seen. Use `import module` if you need current values.
- Never name your own files after standard library or installed modules. Keep top-level code light, and put imports at the top of the file in three groups.
