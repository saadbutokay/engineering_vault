## Overview

When you use a name in your code, how does Python know which variable you mean? If two functions both have a variable called `total`, why do they not interfere? The answer is **scope**: the region of a program in which a name is visible. Understanding scope explains many confusing errors, including the `UnboundLocalError` you met in Chapter 1.7, and it leads directly to one of the most important design rules in programming: avoid hidden shared state.

By the end of this note you will be able to:

- Explain what scope is and why it exists.
- Apply the LEGB rule to predict which variable a name refers to.
- Use `global` and `nonlocal`, and know when not to.
- Explain why loop variables, `if` blocks, and comprehensions behave the way they do.
- Describe why global state makes programs harder to test and maintain, and what to do instead.

> [!note] Scope in one sentence
> A name created inside a function belongs to that function, and disappears when the function returns. Names created at the top level of a file belong to the file (the module).

## Core concepts

### What scope means

A **scope** is a region of code in which a name is defined and can be used. Scopes isolate parts of a program from each other. They let you write a function with a local variable called `result` without worrying that some other function also uses `result`.

### Local scope

Any name **assigned** inside a function (including its parameters) is **local** to that function. It is created when the function is called and discarded when it returns. Code outside the function cannot see it.

### Global scope

Names assigned at the top level of a file, outside any function or class, are **global** to that module. Functions in the same file can **read** them.

### Enclosing scope

When a function is defined inside another function, the inner function can read the names of the outer one. The outer function's local names are called the **enclosing scope** from the inner function's point of view (Chapter 3.5).

### Built-in scope

Python's built-in names such as `print`, `len`, `range`, and `int` live in a special outermost scope that is always available.

### The LEGB rule

When Python looks up a name, it searches the scopes in this order and stops at the first match:

| Letter | Scope | Where |
| --- | --- | --- |
| L | Local | The current function |
| E | Enclosing | Any outer functions, from nearest to farthest |
| G | Global | The module's top level |
| B | Built-in | Python's pre-defined names |

If the name is found nowhere, you get a `NameError`.

### Assignment decides locality

This is the rule behind most scope surprises: **if a function assigns to a name anywhere in its body, that name is local to the whole function**, even in lines before the assignment. Python decides this when it compiles the function, not when it runs.

### The `global` statement

`global name` inside a function says: "when I assign to `name`, I mean the module-level variable, not a new local." It is only needed to **rebind** a global name. Reading a global, or modifying a mutable global object in place (such as calling `.append()`), needs no declaration.

### The `nonlocal` statement

`nonlocal name` inside an inner function says: "when I assign to `name`, I mean the variable in the nearest enclosing function." It is how a closure updates a variable it captured (Chapter 3.5). It cannot refer to a global.

### Which constructs create a scope?

| Construct | Creates a new scope? |
| --- | --- |
| Function (`def`, `lambda`) | Yes |
| Class body | Yes (a special one) |
| Module (a file) | Yes, the global scope |
| Comprehension and generator expression | Yes |
| `if`, `for`, `while`, `try`, `with` blocks | **No** |

Python has function scope, not block scope. A variable created inside an `if` or `for` block is still visible after it.

## Basic syntax examples

### Local versus global

```python
x = "global"


def show():
    x = "local"
    print("inside:", x)


show()
print("outside:", x)
```

```text
inside: local
outside: global
```

The assignment inside `show` created a new local `x`. The global `x` was never touched.

### Reading a global

```python
tax_rate = 0.2


def add_tax(amount):
    return amount * (1 + tax_rate)


print(add_tax(100))
```

```text
120.0
```

`tax_rate` is not local to `add_tax`, so Python found it in the global scope. Reading is allowed.

### LEGB in action

```python
level = "global"


def outer():
    level = "enclosing"

    def inner():
        level = "local"
        print("inner sees:", level)

    inner()
    print("outer sees:", level)


outer()
print("module sees:", level)
```

```text
inner sees: local
outer sees: enclosing
module sees: global
```

Each of the three assignments created a separate variable in a separate scope. Delete the line `level = "local"` and `inner` would find the enclosing value instead.

### The built-in scope

```python
print(len("abc"))
```

`len` is not defined anywhere in your file. Python found it in the built-in scope, the last stop.

### The assignment-makes-it-local trap

