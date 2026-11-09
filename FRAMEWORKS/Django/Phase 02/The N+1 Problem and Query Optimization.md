## Overview

The most common performance problem in Django applications is not a slow query; it is a fast query run hundreds of times. Because relations load lazily, code that looks harmless, such as printing each book's author name in a loop, quietly issues one query per row. This is the N+1 problem: one query to fetch N rows, plus N more to fetch something related to each. This note explains how it arises, how `select_related` and `prefetch_related` fix it, how to control exactly which rows and columns are loaded, how to measure queries in development and in tests, and how to read a query plan. It ends with a tested order-summary selector whose query count stays constant no matter how much data there is.

## Core concepts

### Lazy relation access

When you access `book.author`, Django runs a query for the author the first time. The relation is not loaded when the book is. In a loop over N books, that is N extra queries.

### select_related

`select_related("author")` joins the related table into the original query, so books and authors arrive together in one query. It works for relations where each row has at most one related row: `ForeignKey` and `OneToOneField` (forward, and reverse one-to-one). It can follow chains with double underscores: `select_related("book__author")`.

### prefetch_related

`prefetch_related("items")` runs a second query for the related rows using an `IN` clause (`WHERE order_id IN (...)`) and then matches them to their parents in Python. It works for relations with many related rows: `ManyToManyField` and reverse foreign keys. It can also be used for forward foreign keys, at the cost of an extra query.

### Prefetch objects

`Prefetch("items", queryset=..., to_attr=...)` customizes the second query: filter or order the related rows, apply `select_related` to them, or store the results under a different attribute name.

### only and defer

- `only("title", "price")` loads just those columns (and the primary key).
- `defer("body")` loads everything except the named columns.

Accessing a deferred column later triggers one extra query per object.

### values and values_list

`values("title", "price")` returns dictionaries and `values_list("title", flat=True)` returns plain values. No model instances are built, which is faster and lighter, but you lose methods and properties.

### iterator

`iterator(chunk_size=2000)` reads results in chunks without building the full result cache, keeping memory use low for large exports and batch jobs.

### Counting queries

- `connection.queries` records executed queries when `DEBUG` is true.
- `CaptureQueriesContext` captures them for a block of code.
- `assertNumQueries(n)` fails a test when the count is not exactly `n`.
- `django-debug-toolbar` shows the queries behind a page and flags duplicates.

### QuerySet.explain()

`queryset.explain()` returns the database's execution plan for the query, and `explain(analyze=True)` on PostgreSQL also runs it and reports real timings.

### Accidental quadratic behavior

Hidden loops cause cost to grow with the square of the data: a template that loops over orders and, inside, calls `order.items.count`, or a serializer method that queries per object. They are invisible in small development databases and painful in production.

## Basic syntax examples

### The N+1 pattern

```python
from apps.bookstore.models import Book

for book in Book.objects.all():          # 1 query
    print(book.title, book.author.name)  # +1 query per book
```

With 100 books that is 101 queries.

### The fix with select_related

```python
for book in Book.objects.select_related("author"):  # 1 query with a JOIN
    print(book.title, book.author.name)             # no extra queries
```

### The fix with prefetch_related

```python
from apps.bookstore.models import Order

for order in Order.objects.prefetch_related("items"):   # 2 queries in total
    for item in order.items.all():                      # uses the prefetched rows
        print(order.pk, item.quantity)
```

### Chaining through relations

```python
Order.objects.prefetch_related("items__book__author")
OrderItem.objects.select_related("book__author")
```

### Prefetch with a custom queryset

```python
from django.db.models import Prefetch

from apps.bookstore.models import Order, OrderItem

items = OrderItem.objects.select_related("book__author").order_by("id")
orders = Order.objects.prefetch_related(Prefetch("items", queryset=items))
```

### Filtering a prefetch

```python
big_items = OrderItem.objects.filter(quantity__gte=5)
orders = Order.objects.prefetch_related(
    Prefetch("items", queryset=big_items, to_attr="big_items")
)
# order.big_items is a plain list of the filtered items
```

