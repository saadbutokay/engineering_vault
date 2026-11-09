## Overview

As an application grows, the same filters appear everywhere: only active books, only in-stock books, only reviews from the last month. Copy-pasting `filter(...)` calls spreads business rules across views, templates, and tasks, and the copies drift apart. Django's answer is to give models a vocabulary of their own: custom `QuerySet` classes with chainable methods, managers that expose them, and model methods and properties for behavior tied to a single instance. This note covers default and custom managers, `QuerySet` subclasses, `as_manager` and `from_queryset`, model methods, validation methods and why `save()` does not call them, abstract base classes, proxy models, multi-table inheritance and its costs, where business logic belongs (fat models versus a service layer), and raw SQL for the cases the ORM cannot express.

## Core concepts

### Managers

A manager is the interface through which queries start: `Book.objects`. Every model has at least one. The first manager declared on a model is its default manager, available as `Model._default_manager`, and Django uses it internally, for example in related lookups and the admin.

### Custom QuerySets

A `QuerySet` subclass holds methods that return querysets, so they chain: `Book.objects.alive().in_stock().low_stock()`. This is the preferred place for reusable filters, annotations, and bulk actions, because methods defined on the queryset are available at every step of a chain.

### as_manager and from_queryset

- `BookQuerySet.as_manager()` creates a manager that exposes the queryset's methods.
- `models.Manager.from_queryset(BookQuerySet)` creates a manager class you can subclass or reuse, and it is the form that `django-stubs` understands well for type checking.

### Model methods and properties

Methods and properties on the model class describe behavior of one instance: `is_available`, `display_price`, `apply_to(price)`. They are good for logic that depends only on the instance's own data.

### Validation methods

- `clean_fields()` validates each field.
- `clean()` is your hook for rules involving several fields, and the place to normalize values.
- `validate_unique()` and `validate_constraints()` check uniqueness and constraints against the database.
- `full_clean()` runs all of them.

`save()` calls none of them. Forms and serializers call them; your own code must call `full_clean()` explicitly when it needs the checks.

### Abstract base classes

An abstract model (`class Meta: abstract = True`) defines fields and methods shared by several models but creates no table of its own. Each child gets its own copy of the columns. Use it for shared timestamps, ownership fields, or soft delete columns.

### Proxy models

A proxy model (`class Meta: proxy = True`) is a Python-level subclass over the same table. It can have a different default manager, ordering, methods, and admin registration, but it cannot add fields.

### Multi-table inheritance

A non-abstract subclass of a concrete model creates a second table linked to the parent's by an implicit one-to-one key. Every query on the child joins both tables, and the parent and child rows must be kept in step. It is rarely the best choice.

### Where business logic belongs

Logic that derives from one instance's state (a total, a status check, a formatted label) fits naturally on the model. Logic that coordinates several models, touches external systems, needs transactions, or has side effects fits better in a service function. Keep models focused and services explicit, and avoid hiding side effects in `save()` or signals (note 5.3).

### Raw SQL

When a query cannot be expressed or optimized through the ORM, you can use `Model.objects.raw()` (returns model instances) or `connection.cursor()` (returns tuples). Always pass values as parameters, never through string formatting.

## Basic syntax examples

### A custom QuerySet and manager

```python
from decimal import Decimal
from typing import Self

from django.db import models


class BookQuerySet(models.QuerySet["Book"]):
    def alive(self) -> Self:
        return self.filter(deleted_at__isnull=True)

    def in_stock(self) -> Self:
        return self.filter(stock__gt=0)

    def priced_between(self, minimum: Decimal, maximum: Decimal) -> Self:
        return self.filter(price__gte=minimum, price__lte=maximum)


BookManager = models.Manager.from_queryset(BookQuerySet)


class Book(models.Model):
    # ... fields ...
    objects = BookManager()
```

Now both of these work and return a `BookQuerySet`:

```python
Book.objects.alive().in_stock().priced_between(Decimal("5"), Decimal("15"))
Book.objects.filter(format="ebook").in_stock()
```

### The as_manager shortcut

```python
class Review(models.Model):
    # ... fields ...
    objects = ReviewQuerySet.as_manager()
```

