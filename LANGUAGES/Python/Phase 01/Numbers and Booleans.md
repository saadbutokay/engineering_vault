## Overview

Numbers and true-or-false values are the raw material of almost every program: prices, counts, durations, flags, permissions, and conditions. Python offers several numeric types, and choosing the wrong one is a classic source of subtle bugs, especially when money is involved.

This chapter covers the numeric types, every arithmetic, comparison, logical, and bitwise operator, how Python decides whether a value counts as true or false, and the everyday parts of the `math` module.

By the end of this note you will be able to:

- Choose between `int`, `float`, `Decimal`, and `Fraction` for a given job.
- Use arithmetic, comparison, logical, and bitwise operators correctly.
- Explain why `0.1 + 0.2` is not exactly `0.3` and how to handle it.
- Predict the truthiness of any value.
- Use the `math` module for common calculations.

> [!warning] Money is not a float
> Never store prices or balances in `float`. This chapter shows why, and the industry-level snippet shows two safe alternatives.

## Core concepts

### The numeric types

| Type | Examples | Notes |
| --- | --- | --- |
| `int` | `42`, `-7`, `1_000_000`, `0xFF` | Whole numbers of unlimited size |
| `float` | `3.14`, `2.5e6`, `1e-3` | 64-bit binary floating point, about 15 to 17 significant digits |
| `complex` | `3 + 4j` | Real and imaginary parts, used in science and engineering |
| `bool` | `True`, `False` | A subtype of `int` |
| `Decimal` | `Decimal("0.10")` | Exact decimal arithmetic, from the `decimal` module |
| `Fraction` | `Fraction(1, 3)` | Exact fractions, from the `fractions` module |

### Number literals

```python
population = 8_000_000_000   # underscores for readability
mask = 0xFF                  # hexadecimal, 255
flags = 0b1010               # binary, 10
mode = 0o755                 # octal, 493
speed_of_light = 3.0e8       # scientific notation
```

### Arithmetic operators

| Operator | Meaning | Example | Result |
| --- | --- | --- | --- |
| `+` | Addition | `7 + 2` | `9` |
| `-` | Subtraction | `7 - 2` | `5` |
| `*` | Multiplication | `7 * 2` | `14` |
| `/` | True division, always a float | `7 / 2` | `3.5` |
| `//` | Floor division (rounds down) | `7 // 2` | `3` |
| `%` | Remainder (modulo) | `7 % 2` | `1` |
| `**` | Power | `2 ** 10` | `1024` |

Order of evaluation, from first to last:

1. Parentheses `( )`
2. Power `**` (it groups right to left)
3. Unary signs: `-x`, `+x`, `~x`
4. `*`, `/`, `//`, `%`
5. `+`, `-`
6. Bit shifts `<<`, `>>`
7. Bitwise and `&`, then xor `^`, then or `|`
8. Comparisons
9. `not`, then `and`, then `or`

When in doubt, add parentheses. Readers will thank you.

### Comparison operators

| Operator | Meaning |
| --- | --- |
| `==` | Equal values |
| `!=` | Different values |
| `<`, `<=` | Less than, less than or equal |
| `>`, `>=` | Greater than, greater than or equal |

Comparisons can be **chained**: `0 < x < 10` means `0 < x and x < 10`, and `x` is evaluated only once.

### Logical operators

| Operator | Meaning |
| --- | --- |
| `and` | True when both sides are true |
| `or` | True when at least one side is true |
| `not` | Reverses true and false |

Both `and` and `or` use **short-circuit evaluation**: they stop as soon as the answer is known. They also **return one of their operands**, not necessarily `True` or `False`.

- `a or b` returns `a` if `a` is truthy, otherwise `b`.
- `a and b` returns `a` if `a` is falsy, otherwise `b`.

### Bitwise operators

Bitwise operators work on the binary digits of integers.

- `a & b`: a bit is 1 only if it is 1 in both
- `a | b`: a bit is 1 if it is 1 in either
- `a ^ b`: a bit is 1 if it is 1 in exactly one
- `~a`: flips every bit
- `a << n`: shifts bits left by n places (multiplies by 2 to the power n)
- `a >> n`: shifts bits right by n places

### Truthiness and falsiness

