## Overview

Code is read far more often than it is written. You will read your own code months later, teammates will read it during review, and future students will read it while learning. Consistent style makes code easier to scan, easier to debug, and less likely to hide mistakes.

Python has an official style guide called **PEP 8** and a short statement of philosophy called **PEP 20**, the Zen of Python. A PEP is a Python Enhancement Proposal, a numbered design document. Learning these early means you form good habits instead of unlearning bad ones.

By the end of this note you will be able to:

- Format Python code according to PEP 8 conventions.
- Choose correct names for variables, functions, classes, constants, and modules.
- Explain the main ideas of the Zen of Python.
- Write comments and docstrings in Google style and NumPy style.

> [!note] Style versus correctness
> Style does not change what your program does. It changes how easily humans understand it. Later you will meet tools (Chapter 0.9) that apply most of these rules automatically, but you still need to know what they enforce and why.

## Core concepts

### PEP 8 in one page

| Topic | Convention |
| --- | --- |
| Indentation | 4 spaces per level, never tabs mixed with spaces |
| Line length | 79 characters in PEP 8, many teams use 88 or 100. Choose one and automate it |
| Blank lines | Two between top-level functions and classes, one between methods inside a class |
| Imports | One per line, at the top, grouped: standard library, third-party, local, with a blank line between groups |
| Spaces around operators | `x = a + b`, not `x=a+b` |
| Spaces inside brackets | `foo(a, b)`, not `foo( a, b )` |
| Trailing whitespace | None |
| Quotes | Pick one style for strings and stay consistent. Ruff and Black default to double quotes |
| End of file | A single newline |

### Naming conventions

| Thing | Style | Example |
| --- | --- | --- |
| Variable | `snake_case` | `user_count` |
| Function | `snake_case`, usually a verb | `calculate_total` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Class | `PascalCase` | `InvoiceService` |
| Module (file) | short `snake_case` | `invoice_utils.py` |
| Package (folder) | short lowercase, underscores avoided if possible | `billing` |
| Internal use only | leading underscore | `_cache` |
| Exception class | `PascalCase` ending in `Error` | `PaymentError` |

Names to avoid:

- A single lowercase `l`, uppercase `O`, or uppercase `I` as variable names, because they look like digits.
- Names that shadow built-ins, such as `list`, `dict`, `str`, `id`, `type`, `input`, `sum`.
- Cryptic abbreviations such as `usr_cnt_tmp2`.

Good names describe **what a value is** or **what a function does**:

| Weak | Better |
| --- | --- |
| `d` | `days_until_due` |
| `data` | `customer_records` |
| `process()` | `send_welcome_email()` |
| `flag` | `is_active` |

Boolean variables and functions usually read like a question: `is_valid`, `has_permission`, `can_retry`.

### Comments

- Explain **why**, not what. The code already shows what it does.
- Keep comments up to date. A wrong comment is worse than none.
- Use full sentences for block comments, and two spaces before an inline `#`.

### Docstrings

A **docstring** is a string placed as the first statement inside a module, function, or class. It documents what the thing does. Tools and editors display it when you hover or call `help()`. Use triple double quotes.

Two popular formats exist. Pick one per project.

### PEP 20: The Zen of Python

Run this in a terminal and read what appears:

```bash
python -c "import this"
```

The core ideas, in plain words:

| Principle | What it means in practice |
| --- | --- |
| Beautiful is better than ugly | Layout and naming matter |
| Explicit is better than implicit | Write code that shows what it does, avoid hidden magic |
| Simple is better than complex | Choose the simplest design that works |
| Readability counts | Optimize for the next reader |
| Errors should never pass silently | Do not hide problems with empty exception handlers |
| There should be one obvious way to do it | Prefer the standard idiom over a clever trick |
| If the implementation is hard to explain, it is a bad idea | Rewrite things you cannot explain clearly |

## Basic syntax examples

### Bad style versus good style

Bad:

```python
MAXVAL=100
def Calc(a,b):
    x=a+b
    if x>10 :
        return True
    else :
        return False
```

Good:

```python
MAX_VALUE = 100


def exceeds_ten(first: int, second: int) -> bool:
    total = first + second
    return total > 10
```

What changed: the constant is in `UPPER_SNAKE_CASE` with spaces around `=`, two blank lines separate top-level definitions, the function has a descriptive snake_case name, operators have spaces around them, the stray space before the colon is gone, and the redundant `if/else` is replaced by returning the comparison directly.

### Line length and wrapping

Break long expressions inside brackets, not with backslashes:

```python
total_price = (
    base_price
    + shipping_cost
    + handling_fee
    - discount
)

result = send_invoice(
    customer_id=customer_id,
    amount=amount,
    currency="EUR",
)
```

A trailing comma after the last item makes future changes cleaner in version control.

### Import grouping

```python
import json
import logging
from pathlib import Path

import requests

from billing.models import Invoice
```

