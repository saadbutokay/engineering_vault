## Overview

A `QuerySet` represents a database query that Django has not necessarily run yet. You build it step by step with methods such as `filter` and `order_by`, and Django sends SQL only when you actually need the results. Understanding when that happens, and which methods run one query versus many, is the foundation of writing efficient Django code. This note covers creating, reading, updating, and deleting objects, lazy evaluation, the result cache, the common retrieval and bulk methods, and the exceptions you will meet. It ends with a realistic bulk import service that validates rows, reuses related objects, and reports results.

All examples use the bookstore models from note 2.2.

## Core concepts

### Managers and QuerySets

Every model has a default manager, `objects`, which is the entry point for queries: `Book.objects.all()`. A manager hands you a `QuerySet`. A `QuerySet` is a lazy, chainable description of a query.

### Creating objects

- `Book.objects.create(...)` builds and saves in one step.
- `Book(...)` followed by `book.save()` lets you adjust the object before saving.
- `Book.objects.bulk_create([...])` inserts many rows in few queries.

### Retrieving objects

- `all()`: every row.
- `get(**lookups)`: exactly one row, raising an exception if there is none or several.
- `filter(**lookups)`: rows that match.
- `exclude(**lookups)`: rows that do not match.
- `order_by("field", "-other")`: sort, with a minus for descending.
- Slicing (`qs[:10]`, `qs[20:30]`): translated to `LIMIT` and `OFFSET`.
- `first()` and `last()`: one object or `None`.
- `exists()`: does at least one row match?
- `count()`: how many rows match?

### Updating objects

- Change attributes and call `save()` (optionally `save(update_fields=[...])`).
- `queryset.update(field=value)`: one `UPDATE` statement for all matching rows.
- `bulk_update(objs, fields)`: update many instances with their own values.

### Deleting objects

`instance.delete()` and `queryset.delete()` remove rows and, according to each relation's `on_delete`, may remove or block related rows.

### Lazy evaluation

Building a queryset runs no SQL. The query executes when you iterate, convert to a list, slice with a step, call `len()`, `bool()`, or `repr()` on it, or call methods that need a result (`get`, `count`, `exists`, `first`).

### The result cache

Once a queryset is evaluated, it stores its results. Iterating the same queryset object again reuses them without another query. Calling `filter` or `order_by` on it creates a new queryset that has no cache.

### get_or_create and update_or_create

- `get_or_create(defaults={...}, **lookups)` returns `(object, created)`.
- `update_or_create(defaults={...}, **lookups)` updates the match or creates it, returning `(object, created)`.

### Exceptions

- `Model.DoesNotExist`: `get()` found no row.
- `Model.MultipleObjectsReturned`: `get()` found more than one.
- Both inherit from `ObjectDoesNotExist` and `MultipleObjectsReturned` in `django.core.exceptions` for generic handling.

## Basic syntax examples

Run these in `shell_plus` or `manage.py shell`.

### Creating

```python
from decimal import Decimal

from apps.bookstore.models import Author, Book

author = Author.objects.create(name="Ursula K. Le Guin")

book = Book(
    title="A Wizard of Earthsea",
    isbn="9780306406157",
    author=author,
    price=Decimal("12.99"),
)
book.save()
```

### Retrieving

```python
Book.objects.all()
Book.objects.get(isbn="9780306406157")
Book.objects.filter(price__lt=Decimal("20"), stock__gt=0)
Book.objects.exclude(format=Book.Format.EBOOK)
Book.objects.order_by("-price", "title")[:5]
Book.objects.filter(stock=0).exists()
Book.objects.filter(stock=0).count()
Book.objects.order_by("price").first()
```

### Chaining

```python
cheap_in_stock = (
    Book.objects
    .filter(stock__gt=0)
    .filter(price__lt=Decimal("15"))
    .order_by("title")
)
```

No SQL has run yet. The query runs when you iterate:

```python
for book in cheap_in_stock:
    print(book.title)
```

### Looking at the SQL

```python
print(cheap_in_stock.query)
```

### Updating

```python
book = Book.objects.get(isbn="9780306406157")
book.price = Decimal("11.99")
book.save(update_fields=["price"])

Book.objects.filter(stock=0).update(price=Decimal("9.99"))
```

