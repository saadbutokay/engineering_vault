## Overview

A single module is enough for a small tool. A real application has dozens or hundreds of modules, and they need structure. A **package** is how Python groups related modules into a folder with a name, so that `billing.invoices` and `billing.taxes` live together and are imported with dotted names. Every serious Python project, and every library you install, is organized as one or more packages.

This chapter explains how packages work, how modules inside a package refer to each other, how to decide what a package exposes to the outside world, and how to avoid the confusing errors people hit the first time they run code that lives inside one.

By the end of this note you will be able to:

- Create a package with an `__init__.py` file and subpackages.
- Import from packages with absolute and relative imports, and explain the difference.
- Use `__init__.py` to present a clean public interface, and `__all__` to control it.
- Explain what a namespace package is and when it appears.
- Run code inside a package correctly with `python -m`.
- Lay out a real library with a private implementation and a public API.

> [!note] Package, module, and distribution
> Three words that get mixed up. A **module** is one `.py` file. A **package** is a folder of modules you can import. A **distribution** is something you install with `pip`, such as `requests`, which contains one or more packages. This chapter is about the middle one.

## Core concepts

### What a package is

A **regular package** is a folder that contains a file named `__init__.py`. Python treats the folder as importable, and its name becomes the package name. Inside it you can put modules and other packages (**subpackages**), to any depth.

```text
shop/
    __init__.py          the package itself
    pricing.py           module shop.pricing
    orders.py            module shop.orders
    utils/               subpackage shop.utils
        __init__.py
        text.py          module shop.utils.text
```

### Dotted names

The path through the folders becomes a dotted name: `shop.utils.text` means "the module `text` inside the subpackage `utils` of the package `shop`". The dots mirror the folders.

### The `__init__.py` file

When a package is imported, Python **runs its `__init__.py`** first, and the resulting namespace is the package's own. The file may be empty. Typical uses:

| Use | Example |
| --- | --- |
| Mark the folder as a package | An empty file |
| Re-export names for a friendlier API | `from .pricing import apply_discount` |
| Define package metadata | `__version__ = "1.0.0"` |
| Declare the public interface | `__all__ = [...]` |
| Package-wide setup that must run once | Rare. Keep it minimal |

### Ways to import from a package

| Statement | Result |
| --- | --- |
| `import shop.pricing` | Binds `shop`. Use `shop.pricing.apply_discount(...)` |
| `from shop import pricing` | Binds `pricing`. Use `pricing.apply_discount(...)` |
| `from shop.pricing import apply_discount` | Binds `apply_discount` |
| `import shop.pricing as p` | Binds `p` to the submodule |

Importing `shop.pricing` first imports `shop`, running `shop/__init__.py`, then the submodule.

### Absolute imports

An **absolute import** gives the full dotted path starting from a top-level package that Python can find:

```python
from shop.pricing import apply_discount
```

It means the same thing from any file in the project, and tells the reader exactly where the name comes from. PEP 8 recommends absolute imports as the default.

### Relative imports

A **relative import** gives a path relative to the current module, using leading dots:

| Syntax | Meaning |
| --- | --- |
| `from . import pricing` | The module `pricing` in the **same** package as this file |
| `from .pricing import apply_discount` | A name from the sibling module `pricing` |
| `from .. import pricing` | A module in the **parent** package |
| `from ..utils.text import label` | A module in a sibling subpackage |

Relative imports are written only with `from ... import ...`, work **only inside a package**, and keep working if you rename the top-level package. They are shorter, but the reader must work out where the dots lead.

### `__all__`

`__all__` is a list of strings naming a module's or package's public names. It controls what `from package import *` brings in, and it documents the intended interface for readers and for tools. Names that start with an underscore are excluded from star imports anyway, but `__all__` is the explicit statement.

### Namespace packages

A folder **without** an `__init__.py` can still be imported, as a **namespace package**. The special feature is that several folders in different places can contribute to one package name, and Python merges them. Namespace packages exist for large projects whose parts are shipped separately, such as plug-ins that all live under one name. For ordinary projects, always include `__init__.py`.