Standard library first, third-party second, your own code third, separated by blank lines.

### Comments: weak and strong

```python
# Weak: repeats the code
count = count + 1  # add one to count

# Strong: explains the reason
# The API counts pages starting at 1, but our loop starts at 0.
page_number = index + 1
```

### Google-style docstring

```python
def apply_discount(price: float, percent: float) -> float:
    """Return the price after subtracting a percentage discount.

    Args:
        price: The original price. Must not be negative.
        percent: The discount as a number from 0 to 100.

    Returns:
        The discounted price.

    Raises:
        ValueError: If price is negative or percent is outside 0 to 100.
    """
    if price < 0 or not 0 <= percent <= 100:
        raise ValueError("price must be >= 0 and percent between 0 and 100")
    return price * (1 - percent / 100)
```

### NumPy-style docstring

```python
def apply_discount(price: float, percent: float) -> float:
    """Return the price after subtracting a percentage discount.

    Parameters
    ----------
    price : float
        The original price. Must not be negative.
    percent : float
        The discount as a number from 0 to 100.

    Returns
    -------
    float
        The discounted price.

    Raises
    ------
    ValueError
        If price is negative or percent is outside 0 to 100.
    """
    if price < 0 or not 0 <= percent <= 100:
        raise ValueError("price must be >= 0 and percent between 0 and 100")
    return price * (1 - percent / 100)
```

| Aspect | Google style | NumPy style |
| --- | --- | --- |
| Look | Compact, indented sections | Underlined section headings |
| Typical community | General Python and backend work | Scientific and data libraries |
| Takes more vertical space | No | Yes |

This roadmap prefers Google style for application code.

### A short docstring is fine for simple things

```python
def is_adult(age: int) -> bool:
    """Return True if age is 18 or older."""
    return age >= 18
```

## Deeper behavior

### Why indentation is syntax in Python

Other languages use braces to mark blocks. Python uses indentation. Mixing tabs and spaces can produce errors or, worse, code that looks right but behaves differently. Configure your editor to insert 4 spaces when you press Tab.

### Line length is a trade-off

Short lines fit side-by-side diffs and split-screen editing, and they discourage deeply nested logic. Overly strict limits force awkward breaks. Pick a number, put it in your tool configuration, and stop debating. The industry-level example below assumes 88.

### Consistency beats personal preference

PEP 8 itself says that consistency within a project is more important than strict conformance to the guide. When joining an existing codebase, follow its conventions even if you would choose otherwise.

### Magic numbers and constants

A number or string with unexplained meaning in the middle of code is a **magic value**. Give it a name:

```python
# Hard to understand
if retries > 3:
    ...

# Clear
MAX_RETRIES = 3
if retries > MAX_RETRIES:
    ...
```

### Naming and scope

A short name is acceptable for a tiny scope, such as `i` in a three-line loop. The larger the scope, the more descriptive the name should be.

### Private by convention

A leading underscore, such as `_cache`, signals "internal, do not rely on this from outside." Python does not enforce it, it is a courtesy to other developers. Chapter 6.9 covers this properly.

### Comments rot, code names do not lie as easily

If you feel the need to write a long comment explaining a confusing block, first try to rename things or split the block into a function with a good name. Keep comments for context that code cannot express, such as business rules, workarounds, and links to design decisions.

### Type hints are part of modern style

Annotations such as `price: float` and `-> bool` document intent and enable tools like mypy. Phase 8 teaches them fully, and this note already uses them in examples.

### Tools enforce style so humans do not have to

Formatters and linters fix spacing and flag naming problems automatically. Code reviews should focus on design and correctness, not on arguing about whitespace.

## Industry-level snippet

This module follows PEP 8, uses Google-style docstrings, type hints, constants, logging, input validation, and specific error handling. It calculates an invoice total for a billing service. It uses `Decimal`, which handles money exactly (Chapter 1.3 explains why).