### Deleting

```python
Book.objects.filter(stock=0, published_on__isnull=True).delete()
```

### Bulk operations

```python
Book.objects.bulk_create(
    [
        Book(title="Book A", isbn="9780131103627", author=author, price=Decimal("10")),
        Book(title="Book B", isbn="9780201633610", author=author, price=Decimal("11")),
    ],
    batch_size=500,
)

books = list(Book.objects.filter(author=author))
for book in books:
    book.stock += 5
Book.objects.bulk_update(books, ["stock"], batch_size=500)
```

### get_or_create and update_or_create

```python
author, created = Author.objects.get_or_create(
    name="Octavia Butler",
    defaults={"born_on": None},
)

book, created = Book.objects.update_or_create(
    isbn="9780131103627",
    defaults={"title": "The C Programming Language", "price": Decimal("49.00"), "author": author},
)
```

### Handling the exceptions

```python
from django.core.exceptions import ObjectDoesNotExist

try:
    book = Book.objects.get(isbn="0000000000000")
except Book.DoesNotExist:
    book = None
except Book.MultipleObjectsReturned:
    raise  # a data integrity problem worth surfacing
```

A shorter pattern when absence is normal:

```python
book = Book.objects.filter(isbn="0000000000000").first()
```

## Deeper behavior

### What triggers a query

```python
qs = Book.objects.filter(stock__gt=0)  # no query
qs = qs.order_by("title")              # no query
first_ten = qs[:10]                    # no query, adds LIMIT 10
titles = [b.title for b in first_ten]  # query runs here
more = [b.title for b in first_ten]    # no query, uses the result cache
```

Slicing with `[:10]` returns a new queryset and still does not run SQL. Slicing with a step (`[::2]`) or indexing (`qs[3]`) evaluates immediately, and indexing on an unevaluated queryset runs its own `LIMIT 1 OFFSET 3` query each time.

### Choosing the right check

- `if qs:` evaluates the whole queryset and caches every row. Use it only when you will use the rows next.
- `qs.exists()` runs a cheap `SELECT 1 ... LIMIT 1`.
- `len(qs)` loads all rows. `qs.count()` runs `SELECT COUNT(*)`.
- If you have already evaluated the queryset, `len(qs)` and iteration are free, and `count()` reuses the cache too.

### Why first() and last() need ordering

`first()` orders by primary key if the queryset has no ordering, and `last()` reverses the ordering. Without a deterministic order, "first" has no meaning, so give querysets an `order_by` whenever results will be paginated or compared.

### Querysets are cloned

Every chaining method returns a new queryset and leaves the original untouched. That is why `qs.filter(...)` without assigning the result does nothing.

### save() performs UPDATE then INSERT

If the instance has a primary key, `save()` runs an `UPDATE` and falls back to an `INSERT` if no row was updated. `save(update_fields=[...])` writes only those columns, which is faster and avoids overwriting concurrent changes to other columns. `save()` also sets `auto_now` fields and sends the `pre_save` and `post_save` signals.

### update() and delete() are different

`queryset.update(...)` issues one SQL statement, skips `save()`, does not send signals, and does not update `auto_now` fields. It is the right tool for bulk changes and for race-free arithmetic with `F` expressions (note 2.5). `queryset.delete()` collects related objects according to `on_delete` and may issue several statements. It is quick for simple models and heavy for models with many related rows.

### bulk_create limits

- It does not call `save()` and does not send signals.
- Multi-table inheritance models are not supported.
- On PostgreSQL, primary keys are set on the returned objects. On databases without `RETURNING` support they may not be.
- `ignore_conflicts=True` skips rows that violate constraints silently, so primary keys are not set on those objects. `update_conflicts=True` with `unique_fields` and `update_fields` performs an upsert.
- Use `batch_size` to stay within database parameter limits.

### get_or_create and races

`get_or_create` first tries `get()` and then `create()`. Two concurrent requests can both miss and both try to create. If the lookup fields are covered by a unique constraint, the loser raises `IntegrityError`, and Django retries the `get()` for you. Without a unique constraint you can end up with duplicates and later a `MultipleObjectsReturned` error. Always back `get_or_create` with a unique constraint on the lookup fields.

