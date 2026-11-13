## Overview

Chapter 3.4 showed that a function can build and return another function. This chapter explains the mechanism that makes that useful: the returned function **remembers** the variables of the function that created it, even after that function has finished. A function bundled together with the variables it remembers is called a **closure**.

Closures are how Python functions can carry configuration and private state without a class. They power decorators (Chapter 3.6), callbacks, caches, counters, and function factories. They also contain one famous trap, the **late-binding pitfall**, which this note shows you how to recognize and fix.

By the end of this note you will be able to:

- Explain what a free variable and a closure are.
- Build function factories that produce configured functions.
- Keep private state in a closure with `nonlocal`.
- Predict and fix the late-binding bug in loops.
- Choose between a closure and a class for holding state.

> [!note] The one-sentence version
> A closure is a function plus the surrounding variables it uses. It keeps those variables alive after the outer function has returned.

## Core concepts

### Nested functions

A function can be defined **inside** another function. The inner function is created each time the outer function runs, and is only visible inside it unless the outer function returns it.

### Free variables

When an inner function uses a name that is neither its own parameter nor assigned locally, but belongs to an enclosing function, that name is a **free variable**. Python looks it up through the enclosing scopes (the "E" in LEGB, Chapter 3.3).

### What a closure is

When the inner function is **returned** or otherwise outlives the outer call, Python keeps the free variables it needs alive, stored in small objects called **cells**. The inner function together with its captured cells is the closure.

```python
def make_adder(n):
    def add(x):
        return x + n      # n is a free variable
    return add
```

After `make_adder(5)` returns, its local variable `n` should have vanished. But the returned function `add` still needs it, so Python keeps it.

### Variables, not values

A closure captures the **variable**, not a snapshot of its value at that moment. If the variable changes later, the closure sees the new value. This is both what makes shared state possible and the cause of the late-binding trap.

### Function factories

A **function factory** is a function that returns a customized function. Each call to the factory creates a new closure with its own captured variables. The closures are independent of each other.

### State with nonlocal

An inner function can **read** a captured variable freely. To **rebind** it, you must declare it `nonlocal` (Chapter 3.3). That lets a closure keep private, changing state between calls.

### Late binding

Closures look up captured variables **when the inner function runs**, not when it is defined. Inside a loop, this means that functions created in different iterations may all see the loop variable's **final** value.

### Three ways to fix late binding

| Fix | Idea |
| --- | --- |
| Default argument | `lambda i=i: i` evaluates `i` when the function is **created** |
| Factory function | A helper that takes the value as a parameter, creating a new scope per call |
| `functools.partial` | Binds the argument up front (Chapter 3.7) |

## Basic syntax examples

### A first closure

```python
def make_adder(n):
    def add(x):
        return x + n
    return add


add5 = make_adder(5)
add10 = make_adder(10)
print(add5(1), add10(1), add5(100))
```

```text
6 11 105
```

`add5` and `add10` are two separate closures with different values for `n`. Neither is affected by the other.

### Looking inside

```python
def make_adder(n):
    def add(x):
        return x + n
    return add


add5 = make_adder(5)
print(add5.__code__.co_freevars)
print(add5.__closure__[0].cell_contents)
```

```text
('n',)
5
```

`co_freevars` lists the free variables, and `__closure__` holds the cells that store them. You will rarely need these, but they prove the captured value really lives inside the function object.

### A configurable function

```python
def make_formatter(currency, decimals=2):
    def format_amount(value):
        return f"{currency} {value:,.{decimals}f}"
    return format_amount


euros = make_formatter("EUR")
yen = make_formatter("JPY", decimals=0)
print(euros(1234.5))
print(yen(1234.5))
```

```text
EUR 1,234.50
JPY 1,234
```

Configure once, then use the result everywhere without repeating the settings.

### A counter with private state

```python
def make_counter(start=0, step=1):
    count = start

    def increment():
        nonlocal count
        count += step
        return count

    return increment


a = make_counter()
b = make_counter(start=100, step=10)
print(a(), a(), a())
print(b(), b())
```