```python
count = 0


def increment():
    count = count + 1       # tries to read the local count before it has a value


increment()
```

```text
UnboundLocalError: cannot access local variable 'count' where it is not associated with a value
```

Because the function assigns to `count`, Python treats `count` as local everywhere inside it, so the right-hand side reads a local that does not exist yet.

### Fixing it with `global`

```python
count = 0


def increment():
    global count
    count += 1


increment()
increment()
print(count)
```

```text
2
```

This works, but it is usually a sign that you should pass the value in and return the new one:

```python
def increment(count):
    return count + 1


count = 0
count = increment(count)
count = increment(count)
print(count)
```

```text
2
```

### Mutating a global object needs no declaration

```python
visited = []


def record(page):
    visited.append(page)      # modifies the object, does not rebind the name


record("/home")
record("/docs")
print(visited)
```

```text
['/home', '/docs']
```

No `global` was needed, because `visited` was never assigned. This is legal but is also hidden shared state, discussed below.

### `nonlocal`

```python
def make_counter():
    count = 0

    def increment():
        nonlocal count
        count += 1
        return count

    return increment


counter = make_counter()
print(counter(), counter(), counter())
```

```text
1 2 3
```

`increment` updates the `count` that belongs to `make_counter`. Without the `nonlocal` line, you would get the same `UnboundLocalError` as before.

### Function scope, not block scope

```text
>>> if True:
...     message = "set inside the if"
...
>>> message
'set inside the if'
>>> for j in range(3):
...     pass
...
>>> j
2
```

### Comprehensions have their own scope

```text
>>> [i for i in range(3)]
[0, 1, 2]
>>> i
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'i' is not defined
```

The comprehension's loop variable does not leak into the surrounding code, unlike a plain `for` loop.

### Shadowing a built-in

```text
>>> list = [1, 2, 3]
>>> list("abc")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'list' object is not callable
>>> del list
>>> list("abc")
['a', 'b', 'c']
```

Assigning to the name `list` created a global that **shadows** the built-in, so LEGB found it first. Deleting it exposes the built-in again.

### Looking at scopes directly

```python
total = 10


def demo(a, b):
    c = a + b
    print(sorted(locals()))


demo(1, 2)
print("total" in globals())
```

```text
['a', 'b', 'c']
True
```

`locals()` and `globals()` return dictionaries of the current local and global names. They are handy for exploration and rarely appear in production code.

## Deeper behavior

### Scope is decided when the function is compiled

When Python compiles a function, it scans the body. Every name that is assigned anywhere in it (including by `for`, `import`, `with ... as`, and `def`) is marked local. Names that are only read are looked up through the enclosing, global, and built-in scopes at run time. That is why the `UnboundLocalError` appears even though the global `count` exists.

### `global` and `nonlocal` are about rebinding only

You need these statements only to **rebind** a name (`x = ...`, `x += ...`, `del x`). Calling methods such as `.append()` on a global or enclosing list, or changing items with `data["key"] = value`, modifies an existing object and works without declarations.

### The exception variable disappears

```python
try:
    1 / 0
except ZeroDivisionError as exc:
    pass

try:
    print(exc)
except NameError as error:
    print("NameError:", error)
```

```text
NameError: name 'exc' is not defined
```

The name after `as` in an `except` clause is deleted at the end of the block, to avoid keeping the traceback alive. If you need the exception afterwards, copy it to another name inside the block.

### Modules have their own global scope

"Global" means global **to one module**, not to the whole program. A variable in `config.py` is not visible in `main.py` unless you import it. There is no truly program-wide variable in Python, which is a good thing.

### Imports create names

`import math` creates the name `math` in the current scope. `from math import sqrt` creates `sqrt`. Imports inside a function create local names. Importing at the top of the file is the normal style.

### Local lookups are faster

Reading a local variable is quicker than reading a global or a built-in, because locals are stored in a fixed array while globals need a dictionary lookup. The difference is tiny and rarely matters, but it explains why some performance-sensitive code copies a global into a local first.

### Why global state is a problem

Global variables that **change** make programs hard to reason about:

| Problem | Why it hurts |
| --- | --- |
| Hidden dependencies | A function's behavior depends on something that does not appear in its signature |
| Hard to test | Tests must set up and clean up shared state, and can affect each other |
| Order-dependent bugs | The result depends on which code ran first |
| Concurrency hazards | Several threads or requests changing the same variable cause races (Chapter 10.2) |
| Hard to reuse | A function tied to a global cannot be used in a different context |

