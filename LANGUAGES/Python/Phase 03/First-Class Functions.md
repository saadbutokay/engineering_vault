## Overview

In Python, a function is not a special kind of thing that lives apart from your data. A function is an **object**, just like a number, a string, or a list. You can store it in a variable, put it in a list or dictionary, pass it as an argument, and return it from another function. Languages that allow this are said to have **first-class functions**.

This idea unlocks a powerful style of programming. Sorting by a custom rule, running a callback when an event happens, choosing behavior from a table, building pipelines of transformations: all of these work by passing functions around. You have already used it: `sorted(words, key=len)` passes the function `len` as an argument.

By the end of this note you will be able to:

- Treat functions as values: assign them, store them, pass them, and return them.
- Write `lambda` expressions and know their limits.
- Use `map`, `filter`, and `sorted(key=)` with functions, and know when comprehensions are clearer.
- Replace long `if/elif` chains with dispatch tables.
- Use the `operator` module for common small functions.

> [!note] A function name is just a name
> Writing `greet` refers to the function object. Writing `greet("Ada")` **calls** it. The parentheses are what run the code.

## Core concepts

### Functions are objects

Defining a function creates an object and binds it to a name (Chapter 3.1). That object has attributes such as `__name__` and `__doc__`, can be compared by identity, and can be bound to other names.

### Higher-order functions

A **higher-order function** is a function that takes another function as an argument, returns a function, or both. `sorted`, `map`, `filter`, `max`, and `min` (with a `key`) are higher-order. So is any function you write that accepts a callback.

### Passing a function as an argument

The caller decides **what** to do, and the receiving function decides **when and how often**:

```python
def apply_twice(func, value):
    return func(func(value))
```

### Returning a function

A function can build and return another function, tailored by its arguments. This is the basis of closures (Chapter 3.5) and decorators (Chapter 3.6).

### Lambda expressions

A **lambda** is a small anonymous function written as a single expression:

```python
lambda parameters: expression
```

It returns the value of the expression automatically. Lambdas are limited: the body must be **one expression**, with no statements, no assignments (apart from the walrus operator), and no `return` keyword. They are best used for short throwaway functions, typically as a `key` or a callback.

| `def` | `lambda` |
| --- | --- |
| Has a name | Anonymous |
| Any number of statements | One expression |
| Can have a docstring | Cannot |
| Better for anything reused or non-trivial | Better for a tiny inline function |

PEP 8 advises against assigning a lambda to a name (`add = lambda a, b: a + b`). If it needs a name, use `def`.

### map and filter

| Function | What it does | Returns |
| --- | --- | --- |
| `map(func, iterable)` | Applies `func` to every item | An iterator of the results |
| `filter(func, iterable)` | Keeps the items for which `func(item)` is truthy | An iterator of those items |
| `filter(None, iterable)` | Keeps the truthy items | An iterator |

Both return **iterators**, which produce their values lazily and only once. Wrap them in `list(...)` to see all the results.

### Comprehensions or map and filter?

Most Python programmers prefer comprehensions (Chapter 2.1) for readability:

```python
[n * n for n in numbers if n % 2 == 0]
```

instead of `list(map(lambda n: n * n, filter(lambda n: n % 2 == 0, numbers)))`. `map` and `filter` still shine when you already have a **named function** to apply, as in `map(int, strings)`.

### The key function

Many tools accept a `key=` argument: a function that is called once per item and returns the value to compare. This applies to `sorted`, `list.sort`, `min`, `max`, `heapq.nlargest` (Chapter 2.7), and `itertools.groupby` (Chapter 7.3).

### The operator module

The `operator` module provides function versions of operators and common extractors, so you rarely need a lambda for simple jobs:

| Function | Equivalent lambda |
| --- | --- |
| `operator.add` | `lambda a, b: a + b` |
| `operator.mul` | `lambda a, b: a * b` |
| `operator.itemgetter(1)` | `lambda x: x[1]` |
| `operator.itemgetter("age")` | `lambda x: x["age"]` |
| `operator.attrgetter("name")` | `lambda x: x.name` |

### Dispatch tables

A **dispatch table** is a dictionary whose values are functions. It replaces a chain of `if/elif` comparisons with a lookup:

```python
OPERATIONS = {"add": add, "sub": sub}
result = OPERATIONS[name](a, b)
```

### Callables

Anything that can be called with parentheses is a **callable**: functions, lambdas, built-in functions like `len`, methods, classes (calling a class creates an instance), and objects that define a `__call__` method (Chapter 6.3). The built-in `callable(x)` tells you whether `x` can be called.

## Basic syntax examples

### Functions as values

```text
>>> def square(n):
...     return n * n
...
>>> f = square
>>> f(5)
25
>>> f.__name__
'square'
>>> callable(f), callable(5)
(True, False)
>>> operations = [square, abs, str]
>>> [op(-3) for op in operations]
[9, 3, '-3']
>>> {"double": lambda n: n * 2}["double"](21)
42
```

`f` and `square` refer to the same object. A list can hold functions just as easily as numbers, and each can be called in turn.

### Passing a function as an argument

```python
def apply_twice(func, value):
    return func(func(value))


def add_exclamation(text):
    return text + "!"


print(apply_twice(add_exclamation, "Hello"))
print(apply_twice(lambda n: n * 3, 2))
print(apply_twice(str.upper, "ada"))
```

```text
Hello!!
18
ADA
```

`str.upper` is itself a function: the method looked up on the class, ready to be applied to a string.

### Returning a function

```python
def make_multiplier(factor):
    def multiply(n):
        return n * factor
    return multiply


double = make_multiplier(2)
triple = make_multiplier(3)
print(double(10), triple(10))
```

```text
20 30
```

Each call to `make_multiplier` produces a different function that remembers its own `factor`. Chapter 3.5 explains how.

### Lambda

```text
>>> square = lambda n: n * n
>>> square(4)
16
>>> (lambda a, b: a + b)(2, 3)
5
>>> sorted(["bb", "a", "ccc"], key=lambda word: -len(word))
['ccc', 'bb', 'a']
>>> max(["bb", "a", "ccc"], key=len)
'ccc'
```

The second line shows that a lambda is an expression: you can define and call it in one go, although there is rarely a reason to.

### map and filter

```text
>>> def square(n):
...     return n * n
...
>>> list(map(square, [1, 2, 3]))
[1, 4, 9]
>>> list(map(int, ["1", "2", "3"]))
[1, 2, 3]
>>> list(map(lambda a, b: a + b, [1, 2], [10, 20]))
[11, 22]
>>> list(filter(lambda n: n % 2 == 0, range(10)))
[0, 2, 4, 6, 8]
>>> list(filter(None, [0, 1, "", "a", [], [0]]))
[1, 'a', [0]]
>>> [n for n in range(10) if n % 2 == 0]
[0, 2, 4, 6, 8]
```

`map` with two iterables passes one item from each to the function. `filter(None, ...)` discards everything falsy. The last line is the comprehension equivalent of the `filter` before it.

### Iterators are used up

```text
>>> numbers = map(abs, [-1, -2, -3])
>>> next(numbers)
1
>>> list(numbers)
[2, 3]
>>> list(numbers)
[]
```

A `map` or `filter` object can be consumed only once. If you need the results again, store them in a list.

### Sorting with key

```text
>>> people = [{"name": "Ada", "age": 36}, {"name": "Grace", "age": 45}, {"name": "Linus", "age": 28}]
>>> [p["name"] for p in sorted(people, key=lambda p: p["age"])]
['Linus', 'Ada', 'Grace']
>>> from operator import itemgetter
>>> [p["name"] for p in sorted(people, key=itemgetter("age"), reverse=True)]
['Grace', 'Ada', 'Linus']
>>> words = ["banana", "Apple", "cherry"]
>>> sorted(words, key=str.lower)
['Apple', 'banana', 'cherry']
```

