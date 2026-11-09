## Overview

Validation in Python can be bypassed: `save()` does not run it, `bulk_create()` does not run it, a management script or a raw SQL statement does not run it, and two concurrent requests can both pass a check that only one should. The database is the only layer that sees every write, so rules that must always hold belong there as constraints. Indexes are the other half of schema design: they make the queries you actually run fast, at a cost on writes. This note covers `CheckConstraint`, `UniqueConstraint` with conditions and expressions, indexes (composite, partial, covering), deciding what to index, soft delete and its costs, when `JSONField` is the wrong choice, database defaults, PostgreSQL-specific fields, and designing a schema that can evolve. It ends with the bookstore models hardened with constraints and indexes, and tests that prove each one holds.

## Core concepts

### Why constraints belong in the database

A constraint is a rule the database enforces on every write, no matter which code path performs it. It protects you from bugs in application code, from future code written by someone who does not know the rules, and from race conditions. The layers of defense are covered in note 3.5; this note builds the bottom one.

### CheckConstraint

A `CheckConstraint` requires every row to satisfy a condition written as a `Q` object: a price is not negative, a rating is between 1 and 5, an end date is not before a start date. In Django 5.1 and later the argument is named `condition`. (In earlier versions it was `check`, and `django-upgrade` renames it, as note 0.7 showed.)

### UniqueConstraint

`UniqueConstraint` is the full-featured way to require uniqueness, more flexible than `unique=True` and `unique_together`:

- `fields=[...]`: uniqueness over one or several columns.
- `condition=Q(...)`: a partial constraint that applies only to rows that match, such as active rows.
- Expressions: uniqueness over a computed value, such as `Lower("title")`.
- `deferrable`: allow the check to wait until the end of the transaction (PostgreSQL).
- `include`: add non-key columns to the index (PostgreSQL).
- `nulls_distinct` (Django 5.0, PostgreSQL 15 and later): control whether `NULL` values count as equal.

### Naming and messages

Every constraint needs a unique `name`. Set `violation_error_message` (and, from Django 5.0, `violation_error_code`) so model validation produces a readable error.

### Indexes

`Meta.indexes` declares indexes with `models.Index`. An index may cover several columns, may be partial (`condition=`), may be covering (`include=`, PostgreSQL), and may use descending order (`"-published_on"`). Index names are limited to 30 characters.

### Composite indexes and column order

A composite index on `(author, published_on)` can serve queries filtering on `author` alone, or on `author` and `published_on` together, but not on `published_on` alone. Put the columns used with equality first and the range or sort column last.

### Partial and covering indexes

- A partial index covers only rows matching a condition, making it small and fast for queries that always include that condition, such as low-stock books.
- A covering index stores extra columns so a query can be answered from the index alone, avoiding a trip to the table.

### Deciding what to index

Index columns that appear in `WHERE`, `JOIN`, and `ORDER BY` of queries that matter, and that are selective enough that the index narrows the result. Foreign keys are indexed automatically. Skip indexes on tiny tables, on columns with few distinct values that are queried alone, and on columns you never filter or sort by. Always confirm with `EXPLAIN` (note 2.7).

### Soft delete and its costs

Soft delete marks rows as deleted (`deleted_at`) instead of removing them. It supports undo and audit trails, but it complicates everything: every query must filter deleted rows, unique constraints must become partial, foreign keys still point at hidden rows, and the data never leaves. Choose it only when the business needs it.

### JSONField and when it is the wrong choice

`JSONField` stores flexible, schemaless data and supports queries into it. It is a good fit for sparse, optional attributes and for storing payloads exactly as received. It is the wrong choice for data you filter, join, sort, constrain, or aggregate often, because columns give you types, constraints, foreign keys, and simple indexes.

### Database-level defaults

`db_default` (Django 5.0) makes the database fill in a value even for writes that bypass Python, so rules like "new rows start with stock zero" hold for raw SQL too.

### PostgreSQL-specific fields

`django.contrib.postgres` offers `ArrayField`, `HStoreField`, range fields (`DateRangeField`, `IntegerRangeField`), `ExclusionConstraint`, and full-text search tools. They are powerful and tie your schema to PostgreSQL, which is acceptable once you have decided on it as your production database.