### Refreshing

`instance.refresh_from_db()` reloads the instance from the database, picking up changes made by `update()`, by other processes, or by database defaults and generated fields.

### Debugging queries

`str(queryset.query)` shows the SQL. `connection.queries` records executed queries when `DEBUG` is true, and `django.test.utils.CaptureQueriesContext` captures them in tests (note 2.7).

## Industry-level snippet

A bulk book import is a typical backend job: it takes many input rows, validates each, reuses existing data, skips what is already present, reports problems per row without failing the whole batch, and writes efficiently. The implementation uses `values_list` for one existence query, builds unsaved instances, validates their fields without hitting the database per row, reuses authors by name, and inserts everything with one `bulk_create` inside a transaction.

`apps/bookstore/services/importing.py`:

```python
"""Bulk import of books."""

import logging
from collections.abc import Sequence
from dataclasses import dataclass, field
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.db import transaction

from apps.bookstore.models import Author, Book

logger = logging.getLogger(__name__)

# Fields filled in by the database or set after validation.
_SKIPPED_FIELDS = {"author", "created_at", "title_lower"}


@dataclass(frozen=True)
class BookRow:
    isbn: str
    title: str
    author_name: str
    price: Decimal
    stock: int = 0


@dataclass(frozen=True)
class RowError:
    index: int
    message: str


@dataclass
class ImportResult:
    created: int = 0
    already_present: int = 0
    errors: list[RowError] = field(default_factory=list)


def import_books(rows: Sequence[BookRow], *, batch_size: int = 500) -> ImportResult:
    """Import ``rows`` and report what happened.

    Rows with invalid data are reported in ``errors`` and skipped. Rows whose
    ISBN already exists, or appears earlier in the input, are counted in
    ``already_present``. All valid rows are written in one transaction.
    """
    if batch_size <= 0:
        raise ValueError("batch_size must be positive")

    result = ImportResult()
    if not rows:
        return result

    existing = set(
        Book.objects.filter(isbn__in=[row.isbn.strip() for row in rows]).values_list(
            "isbn", flat=True
        )
    )
    seen: set[str] = set()
    pending: list[tuple[str, Book]] = []

    for index, row in enumerate(rows):
        isbn = row.isbn.strip()
        if isbn in existing or isbn in seen:
            result.already_present += 1
            continue

        author_name = row.author_name.strip()
        if not author_name:
            result.errors.append(RowError(index, "author_name is required"))
            continue

        candidate = Book(
            title=row.title.strip(),
            isbn=isbn,
            price=row.price,
            stock=row.stock,
        )
        try:
            candidate.clean_fields(exclude=_SKIPPED_FIELDS)
        except ValidationError as exc:
            result.errors.append(RowError(index, "; ".join(exc.messages)))
            continue

        seen.add(isbn)
        pending.append((author_name, candidate))

    if not pending:
        return result

    with transaction.atomic():
        authors: dict[str, Author] = {}
        for name in {author_name for author_name, _ in pending}:
            authors[name] = Author.objects.filter(name=name).first() or (
                Author.objects.create(name=name)
            )
        books = []
        for author_name, book in pending:
            book.author = authors[author_name]
            books.append(book)
        Book.objects.bulk_create(books, batch_size=batch_size)

    result.created = len(pending)
    logger.info(
        "Imported %d books (%d already present, %d errors)",
        result.created,
        result.already_present,
        len(result.errors),
    )
    return result
```

`apps/bookstore/tests/test_importing.py`:

```python
from decimal import Decimal

from django.test import TestCase

from apps.bookstore.models import Author, Book
from apps.bookstore.services.importing import BookRow, import_books

ISBN_A = "9780306406157"
ISBN_B = "9780131103627"
ISBN_C = "9780201633610"


DEFAULT_PRICE = Decimal("10.00")


def row(
    isbn: str,
    *,
    author_name: str = "Ursula",
    price: Decimal = DEFAULT_PRICE,
    stock: int = 0,
) -> BookRow:
    return BookRow(
        isbn=isbn,
        title=f"Book {isbn}",
        author_name=author_name,
        price=price,
        stock=stock,
    )


class ImportBooksTests(TestCase):
    def test_creates_books_and_reuses_authors(self) -> None:
        result = import_books([row(ISBN_A), row(ISBN_B)])
        self.assertEqual(result.created, 2)
        self.assertEqual(Book.objects.count(), 2)
        self.assertEqual(Author.objects.count(), 1)

    def test_existing_isbn_is_not_duplicated(self) -> None:
        import_books([row(ISBN_A)])
        result = import_books([row(ISBN_A), row(ISBN_B)])
        self.assertEqual((result.created, result.already_present), (1, 1))
        self.assertEqual(Book.objects.count(), 2)

    def test_duplicate_inside_input_is_created_once(self) -> None:
        result = import_books([row(ISBN_A), row(ISBN_A)])
        self.assertEqual((result.created, result.already_present), (1, 1))

    def test_invalid_rows_are_reported_and_valid_rows_still_import(self) -> None:
        rows = [
            row("9780306406158"),                 # bad check digit
            row(ISBN_A, price=Decimal("-1")),     # negative price
            row(ISBN_B, author_name="  "),        # blank author
            row(ISBN_C),                          # fine
        ]
        result = import_books(rows)
        self.assertEqual(result.created, 1)
        self.assertEqual([error.index for error in result.errors], [0, 1, 2])
        self.assertTrue(Book.objects.filter(isbn=ISBN_C).exists())

    def test_empty_input_runs_no_queries(self) -> None:
        with self.assertNumQueries(0):
            result = import_books([])
        self.assertEqual(result.created, 0)

    def test_batch_size_must_be_positive(self) -> None:
        with self.assertRaises(ValueError):
            import_books([row(ISBN_A)], batch_size=0)
```

## Common mistakes

- Calling `qs.filter(...)` and discarding the result, expecting the original queryset to change.
- Using `if queryset:` or `len(queryset)` to check for rows or count them, loading every row.
- Calling `get()` for something that may not exist without handling `DoesNotExist`.
- Looping over objects and calling `save()` on each when a single `update()` or `bulk_update()` would do.
- Expecting `update()` to refresh `auto_now` fields or send signals.
- Expecting `bulk_create` to call `save()` or fire signals.
- Using `get_or_create` without a unique constraint on the lookup fields.
- Paginating or taking `first()` on an unordered queryset and getting inconsistent results.
- Using `delete()` on a huge queryset inside a web request without thinking about cascade cost.
- Loading entire tables into memory with `list(Model.objects.all())` for large data.
- Assuming a modified instance is saved without calling `save()`.

## Practice exercises

1. In the shell, create three authors and six books. Retrieve the five most expensive in-stock books, then print the SQL for the queryset and match each part of it to your code.
2. Write a function `find_book(isbn: str) -> Book | None` using `.first()` and another using `get()` with exception handling. List a situation where each is the better choice.
3. Demonstrate the result cache: evaluate a queryset, then iterate it again and use `CaptureQueriesContext` to prove no second query ran. Then call `.filter()` on it and prove a new query runs.
4. Replace a loop that calls `book.save()` for each book with a single `update()` that raises the price of every in-stock paperback by one dollar, using `F`. (You will learn `F` in note 2.5, so first try a loop, then rewrite it.)
5. Write `restock_all(quantity: int) -> int` using `update()` that returns the number of rows changed and rejects non-positive quantities.
6. Show that `update()` does not change an `auto_now` field and `save()` does, by adding an `updated_at` field to a scratch model.
7. Write a test proving that a row whose title is longer than 200 characters is reported in `errors` with the right row index, and explain which part of `import_books` produces that message.

## Summary

A `QuerySet` is a lazy, chainable query. Nothing hits the database until you iterate, slice with a step, call `len` or `bool`, or call a method such as `get`, `count`, `exists`, or `first`. Evaluated querysets cache their results, and every chaining method returns a fresh clone. Use `exists()` and `count()` instead of loading rows, `update()` and `bulk_*` methods instead of looping with `save()`, and `get_or_create` only with a unique constraint behind it. Know what bypasses `save()` and signals, give querysets an explicit order when order matters, and wrap multi-step writes in a transaction.
