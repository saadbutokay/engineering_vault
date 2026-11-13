## Overview

An `import` statement looks simple, but a lot happens behind it: Python decides whether it has seen the module before, searches a list of folders to find it, runs its code, and remembers the result. Most import problems that confuse people (`ModuleNotFoundError` when the file is right there, the wrong version of a module being loaded, code that runs twice, and the notorious circular import) all come down to this machinery.

Understanding it turns those mysteries into things you can diagnose in a minute. This chapter explains the steps, the two data structures at the heart of the system (`sys.path` and `sys.modules`), how to find out where a module really came from, and how to fix circular imports properly.

By the end of this note you will be able to:

- Describe what Python does when it executes an `import`.
- Explain `sys.path` and `sys.modules`, and inspect both.
- Work out which file a module was loaded from, and why.
- Explain import caching and its consequences.
- Recognize a circular import, explain why it fails, and fix it in several ways.
- Load modules by name at run time, safely.

> [!note] Two data structures to remember
> `sys.path` is the **list of places to look** for modules. `sys.modules` is the **dictionary of modules already loaded**. Almost every import question is answered by looking at one of them.

## Core concepts

### What happens on `import x`

```text
import x
  1. Is "x" already in sys.modules?
        yes -> use that module object. Done.
        no  -> continue
  2. Ask the finders to locate "x" by searching sys.path
        not found -> raise ModuleNotFoundError
  3. Create an empty module object for "x"
  4. Store it in sys.modules["x"]   (before running the code)
  5. Run the module's code, filling in its namespace
        error -> remove it from sys.modules and raise the error
  6. Bind the name "x" in the importing module
```

Step 4 happens **before** step 5. That detail is the key to understanding circular imports below.

### sys.modules: the import cache

`sys.modules` is an ordinary dictionary that maps module names to module objects. The first thing `import` does is look there. If the name is present, Python uses the stored object and **does not run the file again**. This is why a module's top-level code runs once per program, however many files import it.

Consequences:

- Importing the same module from ten files is cheap and gives all of them the **same object**.
- A change to a module-level variable is visible to everyone, because they share the object.
- Deleting an entry from `sys.modules` makes the next import load the file afresh. Tests sometimes do this.

### Finders and loaders

Python delegates the search to objects called **finders**, kept in the list `sys.meta_path`. Each is asked, in turn, "can you provide this module?". The standard ones are:

| Finder | Provides |
| --- | --- |
| `BuiltinImporter` | Modules compiled into the interpreter, such as `sys` |
| `FrozenImporter` | Modules frozen into the interpreter, used for start-up essentials |
| `PathFinder` | Everything else, by searching the folders in `sys.path` |

Tools you have installed can add finders of their own, so you may see more entries on your machine. For everyday code, `PathFinder` is the one that matters.

### sys.path: where Python looks

`PathFinder` walks through `sys.path`, a list of folder names (and zip files), **in order**, and takes the first match.

| Position | Entry | Comes from |
| --- | --- | --- |
| First | The folder of the script you ran, or the current folder (shown as an empty string) for `python -c`, the REPL, and `python -m` | Python itself |
| Next | Folders listed in the `PYTHONPATH` environment variable | You, via the environment |
| Then | The standard library folders | Python installation |
| Last | `site-packages` of the current environment, plus any folders added by `.pth` files | `pip`, `uv`, virtual environments |

Because the search stops at the first match, **order is priority**. A file in your script's folder beats the standard library, and the standard library beats installed packages with the same name.

### Changing the search path

| Way | Effect | Verdict |
| --- | --- | --- |
| Install the project (`pip install -e .`) | Puts your package on the path properly | Best |
| Set `PYTHONPATH` | Adds folders for one shell or one command | Fine for experiments |
| `sys.path.insert(0, folder)` in code | Changes the path at run time | A last resort. It hides problems and breaks portability |

### Finding where a module came from

A module remembers its origin:

| Tool | Answers |
| --- | --- |
| `module.__file__` | The file it was loaded from (`None` for namespace packages, absent for built-ins) |
| `module.__spec__` | A record of how it was found: name, loader, origin |
| `importlib.util.find_spec("name")` | Where Python **would** load a module from, without importing it |
| `pip show -f package` | The files an installed distribution put on disk |