### Schema design for evolution

Schemas outlive code. Prefer additive changes (new nullable columns, new tables), avoid renaming and repurposing columns, use surrogate keys, keep natural keys as unique constraints, and plan multi-step migrations (note 2.3) so old and new code can run together during a deploy.

## Basic syntax examples

### CheckConstraint

```python
from django.db import models
from django.db.models import Q


class Review(models.Model):
    rating = models.PositiveSmallIntegerField()

    class Meta:
        constraints = [
            models.CheckConstraint(
                condition=Q(rating__gte=1, rating__lte=5),
                name="review_rating_between_1_and_5",
                violation_error_message="Rating must be between 1 and 5.",
            ),
        ]
```

### UniqueConstraint over several columns

```python
class Review(models.Model):
    book = models.ForeignKey("bookstore.Book", on_delete=models.CASCADE)
    reviewer_name = models.CharField(max_length=100)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["book", "reviewer_name"],
                name="review_one_per_reviewer",
                violation_error_message="You have already reviewed this book.",
            ),
        ]
```

### A conditional (partial) UniqueConstraint

```python
class Book(models.Model):
    isbn = models.CharField(max_length=13)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["isbn"],
                condition=Q(deleted_at__isnull=True),
                name="book_unique_active_isbn",
            ),
        ]
```

Only rows that are not soft-deleted must have a unique ISBN, so a deleted book does not block re-adding the same ISBN.

### A UniqueConstraint over an expression

```python
from django.db.models.functions import Lower


class Meta:
    constraints = [
        models.UniqueConstraint(
            Lower("title"),
            "author",
            name="book_unique_title_per_author",
            violation_error_message="This author already has a book with that title.",
        ),
    ]
```

### Indexes

```python
class Meta:
    indexes = [
        models.Index(fields=["author", "-published_on"], name="book_author_published_idx"),
        models.Index(
            fields=["stock"],
            condition=Q(stock__lte=5),
            name="book_low_stock_idx",
        ),
        models.Index(
            fields=["format"],
            include=["price"],
            name="book_format_price_idx",
        ),
    ]
```

The third index is a covering index and works on PostgreSQL only; other databases ignore `include`.

### JSONField

```python
class Book(models.Model):
    extra_metadata = models.JSONField(default=dict, blank=True)


Book.objects.filter(extra_metadata__series="Earthsea")
Book.objects.filter(extra_metadata__awards__0__year=1969)
```

### PostgreSQL-specific fields

```python
from django.contrib.postgres.fields import ArrayField, DateRangeField


class Edition(models.Model):
    keywords = ArrayField(models.CharField(max_length=40), default=list, blank=True)
    on_sale = DateRangeField()
```

Add `"django.contrib.postgres"` to `INSTALLED_APPS` to use these fields' lookups and forms.

### An exclusion constraint (PostgreSQL)

```python
from django.contrib.postgres.constraints import ExclusionConstraint
from django.contrib.postgres.fields import DateTimeRangeField, RangeOperators


class RoomBooking(models.Model):
    room = models.PositiveIntegerField()
    period = DateTimeRangeField()

    class Meta:
        constraints = [
            ExclusionConstraint(
                name="no_overlapping_bookings",
                expressions=[
                    ("period", RangeOperators.OVERLAPS),
                    ("room", RangeOperators.EQUAL),
                ],
            ),
        ]
```

This needs the `btree_gist` extension, installed with a `BtreeGistExtension()` operation in a migration.

### Inspecting the result

```bash
uv run python manage.py sqlmigrate bookstore 0006
```

```sql
CREATE UNIQUE INDEX "book_unique_active_isbn" ON "bookstore_book" ("isbn") WHERE "deleted_at" IS NULL;
ALTER TABLE "bookstore_book" ADD CONSTRAINT "book_price_non_negative" CHECK ("price" >= 0);
```

## Deeper behavior

### Constraints are enforced by the database, and checked by validation