### Loading fewer columns

```python
Book.objects.only("title", "price")
Book.objects.defer("created_at")
Book.objects.values("title", "price")
Book.objects.values_list("title", flat=True)
```

### Streaming a large result

```python
for title in Book.objects.values_list("title", flat=True).iterator(chunk_size=2000):
    ...
```

### Counting queries in a shell or test

```python
from django.db import connection
from django.test.utils import CaptureQueriesContext

with CaptureQueriesContext(connection) as context:
    list(Book.objects.select_related("author"))

print(len(context))
for query in context.captured_queries:
    print(query["sql"])
```

### Reading a plan

```python
print(Book.objects.filter(author_id=1).explain())
print(Book.objects.filter(author_id=1).explain(analyze=True))  # PostgreSQL
```

## Deeper behavior

### select_related is a JOIN

`select_related` makes the query wider and may return duplicate parent data in each row, but it is a single round trip. For nullable foreign keys it uses a `LEFT JOIN` so rows without a relation are not lost. Because it follows only single-valued relations, using it for a reverse foreign key has no effect and Django raises an error for names it cannot follow.

### prefetch_related is two queries and a Python join

Django fetches the parents, collects their ids, runs `WHERE parent_id IN (...)` for the children, and attaches them in memory. This avoids the row multiplication a join would cause for many-valued relations. It also means the related rows are loaded entirely in memory, so avoid prefetching huge related sets.

### Filtering a prefetched relation throws the cache away

```python
order.items.all()                   # uses the prefetched rows
order.items.filter(quantity__gt=1)  # ignores the prefetch and runs a new query
```

Any method that changes the query (`filter`, `order_by`, `count` on a filtered manager) bypasses the prefetch cache. Put the filtering inside a `Prefetch` object instead, optionally with `to_attr`.

### Prefetch and slicing

You can slice the parent queryset and still prefetch: `Order.objects.prefetch_related("items")[:50]` fetches 50 orders and then their items in one more query. Slicing the related queryset inside a `Prefetch` is limited in older versions, so use window functions or per-parent limits carefully and test the result.

### The only/defer trap

Deferring a column turns it into a lazy attribute. If you later touch it for every row, you have traded one wide query for N narrow ones. Use `only` and `defer` for columns you are sure you will not touch, such as large text bodies in list views. When combining with `select_related`, include the relation field in `only` (`only("title", "author__name")`) or Django raises an error because the join needs it.

### values versus model instances

`values()` and `values_list()` skip model construction, so they are the best fit for reports, exports, and autocomplete endpoints. They bypass model methods, properties, and custom field conversions beyond the basic ones, so do not use them if the template or serializer needs the object.

### iterator and the result cache

`iterator()` does not populate the result cache, so iterating the queryset again runs the query again. It keeps memory flat, and with `prefetch_related` it needs a `chunk_size` (Django 4.1 and later) so that prefetching happens per chunk. On PostgreSQL it uses server-side cursors by default, which does not work behind PgBouncer in transaction pooling mode (note 2.12). Set `DISABLE_SERVER_SIDE_CURSORS` in that case.

### Count, exists, and len

`qs.count()` runs `COUNT(*)`. `len(qs)` loads all rows. In templates, `{{ items|length }}` on an evaluated queryset uses the cache, while `{{ order.items.count }}` runs a query each time unless the result is already cached on the queryset. Compute counts once in the view or with `annotate(Count(...))` (note 2.8).

### Finding N+1 in practice

- In development, keep `django-debug-toolbar` open and look at the number of queries and the "duplicate" markers.
- In tests, wrap the code in `assertNumQueries` using a fixed expected number, and create enough rows (at least three or four) so that an N+1 shows up as a different count.
- In production, tracing tools (Phase 11) and slow query logs reveal repeated statements.

### Plans and indexes