```text
1 2 3
110 120
```

Each counter has its own hidden `count`. Nothing outside can read or change it except by calling the function.

### Forgetting nonlocal

```python
def make_counter():
    count = 0

    def increment():
        count += 1          # no nonlocal declaration
        return count

    return increment


make_counter()()
```

```text
UnboundLocalError: cannot access local variable 'count' where it is not associated with a value
```

This is the same trap as in Chapter 3.3. The assignment makes `count` local to `increment`, so it cannot read the enclosing value first.

### Mutable captured objects need no nonlocal

```python
def make_recorder():
    seen = []

    def record(item):
        seen.append(item)        # modifies the list, never rebinds the name
        return list(seen)        # return a copy, not the private list

    return record


record = make_recorder()
record("a")
print(record("b"))
```

```text
['a', 'b']
```

Because `seen` is only modified in place, no `nonlocal` is required.

### Several functions sharing one variable

```python
def make_account(balance):
    def deposit(amount):
        nonlocal balance
        balance += amount
        return balance

    def current():
        return balance

    return deposit, current


deposit, current = make_account(100)
deposit(50)
print(current())
```

```text
150
```

Both inner functions captured the **same** variable, so a change made by one is seen by the other.

### The late-binding pitfall

```python
functions = [lambda: i for i in range(3)]
print([f() for f in functions])
```

```text
[2, 2, 2]
```

All three lambdas share the single variable `i`. By the time they are called, the loop has finished and `i` is `2`.

### Fix 1: a default argument

```python
functions = [lambda i=i: i for i in range(3)]
print([f() for f in functions])
```

```text
[0, 1, 2]
```

A default value is evaluated when the lambda is **created**, so each one gets its own copy of the current `i`.

### Fix 2: a factory function

```python
def make_returner(value):
    return lambda: value


functions = [make_returner(i) for i in range(3)]
print([f() for f in functions])
```

```text
[0, 1, 2]
```

Each call to `make_returner` has its own `value`, so each lambda captures a separate variable.

### Fix 3: partial

```python
from functools import partial


def identity(value):
    return value


functions = [partial(identity, i) for i in range(3)]
print([f() for f in functions])
```

```text
[0, 1, 2]
```

## Deeper behavior

### Cells and the lifetime of captured variables

Python stores each captured variable in a **cell** object that is shared between the outer and inner functions. A cell lives as long as any function refers to it. That is how `n` survives after `make_adder` returns, and also why a closure that captures a large object keeps that object in memory for as long as the closure exists.

### Each call to the outer function creates new cells

```python
def make_counter():
    count = 0

    def increment():
        nonlocal count
        count += 1
        return count

    return increment


first = make_counter()
second = make_counter()
first()
first()
print(first(), second())
```

```text
3 1
```

The two counters have different cells, so they never interfere.

### Closure versus class

Both can hold state and behavior together. The choice is about size and clarity:

| Use a closure when | Use a class when |
| --- | --- |
| There is one operation and a little state | There are several operations on the same state |
| The state is private and should stay that way | Callers need to read or change the state |
| You are building a decorator or callback | You need inheritance, a readable `repr`, or many attributes |
| The result is naturally "a function" | The result is naturally "a thing" with a name and several methods |

A closure with three `nonlocal` variables and five inner functions is a class in disguise. Switch to a class (Chapter 6.1) when it grows.

### Late binding is not a bug in Python

Looking variables up at call time is what allows closures to see updates, such as the shared `balance` above. The loop case is surprising only because the loop variable is rebound on every iteration, and all the closures share that single variable. Whenever you create functions in a loop, ask: "should each function remember its own value?" If so, use one of the fixes.

### Closures and `lambda` in callbacks

A very common real-world form of the bug:

```python
buttons = {}
for label in ["save", "load", "quit"]:
    buttons[label] = lambda: f"{label} clicked"       # every lambda sees the last label
```