When a module behaves strangely, `print(module.__file__)` is the first thing to try.

### The bytecode cache

When a module is imported, Python compiles it and stores the result in a `__pycache__` folder as a `.pyc` file (Chapter 1.1). On the next run it reuses the file if the source has not changed, which makes start-up faster. The cache is safe to delete, and belongs in `.gitignore`.

### Circular imports

A **circular import** happens when module A imports module B while B (directly or through other modules) imports A. Because Python adds a module to `sys.modules` **before** it finishes running, the second module finds a half-built version of the first one, and the names it wants may not exist yet.

### Ways to fix a circular import

| Fix | Idea | When to use |
| --- | --- | --- |
| Restructure | Move the shared code into a third module that both import | The best fix. It repairs the design |
| `import module` instead of `from module import name` | Names are looked up later, when used, not at import time | Quick fix when the dependency is genuine |
| Import inside a function | The import runs only when the function is called, after everything has loaded | When one side needs the other only occasionally |
| `TYPE_CHECKING` imports | Import a name only for type hints, never at run time | When the cycle exists only for annotations |
| Pass things in as arguments | Callers hand over what the function needs | When it reduces coupling |

## Basic syntax examples

### Watching `sys.modules`

```text
>>> import sys
>>> "colorsys" in sys.modules
False
>>> import colorsys
>>> "colorsys" in sys.modules
True
>>> sys.modules["colorsys"] is colorsys
True
>>> del sys.modules["colorsys"]
>>> import colorsys as fresh
>>> fresh is colorsys
False
```

The last lines show that removing the cache entry forces Python to load and run the file again, producing a **new** module object.

### A module's code runs once

Using the `greetings.py` module from Chapter 4.1, which prints a line when it loads:

```python
import sys

print("greetings" in sys.modules)
import greetings
print("greetings" in sys.modules, sys.modules["greetings"] is greetings)
import greetings as again
print(again is greetings)
```

```text
False
loading greetings (__name__ = greetings)
True True
True
```

The "loading" message appears once, at the first import. The second import returned the cached object.

### Looking at sys.path

```bash
python -c "import sys; print(repr(sys.path[0]))"
```

```text
''
```

With `python -c` (and in the REPL) the first entry is the empty string, meaning "the current folder". With a script, it is the script's folder:

```python
# showpath.py
import sys

print(sys.path[0])
print(len(sys.path) > 3)
```

```bash
python showpath.py
```

```text
/home/learner/project
True
```

The folder of the script is first, ahead of everything else, which is why a file next to your script can shadow a standard module (Chapter 4.1).

### PYTHONPATH

With a file `extra/helper_mod.py` containing `X = 1`:

```bash
PYTHONPATH=extra python -c "import helper_mod; print(helper_mod.X)"
python -c "import helper_mod"
```

```text
1
ModuleNotFoundError: No module named 'helper_mod'
```

The first command added `extra` to the search path for that one run. The second did not, so the module could not be found.

On Windows PowerShell, set the variable first:

```powershell
$env:PYTHONPATH = "extra"
python -c "import helper_mod; print(helper_mod.X)"
```

### `from package import submodule`

`from x import y` first tries the **attribute** `y` of module `x`. If there is none, and `x` is a package, Python then tries to import a submodule named `x.y`.

```text
>>> import xml
>>> hasattr(xml, "dom")
False
>>> from xml import dom
>>> xml.dom.__name__
'xml.dom'
```

That is why `from shop import pricing` works even when `shop/__init__.py` never imported `pricing`.

### A failed import is not cached

With `bad_mod.py` containing a single line, `raise RuntimeError("boom")`:

```python
import sys

try:
    import bad_mod
except RuntimeError as exc:
    print("first attempt:", exc)

print("bad_mod" in sys.modules)
```

```text
first attempt: boom
False
```

Because the module did not finish running, Python removed it from `sys.modules`, so a later import would try again from the beginning.

### Finding a module's origin without importing it

```python
import importlib
import importlib.util

print(importlib.util.find_spec("json").origin.endswith("json/__init__.py"))
print(importlib.util.find_spec("no_such_module_xyz"))
print(importlib.import_module("math").sqrt(16))
```

