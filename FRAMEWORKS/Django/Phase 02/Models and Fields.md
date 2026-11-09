## Overview

A Django model is a Python class that describes one database table. Each attribute that is a field becomes a column, and each instance of the class represents one row. Models are the center of a Django application: forms, the admin, serializers, and queries are all built from them. This note covers defining models, the field types and their options, the difference between `null` and `blank`, `default` and `db_default`, choices, `Meta` options, `__str__`, generated fields, primary keys, and how models map onto tables.

This note starts the bookstore app (`apps/bookstore`) that the rest of Phase 2 builds on. Create the app as shown in note 0.6 and register `apps.bookstore.apps.BookstoreConfig` in `INSTALLED_APPS`.

## Core concepts

### A model is a table definition

```python
class Author(models.Model):
    name = models.CharField(max_length=200)
```

This describes a table named `bookstore_author` with an auto-generated primary key column `id` and a `name` column of type `varchar(200)`. Django never writes the SQL for you to read; it generates migrations (note 2.3) that create the table.

### Common field types

- Text: `CharField` (needs `max_length`), `TextField` (unbounded), `SlugField`, `EmailField`, `URLField`.
- Numbers: `IntegerField`, `BigIntegerField`, `PositiveIntegerField`, `SmallIntegerField`, `FloatField`, `DecimalField(max_digits, decimal_places)`.
- Truth and time: `BooleanField`, `DateField`, `DateTimeField`, `TimeField`, `DurationField`.
- Identity and structure: `UUIDField`, `JSONField`, `BinaryField`.
- Files: `FileField`, `ImageField` (Phase 3).
- Relations: `ForeignKey`, `OneToOneField`, `ManyToManyField` (note 2.6).

### Field options you will use constantly

- `null=True`: the database column may hold `NULL`.
- `blank=True`: validation allows an empty value (forms and `full_clean`). It has no effect on the database.
- `default=`: a Python value or callable used when no value is supplied.
- `db_default=` (Django 5.0): a database-side default expression, applied by the database itself.
- `unique=True`: the column must be unique, enforced by a unique index.
- `db_index=True`: create an index on the column.
- `choices=`: restrict the allowed values for validation and give human-readable labels.
- `validators=`: functions that raise `ValidationError` for bad values.
- `editable=False`: exclude the field from forms and the admin.
- `verbose_name` and `help_text`: labels and descriptions for forms and the admin.

### null versus blank

These are independent.

- `null` is about storage: can the column be `NULL`?
- `blank` is about validation: may a form submit nothing?

For string fields, use `blank=True` without `null=True`, and store the empty string for "no value". Allowing both `NULL` and `''` creates two representations of "empty". For non-string fields such as dates, numbers, and foreign keys, an optional field needs both `null=True` and `blank=True`.

### Choices

`TextChoices` and `IntegerChoices` define choices as enumerations with labels.

```python
class Format(models.TextChoices):
    HARDCOVER = "hardcover", "Hardcover"
    PAPERBACK = "paperback", "Paperback"
```

### Meta options

The inner `Meta` class controls table-level behavior: `ordering`, `verbose_name`, `db_table`, `constraints`, `indexes`, `abstract`, `managed`, and more.

### __str__

Define `__str__` on every model. The admin, the shell, and error messages use it. The Ruff `DJ008` rule flags models without it.

### Primary keys

Without a declared primary key, Django adds `id`, typed by `default_auto_field` (set to `BigAutoField` in the `AppConfig`). Declare a field with `primary_key=True` to choose your own. A `UUIDField` primary key avoids guessable IDs but is larger and less cache-friendly than an integer. A common compromise is an integer primary key plus a separate `public_id` UUID used in URLs and APIs.

### GeneratedField (Django 5.0)

A `GeneratedField` is a column whose value the database computes from other columns. It is read-only in Python and always consistent. On Django versions before 5.0, compute the value in a property or with an annotation.

### Mapping to tables and inspectdb

The table is named `<app_label>_<model_name>` in lowercase. `ForeignKey` fields create a column named `<field>_id`. For an existing database, `manage.py inspectdb` prints model classes that match the tables it finds, which is a starting point you must review and edit.

## Basic syntax examples

### A first model

