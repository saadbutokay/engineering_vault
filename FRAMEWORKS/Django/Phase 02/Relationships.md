## Overview

Real data is connected: books have authors, authors have profiles, books belong to many genres, orders contain many books. Django models express these connections with three relationship fields (`ForeignKey`, `OneToOneField`, and `ManyToManyField`), and the database enforces them with foreign key constraints. This note covers each field, the `on_delete` options and their consequences, `related_name`, forward and reverse access, explicit `through` models, self-referencing relations, string references across apps, a brief look at generic relations, and composite primary keys introduced in Django 5.2. It ends with an order model that snapshots prices, protects history, and has tested deletion behavior.

This note extends the bookstore models from note 2.2.

## Core concepts

### ForeignKey (many-to-one)

Many rows on one side point to one row on the other. Many books have one author. The foreign key lives on the "many" side and creates a column named `<field>_id` with a database foreign key constraint and an index.

### OneToOneField

A foreign key with a unique constraint: each row can be referenced by at most one other row. Use it to split one conceptual entity across two tables, such as an author and an optional profile with rarely used columns.

### ManyToManyField

Many rows on each side: books and genres. The database stores the pairs in a separate join table, which Django creates automatically.

### The through model

When the relationship itself carries data, define the join table yourself and name it with `through=`. An order and its books are linked by order items that store a quantity and the price at the time of purchase.

### on_delete

Every `ForeignKey` and `OneToOneField` must say what happens when the referenced row is deleted:

- `CASCADE`: delete the referencing rows too.
- `PROTECT`: refuse the deletion by raising `ProtectedError`.
- `RESTRICT`: like `PROTECT`, but allows the delete if it would also be removed through another cascade path.
- `SET_NULL`: set the column to `NULL` (requires `null=True`).
- `SET_DEFAULT`: set the column to its default.
- `SET(value_or_callable)`: set it to a given value.
- `DO_NOTHING`: do nothing in Django and let the database decide, which usually raises an integrity error unless the database defines its own behavior.

These rules are enforced by Django's delete collector, not by the database, so deleting through raw SQL bypasses them.

### related_name and reverse access

Each relation has a reverse side. `related_name="books"` on `Book.author` lets you write `author.books.all()`. Without it Django uses `book_set`. `related_name="+"` disables the reverse accessor.

### Forward and reverse access

- Forward: `book.author` (one query on first access, then cached), and `book.author_id` (no query).
- Reverse many: `author.books` is a manager, so use `.all()`, `.filter(...)`, `.count()`.
- Reverse one-to-one: `author.profile` returns the object or raises `RelatedObjectDoesNotExist`.

### Self-referencing relations

A model may point at itself with `ForeignKey("self", ...)`, for example a genre tree with `parent` and `children`.

### String references across apps

Refer to a model by `"app_label.ModelName"` (for example `"orders.Order"`) when it is defined later in the file or in another app. This avoids circular imports.

### Generic relations (overview)

`GenericForeignKey` from `django.contrib.contenttypes` lets one field point at a row in any table, for example comments attached to many kinds of objects. It has no database-level integrity, cannot be joined efficiently, and complicates queries, so prefer explicit foreign keys unless you truly need "any model".

### Composite primary keys (Django 5.2)

`models.CompositePrimaryKey("a_id", "b_id")` defines a primary key made of several columns. Django 5.2 supports it for tables whose natural key is a combination, but with limits: the admin does not support these models, a composite key cannot be the target of a `ForeignKey`, and not every feature works yet. Most applications keep a single surrogate `id` column.

## Basic syntax examples

### ForeignKey

```python
from django.db import models


class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        "bookstore.Author",
        on_delete=models.PROTECT,
        related_name="books",
    )
```

### OneToOneField

```python
class AuthorProfile(models.Model):
    author = models.OneToOneField(
        "bookstore.Author",
        on_delete=models.CASCADE,
        related_name="profile",
    )
    biography = models.TextField(blank=True)
    website = models.URLField(blank=True)
```

### ManyToManyField