### The operator module

```text
>>> import operator
>>> operator.add(2, 3), operator.mul(4, 5)
(5, 20)
>>> from functools import reduce
>>> reduce(operator.mul, [1, 2, 3, 4])
24
>>> from operator import attrgetter
>>> from collections import namedtuple
>>> Person = namedtuple("Person", "name age")
>>> people = [Person("Ada", 36), Person("Linus", 28)]
>>> [p.name for p in sorted(people, key=attrgetter("age"))]
['Linus', 'Ada']
```

### A dispatch table

```python
def add(a, b):
    return a + b


def subtract(a, b):
    return a - b


def multiply(a, b):
    return a * b


OPERATIONS = {"+": add, "-": subtract, "*": multiply}


def calculate(a, operator_symbol, b):
    try:
        operation = OPERATIONS[operator_symbol]
    except KeyError:
        raise ValueError(f"unsupported operator: {operator_symbol!r}") from None
    return operation(a, b)


print(calculate(6, "*", 7))
print(calculate(6, "-", 7))
```

```text
42
-1
```

Adding a new operation means adding one line to the table. No other code changes.

### A callback

```python
def process_all(items, on_done):
    results = [item * 2 for item in items]
    on_done(results)


process_all([1, 2, 3], on_done=print)
process_all([4, 5], on_done=lambda r: print("total:", sum(r)))
```

```text
[2, 4, 6]
total: 18
```

The caller hands over **what to do when finished**. This is how event handlers, framework hooks, and asynchronous callbacks work.

## Deeper behavior

### Functions have attributes

```python
def greet(name: str) -> str:
    """Return a greeting."""
    return f"Hello, {name}!"


print(greet.__name__, "|", greet.__doc__)
print(greet.__annotations__)
greet.version = 2          # you can even attach your own data
print(greet.version)
```

```text
greet | Return a greeting.
{'name': <class 'str'>, 'return': <class 'str'>}
2
```

Decorators use these attributes, and tools such as documentation generators and web frameworks read them to learn what a function expects.

### Methods are functions too

`"text".upper` is a **bound method**: a function already tied to its object. `str.upper` is the plain function, waiting to be told which string to work on. Both work as arguments to `map` or `key=`:

```text
>>> list(map(str.upper, ["a", "b"]))
['A', 'B']
>>> upper = "hello".upper
>>> upper()
'HELLO'
```

### Lambdas capture variables, not values

A lambda that mentions a variable from outside looks it up **when it is called**, not when it is created. In a loop this causes a surprising result that Chapter 3.5 explains. For now, remember that a lambda's body is evaluated late.

### When a lambda is the wrong tool

- **Anything with a name.** If you reuse it or it deserves a docstring, write a `def`.
- **Anything with more than one simple operation.** A lambda with a conditional expression and several calls is hard to read.
- **Anything with a statement.** You cannot write `try`, `if` statements, or assignments in a lambda.
- **Anything that needs a clear traceback.** Errors in lambdas show as `<lambda>`, which tells you little.

### `map` and `filter` versus comprehensions

| Situation | Prefer |
| --- | --- |
| Applying an existing named function to each item | `map(func, items)` is fine |
| Any condition or expression on the item | A comprehension |
| You need the result more than once | `list(...)` or a comprehension |
| Lazy processing of a huge stream | `map`, `filter`, or a generator expression |
| Combining transformation and filtering | A comprehension |

Neither is meaningfully faster in everyday code, so choose the more readable one.

### Dispatch tables versus `match`

A dispatch table is data: you can build it at run time, extend it from other modules, and test each function separately. A `match` statement (Chapter 1.6) is better when the decision depends on the **shape** of the data rather than a simple key. Use a table for "name to action", and `match` for "take this structure apart".