When a write violates a constraint, the database raises an error and Django surfaces it as `IntegrityError`. Since Django 4.1, constraints also take part in model validation: `full_clean()` and `ModelForm.is_valid()` call `validate_constraints()` and turn violations into `ValidationError` using `violation_error_message`. That gives users a friendly message in forms and still leaves the database as the final guard.

### Adding constraints to existing data

If existing rows already violate the new rule, the migration fails. Find and fix them first, with a data migration or a cleanup script (note 2.3). On large tables, validating a new constraint scans the whole table and can block writes, so plan for it as note 12.5 describes.

### How NULL interacts with constraints

A `CHECK` constraint passes when its condition evaluates to unknown, which is what comparisons with `NULL` produce, so a nullable column needs an explicit `IS NOT NULL` if the value must exist. In unique constraints, rows with `NULL` in a column are treated as distinct on most databases, so two rows with `NULL` do not collide unless you use `nulls_distinct=False` on PostgreSQL 15 or later.

### Choosing check versus choices

`choices` only affect validation. A `CheckConstraint(condition=Q(format__in=[...]))` is what stops a bad value from `bulk_create()` or raw SQL. Maintaining the list in two places has a cost; for large or changing sets, use a lookup table and a foreign key instead.

### How the planner chooses indexes

The planner estimates how many rows a condition returns. Highly selective conditions favor an index; conditions that return a large share of the table favor a sequential scan. A partial index is used only when the query's `WHERE` clause implies the index's condition, so write the query with the same condition.

### Leftmost prefix rule

An index on `(a, b, c)` helps queries on `a`, on `a, b`, and on `a, b, c`. It does not help a query on `b` alone. It can also satisfy an `ORDER BY a, b` without a separate sort. Design composite indexes around your real queries, not every possible combination.

### The cost side of indexes

Each index must be updated on every insert and delete and on updates to its columns, and it takes disk space and memory. Unused indexes are pure cost. PostgreSQL's `pg_stat_user_indexes` view shows how often each index is used, so you can find and drop dead ones.

### Soft delete in detail

- Every query that should hide deleted rows needs the filter, so put it in a queryset method or manager (note 2.11) and test it.
- Unique constraints must be partial (`condition=Q(deleted_at__isnull=True)`), as in the example, or deleted rows block new ones.
- Foreign keys from other tables still reference deleted rows, so decide what "deleted" means for related data.
- Soft-deleted personal data may still have to be erased for legal reasons, so soft delete is not a substitute for a retention and deletion policy.
- A simpler alternative for audit needs is a separate history table (note 11.6).

### JSONField trade-offs

You can index JSON with a GIN index on PostgreSQL and query keys with lookups, but you cannot declare a foreign key to a value inside it, you cannot rely on column types, and a typo in a key silently creates a new key. Use it for payloads you store and display, not for the core attributes that your business logic depends on.

### db_default versus default

`default` is applied in Python, `db_default` in the database (note 2.2). A database default survives raw SQL, `bulk_create`, and other applications writing to the same database. Use it for invariants that must hold universally.

### Designing for change

- Add a column as nullable, backfill, then tighten it (note 2.3).
- Never reuse a column for a different meaning.
- Keep old and new code compatible during a rolling deploy, which means no migration may break the version of the code that is still running.
- Prefer small tables with clear responsibilities over wide tables of optional columns.

## Industry-level snippet

The bookstore models hardened with constraints and indexes: non-negative prices, valid formats, one title per author regardless of case, ISBN uniqueness among active books only (with soft delete), one review per reviewer per book, ratings between 1 and 5, and indexes shaped for the queries from earlier notes. The tests prove each rule at the database level and at the validation level.

Changes to `apps/bookstore/models.py`:

```python
# Book: remove unique=True from isbn, add soft delete, and replace Meta.
isbn = models.CharField(max_length=13, validators=[validate_isbn13])
deleted_at = models.DateTimeField(null=True, blank=True)


class Meta:
    ordering = ["title"]
    constraints = [
        models.CheckConstraint(
            condition=models.Q(price__gte=0),
            name="book_price_non_negative",
            violation_error_message="Price must not be negative.",
        ),
        # Keep in sync with Book.Format.
        models.CheckConstraint(
            condition=models.Q(format__in=["hardcover", "paperback", "ebook"]),
            name="book_format_valid",
        ),
        models.UniqueConstraint(
            Lower("title"),
            "author",
            name="book_unique_title_per_author",
            violation_error_message="This author already has a book with that title.",
        ),
        models.UniqueConstraint(
            fields=["isbn"],
            condition=models.Q(deleted_at__isnull=True),
            name="book_unique_active_isbn",
            violation_error_message="An active book with this ISBN already exists.",
        ),
    ]
    indexes = [
        models.Index(fields=["author", "-published_on"], name="book_author_published_idx"),
        models.Index(
            fields=["stock"],
            condition=models.Q(stock__lte=5),
            name="book_low_stock_idx",
        ),
    ]
```

```python
# Review: add Meta.
class Meta:
    constraints = [
        models.CheckConstraint(
            condition=models.Q(rating__gte=1, rating__lte=5),
            name="review_rating_between_1_and_5",
            violation_error_message="Rating must be between 1 and 5.",
        ),
        models.UniqueConstraint(
            fields=["book", "reviewer_name"],
            name="review_one_per_reviewer",
            violation_error_message="You have already reviewed this book.",
        ),
    ]
    indexes = [
        models.Index(fields=["book", "-created_at"], name="review_book_created_idx"),
    ]
```

Before running `makemigrations`, make sure existing data satisfies the new rules, as the migration will otherwise fail while creating the constraints. Because `isbn` is no longer unique at the field level, the import service from note 2.4 should keep treating any existing ISBN, including soft-deleted rows, as already present, or filter on `deleted_at__isnull=True` if re-importing deleted books is wanted.

`apps/bookstore/tests/test_constraints.py`:

```python
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.db import IntegrityError, connection, transaction
from django.test import TestCase
from django.utils import timezone

from apps.bookstore.models import Book, Review

from .factories import make_author, make_book, make_isbn


class BookConstraintTests(TestCase):
    def setUp(self) -> None:
        self.author = make_author()

    def create_book(self, **overrides: object) -> Book:
        values: dict[str, object] = {
            "title": "Dune",
            "isbn": make_isbn(900_001),
            "author": self.author,
            "price": Decimal("10.00"),
        }
        values.update(overrides)
        return Book.objects.create(**values)

    def test_database_rejects_a_negative_price(self) -> None:
        with self.assertRaises(IntegrityError), transaction.atomic():
            self.create_book(price=Decimal("-1.00"))

    def test_database_rejects_an_unknown_format(self) -> None:
        with self.assertRaises(IntegrityError), transaction.atomic():
            self.create_book(format="comic")

    def test_title_is_unique_per_author_ignoring_case(self) -> None:
        self.create_book(title="Dune")
        with self.assertRaises(IntegrityError), transaction.atomic():
            self.create_book(title="DUNE", isbn=make_isbn(900_002))

    def test_same_title_is_allowed_for_a_different_author(self) -> None:
        self.create_book(title="Dune")
        other = make_author("Someone Else")
        self.create_book(title="Dune", isbn=make_isbn(900_003), author=other)
        self.assertEqual(Book.objects.filter(title="Dune").count(), 2)

    def test_isbn_must_be_unique_only_among_active_books(self) -> None:
        isbn = make_isbn(900_004)
        old = self.create_book(title="Old edition", isbn=isbn)
        old.deleted_at = timezone.now()
        old.save(update_fields=["deleted_at"])

        self.create_book(title="New edition", isbn=isbn)  # allowed: the old one is deleted

        with self.assertRaises(IntegrityError), transaction.atomic():
            self.create_book(title="Third edition", isbn=isbn)

    def test_model_validation_reports_constraint_violations_politely(self) -> None:
        self.create_book(title="Dune")
        duplicate = Book(
            title="dune",
            isbn=make_isbn(900_005),
            author=self.author,
            price=Decimal("10.00"),
        )
        with self.assertRaises(ValidationError) as context:
            duplicate.full_clean()
        self.assertIn("already has a book with that title", str(context.exception))

    def test_declared_indexes_exist(self) -> None:
        with connection.cursor() as cursor:
            constraints = connection.introspection.get_constraints(
                cursor, Book._meta.db_table
            )
        self.assertIn("book_author_published_idx", constraints)
        self.assertIn("book_low_stock_idx", constraints)


class ReviewConstraintTests(TestCase):
    def setUp(self) -> None:
        self.book = make_book()

    def test_ratings_outside_one_to_five_are_rejected(self) -> None:
        for rating in (0, 6):
            with self.subTest(rating=rating):
                with self.assertRaises(IntegrityError), transaction.atomic():
                    Review.objects.create(
                        book=self.book, reviewer_name=f"R{rating}", rating=rating
                    )

    def test_one_review_per_reviewer_and_book(self) -> None:
        Review.objects.create(book=self.book, reviewer_name="Ada", rating=5)
        with self.assertRaises(IntegrityError), transaction.atomic():
            Review.objects.create(book=self.book, reviewer_name="Ada", rating=4)

    def test_validation_message_for_duplicate_review(self) -> None:
        Review.objects.create(book=self.book, reviewer_name="Ada", rating=5)
        duplicate = Review(book=self.book, reviewer_name="Ada", rating=4)
        with self.assertRaises(ValidationError) as context:
            duplicate.full_clean()
        self.assertIn("already reviewed", str(context.exception))
```