```python
class Genre(models.Model):
    name = models.CharField(max_length=100, unique=True)
    parent = models.ForeignKey(
        "self",
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name="children",
    )

    def __str__(self) -> str:
        return self.name


class Book(models.Model):
    # ... other fields ...
    genres = models.ManyToManyField(Genre, related_name="books", blank=True)
```

### Working with relations

```python
fantasy = Genre.objects.create(name="Fantasy")
epic = Genre.objects.create(name="Epic fantasy", parent=fantasy)

book.genres.add(fantasy, epic)
book.genres.remove(epic)
book.genres.set([fantasy])          # replace the whole set
book.genres.clear()

fantasy.books.all()                  # reverse side
fantasy.children.all()               # self-referencing reverse side

author.books.filter(stock__gt=0)     # reverse foreign key
book.author                          # forward access, runs a query once
book.author_id                       # the raw value, no query
```

### A through model

```python
class Order(models.Model):
    customer_email = models.EmailField()
    books = models.ManyToManyField(
        "bookstore.Book",
        through="bookstore.OrderItem",
        related_name="orders",
    )


class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
    book = models.ForeignKey(
        "bookstore.Book", on_delete=models.PROTECT, related_name="order_items"
    )
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=8, decimal_places=2)
```

With a through model, create the links by creating `OrderItem` rows (or by `order.books.add(book, through_defaults={...})`).

### on_delete in practice

```python
from django.db.models import ProtectedError

try:
    book.delete()
except ProtectedError as exc:
    print("Cannot delete: referenced by", len(exc.protected_objects), "order items")
```

### A composite primary key (Django 5.2)

```python
class BookAward(models.Model):
    pk = models.CompositePrimaryKey("book_id", "year")
    book = models.ForeignKey("bookstore.Book", on_delete=models.CASCADE)
    year = models.PositiveSmallIntegerField()
    name = models.CharField(max_length=200)
```

### A generic relation (overview)

```python
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models


class Note(models.Model):
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveBigIntegerField()
    target = GenericForeignKey("content_type", "object_id")
    text = models.TextField()
```

## Deeper behavior

### What the database creates

For `Book.author` the table gets an `author_id bigint NOT NULL REFERENCES bookstore_author (id)` column and an index on it. A `ManyToManyField` without `through` creates a table named `bookstore_book_genres` with `book_id` and `genre_id` columns and a unique pair constraint. Because Django indexes foreign keys automatically, you rarely add `db_index=True` to them.

### Forward access costs a query

`book.author` loads the author on first access and caches it on the instance. Looping over a hundred books and printing `book.author.name` runs a hundred queries. This is the N+1 problem, solved with `select_related` in note 2.7. Use `book.author_id` when you only need the key.

### Choosing on_delete

Ask what the relationship means.