```python
from django.db import models


class Author(models.Model):
    name = models.CharField(max_length=200)
    born_on = models.DateField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self) -> str:
        return self.name
```

### null and blank

```python
class Profile(models.Model):
    nickname = models.CharField(max_length=50, blank=True)   # '' means none
    birth_date = models.DateField(null=True, blank=True)     # NULL means none
    score = models.IntegerField(null=True, blank=True)
```

### default and callable defaults

```python
import uuid

from django.db import models
from django.utils import timezone


class Ticket(models.Model):
    code = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    opened_at = models.DateTimeField(default=timezone.now)
```

Pass the function itself (`default=uuid.uuid4`), not its result (`uuid.uuid4()`), otherwise every row would get the same value computed at import time.

### Choices with TextChoices and IntegerChoices

```python
class Book(models.Model):
    class Format(models.TextChoices):
        HARDCOVER = "hardcover", "Hardcover"
        PAPERBACK = "paperback", "Paperback"
        EBOOK = "ebook", "E-book"

    format = models.CharField(
        max_length=10,
        choices=Format,
        default=Format.PAPERBACK,
    )


class Review(models.Model):
    class Rating(models.IntegerChoices):
        ONE = 1, "One star"
        TWO = 2, "Two stars"
        THREE = 3, "Three stars"
        FOUR = 4, "Four stars"
        FIVE = 5, "Five stars"

    rating = models.PositiveSmallIntegerField(choices=Rating)
```

Passing the enumeration class directly to `choices` works from Django 5.0. On older versions write `choices=Format.choices`. Use `book.get_format_display()` for the label and `Book.Format.PAPERBACK` for comparisons.

### Meta

```python
class Author(models.Model):
    name = models.CharField(max_length=200)

    class Meta:
        ordering = ["name"]
        verbose_name = "author"
        verbose_name_plural = "authors"
```

### db_default and GeneratedField

```python
from django.db.models.functions import Lower, Now


class Note(models.Model):
    title = models.CharField(max_length=200)
    created_at = models.DateTimeField(db_default=Now())
    title_lower = models.GeneratedField(
        expression=Lower("title"),
        output_field=models.CharField(max_length=200),
        db_persist=True,
    )
```

### inspectdb

```bash
uv run python manage.py inspectdb > legacy_models.py
```

## Deeper behavior

### What the database actually stores

For the `Book` model above, on PostgreSQL Django generates a table roughly like this:

```sql
CREATE TABLE "bookstore_book" (
    "id" bigint NOT NULL PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    "title" varchar(200) NOT NULL,
    "format" varchar(10) NOT NULL,
    "price" numeric(8, 2) NOT NULL,
    "author_id" bigint NOT NULL REFERENCES "bookstore_author" ("id")
);
```

You will see real output with `sqlmigrate` in note 2.3. Choices are not checked by the database. Only validation methods enforce them, which is why note 2.10 adds a check constraint when the database must enforce the rule.

### default versus db_default

`default` is applied by Python when you create an instance. `db_default` is applied by the database when a row is inserted without a value, including inserts that bypass your Python code, such as raw SQL and `bulk_create`. Use `db_default` for values that should hold no matter who writes the row. Before an instance with a `db_default` field is saved, the attribute holds a placeholder expression rather than a value, so use `default` as well if Python code reads the field before saving. After saving, Django refreshes the value on databases that support `RETURNING`, which includes PostgreSQL and modern SQLite.

### auto_now and auto_now_add

`auto_now_add=True` sets the field once, on creation. `auto_now=True` updates it on every `save()`, but not on `QuerySet.update()` or `bulk_update()`, which surprises many people. Both make the field non-editable. If you need a value you can override, use `default=timezone.now` instead.

### Why not use floats for money

`FloatField` stores binary floating point numbers, which cannot represent most decimal fractions exactly. Use `DecimalField(max_digits=8, decimal_places=2)` and `Decimal` values in Python, or store integer cents.

### Validation does not run on save

`save()` does not call `full_clean()`. Validators, `blank`, `choices`, and field-level checks only run when a form, serializer, or your own code calls validation. Only database constraints (`NOT NULL`, `UNIQUE`, `CHECK`, foreign keys, field lengths) protect rows written by `save()`. This is why Phase 3 describes validation in layers.

