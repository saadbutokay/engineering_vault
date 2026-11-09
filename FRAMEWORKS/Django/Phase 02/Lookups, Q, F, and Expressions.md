## Overview

Basic `filter(field=value)` calls only go so far. Real applications need comparisons, text matching, conditions that combine with OR and NOT, updates that compute a new value from the old one, and values computed inside the database. Django covers all of this with field lookups, `Q` objects, `F` expressions, and a family of expression classes (`Value`, `Func`, `Case`, `When`, and many built-in database functions). The skill is to push computation into the database, where it is fast and free of race conditions, rather than pulling rows into Python. This note covers each tool and ends with a typed, tested search and inventory module that combines them.

## Core concepts

### Field lookups

A lookup is a suffix after a double underscore: `price__lt=20`. If you omit it, the lookup is `exact`.

- Comparison: `exact`, `gt`, `gte`, `lt`, `lte`.
- Text: `contains`, `icontains` (case-insensitive), `startswith`, `istartswith`, `endswith`, `iexact`, `regex`.
- Sets and ranges: `in`, `range`.
- Nulls: `isnull=True` or `False`.
- Dates and times: transforms such as `year`, `month`, `date`, `week_day` that chain with lookups, for example `published_on__year__gte=2020`.

### Traversing relations

Double underscores also follow relationships: `author__name__icontains="le guin"` joins to the author table. Reverse relations work the same way using the `related_name`: `Author.objects.filter(books__price__gt=20)`.

### Q objects

A `Q` object wraps a condition so conditions can be combined with `&` (AND), `|` (OR), and `~` (NOT). Keyword arguments to `filter()` are always ANDed, so you need `Q` for OR and NOT.

### F expressions

`F("field")` refers to a column value inside the database. It lets you compare two columns, update a column from its own value, and order or annotate by computed values, without loading the row into Python.

### Value, Func, and database functions

- `Value(x)` wraps a Python constant as an expression.
- `Func` calls a database function, and Django ships many wrappers: `Lower`, `Upper`, `Length`, `Concat`, `Coalesce`, `Cast`, `Round`, `Substr`, `Now`, `TruncMonth`, and more.
- `Coalesce(a, b)` returns the first non-null argument.
- `Cast(expr, output_field)` converts a type.

### Case and When

Conditional expressions are the SQL `CASE WHEN ... THEN ... ELSE ... END`. They compute a value per row, label rows, or perform conditional updates.

### annotate and alias

`annotate(name=expression)` adds a computed attribute to every object in the queryset. `alias(name=expression)` computes the expression for use in later filters or ordering but does not select it.

## Basic syntax examples

### Lookups

```python
from decimal import Decimal

from apps.bookstore.models import Author, Book

Book.objects.filter(price__gte=Decimal("10"), price__lt=Decimal("20"))
Book.objects.filter(title__icontains="earth")
Book.objects.filter(title__startswith="The")
Book.objects.filter(isbn__in=["9780306406157", "9780131103627"])
Book.objects.filter(published_on__isnull=True)
Book.objects.filter(published_on__year=1968)
Book.objects.filter(price__range=(Decimal("5"), Decimal("15")))
```

### Following relations

```python
Book.objects.filter(author__name__icontains="le guin")
Author.objects.filter(books__stock__gt=0).distinct()
```

### Q for OR and NOT

```python
from django.db.models import Q

Book.objects.filter(Q(title__icontains="dune") | Q(author__name__icontains="herbert"))
Book.objects.filter(~Q(format=Book.Format.EBOOK))
Book.objects.filter(
    Q(stock=0) | Q(price__gt=Decimal("50")),
    format=Book.Format.HARDCOVER,   # ANDed with the Q object
)
```

Positional `Q` arguments must come before keyword arguments.

### Building a condition dynamically

```python
condition = Q()
if text:
    condition &= Q(title__icontains=text)
if max_price is not None:
    condition &= Q(price__lte=max_price)
books = Book.objects.filter(condition)
```

An empty `Q()` matches everything.

