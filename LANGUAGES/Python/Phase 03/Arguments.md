## Overview

Chapter 3.1 showed the simplest way to pass data into a function. Real functions need more flexibility: optional settings with sensible defaults, arguments passed by name for clarity, any number of values, and signatures that stop callers from misusing them. Python's argument system gives you all of this, and designing function signatures well is a large part of designing good APIs.

By the end of this note you will be able to:

- Pass arguments by position and by keyword.
- Give parameters default values, and avoid the mutable default argument bug.
- Accept any number of arguments with `*args` and `**kwargs`.
- Force callers to use positional-only or keyword-only arguments.
- Unpack lists and dictionaries into arguments at the call site.
- Read the error messages Python gives for wrong calls.

> [!note] Your signature is a public promise
> Once other code (or other people) call your function, its parameter names, order, and defaults become a contract. Choosing them carefully now prevents painful changes later.

## Core concepts

### Positional arguments

Arguments matched to parameters **by position**: the first argument goes to the first parameter, and so on.

```python
def power(base, exponent):
    return base ** exponent


power(2, 3)      # base=2, exponent=3
```

### Keyword arguments

Arguments matched **by name** using `name=value`. Order does not matter, and the call documents itself:

```python
power(exponent=3, base=2)
```

You can mix both styles in one call, but **positional arguments must come first**.

### Default values

A parameter with a default becomes optional. The default is used when the caller omits the argument:

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"
```

Parameters **with** defaults must come **after** those without.

### Mutable defaults are shared

A default value is created **once**, when the `def` statement runs, not on every call. For an immutable value like a number or string, this does not matter. For a list, dictionary, or set, every call that omits the argument shares the same object, which is a classic bug (Chapter 2.5). The standard pattern is to use `None` as the default and create the real value inside:

```python
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### Variable positional arguments: `*args`

A parameter written `*args` collects **any extra positional arguments** into a **tuple**. The name `args` is a convention, the star is what matters.

### Variable keyword arguments: `**kwargs`

A parameter written `**kwargs` collects **any extra keyword arguments** into a **dictionary**. Again, the name is a convention.

### Positional-only and keyword-only parameters

Two special markers in the parameter list restrict how arguments can be passed:

| Marker | Effect | Example |
| --- | --- | --- |
| `/` | Parameters **before** it can only be passed by position | `def f(a, b, /)` |
| `*` | Parameters **after** it can only be passed by keyword | `def f(a, *, b)` |

Keyword-only parameters make calls readable: `connect(host, port=5432)` is clearer than `connect(host, 5432)`. Positional-only parameters let you rename a parameter later without breaking callers, and are used by many built-in functions.

### The full parameter order

When you combine all the kinds, they must appear in this order:

```text
def f(positional_only, /, normal, with_default=1, *args, keyword_only, kw_default=2, **kwargs):
```

Most functions use only a small part of this. The important rule: a bare `*` or `*args` begins the keyword-only section.

### Unpacking at the call site

The same stars work when **calling** a function:

| Call | Meaning |
| --- | --- |
| `f(*sequence)` | Pass the items of any iterable as separate positional arguments |
| `f(**mapping)` | Pass the key-value pairs of a mapping as keyword arguments |

You can use both in one call, and more than one of each.

### Common errors

| Mistake | Error message pattern |
| --- | --- |
| Too few arguments | `missing 1 required positional argument: 'name'` |
| Too many arguments | `takes from 1 to 2 positional arguments but 3 were given` |
| Misspelled keyword | `got an unexpected keyword argument 'nme'` |
| Same parameter twice | `got multiple values for argument 'name'` |
| Keyword for positional-only | `got some positional-only arguments passed as keyword arguments` |
| Positional for keyword-only | `takes 1 positional argument but 2 were given` |

## Basic syntax examples

### Positional, keyword, and default arguments

```text
>>> def greet(name, greeting="Hello"):
...     return f"{greeting}, {name}!"
...
>>> greet("Ada")
'Hello, Ada!'
>>> greet("Ada", "Hi")
'Hi, Ada!'
>>> greet(name="Ada", greeting="Hey")
'Hey, Ada!'
>>> greet(greeting="Hey", name="Ada")
'Hey, Ada!'
>>> greet("Ada", greeting="Yo")
'Yo, Ada!'
```

### What a bad call looks like

```text
>>> def greet(name, greeting="Hello"):
...     return f"{greeting}, {name}!"
...
>>> greet()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: greet() missing 1 required positional argument: 'name'
>>> greet("Ada", "Hi", "extra")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: greet() takes from 1 to 2 positional arguments but 3 were given
>>> greet("Ada", nme="x")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: greet() got an unexpected keyword argument 'nme'
>>> greet("Ada", name="Grace")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: greet() got multiple values for argument 'name'
```