### Running code that lives in a package

A module inside a package expects to be imported as part of that package, so that its relative imports can be resolved. Running it as a plain file breaks them. Use the `-m` flag, which runs a module by its **dotted name**:

```bash
python -m shop.orders
```

A package can also be made runnable by adding a `__main__.py` file, so that `python -m shop` works.

### Layout for a real project

As in Chapter 0.6, a project places its packages inside a `src/` folder and its tests beside it, and installs itself in editable mode so that imports work everywhere without path tricks:

```text
orderlib-project/
    pyproject.toml
    README.md
    src/
        orderlib/
            __init__.py
            errors.py
            models.py
            pricing.py
            _validation.py
            __main__.py
    tests/
        test_pricing.py
```

## Basic syntax examples

### A small package

These files make up the `shop` package. Each block starts with a comment naming the file.

```python
# shop/__init__.py
"""A small shop package."""

from .pricing import apply_discount, total_price

__all__ = ["apply_discount", "total_price", "__version__"]
__version__ = "1.0.0"
```

```python
# shop/pricing.py
"""Pricing helpers."""


def apply_discount(price, percent):
    return round(price * (1 - percent / 100), 2)


def total_price(prices):
    return round(sum(prices), 2)
```

```python
# shop/orders.py
"""Order helpers."""

from . import pricing
from .pricing import apply_discount
from .utils.text import label


def describe(prices):
    return label("total", pricing.total_price(prices))


if __name__ == "__main__":
    print(describe([10, 5.5]), apply_discount(100, 15))
```

```python
# shop/utils/text.py
"""Text helpers."""

from ..pricing import total_price


def label(name, value):
    return f"{name.upper()}: {value}"
```

The file `shop/utils/__init__.py` is empty. Notice the three relative imports: `.pricing` (a sibling), `.utils.text` (a subpackage below), and `..pricing` (a module in the parent package).

### Inspecting a package

```python
import shop

print(shop.__name__, shop.__version__)
print(shop.__file__)
print(shop.__path__)
print(shop.__package__)

import shop.orders

print(shop.orders.__name__, shop.orders.__package__)
print(shop.orders.describe([1, 2]))
```

```text
shop 1.0.0
/home/learner/project/shop/__init__.py
['/home/learner/project/shop']
shop
shop.orders shop
TOTAL: 3
```

A package has `__path__`, the list of folders that hold its modules. A plain module does not.

### Running a file inside a package directly fails

```bash
python shop/orders.py
```

```text
Traceback (most recent call last):
  File "/home/learner/project/shop/orders.py", line 3, in <module>
    from . import pricing
ImportError: attempted relative import with no known parent package
```

Running the file directly makes Python treat it as a stand-alone script named `__main__`, with no package around it, so `from . import pricing` has nothing to be relative to. The fix is to run it as a module of its package, from the folder that **contains** `shop`:

```bash
python -m shop.orders
```

```text
TOTAL: 15.5 85.0
```

### Making a package runnable

```python
# shop/__main__.py
"""Run the shop package: python -m shop."""

from .orders import describe

print(describe([1, 2]))
```

```bash
python -m shop
```

```text
TOTAL: 3
```

### Subpackages are not imported automatically

```text
>>> import shop.utils
>>> shop.utils.text
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: module 'shop.utils' has no attribute 'text'
```

Importing a package does not import every module inside it. `shop.utils.text` only exists once something has imported it:

```text
>>> import shop.utils.text
>>> shop.utils.text.label("x", 1)
'X: 1'
```

### What `import *` brings in

```python
from shop import *

print(sorted(name for name in dir() if not name.startswith("__")))
print(__version__)
print("pricing" in dir())
```

```text
['apply_discount', 'total_price']
1.0.0
False
```

Only the names in `shop.__all__` were imported. The submodule `pricing` was not, even though it exists inside the package.

### A namespace package

Two separate folders, both on the import path, each with a `plugins` folder that has **no** `__init__.py`:

```text
ns_a/plugins/alpha.py     contains:  NAME = "alpha"
ns_b/plugins/beta.py      contains:  NAME = "beta"
```

```python
import plugins.alpha
import plugins.beta

print(type(plugins.__path__).__name__, sorted(plugins.__path__))
print(plugins.__file__)
print(plugins.alpha.NAME, plugins.beta.NAME)
```

```text
_NamespacePath ['/home/learner/project/ns_a/plugins', '/home/learner/project/ns_b/plugins']
None
alpha beta
```

(This assumes both `ns_a` and `ns_b` are on `sys.path`, for example by setting the `PYTHONPATH` environment variable.) One package name, `plugins`, was assembled from two places. A namespace package has no `__init__.py`, so its `__file__` is `None`.

### Package data files

```python
from importlib.resources import files

print(files("shop").joinpath("defaults.txt").read_text().strip())
```

```text
tax=0.2
```

`importlib.resources` reads a data file that ships **inside** a package, and keeps working when the package is installed from a wheel or a zip file, where `open(__file__ + ...)` tricks would fail.

## Deeper behavior

### What `import a.b.c` really does

Python imports `a`, then `a.b`, then `a.b.c`, **in that order**, running each `__init__.py` along the way. It binds only the name `a` in your namespace, and you reach the rest through attributes: `a.b.c`. This explains the `AttributeError` above, and why a heavy `__init__.py` slows every import of anything in the package.

### Relative imports depend on `__package__`

A relative import is resolved using the module's `__package__` attribute. For `shop/orders.py` imported normally, it is `"shop"`, so `.pricing` becomes `shop.pricing`. When the file runs as a script, `__package__` is empty and the resolution fails. This is the entire reason for the error above.

### Absolute or relative?

| | Absolute (`from shop.pricing import x`) | Relative (`from .pricing import x`) |
| --- | --- | --- |
| Reads clearly | Yes, the origin is explicit | Needs counting dots |
| Survives renaming the package | No | Yes |
| Survives moving the module to another folder | Yes | No |
| Works in scripts | Yes (if on the path) | No |
| Recommended by PEP 8 | As the default | Acceptable inside complex packages |

Pick one convention per project and apply it consistently. Many teams use absolute imports everywhere and relative imports only for closely related sibling modules.

### Keep `__init__.py` light

Everything the `__init__.py` imports is loaded the moment anyone imports **any** part of the package. If `shop/__init__.py` imported a heavy dependency, `import shop.pricing` would pay for it. Re-exports are convenient, but they have costs: slower start-up, a greater chance of circular imports (Chapter 4.3), and a public API that is harder to see. For packages with expensive parts, load them lazily. Python allows a module-level `__getattr__` function (PEP 562) to import on first use.

### Controlling the public API

A good package decides what it promises. A common design:

- **Public** names are re-exported in `__init__.py` and listed in `__all__`. Users write `from orderlib import Order`.
- **Implementation** modules, such as `models.py` and `pricing.py`, can be reorganized freely, because users do not import from them directly.
- **Private** modules have a leading underscore (`_validation.py`) and are never part of the contract.

Strict type checking agrees with this design: with `mypy --strict`, a name imported into `__init__.py` counts as re-exported only if it is listed in `__all__` or imported with `from x import y as y`. An unlisted import is treated as private.

### Namespace packages and their traps

Python builds a namespace package by scanning every folder on `sys.path` for a directory with the right name and merging them all. Two consequences:

- **A forgotten `__init__.py` silently creates a namespace package.** Code may still import, but tools that rely on `__init__.py` (some test runners, packaging tools) behave differently, and the folder can be shadowed by an unrelated one elsewhere on the path.
- A **regular package** found earlier on the path takes priority over namespace portions with the same name.

Unless you have a real reason, give every package an `__init__.py`.

### Why `-m` also fixes the import path

`python -m shop.orders` puts the current directory on the module search path and runs `shop.orders` as a member of the package. `python shop/orders.py` instead puts the **`shop` folder** on the path, which is why, in that case, sibling modules appear to import oddly, and why files in different packages with the same name can shadow each other. Prefer `-m`, or better, install your project in editable mode so that imports do not depend on where you stand (Chapter 0.6).

