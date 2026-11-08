## Overview

In many languages a variable is a box that holds a value. Python works differently, and understanding the difference early prevents some of the most confusing bugs you will ever meet. In Python, **everything is an object**, and a **variable is a name that points to an object**.

This chapter explains what that means, how assignment really works, what it means for something to be mutable, and why `is` and `==` are not the same operator.

By the end of this note you will be able to:

- Explain the difference between a name and an object.
- Use `type()`, `id()`, and `isinstance()` to inspect objects.
- Describe dynamic typing and why Python is still strongly typed.
- Tell mutable objects from immutable ones and predict how each behaves when shared.
- Choose between `is` and `==` correctly, and use `None` properly.
- Follow Python's naming rules.

> [!note] A useful picture
> Think of a name as a sticky label and an object as a thing in a room. Assignment sticks a label onto a thing. You can move a label to a different thing, or stick two labels on the same thing, but putting a label on something never copies it.

## Core concepts

### Objects have identity, type, and value

Every object has three properties:

| Property | Meaning | How to inspect it |
| --- | --- | --- |
| Identity | A unique number for the object while it exists. In CPython it is related to its memory address | `id(obj)` |
| Type | What kind of object it is, which decides what it can do | `type(obj)` |
| Value | The data it holds | Print it or evaluate it |

The identity never changes during the object's life. The type never changes either. The value may change, but only for certain types (see mutability below).

### Names and assignment

```python
price = 10
```

This statement does three things: Python creates an integer object holding `10`, creates (or reuses) the name `price`, and binds that name to the object. Assignment is binding, not copying and not storing inside a box.

Rebinding a name to something else does not change the original object:

```python
price = 10
price = 12
```

Now `price` points to a different integer. The object `10` was not modified, it simply has no label from this name any more.

### Dynamic typing and strong typing

Python is **dynamically typed**: names have no fixed type. The same name can point to an integer now and a string a moment later. Types belong to objects, and they are checked when an operation runs.

Python is also **strongly typed**: it does not silently convert between unrelated types. Adding a string to a number is an error, not a guess.

| Term | Python's behavior |
| --- | --- |
| Dynamic | A name can be rebound to objects of different types |
| Strong | `"5" + 5` raises `TypeError`, it does not produce `10` or `"55"` |

### Mutable and immutable objects

A **mutable** object can be changed in place after it is created. An **immutable** object cannot, any "change" produces a new object.

| Immutable | Mutable |
| --- | --- |
| `int`, `float`, `bool`, `complex` | `list` |
| `str` | `dict` |
| `tuple` | `set` |
| `frozenset` | `bytearray` |
| `bytes` | Most user-defined class instances |
| `None` | |

This matters most when several names share one object, as the next sections show.

### Identity versus equality: `is` and `==`

- `==` asks: do these two objects have **equal values**?
- `is` asks: are these two names pointing to the **very same object**?

Two separate lists with the same contents are equal but not identical.

### None

`None` is a special object meaning "no value here". There is exactly one `None` object in a running program, so you compare to it with `is`:

```python
if result is None:
    ...
```

A function that does not execute a `return` statement returns `None` automatically. `None` is **falsy**, but so are `0` and `""`, so test `is None` when you specifically mean "missing".

### Naming rules

| Rule | Valid | Invalid |
| --- | --- | --- |
| Letters, digits, underscores only | `user_name`, `total2`, `_cache` | `user-name`, `total$`, `my var` |
| Must not start with a digit | `item1` | `1item` |
| Case-sensitive | `Count` and `count` differ | not an error, but confusing |
| Cannot be a keyword | `class_name` | `class`, `for`, `if`, `None` |

Python has about 35 reserved **keywords**. You can list them with the `keyword` module. A few words, such as `match`, `case`, and `type`, are **soft keywords** and still work as ordinary names in most places, but avoid them anyway.

Follow the style rules from Chapter 0.7: `snake_case` for variables, `UPPER_SNAKE_CASE` for constants, and never reuse built-in names such as `list`, `str`, `id`, or `sum`.

## Basic syntax examples