- Dependent data that has no meaning alone (order items without an order): `CASCADE`.
- Historical or financial records that must survive (order items referencing a book): `PROTECT`.
- Optional references that may disappear (a comment's moderator): `SET_NULL`.
- `CASCADE` is the most dangerous choice on a relation to a widely referenced row, because one deletion can remove thousands of rows. Think through the whole cascade chain before choosing it.

### PROTECT versus RESTRICT

Both refuse the delete. `RESTRICT` allows it if the protected rows would be deleted anyway by another `CASCADE` path in the same operation. This matters in graphs where the same rows are reachable by two routes.

### Many-to-many and through models

A plain `ManyToManyField` has automatic helper methods (`add`, `remove`, `set`, `clear`). With a custom `through` model Django still allows `add`, `set`, and `create` if you provide `through_defaults` for the extra required fields. If you store extra data, ordering, or history, create the through rows yourself, usually with `bulk_create`.

### Self-referencing many-to-many

A `ManyToManyField("self")` is symmetrical by default, meaning if A is related to B then B is related to A. Pass `symmetrical=False` for directional relations such as "follows".

### The relation attribute is a descriptor

`book.author = other_author` assigns the foreign key but does not save it. Call `book.save()` or `book.save(update_fields=["author"])`. Assigning an unsaved object raises an error when you save.

### Reverse one-to-one access

`author.profile` raises an exception when the profile does not exist, instead of returning `None`. Catch `ObjectDoesNotExist` or check `hasattr(author, "profile")` carefully, remembering that `hasattr` swallows the exception.

### limit_choices_to and to_field

`limit_choices_to=Q(...)` restricts the choices in forms and the admin but does not enforce anything at the database level. `to_field="isbn"` makes a foreign key point at a unique non-primary column. Use these sparingly.

### String references and circular imports

If `orders.Order` needs `catalog.Product` and `catalog` also needs something from `orders`, use string references in the fields and import models lazily with `apps.get_model("orders", "Order")` in functions. Keep app dependencies pointing in one direction.

### Generic relations in practice

Joining across a `GenericForeignKey` in a query is not possible without extra work, there is no foreign key constraint, and orphaned rows are easy to create. If an object needs comments from three models, three nullable foreign keys, or three small comment tables, are often easier to reason about.

### Composite keys and their limits

With `CompositePrimaryKey`, a model has no separate `id` column, `pk` returns a tuple, lookups use `pk=(1, 2)`, and migrations changing between a single and a composite key are not supported automatically. Treat it as a feature for tables that map an existing composite key, not as a new default.

## Industry-level snippet

An order creation service on top of the relationship models. The through model records the quantity and a snapshot of each book's price, so later price changes never rewrite history. Order items cascade with their order but protect their book, so a book that was ever sold cannot be deleted. The service validates its input and reports problems with specific exceptions. (Stock handling and locking arrive in note 2.9.)

Add to `apps/bookstore/models.py`:

```python
class Genre(models.Model):
    name = models.CharField(max_length=100, unique=True)
    parent = models.ForeignKey(
        "self",
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name="children",
    )

    def __str__(self) -> str:
        return self.name


class AuthorProfile(models.Model):
    author = models.OneToOneField(
        Author, on_delete=models.CASCADE, related_name="profile"
    )
    biography = models.TextField(blank=True)
    website = models.URLField(blank=True)

    def __str__(self) -> str:
        return f"Profile of {self.author_id}"


class Review(models.Model):
    class Rating(models.IntegerChoices):
        ONE = 1, "One star"
        TWO = 2, "Two stars"
        THREE = 3, "Three stars"
        FOUR = 4, "Four stars"
        FIVE = 5, "Five stars"

    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name="reviews")
    reviewer_name = models.CharField(max_length=100)
    rating = models.PositiveSmallIntegerField(choices=Rating)
    body = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self) -> str:
        return f"{self.reviewer_name}: {self.rating}/5 for {self.book_id}"


class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = "pending", "Pending"
        PAID = "paid", "Paid"
        SHIPPED = "shipped", "Shipped"
        CANCELLED = "cancelled", "Cancelled"

    customer_email = models.EmailField()
    status = models.CharField(
        max_length=10, choices=Status, default=Status.PENDING
    )
    created_at = models.DateTimeField(auto_now_add=True)
    books = models.ManyToManyField(Book, through="OrderItem", related_name="orders")

    def __str__(self) -> str:
        return f"Order {self.pk} for {self.customer_email}"


class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
    book = models.ForeignKey(
        Book, on_delete=models.PROTECT, related_name="order_items"
    )
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=8, decimal_places=2)

    def __str__(self) -> str:
        return f"{self.quantity} x book {self.book_id}"
```

Also add `genres = models.ManyToManyField(Genre, related_name="books", blank=True)` to `Book` (place `Genre` above `Book`, or use the string `"bookstore.Genre"`), then run `makemigrations`.

`apps/bookstore/services/orders.py`:

```python
"""Order creation."""

import logging
from collections.abc import Sequence
from dataclasses import dataclass
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.core.validators import validate_email
from django.db import transaction

from apps.bookstore.models import Book, Order, OrderItem

logger = logging.getLogger(__name__)


class OrderError(ValueError):
    """The order request is invalid."""


class UnknownBookError(OrderError):
    """The order refers to a book that does not exist."""


@dataclass(frozen=True)
class OrderLine:
    book_id: int
    quantity: int


def create_order(*, customer_email: str, lines: Sequence[OrderLine]) -> Order:
    """Create an order whose items snapshot each book's current price."""
    email = customer_email.strip().lower()
    try:
        validate_email(email)
    except ValidationError as exc:
        raise OrderError(f"Invalid customer email: {customer_email!r}") from exc

    if not lines:
        raise OrderError("An order needs at least one line")

    quantities: dict[int, int] = {}
    for line in lines:
        if line.quantity <= 0:
            raise OrderError(f"Quantity for book {line.book_id} must be positive")
        quantities[line.book_id] = quantities.get(line.book_id, 0) + line.quantity

    books = Book.objects.in_bulk(quantities.keys())
    missing = sorted(set(quantities) - set(books))
    if missing:
        raise UnknownBookError(f"Unknown books: {missing}")

    with transaction.atomic():
        order = Order.objects.create(customer_email=email)
        OrderItem.objects.bulk_create(
            [
                OrderItem(
                    order=order,
                    book=books[book_id],
                    quantity=quantity,
                    unit_price=books[book_id].price,
                )
                for book_id, quantity in quantities.items()
            ]
        )

    logger.info("Created order %s with %d items", order.pk, len(quantities))
    return order


def calculate_total(order: Order) -> Decimal:
    """Sum quantity times the snapshotted unit price over the order's items."""
    return sum(
        (item.quantity * item.unit_price for item in order.items.all()),
        start=Decimal("0"),
    )
```

`apps/bookstore/tests/test_relationships.py`:

```python
from decimal import Decimal

from django.core.exceptions import ObjectDoesNotExist
from django.db.models import ProtectedError
from django.test import TestCase

from apps.bookstore.models import (
    AuthorProfile,
    Book,
    Genre,
    Order,
    OrderItem,
    Review,
)
from apps.bookstore.services.orders import (
    OrderError,
    OrderLine,
    UnknownBookError,
    calculate_total,
    create_order,
)

from .factories import make_author, make_book


class CreateOrderTests(TestCase):
    def test_items_snapshot_the_price_at_purchase_time(self) -> None:
        book = make_book(price=Decimal("10.00"))
        order = create_order(
            customer_email="Ada@Example.com",
            lines=[OrderLine(book.pk, 2)],
        )
        book.price = Decimal("99.00")
        book.save(update_fields=["price"])

        self.assertEqual(order.customer_email, "ada@example.com")
        self.assertEqual(calculate_total(order), Decimal("20.00"))

    def test_duplicate_lines_are_merged(self) -> None:
        book = make_book()
        order = create_order(
            customer_email="a@example.com",
            lines=[OrderLine(book.pk, 1), OrderLine(book.pk, 2)],
        )
        item = order.items.get()
        self.assertEqual(item.quantity, 3)

    def test_invalid_requests_are_rejected_before_writing(self) -> None:
        book = make_book()
        with self.assertRaises(OrderError):
            create_order(customer_email="not-an-email", lines=[OrderLine(book.pk, 1)])
        with self.assertRaises(OrderError):
            create_order(customer_email="a@example.com", lines=[])
        with self.assertRaises(OrderError):
            create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 0)])
        self.assertEqual(Order.objects.count(), 0)

    def test_unknown_book_is_reported(self) -> None:
        with self.assertRaises(UnknownBookError):
            create_order(customer_email="a@example.com", lines=[OrderLine(999_999, 1)])
        self.assertEqual(Order.objects.count(), 0)


class RelationshipBehaviorTests(TestCase):
    def test_deleting_an_order_deletes_its_items(self) -> None:
        book = make_book()
        order = create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        order.delete()
        self.assertEqual(OrderItem.objects.count(), 0)
        self.assertTrue(Book.objects.filter(pk=book.pk).exists())

    def test_a_book_that_was_sold_cannot_be_deleted(self) -> None:
        book = make_book()
        create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        with self.assertRaises(ProtectedError):
            book.delete()

    def test_reverse_accessors_follow_related_names(self) -> None:
        author = make_author()
        book = make_book(author=author)
        order = create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        Review.objects.create(
            book=book, reviewer_name="Ada", rating=Review.Rating.FIVE
        )
        self.assertEqual(author.books.count(), 1)
        self.assertEqual(book.reviews.count(), 1)
        self.assertEqual(list(order.books.all()), [book])
        self.assertEqual(list(book.orders.all()), [order])
        self.assertEqual(book.order_items.get().quantity, 1)

    def test_many_to_many_genres_and_the_genre_tree(self) -> None:
        fantasy = Genre.objects.create(name="Fantasy")
        epic = Genre.objects.create(name="Epic fantasy", parent=fantasy)
        book = make_book()
        book.genres.add(fantasy, epic)
        self.assertEqual(set(book.genres.all()), {fantasy, epic})
        self.assertEqual(list(fantasy.children.all()), [epic])
        self.assertEqual(list(epic.books.all()), [book])

    def test_missing_reverse_one_to_one_raises(self) -> None:
        author = make_author()
        with self.assertRaises(ObjectDoesNotExist):
            author.profile
        AuthorProfile.objects.create(author=author, biography="Writer")
        author.refresh_from_db()
        self.assertEqual(author.profile.biography, "Writer")
```

Two details to notice in the tests: the order total is computed from the snapshotted `unit_price`, not the book's current price, and deleting a book fails because of the `PROTECT` rule on `OrderItem.book`.

## Common mistakes

- Choosing `CASCADE` everywhere and letting one deletion silently remove important history.
- Forgetting that `on_delete` is enforced by Django, not the database, so raw SQL deletes bypass it.
- Leaving out `related_name` and then using confusing `_set` accessors, or clashing names when two foreign keys point at the same model.
- Looping over objects and reading `obj.related.field`, causing N+1 queries.
- Forgetting `through_defaults` or trying `add()` on a through relation that has required extra fields.
- Storing a price on the product and computing old order totals from the current price instead of snapshotting it.
- Assuming `hasattr(author, "profile")` is a reliable existence check without understanding that it swallows the exception.
- Using generic relations when a plain foreign key would do.
- Creating circular imports by importing models from each other at module level instead of using string references.
- Assigning a related object and forgetting to `save()`.
- Adopting a composite primary key for new tables without checking admin, foreign key, and migration limits.

## Practice exercises

1. Add `Publisher` to the bookstore with a foreign key from `Book` using `SET_NULL`, and write a test that deleting a publisher leaves the books with no publisher.
2. Add `Series` and a `series_position` to `Book`. Explain which `on_delete` you chose and test the behavior.
3. Create a `Follow` self-referencing many-to-many relation between authors that is not symmetrical, and write a test showing that `a.following.add(b)` does not make `b` follow `a`.
4. Write a function `books_by_genre_tree(genre: Genre) -> list[Book]` that returns the books of a genre and all of its descendants, and note how many queries it needs. Improve it if you can.
5. Use `sqlmigrate` on the migration that created the order tables. Identify the foreign key constraints, the indexes Django created, and the through table.
6. Write a test proving that `RESTRICT` allows a delete when the restricted rows are also removed by a cascade through another path, using a small scratch model graph.
7. Explain, with an example from the bookstore, a case where a generic relation would be tempting and the explicit-foreign-key alternative you would use instead.

## Summary

`ForeignKey`, `OneToOneField`, and `ManyToManyField` express the connections between models, and Django adds indexes, constraints, and reverse accessors for you. `related_name` makes reverse access readable, and a `through` model stores data about the relationship itself, such as a quantity and a price snapshot. Choose `on_delete` by the meaning of the data: `CASCADE` for dependent rows, `PROTECT` for history, `SET_NULL` for optional references. Remember that forward access costs a query, that deletion rules live in Django rather than in the database, and that generic relations and composite keys trade integrity and convenience for flexibility. Prefer explicit, enforced relationships.