### Do not edit `sys.path` to make imports work

Code that begins with `sys.path.append("..")` is a symptom of a package that is not installed properly. It makes imports depend on the working directory and breaks as soon as the code is moved or run elsewhere. Use the `src/` layout and `pip install -e .`.

### Packages and distribution

When you build a wheel for your project (Chapter 13.1), the packages under `src/` are what ends up inside it. Your `tests/` folder is usually not a package and is not included. The package's name for `import` and the project's name for `pip install` are separate choices, and often differ in punctuation: `pip install my-project` can provide `import my_project`.

### Reading package metadata

The standard library's `importlib.metadata` reads information about **installed** distributions, so a package can report its own version without repeating it in the code:

```python
from importlib.metadata import PackageNotFoundError, version

try:
    __version__ = version("orderlib")
except PackageNotFoundError:
    __version__ = "0+unknown"       # not installed, for example run from a checkout
```

### Structure guidelines

| Guideline | Reason |
| --- | --- |
| Keep nesting shallow, two or three levels at most | Long dotted names are hard to read and import |
| Group by purpose (`billing`, `accounts`) rather than by kind (`models`, `views`) in large projects | Related code changes together |
| Avoid circular dependencies between packages | They cause import errors and design trouble |
| Do not put real logic in `__init__.py` | It runs on every import and is easy to overlook |
| Keep tests outside the package | They are not shipped to users |

## Industry-level snippet

This is the complete source of a small, realistic library called `orderlib`, laid out the way real projects are: a public API re-exported from `__init__.py`, implementation modules, a private helper module, a base exception class, a `__main__.py` so that `python -m orderlib` works, and metadata read from the installed distribution. Each block starts with a comment naming its file.

```text
orderlib-project/
    pyproject.toml
    src/
        orderlib/
            __init__.py
            __main__.py
            _validation.py
            errors.py
            models.py
            pricing.py
```

```python
# src/orderlib/errors.py
"""Exceptions raised by orderlib."""

__all__ = ["InvalidLineError", "OrderError"]


class OrderError(Exception):
    """Base class for every error raised by orderlib."""


class InvalidLineError(OrderError, ValueError):
    """Raised when an order line contains invalid data."""
```

```python
# src/orderlib/_validation.py
"""Private validation helpers. Not part of the public API."""

from .errors import InvalidLineError


def require_non_negative_int(name: str, value: int) -> int:
    """Return value if it is a whole number of zero or more."""
    if isinstance(value, bool) or not isinstance(value, int) or value < 0:
        raise InvalidLineError(
            f"{name} must be a whole number of 0 or more, got {value!r}"
        )
    return value


def require_positive_int(name: str, value: int) -> int:
    """Return value if it is a whole number of one or more."""
    if require_non_negative_int(name, value) < 1:
        raise InvalidLineError(f"{name} must be at least 1, got {value!r}")
    return value
```

```python
# src/orderlib/models.py
"""Data classes describing an order."""

from dataclasses import dataclass, field

from ._validation import require_non_negative_int, require_positive_int
from .errors import InvalidLineError

__all__ = ["Order", "OrderLine"]


@dataclass(frozen=True)
class OrderLine:
    """One product on an order. Prices are in cents."""

    sku: str
    unit_price_cents: int
    quantity: int

    def __post_init__(self) -> None:
        if not self.sku.strip():
            raise InvalidLineError("sku must not be blank")
        require_non_negative_int("unit_price_cents", self.unit_price_cents)
        require_positive_int("quantity", self.quantity)

    def total_cents(self) -> int:
        """Return the price of this line before any discount."""
        return self.unit_price_cents * self.quantity


@dataclass(frozen=True)
class Order:
    """An immutable collection of order lines."""

    lines: tuple[OrderLine, ...] = field(default_factory=tuple)
```