```text
True
None
4.0
```

`find_spec` returns `None` for a module that does not exist and an object describing where it is otherwise. `import_module` imports a module whose name you hold as a string.

### Measuring import time

```bash
python -X importtime -c "import json"
```

```text
import time: self [us] | cumulative | imported package
import time:       771 |        771 |       _json
import time:       667 |       1438 |     json.scanner
import time:       545 |       7909 |   json.decoder
import time:       540 |        540 |   json.encoder
import time:       930 |       9377 | json
```

The numbers will differ on your machine. The times are in microseconds, and the indentation shows which module imported which. Slow start-up is often one heavy import, and this view finds it.

### A circular import

Two files that import names from each other:

```python
# circ_a.py
from circ_b import b_value

a_value = "A"
```

```python
# circ_b.py
from circ_a import a_value

b_value = "B"
```

```bash
python run_circ.py     # a file containing only:  import circ_a
```

```text
Traceback (most recent call last):
  File "/home/learner/project/run_circ.py", line 1, in <module>
    import circ_a
  File "/home/learner/project/circ_a.py", line 1, in <module>
    from circ_b import b_value
  File "/home/learner/project/circ_b.py", line 1, in <module>
    from circ_a import a_value
ImportError: cannot import name 'a_value' from partially initialized module 'circ_a' (most likely due to a circular import) (/home/learner/project/circ_a.py)
```

Follow the chain in the traceback. Importing `circ_a` starts running its first line, which imports `circ_b`. That starts running **its** first line, which asks `circ_a` for `a_value`. But `circ_a` is stopped at line 1 and has not defined `a_value` yet. The words "partially initialized module" are Python telling you exactly that.

### Fix 1: import the module, use the names later

```python
# cf_a.py
import cf_b


def from_a():
    return "A calls " + cf_b.from_b()
```

```python
# cf_b.py
import cf_a


def from_b():
    return "B"


def calls_a():
    return "B calls " + cf_a.from_a()
```

```python
import cf_a
import cf_b

print(cf_a.from_a())
print(cf_b.calls_a())
```

```text
A calls B
B calls A calls B
```

Now each module only needs the **other module object** while it loads, and that exists (even though half built). The attribute lookups `cf_b.from_b` happen later, when functions run and everything has finished loading.

### Fix 2: import inside the function

```python
# cl_a.py
from cl_b import b_value

a_value = "A"
```

```python
# cl_b.py
b_value = "B"


def a_plus_b():
    from cl_a import a_value      # runs only when the function is called
    return a_value + b_value
```

```python
import cl_a
import cl_b

print(cl_b.a_plus_b())
```

```text
AB
```

### Fix 3: imports for type hints only

```python
# ty_user.py
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ty_order import Order


class User:
    def __init__(self, name: str) -> None:
        self.name = name
        self.orders: list[Order] = []
```

```python
# ty_order.py
from ty_user import User


class Order:
    def __init__(self, owner: User) -> None:
        self.owner = owner
        owner.orders.append(self)
```

```python
from ty_order import Order
from ty_user import User

user = User("ada")
Order(user)
print(len(user.orders))
```

```text
1
```

`TYPE_CHECKING` is `False` when the program runs and `True` when a type checker reads it. The import of `Order` exists only so that tools understand the annotation. With `from __future__ import annotations`, annotations are not evaluated at run time, so the missing name causes no error.

### Fix 4: restructure

The best repair is usually to ask **why** the modules depend on each other. If two modules both need the same `User` and `Order` definitions, move those into a third module, `models.py`, that imports neither. Both original modules then import from `models`, and the arrows point one way:

```text
Before:   a  <-->  b
After:    a  -->  models  <--  b
```

### Running a script twice by accident

```python
# dbl_a.py
print("running dbl_a as", __name__)
import dbl_b
```

```python
# dbl_b.py
import dbl_a
```

```bash
python dbl_a.py
```

```text
running dbl_a as __main__
running dbl_a as dbl_a
```

When you run a file directly, Python loads it under the name `__main__`. If another module then imports it by its **file name**, Python does not recognize it as the same module and loads a second copy, `dbl_a`. Two copies means two sets of module-level state. Avoid importing the entry-point script from other modules.