### F expressions

```python
from django.db.models import F

# Compare two columns of the same row (a contrived but runnable example)
Book.objects.filter(stock__gt=F("price"))

# Race-free update, one UPDATE statement, no read in Python
Book.objects.filter(pk=1).update(stock=F("stock") + 5)

# Order by a computed value
Book.objects.order_by((F("price") * F("stock")).desc())
```

### Value, Coalesce, Concat, Lower

```python
from django.db.models import CharField, Value
from django.db.models.functions import Coalesce, Concat, Lower

Book.objects.annotate(
    label=Concat("title", Value(" by "), "author__name", output_field=CharField()),
    normalized=Lower("title"),
    subtitle_or_title=Coalesce("title", Value("untitled")),
)
```

### Case and When

```python
from django.db.models import Case, CharField, Value, When

Book.objects.annotate(
    availability=Case(
        When(stock=0, then=Value("sold_out")),
        When(stock__lte=5, then=Value("low")),
        default=Value("available"),
        output_field=CharField(),
    )
)
```

### Cast and Round

```python
from django.db.models import FloatField
from django.db.models.functions import Cast, Round

Book.objects.annotate(price_float=Cast("price", FloatField()))
Book.objects.update(price=Round(F("price") * Decimal("0.90"), precision=2))
```

### Combining querysets

```python
first = Book.objects.filter(stock=0)
second = Book.objects.filter(price__gt=Decimal("50"))

either = first | second        # OR of the two conditions (same model, no slicing)
both = first & second          # AND of the two conditions
combined = first.union(second) # SQL UNION, limited operations afterwards
common = first.intersection(second)
```

## Deeper behavior

### How lookups resolve

Django splits a name on `__`. The first parts that are fields or relations are followed, the next part may be a transform (such as `year`), and the last part is the lookup. `author__name__icontains` means: follow `author`, take `name`, apply `icontains`. If a name matches nothing, Django raises `FieldError` listing the valid choices, which is helpful when you mistype.

### Case sensitivity depends on the database

`exact` and `contains` are case-sensitive on PostgreSQL and often case-insensitive on SQLite for ASCII. Use the `i` variants (`iexact`, `icontains`) when you mean case-insensitive, and the behavior stays the same everywhere. Django escapes `%` and `_` in `contains`-style lookups for you.

### Edge cases to remember

- `filter(field__in=[])` matches nothing, and `exclude(field__in=[])` matches everything.
- Comparing to `None` with `exact` becomes `IS NULL`, but use `isnull` to be explicit.
- Filtering through a nullable relation uses a `LEFT JOIN` in some cases and an `INNER JOIN` in others, depending on the condition.

### Multi-valued relations: one filter or two

For relations that can have many rows, these two queries mean different things:

```python
# One review that is both rated 5 and mentions "classic"
Book.objects.filter(reviews__rating=5, reviews__body__icontains="classic")

# One review rated 5, and possibly a different review mentioning "classic"
Book.objects.filter(reviews__rating=5).filter(reviews__body__icontains="classic")
```

Conditions inside a single `filter()` call apply to the same related row, while chained `filter()` calls on a multi-valued relation may match different related rows. `exclude()` has its own subtle semantics for multi-valued relations, so test those queries with real data.

### Q object details

`Q` objects combine freely and nest: `Q(a) & (Q(b) | ~Q(c))`. Operator precedence follows Python (`~`, then `&`, then `|`), so use parentheses for clarity. `Q` objects can be built from a dictionary with `Q(**conditions)`, which is useful for dynamic filters.

### F expressions and race conditions

The naive way to increment a counter is a read-modify-write cycle:

```python
book = Book.objects.get(pk=1)
book.stock += 5      # computed in Python from a value that may already be stale
book.save()
```

If another request changed `stock` between the read and the write, its change is lost. The `F` version, `update(stock=F("stock") + 5)`, performs the arithmetic in one atomic database statement, so concurrent updates add up correctly. It is the first line of defense against lost updates, and note 2.9 builds the rest.

### F on instances

