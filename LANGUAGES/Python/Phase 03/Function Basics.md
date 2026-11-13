## Overview

A **function** is a named, reusable block of code. You have been calling functions since your first `print("hello")`. Now you will write your own. Functions are the main tool for organizing programs: they let you give a name to a piece of logic, use it in many places, test it on its own, and change it in one place instead of twenty.

Every professional codebase is mostly functions. A web request handler is a function. So is a validator, a price calculator, and a database query helper. Learning to design small, clear functions is one of the highest-value skills in programming.

By the end of this note you will be able to:

- Define and call functions with parameters and return values.
- Explain the difference between a parameter and an argument, and between `print` and `return`.
- Return several values and unpack them.
- Understand what happens when a function has no `return` statement.
- Write docstrings that explain what a function does.
- Design functions that do one job and are easy to test.

> [!note] The idea of abstraction
> A function lets the caller ignore **how** something is done and think only about **what** it does. `sorted(items)` is useful because you do not need to know which sorting algorithm it uses.

## Core concepts

### Defining a function

```python
def greet(name):
    print(f"Hello, {name}!")
```

| Part | Meaning |
| --- | --- |
| `def` | The keyword that starts a function definition |
| `greet` | The function's name, following the naming rules of Chapter 1.2 |
| `(name)` | The **parameter** list, the names the function uses for incoming values |
| `:` | Starts the body, which must be indented |
| The indented block | The **body**, the code that runs when the function is called |

### Calling a function

You run the body by writing the function's name followed by parentheses containing the **arguments**:

```python
greet("Ada")
```

The parentheses are required. Writing just `greet` refers to the function object without running it.

### Parameters and arguments

These two words are often confused, and the distinction is useful:

- A **parameter** is a name in the function definition: `name` in `def greet(name)`.
- An **argument** is a value passed in a call: `"Ada"` in `greet("Ada")`.

When the call runs, each parameter is bound to the corresponding argument, exactly like an assignment (Chapter 1.2).

### Return values

The `return` statement sends a value back to the caller **and immediately ends the function**. The call expression then has that value:

```python
def add(a, b):
    return a + b


total = add(2, 3)
```

A function may contain several `return` statements, for example in different branches of an `if`. Only the first one reached runs.

### No return means None

If a function ends without executing a `return` statement, or uses `return` with no value, it returns `None`. Functions that exist for their effects, such as printing or saving, usually return nothing. A function that exists to produce a value should always `return` it.

### print versus return

This distinction trips up almost every beginner.

| | `print(x)` | `return x` |
| --- | --- | --- |
| Shows the value on screen | Yes | No |
| Gives the value back to the caller | No (the call evaluates to `None`) | Yes |
| Ends the function | No | Yes |
| Can the caller use the value? | No | Yes |

If you want to calculate something and use it later, **return** it. Printing is for showing information to a person.

### Returning several values

A function returns one object, but that object can be a tuple, so in practice you return several values by separating them with commas. The caller unpacks them (Chapter 2.2):

```python
def min_max(numbers):
    return min(numbers), max(numbers)


low, high = min_max([3, 9, 1])
```

### Docstrings

A **docstring** is a string literal placed as the very first statement in the body. It documents the function. Tools, editors, and the built-in `help()` function show it.

```python
def area(width, height):
    """Return the area of a rectangle."""
    return width * height
```

For anything beyond a one-liner, use the Google style from Chapter 0.7, with `Args`, `Returns`, and `Raises` sections.

### Type hints

You can annotate parameters and the return value. Python does not enforce them, but editors and `mypy` use them to catch mistakes (Chapter 0.9):

```python
def area(width: float, height: float) -> float:
    return width * height
```

A function that returns nothing is annotated `-> None`.

### What makes a good function

| Principle | Meaning |
| --- | --- |
| One job | A function should do one thing, and its name should say what |
| A verb for a name | `calculate_total`, `send_email`, `is_valid`. Booleans read as questions |
| Few parameters | More than three or four often signals a missing concept |
| Small | If it does not fit on a screen, consider splitting it |
| Predictable | Same inputs give same outputs, with no surprises |
| Documented | A docstring says what, the code shows how |

## Basic syntax examples

### A function that prints

```python
def greet(name):
    print(f"Hello, {name}!")


greet("Ada")
greet("Grace")
result = greet("Linus")
print(result)
```