Every object can be used as a condition. These are **falsy**:

| Falsy value | Type |
| --- | --- |
| `False` | bool |
| `None` | NoneType |
| `0`, `0.0`, `0j`, `Decimal("0")` | numbers |
| `""` | empty string |
| `[]`, `()`, `{}`, `set()`, `range(0)` | empty containers |

**Everything else is truthy**, including the string `"0"`, the string `"False"`, and a list holding a falsy item such as `[0]`.

### Converting between types

`int()`, `float()`, and `bool()` convert values. `int()` on a float drops the fractional part (truncation toward zero). `int()` on a string accepts only whole numbers, optionally with surrounding spaces and a sign.

### Built-in helpers and the math module

| Function | Purpose |
| --- | --- |
| `abs(x)` | Absolute value |
| `round(x, n)` | Round to n decimal places |
| `min(...)`, `max(...)` | Smallest, largest |
| `sum(items)` | Total of an iterable |
| `divmod(a, b)` | Quotient and remainder together |
| `pow(a, b)` | Same as `a ** b` |

The `math` module (`import math`) adds `sqrt`, `floor`, `ceil`, `trunc`, `pi`, `e`, `log`, `log10`, `gcd`, `factorial`, `isqrt`, `hypot`, `isclose`, `isnan`, `inf`, and `nan`.

## Basic syntax examples

### Arithmetic

```text
>>> 7 / 2
3.5
>>> 7 // 2
3
>>> -7 // 2
-4
>>> 7 % 3
1
>>> -7 % 3
2
>>> 2 ** 10
1024
>>> 2 ** -1
0.5
>>> divmod(17, 5)
(3, 2)
>>> 10 / 2
5.0
>>> 7.5 // 2
3.0
>>> 3 + 4.0
7.0
```

Note that `/` always gives a float, even when the result is whole. `//` rounds toward negative infinity, so `-7 // 2` is `-4`, not `-3`.

### Precedence

```text
>>> 2 + 3 * 4
14
>>> (2 + 3) * 4
20
>>> -2 ** 2
-4
>>> 2 ** 3 ** 2
512
```

### Big integers and literals

```text
>>> 2 ** 100
1267650600228229401496703205376
>>> 1_000_000 + 0xFF + 0b101
1000260
```

Python integers never overflow. They grow as large as memory allows.

### Conversions

```text
>>> int("42")
42
>>> int(" 42 ")
42
>>> int(3.9)
3
>>> int(-3.9)
-3
>>> float("3.14")
3.14
>>> float("1e3")
1000.0
>>> int("ff", 16)
255
>>> int("3.5")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: invalid literal for int() with base 10: '3.5'
```

### Comparisons and chaining

```text
>>> 3 == 3.0
True
>>> 0 < 5 < 10
True
>>> 10 > 5 > 7
False
```

### Logical operators return operands

```text
>>> True and False
False
>>> not True
False
>>> "" or "default"
'default'
>>> 0 or 10
10
>>> 3 and 4
4
>>> 0 and 4
0
>>> None or []
[]
```

The `value or default` pattern is common, but it replaces any falsy value, including `0`. Use it only when that is what you want.

### Truthiness

```text
>>> bool(0), bool(0.0), bool(""), bool([]), bool(None)
(False, False, False, False, False)
>>> bool(1), bool(-3), bool("0"), bool("False"), bool([0])
(True, True, True, True, True)
```

### Booleans are integers

```text
>>> True + True
2
>>> isinstance(True, int)
True
>>> sum([True, False, True, True])
3
>>> int(True)
1
```

Because `True` counts as `1`, summing a list of booleans counts how many are true.

### Bitwise operations

```text
>>> 5 & 3
1
>>> 5 | 3
7
>>> 5 ^ 3
6
>>> ~5
-6
>>> 1 << 3
8
>>> 16 >> 2
4
>>> bin(10)
'0b1010'
```

### The math module

```text
>>> import math
>>> math.sqrt(16)
4.0
>>> math.floor(-2.5), math.ceil(-2.5), math.trunc(-2.5)
(-3, -2, -2)
>>> math.pi
3.141592653589793
>>> math.gcd(12, 18)
6
>>> math.factorial(5)
120
>>> math.isqrt(17)
4
>>> math.hypot(3, 4)
5.0
>>> math.log10(1000)
3.0
>>> math.sqrt(-1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: math domain error
```