### Assignment and inspection

```python
count = 3
label = "items"

print(type(count))
print(type(label))
print(id(count))      # a large number, different on every run
print(isinstance(count, int))
```

```text
<class 'int'>
<class 'str'>
140234866534704
True
```

The exact `id` value on your machine will differ. Use `isinstance(obj, SomeType)` rather than comparing `type(obj) == SomeType`, because `isinstance` also accepts subclasses.

### Dynamic typing

```text
>>> x = 42
>>> type(x)
<class 'int'>
>>> x = "now a string"
>>> type(x)
<class 'str'>
>>> isinstance(x, str)
True
```

### Strong typing

```text
>>> "5" + 5
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can only concatenate str (not "int") to str
```

You must be explicit about what you want: `int("5") + 5` gives `10`, and `"5" + str(5)` gives `"55"`.

### Multiple assignment, swapping, and augmented assignment

```text
>>> a, b = 1, 2
>>> a, b = b, a
>>> a, b
(2, 1)
>>> x = y = 0
>>> x += 5
>>> x, y
(5, 0)
```

Python can swap two names in one line without a temporary variable. `x += 5` is shorthand for `x = x + 5`. The other augmented operators are `-=`, `*=`, `/=`, `//=`, `%=`, and `**=`.

### Names pointing at the same object

```text
>>> a = [1, 2, 3]
>>> b = a
>>> b.append(4)
>>> a
[1, 2, 3, 4]
>>> a is b
True
>>> c = [1, 2, 3, 4]
>>> a == c
True
>>> a is c
False
```

`b = a` did not copy the list. It put a second label on the same list, so a change through `b` is visible through `a`. The list `c` has equal contents but is a different object.

### Immutable objects behave differently

```text
>>> name = "ada"
>>> name.upper()
'ADA'
>>> name
'ada'
>>> numbers = [3, 1, 2]
>>> numbers.sort()
>>> numbers
[1, 2, 3]
```

The string method returned a new string and left `name` alone. The list method changed the list in place.

### Sharing immutable data is safe

```text
>>> t = (1, 2)
>>> u = t
>>> t += (3,)
>>> t, u
((1, 2, 3), (1, 2))
>>> a = [1, 2]
>>> b = a
>>> a += [3]
>>> a, b
([1, 2, 3], [1, 2, 3])
```

For the tuple, `+=` built a new tuple and rebound `t`, so `u` is unaffected. For the list, `+=` extended the existing list in place, so both names see the change. The same operator behaves differently depending on mutability.

### Using None

```python
def find_user(user_id):
    users = {1: "Ada", 2: "Grace"}
    return users.get(user_id)   # returns None when the key is missing


name = find_user(3)
if name is None:
    print("No such user")
else:
    print(f"Found {name}")
```

```text
No such user
```

### Deleting a name

```text
>>> x = 10
>>> del x
>>> x
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'x' is not defined
```

`del` removes the name, not necessarily the object. The object is cleaned up only when nothing refers to it any more.

### Checking for keywords

```text
>>> import keyword
>>> keyword.iskeyword("for")
True
>>> keyword.iskeyword("print")
False
```

`print` is a built-in function, not a keyword. You *can* assign to it, but you should not.

## Deeper behavior

### Passing objects to functions

When you call a function, the parameter name inside the function is bound to the same object as the argument. This is often called "pass by object reference" or "pass by assignment".

```python
def add_item(items):
    items.append("new")      # mutates the shared object


def reset(items):
    items = []               # rebinds the local name only


shopping = ["milk"]
add_item(shopping)
print(shopping)              # the caller sees the change

reset(shopping)
print(shopping)              # the caller does not
```

```text
['milk', 'new']
['milk', 'new']
```

Mutating the object affects everyone who shares it. Rebinding the name inside the function does not touch the caller's name. This is why a function that modifies its argument in place can surprise its caller.

### New object or same object?

```python
text = "abc"
before = id(text)
text += "d"
print(id(text) == before)

numbers = [1, 2]
before = id(numbers)
numbers += [3]
print(id(numbers) == before)
```