### What to do instead

| Instead of... | Use... |
| --- | --- |
| A changing global counter | A parameter and return value, or an object that owns the state |
| A global configuration read everywhere | Load it once, then **pass it** to the functions that need it |
| A global cache dictionary | `functools.lru_cache` (Chapter 3.7) or an object with a clear lifetime |
| Modifying a global list from many places | A function that returns the data, or a class (Chapter 6.1) |

### Globals that are acceptable

- **Constants**: `MAX_RETRIES = 5`, never reassigned.
- **A module logger**: `logger = logging.getLogger(__name__)`.
- **Registries filled at import time**, such as a table of handlers.
- **A lazily created shared resource**, such as a connection pool or settings object, accessed through a function.

### `global` for lazy initialization

A legitimate use of `global` is creating an expensive shared object on first use, behind a function that hides it. The industry-level snippet below shows the pattern, along with how to keep the rest of the code independent of it.

### Class scope is special

Names defined in a class body live in the class's own namespace. Methods inside the class cannot see class-level names as bare variables. They must write `self.name` or `ClassName.name`. Chapter 6.1 explains this.

## Industry-level snippet

Settings are the most common legitimate "global" in a backend: loaded once from the environment, needed in many places. The wrong design lets every function reach for the global. The better design loads settings behind one function, but passes the settings object **into** the business logic, so the logic stays pure and testable. This module shows both halves.

```python
"""Application settings loaded once, and passed explicitly to business logic."""

import logging
import os
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation

logger = logging.getLogger(__name__)

DEFAULT_TAX_RATE = "0.20"
DEFAULT_CURRENCY = "EUR"


class SettingsError(Exception):
    """Raised when the environment holds invalid settings."""


@dataclass(frozen=True)
class Settings:
    """Immutable application settings."""

    tax_rate: Decimal
    currency: str


_settings: Settings | None = None  # cache, accessed only through get_settings()


def load_settings() -> Settings:
    """Read and validate settings from environment variables."""
    raw_rate = os.environ.get("TAX_RATE", DEFAULT_TAX_RATE)
    try:
        tax_rate = Decimal(raw_rate)
    except InvalidOperation as exc:
        raise SettingsError(f"TAX_RATE must be a number, got {raw_rate!r}") from exc
    if not Decimal("0") <= tax_rate < Decimal("1"):
        raise SettingsError(f"TAX_RATE must be in [0, 1), got {tax_rate}")
    currency = os.environ.get("CURRENCY", DEFAULT_CURRENCY).strip().upper()
    if len(currency) != 3 or not currency.isalpha():
        raise SettingsError(f"CURRENCY must be a 3-letter code, got {currency!r}")
    return Settings(tax_rate=tax_rate, currency=currency)


def get_settings() -> Settings:
    """Return the shared settings, loading them on first use."""
    global _settings
    if _settings is None:
        _settings = load_settings()
        logger.info("Settings loaded: tax_rate=%s currency=%s",
                    _settings.tax_rate, _settings.currency)
    return _settings


def reset_settings() -> None:
    """Forget the cached settings. Intended for tests."""
    global _settings
    _settings = None


def price_with_tax(net_cents: int, settings: Settings) -> int:
    """Return the gross price in cents. Pure: everything it needs is a parameter."""
    if net_cents < 0:
        raise ValueError("net_cents must not be negative")
    gross = Decimal(net_cents) * (1 + settings.tax_rate)
    return int(gross.to_integral_value())
```

Try it:

```python
import os

os.environ["TAX_RATE"] = "0.25"
settings = get_settings()
print(settings)
print(price_with_tax(1000, settings))

# The business logic does not need the global: tests can build their own settings.
test_settings = Settings(tax_rate=Decimal("0.10"), currency="USD")
print(price_with_tax(1000, test_settings))

os.environ["TAX_RATE"] = "abc"
reset_settings()
try:
    get_settings()
except SettingsError as exc:
    print("SettingsError:", exc)
```

```text
Settings(tax_rate=Decimal('0.25'), currency='EUR')
1250
1100
SettingsError: TAX_RATE must be a number, got 'abc'
```

Why it is written this way:

- The shared state is **one private variable**, `_settings`, touched by exactly two functions that use `global`. Nothing else in the program mentions it.
- `Settings` is a frozen dataclass, so even though it is shared, nobody can change it. Sharing immutable data is safe.
- `price_with_tax` is **pure**: it takes `settings` as a parameter instead of reading the global. It can be tested with any settings, in any order, without environment variables.
- `reset_settings` exists so that tests can start from a clean state.
- Validation happens once, at load time, with clear errors, so the program fails at startup rather than in the middle of a request.

## Common mistakes

1. **Assigning to a global inside a function without `global`,** which creates a new local and leaves the global unchanged.
2. **Reading a variable and assigning to it in the same function** (`count = count + 1`), causing `UnboundLocalError`.
3. **Using `global` to avoid passing arguments.** It trades a small convenience for hidden dependencies.
4. **Assuming `if` or `for` blocks have their own scope.** Names defined inside them stay visible afterwards.
5. **Expecting a comprehension variable to exist after the comprehension.** It does not.
6. **Shadowing built-ins** such as `list`, `dict`, `str`, `id`, `input`, `sum`, and then wondering why they stop working.
7. **Using `nonlocal` for a global variable.** It only works with enclosing function scopes.
8. **Forgetting that mutating a global list or dictionary affects every user of it.**
9. **Relying on a variable from a previous loop** when the loop may not have run at all.
10. **Reusing the same variable name at several levels** so that it is unclear which one is meant.
11. **Using a module-level mutable object as a cache or counter in code that runs in several threads,** without any protection.
12. **Using `globals()` or `locals()` to create variables dynamically.** Use a dictionary instead.

## Practice exercises

1. Predict the output of this program, then run it and explain each line using LEGB:

   ```python
   x = 1

   def f():
       x = 2

       def g():
           print(x)

       g()

   f()
   print(x)
   ```

2. Write a function that reads a global constant and one that tries to modify a global number without `global`. Trigger the error and explain it.
3. Fix the broken `increment` function in two ways: with `global`, and by passing the value and returning the new one. State which you prefer and why.
4. Write `make_counter(start=0, step=1)` that returns a function which increases and returns an internal count each time it is called. Make two counters and show that they are independent.
5. Show that a plain `for` loop variable survives the loop and a comprehension variable does not. Explain what happens to the loop variable if the loop iterates over an empty list.
6. Shadow the built-in `max` by accident in a program, observe the failure, and then repair it two ways.
7. Write a function that appends to a global list, and another that rebinds the global list to a new list. Explain which needs `global` and why.
8. Use `locals()` inside a function to print the names of its local variables, and `globals()` to check whether a given name exists at module level.
9. Find all the global variables in a small program of your own, and for each one decide whether it is a constant, a shared mutable state, or something that should be passed as a parameter.
10. Refactor a program that uses three global counters (`requests`, `errors`, `bytes_sent`) into one that keeps them in a single dictionary or dataclass passed explicitly to the functions that update them.
11. Demonstrate that the `except ... as e` variable disappears after the block, and show a safe way to keep the exception for later.
12. Extend the settings module with a `with_overrides(settings, **changes)` function that returns a new `Settings` object with some fields changed, without modifying the original and without touching the global cache. Write tests showing that both `get_settings()` and the original object are unaffected.

## Summary

- A **scope** is the region where a name is visible. Python looks names up using the **LEGB** order: Local, Enclosing, Global, Built-in.
- Names assigned in a function are local to it. Assignment anywhere in the function makes the name local throughout, which causes `UnboundLocalError` if you read it first.
- A function may **read** globals freely. To **rebind** a global, declare it with `global`. To rebind a variable of an enclosing function, declare it with `nonlocal`. Modifying a mutable object in place needs neither.
- Python has function scope, not block scope: `if`, `for`, and `while` blocks do not create scopes. Comprehensions and generator expressions do.
- The variable after `as` in an `except` clause is deleted when the block ends.
- Shadowing a built-in hides it for that scope. Do not reuse built-in names.
- Changing global state creates hidden dependencies and makes code hard to test and unsafe under concurrency. Prefer parameters and return values, objects that own their state, and explicit configuration objects.
- Constants, loggers, registries, and carefully wrapped lazy singletons are acceptable globals. Keep the business logic pure by passing dependencies in.