### Model methods and properties

```python
class Book(models.Model):
    # ... fields ...

    @property
    def is_available(self) -> bool:
        return self.deleted_at is None and self.stock > 0

    def price_with_tax(self, rate: Decimal) -> Decimal:
        return (self.price * (1 + rate)).quantize(Decimal("0.01"))
```

### clean and full_clean

```python
from django.core.exceptions import ValidationError


class Coupon(models.Model):
    valid_from = models.DateTimeField()
    valid_until = models.DateTimeField()

    def clean(self) -> None:
        super().clean()
        if self.valid_from and self.valid_until and self.valid_until <= self.valid_from:
            raise ValidationError({"valid_until": "Must be after the start."})


coupon = Coupon(valid_from=start, valid_until=start)
coupon.full_clean()   # raises ValidationError
coupon.save()         # would save it without any validation
```

### An abstract base class

```python
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Publisher(TimeStampedModel):
    name = models.CharField(max_length=200)
```

### A proxy model

```python
class SoldOutBookManager(models.Manager["Book"]):
    def get_queryset(self) -> models.QuerySet["Book"]:
        return super().get_queryset().filter(stock=0)


class SoldOutBook(Book):
    objects = SoldOutBookManager()

    class Meta:
        proxy = True
```

### Raw SQL with model instances

```python
books = Book.objects.raw(
    "SELECT * FROM bookstore_book WHERE stock < %s ORDER BY stock",
    [5],
)
for book in books:
    print(book.title)
```

### Raw SQL with a cursor

```python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT COUNT(*) FROM bookstore_book WHERE stock = %s", [0])
    (count,) = cursor.fetchone()
```

## Deeper behavior

### Why querysets, not manager methods

If you define `in_stock()` only on the manager, then `Book.objects.filter(...).in_stock()` fails, because `filter()` returns a queryset that does not have the method. Methods on the `QuerySet` subclass work anywhere in a chain, and `from_queryset` and `as_manager` copy them onto the manager so `Book.objects.in_stock()` works too. Methods whose names start with an underscore are not copied, and a method can opt out with `queryset_only = True`.

### Default manager pitfalls

If you replace the default manager's `get_queryset()` to hide rows (for instance, soft-deleted ones), those rows disappear from every query through that manager, including `get()` calls inside your own code, and the admin and related-object access may hide them too in surprising ways. Prefer keeping the default manager unfiltered and adding explicit methods such as `alive()`. If you do hide rows, keep a second manager that sees everything and declare the unfiltered one first.

### Related managers and base managers

Related access (`author.books`) uses a manager built from the model's default manager, and some internal operations use `_base_manager`, which is unfiltered by default. This is another reason to avoid filtering default managers.

### Custom managers and migrations

A manager is not part of migrations unless it sets `use_in_migrations = True`. That is why historical models in `RunPython` lack your custom methods (note 2.3).

### Validation in depth

`full_clean()` runs `clean_fields()`, then `clean()`, then `validate_unique()`, then `validate_constraints()`, and gathers all errors into one `ValidationError`. Raising a dictionary from `clean()` attaches the error to a field. Raising a plain message attaches it to `NON_FIELD_ERRORS`. Because `save()` skips this, put validation where every write path passes through it: a form, a serializer, a service function, and database constraints for the rules that must be unbreakable.

### Overriding save

Overriding `save()` to validate or to trigger side effects looks convenient, but `bulk_create`, `update`, and raw SQL bypass it, so the rule holds only sometimes. That inconsistency is worse than an explicit service call. If you override it, call `super().save(*args, **kwargs)`, and keep the override small.

### Abstract base class details