`explain()` shows whether a filter uses an index (`Index Scan`) or reads the whole table (`Seq Scan`), which join strategy was chosen, and how many rows the planner expected versus found. Large gaps between expected and actual rows suggest stale statistics (`ANALYZE`) or a missing index.

## Industry-level snippet

An order summary selector for an admin report or an API list. It fetches orders, their items, and each item's book and author in exactly two queries, builds immutable summary objects, validates its input, and exposes a streaming helper for large exports. Tests lock in the query count and compare it with the naive approach.

`apps/bookstore/selectors_orders.py`:

```python
"""Query-efficient order reporting."""

import logging
from collections.abc import Iterator
from dataclasses import dataclass
from decimal import Decimal

from django.db.models import Prefetch

from apps.bookstore.models import Book, Order, OrderItem

logger = logging.getLogger(__name__)

MAX_LIMIT = 500


@dataclass(frozen=True)
class OrderLineSummary:
    title: str
    author: str
    quantity: int
    line_total: Decimal


@dataclass(frozen=True)
class OrderSummary:
    order_id: int
    customer_email: str
    lines: tuple[OrderLineSummary, ...]
    total: Decimal


def order_summaries(*, limit: int = 50) -> list[OrderSummary]:
    """Return the newest ``limit`` orders with their lines, in two queries."""
    if not 1 <= limit <= MAX_LIMIT:
        raise ValueError(f"limit must be between 1 and {MAX_LIMIT}")

    items = OrderItem.objects.select_related("book__author").order_by("id")
    orders = Order.objects.order_by("-created_at", "-id").prefetch_related(
        Prefetch("items", queryset=items)
    )[:limit]

    summaries: list[OrderSummary] = []
    for order in orders:
        lines = tuple(
            OrderLineSummary(
                title=item.book.title,
                author=item.book.author.name,
                quantity=item.quantity,
                line_total=item.quantity * item.unit_price,
            )
            for item in order.items.all()
        )
        total = sum((line.line_total for line in lines), start=Decimal("0"))
        summaries.append(OrderSummary(order.pk, order.customer_email, lines, total))

    logger.debug("Built %d order summaries", len(summaries))
    return summaries


def iter_book_titles(*, chunk_size: int = 1000) -> Iterator[str]:
    """Stream every book title without loading the table into memory."""
    if chunk_size <= 0:
        raise ValueError("chunk_size must be positive")
    queryset = Book.objects.order_by("pk").values_list("title", flat=True)
    yield from queryset.iterator(chunk_size=chunk_size)
```

`apps/bookstore/tests/test_query_counts.py`:

```python
from decimal import Decimal

from django.db import connection
from django.test import TestCase
from django.test.utils import CaptureQueriesContext

from apps.bookstore.models import Book, Order
from apps.bookstore.selectors_orders import iter_book_titles, order_summaries
from apps.bookstore.services.orders import OrderLine, create_order

from .factories import make_book


class OrderSummaryTests(TestCase):
    def setUp(self) -> None:
        self.books = [
            make_book(title=f"Book {index}", price=Decimal("10.00"))
            for index in range(3)
        ]
        for index in range(5):
            create_order(
                customer_email=f"c{index}@example.com",
                lines=[OrderLine(book.pk, index + 1) for book in self.books],
            )

    def test_query_count_does_not_grow_with_the_data(self) -> None:
        with self.assertNumQueries(2):  # one for orders, one for items with books
            summaries = order_summaries()
        self.assertEqual(len(summaries), 5)

    def test_limit_keeps_the_query_count_constant(self) -> None:
        with self.assertNumQueries(2):
            summaries = order_summaries(limit=2)
        self.assertEqual(len(summaries), 2)

    def test_newest_order_comes_first_with_correct_totals(self) -> None:
        newest = order_summaries()[0]
        self.assertEqual(newest.customer_email, "c4@example.com")
        self.assertEqual(newest.total, Decimal("150.00"))  # 3 books x 5 units x 10.00
        self.assertEqual(len(newest.lines), 3)

    def test_naive_loop_issues_one_query_per_relation_access(self) -> None:
        with CaptureQueriesContext(connection) as context:
            for order in Order.objects.all():          # 1 query
                for item in order.items.all():         # 5 queries (one per order)
                    _ = item.book.author.name          # 15 + 15 queries
        self.assertEqual(len(context), 36)

    def test_invalid_limit_is_rejected(self) -> None:
        for limit in (0, 501):
            with self.subTest(limit=limit):
                with self.assertRaises(ValueError):
                    order_summaries(limit=limit)


class BookLoadingTests(TestCase):
    def test_deferred_column_costs_one_extra_query(self) -> None:
        book = make_book(price=Decimal("12.00"))
        loaded = Book.objects.only("title").get(pk=book.pk)
        with self.assertNumQueries(1):
            self.assertEqual(loaded.price, Decimal("12.00"))

    def test_titles_stream_in_primary_key_order_with_one_query(self) -> None:
        make_book(title="First")
        make_book(title="Second")
        with self.assertNumQueries(1):
            titles = list(iter_book_titles(chunk_size=1))
        self.assertEqual(titles, ["First", "Second"])

    def test_chunk_size_must_be_positive(self) -> None:
        with self.assertRaises(ValueError):
            list(iter_book_titles(chunk_size=0))
```