### null=True on strings

With `CharField(null=True)`, "no value" can be `NULL` or `''`, and queries such as `filter(nickname="")` silently miss the `NULL` rows. The exception is a unique string field that must allow many empty values, because `NULL`s do not collide in a unique index and empty strings do.

### Primary key choices

- `BigAutoField` is the default and the right choice for most tables.
- Random UUID primary keys scatter inserts across the index, which can hurt write performance on large tables, and they are bigger in every foreign key.
- Exposing sequential integer IDs in URLs makes records enumerable, so pair them with authorization checks (Phase 4), or expose a `public_id`.

### GeneratedField details

- `db_persist=True` stores the value; `False` computes it on read where the database supports it.
- The expression must be deterministic and can reference only other fields in the same row.
- It cannot be set in the constructor or assigned later, and it is excluded from forms.
- Because the database computes it, read it after `refresh_from_db()` where your database does not return it from the insert.

### Legacy tables

`inspectdb` output uses `managed = False` in `Meta` so Django does not try to create or alter those tables. Review field types, names, and relations by hand, and add what it cannot infer (such as `choices`, validators, and `related_name`).

## Industry-level snippet

A production-style version of the bookstore's two core models, with a real ISBN-13 checksum validator, choices, a decimal price with a non-negative validator, a public UUID, a database-computed lowercase title, and a database default timestamp. Tests prove the behavior, including the important fact that `save()` does not validate.

`apps/bookstore/models.py`:

```python
import uuid
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.core.validators import MinValueValidator
from django.db import models
from django.db.models.functions import Lower, Now


def validate_isbn13(value: str) -> None:
    """Validate the length, digits, and check digit of an ISBN-13."""
    if len(value) != 13 or not (value.isascii() and value.isdigit()):
        raise ValidationError("ISBN must be exactly 13 digits.", code="invalid_isbn")
    total = sum(int(digit) * (1 if index % 2 == 0 else 3) for index, digit in enumerate(value))
    if total % 10 != 0:
        raise ValidationError("ISBN check digit is incorrect.", code="invalid_isbn_checksum")


class Author(models.Model):
    name = models.CharField(max_length=200)
    born_on = models.DateField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["name"]

    def __str__(self) -> str:
        return self.name


class Book(models.Model):
    class Format(models.TextChoices):
        HARDCOVER = "hardcover", "Hardcover"
        PAPERBACK = "paperback", "Paperback"
        EBOOK = "ebook", "E-book"

    public_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    title = models.CharField(max_length=200)
    isbn = models.CharField(
        max_length=13,
        unique=True,
        validators=[validate_isbn13],
        help_text="13 digits, no hyphens.",
    )
    author = models.ForeignKey(Author, on_delete=models.PROTECT, related_name="books")
    price = models.DecimalField(
        max_digits=8,
        decimal_places=2,
        validators=[MinValueValidator(Decimal("0"))],
    )
    stock = models.PositiveIntegerField(default=0)
    format = models.CharField(
        max_length=10,
        choices=Format,
        default=Format.PAPERBACK,
    )
    published_on = models.DateField(null=True, blank=True)
    created_at = models.DateTimeField(db_default=Now())
    title_lower = models.GeneratedField(
        expression=Lower("title"),
        output_field=models.CharField(max_length=200),
        db_persist=True,
    )

    class Meta:
        ordering = ["title"]

    def __str__(self) -> str:
        return self.title
```

`apps/bookstore/tests/test_models.py` (with an empty `__init__.py` in `tests/`):