### `*args`

```text
>>> def total(*numbers):
...     return sum(numbers)
...
>>> total(1, 2, 3)
6
>>> total()
0
>>> def show(*args, **kwargs):
...     return args, kwargs
...
>>> show(1, 2, a=3)
((1, 2), {'a': 3})
>>> show()
((), {})
```

Inside the function, `numbers` is an ordinary tuple, and `kwargs` an ordinary dictionary. Both are empty when no extra arguments are given.

### A function with required, optional, and variable arguments

```python
def log_message(level, message, *details, sep=" | "):
    parts = [message, *details]
    return f"[{level}] " + sep.join(parts)


print(log_message("INFO", "started"))
print(log_message("WARN", "slow request", "path=/api", "ms=900"))
print(log_message("WARN", "slow request", "path=/api", sep=" / "))
```

```text
[INFO] started
[WARN] slow request | path=/api | ms=900
[WARN] slow request / path=/api
```

The parameter `sep` comes after `*details`, so it is keyword-only. Without that, it would be swallowed into `details`.

### Unpacking at the call site

```text
>>> def greet(name, greeting="Hello"):
...     return f"{greeting}, {name}!"
...
>>> def total(*numbers):
...     return sum(numbers)
...
>>> values = [1, 2, 3]
>>> total(*values)
6
>>> total(*values, *[10, 20])
36
>>> options = {"greeting": "Yo"}
>>> greet("Ada", **options)
'Yo, Ada!'
>>> pair = ("Grace", "Welcome")
>>> greet(*pair)
'Welcome, Grace!'
```

Unpacking is how you hand a list of arguments you already hold to a function that expects them separately.

### Positional-only and keyword-only

```text
>>> def power(base, exponent, /):
...     return base ** exponent
...
>>> power(2, 3)
8
>>> power(base=2, exponent=3)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: power() got some positional-only arguments passed as keyword arguments: 'base, exponent'
>>> def connect(host, *, port=5432, timeout=5):
...     return host, port, timeout
...
>>> connect("db", port=6543)
('db', 6543, 5)
>>> connect("db", 6543)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: connect() takes 1 positional argument but 2 were given
```

### The mutable default trap

```python
def add_item(item, items=[]):
    items.append(item)
    return items


print(add_item("a"))
print(add_item("b"))
```

```text
['a']
['a', 'b']
```

The second call did not start with an empty list: it reused the list created when `def` ran. The fix with `None`:

```python
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items


print(add_item("a"))
print(add_item("b"))
```

```text
['a']
['b']
```

### Defaults are evaluated once

```python
import time


def stamp(when=time.time()):
    return when


first = stamp()
time.sleep(0.01)
second = stamp()
print(first == second)
```

```text
True
```

The time was taken when the function was **defined**, so every call returns the same value. If you want "the time of the call", use `None` and read the clock inside the function.

### Distinguishing "not given" from `None`

Sometimes `None` is itself a valid value for a parameter. Use a unique placeholder object, called a **sentinel**, as the default:

```python
_MISSING = object()


def get_setting(config, key, default=_MISSING):
    if key in config:
        return config[key]
    if default is _MISSING:
        raise KeyError(f"setting {key!r} is required")
    return default


print(get_setting({"timeout": None}, "timeout", default=5))
print(get_setting({}, "timeout", default=None))
```

```text
None
None
```

A call without a default raises a `KeyError`, while `default=None` is accepted as a real answer.

## Deeper behavior

### How Python matches arguments to parameters

When you call a function, Python binds arguments to parameters in a fixed order:

1. Positional arguments fill positional parameters from left to right. Any left over go into `*args`, if present, otherwise it is an error.
2. Keyword arguments are matched to parameters by name. A keyword that matches an already-filled parameter is an error ("multiple values"). A keyword that matches no parameter goes into `**kwargs`, if present, otherwise it is an error.
3. Defaults fill any remaining parameters. A parameter with no value and no default is an error ("missing").

### Seeing a signature

The `inspect` module prints a function's signature, which is useful when reading unfamiliar code:

```python
import inspect


def f(a, b, /, c, d=1, *args, e, g=2, **kwargs): ...


print(inspect.signature(f))
```

```text
(a, b, /, c, d=1, *args, e, g=2, **kwargs)
```

### `*args` is a tuple and `**kwargs` is a dictionary

Each call builds a fresh tuple and dictionary, so modifying them does not affect the caller. Keyword arguments keep the order in which they were written.

### Forwarding arguments

A function that wraps another one without knowing its exact signature can pass everything along:

```python
def call_with_logging(func, *args, **kwargs):
    print(f"calling {func.__name__}")
    return func(*args, **kwargs)
```