```text
Hello, Ada!
Hello, Grace!
Hello, Linus!
None
```

The function printed three greetings and returned nothing, so `result` is `None`.

### A function that returns

```python
def add(a, b):
    return a + b


total = add(2, 3)
print(total * 10)
print(add(add(1, 2), 4))
```

```text
50
7
```

Because `add` returns its result, you can store it, do arithmetic with it, or pass it to another call.

### The print-versus-return mistake

```python
def add_and_print(a, b):
    print(a + b)


answer = add_and_print(2, 3)
print(answer)
print(answer + 1)
```

```text
5
None
```

The final line raises `TypeError: unsupported operand type(s) for +: 'NoneType' and 'int'`, because `answer` is `None`. The fix is to `return a + b` instead of printing it.

### Return ends the function

```python
def first_negative(numbers):
    for n in numbers:
        if n < 0:
            return n          # leaves the function right here
    return None               # reached only if the loop found nothing


print(first_negative([4, 8, -2, -7]))
print(first_negative([1, 2, 3]))
```

```text
-2
None
```

### Early returns for clarity

```python
def classify_age(age):
    if age < 0:
        return "invalid"
    if age < 18:
        return "minor"
    return "adult"


print(classify_age(-1), classify_age(12), classify_age(40))
```

```text
invalid minor adult
```

Handling the unusual cases first and returning immediately avoids deep nesting and leaves the main logic at the end, unindented.

### Returning several values

```python
def stats(numbers):
    return min(numbers), max(numbers), sum(numbers) / len(numbers)


low, high, mean = stats([3, 9, 6])
print(low, high, mean)
print(stats([3, 9, 6]))
```

```text
3 9 6.0
(3, 9, 6.0)
```

### A function with no parameters

```python
def show_banner():
    print("=" * 20)
    print("Daily Report")
    print("=" * 20)


show_banner()
```

```text
====================
Daily Report
====================
```

### Docstrings and help

```python
def area(width: float, height: float) -> float:
    """Return the area of a rectangle."""
    return width * height


print(area.__doc__)
help(area)
```

```text
Return the area of a rectangle.
Help on function area in module __main__:

area(width: float, height: float) -> float
    Return the area of a rectangle.

```

### Functions are values too

```text
>>> def square(n):
...     return n * n
...
>>> square(4)
16
>>> type(square)
<class 'function'>
>>> square.__name__
'square'
>>> callable(square)
True
>>> f = square
>>> f(5)
25
```

A function is an object. You can store it under another name, put it in a list, or pass it to other functions (Chapter 3.4).

### Calling before defining

```python
print(square(3))


def square(n):
    return n * n
```

```text
NameError: name 'square' is not defined
```

The `def` statement must run before the call does (Chapter 1.1). The usual layout is to put all the definitions first and the code that starts the program at the bottom.

### Placeholders

```python
def not_written_yet():
    pass                     # or write ... (an ellipsis)
```

An empty function body is a syntax error, so use `pass` or `...` while you are sketching.

## Deeper behavior

### Every call gets its own workspace

Each time a function is called, Python creates a fresh set of **local variables** for that call, which disappear when it returns. Two calls never share them. The calls are stacked: if `main` calls `total_cost`, which calls `float`, the innermost call finishes first. This **call stack** is the list of frames you read in a traceback (Chapter 1.7).

```python
def counter_demo(start):
    count = start
    count += 1
    return count


print(counter_demo(10), counter_demo(10))
```

```text
11 11
```

### The `def` statement runs at run time

`def` is an executable statement. It creates a function object and binds it to a name. If you define two functions with the same name in one file, the second silently replaces the first. You can even define a function inside an `if`, or inside another function (Chapter 3.5).

### Implicit `None` and type checkers

A function that returns a value on some paths but falls off the end on others quietly returns `None` on those paths. This is a frequent source of `AttributeError: 'NoneType' object has no attribute ...` far from the real mistake. `mypy` reports it as "Missing return statement" (Chapter 0.9). Make every path return something deliberate.

### Pure functions and side effects

A **pure function** depends only on its arguments and produces only its return value. It has no **side effects**: it does not print, write files, change global variables, or modify its arguments.