```python
# src/orderlib/pricing.py
"""Price calculations."""

import logging

from .errors import OrderError
from .models import Order

__all__ = ["order_total_cents"]

logger = logging.getLogger(__name__)

MAX_DISCOUNT_PERCENT = 50


def order_total_cents(order: Order, discount_percent: int = 0) -> int:
    """Return the order total in cents after a whole-percent discount.

    Raises:
        OrderError: If the discount is outside 0 to MAX_DISCOUNT_PERCENT.
    """
    if not 0 <= discount_percent <= MAX_DISCOUNT_PERCENT:
        raise OrderError(
            f"discount must be between 0 and {MAX_DISCOUNT_PERCENT} percent"
        )
    subtotal = sum(line.total_cents() for line in order.lines)
    discount = subtotal * discount_percent // 100
    logger.debug("subtotal=%d discount=%d", subtotal, discount)
    return subtotal - discount
```

```python
# src/orderlib/__init__.py
"""orderlib: a tiny library for pricing orders.

Import everything you need from the top-level package:

    from orderlib import Order, OrderLine, order_total_cents

The submodules are implementation details and may be reorganized.
"""

from importlib.metadata import PackageNotFoundError, version

from .errors import InvalidLineError, OrderError
from .models import Order, OrderLine
from .pricing import order_total_cents

try:
    __version__ = version("orderlib")
except PackageNotFoundError:  # running from a source checkout, not installed
    __version__ = "0+unknown"

__all__ = [
    "InvalidLineError",
    "Order",
    "OrderError",
    "OrderLine",
    "__version__",
    "order_total_cents",
]
```

```python
# src/orderlib/__main__.py
"""A small demonstration: python -m orderlib."""

from . import Order, OrderLine, __version__, order_total_cents


def main() -> None:
    """Print a sample order total."""
    order = Order((OrderLine("PEN", 120, 10), OrderLine("BOOK", 1599, 1)))
    print(f"orderlib {__version__}")
    print(f"total: {order_total_cents(order, discount_percent=10)} cents")


if __name__ == "__main__":
    main()
```

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "orderlib"
version = "0.1.0"
description = "A tiny library for pricing orders"
requires-python = ">=3.12"
dependencies = []
```

After installing it with `python -m pip install -e .`, anyone can use the library through the public names:

```python
from orderlib import InvalidLineError, Order, OrderLine, order_total_cents

order = Order((OrderLine("PEN", 120, 10), OrderLine("BOOK", 1599, 1)))
print(order_total_cents(order))
print(order_total_cents(order, discount_percent=10))

try:
    OrderLine("PEN", 120, 0)
except InvalidLineError as exc:
    print("InvalidLineError:", exc)

import orderlib