This pattern is the heart of decorators (Chapter 3.6).

### `**kwargs` hides typos

A function that accepts `**kwargs` and does not use them all silently ignores misspelled options. `connect(timout=5)` would be accepted and have no effect. Prefer explicit keyword-only parameters, which turn a typo into an immediate `TypeError`. Use `**kwargs` only when you genuinely forward unknown options to something else.

### Duplicate keys in `**` unpacking

```text
>>> def f(**kwargs):
...     return kwargs
...
>>> f(**{"a": 1}, **{"a": 2})
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: f() got multiple values for keyword argument 'a'
```

This differs from building a dictionary with `{**a, **b}`, where the later value silently wins.

### API design guidelines

| Guideline | Why |
| --- | --- |
| Make optional settings **keyword-only** | `send(msg, retries=3)` is self-explanatory, `send(msg, 3)` is not |
| Avoid positional booleans | `render(page, True, False)` tells the reader nothing |
| Put the most important, always-required parameters first | Callers can leave off the rest |
| Use `None` or a sentinel for mutable or computed defaults | Avoids shared state |
| Keep the parameter count small | More than four usually means you should group them into an object |
| Use positional-only for parameters whose names are not meaningful | Frees you to rename them |
| Keep defaults immutable | Numbers, strings, tuples, `None`, `True`, `False` |

When a function needs many related options, bundle them into a dataclass (Chapter 6.7) and pass one object.

### Argument unpacking works with any iterable

`f(*range(3))`, `f(*"ab")`, and `f(*some_set)` all work. The order for a set is not defined, so avoid it there.

### Typing keyword arguments

Type hints for `*args` and `**kwargs` describe each element: `*args: int` means "any number of integers" and `**kwargs: str` means "string values". For heterogeneous options, Chapter 8 introduces more precise tools such as `TypedDict` and `Unpack`.

## Industry-level snippet

Services build URLs and request options constantly. This module defines a URL builder with a positional-only base, variable path segments, and keyword query parameters, and a validated options factory with keyword-only parameters and safe defaults. It rejects path segments that could escape the intended path, and never uses a mutable default.

```python
"""Building URLs and request options with a clear, safe signature."""

import logging
from collections.abc import Mapping
from dataclasses import dataclass
from urllib.parse import quote, urlencode

logger = logging.getLogger(__name__)

DEFAULT_TIMEOUT_SECONDS = 10.0
MAX_RETRIES = 5


@dataclass(frozen=True)
class RequestOptions:
    """Validated settings for an outgoing HTTP request."""

    timeout: float
    retries: int
    headers: dict[str, str]


def build_url(
    base_url: str, /, *segments: str | int, **query: str | int | bool
) -> str:
    """Join path segments onto a base URL and append a query string.

    Args:
        base_url: The service root, such as "https://api.example.com/v1".
        *segments: Path pieces. Each is URL-encoded as a single segment.
        **query: Query parameters. Booleans become "true" or "false".

    Returns:
        The complete URL.

    Raises:
        ValueError: If a segment is empty, "." or "..".
    """
    encoded: list[str] = []
    for segment in segments:
        text = str(segment).strip("/")
        if text in {"", ".", ".."}:
            raise ValueError(f"invalid path segment: {segment!r}")
        encoded.append(quote(text, safe=""))  # a "/" inside a segment becomes %2F

    url = base_url.rstrip("/")
    if encoded:
        url += "/" + "/".join(encoded)
    if query:
        pairs = {
            name: str(value).lower() if isinstance(value, bool) else value
            for name, value in query.items()
        }
        url += "?" + urlencode(pairs)
    return url


def make_options(
    *,
    timeout: float = DEFAULT_TIMEOUT_SECONDS,
    retries: int = 0,
    headers: Mapping[str, str] | None = None,
) -> RequestOptions:
    """Create validated request options. All arguments are keyword-only.

    Raises:
        ValueError: If timeout or retries is out of range.
    """
    if timeout <= 0:
        raise ValueError("timeout must be greater than zero")
    if not 0 <= retries <= MAX_RETRIES:
        raise ValueError(f"retries must be between 0 and {MAX_RETRIES}")
    options = RequestOptions(
        timeout=timeout,
        retries=retries,
        headers=dict(headers) if headers is not None else {},  # a fresh dictionary
    )
    logger.debug("Created request options: timeout=%s retries=%s", timeout, retries)
    return options
```

Try it:

```python
print(build_url("https://api.example.com/v1/", "users", 42, "orders",
                status="open", limit=10, expand=True))
print(build_url("https://api.example.com", "files", "a/b report.pdf"))

try:
    build_url("https://api.example.com", "users", "..", "admin")
except ValueError as exc:
    print("ValueError:", exc)

print(make_options(retries=2, headers={"Accept": "application/json"}))

try:
    make_options(5)
except TypeError as exc:
    print("TypeError:", exc)
```