- Fields and methods are inherited, and `Meta` is inherited unless the child defines its own (child `Meta` should extend the parent's: `class Meta(Parent.Meta)`).
- A relation defined on an abstract class needs `related_name="%(app_label)s_%(class)s_items"` so reverse names do not clash between children.
- Abstract models cannot be queried, and they do not appear in the database.

### Proxy model details

A proxy shares the table, so rows saved through the proxy are visible through the parent and vice versa. Use proxies to attach different behavior or admin configuration to a subset of the same data. Note that permissions and content types are created separately for each proxy.

### Multi-table inheritance costs

With `class Restaurant(Place)`, each query on `Restaurant` joins `Place`, `Place.restaurant` runs an extra query, and deleting the parent deletes the child. Bulk operations are restricted. Alternatives are an abstract base class (shared columns), a one-to-one profile table (optional extra data), or composition. Reach for multi-table inheritance only when you genuinely need to query parents polymorphically.

### Fat models versus services

A "fat model" keeps all business logic in the model, which is simple at first and tangled later: methods that open transactions, send email, and call other models create hidden coupling and hard-to-test behavior. A service layer keeps orchestration in functions that are easy to read, test, and reuse from views, commands, and tasks (note 13.1). A pragmatic rule: pure computation on one instance goes on the model, and anything with side effects or several models goes in a service.

### When raw SQL is justified

- A query the ORM cannot express, or expresses very inefficiently, such as recursive common table expressions or database-specific features.
- Performance-critical reports verified with `EXPLAIN`.
- Bulk data operations too heavy for row-by-row ORM calls.

Costs: raw SQL is tied to a database dialect, bypasses model logic and type checking, and is a security risk if built with string formatting. Parameters (`%s` placeholders with a separate list) are always handled safely by the driver. Table and column names cannot be parameters, so take them from `Model._meta` and quote them with `connection.ops.quote_name`, never from user input. `Model.objects.raw()` must select the primary key column, and fields you do not select are loaded lazily, one query per object.

## Industry-level snippet

A custom queryset and manager for books, a shared abstract base class, a `Coupon` model with validation, a constraint, and pure behavior methods, a proxy model, and a raw SQL sales report that takes its table names from model metadata and its values from parameters. Everything has tests.

`apps/bookstore/managers.py`:

```python
"""Custom QuerySet and manager for books."""

from decimal import Decimal
from typing import Self

from django.db import models
from django.db.models import Avg, Count
from django.utils import timezone


class BookQuerySet(models.QuerySet["Book"]):
    def alive(self) -> Self:
        """Books that have not been soft-deleted."""
        return self.filter(deleted_at__isnull=True)

    def in_stock(self) -> Self:
        return self.filter(stock__gt=0)

    def low_stock(self, threshold: int = 5) -> Self:
        if threshold < 0:
            raise ValueError("threshold must not be negative")
        return self.filter(stock__gt=0, stock__lte=threshold)

    def priced_between(self, minimum: Decimal, maximum: Decimal) -> Self:
        if minimum > maximum:
            raise ValueError("minimum must not exceed maximum")
        return self.filter(price__gte=minimum, price__lte=maximum)

    def by_format(self, *formats: str) -> Self:
        return self.filter(format__in=formats)

    def with_review_stats(self) -> Self:
        return self.annotate(
            review_count=Count("reviews"),
            average_rating=Avg("reviews__rating"),
        )

    def soft_delete(self) -> int:
        """Mark alive books as deleted; return how many changed."""
        return self.alive().update(deleted_at=timezone.now())

    def restore(self) -> int:
        return self.filter(deleted_at__isnull=False).update(deleted_at=None)


BookManager = models.Manager.from_queryset(BookQuerySet)
```

In `apps/bookstore/models.py`, import the manager and attach it (and add the proxy and coupon):

```python
from datetime import datetime
from decimal import ROUND_HALF_UP, Decimal

from django.core.exceptions import ValidationError
from django.utils import timezone

from apps.bookstore.managers import BookManager
from apps.core.models import TimeStampedModel


class Book(models.Model):
    # ... existing fields and Meta ...
    objects = BookManager()

    @property
    def is_available(self) -> bool:
        return self.deleted_at is None and self.stock > 0


class SoldOutBookManager(models.Manager["Book"]):
    def get_queryset(self) -> models.QuerySet["Book"]:
        return super().get_queryset().filter(stock=0)


class SoldOutBook(Book):
    objects = SoldOutBookManager()

    class Meta:
        proxy = True
        verbose_name = "sold-out book"


class Coupon(TimeStampedModel):
    code = models.CharField(max_length=32, unique=True)
    percent_off = models.PositiveSmallIntegerField()
    valid_from = models.DateTimeField()
    valid_until = models.DateTimeField()

    class Meta:
        constraints = [
            models.CheckConstraint(
                condition=models.Q(percent_off__gte=1, percent_off__lte=100),
                name="coupon_percent_valid",
            ),
            models.CheckConstraint(
                condition=models.Q(valid_until__gt=models.F("valid_from")),
                name="coupon_window_valid",
            ),
        ]

    def __str__(self) -> str:
        return self.code

    def clean(self) -> None:
        super().clean()
        self.code = self.code.strip().upper()
        if self.valid_from and self.valid_until and self.valid_until <= self.valid_from:
            raise ValidationError({"valid_until": "Must be after the start."})

    def is_active(self, at: datetime | None = None) -> bool:
        moment = at or timezone.now()
        return self.valid_from <= moment < self.valid_until

    def apply_to(self, price: Decimal) -> Decimal:
        if price < 0:
            raise ValueError("price must not be negative")
        discounted = price * (Decimal(100) - self.percent_off) / Decimal(100)
        return discounted.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```

The proxy's manager subclasses plain `models.Manager` rather than the dynamically created `BookManager`, which keeps it simple for type checkers. Note that `SoldOutBook.objects` therefore does not have the `BookQuerySet` methods; exercise 3 asks you to add them.

`apps/core/models.py`:

```python
from django.db import models


class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

`apps/bookstore/reports.py`:

```python
"""Reports written in raw SQL."""

import logging
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal

from django.db import connection

from apps.bookstore.models import Book, Order, OrderItem

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class SalesRow:
    book_id: int
    title: str
    units: int
    revenue: Decimal


def sales_report(*, since: datetime, limit: int = 10) -> list[SalesRow]:
    """Best-selling books since ``since``, excluding cancelled orders."""
    if since.tzinfo is None:
        raise ValueError("since must be timezone-aware")
    if not 1 <= limit <= 100:
        raise ValueError("limit must be between 1 and 100")

    quote = connection.ops.quote_name
    # Table names come from model metadata, never from user input.
    sql = f"""
        SELECT b.id, b.title,
               SUM(oi.quantity) AS units,
               SUM(oi.quantity * oi.unit_price) AS revenue
        FROM {quote(Book._meta.db_table)} AS b
        JOIN {quote(OrderItem._meta.db_table)} AS oi ON oi.book_id = b.id
        JOIN {quote(Order._meta.db_table)} AS o ON o.id = oi.order_id
        WHERE o.created_at >= %s AND o.status <> %s
        GROUP BY b.id, b.title
        ORDER BY units DESC, b.id
        LIMIT %s
    """  # noqa: S608
    with connection.cursor() as cursor:
        cursor.execute(sql, [since, Order.Status.CANCELLED.value, limit])
        rows = cursor.fetchall()

    logger.debug("Sales report returned %d rows", len(rows))
    return [
        SalesRow(int(row[0]), str(row[1]), int(row[2]), Decimal(str(row[3])))
        for row in rows
    ]
```

`apps/bookstore/tests/test_model_design.py`:

```python
from datetime import timedelta
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.db import IntegrityError, transaction
from django.test import TestCase
from django.utils import timezone

from apps.bookstore.models import Book, Coupon, Order, Review, SoldOutBook
from apps.bookstore.reports import sales_report
from apps.bookstore.services.orders import OrderLine, create_order

from .factories import make_book


class BookQuerySetTests(TestCase):
    def test_methods_chain_from_the_manager_and_from_querysets(self) -> None:
        make_book(title="Cheap", stock=3, price=Decimal("5.00"))
        make_book(title="Dear", stock=3, price=Decimal("50.00"))
        make_book(title="Empty", stock=0, price=Decimal("5.00"))
        titles = Book.objects.in_stock().priced_between(Decimal("1"), Decimal("10"))
        self.assertEqual(list(titles.values_list("title", flat=True)), ["Cheap"])
        chained = Book.objects.filter(title__startswith="C").alive().in_stock()
        self.assertEqual(chained.count(), 1)

    def test_low_stock_excludes_sold_out_books(self) -> None:
        make_book(title="Low", stock=2)
        make_book(title="Zero", stock=0)
        make_book(title="Plenty", stock=50)
        titles = Book.objects.low_stock(threshold=5).values_list("title", flat=True)
        self.assertEqual(list(titles), ["Low"])

    def test_invalid_arguments_are_rejected(self) -> None:
        with self.assertRaises(ValueError):
            Book.objects.low_stock(threshold=-1)
        with self.assertRaises(ValueError):
            Book.objects.priced_between(Decimal("10"), Decimal("1"))

    def test_soft_delete_and_restore(self) -> None:
        make_book()
        make_book()
        self.assertEqual(Book.objects.soft_delete(), 2)
        self.assertEqual(Book.objects.alive().count(), 0)
        self.assertEqual(Book.objects.soft_delete(), 0)   # already deleted
        self.assertEqual(Book.objects.restore(), 2)
        self.assertEqual(Book.objects.alive().count(), 2)

    def test_review_statistics_annotation(self) -> None:
        book = make_book(title="Reviewed")
        make_book(title="Unreviewed")
        Review.objects.create(book=book, reviewer_name="A", rating=5)
        Review.objects.create(book=book, reviewer_name="B", rating=3)
        stats = {
            title: (count, rating)
            for title, count, rating in Book.objects.with_review_stats().values_list(
                "title", "review_count", "average_rating"
            )
        }
        self.assertEqual(stats, {"Reviewed": (2, 4.0), "Unreviewed": (0, None)})

    def test_is_available_property(self) -> None:
        book = make_book(stock=1)
        self.assertTrue(book.is_available)
        book.deleted_at = timezone.now()
        self.assertFalse(book.is_available)


class ProxyModelTests(TestCase):
    def test_proxy_sees_only_sold_out_rows_of_the_same_table(self) -> None:
        make_book(title="Gone", stock=0)
        make_book(title="Here", stock=4)
        self.assertEqual(list(SoldOutBook.objects.values_list("title", flat=True)), ["Gone"])
        self.assertEqual(Book.objects.count(), 2)


class CouponTests(TestCase):
    def setUp(self) -> None:
        self.start = timezone.now()
        self.coupon = Coupon(
            code="  spring20 ",
            percent_off=20,
            valid_from=self.start,
            valid_until=self.start + timedelta(days=7),
        )

    def test_full_clean_normalizes_the_code(self) -> None:
        self.coupon.full_clean()
        self.assertEqual(self.coupon.code, "SPRING20")

    def test_inverted_window_fails_validation(self) -> None:
        self.coupon.valid_until = self.start - timedelta(days=1)
        with self.assertRaises(ValidationError) as context:
            self.coupon.full_clean()
        self.assertIn("valid_until", context.exception.error_dict)

    def test_database_enforces_the_window_even_without_validation(self) -> None:
        self.coupon.valid_until = self.start
        with self.assertRaises(IntegrityError), transaction.atomic():
            self.coupon.save()

    def test_database_enforces_the_percentage_range(self) -> None:
        for percent in (0, 101):
            with self.subTest(percent=percent):
                self.coupon.percent_off = percent
                self.coupon.code = f"C{percent}"
                with self.assertRaises(IntegrityError), transaction.atomic():
                    self.coupon.save()

    def test_is_active_includes_start_and_excludes_end(self) -> None:
        self.assertTrue(self.coupon.is_active(self.coupon.valid_from))
        self.assertFalse(self.coupon.is_active(self.coupon.valid_until))
        self.assertFalse(self.coupon.is_active(self.coupon.valid_from - timedelta(seconds=1)))

    def test_apply_to_rounds_half_up(self) -> None:
        self.assertEqual(self.coupon.apply_to(Decimal("19.99")), Decimal("15.99"))
        self.assertEqual(self.coupon.apply_to(Decimal("0.00")), Decimal("0.00"))
        with self.assertRaises(ValueError):
            self.coupon.apply_to(Decimal("-1"))


class SalesReportTests(TestCase):
    def test_report_ranks_books_and_ignores_cancelled_orders(self) -> None:
        first = make_book(title="First", price=Decimal("10.00"), stock=100)
        second = make_book(title="Second", price=Decimal("20.00"), stock=100)
        create_order(customer_email="a@example.com", lines=[OrderLine(first.pk, 5)])
        create_order(customer_email="b@example.com", lines=[OrderLine(second.pk, 2)])
        cancelled = create_order(
            customer_email="c@example.com", lines=[OrderLine(second.pk, 100)]
        )
        Order.objects.filter(pk=cancelled.pk).update(status=Order.Status.CANCELLED)

        rows = sales_report(since=timezone.now() - timedelta(days=1))

        self.assertEqual(
            [(row.title, row.units, row.revenue) for row in rows],
            [("First", 5, Decimal("50.00")), ("Second", 2, Decimal("40.00"))],
        )

    def test_arguments_are_validated(self) -> None:
        with self.assertRaises(ValueError):
            sales_report(since=timezone.now().replace(tzinfo=None))
        with self.assertRaises(ValueError):
            sales_report(since=timezone.now(), limit=0)
```

The report shows the raw SQL discipline in one place: identifiers from model metadata, every value as a parameter, validated arguments, and typed results instead of tuples leaking to callers.

## Common mistakes

- Putting reusable filters on the manager only, so they cannot be chained after `filter()`.
- Replacing the default manager's `get_queryset()` to hide rows, then being surprised when related lookups and admin pages hide them too.
- Assuming `save()` runs `full_clean()`.
- Hiding side effects (emails, network calls, other models' changes) inside `save()` or model methods, where `bulk_create` and `update()` bypass them and tests cannot see them.
- Using multi-table inheritance when an abstract base class or a one-to-one profile would do.
- Forgetting `related_name="%(class)s_..."` on relations in abstract base classes.
- Treating a proxy model as a different table.
- Building raw SQL with f-strings or `%` formatting that includes user-provided values.
- Using `raw()` without selecting the primary key.
- Reaching for raw SQL before checking whether annotations, subqueries, and window functions already express the query.
- Writing a fat model that opens transactions, calls APIs, and coordinates several other models.

## Practice exercises

1. Add `ReviewQuerySet` with `top_rated(minimum: int = 4)` and `recent(days: int = 30)` methods, attach it with `as_manager()`, and test chaining the two.
2. Add a `published()` method to `BookQuerySet` that returns books with a non-null `published_on` that is not in the future, and test the boundaries with a controlled clock.
3. Give `SoldOutBook` the `BookQuerySet` vocabulary by building its manager with `models.Manager.from_queryset(BookQuerySet)` and a restricted starting queryset, then test that `SoldOutBook.objects.low_stock()` and `SoldOutBook.objects.alive()` work and decide whether `low_stock()` makes sense on this proxy.
4. Create an abstract `OwnedModel` with an `owner_email` field and a relation that needs `related_name="%(class)s_items"`, use it in two models, and show the clash that happens without the placeholder.
5. Demonstrate that `save()` skips validation, `bulk_create()` skips `save()`, and `update()` skips both, with one test for each fact.
6. Rewrite `sales_report` using the ORM (annotations, joins, and `Sum`). Compare readability, query count, and result with the raw version, and decide which you would keep.
7. Write `Book.objects.raw()` for "books with stock below five" and show that omitting the primary key column raises an error, and that leaving out other columns causes extra queries when they are accessed.

## Summary

Custom `QuerySet` classes give your models a vocabulary of chainable filters, and `from_queryset` or `as_manager` expose them on `objects`. Keep the default manager unfiltered and add explicit methods instead. Put behavior derived from one instance on the model, and keep orchestration and side effects in service functions. Remember that `save()` does not validate: call `full_clean()` where needed and enforce unbreakable rules with constraints. Use abstract base classes to share fields, proxy models to vary behavior over one table, and avoid multi-table inheritance unless you need polymorphic queries. Use raw SQL only when the ORM falls short, with parameterized values, identifiers from model metadata, validated inputs, and typed results.