### Division by zero

```text
>>> 1 / 0
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ZeroDivisionError: division by zero
```

### Complex numbers

```text
>>> z = 3 + 4j
>>> z.real, z.imag
(3.0, 4.0)
>>> abs(z)
5.0
>>> z * z
(-7+24j)
```

## Deeper behavior

### Why `0.1 + 0.2` is not `0.3`

Computers store floats in binary. Just as one third cannot be written exactly as a decimal (0.3333...), the decimal number 0.1 cannot be written exactly in binary. Python stores the closest value it can.

```text
>>> 0.1 + 0.2
0.30000000000000004
>>> 0.1 + 0.2 == 0.3
False
>>> format(0.1, ".20f")
'0.10000000000000000555'
>>> 1.1 * 3
3.3000000000000003
```

This is not a Python bug. Every language that uses standard IEEE 754 doubles behaves the same way.

Rules that follow from this:

- Do not compare floats with `==`. Use `math.isclose`.
- Do not use floats for money.
- Expect tiny errors to accumulate in long calculations.

```text
>>> import math
>>> math.isclose(0.1 + 0.2, 0.3)
True
```

### Large floats lose whole-number precision

```text
>>> 9007199254740992.0 + 1
9007199254740992.0
```

Above about nine quadrillion, a float cannot represent every integer. Python's `int` has no such limit.

### Special float values

```text
>>> 1e308 * 10
inf
>>> float("nan") == float("nan")
False
>>> import math
>>> math.isnan(float("nan"))
True
```

`inf` is infinity. `nan` ("not a number") is never equal to anything, including itself, so test for it with `math.isnan`.

### Decimal: exact decimal arithmetic

```text
>>> from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')
>>> Decimal("0.1") + Decimal("0.2") == Decimal("0.3")
True
>>> Decimal(0.1)
Decimal('0.1000000000000000055511151231257827021181583404541015625')
```

Always create a `Decimal` from a **string**. Building it from a float copies the float's inexact value, which defeats the purpose.

`Decimal` and `float` do not mix:

```text
>>> from decimal import Decimal
>>> Decimal("1.1") + 1.1
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for +: 'decimal.Decimal' and 'float'
```

Python forces you to decide. Mixing `Decimal` with `int` is fine.

### Rounding

```text
>>> round(2.5)
2
>>> round(3.5)
4
>>> round(2.675, 2)
2.67
```

`round()` uses **banker's rounding**: exact halves go to the nearest even number. It also works on the stored binary value, which is why `round(2.675, 2)` gives `2.67` (the stored value is slightly below 2.675). For business rounding, use `Decimal` and choose the rule explicitly:

```text
>>> from decimal import Decimal, ROUND_HALF_UP
>>> Decimal("2.665").quantize(Decimal("0.01"))
Decimal('2.66')
>>> Decimal("2.665").quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
Decimal('2.67')
```

### Fraction: exact rationals

```text
>>> from fractions import Fraction
>>> Fraction(1, 3) + Fraction(1, 6)
Fraction(1, 2)
>>> Fraction("0.25")
Fraction(1, 4)
>>> float(Fraction(1, 3))
0.3333333333333333
```

### Floor division and modulo with negative numbers

`//` always rounds down and `%` takes the sign of the divisor. They obey one rule: `a == (a // b) * b + a % b`.

```text
>>> (-7 // 2) * 2 + (-7 % 2)
-7
```

This behavior makes `%` convenient for wrapping around, such as cycling through array positions.

### Booleans in disguise

Because `bool` is a subtype of `int`, `True == 1` and `False == 0`. That has side effects, for example in dictionaries where `1` and `True` count as the same key. Do not rely on it, but do not be surprised by it.

### Short-circuiting is useful and dangerous

```python
user = None
if user is not None and user.is_active:   # safe: second part skipped when user is None
    print("active")
```

The second condition would crash on `None`, but it is never evaluated. Likewise, `x or default` skips evaluating `default` if `x` is truthy.

### Bit flags

Bitwise operators let you pack several on-off options into one integer, a common technique for permissions.