```text
https://api.example.com/v1/users/42/orders?status=open&limit=10&expand=true
https://api.example.com/files/a%2Fb%20report.pdf
ValueError: invalid path segment: '..'
RequestOptions(timeout=10.0, retries=2, headers={'Accept': 'application/json'})
TypeError: make_options() takes 0 positional arguments but 1 was given
```

Why it is written this way:

- `base_url` is positional-only (`/`), because its name carries no meaning for the caller and the function may rename it later.
- `*segments` accepts any number of path pieces, and `**query` accepts arbitrary parameters, which is exactly what those arguments are: open-ended lists of things.
- `make_options` is entirely keyword-only (`*` first), so call sites always read `make_options(retries=2)`, and a mistaken positional call fails immediately.
- `headers` defaults to `None`, and a **fresh dictionary** is built inside, so two option objects never share state.
- Rejecting `".."` and encoding slashes inside a segment prevents a caller-supplied value from escaping the intended URL path.

## Common mistakes

1. **Using a mutable default** such as `items=[]` or `options={}`. Use `None` and create the object inside.
2. **Putting a parameter with a default before one without.** It is a syntax error.
3. **Passing positional arguments after keyword arguments** in a call.
4. **Misspelling a keyword name,** or relying on `**kwargs` and silently ignoring the typo.
5. **Forgetting the star when unpacking.** `total(values)` passes one list, `total(*values)` passes its items.
6. **Using `*args` when a list parameter would be clearer.** If callers usually already have a list, accept the list.
7. **Boolean flag arguments passed positionally,** making calls unreadable.
8. **Giving a function too many parameters.** Group related ones into an object.
9. **Evaluating something expensive or time-dependent in a default,** such as `when=time.time()`.
10. **Assuming `**kwargs` keeps unknown options out of the way.** Validate them or forward them deliberately.
11. **Changing the name of a parameter** in a public function without realizing callers use it as a keyword.
12. **Passing the same keyword twice** through `**` unpacking.

## Practice exercises

1. Write `describe_person(name, age, city="unknown")` and call it using positional arguments, keyword arguments, and a mix. Then try three wrong calls and read each error.
2. Write `average(*numbers)` that accepts any number of values and returns their mean, raising a clear error if none are given. Call it with a list using unpacking.
3. Write `format_name(first, last, *, middle=None, title=None)` and make sure that `middle` and `title` can only be passed by keyword. Show the error from a positional attempt.
4. Reproduce the mutable default bug with a function that collects tags, then fix it with `None`.
5. Write `join_path(*parts, sep="/")` that joins any number of parts, trimming extra separators at the joins.
6. Write a function `make_user(**fields)` that accepts any fields but requires `username` and rejects any field name that is not in an allowed set, listing the unknown ones in the error.
7. Write `describe(a, b, /, c, *, d)` and call it in every valid way you can find. List which calls are errors and why.
8. Use `inspect.signature` on three built-in or standard-library functions (for example `round`, `sorted`, and `max`) and explain each piece of what you see.
9. Write a wrapper `timed_call(func, *args, **kwargs)` that returns both the function's result and the seconds it took, forwarding all arguments unchanged.
10. Create a function with a sentinel default so that `None` is a legitimate value, and demonstrate three calls: omitted, `None`, and a real value.
11. Redesign this signature to be safer and clearer, and explain each change: `def send(message, to, True, False, 3)` where the unnamed values mean "urgent", "encrypt", and "retries".
12. Extend `build_url` so that a query value that is a list produces repeated parameters (`tag=a&tag=b`), and a value of `None` is omitted. Write tests for both cases, and for an empty query.

## Summary

- Arguments can be passed by **position** or by **keyword**. Positional arguments must come before keyword arguments in a call.
- A **default value** makes a parameter optional. Parameters with defaults go after those without.
- Defaults are evaluated **once**, at definition time. Never use a mutable default. Use `None` (or a sentinel object) and create the real value inside the function.
- `*args` collects extra positional arguments into a tuple. `**kwargs` collects extra keyword arguments into a dictionary.
- A bare `*` makes every parameter after it keyword-only. A `/` makes every parameter before it positional-only.
- At a call site, `*iterable` and `**mapping` unpack into separate arguments.
- Learn the common `TypeError` messages: missing arguments, too many arguments, unexpected keyword, multiple values.
- Good signatures make options keyword-only, avoid positional booleans, keep the parameter count low, and keep defaults immutable.
- A function's signature is a contract with its callers, so choose names and order with care.