### Callable objects

Any object whose class defines `__call__` behaves like a function but can also hold state:

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, n):
        return n * self.factor


double = Multiplier(2)
print(double(21), callable(double))
```

```text
42 True
```

Chapter 6.3 covers this. It is one way to give a "function" its own memory, alongside closures.

### Functions in data structures make code extensible

A list of functions is a pipeline. A dictionary of functions is a plug-in registry. Both let you add behavior by adding data, without editing the code that uses them. This is the **strategy pattern** in its simplest form (Chapter 13.4).

### Order of evaluation in a pipeline

When you chain `map` and `filter`, nothing runs until you ask for results, and then each item flows through the whole chain before the next item starts. This laziness allows processing data larger than memory.

## Industry-level snippet

Backend services react to many kinds of events, and text often has to pass through several cleaning steps. This module shows both patterns built from first-class functions: an **event router** that looks handlers up in a dispatch table and isolates their failures, and a `compose` helper that builds a text-cleaning pipeline out of small functions. The router receives its table as a parameter, so there is no global state to manage.

```python
"""Event routing and text pipelines built from first-class functions."""

import logging
from collections.abc import Callable, Mapping
from typing import Any

logger = logging.getLogger(__name__)

Event = Mapping[str, Any]
Handler = Callable[[Event], None]


class UnknownEventError(Exception):
    """Raised when an event has no registered handler."""


def route_event(event: Event, handlers: Mapping[str, Handler]) -> None:
    """Send an event to the handler registered for its type.

    A failure inside a handler is logged and re-raised, so the caller decides
    what to do, and the log always says which event failed.

    Raises:
        UnknownEventError: If the event has no type or no handler for it.
    """
    event_type = event.get("type")
    handler = handlers.get(event_type) if isinstance(event_type, str) else None
    if handler is None:
        raise UnknownEventError(f"no handler for event type {event_type!r}")
    try:
        handler(event)
    except Exception:
        logger.exception("Handler for %r failed", event_type)
        raise


def compose(*steps: Callable[[str], str]) -> Callable[[str], str]:
    """Combine text-processing functions into one that applies them in order."""

    def pipeline(text: str) -> str:
        for step in steps:
            text = step(text)
        return text

    return pipeline


def collapse_spaces(text: str) -> str:
    """Replace every run of whitespace with a single space."""
    return " ".join(text.split())


def strip_quotes(text: str) -> str:
    """Remove straight and curly quotation marks."""
    return text.translate({ord(char): None for char in "\"'\u2018\u2019\u201c\u201d"})


normalize_name = compose(str.strip, collapse_spaces, strip_quotes, str.casefold)
```

Try it:

```python
received: list[str] = []

handlers: dict[str, Handler] = {
    "user.created": lambda event: received.append(f"welcome {event['name']}"),
    "order.paid": lambda event: received.append(f"ship order {event['id']}"),
}

route_event({"type": "user.created", "name": "ada"}, handlers)
route_event({"type": "order.paid", "id": 17}, handlers)
print(received)

try:
    route_event({"type": "user.deleted"}, handlers)
except UnknownEventError as exc:
    print("UnknownEventError:", exc)