## Deeper behavior

### Why a failed import is removed from sys.modules

If a module raised an error halfway through, its namespace is incomplete. Leaving it in the cache would let later imports receive a broken module with no warning. Python removes it so the next attempt starts from scratch, and shows the same error again. Any side effects that happened before the error (a file written, a connection opened) are **not** undone.

### Partially initialized modules are visible

Because step 4 puts the module in `sys.modules` before running it, any code that runs **during** the import, such as a module it imports in turn, can see the half-built object. That is useful, and it is what makes `import module` style work for cycles. It is also dangerous: code can silently observe a module that is missing names.

### Import system performance

The first import of a module involves searching the path, reading the file or its cached bytecode, and running the top-level code. Later imports are a dictionary lookup. Start-up time of a command-line tool is mostly the sum of its first imports. Ways to keep it low:

- Keep top-level code in modules light, and put work in functions.
- Import heavy, rarely used modules **inside the function** that needs them.
- Use `-X importtime` to find the expensive ones.
- Avoid importing a large package only to use one small thing.

### Optional dependencies

A feature that needs a library that not every user installs can load it on demand and give a helpful message when it is missing. The industry-level snippet below shows the pattern, including a subtle point: telling apart "the optional package is missing" from "the package is installed but one of **its** dependencies is missing".

### Import safety

The search path is a security boundary. Python will execute any module it finds first, so:

- Never put a folder that untrusted users can write to on `sys.path`.
- Be careful running scripts in a directory that contains unknown files, because the script's folder comes first. Since Python 3.11, `python -P` (or the environment variable `PYTHONSAFEPATH=1`) stops Python from adding the script folder or current folder automatically.
- `python -I` runs in **isolated mode**: it ignores `PYTHONPATH` and the user's site folder, which is useful for system scripts.
- Never import a module whose name comes straight from user input. Use an allow-list, as the plug-in loader below does.

### Virtual environments and site-packages

Inside a virtual environment, `sys.prefix` differs from `sys.base_prefix` (Chapter 0.5), and the environment's own `site-packages` folder is on `sys.path`. That is how a package installed in one environment is invisible in another. A "module not found" error for something you just installed usually means you installed it into a different environment than the one running your code. Compare `sys.executable` with the interpreter you meant to use.

```text
/home/learner/project/.venv/lib/python3.12/site-packages
```

### Editable installs

`pip install -e .` does not copy your code. It adds a small pointer, using the same mechanisms as above, so that your `src/` folder is found when someone imports your package. Edits take effect immediately, and the package can be imported from anywhere, with no `sys.path` tricks (Chapter 0.6).

### `import` is a call to `importlib`

The `import` statement ultimately calls the built-in `__import__` function, which is implemented by the `importlib` module. When you need to import by name, call `importlib.import_module(name)` instead of using `__import__` directly. It is clearer and handles dotted names and packages properly.

### Reloading is dangerous

`importlib.reload(module)` re-executes a module's code into the **same** module object. Existing references, instances of classes defined in the old version, and names copied with `from module import name` keep pointing at the old objects. It is fine in a notebook or while experimenting, but not a way to update a running service.

### Zip imports

Python can import from a `.zip` file placed on `sys.path`, and a whole application can be shipped as a single `.pyz` archive. This is why reading package data with `importlib.resources`, rather than `__file__`, matters (Chapter 4.2).

## Industry-level snippet

Applications often let users configure behavior by naming code to run, such as a custom exporter or validator written as `"package.module:function"`. Loading such a reference is a security-sensitive use of the import system. This module loads plug-ins with an **allow-list** of package prefixes, checks the result honors the contract, reports every failure as one clear error type, and provides two helpers: one for optional dependencies, and one that reports where a module would come from, which is invaluable when diagnosing shadowing.