| Pure | Has side effects |
| --- | --- |
| `def add(a, b): return a + b` | `def log(msg): print(msg)` |
| `def total(prices): return sum(prices)` | `def clear(items): items.clear()` |
| Easy to test: call and compare | Needs setup and checking of the effect |
| Safe to call anywhere, any number of times | Calling it again repeats the effect |

You cannot avoid side effects in a real program, but you can **separate** them: write pure functions that calculate, and thin functions at the edges that print, read files, and talk to the network. That separation is the single best habit for testable code.

### Consistent return types

A function should return the same kind of thing on every path. Returning a list in one branch and `False` in another forces every caller to check. Prefer returning an empty list, or raising an exception (Chapter 5.1) for genuine errors, over returning special marker values.

### Functions should not surprise

Be suspicious of a function whose name promises one thing and does another. `get_user()` that also creates a user, or `is_valid()` that also modifies data, causes bugs. A name should describe everything the function does.

### Arguments are evaluated first

Python evaluates the arguments of a call, from left to right, before running the function. In `add(add(1, 2), 4)`, the inner call runs first.

### Function introspection

Every function carries information about itself:

```python
def area(width: float, height: float) -> float:
    """Return the area of a rectangle."""
    return width * height


print(area.__name__)
print(area.__annotations__)
```

```text
area
{'width': <class 'float'>, 'height': <class 'float'>, 'return': <class 'float'>}
```

Frameworks use this information to generate documentation and validate inputs.

### Keep functions short, not artificially tiny

A long function that does several jobs should be split, but splitting into dozens of one-line functions can make code harder to follow. A good guideline: if you can describe the function in one sentence without the word "and", it is probably the right size.

### Recursion is a function calling itself

Nothing prevents a function from calling itself. Chapter 3.8 covers when and how to do that safely.

## Industry-level snippet

A sign-up endpoint must validate a username, an email address, and a password, and report **every** problem at once instead of one at a time. This module shows good function design: each check is a small pure function that returns a list of problems, one orchestrating function combines them, and the only side effect (logging) is kept out of the checks and carefully avoids logging the password.

```python
"""Small, single-purpose functions for validating a sign-up request."""

import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

USERNAME_MIN_LENGTH = 3
USERNAME_MAX_LENGTH = 20
PASSWORD_MIN_LENGTH = 12


class ValidationError(ValueError):
    """Raised when a sign-up request has one or more problems."""


@dataclass(frozen=True)
class SignUp:
    """A validated sign-up request."""

    username: str
    email: str


def check_username(username: str) -> list[str]:
    """Return a list of problems with a username, empty if it is acceptable."""
    problems: list[str] = []
    if not USERNAME_MIN_LENGTH <= len(username) <= USERNAME_MAX_LENGTH:
        problems.append(
            f"username must be {USERNAME_MIN_LENGTH} to "
            f"{USERNAME_MAX_LENGTH} characters"
        )
    if not (username.isascii() and username.isalnum()):
        problems.append("username may contain only English letters and digits")
    return problems


def check_email(email: str) -> list[str]:
    """Return a list of problems with an email address."""
    local, separator, domain = email.strip().partition("@")
    if not (separator and local and "." in domain):
        return ["email address is not valid"]
    return []


def check_password(password: str, username: str) -> list[str]:
    """Return a list of problems with a password."""
    problems: list[str] = []
    if len(password) < PASSWORD_MIN_LENGTH:
        problems.append(f"password must be at least {PASSWORD_MIN_LENGTH} characters")
    if username and username.casefold() in password.casefold():
        problems.append("password must not contain the username")
    if password.isalpha() or password.isdigit():
        problems.append("password must mix letters and digits or symbols")
    return problems


def normalize_email(email: str) -> str:
    """Return the canonical form used for storing and comparing emails."""
    return email.strip().casefold()


def validate_signup(username: str, email: str, password: str) -> SignUp:
    """Validate every field and return a SignUp, or raise ValidationError.

    All problems are reported together so the user can fix them in one go.
    The password is never logged.

    Raises:
        ValidationError: With a message listing every problem found.
    """
    problems = [
        *check_username(username),
        *check_email(email),
        *check_password(password, username),
    ]
    if problems:
        logger.info("Rejected sign-up with %d problem(s)", len(problems))
        raise ValidationError("; ".join(problems))
    return SignUp(username=username, email=normalize_email(email))
```

Try it:

```python
signup = validate_signup("ada99", " Ada@Example.COM ", "correct-horse-battery")
print(signup)

try:
    validate_signup("a!", "not-an-email", "ada")
except ValidationError as exc:
    for problem in str(exc).split("; "):
        print("-", problem)
```

```text
SignUp(username='ada99', email='ada@example.com')
- username must be 3 to 20 characters
- username may contain only English letters and digits
- email address is not valid
- password must be at least 12 characters
- password must mix letters and digits or symbols
```

Why it is written this way:

- Each `check_*` function is **pure**: it takes values and returns a list. They can be tested without any setup, and reused elsewhere.
- Returning a list of problems (empty meaning "fine") means the caller never has to check for `None` or `False`.
- `validate_signup` is the only function with a side effect, and its log line carries a count, not user data.
- The star syntax `[*a, *b, *c]` joins the three lists into one, a use of unpacking from Chapter 2.2.
- Names are verbs or questions (`check_username`, `normalize_email`) and each docstring states what the function returns.

## Common mistakes

1. **Using `print` where `return` is needed.** The caller receives `None`.
2. **Forgetting the parentheses when calling.** `greet` is the function object, `greet("Ada")` runs it.
3. **Forgetting the colon or the indentation** in the `def` line and body.
4. **Calling a function before its `def` has run.**
5. **Not returning on every path.** The missing paths silently return `None`.
6. **Writing code after `return`.** It never runs.
7. **Giving two functions the same name.** The later one replaces the earlier one.
8. **Modifying an argument without saying so.** Callers are surprised (Chapter 2.5).
9. **Writing one huge function that does everything.** Split it by responsibility.
10. **Using vague names** such as `process`, `handle`, or `do_it`. Say what is processed or handled.
11. **Returning different types on different paths**, such as a list in one case and `False` in another.
12. **Putting input and output inside calculation functions.** Keep calculation pure.
13. **Forgetting the docstring** for anything another person will call.

## Practice exercises

1. Write a function `greet(name)` that prints a greeting, and a second function `make_greeting(name)` that **returns** it. Show a situation where only the second one is useful.
2. Write `area_of_circle(radius)` that returns the area, using `math.pi`. Call it for three radii and print the results.
3. Write `is_even(n)` that returns `True` or `False`, and use it to print which of the numbers 1 to 10 are even.
4. Write `describe(numbers)` that returns the minimum, maximum, and average of a list, and unpack the result into three names.
5. Write a function `first_vowel(text)` that returns the first vowel in the text, or `None` if there is none. Show how a caller should use the result safely.
6. Write `classify_bmi(weight_kg, height_m)` with early returns for each category, and test the boundary values.
7. Rewrite this function so it returns the result instead of printing it, then use it in a calculation:

   ```python
   def total_price(price, quantity):
       print(price * quantity)
   ```

8. Write `count_words(text)` with a Google-style docstring, a type hint on the parameter and the return value, and print its `help()`.
9. Write three functions, `read_numbers(text)`, `average(numbers)`, and `show_report(avg)`, where only the last one prints. Explain why this is easier to test than one function that does all three.
10. Find the bug: a function is supposed to return the larger of two numbers but sometimes returns `None`. Write a version with a missing `else` path, show the failure, and fix it.
11. Write a function `safe_divide(a, b)` and decide how to handle `b == 0`. Compare the design choices of returning `None`, returning `0`, and raising `ValueError`. Which would you pick and why?
12. Extend the sign-up validator with `check_age(age)` that rejects non-positive ages and anyone younger than a minimum, and include it in `validate_signup` via a new optional parameter.

## Summary

- A function is defined with `def name(parameters):` and an indented body, and run by calling `name(arguments)`.
- Parameters are the names in the definition, arguments are the values passed in a call.
- `return` sends a value back and ends the function. Without a `return`, the function returns `None`.
- `print` shows a value, `return` gives it to the caller. Use `return` for anything you want to use later.
- Return several values as a tuple and unpack them at the call site.
- A docstring as the first statement documents the function, and type hints describe its inputs and output.
- Each call has its own local variables, and functions are ordinary objects with a name, a docstring, and annotations.
- Prefer small, pure functions with clear verb names, and keep side effects such as printing, logging, and file access at the edges.
- Return the same kind of value on every path, and use early returns to keep code flat.