You may assign an `F` expression to an attribute and save: `book.stock = F("stock") + 1; book.save()`. Afterward the attribute still holds the expression, not the number, so call `book.refresh_from_db()` before reading it. Reusing that instance without refreshing applies the expression again on the next save.

### Output types and mixed expressions

When an expression mixes types, Django cannot always infer the result type and raises `FieldError: Expression contains mixed types`. Set `output_field` explicitly, as in the `Concat` example above.

### Expressions in filters and ordering

Expressions can appear in `filter` (with comparison lookups on annotations), `order_by` (`F("price").desc(nulls_last=True)`), `annotate`, and `update`. A filter can also compare an annotation: `annotate(total=F("price") * F("stock")).filter(total__gt=1000)`.

### alias versus annotate

Use `alias()` when you only need the computed value for a filter or ordering. It keeps the value out of the `SELECT` list and out of the objects, which makes the query lighter.

### Case in updates

`Case` works in `update()` too, which allows several conditional changes in one statement:

```python
Book.objects.update(
    price=Case(
        When(stock__gte=100, then=Round(F("price") * Decimal("0.80"), precision=2)),
        default=F("price"),
    )
)
```

## Industry-level snippet

A reusable search and inventory module: a validated filter object turns optional parameters into a single `Q` condition, annotations label availability, and stock changes happen race-free in the database. It uses small shared test factories.

`apps/bookstore/selectors.py`:

```python
"""Read and inventory operations for books built from lookups and expressions."""

import logging
from dataclasses import dataclass
from decimal import Decimal

from django.db.models import Case, CharField, F, Q, QuerySet, Value, When
from django.db.models.functions import Concat, Round

from apps.bookstore.models import Book

logger = logging.getLogger(__name__)

LOW_STOCK_THRESHOLD = 5
CLEARANCE_STOCK = 50
CLEARANCE_FACTOR = Decimal("0.90")


@dataclass(frozen=True)
class BookFilters:
    text: str = ""
    author_name: str = ""
    min_price: Decimal | None = None
    max_price: Decimal | None = None
    formats: tuple[str, ...] = ()
    in_stock: bool | None = None

    def __post_init__(self) -> None:
        if (
            self.min_price is not None
            and self.max_price is not None
            and self.min_price > self.max_price
        ):
            raise ValueError("min_price must not be greater than max_price")
        unknown = set(self.formats) - set(Book.Format.values)
        if unknown:
            raise ValueError(f"Unknown formats: {sorted(unknown)}")


def build_condition(filters: BookFilters) -> Q:
    """Translate ``filters`` into one ``Q`` object. Empty filters match all."""
    condition = Q()

    text = filters.text.strip()
    if text:
        condition &= Q(title__icontains=text) | Q(author__name__icontains=text)

    author_name = filters.author_name.strip()
    if author_name:
        condition &= Q(author__name__iexact=author_name)

    if filters.min_price is not None:
        condition &= Q(price__gte=filters.min_price)
    if filters.max_price is not None:
        condition &= Q(price__lte=filters.max_price)

    if filters.formats:
        condition &= Q(format__in=filters.formats)

    if filters.in_stock is True:
        condition &= Q(stock__gt=0)
    elif filters.in_stock is False:
        condition &= Q(stock=0)

    return condition


def search_books(filters: BookFilters) -> QuerySet[Book]:
    return (
        Book.objects.select_related("author")
        .filter(build_condition(filters))
        .annotate(
            availability=Case(
                When(stock=0, then=Value("sold_out")),
                When(stock__lte=LOW_STOCK_THRESHOLD, then=Value("low")),
                default=Value("available"),
                output_field=CharField(),
            ),
            display_title=Concat(
                "title", Value(" by "), "author__name", output_field=CharField()
            ),
        )
    )


def restock(*, book_id: int, quantity: int) -> None:
    """Add ``quantity`` units to a book's stock in one atomic statement."""
    if quantity <= 0:
        raise ValueError("quantity must be positive")
    updated = Book.objects.filter(pk=book_id).update(stock=F("stock") + quantity)
    if updated == 0:
        raise Book.DoesNotExist(f"Book {book_id} does not exist")
    logger.info("Restocked book %s with %d units", book_id, quantity)


def mark_down_overstock() -> int:
    """Reduce the price of heavily stocked books by 10 percent; return the count."""
    updated = Book.objects.filter(stock__gte=CLEARANCE_STOCK).update(
        price=Round(F("price") * CLEARANCE_FACTOR, precision=2)
    )
    logger.info("Marked down %d overstocked books", updated)
    return updated
```

