## Overview

A **decorator** is a function that takes another function and returns a new one that **adds behavior around it**, without changing the original's code. You write `@something` on the line above a `def`, and from then on every call to that function goes through the extra behavior. Logging, timing, retrying, caching, access checks, and registering functions in a table are all done this way.

Decorators look like magic until you realize they are nothing more than the closures and first-class functions of the last two chapters, plus one line of syntax. Once you see that, they become one of the most practical tools in Python. Web frameworks, testing libraries, and the standard library are full of them.

By the end of this note you will be able to:

- Explain what `@decorator` really does.
- Write a basic decorator that works with any function signature.
- Preserve a function's name and docstring with `functools.wraps`.
- Write decorators that take arguments.
- Stack several decorators and predict the order.
- Avoid the common mistakes that make decorators fail silently.

> [!note] The key equivalence
> Writing `@decorator` on the line above `def func(): ...` means exactly the same as defining `func` normally and then writing `func = decorator(func)`.

## Core concepts

### The idea

A decorator receives a function, builds a **wrapper** function that calls the original and does something extra, and returns the wrapper. The name of the original function is then bound to the wrapper, so every caller uses the new behavior without knowing.

```python
def decorator(func):
    def wrapper(*args, **kwargs):
        # code before
        result = func(*args, **kwargs)
        # code after
        return result
    return wrapper
```

The wrapper takes `*args, **kwargs` so that it accepts **any** arguments and passes them on unchanged (Chapter 3.2). It returns the original's result, unless the decorator intends to alter it.

### The @ syntax

`@decorator` placed directly above a `def` is shorthand for `name = decorator(name)` after the function is created. It runs once, when the function is **defined**.

### Losing the function's identity

After decoration, the name refers to `wrapper`, so `func.__name__` is `"wrapper"` and the docstring is gone. Debugging, documentation, and tools that inspect functions are all confused. The fix is `functools.wraps`.

### functools.wraps

`functools.wraps(func)` is itself a decorator you place on the wrapper. It copies the original's name, docstring, module, and other metadata onto the wrapper, and records the original as `wrapper.__wrapped__`. **Always use it.**

```python
import functools

def decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### Decorators with arguments

Sometimes you want `@repeat(3)` instead of plain `@repeat`. Since `@repeat(3)` first **calls** `repeat(3)` and then uses the result as the decorator, you need **three layers**:

1. The outer function receives the decorator's arguments and returns the decorator.
2. The decorator receives the function and returns the wrapper.
3. The wrapper receives the call's arguments and runs the function.

```python
def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            ...
        return wrapper
    return decorator