```python
"""Invoice calculations for the billing service."""

import logging
from dataclasses import dataclass
from decimal import ROUND_HALF_UP, Decimal

logger = logging.getLogger(__name__)

ZERO = Decimal("0")
TWO_PLACES = Decimal("0.01")
MAX_DISCOUNT_PERCENT = Decimal("50")


class InvoiceError(Exception):
    """Raised when an invoice cannot be calculated from the given data."""


@dataclass(frozen=True)
class LineItem:
    """One purchased product on an invoice.

    Attributes:
        description: Human-readable name of the product.
        unit_price: Price of one unit, in the invoice currency.
        quantity: Number of units purchased. Must be at least 1.
    """

    description: str
    unit_price: Decimal
    quantity: int


def _round_money(amount: Decimal) -> Decimal:
    """Round an amount to two decimal places using commercial rounding."""
    return amount.quantize(TWO_PLACES, rounding=ROUND_HALF_UP)


def calculate_total(
    items: list[LineItem],
    discount_percent: Decimal = ZERO,
    tax_rate: Decimal = ZERO,
) -> Decimal:
    """Return the invoice total after discount and tax.

    Args:
        items: The line items to charge for. Must not be empty.
        discount_percent: Discount from 0 up to MAX_DISCOUNT_PERCENT.
        tax_rate: Tax as a fraction, for example Decimal("0.2") for 20 percent.

    Returns:
        The total amount due, rounded to two decimal places.

    Raises:
        InvoiceError: If items are empty, or any quantity, price, discount,
            or tax rate is invalid.
    """
    if not items:
        raise InvoiceError("an invoice needs at least one line item")
    if not ZERO <= discount_percent <= MAX_DISCOUNT_PERCENT:
        raise InvoiceError(f"discount must be between 0 and {MAX_DISCOUNT_PERCENT}")
    if tax_rate < ZERO:
        raise InvoiceError("tax rate must not be negative")

    subtotal = ZERO
    for item in items:
        if item.quantity < 1:
            raise InvoiceError(f"invalid quantity for {item.description!r}")
        if item.unit_price < 0:
            raise InvoiceError(f"negative price for {item.description!r}")
        subtotal += item.unit_price * item.quantity

    discounted = subtotal * (1 - discount_percent / 100)
    total = _round_money(discounted * (1 + tax_rate))
    logger.info("Calculated invoice total %s for %d line items", total, len(items))
    return total
```

Try it:

```python
from decimal import Decimal

items = [
    LineItem("Notebook", Decimal("4.99"), 3),
    LineItem("Pen", Decimal("1.20"), 10),
]
print(calculate_total(items, discount_percent=Decimal("10"), tax_rate=Decimal("0.2")))
```

Style points to notice:

- Module docstring first, then imports in alphabetical order, then constants, then classes and functions.
- Constants in capitals, classes in PascalCase, functions in snake_case, a private helper with a leading underscore.
- Every public function has a docstring explaining arguments, return value, and errors.
- Long signatures are wrapped inside brackets with one parameter per line and a trailing comma.
- Error messages are specific and include the value that caused the problem.
- Nothing is hidden in the middle of the code: limits live in named constants.

## Common mistakes

1. **Mixing tabs and spaces.** Configure your editor to use 4 spaces.
2. **Naming variables after built-ins.** `list = [1, 2]` makes the real `list` unavailable in that scope.
3. **Using one-letter names everywhere.** Only tiny scopes justify them.
4. **Writing comments that restate the code.** Use them to explain why.
5. **Leaving commented-out code behind.** Version control already remembers old code. Delete it.
6. **Star imports.** `from module import *` hides where names come from and can overwrite existing ones.
7. **Inconsistent docstring style within a project.** Choose Google or NumPy and stay with it.
8. **Hard-coding magic numbers.** Extract named constants.
9. **Ignoring the existing style of a codebase.** Match it unless the team agrees to change it.
10. **Believing style tools replace thinking.** They fix formatting, not design.

## Practice exercises

1. Reformat the "bad style" example from this note by hand, then explain each change you made.
2. Rename each of these to follow conventions: a variable `UserAge`, a function `CalculateTotal`, a constant `maxretries`, and a class `invoice_service`.
3. Write three weak names from your own past code or ideas and replace them with names that describe meaning.
4. Take this snippet and rewrite it so it fits within 88 characters per line without backslashes: `final_price = base_price + shipping_cost + handling_fee + insurance_fee - promotional_discount - loyalty_discount`.
5. Group these imports correctly and sort them: `from billing.models import Invoice`, `import requests`, `import os`, `import json`, `from pathlib import Path`.
6. Write a function that checks whether a username is valid (3 to 20 characters, letters and digits only). Add a Google-style docstring.
7. Rewrite the same function's docstring in NumPy style. Name two differences you noticed.
8. Run `python -c "import this"`, then pick three lines and give a short code example of each idea.
9. Find a magic number in some code you have written and replace it with a named constant. Explain how that helps a reader.
10. Extend the invoice module with a function `format_total` that returns a string like `EUR 123.45`. Follow all conventions in this note, include type hints and a docstring, and reject unsupported currency codes with a clear error.

## Summary

- PEP 8 is the official style guide. Use 4-space indentation, clear spacing, grouped imports, and consistent line length.
- Names follow conventions: `snake_case` for variables, functions, and modules, `PascalCase` for classes, `UPPER_SNAKE_CASE` for constants.
- Good names describe meaning. Avoid single letters beyond tiny scopes and avoid shadowing built-ins.
- Comments explain why. Docstrings explain what a module, function, or class does, using Google or NumPy style consistently.
- PEP 20 reminds you that readability counts and that simple, explicit code is better than clever code.
- Consistency within a project matters more than personal taste.
- Tools can enforce formatting, but understanding the reasons makes you a better reviewer and teammate.