```text
False
True
```

This is a direct way to see immutability: the string "changed" by becoming a different object, the list stayed the same object.

### Why `is` is wrong for comparing values

CPython reuses some objects as an optimization. For example, small integers (roughly -5 to 256) are pre-created and shared, and some identical strings may be shared. That means `is` can appear to work on numbers and short strings, and then fail on larger or computed ones. It is an implementation detail you must not rely on.

Python warns about the most obvious misuse:

```python
x = 5
if x is 5:
    print("five")
```

```text
SyntaxWarning: "is" with 'int' literal. Did you mean "=="?
```

The rule is simple: use `==` to compare values, use `is` only for `None` (and, rarely, `True` and `False` or deliberate identity checks).

### What `==` really does

`a == b` asks the object `a` how to compare itself to `b`. For built-in types the rules are sensible: lists are equal when they have equal items in the same order, and `1 == 1.0` is `True`. For your own classes, you define the behavior (Chapter 6.3).

### Reference counting and cleanup

CPython tracks how many names and containers refer to each object. When that count reaches zero, the object is destroyed and its memory is released. A garbage collector handles tricky cases such as objects that refer to each other. Chapter 9.5 covers this. For now, remember that objects live as long as something refers to them.

### Variables in containers are references too

A list does not contain objects, it contains references to them. Copying a list copies the references, not the objects they point to. Chapter 2.5 shows how this causes shallow-copy bugs with nested lists.

### Type hints do not change how names work

You will later write `count: int = 3`. The annotation documents intent and helps tools such as mypy (Chapter 0.9). Python does not enforce it at run time, and the name can still be rebound to any object.

### Constants are a convention

Python has no true constants. `MAX_RETRIES = 3` is a signal to other developers that the value should not change. Nothing stops you from reassigning it.

### Unpacking

Assignment can take apart sequences:

```python
first, second = [10, 20]
head, *rest = [1, 2, 3, 4]
print(first, second, head, rest)
```

```text
10 20 1 [2, 3, 4]
```

A name starting with `*` collects the remaining items in a list. Chapter 2.2 covers this in depth.

## Industry-level snippet

Web backends build headers and read optional request parameters all day. Two classic traps appear: mutating a dictionary the caller still owns, and confusing "missing" with "empty or zero". This module avoids both, using `None` deliberately and creating a new object instead of modifying the argument.

```python
"""Helpers for building outbound request headers and reading paging parameters."""

import logging

logger = logging.getLogger(__name__)

DEFAULT_PAGE_SIZE = 20
MAX_PAGE_SIZE = 100


def build_headers(
    base: dict[str, str],
    extra: dict[str, str] | None = None,
    token: str | None = None,
) -> dict[str, str]:
    """Return a new headers dictionary without modifying any argument.

    Args:
        base: Headers shared by every request. Never modified.
        extra: Optional per-request headers that override base values.
        token: Optional bearer token. None means "send no credentials".

    Returns:
        A fresh dictionary containing the combined headers.

    Raises:
        ValueError: If a token is supplied but is blank.
    """
    headers = dict(base)  # a new object, so the caller's dict stays untouched
    if extra is not None:
        headers.update(extra)
    if token is not None:
        if not token.strip():
            raise ValueError("token must not be blank")
        headers["Authorization"] = f"Bearer {token}"
    # Log only the header names: values may contain credentials.
    logger.debug("Built request headers: %s", sorted(headers))
    return headers


def parse_page_size(raw: str | None, default: int = DEFAULT_PAGE_SIZE) -> int:
    """Convert a query-string value into a validated page size.

    Args:
        raw: The text of the parameter, or None if it was not supplied.
        default: The value to use when the parameter is absent.

    Returns:
        A page size between 1 and MAX_PAGE_SIZE.

    Raises:
        ValueError: If the text is not a whole number in the allowed range.
    """
    if raw is None:  # absent is different from empty or invalid
        return default
    try:
        size = int(raw)
    except ValueError as exc:
        raise ValueError(f"page size must be a whole number, got {raw!r}") from exc
    if not 1 <= size <= MAX_PAGE_SIZE:
        raise ValueError(f"page size must be between 1 and {MAX_PAGE_SIZE}, got {size}")
    return size
```

