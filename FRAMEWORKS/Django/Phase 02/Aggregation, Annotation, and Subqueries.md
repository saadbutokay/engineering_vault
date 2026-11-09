## Overview

Reports, dashboards, rankings, and list pages with counts all need numbers computed over groups of rows: how many books per author, the average rating per book, the top three books per author, which books have no reviews. The database is built to do this work, and doing it in Python means loading far more data than necessary and getting slower as the data grows. Django exposes the database's tools through `aggregate`, `annotate`, conditional aggregates, `Subquery`, `OuterRef`, `Exists`, and window functions. This note explains each, warns about the join multiplication pitfall that produces silently wrong totals, and ends with a tested statistics module that avoids it.

## Core concepts

### aggregate versus annotate

- `aggregate(...)` collapses the whole queryset into a single dictionary of values: the average price over all books.
- `annotate(...)` computes a value for each row (or each group) and attaches it to the objects: the number of books for each author.

### Aggregate functions

`Count`, `Sum`, `Avg`, `Min`, and `Max`, from `django.db.models`. Each accepts a field name or an expression. `Sum` and `Avg` return `None` over an empty set unless you pass `default=`.

### Grouping with values().annotate()

Calling `values("field")` before `annotate()` groups by that field and returns one dictionary per group: the number of books per format.

### Filtered aggregates

`Count("books", filter=Q(books__stock__gt=0))` counts only the matching related rows, which lets one query compute several conditional counts side by side.

### Subquery, OuterRef, and Exists

- `Subquery(queryset)` embeds a query that returns one column and at most one row as a value.
- `OuterRef("field")` refers to a column of the outer query from inside the subquery.
- `Exists(queryset)` is a boolean test for whether the inner query returns any row, and is usually the best way to express "has at least one related row".

### Window functions

A window function computes a value over a set of related rows without collapsing them: row numbers, ranks, running totals, and previous or next values. Use `Window(expression=..., partition_by=..., order_by=...)` with expressions such as `RowNumber`, `Rank`, `DenseRank`, `Lag`, `Lead`, or an aggregate.

### alias

`alias()` defines an expression you can reuse in `filter` and `order_by` without selecting it.

### Avoid Python-side aggregation

A loop that adds up values or counts rows in Python loads every row into memory and sends all of it over the network. The same result from the database costs one query and a few bytes.

### The join multiplication pitfall

Aggregating across two multi-valued relations at once multiplies rows. Joining authors to their books and to the books' reviews yields one row per review, so a `Sum` over book stock counts each book once per review. The totals are wrong, and no error is raised.

## Basic syntax examples

### aggregate

```python
from django.db.models import Avg, Count, Max, Min, Sum

from apps.bookstore.models import Author, Book

Book.objects.aggregate(
    average_price=Avg("price"),
    most_expensive=Max("price"),
    cheapest=Min("price"),
    total_stock=Sum("stock", default=0),
    books=Count("id"),
)
# {'average_price': Decimal('12.50'), 'most_expensive': ..., ...}
```

### annotate per row

```python
authors = Author.objects.annotate(book_count=Count("books"))
for author in authors:
    print(author.name, author.book_count)
```

### Grouping with values().annotate()

```python
Book.objects.values("format").annotate(total=Count("id")).order_by("format")
# <QuerySet [{'format': 'ebook', 'total': 3}, {'format': 'hardcover', 'total': 7}, ...]>
```

### Filtered aggregates

```python
from django.db.models import Q

Author.objects.annotate(
    books_total=Count("books"),
    books_in_stock=Count("books", filter=Q(books__stock__gt=0)),
)
```

### Filtering on an annotation

```python
Author.objects.annotate(book_count=Count("books")).filter(book_count__gte=3)
```

### Exists

```python
from django.db.models import Exists, OuterRef

from apps.bookstore.models import Review

Book.objects.filter(Exists(Review.objects.filter(book=OuterRef("pk"))))      # has reviews
Book.objects.filter(~Exists(Review.objects.filter(book=OuterRef("pk"))))     # no reviews
Book.objects.annotate(has_reviews=Exists(Review.objects.filter(book=OuterRef("pk"))))
```

### Subquery

```python
from django.db.models import Subquery

latest_review = (
    Review.objects.filter(book=OuterRef("pk"))
    .order_by("-created_at", "-id")
    .values("body")[:1]
)
Book.objects.annotate(latest_review=Subquery(latest_review))
```

### Window functions

```python
from django.db.models import F, Window
from django.db.models.functions import RowNumber

ranked = Book.objects.annotate(
    price_rank=Window(
        expression=RowNumber(),
        partition_by=F("author_id"),
        order_by=F("price").desc(),
    )
)
top_two_per_author = ranked.filter(price_rank__lte=2)
```