The pair of tests, a constant two queries for the selector and thirty-six for the naive loop on the same data, is the clearest way to see the problem and prove it is fixed.

## Common mistakes

- Accessing related objects in a loop, template, or serializer without `select_related` or `prefetch_related`.
- Using `select_related` for many-valued relations, or `prefetch_related` where a join would be simpler for a single-valued relation.
- Filtering a prefetched relation with `.filter()` and silently running a new query per parent.
- Overusing `only` and `defer`, then touching the deferred columns for every row.
- Calling `.count()` or `.exists()` on a relation inside a loop.
- Testing with one or two rows, which hides N+1 patterns. Use enough rows that the query count visibly depends on the data.
- Loading huge tables into memory with `list()` instead of streaming with `iterator()`.
- Using `len(qs)` or `if qs:` where `count()` or `exists()` would do.
- Optimizing without measuring, adding indexes or caching before looking at the query count and the plan.
- Prefetching enormous related sets and exhausting memory.

## Practice exercises

1. Write a view that lists all reviews with the book title and author name. Count its queries with `CaptureQueriesContext`, then fix it and show the new count.
2. Write a test using `assertNumQueries` that fails if someone removes `select_related` from a list view, creating enough rows to make the difference obvious.
3. Add a prefetch of `book.genres` to the book list, then show with a query count that `book.genres.all()` in a template no longer queries per book.
4. Use `Prefetch` with `to_attr` to load only reviews rated four or above for each book, and explain why `book.reviews.filter(rating__gte=4)` in a loop would not use the prefetch.
5. Write an export function that streams all orders as CSV rows using `iterator()` and `values_list`, and measure memory behavior on 100,000 generated rows.
6. Run `explain(analyze=True)` on `Book.objects.filter(author_id=...)` on PostgreSQL with 100,000 rows, then drop the foreign key index (in a scratch database only) and compare the plans.
7. A template loops over orders and, for each order, prints `{{ order.items.count }}` and then loops over `order.items.all`, printing `item.book.price` for each item. Count the queries this produces for ten orders of three items each, find every hidden per-row query, and fix them all in the view.

## Summary

N+1 queries arise because relations load lazily, and they are invisible in small datasets. `select_related` joins single-valued relations into one query, `prefetch_related` runs one extra query per many-valued relation and joins in Python, and `Prefetch` customizes that second query. Load only what you need with `only`, `defer`, `values`, and `values_list`, and stream large results with `iterator`. Filtering a prefetched relation bypasses the prefetch, so put filters inside `Prefetch`. Measure with `CaptureQueriesContext`, `assertNumQueries`, and the debug toolbar, create realistic test volumes, and read the plan with `explain()` before optimizing further.