```python
READ = 0b001
WRITE = 0b010
DELETE = 0b100

permissions = READ | WRITE
print(bool(permissions & WRITE))    # is WRITE set?
print(bool(permissions & DELETE))   # is DELETE set?

permissions |= DELETE               # turn DELETE on
permissions &= ~WRITE               # turn WRITE off
print(bin(permissions))
```

```text
True
False
0b101
```

Chapter 6.8 shows a cleaner way with `enum.Flag`.

### Integer to text conversion limits

Since Python 3.11, converting an integer with more than 4300 digits to or from a string raises `ValueError`. This protects servers from denial-of-service attacks that send absurdly long numbers. Normal programs never hit it.

### Mixing types

`int` with `float` gives a `float`. `int` with `Decimal` gives a `Decimal`. `float` with `Decimal` is an error. `bool` acts as an `int`. Python picks the "wider" type when it is safe and refuses when it is not.

## Industry-level snippet

Payment systems avoid floats completely. A common approach is to store money as **integer cents**, so that every calculation is exact, and use `Decimal` only at the boundary where text from a user is parsed. This module parses amounts, formats them, and splits a total fairly across several payers without losing or inventing a single cent.

```python
"""Money helpers: parse text into integer cents and split amounts fairly."""

import logging
from decimal import ROUND_HALF_UP, Decimal, InvalidOperation

logger = logging.getLogger(__name__)

CENTS_PER_UNIT = 100
MAX_UNITS = Decimal("1000000000")  # a sanity limit of one billion, not a business rule


class MoneyError(ValueError):
    """Raised when an amount cannot be parsed, formatted, or split."""


def parse_to_cents(text: str) -> int:
    """Convert text such as "1,234.50" into a whole number of cents.

    Args:
        text: The amount as typed by a user. Commas are ignored.

    Returns:
        The amount in cents, rounded half up.

    Raises:
        MoneyError: If the text is not a finite, non-negative amount in range.
    """
    cleaned = text.strip().replace(",", "")
    try:
        amount = Decimal(cleaned)
    except InvalidOperation as exc:
        raise MoneyError(f"not a valid amount: {text!r}") from exc
    # Decimal accepts the words "NaN" and "Infinity", so check explicitly.
    if not amount.is_finite():
        raise MoneyError(f"amount must be a finite number: {text!r}")
    if amount < 0 or amount > MAX_UNITS:
        raise MoneyError(f"amount out of range: {text!r}")
    cents = (amount * CENTS_PER_UNIT).quantize(Decimal("1"), rounding=ROUND_HALF_UP)
    return int(cents)


def format_cents(cents: int) -> str:
    """Format a non-negative number of cents as text such as "1,234.50"."""
    if cents < 0:
        raise MoneyError("cannot format a negative amount")
    units, remainder = divmod(cents, CENTS_PER_UNIT)
    return f"{units:,}.{remainder:02d}"


def split_evenly(total_cents: int, parts: int) -> list[int]:
    """Split an amount into shares that differ by at most one cent.

    The shares always add up to exactly total_cents.

    Raises:
        MoneyError: If parts is less than 1 or total_cents is negative.
    """
    if parts < 1:
        raise MoneyError("parts must be at least 1")
    if total_cents < 0:
        raise MoneyError("total must not be negative")
    base, extra = divmod(total_cents, parts)
    shares = [base + 1 if index < extra else base for index in range(parts)]
    logger.debug("Split %d cents into %d shares", total_cents, parts)
    return shares
```

Try it:

```python
cents = parse_to_cents("100.00")
shares = split_evenly(cents, 3)

print(cents)
print(shares)
print(sum(shares) == cents)
print([format_cents(share) for share in shares])
print(parse_to_cents("1,234.505"))
```

```text
10000
[3334, 3333, 3333]
True
['33.34', '33.33', '33.33']
123451
```

Why it is written this way:

- All arithmetic after parsing uses `int`, which is exact. There is no rounding error to accumulate.
- `divmod` returns the quotient and remainder in one step, which is the natural way to split cents and format amounts.
- The `is_finite` check closes a hole: `Decimal("NaN")` and `Decimal("Infinity")` parse successfully, so without it a malicious input could slip through.
- The range check comes before the multiplication, so absurdly large exponents such as `1e999` cannot cause trouble.
- Rounding rules are explicit (`ROUND_HALF_UP`), not left to a default.
- A custom `MoneyError` lets callers handle money problems without catching unrelated errors.