```python
"""Safe plug-in loading and optional-dependency helpers."""

import importlib
import importlib.util
import logging
from collections.abc import Callable, Sequence
from types import ModuleType
from typing import Any

__all__ = ["PluginError", "find_origin", "load_plugin", "require_optional"]

logger = logging.getLogger(__name__)


class PluginError(Exception):
    """Raised when a plug-in cannot be loaded or breaks the contract."""


def _is_allowed(module_name: str, allowed_prefixes: Sequence[str]) -> bool:
    """Return True if module_name is one of the prefixes or inside one."""
    return any(
        module_name == prefix or module_name.startswith(prefix + ".")
        for prefix in allowed_prefixes
    )


def load_plugin(
    reference: str, allowed_prefixes: Sequence[str]
) -> Callable[..., Any]:
    """Import and return the callable named by "package.module:attribute".

    Only modules inside the allowed package prefixes may be imported.

    Raises:
        PluginError: If the reference is malformed, not allowed, cannot be
            imported, or does not name a callable.
    """
    module_name, separator, attribute = reference.partition(":")
    if not (module_name and separator and attribute):
        raise PluginError(
            f"plug-in reference must look like 'package.module:name', got {reference!r}"
        )
    if not _is_allowed(module_name, allowed_prefixes):
        raise PluginError(f"module {module_name!r} is not in an allowed package")
    try:
        module = importlib.import_module(module_name)
    except ImportError as exc:
        raise PluginError(f"cannot import {module_name!r}: {exc}") from exc
    try:
        target = getattr(module, attribute)
    except AttributeError as exc:
        raise PluginError(
            f"module {module_name!r} has no attribute {attribute!r}"
        ) from exc
    if not callable(target):
        raise PluginError(f"{reference!r} is not callable")
    logger.info("Loaded plug-in %s", reference)
    return target  # type: ignore[no-any-return]


def require_optional(
    module_name: str, purpose: str, package: str | None = None
) -> ModuleType:
    """Import an optional dependency, or explain how to install it.

    Args:
        module_name: The module to import.
        purpose: What the caller needs it for, used in the message.
        package: The name to install, if it differs from the module name.

    Raises:
        ImportError: With installation advice if the module is missing. Errors
            raised by the module's own imports are passed through unchanged.
    """
    try:
        return importlib.import_module(module_name)
    except ModuleNotFoundError as exc:
        missing = exc.name or ""
        if not (module_name == missing or module_name.startswith(missing + ".")):
            raise  # the module exists, but one of ITS dependencies is missing
        raise ImportError(
            f"{purpose} requires the optional package {package or module_name!r}. "
            f"Install it with: pip install {package or module_name}"
        ) from exc


def find_origin(module_name: str) -> str | None:
    """Return where a module would be loaded from, without importing it.

    Returns None if the module cannot be found. Helpful for spotting a local
    file that shadows a standard or installed module.
    """
    try:
        spec = importlib.util.find_spec(module_name)
    except (ImportError, ValueError):
        return None
    return None if spec is None else spec.origin
```

Try it:

```python
print(load_plugin("json:dumps", ["json"])({"a": 1}))

for reference in ["os:system", "json", "json:nothing", "json:decoder", "jsonx.tools:run"]:
    try:
        load_plugin(reference, ["json", "jsonx"])
    except PluginError as exc:
        print("PluginError:", exc)

try:
    require_optional("no_such_module_xyz", "Chart rendering", package="chartlib")
except ImportError as exc:
    print("ImportError:", exc)

print(require_optional("math", "Calculations").sqrt(9))
print(find_origin("sys"), find_origin("no_such_module_xyz"))
```

```text
{"a": 1}
PluginError: module 'os' is not in an allowed package
PluginError: plug-in reference must look like 'package.module:name', got 'json'
PluginError: module 'json' has no attribute 'nothing'
PluginError: 'json:decoder' is not callable
PluginError: cannot import 'jsonx.tools': No module named 'jsonx'
ImportError: Chart rendering requires the optional package 'chartlib'. Install it with: pip install chartlib
3.0
built-in None
```

Why it is written this way:

- The **allow-list** runs before any import happens. A configuration file that names `os:system` is rejected without ever loading `os`, so a malicious or mistaken setting cannot run arbitrary code.
- All the failure modes (malformed text, not allowed, import failure, missing attribute, not callable) become one `PluginError` with the original cause attached, so the caller needs a single `except`.
- `require_optional` checks `exc.name` to tell "this package is missing" from "this package is installed but one of its own dependencies is missing". Reporting the second case as "please install the package" would send the user in the wrong direction.
- `find_origin` uses `find_spec`, so it can answer "where would this come from?" without running any code from the module.
- The `# type: ignore` is deliberate and narrow: `getattr` returns an untyped value, and the `callable` check has just established what the type checker cannot see.