All three would say `quit clicked`. Writing `lambda label=label: ...` or a factory function fixes it.

### Thread safety

`count += 1` inside a closure is not atomic: it reads, adds, and writes. Two threads calling the same counter can lose updates (Chapter 10.2). Use a lock, or a thread-safe structure, when a closure's state is shared between threads.

### Debugging closures

A traceback shows the inner function's name and where it was **defined**, which may be a different file from where the problem happened. Give the inner function a clear name (not a bare lambda) so you can recognize it, and set `__name__` or use `functools.wraps` when wrapping other functions (Chapter 3.6).

### Closures are the basis of decorators

A decorator is a function that takes a function and returns a new one that remembers the original. That is exactly a closure. Chapter 3.6 builds on everything in this chapter.

### Not every nested function is a closure

An inner function that uses none of the outer function's variables is just a nested function. It becomes a closure only if it **uses** at least one free variable.

## Industry-level snippet

A **token bucket** is a classic algorithm for rate limiting: a bucket holds up to `capacity` tokens, refills at a steady rate, and each request spends tokens. Because its state is a few private numbers and its only operation is "try to take tokens", a closure expresses it neatly. This module builds a token-bucket factory, plus a formatter factory, and takes the clock as a parameter so that tests and demos can control time.

```python
"""Closure-based factories: a token-bucket rate limiter and a money formatter."""

import logging
import time
from collections.abc import Callable

logger = logging.getLogger(__name__)


def make_token_bucket(
    capacity: float,
    refill_per_second: float,
    clock: Callable[[], float] = time.monotonic,
) -> Callable[[float], bool]:
    """Create a rate limiter and return its try_acquire function.

    The bucket starts full. try_acquire(cost) removes cost tokens and returns
    True if enough tokens are available, otherwise it changes nothing and
    returns False.

    Raises:
        ValueError: If capacity or refill_per_second is not positive.
    """
    if capacity <= 0:
        raise ValueError("capacity must be greater than zero")
    if refill_per_second <= 0:
        raise ValueError("refill_per_second must be greater than zero")

    tokens = capacity
    last_refill = clock()

    def try_acquire(cost: float = 1.0) -> bool:
        nonlocal tokens, last_refill
        now = clock()
        elapsed = now - last_refill
        tokens = min(capacity, tokens + elapsed * refill_per_second)
        last_refill = now
        if cost > tokens:
            logger.debug(
                "Rejected request costing %.1f (%.2f tokens left)", cost, tokens
            )
            return False
        tokens -= cost
        return True

    return try_acquire


def make_money_formatter(currency: str, decimals: int = 2) -> Callable[[float], str]:
    """Return a function that formats amounts in a fixed currency and precision."""
    if len(currency) != 3 or not currency.isalpha():
        raise ValueError(f"currency must be a 3-letter code, got {currency!r}")
    if decimals < 0:
        raise ValueError("decimals must not be negative")
    code = currency.upper()

    def format_amount(value: float) -> str:
        return f"{code} {value:,.{decimals}f}"

    return format_amount
```

Try it:

```python
now = [0.0]
try_acquire = make_token_bucket(capacity=3, refill_per_second=1, clock=lambda: now[0])

print([try_acquire() for _ in range(4)])
now[0] = 2.0
print([try_acquire() for _ in range(3)])

per_user = {name: make_token_bucket(2, 0.5, clock=lambda: now[0]) for name in ["ada", "grace"]}
print(per_user["ada"](), per_user["ada"](), per_user["ada"](), per_user["grace"]())

format_eur = make_money_formatter("eur")
print(format_eur(1234.5))
```

```text
[True, True, True, False]
[True, True, False]
True True False True
EUR 1,234.50
```

Why it is written this way:

- `tokens` and `last_refill` are **private**: nothing outside `make_token_bucket` can read or corrupt them. The only way to interact is through the returned function.
- `nonlocal` declares exactly the two variables the inner function rebinds. `capacity` and `refill_per_second` are only read, so they need no declaration.
- Every call to `make_token_bucket` creates a fresh set of variables, so each user gets an independent limiter, as the `per_user` dictionary shows.
- The clock is a parameter, the same technique used for the sliding-window limiter in Chapter 2.6, so behavior over time can be tested instantly.
- The lambda `lambda: now[0]` inside the comprehension reads `now` at call time on purpose: it is **meant** to see the later assignments. This is late binding being useful.
- If the limiter needed more operations, such as `reset()`, `remaining()`, and a readable name, that would be the moment to turn it into a class.

## Common mistakes

1. **Creating functions in a loop that all share the loop variable** (late binding). Use a default argument, a factory, or `partial`.
2. **Forgetting `nonlocal`** when rebinding a captured variable, which gives `UnboundLocalError`.
3. **Using `global` instead of `nonlocal`** for an enclosing function's variable.
4. **Expecting a closure to capture a snapshot** of a value, not the variable itself.
5. **Returning a captured mutable object directly,** letting callers modify the private state. Return a copy.
6. **Building a closure with many variables and many inner functions** when a class would be clearer.
7. **Using lambdas for closures that need good error messages and documentation.** Use a named inner function with a docstring.
8. **Forgetting that a closure keeps its captured objects alive,** holding on to large data longer than expected.
9. **Sharing a closure's mutable state between threads** without a lock.
10. **Calling the factory when you meant to pass it,** or the reverse: `make_counter` is the factory and `make_counter()` is the counter function.
11. **Confusing a closure with a global variable.** A closure's variables are private to the function that created them.

## Practice exercises

1. Write `make_power(exponent)` that returns a function raising its argument to that power. Create `square` and `cube` from it and test them.
2. Write `make_counter(start, step)` and make three independent counters. Show that calling one does not change the others.
3. Write `make_accumulator()` that returns a function which adds its argument to a running total and returns the total. Show it keeps state between calls.
4. Reproduce the late-binding bug with `[lambda: i for i in range(3)]`, and then fix it in all three ways. Explain in a sentence why each fix works.
5. Write `make_logger(prefix)` that returns a function printing messages with the prefix, and use it to create loggers named `"auth"` and `"db"`.
6. Show that two functions returned from the same call of an outer function can share a variable, using a `make_account` with `deposit` and `balance` functions.
7. Write a function that returns a closure containing a private list, and make sure callers cannot modify the list directly. Demonstrate what happens if you return the list itself instead of a copy.
8. Use `__code__.co_freevars` and `__closure__` to inspect a closure, and then check which variables of a nested function are free and which are local.
9. Write `make_memoizer(func)` that returns a function that remembers results in a private dictionary, and use it on a slow function. (Chapter 3.7 introduces `functools.cache`, which does this for you.)
10. Write `make_rate_limiter(max_calls)` that returns a function allowing only `max_calls` calls and then returning `False`. Rewrite it as a class and compare which is clearer.
11. Make a closure that captures a very large list, delete the outer reference to the list, and explain why the memory is not freed until the closure is deleted too.
12. Extend `make_token_bucket` to return **two** functions, `try_acquire` and `tokens_remaining`, that share the same state. Write tests with a fake clock covering refill, capacity limits, and rejection.

## Summary

- A function defined inside another function can use the outer function's variables. These are **free variables**.
- A **closure** is an inner function bundled with the free variables it uses. Python keeps those variables alive in **cells** after the outer function returns.
- Closures capture **variables**, not values, so they see later changes. Each call to the outer function creates a new, independent set.
- A **function factory** returns functions configured by its arguments.
- To rebind a captured variable, declare it `nonlocal`. Modifying a captured mutable object in place needs no declaration.
- **Late binding**: closures look up variables when called. Functions created in a loop all see the loop variable's final value. Fix it with a default argument, a factory function, or `functools.partial`.
- Return copies of private mutable state, not the state itself.
- Use a closure for one operation with a little private state, and a class when the state and behavior grow.
- Closures are the foundation of decorators, which come next.