`apps/bookstore/tests/factories.py` (reused by later notes):

```python
"""Small factories for tests."""

import itertools
from decimal import Decimal

from apps.bookstore.models import Author, Book

_counter = itertools.count(1)
DEFAULT_PRICE = Decimal("10.00")


def make_isbn(seed: int) -> str:
    """Build a valid ISBN-13 from a numeric seed."""
    body = f"978{seed:09d}"
    total = sum(int(digit) * (1 if index % 2 == 0 else 3) for index, digit in enumerate(body))
    return body + str((10 - total % 10) % 10)


def make_author(name: str = "Ursula K. Le Guin") -> Author:
    return Author.objects.create(name=name)


def make_book(
    *,
    author: Author | None = None,
    title: str | None = None,
    price: Decimal = DEFAULT_PRICE,
    stock: int = 0,
    format: str = Book.Format.PAPERBACK,
) -> Book:
    seed = next(_counter)
    return Book.objects.create(
        author=author or make_author(),
        title=title or f"Book {seed}",
        isbn=make_isbn(seed),
        price=price,
        stock=stock,
        format=format,
    )
```

`apps/bookstore/tests/test_selectors.py`:

```python
from decimal import Decimal

from django.test import SimpleTestCase, TestCase

from apps.bookstore.models import Book
from apps.bookstore.selectors import (
    BookFilters,
    mark_down_overstock,
    restock,
    search_books,
)

from .factories import make_author, make_book


class BookFiltersTests(SimpleTestCase):
    def test_inverted_price_range_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            BookFilters(min_price=Decimal("20"), max_price=Decimal("10"))

    def test_unknown_format_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            BookFilters(formats=("scroll",))


class SearchBooksTests(TestCase):
    def titles(self, filters: BookFilters) -> list[str]:
        return list(search_books(filters).values_list("title", flat=True))

    def test_text_matches_title_or_author(self) -> None:
        herbert = make_author("Frank Herbert")
        make_book(title="Dune", author=herbert)
        make_book(title="Earthsea")
        self.assertEqual(self.titles(BookFilters(text="dune")), ["Dune"])
        self.assertEqual(self.titles(BookFilters(text="herbert")), ["Dune"])

    def test_price_range_is_inclusive(self) -> None:
        make_book(title="Cheap", price=Decimal("5.00"))
        make_book(title="Mid", price=Decimal("10.00"))
        make_book(title="Dear", price=Decimal("20.00"))
        filters = BookFilters(min_price=Decimal("5"), max_price=Decimal("10"))
        self.assertEqual(self.titles(filters), ["Cheap", "Mid"])

    def test_format_and_stock_filters_combine(self) -> None:
        make_book(title="Print", stock=3, format=Book.Format.HARDCOVER)
        make_book(title="Digital", stock=3, format=Book.Format.EBOOK)
        make_book(title="Empty", stock=0, format=Book.Format.HARDCOVER)
        filters = BookFilters(formats=(Book.Format.HARDCOVER,), in_stock=True)
        self.assertEqual(self.titles(filters), ["Print"])
        self.assertEqual(self.titles(BookFilters(in_stock=False)), ["Empty"])

    def test_empty_filters_return_everything(self) -> None:
        make_book(title="One")
        make_book(title="Two")
        self.assertEqual(len(self.titles(BookFilters())), 2)

    def test_availability_labels(self) -> None:
        make_book(title="A", stock=0)
        make_book(title="B", stock=5)
        make_book(title="C", stock=6)
        labels = dict(search_books(BookFilters()).values_list("title", "availability"))
        self.assertEqual(labels, {"A": "sold_out", "B": "low", "C": "available"})

    def test_display_title_joins_author(self) -> None:
        make_book(title="Dune", author=make_author("Frank Herbert"))
        values = list(search_books(BookFilters()).values_list("display_title", flat=True))
        self.assertEqual(values, ["Dune by Frank Herbert"])


class InventoryTests(TestCase):
    def test_restock_adds_to_existing_stock(self) -> None:
        book = make_book(stock=3)
        restock(book_id=book.pk, quantity=4)
        book.refresh_from_db()
        self.assertEqual(book.stock, 7)

    def test_restock_validates_quantity_and_book(self) -> None:
        book = make_book()
        with self.assertRaises(ValueError):
            restock(book_id=book.pk, quantity=0)
        with self.assertRaises(Book.DoesNotExist):
            restock(book_id=999_999, quantity=1)

    def test_mark_down_only_changes_overstocked_books(self) -> None:
        heavy = make_book(stock=50, price=Decimal("20.00"))
        light = make_book(stock=49, price=Decimal("20.00"))
        self.assertEqual(mark_down_overstock(), 1)
        heavy.refresh_from_db()
        light.refresh_from_db()
        self.assertEqual(heavy.price, Decimal("18.00"))
        self.assertEqual(light.price, Decimal("20.00"))
```