Filtering directly on a window annotation needs Django 4.2 or newer.

### alias

```python
Book.objects.alias(value=F("price") * F("stock")).filter(value__gt=1000)
```

### Looking at the SQL

```python
print(Author.objects.annotate(book_count=Count("books")).query)
```

```sql
SELECT "bookstore_author"."id", "bookstore_author"."name", COUNT("bookstore_book"."id") AS "book_count"
FROM "bookstore_author"
LEFT OUTER JOIN "bookstore_book" ON ("bookstore_author"."id" = "bookstore_book"."author_id")
GROUP BY "bookstore_author"."id"
ORDER BY "bookstore_author"."name" ASC
```

## Deeper behavior

### How the GROUP BY is chosen

With `annotate()` on a plain queryset, Django groups by the primary key of the model (and selected columns), producing one row per object. With `values("x").annotate(...)`, it groups by `x`. The order matters:

```python
# Group by format, one row per format
Book.objects.values("format").annotate(total=Count("id"))

# Annotate each book, then project columns: one row per book
Book.objects.annotate(total=Count("reviews")).values("format", "total")
```

### Default ordering pollutes groupings

A model's `Meta.ordering` adds an `ORDER BY` on those columns, and the ordering columns are added to the `GROUP BY`, which splits groups unexpectedly. Inside grouped subqueries and `values().annotate()` queries, call `.order_by()` with no arguments to clear the ordering, or order explicitly by what you intend.

### Join multiplication, in detail

```python
# Wrong: both aggregates share the same joined rows
Author.objects.annotate(
    total_stock=Sum("books__stock"),          # counted once per review
    review_count=Count("books__reviews"),
)
```

If a book with stock 10 has three reviews, the join produces three rows for it, and `Sum` adds 10 three times. There are three reliable fixes:

- Count distinct related rows: `Count("books", distinct=True)` works for counts, but not for sums.
- Compute each aggregate in its own subquery, as the industry example does, so no joins multiply.
- Run separate queries and merge the results in Python, which is perfectly acceptable for a small number of groups.

A good habit is to verify any query that aggregates across more than one multi-valued relation against a small hand-computed example.

### NULL handling

`Sum` and `Avg` over no rows return `None`, and `Count` returns 0. Use `Sum("stock", default=0)`, or `Coalesce(..., Value(0))` for subqueries, so templates and arithmetic do not fail on `None`. `Avg` ignores `NULL` values, which is usually what you want.

### Subquery rules

A `Subquery` must return exactly one column, and at most one row per outer row, otherwise the database raises an error. Slice with `[:1]` after ordering to pick "the latest" or "the best". Set `output_field` when Django cannot infer the type, such as when you wrap an aggregate inside the subquery.

### Exists versus count

`Exists` stops at the first matching row, so it is cheaper than counting and comparing to zero. Use it for "has any" and "has none" filters. Filtering through a join (`filter(reviews__isnull=False)`) can return duplicate parent rows and needs `distinct()`, which `Exists` avoids.

### Window functions in detail

- `partition_by` splits rows into groups, and `order_by` orders rows within each group.
- `RowNumber` gives unique positions, `Rank` gives equal positions to ties and leaves gaps, and `DenseRank` gives equal positions to ties without gaps.
- `Lag` and `Lead` read the previous or next row's value, for example to compute day-over-day change.
- An aggregate over a window with an ordering produces a running total: `Window(Sum("amount"), order_by=F("created_at").asc())`.
- Include a unique tiebreaker such as the id in `order_by`, otherwise ties make results nondeterministic.

### Why the database should do the work

Python-side aggregation loads full model instances, builds objects, and sums in a slow interpreter loop. A database aggregate reads the same rows near the data, often using indexes, and returns one number. For very large reports, consider materialized views or pre-aggregated tables, but start with correct SQL from the ORM.

### Aggregates in updates and subqueries

You cannot call `aggregate()` inside `update()`, but you can annotate with a `Subquery` and use that in `update()` through `F`, or use `Subquery` directly in `update(field=Subquery(...))`.

## Industry-level snippet

A statistics module for the bookstore that avoids join multiplication with subqueries, ranks books per author with a window function, finds unreviewed books with `Exists`, picks the latest review per book, and builds a rating histogram. Tests include the multiplication bug itself, so the difference is visible and permanent.

Make sure `Review` has no default `Meta.ordering`, and note the `.order_by()` calls that clear ordering inside grouped subqueries.

`apps/bookstore/selectors_stats.py`:

```python
"""Aggregated reporting queries for the bookstore."""

import logging
from dataclasses import dataclass
from decimal import Decimal

from django.db.models import (
    Avg,
    Count,
    Exists,
    F,
    FloatField,
    IntegerField,
    OuterRef,
    QuerySet,
    Subquery,
    Sum,
    Value,
    Window,
)
from django.db.models.functions import Coalesce, RowNumber

from apps.bookstore.models import Author, Book, Review

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class AuthorStats:
    author_id: int
    name: str
    book_count: int
    total_stock: int
    avg_rating: float | None


def author_stats() -> list[AuthorStats]:
    """Per-author book count, total stock, and average review rating.

    Stock and ratings come from subqueries so the two one-to-many joins
    cannot multiply each other's rows.
    """
    stock_per_author = (
        Book.objects.filter(author=OuterRef("pk"))
        .order_by()  # clear Meta.ordering so it does not split the groups
        .values("author")
        .annotate(total=Sum("stock"))
        .values("total")
    )
    rating_per_author = (
        Review.objects.filter(book__author=OuterRef("pk"))
        .order_by()
        .values("book__author")
        .annotate(average=Avg("rating"))
        .values("average")
    )
    rows = (
        Author.objects.annotate(
            book_count=Count("books"),
            total_stock=Coalesce(
                Subquery(stock_per_author, output_field=IntegerField()), Value(0)
            ),
            avg_rating=Subquery(rating_per_author, output_field=FloatField()),
        )
        .order_by("name", "pk")
        .values("pk", "name", "book_count", "total_stock", "avg_rating")
    )
    return [
        AuthorStats(
            author_id=row["pk"],
            name=row["name"],
            book_count=row["book_count"],
            total_stock=row["total_stock"],
            avg_rating=row["avg_rating"],
        )
        for row in rows
    ]


def top_books_per_author(*, per_author: int = 2) -> list[tuple[str, str, Decimal]]:
    """Return ``(author, title, price)`` for each author's priciest books."""
    if per_author <= 0:
        raise ValueError("per_author must be positive")
    ranked = Book.objects.annotate(
        price_rank=Window(
            expression=RowNumber(),
            partition_by=F("author_id"),
            order_by=[F("price").desc(), F("id").asc()],
        )
    )
    rows = (
        ranked.filter(price_rank__lte=per_author)
        .order_by("author__name", "price_rank")
        .values_list("author__name", "title", "price")
    )
    return [(name, title, price) for name, title, price in rows]


def books_without_reviews() -> QuerySet[Book]:
    return Book.objects.filter(~Exists(Review.objects.filter(book=OuterRef("pk"))))


def latest_review_bodies() -> dict[str, str | None]:
    """Map each book title to the text of its newest review, or ``None``."""
    latest = (
        Review.objects.filter(book=OuterRef("pk"))
        .order_by("-created_at", "-id")
        .values("body")[:1]
    )
    rows = Book.objects.annotate(latest_review=Subquery(latest)).values_list(
        "title", "latest_review"
    )
    return dict(rows)


def rating_distribution(*, book_id: int) -> dict[int, int]:
    """Count reviews per star rating, including ratings with zero reviews."""
    counts = dict(
        Review.objects.filter(book_id=book_id)
        .values_list("rating")
        .annotate(total=Count("id"))
        .order_by("rating")
    )
    return {rating: counts.get(rating, 0) for rating in Review.Rating.values}
```

`apps/bookstore/tests/test_stats.py`:

```python
from decimal import Decimal

from django.db.models import Count, Sum
from django.test import TestCase

from apps.bookstore.models import Author, Book, Review
from apps.bookstore.selectors_stats import (
    AuthorStats,
    author_stats,
    books_without_reviews,
    latest_review_bodies,
    rating_distribution,
    top_books_per_author,
)

from .factories import make_author, make_book


def review(book: Book, rating: int, body: str = "") -> Review:
    return Review.objects.create(book=book, reviewer_name="Ada", rating=rating, body=body)


class AuthorStatsTests(TestCase):
    def setUp(self) -> None:
        self.busy = make_author("Busy Author")
        self.idle = make_author("Idle Author")
        self.reviewed = make_book(author=self.busy, stock=10)
        make_book(author=self.busy, stock=5)
        for rating in (5, 4, 3):
            review(self.reviewed, rating)

    def test_naive_join_inflates_the_total_stock(self) -> None:
        naive = (
            Author.objects.filter(pk=self.busy.pk)
            .annotate(total=Sum("books__stock"), reviews=Count("books__reviews"))
            .values_list("total", flat=True)
            .get()
        )
        self.assertEqual(naive, 35)  # 10 x 3 review rows + 5, not the real 15

    def test_subqueries_give_the_correct_totals(self) -> None:
        stats = {item.name: item for item in author_stats()}
        self.assertEqual(
            stats["Busy Author"],
            AuthorStats(self.busy.pk, "Busy Author", 2, 15, 4.0),
        )

    def test_author_without_books_has_zeros_and_no_rating(self) -> None:
        stats = {item.name: item for item in author_stats()}
        self.assertEqual(
            stats["Idle Author"],
            AuthorStats(self.idle.pk, "Idle Author", 0, 0, None),
        )


class RankingTests(TestCase):
    def test_top_books_per_author_respects_the_limit_and_order(self) -> None:
        first = make_author("Alpha")
        second = make_author("Beta")
        make_book(author=first, title="A-30", price=Decimal("30.00"))
        make_book(author=first, title="A-20", price=Decimal("20.00"))
        make_book(author=first, title="A-10", price=Decimal("10.00"))
        make_book(author=second, title="B-5", price=Decimal("5.00"))

        rows = top_books_per_author(per_author=2)

        self.assertEqual(
            rows,
            [
                ("Alpha", "A-30", Decimal("30.00")),
                ("Alpha", "A-20", Decimal("20.00")),
                ("Beta", "B-5", Decimal("5.00")),
            ],
        )

    def test_per_author_must_be_positive(self) -> None:
        with self.assertRaises(ValueError):
            top_books_per_author(per_author=0)


class ReviewQueryTests(TestCase):
    def test_books_without_reviews(self) -> None:
        reviewed = make_book(title="Reviewed")
        make_book(title="Lonely")
        review(reviewed, 5)
        self.assertEqual(
            list(books_without_reviews().values_list("title", flat=True)), ["Lonely"]
        )

    def test_latest_review_per_book(self) -> None:
        book = make_book(title="Dune")
        make_book(title="Unread")
        review(book, 4, "first")
        review(book, 5, "second")
        self.assertEqual(latest_review_bodies(), {"Dune": "second", "Unread": None})

    def test_rating_distribution_includes_empty_buckets(self) -> None:
        book = make_book()
        for rating in (5, 5, 3):
            review(book, rating)
        self.assertEqual(
            rating_distribution(book_id=book.pk), {1: 0, 2: 0, 3: 1, 4: 0, 5: 2}
        )
```

The first test is the most important one: it keeps the broken query alive as a regression exhibit, showing exactly what the subquery version avoids.

## Common mistakes

- Aggregating across two multi-valued relations in one query and trusting the totals.
- Forgetting that `Meta.ordering` adds columns to `GROUP BY`, and not calling `.order_by()` in grouped subqueries.
- Looping in Python to sum or count rows the database could aggregate.
- Putting `values()` after `annotate()` when grouping was intended, or the reverse, and getting the wrong grain.
- Expecting `Sum` over an empty set to return `0` instead of `None`.
- Using `filter(relation__isnull=False)` for "has related rows" and getting duplicate parents, instead of `Exists`.
- Writing a `Subquery` that returns several rows or columns.
- Using window functions without a unique tiebreaker in `order_by`.
- Filtering on window annotations on Django versions before 4.2.
- Computing counts per row in a template (`{{ obj.related.count }}`) rather than annotating once.
- Not verifying an aggregate query against a small dataset whose answer you computed by hand.

## Practice exercises

1. Write a query that returns each genre with the number of books it contains and the number of those books that are in stock, using a filtered aggregate. Verify the result against test data.
2. Write `average_rating_by_format()` returning a dictionary from book format to average review rating, using `values().annotate()`. Explain what would go wrong if you forgot to clear the default ordering.
3. Reproduce the join multiplication bug with orders, items, and reviews of the same book, then fix it using a subquery. Write a test that fails for the buggy version.
4. Write `authors_with_no_stock()` using `Exists` or `~Exists`, and compare its SQL with a version that uses `filter(books__isnull=True)`.
5. Use a window function to compute each order's running total of item quantities for a customer, ordered by creation time, and test it with three orders.
6. Annotate each book with its price rank within its format (`Rank` or `DenseRank`) and list the books ranked first in each format. Show the effect of ties on `Rank` versus `DenseRank`.
7. Write a function that reports the monthly count of new books using `TruncMonth` and `values().annotate()`, and decide how months with no books should appear in the result.

## Summary

`aggregate` collapses a queryset to one dictionary, `annotate` computes a value per object or per group, and `values().annotate()` groups. Filtered aggregates, `Subquery`, `OuterRef`, `Exists`, and window functions let the database answer reporting questions in one query instead of looping in Python. The main hazards are silent: joining two multi-valued relations multiplies rows and inflates sums, `Meta.ordering` splits groups, and aggregates over empty sets return `None`. Defend against them with subqueries, `.order_by()` in grouped queries, `default=` and `Coalesce`, unique tiebreakers in windows, and tests built on small datasets with hand-computed answers.