print(orderlib.__all__)
```

```text
2799
2520
InvalidLineError: quantity must be at least 1, got 0
['InvalidLineError', 'Order', 'OrderError', 'OrderLine', '__version__', 'order_total_cents']
```

And the package runs as a program:

```bash
python -m orderlib
```

```text
orderlib 0+unknown
total: 2520 cents
```

(When the package is installed, the first line shows the installed version, `0.1.0`.)

Why it is laid out this way:

- Users import from `orderlib`, and only names in `__all__` are promised. The modules `models.py` and `pricing.py` can be split or renamed without breaking anyone.
- `_validation.py` has a leading underscore and is imported only inside the package, so it is clearly private.
- A base class `OrderError` lets callers catch every library error with one `except`, while `InvalidLineError` also inherits `ValueError` so that generic code that handles bad values still works.
- Relative imports inside the package (`from .errors import ...`) keep the modules working if the top-level name changes, while the public example uses the absolute form.
- The modules depend on each other in one direction only (`errors`, then `_validation`, then `models`, then `pricing`, then `__init__`), so there are no import cycles.
- The version comes from the installed metadata, so it is written in one place only, in `pyproject.toml`.

## Common mistakes

1. **Forgetting `__init__.py`.** The folder may still import as a namespace package, but tools and path rules behave differently. Add the file.
2. **Running a module inside a package as a script** (`python shop/orders.py`) and getting "attempted relative import with no known parent package". Use `python -m shop.orders` from the project root.
3. **Assuming `import shop` imports its submodules.** It runs `__init__.py` only.
4. **Using relative imports in a top-level script,** where there is no package.
5. **Fixing import errors with `sys.path.append(...)`.** Install the project in editable mode.
6. **Putting heavy imports or real logic in `__init__.py`,** slowing every import.
7. **Creating circular imports between modules of the same package,** typically because `__init__.py` imports everything and modules import from `__init__`.
8. **Importing a package's private modules from outside it.** Use the documented public names.
9. **Giving a package the same name as a standard module,** or giving a package and a module in the same folder the same name.
10. **Opening data files with `open(__file__ ...)`.** Use `importlib.resources`.
11. **Forgetting to list a new public name in `__all__`,** or listing a name that does not exist (star imports then fail).
12. **Mixing absolute and relative imports without a convention.**
13. **Deeply nested package trees,** which make imports long and the structure hard to remember.

## Practice exercises

1. Create the `shop` package from this note with the three modules and the `utils` subpackage. Import each module in a different way (`import`, `from ... import`, with an alias) and call a function from each.
2. Run `shop/orders.py` directly and read the error. Then fix it by running it with `python -m`. Explain in your own words why the first fails and the second works.
3. Add a `__main__.py` to `shop` so that `python -m shop` prints a small price list. Check what `__name__` is inside it.
4. Remove the `__init__.py` from `shop/utils` and observe whether `shop.utils.text` still imports. Explain what kind of package `utils` is now, and what could go wrong with it.
5. Set `__all__` in `shop/__init__.py` to a single name and use `from shop import *` in a second file. Print which names arrived. Add a name that does not exist to `__all__` and read the error.
6. Rewrite the three relative imports in `shop` as absolute imports. Then rename the package folder to `store` and see which version needs changes.
7. Put a `print("importing shop")` in `shop/__init__.py`, a different print in each submodule, and observe the order of the messages when you run `import shop.utils.text`.
8. Create two folders that both contain a `plugins` folder with no `__init__.py`, add both to `PYTHONPATH`, and import modules from each through the single name `plugins`. Print `plugins.__path__`.
9. Add a data file `defaults.txt` to `shop` and read it with `importlib.resources.files`. Show why reading it with a path built from `__file__` would break if the package were installed from a zip.
10. Install `orderlib` in editable mode into a fresh virtual environment, run `python -m orderlib`, and compare the printed version with the one before installation.
11. Create a circular import between two modules in a package by making `models.py` import from `pricing.py` and the reverse. Read the error, then break the cycle without changing what either module offers.
12. Extend `orderlib` with a `formatting.py` module that provides `format_cents(cents)` and re-export it through the package's public API. Update `__all__`, add one new error subclass for invalid money amounts, and write tests that confirm the new names are importable from the top level only.

## Summary

- A **package** is a folder of modules. A **regular package** contains an `__init__.py`, which runs when the package is imported. Dotted names mirror the folder structure.
- Importing a package runs its `__init__.py` but does **not** import its submodules. `import a.b.c` imports `a`, then `a.b`, then `a.b.c`, and binds only `a`.
- **Absolute imports** (`from shop.pricing import x`) give the full path and are the PEP 8 default. **Relative imports** (`from .pricing import x`, `from ..utils import y`) work only inside packages.
- Running a file inside a package as a script breaks relative imports. Use `python -m package.module`, or install the project in editable mode.
- `__init__.py` can re-export names, set `__version__`, and declare `__all__`, but should stay light and free of real logic.
- `__all__` lists the public names and controls `from package import *`. Type checkers use it to decide what counts as re-exported.
- **Namespace packages** have no `__init__.py` and can be assembled from several folders. A forgotten `__init__.py` creates one by accident, so include the file unless you want that behavior.
- Read data files shipped inside a package with `importlib.resources`, and read the version from installed metadata with `importlib.metadata`.
- Design a library with a small public API in `__init__.py`, implementation modules behind it, private underscore modules, and imports that flow in one direction only.