```

### Stacking decorators

Several decorators can be placed above one function. They are applied from the **bottom up**, so the decorator closest to the function wraps it first:

```python
@outer
@inner
def func(): ...
```

is the same as `func = outer(inner(func))`. When the function is **called**, `outer`'s wrapper runs first, and calls `inner`'s wrapper, which calls the original.

### Decorators that return the original function

A decorator does not have to wrap anything. It can simply **register** the function somewhere and return it unchanged. This is how many web frameworks attach functions to URLs.

### Built-in and library decorators you already know

| Decorator | What it does |
| --- | --- |
| `@property`, `@staticmethod`, `@classmethod` | Change how a method behaves (Chapter 6.2) |
| `@dataclass` | Generates methods for a class (Chapter 6.7) |
| `@functools.lru_cache`, `@functools.cache` | Cache results (Chapter 3.7) |
| `@functools.wraps` | Copies metadata onto a wrapper |
| `@abc.abstractmethod` | Marks a method that subclasses must implement (Chapter 6.6) |

### Typical uses

| Use | What the wrapper does |
| --- | --- |
| Logging | Records each call and its result |
| Timing | Measures how long the call took |
| Retrying | Calls again after a failure |
| Caching | Returns a stored result for repeated arguments |
| Access control | Checks permissions before calling |
| Validation | Checks arguments before calling |
| Registration | Records the function in a table |
| Deprecation | Warns that the function is going away |

## Basic syntax examples

### A decorator without the @ sign

```python
def shout(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        return result.upper()
    return wrapper


def greet(name):
    return f"hello, {name}"


greet = shout(greet)
print(greet("ada"))
```

```text
HELLO, ADA
```

The name `greet` was rebound to the wrapper.

### The same with @

```python
@shout
def greet(name):
    return f"hello, {name}"


print(greet("ada"))
```

```text
HELLO, ADA
```

### What is lost without wraps

```python
@shout
def greet(name):
    """Return a greeting."""
    return f"hello, {name}"


print(greet.__name__)
print(greet.__doc__)
```

```text
wrapper
None
```

### The fix: functools.wraps

```python
import functools


def shout(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs).upper()
    return wrapper


@shout
def greet(name):
    """Return a greeting."""
    return f"hello, {name}"


print(greet.__name__)
print(greet.__doc__)
print(greet.__wrapped__("ada"))
```

```text
greet
Return a greeting.
hello, ada
```

`__wrapped__` is the original function, which is handy for calling the undecorated version in a test.

### Counting calls

```python
import functools


def count_calls(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        wrapper.calls += 1
        return func(*args, **kwargs)

    wrapper.calls = 0
    return wrapper


@count_calls
def ping():
    return "pong"


ping()
ping()
print(ping.calls)
```

```text
2
```

Functions are objects (Chapter 3.4), so you can attach attributes such as `calls` to the wrapper.

### A logging decorator

```python
import functools


def log_calls(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        arguments = ", ".join([*map(repr, args), *(f"{k}={v!r}" for k, v in kwargs.items())])
        print(f"calling {func.__name__}({arguments})")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result!r}")
        return result
    return wrapper


@log_calls
def add(a, b):
    return a + b


add(2, b=3)
```

```text
calling add(2, b=3)
add returned 5
```

### A decorator with arguments

```python
import functools


def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            result = None
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator


@repeat(3)
def say(message):
    print(message)


say("hello")
```

```text
hello
hello
hello
```

`@repeat(3)` calls `repeat(3)`, which returns `decorator`, which is then applied to `say`.

### Stacking decorators

```text
>>> def bold(func):
...     def wrapper():
...         return "<b>" + func() + "</b>"
...     return wrapper
...
>>> def italic(func):
...     def wrapper():
...         return "<i>" + func() + "</i>"
...     return wrapper
...
>>> @bold
... @italic
... def text():
...     return "hi"
...
>>> text()
'<b><i>hi</i></b>'
>>> @italic
... @bold
... def text2():
...     return "hi"
...
>>> text2()
'<i><b>hi</b></i>'
```

The decorator listed first (nearest the top) is the **outermost** layer.

### Registering functions

```python
COMMANDS = {}


def command(func):
    COMMANDS[func.__name__] = func
    return func            # returned unchanged


@command
def start():
    return "starting"


@command
def stop():
    return "stopping"


print(sorted(COMMANDS))
print(COMMANDS["stop"]())
```

```text
['start', 'stop']
stopping
```

### Decorators run at definition time

```python
def announce(func):
    print(f"decorating {func.__name__}")
    return func


@announce
def task():
    pass


print("module loaded")
```

```text
decorating task
module loaded
```

The decorator ran when `def task` executed, before anything called `task`.

### Forgetting the parentheses on a decorator with arguments

```python
@repeat
def say(message):
    print(message)


say("hi")
```

Nothing is printed and no error is reported. `@repeat` without parentheses passes `say` as the argument `times`, so `say` becomes the inner `decorator` function, and `say("hi")` merely builds a wrapper and throws it away. This is a quiet failure, and one reason to test decorators.

## Deeper behavior

### What `functools.wraps` really copies

`wraps` copies `__module__`, `__name__`, `__qualname__`, `__doc__`, `__annotations__`, and `__dict__` from the original function to the wrapper, and sets `wrapper.__wrapped__`. Tools such as `inspect.signature` follow `__wrapped__` to display the **original** signature, so documentation and editors still show the right parameters.

### Decorators and methods

Because the wrapper takes `*args, **kwargs`, it also works on methods: `self` is simply the first positional argument that gets passed along. This is why a decorator written for plain functions usually works on methods too.

### Where decorators add confusion

- **Extra frames in tracebacks.** An error inside the original function shows the wrapper above it. That is normal. Look for the line in your own function.
- **Hidden behavior.** A function with three decorators does a lot that is invisible at the call site. Keep decorators small and focused on one concern, and document what each does.
- **Order matters.** A permission check placed **below** a caching decorator never runs when the cache already holds a result. Think through which behavior should run first.
- **Async functions.** An `async def` function needs an `async` wrapper that awaits the original (Chapter 10.6). A synchronous wrapper would return the unawaited coroutine and skip the work.

### Typing decorators

A decorator that returns a wrapper with `*args, **kwargs` hides the original signature from type checkers. Python's `ParamSpec` (spelled `**P` in the modern syntax) lets a decorator declare "my wrapper accepts exactly the same parameters as the original and returns the same type". The industry-level snippet below uses it. Chapter 9.3 explains it fully.

### State inside decorators

Anything created in the **decorator** (not the wrapper) is shared by every call of that decorated function, which is how a decorator can hold a cache, a counter, or a lock. Each decorated function gets its own copy, because the decorator runs once per function:

```python
def remember(func):
    cache = {}                      # one dictionary per decorated function

    @functools.wraps(func)
    def wrapper(n):
        if n not in cache:
            cache[n] = func(n)
        return cache[n]

    return wrapper
```

Chapter 3.7 shows the ready-made version, `functools.cache`.

### Class decorators

A decorator can also be applied to a class: `@dataclass` takes a class, adds methods, and returns it (Chapter 6.7). The same equivalence holds: `@d class C: ...` means `C = d(C)`.

### Decorators and testing

To test a decorated function **without** its decoration, call `func.__wrapped__` (available because of `wraps`), or keep the core logic in an undecorated helper function and decorate a thin entry point.

### Optional-argument decorators

Some libraries allow both `@decorator` and `@decorator(option=...)`. They do this by checking whether the first argument is a function. It works, but it is more complex, so prefer one clear form unless you are writing a public library.

### Performance

A wrapper adds one extra function call per invocation. This is negligible for almost every use. Do not avoid decorators for speed unless profiling (Chapter 11.5) shows the call is in a very hot loop.

## Industry-level snippet

Calls to databases and other services fail temporarily all the time, and every important call is worth timing. Two decorators cover that: `retry`, which retries on selected exceptions with exponential backoff and logs each failure, and `timed`, which logs how long a call took even when it raises. Both use `functools.wraps` and are typed with `ParamSpec` (written `**P`) so type checkers still see the original signature. The `sleep` function is a parameter, so tests do not need to wait.

```python
"""Reusable decorators for backend code: retry with backoff, and timing."""

import functools
import logging
import time
from collections.abc import Callable

logger = logging.getLogger(__name__)


def retry[**P, R](
    max_attempts: int = 3,
    base_delay: float = 0.1,
    exceptions: tuple[type[Exception], ...] = (ConnectionError, TimeoutError),
    sleep: Callable[[float], None] = time.sleep,
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """Retry the decorated function when it raises one of the given exceptions.

    The delay doubles after each failure: base_delay, 2 * base_delay, and so on.
    The last failure is re-raised unchanged.

    Args:
        max_attempts: Total tries including the first. At least 1.
        base_delay: Seconds to wait after the first failure.
        exceptions: Exception types that justify another attempt.
        sleep: The waiting function. Replace it in tests.

    Raises:
        ValueError: If max_attempts or base_delay is invalid.
    """
    if max_attempts < 1:
        raise ValueError("max_attempts must be at least 1")
    if base_delay < 0:
        raise ValueError("base_delay must not be negative")

    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    if attempt == max_attempts:
                        logger.error(
                            "%s failed after %d attempts: %s",
                            func.__qualname__,
                            attempt,
                            exc,
                        )
                        raise
                    delay = base_delay * 2 ** (attempt - 1)
                    logger.warning(
                        "%s failed (attempt %d/%d): %s; retrying in %.2f s",
                        func.__qualname__,
                        attempt,
                        max_attempts,
                        exc,
                        delay,
                    )
                    sleep(delay)
            raise AssertionError("unreachable")  # satisfies type checkers

        return wrapper

    return decorator


def timed[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    """Log how long each call to func takes, including calls that raise."""

    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed_ms = (time.perf_counter() - start) * 1000
            logger.info("%s took %.1f ms", func.__qualname__, elapsed_ms)

    return wrapper
```

Try it:

```python
import logging
import sys

logging.basicConfig(level=logging.WARNING, format="%(levelname)s %(message)s", stream=sys.stdout)

attempts = {"count": 0}
delays: list[float] = []


@retry(max_attempts=3, base_delay=0.5, sleep=delays.append)
@timed
def fetch_profile(user_id: int) -> dict[str, int]:
    """Fetch a profile from a flaky service."""
    attempts["count"] += 1
    if attempts["count"] < 3:
        raise ConnectionError("connection reset")
    return {"user_id": user_id}


print(fetch_profile(7))
print(attempts["count"], delays)
print(fetch_profile.__name__, "|", fetch_profile.__doc__)
```

```text
WARNING fetch_profile failed (attempt 1/3): connection reset; retrying in 0.50 s
WARNING fetch_profile failed (attempt 2/3): connection reset; retrying in 1.00 s
{'user_id': 7}
3 [0.5, 1.0]
fetch_profile | Fetch a profile from a flaky service.
```

Setting the log level to `INFO` would also show a line such as `INFO fetch_profile took 0.0 ms` for each attempt, with a time that differs on every run.

Why it is written this way:

- `retry` is a decorator **factory**: three nested functions, as described in the core concepts. The outer one validates its settings once, when the decorator is applied, so a typo fails at import time instead of in the middle of a request.
- `functools.wraps` keeps the name and docstring, which is why the logs say `fetch_profile` and not `wrapper`.
- Only the listed exception types are retried. A `ValueError` from a bug would pass straight through, instead of being retried pointlessly.
- The final failure is **re-raised with a bare `raise`**, so the original traceback is preserved.
- `timed` uses `try/finally` so the duration is logged even if the function raises.
- Stacking `@retry` above `@timed` means every individual attempt is timed. Swapping them would time the whole retry sequence once.
- `ParamSpec` (`**P`) and `R` let a type checker see that `fetch_profile` still takes an `int` and returns a `dict`.

## Common mistakes

1. **Forgetting `return wrapper`** at the end of the decorator, so the decorated name becomes `None`.
2. **Forgetting to `return` the result of the original call** in the wrapper, so the decorated function always returns `None`.
3. **Not using `functools.wraps`,** losing the name, docstring, and signature information.
4. **Writing `@repeat` instead of `@repeat(3)`** for a decorator that takes arguments, which fails silently or confusingly.
5. **Writing `@repeat()` for a decorator that does not take arguments.** The extra call passes nothing and breaks it.
6. **Giving the wrapper a fixed signature** such as `def wrapper(x)` instead of `*args, **kwargs`, so it breaks other functions.
7. **Stacking decorators in the wrong order,** such as caching above a permission check, so the check is skipped.
8. **Wrapping an async function with a synchronous wrapper.** The coroutine is never awaited.
9. **Doing slow or failing work at decoration time** (import time) instead of inside the wrapper.
10. **Hiding too much behavior behind decorators.** A reader of the call site cannot see what happens.
11. **Swallowing exceptions in the wrapper** and returning a default value without logging.
12. **Catching `Exception` in a retry decorator by default.** Retrying bugs and invalid input wastes time. Retry only on errors that are genuinely temporary.

## Practice exercises

1. Write a decorator `announce` that prints `"Calling <name>"` before and `"Done"` after a function runs. Apply it to two functions with different parameters.
2. Write `timer` that prints how many milliseconds a function took, and use `time.sleep(0.05)` to test it. Make sure it keeps the original function's name and docstring.
3. Show what happens to `__name__` and `__doc__` if you leave out `functools.wraps`. Fix it and confirm.
4. Write `@require_positive` that checks that every positional argument is a positive number and raises `ValueError` otherwise. Test it on a `square_root` function.
5. Write `@repeat(times)` and apply it to a function that prints. Then deliberately write `@repeat` without parentheses and explain exactly why nothing happens.
6. Stack `@bold` and `@italic` (from this note) in both orders, and write down the order in which the wrappers run for each.
7. Write a registry decorator that records functions by name in a dictionary, and use it to build a command dispatcher that runs a command chosen by a string.
8. Write `@cache_results` that stores results in a private dictionary keyed by the arguments, and apply it to a slow function. Then compare with `functools.cache`.
9. Write a decorator `@deprecated("use new_function instead")` that issues a `DeprecationWarning` using the `warnings` module each time the function is called.
10. Write a decorator that works on both functions and methods, and show it on a class with a method.
11. Use `inspect.signature` on a decorated function with and without `wraps` and explain the difference.
12. Extend `retry` with a `jitter` parameter (a function returning a random multiplier between 0.5 and 1.5) so that clients do not all retry at the same instant. Write tests using a fake sleep and a fixed jitter function.

## Summary

- A decorator is a function that takes a function and returns a new function that adds behavior. `@decorator` above `def func` means `func = decorator(func)`, applied once when the function is defined.
- A basic decorator defines a `wrapper(*args, **kwargs)` that calls the original, does extra work, and returns the result. Always return the wrapper from the decorator, and the original's result from the wrapper.
- Use `@functools.wraps(func)` on the wrapper so the name, docstring, and signature information survive, and the original remains available as `__wrapped__`.
- A decorator that takes arguments, such as `@repeat(3)`, needs three nested layers: the factory, the decorator, and the wrapper.
- Stacked decorators apply from the bottom up. The top one is the outermost layer, and runs first when the function is called.
- A decorator can also just register a function and return it unchanged.
- Decorators are closures, they run at definition time, and anything created in the decorator (not the wrapper) is shared by every call.
- Keep decorators focused on one concern, think about order, remember that async functions need async wrappers, and type them with `ParamSpec` when you want type checkers to keep working.