print(repr(normalize_name('   \u201cAda\u201d   LOVELACE  ')))
```

```text
['welcome ada', 'ship order 17']
UnknownEventError: no handler for event type 'user.deleted'
'ada lovelace'
```

Why it is written this way:

- Handlers are plain functions stored as **dictionary values**. Supporting a new event type means adding an entry, not editing `route_event`.
- The table is a **parameter**, so tests can pass a table containing fakes, and different parts of an application can have different tables.
- `compose` returns a new function. The pipeline `normalize_name` is built from existing functions, including built-in methods like `str.strip` and `str.casefold`, with no lambdas needed.
- `route_event` logs the full traceback with `logger.exception` and re-raises, so errors are neither hidden nor lost.
- Type aliases `Event` and `Handler` make the signatures readable and say exactly what a handler must look like.

## Common mistakes

1. **Calling the function when you meant to pass it.** `sorted(words, key=len())` is an error, `key=len` is correct. Likewise `map(square(), items)` is wrong.
2. **Forgetting the parentheses when you meant to call it.** `result = compute` stores the function, not its result.
3. **Expecting `map` or `filter` to return a list.** They return one-shot iterators.
4. **Consuming an iterator twice** and finding it empty the second time.
5. **Writing complicated lambdas.** If it needs a second line or a name, write a `def`.
6. **Assigning a lambda to a variable** to give it a name. Use `def`.
7. **Using `lambda` where `operator.itemgetter` or a method reference such as `str.lower` would be clearer.**
8. **Building lambdas in a loop** and being surprised when they all use the last value (Chapter 3.5).
9. **Using a long `if/elif` chain** to select behavior that a dispatch dictionary would express more cleanly.
10. **Forgetting that dictionary dispatch needs a plan for unknown keys.** Decide whether to raise a clear error or fall back to a default.
11. **Mixing up `str.upper` (the function) and `"x".upper` (a bound method).**
12. **Hiding errors in callbacks.** A callback that swallows exceptions makes problems invisible. Log them and decide on a policy.

## Practice exercises

1. Store three functions in a list and call each of them on the same argument in a loop. Print the results with the function names.
2. Write `apply_n_times(func, value, n)` that applies a function repeatedly, and use it with a doubling lambda and with `str.strip`.
3. Use `map` to convert a list of strings like `["3", "1", "2"]` to integers and sort them. Then write the same thing as a comprehension.
4. Use `filter` to keep only the strings longer than three characters from a list, and then do it again with a comprehension. Which do you find clearer?
5. Sort a list of dictionaries with keys `"name"` and `"age"` in three ways: with a lambda, with `itemgetter`, and by two keys at once (age descending, then name).
6. Write a calculator that uses a dispatch table for the operations `+`, `-`, `*`, `/`, `**`, and handles division by zero and unknown operators with clear errors.
7. Write `make_greeter(greeting)` that returns a function. Create two greeters with different greetings and show that they work independently.
8. Write a function `retry_call(func, attempts)` that calls `func()` until it succeeds or the attempts run out, and returns the result. Test it with a function that fails twice and then succeeds.
9. Build a text-cleaning pipeline with `compose` that lowercases, removes punctuation, and collapses spaces. Use it on three different strings.
10. Explain in your own words why `sorted(items, key=len())` fails and `sorted(items, key=len)` works.
11. Write a class with a `__call__` method that counts how many times it has been called, and compare it with a function that does the same using a closure from Chapter 3.5.
12. Extend `route_event` so that a handler table may also contain a `"*"` handler used for any event type without its own handler, and so that an optional list of "middleware" functions runs on every event first. Write tests for both features.

## Summary

- Functions are **first-class objects**: you can bind them to names, store them in data structures, pass them to functions, and return them from functions.
- A **higher-order function** takes or returns functions. `sorted`, `map`, `filter`, `min`, and `max` are all examples.
- Pass a function with its name and no parentheses. Parentheses call it.
- A `lambda` is a one-expression anonymous function, good for short keys and callbacks, and wrong for anything complex or reused. Use `def` for those.
- `map` and `filter` return lazy, one-shot iterators. A comprehension is usually clearer when you need a condition or expression.
- The `key=` function is called once per item and decides the sort or comparison.
- The `operator` module (`itemgetter`, `attrgetter`, `add`, `mul`) and method references such as `str.lower` often replace lambdas.
- A dictionary of functions, a **dispatch table**, replaces `if/elif` chains and makes behavior extensible.
- Callbacks and pipelines make code flexible: the caller supplies **what to do**, and the function controls **when**.