Each constraint is tested twice in spirit: once by attempting a write that bypasses Python validation (the database must refuse it), and once through `full_clean()` (the user must get a friendly message).

## Common mistakes

- Relying on `choices` or Python validation alone for rules that must always hold.
- Adding a constraint to a table with violating rows and being surprised when the migration fails.
- Forgetting that `NULL` slips through `CHECK` conditions and through unique constraints.
- Creating indexes for every column "just in case", slowing writes and wasting space.
- Writing a composite index with the columns in an order that does not match the queries.
- Writing a partial index whose condition the query does not imply, so it is never used.
- Using soft delete without partial unique constraints, so deleted rows block new ones.
- Forgetting to filter deleted rows in some queries.
- Storing core, frequently queried attributes in a `JSONField`.
- Exceeding the 30-character limit on index names, or reusing constraint names.
- Adding large constraints or indexes to busy production tables without understanding the locking they cause.
- Renaming or repurposing columns in a single deploy, breaking the old code still running.

## Practice exercises

1. Add a `CheckConstraint` to `Order` so that `status` can only hold the four valid values, plus a constraint on `OrderItem` requiring `quantity > 0` and `unit_price >= 0`. Write tests that bypass Python validation.
2. Add `published_on` and `retired_on` dates to `Book` with a constraint that `retired_on` is either null or not before `published_on`. Test the null case, the equal case, and the violating case.
3. Replace `unique=True` on `Genre.name` with a case-insensitive `UniqueConstraint` and test that "Fantasy" and "fantasy" collide.
4. Use `EXPLAIN` on PostgreSQL to show that the query "books with stock at most 5" uses the partial index when the filter matches, and does not when you filter on `stock__lte=10`. Explain why.
5. Design an index for the query "latest 20 reviews for a book" and prove with `EXPLAIN` that the sort does not need a separate step.
6. Write a data migration plus constraint migration that adds a case-insensitive unique title per author to a table that already contains duplicates, deciding how to resolve the duplicates.
7. Compare two designs for storing book awards, a `JSONField` list on `Book` and an `Award` table, by writing the query "all books that won any award in 1969" for each, and list what each design makes easy or hard.

## Summary

Constraints enforce rules on every write regardless of which code performs it, so rules that must always hold belong in the database: `CheckConstraint` for conditions, `UniqueConstraint` for uniqueness (including partial and expression-based forms), and `db_default` for universal defaults. Constraints also take part in model validation, giving users readable errors. Indexes speed up the queries you actually run, with composite order, partial conditions, and covering columns chosen from real queries and confirmed with `EXPLAIN`, at a cost on every write. Use soft delete, `JSONField`, and PostgreSQL-specific features deliberately, because each carries trade-offs. Design schemas for additive, multi-step evolution so that deploys never break running code.