Check the behavior yourself:

```python
shared = {"Accept": "application/json"}
combined = build_headers(shared, {"X-Trace": "abc"}, token="secret")

print(shared)                  # unchanged
print(combined is shared)      # a different object
print(parse_page_size(None))   # default
print(parse_page_size("50"))
```

```text
{'Accept': 'application/json'}
False
20
50
```

Why this is production style:

- `dict(base)` creates a new object, so two requests can never corrupt each other's headers through a shared dictionary.
- `is None` and `is not None` separate "not provided" from "provided but wrong". An empty string for `raw` is an error here, not a silent default.
- Secrets never appear in the log, only the header names.
- Specific, informative errors include the offending value, and `from exc` keeps the original cause attached.

## Common mistakes

1. **Thinking `b = a` copies a list or dictionary.** It creates a second name for the same object.
2. **Mutating an argument inside a function** and surprising the caller. Return a new object, or document the mutation clearly.
3. **Using `is` to compare numbers or strings.** It may work by accident and fail later. Use `==`.
4. **Using `== None`.** Write `is None`.
5. **Treating all falsy values as "missing".** `if not count:` is true for `0`. If zero is a valid value, test `count is None`.
6. **Reusing built-in names** such as `list`, `dict`, `id`, `type`, `input`, or `sum` as variable names. This hides the real function.
7. **Expecting a name to be locked to one type.** Reassignment to another type is legal, so mistakes surface only when an operation fails.
8. **Forgetting that `+=` on a list changes it in place**, while on a string or tuple it creates a new object.
9. **Starting a name with a digit or using a hyphen.** `2fast` and `user-id` are syntax errors.
10. **Assuming `del x` frees memory immediately.** It only removes one name.

## Practice exercises

1. Create variables holding an integer, a float, a string, a boolean, and `None`. Print the `type` of each.
2. Assign `x = 10`, then `y = x`, then `x = 20`. Predict `y`, then run it and explain why.
3. Assign `a = [1, 2, 3]` and `b = a`. Append a number to `b` and print both lists. Then make `c` a list with the same contents and print `a == c` and `a is c`.
4. Swap two variables using one line. Then do it again for three variables in a rotation (a gets b, b gets c, c gets a).
5. For each name, say whether it is valid and why: `total_2`, `2nd_place`, `my-var`, `_hidden`, `class`, `Total`, `user name`.
6. Write a function `add_tag(tags, tag)` that appends to the list it receives, and another version `with_tag(tags, tag)` that returns a new list without modifying the original. Demonstrate the difference with a caller's list.
7. Use `id()` to prove that `s += "x"` creates a new string but `lst += ["x"]` does not create a new list.
8. Write a function that returns `None` implicitly, call it, and print the result. Then rewrite your code to test it with `is None`.
9. Explain, with a short example, why `if not quantity:` is a bug when `0` is a legitimate quantity.
10. Predict the output before running: `a = (1, 2)`, `b = a`, `a += (3,)`, `print(a, b)`. Repeat with lists instead of tuples and explain the difference.
11. Extend `build_headers` so that it rejects header names that are blank and logs how many headers were produced. Keep all arguments unmodified, and write three calls that prove it.

## Summary

- Everything in Python is an object with an identity, a type, and a value. A variable is a name bound to an object.
- Assignment binds a name. It never copies the object.
- Python is dynamically typed (names can be rebound to any type) and strongly typed (types are not silently mixed).
- Mutable objects (`list`, `dict`, `set`) can change in place. Immutable objects (`int`, `str`, `tuple`, and others) are replaced instead.
- Shared mutable objects cause aliasing: a change through one name is visible through all names.
- `==` compares values. `is` compares identity and should be reserved for `None` checks.
- `None` means "no value". Use `is None` to detect it and do not confuse it with other falsy values.
- Names use letters, digits, and underscores, cannot start with a digit, and cannot be keywords.