## Common mistakes

1. **Using `float` for money.** Use integer cents or `Decimal`.
2. **Comparing floats with `==`.** Use `math.isclose` with a sensible tolerance.
3. **Creating `Decimal` from a float.** Use a string: `Decimal("0.1")`.
4. **Expecting `/` to return an integer.** It always returns a float. Use `//` for whole-number division.
5. **Forgetting that `//` and `%` behave unexpectedly with negatives.** Test them before relying on them.
6. **Using `or` to supply defaults when `0` is valid.** `retries = value or 3` replaces `0` by `3`.
7. **Confusing `=` with `==`.** One assigns, the other compares.
8. **Writing `if x == True:`.** Write `if x:`, or `if x is True:` when you really need the exact boolean.
9. **Mixing up `and`/`or` with `&`/`|`.** The first pair are logical and short-circuit. The second pair are bitwise.
10. **Relying on `round()` for money.** It uses banker's rounding on binary floats.
11. **Assuming `int("3.5")` works.** Convert with `int(float("3.5"))` if you really mean it, or better, validate input first.
12. **Using `==` with `nan`.** It is never equal to anything. Use `math.isnan`.

## Practice exercises

1. Without running code, predict the results of `17 // 5`, `17 % 5`, `-17 // 5`, `-17 % 5`, and `2 ** 3 ** 2`. Then check them.
2. Write a program that converts a number of seconds into hours, minutes, and seconds, using `divmod`.
3. Calculate the hypotenuse of a right triangle with sides 5 and 12 in two ways: with `**` and `math.sqrt`, and with `math.hypot`.
4. Show that `0.1 + 0.2` is not equal to `0.3`, then write a correct comparison. Print `0.1` formatted to 25 decimal places and explain what you see.
5. Add the price `19.99` ten times using floats, then using `Decimal`. Compare the totals.
6. Predict the output of each expression, then verify: `"" or "guest"`, `"Ada" or "guest"`, `0 and 5`, `5 and 0`, `None or 0 or []`, `not []`.
7. Write down every falsy value you can think of, then test each with `bool()`. Add three values you think are falsy but turn out not to be.
8. Use bit flags to model four permissions (read, write, delete, share). Write code that grants all four, revokes one, and tests for each.
9. Write a function `is_leap_year(year)` using only comparisons, `%`, `and`, `or`, and `not`. Test it on 1900, 2000, 2024, and 2100.
10. Explain, with an example, why `round(2.5)` and `round(3.5)` are not rounded the same direction. Then write the same calculation with `Decimal` and half-up rounding.
11. Using `parse_to_cents`, test the inputs `"abc"`, `"NaN"`, `"-5"`, `"1e999"`, `"0.005"`, and `"  12.5  "`. Explain each result.
12. Extend the industry snippet with `apply_percentage(cents, percent)` that returns a percentage of an amount in cents, such as 7.5 percent tax. Use `Decimal` for the percent, round half up, and write a short test showing the result for 19999 cents at 7.5 percent.

## Summary

- Python has `int` (unlimited), `float` (binary approximation), `complex`, `bool`, and, from the standard library, `Decimal` and `Fraction`.
- `/` always returns a float, `//` rounds down, `%` returns the remainder, and `**` is the power operator.
- Comparisons can be chained. `and` and `or` short-circuit and return one of their operands.
- Bitwise operators `&`, `|`, `^`, `~`, `<<`, `>>` work on the binary digits of integers and are useful for flags.
- Falsy values are `False`, `None`, zero in any numeric type, and empty containers. Everything else is truthy.
- Floats cannot represent most decimal fractions exactly. Use `math.isclose` to compare, and never use floats for money.
- Build `Decimal` values from strings, choose rounding rules explicitly, or store money as integer cents.
- `bool` is a subtype of `int`, so `True + True` is `2`.
- The `math` module provides `sqrt`, `floor`, `ceil`, `gcd`, `isclose`, `pi`, and more.