```python
import uuid
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.test import TestCase

from apps.bookstore.models import Author, Book

VALID_ISBN = "9780306406157"


class BookModelTests(TestCase):
    def setUp(self) -> None:
        self.author = Author.objects.create(name="Ursula K. Le Guin")

    def make_book(self, **overrides: object) -> Book:
        values: dict[str, object] = {
            "title": "Dune",
            "isbn": VALID_ISBN,
            "author": self.author,
            "price": Decimal("12.50"),
        }
        values.update(overrides)
        return Book(**values)

    def test_defaults_are_applied(self) -> None:
        book = self.make_book()
        book.save()
        self.assertEqual(book.stock, 0)
        self.assertEqual(book.format, Book.Format.PAPERBACK)
        self.assertIsInstance(book.public_id, uuid.UUID)

    def test_choice_label_is_available(self) -> None:
        book = self.make_book(format=Book.Format.EBOOK)
        self.assertEqual(book.get_format_display(), "E-book")

    def test_database_computes_generated_and_default_columns(self) -> None:
        book = self.make_book(title="Dune")
        book.save()
        book.refresh_from_db()
        self.assertEqual(book.title_lower, "dune")
        self.assertIsNotNone(book.created_at)

    def test_valid_isbn_passes_full_clean(self) -> None:
        self.make_book().full_clean()  # does not raise

    def test_invalid_isbn_reports_a_specific_code(self) -> None:
        cases = {
            "123": "invalid_isbn",
            "97803064061AB": "invalid_isbn",
            "9780306406158": "invalid_isbn_checksum",
        }
        for isbn, code in cases.items():
            with self.subTest(isbn=isbn):
                with self.assertRaises(ValidationError) as context:
                    self.make_book(isbn=isbn).full_clean()
                errors = context.exception.error_dict["isbn"]
                self.assertEqual(errors[0].code, code)

    def test_negative_price_fails_validation(self) -> None:
        with self.assertRaises(ValidationError) as context:
            self.make_book(price=Decimal("-1")).full_clean()
        self.assertIn("price", context.exception.error_dict)

    def test_save_does_not_validate(self) -> None:
        # Only database constraints protect rows written with save().
        book = self.make_book(isbn="not-an-isbn", format="comic")
        book.save()
        self.assertTrue(Book.objects.filter(isbn="not-an-isbn").exists())

    def test_string_representation(self) -> None:
        self.assertEqual(str(self.make_book(title="Dune")), "Dune")
        self.assertEqual(str(self.author), "Ursula K. Le Guin")
```

The last two tests together make an important point: validators protect only the paths that call validation, so rules that must always hold also need database constraints (note 2.10).

## Common mistakes

- Using `null=True` on `CharField` and `TextField`, creating two kinds of "empty".
- Confusing `blank` (validation) with `null` (storage), then wondering why a form accepts an empty value but the database refuses it, or the reverse.
- Passing `default=uuid.uuid4()` or `default=timezone.now()` with parentheses, which freezes one value at import time.
- Storing money in `FloatField`.
- Assuming `choices` are enforced by the database.
- Assuming `save()` validates, and skipping `full_clean()` or constraints.
- Forgetting `__str__`, which makes the admin and debugging output unreadable.
- Relying on `auto_now` fields being updated by `QuerySet.update()` or `bulk_update()`.
- Using random UUIDs as primary keys on very large tables without measuring the effect.
- Editing the model class and forgetting to create and apply a migration.

## Practice exercises

1. Add `Publisher` to the bookstore with a name, an optional website, and a founding year restricted to 1400 through the current year using validators. Write tests for valid and invalid values.
2. Add a `language` field to `Book` using `TextChoices` with at least four languages and a default. Write a test that the label is shown by `get_language_display()`.
3. Decide `null` and `blank` for these fields and justify each choice: a book subtitle, an author's death date, a review's body, an optional ISBN-10.
4. Write a validator `validate_not_future(value: date) -> None` and apply it to `published_on` and `born_on`. Control the date in tests so they stay deterministic.
5. Add a `GeneratedField` to `Book` that stores the number of characters in the title (use `Length`). Observe what happens when you try to assign to it.
6. Run `inspectdb` against a SQLite or PostgreSQL database you created with plain SQL in note 2.1 and list three things you would edit in the generated models.
7. Explain, with a concrete bookstore example, a case where `default` is better than `db_default` and one where `db_default` is better.

## Summary

A model class defines a table, its fields define columns, and its `Meta` class defines table-level behavior. Choose field types deliberately: `DecimalField` for money, `TextChoices` for fixed sets of values, and string fields with `blank=True` rather than `null=True`. `null` is storage and `blank` is validation, `default` runs in Python and `db_default` runs in the database, and `GeneratedField` lets the database compute a value. Remember that `save()` does not validate and `choices` are not a database constraint, so protect important rules at the database level too. Define `__str__` on every model, and review `inspectdb` output carefully when adopting an existing database.