## Common mistakes

1. **Naming a file after a standard or installed module,** and importing that instead of the real one.
2. **Installing a package in one environment and running code in another.** Check `sys.executable`.
3. **Fixing import problems with `sys.path.insert`** instead of installing the project properly.
4. **Creating circular imports** by having every module import from `__init__.py`, or by putting shared definitions in the wrong module.
5. **Using `from module import name` in both halves of a circular dependency.**
6. **Importing the entry-point script from another module,** loading it twice under two names.
7. **Doing slow or side-effect work at import time,** such as opening connections or reading large files.
8. **Relying on `importlib.reload`** to update running code.
9. **Passing user-controlled text to `importlib.import_module`** without an allow-list.
10. **Catching `ImportError` too broadly** around an optional import, hiding real bugs in the imported module.
11. **Assuming a module-level variable is private to one importer.** All importers share the module object.
12. **Deleting entries from `sys.modules` in production code.** Reserve it for tests.
13. **Forgetting that the script's own folder is first on the path,** so unrelated files beside your script can shadow other modules.

## Practice exercises

1. Print `sys.path` in a script and in the REPL. Explain the difference between the first entries, and identify which entries belong to the standard library and which to `site-packages`.
2. Create a module that prints a message when loaded. Import it three times from three places and count the messages. Then remove it from `sys.modules` and import again.
3. Use `module.__file__` and `importlib.util.find_spec` to find the location of five modules: two from the standard library, one installed with `pip`, one of yours, and one built in.
4. Create a file named `json.py` next to a script that imports `json`, observe what happens, and then use `__file__` to prove which file you got. Rename the file and confirm the fix.
5. Reproduce the circular import from this note, then repair it using each of the four fixes in turn: a third module, `import module`, a function-level import, and `TYPE_CHECKING`.
6. Use `PYTHONPATH` to make a folder of modules importable for a single command, and then show that the same import fails without it.
7. Run `python -X importtime` on a script that imports two or three standard modules. Identify the slowest import and its cumulative time.
8. Write a function that imports a heavy module lazily inside it, and show with `sys.modules` that importing your module does not load the heavy one.
9. Build a tiny plug-in system: a folder of modules each defining a `run()` function, and a loader that imports them by name from an allow-list read from a file. Handle a missing module and a module with no `run`.
10. Show how to detect that you are running inside a virtual environment from Python, and print the path of the environment's `site-packages`.
11. Write a script that is both run directly and imported by another file that **also** imports it by file name. Show the double load, and fix the design.
12. Extend `load_plugin` so that it can also accept a `package.module:Class.method` reference by following dots in the attribute part, and add a `required_signature` check that rejects callables that cannot accept a given number of arguments. Write tests for both.

## Summary

- `import x` first checks `sys.modules`. If `x` is absent, finders search `sys.path`, Python creates a module object, stores it in `sys.modules`, **then** runs its code, and finally binds the name.
- `sys.modules` is the cache of loaded modules. A module's top-level code runs once, and all importers share one object.
- `sys.path` lists where to search, in priority order: the script's folder (or current folder), `PYTHONPATH`, the standard library, then `site-packages`. The first match wins.
- Diagnose odd imports with `module.__file__`, `importlib.util.find_spec`, and `sys.executable`. Prefer installing your project in editable mode over editing `sys.path`.
- A failed import is removed from `sys.modules`, but its side effects remain.
- A **circular import** fails because a module is already in `sys.modules` but only partly built. Fix it by restructuring (a third module), using `import module` and late lookups, importing inside functions, or using `TYPE_CHECKING` for annotation-only imports.
- Running a file directly loads it as `__main__`. Importing it by name loads a second copy, so do not import entry-point scripts.
- Keep imports light, delay heavy ones, and measure with `-X importtime`.
- Treat the import path as a security boundary: never import names straight from user input, and use an allow-list for plug-ins.