## Common mistakes

- Reading a value into Python, changing it, and saving it, which creates lost updates. Use `F` expressions for counters and arithmetic.
- Forgetting `distinct()` when filtering through a multi-valued relation and getting duplicate rows.
- Treating `filter(a=1, b=2)` and `filter(a=1).filter(b=2)` as equivalent for multi-valued relations.
- Using `contains` when case-insensitive matching is needed, so results differ between SQLite and PostgreSQL.
- Passing keyword arguments before `Q` objects in `filter()`, which is a syntax error.
- Reading an attribute after saving an `F` expression without `refresh_from_db()`.
- Mixing field types in an expression without `output_field`.
- Doing arithmetic, filtering, or grouping in Python over large querysets instead of in the database.
- Building SQL strings by hand for conditions the lookups already cover.
- Forgetting that `update()` skips `save()`, signals, and `auto_now`.

## Practice exercises

1. Write queries for books whose title starts with "The", costs between 5 and 15, and is not an e-book. Print the SQL and identify each condition in it.
2. Write a query for authors with at least one book in stock and no book priced above 50, using `Q`, `~Q`, and `distinct()` where necessary. Check the result with test data.
3. Demonstrate a lost update: simulate two read-modify-write cycles interleaved in code, show the wrong total, then fix it with `F`.
4. Show the difference between `filter(reviews__rating=5, reviews__body__icontains="x")` and the chained version using test data in which two different reviews satisfy the two conditions.
5. Write an annotation `price_band` using `Case` and `When` that labels books `budget` under 10, `standard` under 25, and `premium` otherwise. Write tests for the boundaries 10 and 25.
6. Extend `BookFilters` with `published_after: date | None` using the `published_on__gte` lookup, and add tests including a `None` publication date.
7. Add `sold_out_titles()` returning `QuerySet.union()` of sold-out hardcovers and sold-out paperbacks. Note what you cannot do with the result afterward.

## Summary

Lookups express comparisons and text matching, double underscores follow relations, `Q` objects combine conditions with AND, OR, and NOT, and `F` expressions let the database compute with its own column values. `Value`, `Func`, `Case`, `When`, and built-in functions like `Coalesce`, `Concat`, `Cast`, and `Round` compute new values per row, usable in `annotate`, `filter`, `order_by`, and `update`. Prefer these over Python loops: they are faster, and `F`-based updates are race-free. Watch multi-valued relations and `distinct()`, use case-insensitive lookups deliberately, specify `output_field` for mixed types, and call `refresh_from_db()` after saving an `F` expression.
