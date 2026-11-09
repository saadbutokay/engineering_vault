## Overview

A web application handles many requests at once. Two customers can try to buy the last copy of a book in the same millisecond, a retry can submit the same payment twice, and a crash can happen halfway through a multi-step change. Transactions and locks are how a database keeps data correct in those situations, and your Django code must use them deliberately. This note covers autocommit, `transaction.atomic`, request-level transactions, savepoints, `transaction.on_commit`, row locking with `select_for_update`, the lost update problem, `F` expressions, optimistic locking, unique constraints as idempotency guards, and isolation levels. It ends with a tested order placement service that decrements stock safely, handles retries, and sends its notification only after the data is committed.

This note adds two fields to the bookstore models: `Book.version` (an integer used for optimistic locking) and `Order.idempotency_key` (a unique key that makes retries safe).

## Core concepts

### Autocommit

By default, Django runs in autocommit mode: every ORM statement is its own transaction, committed immediately. If your code issues three writes and fails after the second, the first two stay. Anything that must succeed or fail as a unit needs an explicit transaction.

### transaction.atomic

`atomic` creates a transaction. Use it as a decorator or a context manager. If the block finishes normally, the transaction commits; if it raises an exception, everything inside it is rolled back.

### ATOMIC_REQUESTS

Setting `ATOMIC_REQUESTS = True` on a database wraps every view in a transaction. It is convenient, and it has costs: transactions stay open while the view renders templates and calls external services, holding locks longer, and a view that catches an exception and returns a normal response still commits partial work unless you handle it. Many teams prefer explicit `atomic` blocks around exactly the operations that need them.

### Savepoints

Nested `atomic` blocks create savepoints inside the outer transaction. An exception in an inner block rolls back only to its savepoint, so the outer code can catch it and continue. This is the correct way to catch a database error inside a transaction.

### transaction.on_commit

`transaction.on_commit(callable)` registers a function to run only after the current transaction commits successfully, and discards it if the transaction rolls back. Use it for side effects that must not happen for data that never made it to the database: sending email, enqueueing a background task, publishing an event.

### Row locking with select_for_update

`select_for_update()` locks the selected rows until the transaction ends, so other transactions that try to lock or modify them wait. It must run inside `atomic` on databases that support it (PostgreSQL, MySQL). SQLite ignores it, which is one more reason to test against PostgreSQL.

### Lost updates and race conditions

A race condition occurs when the outcome depends on the timing of concurrent operations. The classic example is the lost update: two requests read stock 1, both decide there is enough, both subtract, and both orders succeed for one book.

### F expressions as the first defense

`update(stock=F("stock") - 1)` performs the arithmetic inside the database in one statement, so concurrent increments and decrements cannot overwrite each other. Combine it with a condition in the `WHERE` clause (`stock__gte=1`) to also prevent going negative.

### Optimistic locking

Instead of holding locks, keep a `version` column. Read the row and its version, then update with `WHERE id = X AND version = N` and increment the version. If no row was updated, someone else changed it first, and you report a conflict or retry. This suits low-contention data edited by humans, such as forms.

### Unique constraints as idempotency guards

An idempotency key is a client-supplied token for one logical operation. Storing it in a column with a unique constraint lets the database guarantee that the second attempt with the same key cannot create a second order.

### Isolation levels (overview)

- Read committed (the PostgreSQL default): each statement sees only committed data, as of when it starts.
- Repeatable read: a transaction sees a stable snapshot. Concurrent updates to the same rows can fail with serialization errors that you must retry.
- Serializable: transactions behave as if run one after another, at the price of more retries.

Most Django applications run at read committed and use locks, constraints, and `F` expressions for the rare places that need more.

## Basic syntax examples

### atomic as a context manager and decorator

```python
from django.db import transaction
from django.db.models import F

from apps.bookstore.models import Book


def move_stock(from_id: int, to_id: int, quantity: int) -> None:
    with transaction.atomic():
        Book.objects.filter(pk=from_id).update(stock=F("stock") - quantity)
        Book.objects.filter(pk=to_id).update(stock=F("stock") + quantity)


@transaction.atomic
def retire_book(book_id: int) -> None:
    ...
```

### Catching a database error inside a transaction

```python
from django.db import IntegrityError, transaction

from apps.bookstore.models import Book, Order

with transaction.atomic():
    Book.objects.filter(pk=1).update(stock=0)
    try:
        with transaction.atomic():        # a savepoint
            Order.objects.create(customer_email="a@example.com", idempotency_key="k1")
    except IntegrityError:
        pass                              # only the inner block was rolled back
    Book.objects.filter(pk=2).update(stock=0)   # the outer transaction is still usable
```

Without the inner `atomic`, the failed statement breaks the outer transaction and later queries raise `TransactionManagementError`.

### on_commit

```python
from functools import partial

from django.db import transaction


def send_confirmation(order_id: int) -> None:
    ...


with transaction.atomic():
    order = Order.objects.create(customer_email="a@example.com")
    transaction.on_commit(partial(send_confirmation, order.pk))
```

### select_for_update

```python
with transaction.atomic():
    book = Book.objects.select_for_update().get(pk=1)   # other writers wait here
    if book.stock >= 1:
        book.stock -= 1
        book.save(update_fields=["stock"])
```

Variants: `select_for_update(nowait=True)` raises instead of waiting, `select_for_update(skip_locked=True)` skips rows already locked (useful for job queues), and `select_for_update(of=("self",))` locks only the main table when joins are involved.

### The lost update, shown and fixed

```python
# Wrong: read, modify in Python, write. Concurrent requests overwrite each other.
book = Book.objects.get(pk=1)
book.stock = book.stock - 1
book.save()

# Right: one atomic statement with a guard against going negative.
updated = Book.objects.filter(pk=1, stock__gte=1).update(stock=F("stock") - 1)
if not updated:
    raise RuntimeError("Out of stock")
```

### Optimistic locking

```python
updated = Book.objects.filter(pk=book_id, version=expected_version).update(
    price=new_price, version=F("version") + 1
)
if updated == 0:
    ...  # someone else changed the book first
```

### Idempotency with a unique key

```python
try:
    with transaction.atomic():
        order = Order.objects.create(customer_email=email, idempotency_key=key)
except IntegrityError:
    order = Order.objects.get(idempotency_key=key)   # a retry: return the original
```

### Settings

```python
DATABASES = {
    "default": {
        # ...
        "ATOMIC_REQUESTS": False,  # prefer explicit atomic blocks
    }
}
```

## Deeper behavior

### What atomic actually does

`atomic` issues `BEGIN` on entry (or `SAVEPOINT` when nested) and `COMMIT` or `ROLLBACK` on exit. Python state does not roll back: objects you changed in memory keep their new attribute values even if the database rolled back, so reload instances after a failed transaction instead of reusing them.

### Why you must not catch errors inside atomic without a savepoint

When PostgreSQL rejects a statement inside a transaction, the whole transaction is marked as failed, and every later statement raises an error until you roll back. Catching `IntegrityError` around a single statement inside the block hides the problem and then breaks on the next query. Wrap the risky statement in its own `atomic()` so only a savepoint is rolled back, as shown above.

### Keep transactions short

A transaction holds locks and connection resources until it ends. Do not call external services, send email, or do slow computation inside it. Prepare data first, open the transaction for the shortest possible set of writes, and use `on_commit` for side effects.

### on_commit details

- If no transaction is active, the callback runs immediately.
- If the transaction rolls back, the callback never runs.
- Callbacks run in the order they were registered, after the commit.
- With `robust=True` (Django 4.2 and later), an exception in one callback is logged and does not stop the others. Without it, a failing callback raises after the data is already committed, so choose deliberately.
- `on_commit` callbacks are not durable. If the process crashes right after the commit, the side effect is lost. For effects that must not be lost, store a record in the same transaction (the outbox pattern, Phase 9) and process it separately.
- In Django's `TestCase`, which wraps each test in a transaction that never commits, callbacks do not run. Use `self.captureOnCommitCallbacks(execute=True)` to run and inspect them.

### Locks and deadlocks

If transaction A locks book 1 then wants book 2, while transaction B locks book 2 then wants book 1, they wait for each other forever and the database aborts one with a deadlock error. The standard prevention is to lock rows in a consistent order, for example always sorted by primary key, which the example below does. Also be ready to retry the transaction after a deadlock or serialization failure.

### select_for_update details

- It must be used inside `atomic`, otherwise Django raises `TransactionManagementError`.
- The queryset must be evaluated inside the same transaction, so call `list()`, `get()`, or iterate inside the block.
- It locks only the rows returned, and rows that match after you evaluate the queryset are not locked.
- Combining it with joins can lock more rows than you intended, so use `of=("self",)` where needed.

### Optimistic versus pessimistic locking

Pessimistic locking (`select_for_update`) prevents conflicts by making others wait, which is right for high-contention rows like inventory. Optimistic locking allows concurrent work and detects conflicts at write time, which is right when conflicts are rare and holding a lock across user think time is unacceptable.

### Unique constraints as the last line of defense

Application-level checks ("does an order with this key exist?") have a gap between the check and the insert. Only the database can close it, because a unique index is checked atomically. The `IntegrityError` is the signal that another request won, and the correct response is to read and return the winner's result.

### Idempotency in practice

Idempotency keys should be generated by the client for one logical operation and reused on every retry of it. The server stores the key, returns the original result on repeats, and ideally rejects a repeat whose payload differs from the original, since that indicates a client bug. Keys are usually kept for a limited time.

### ATOMIC_REQUESTS and non_atomic_requests

With `ATOMIC_REQUESTS = True`, you can opt a view out with `@transaction.non_atomic_requests`. If you rely on it, remember that exceptions you catch and turn into responses do not roll back the transaction.

### Isolation levels in Django

You can set the level in `DATABASES["default"]["OPTIONS"]["isolation_level"]` for PostgreSQL. Raising it trades throughput for safety and requires retry logic for serialization failures. Start with read committed plus explicit locks and constraints.

## Industry-level snippet

An order placement service that applies everything above. It validates input, makes retries safe with an idempotency key, locks the affected books in primary key order, checks and decrements stock atomically, snapshots prices, rolls back completely if any line fails, and notifies the customer only after the commit. A separate function demonstrates optimistic locking for price edits.

Add to `apps/bookstore/models.py`:

```python
# in Book
version = models.PositiveIntegerField(default=1)

# in Order
idempotency_key = models.CharField(max_length=64, null=True, blank=True, unique=True)
```

(`null=True` on a unique string field is deliberate here: `NULL` values never collide in a unique index, while empty strings would.)

`apps/bookstore/services/checkout.py`:

```python
"""Order placement with stock control, idempotency, and safe notifications."""

import logging
from collections.abc import Sequence
from decimal import Decimal
from functools import partial

from django.core.exceptions import ValidationError
from django.core.validators import validate_email
from django.db import IntegrityError, transaction
from django.db.models import F

from apps.bookstore.models import Book, Order, OrderItem
from apps.bookstore.services.orders import OrderError, OrderLine, UnknownBookError

logger = logging.getLogger(__name__)

MAX_IDEMPOTENCY_KEY_LENGTH = 64


class InsufficientStockError(OrderError):
    def __init__(self, book_id: int, requested: int, available: int) -> None:
        super().__init__(
            f"Book {book_id}: requested {requested}, only {available} in stock"
        )
        self.book_id = book_id
        self.requested = requested
        self.available = available


class StaleVersionError(Exception):
    """The row changed since it was read."""


def notify_order_placed(order_id: int) -> None:
    """Runs after the commit. Replace with a real email or task enqueue."""
    logger.info("Order %s committed; sending confirmation", order_id)


def _normalize_email(raw: str) -> str:
    email = raw.strip().lower()
    try:
        validate_email(email)
    except ValidationError as exc:
        raise OrderError(f"Invalid customer email: {raw!r}") from exc
    return email


def _merge_lines(lines: Sequence[OrderLine]) -> dict[int, int]:
    if not lines:
        raise OrderError("An order needs at least one line")
    quantities: dict[int, int] = {}
    for line in lines:
        if line.quantity <= 0:
            raise OrderError(f"Quantity for book {line.book_id} must be positive")
        quantities[line.book_id] = quantities.get(line.book_id, 0) + line.quantity
    return quantities


def _get_or_create_order(email: str, key: str | None) -> tuple[Order, bool]:
    """Create the order row, or return the existing order for a repeated key."""
    if key is None:
        return Order.objects.create(customer_email=email), True
    try:
        with transaction.atomic():  # savepoint, so a duplicate does not poison the outer transaction
            return Order.objects.create(customer_email=email, idempotency_key=key), True
    except IntegrityError:
        return Order.objects.get(idempotency_key=key), False


def place_order(
    *,
    customer_email: str,
    lines: Sequence[OrderLine],
    idempotency_key: str | None = None,
) -> Order:
    """Place an order, decrementing stock safely.

    Repeating a call with the same ``idempotency_key`` returns the original
    order without touching stock again.

    Raises:
        OrderError: for invalid input.
        UnknownBookError: if a book does not exist.
        InsufficientStockError: if any line cannot be fulfilled; nothing is saved.
    """
    email = _normalize_email(customer_email)
    quantities = _merge_lines(lines)
    if idempotency_key is not None and not (
        1 <= len(idempotency_key) <= MAX_IDEMPOTENCY_KEY_LENGTH
    ):
        raise OrderError(
            f"idempotency_key must be 1 to {MAX_IDEMPOTENCY_KEY_LENGTH} characters"
        )

    with transaction.atomic():
        order, created = _get_or_create_order(email, idempotency_key)
        if not created:
            logger.info("Replayed idempotent request for order %s", order.pk)
            return order

        # Lock in primary key order so concurrent orders cannot deadlock.
        locked = Book.objects.select_for_update().filter(pk__in=quantities.keys()).order_by("pk")
        books = {book.pk: book for book in locked}
        missing = sorted(set(quantities) - set(books))
        if missing:
            raise UnknownBookError(f"Unknown books: {missing}")

        for book_id in sorted(quantities):
            quantity = quantities[book_id]
            book = books[book_id]
            if book.stock < quantity:
                raise InsufficientStockError(book_id, quantity, book.stock)
            # The WHERE guard is a second line of defense on top of the lock.
            updated = Book.objects.filter(pk=book_id, stock__gte=quantity).update(
                stock=F("stock") - quantity
            )
            if updated == 0:
                raise InsufficientStockError(book_id, quantity, book.stock)

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
        transaction.on_commit(partial(notify_order_placed, order.pk), robust=True)

    logger.info("Placed order %s with %d lines", order.pk, len(quantities))
    return order


def change_price(*, book_id: int, expected_version: int, new_price: Decimal) -> Book:
    """Change a price using optimistic locking.

    Raises:
        ValueError: if the price is negative.
        Book.DoesNotExist: if the book does not exist.
        StaleVersionError: if the book changed since ``expected_version`` was read.
    """
    if new_price < 0:
        raise ValueError("price must not be negative")
    updated = Book.objects.filter(pk=book_id, version=expected_version).update(
        price=new_price, version=F("version") + 1
    )
    if updated == 0:
        if not Book.objects.filter(pk=book_id).exists():
            raise Book.DoesNotExist(f"Book {book_id} does not exist")
        raise StaleVersionError(
            f"Book {book_id} was modified after version {expected_version}"
        )
    return Book.objects.get(pk=book_id)
```

`apps/bookstore/tests/test_checkout.py`:

```python
import threading
from decimal import Decimal
from unittest import skipUnless

from django.db import connection, connections
from django.test import TestCase, TransactionTestCase

from apps.bookstore.models import Book, Order
from apps.bookstore.services.checkout import (
    InsufficientStockError,
    StaleVersionError,
    change_price,
    place_order,
)
from apps.bookstore.services.orders import OrderError, OrderLine, UnknownBookError

from .factories import make_book

LOGGER = "apps.bookstore.services.checkout"


class PlaceOrderTests(TestCase):
    def test_successful_order_decrements_stock_and_snapshots_prices(self) -> None:
        book = make_book(stock=5, price=Decimal("10.00"))
        order = place_order(
            customer_email="Ada@Example.com", lines=[OrderLine(book.pk, 2)]
        )
        book.refresh_from_db()
        self.assertEqual(book.stock, 3)
        self.assertEqual(order.customer_email, "ada@example.com")
        self.assertEqual(order.items.get().unit_price, Decimal("10.00"))

    def test_a_failing_line_rolls_back_the_whole_order(self) -> None:
        plenty = make_book(stock=5)
        scarce = make_book(stock=1)
        with self.assertRaises(InsufficientStockError) as context:
            place_order(
                customer_email="a@example.com",
                lines=[OrderLine(plenty.pk, 1), OrderLine(scarce.pk, 2)],
            )
        self.assertEqual(context.exception.available, 1)
        plenty.refresh_from_db()
        self.assertEqual(plenty.stock, 5)          # the first decrement was undone
        self.assertEqual(Order.objects.count(), 0)

    def test_duplicate_lines_are_merged_before_the_stock_check(self) -> None:
        book = make_book(stock=2)
        with self.assertRaises(InsufficientStockError):
            place_order(
                customer_email="a@example.com",
                lines=[OrderLine(book.pk, 1), OrderLine(book.pk, 2)],
            )

    def test_unknown_book_is_reported_and_nothing_is_saved(self) -> None:
        with self.assertRaises(UnknownBookError):
            place_order(customer_email="a@example.com", lines=[OrderLine(999_999, 1)])
        self.assertEqual(Order.objects.count(), 0)

    def test_input_validation(self) -> None:
        book = make_book(stock=1)
        with self.assertRaises(OrderError):
            place_order(customer_email="bad", lines=[OrderLine(book.pk, 1)])
        with self.assertRaises(OrderError):
            place_order(customer_email="a@example.com", lines=[])
        with self.assertRaises(OrderError):
            place_order(
                customer_email="a@example.com",
                lines=[OrderLine(book.pk, 1)],
                idempotency_key="x" * 65,
            )


class IdempotencyTests(TestCase):
    def test_repeating_a_request_returns_the_same_order_once(self) -> None:
        book = make_book(stock=5)
        lines = [OrderLine(book.pk, 2)]
        first = place_order(customer_email="a@example.com", lines=lines, idempotency_key="k1")
        second = place_order(customer_email="a@example.com", lines=lines, idempotency_key="k1")
        book.refresh_from_db()
        self.assertEqual(first.pk, second.pk)
        self.assertEqual(book.stock, 3)             # decremented only once
        self.assertEqual(Order.objects.count(), 1)

    def test_different_keys_create_different_orders(self) -> None:
        book = make_book(stock=5)
        lines = [OrderLine(book.pk, 1)]
        place_order(customer_email="a@example.com", lines=lines, idempotency_key="k1")
        place_order(customer_email="a@example.com", lines=lines, idempotency_key="k2")
        self.assertEqual(Order.objects.count(), 2)


class OnCommitTests(TestCase):
    def test_notification_runs_only_after_commit(self) -> None:
        book = make_book(stock=1)
        with self.assertLogs(LOGGER, level="INFO") as logs:
            with self.captureOnCommitCallbacks(execute=True) as callbacks:
                place_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        self.assertEqual(len(callbacks), 1)
        self.assertTrue(any("committed" in line for line in logs.output))

    def test_no_notification_when_the_order_fails(self) -> None:
        book = make_book(stock=0)
        with self.captureOnCommitCallbacks(execute=True) as callbacks:
            with self.assertRaises(InsufficientStockError):
                place_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        self.assertEqual(callbacks, [])


class OptimisticLockTests(TestCase):
    def test_matching_version_updates_and_increments(self) -> None:
        book = make_book(price=Decimal("10.00"))
        updated = change_price(book_id=book.pk, expected_version=1, new_price=Decimal("12.00"))
        self.assertEqual((updated.price, updated.version), (Decimal("12.00"), 2))

    def test_stale_version_is_rejected(self) -> None:
        book = make_book()
        change_price(book_id=book.pk, expected_version=1, new_price=Decimal("11.00"))
        with self.assertRaises(StaleVersionError):
            change_price(book_id=book.pk, expected_version=1, new_price=Decimal("13.00"))

    def test_missing_book_and_negative_price(self) -> None:
        with self.assertRaises(Book.DoesNotExist):
            change_price(book_id=999_999, expected_version=1, new_price=Decimal("1"))
        with self.assertRaises(ValueError):
            change_price(book_id=1, expected_version=1, new_price=Decimal("-1"))


@skipUnless(connection.vendor == "postgresql", "row locks need PostgreSQL")
class ConcurrentOrderTests(TransactionTestCase):
    def test_only_one_of_two_simultaneous_orders_gets_the_last_copy(self) -> None:
        book = make_book(stock=1)
        outcomes: list[str] = []
        barrier = threading.Barrier(2)

        def worker(index: int) -> None:
            try:
                barrier.wait(timeout=10)
                place_order(
                    customer_email=f"customer{index}@example.com",
                    lines=[OrderLine(book.pk, 1)],
                )
                outcomes.append("ok")
            except InsufficientStockError:
                outcomes.append("sold_out")
            finally:
                connections.close_all()

        threads = [threading.Thread(target=worker, args=(index,)) for index in range(2)]
        for thread in threads:
            thread.start()
        for thread in threads:
            thread.join()

        self.assertEqual(sorted(outcomes), ["ok", "sold_out"])
        book.refresh_from_db()
        self.assertEqual(book.stock, 0)
```

The last test is what proves the locking works. It cannot run on SQLite, which is why note 0.10 set up PostgreSQL for development.

## Common mistakes

- Doing read-modify-write on a counter in Python instead of using `F` expressions or locks.
- Catching `IntegrityError` or `DatabaseError` inside an `atomic` block without a nested `atomic` savepoint.
- Calling external services, sending email, or doing slow work inside a transaction, holding locks the whole time.
- Sending emails or enqueueing tasks inside the transaction instead of `on_commit`, so they fire for data that rolls back.
- Using `select_for_update` outside `atomic`, or evaluating the queryset after the block ended.
- Locking rows in inconsistent order across code paths, causing deadlocks.
- Testing concurrency only on SQLite, where row locks do nothing.
- Expecting `on_commit` callbacks to run inside a `TestCase` without `captureOnCommitCallbacks`.
- Checking for an existing record and then inserting, without a unique constraint to close the gap.
- Treating an idempotency key as valid for a different payload, instead of rejecting the mismatch.
- Reusing model instances after a rolled-back transaction, whose in-memory values may no longer match the database.
- Turning on `ATOMIC_REQUESTS` and assuming every caught exception rolls back.

## Practice exercises

1. Write a test that shows the lost update with two interleaved read-modify-write cycles on a book's stock, then fix it with a guarded `F` update and show the corrected result.
2. Write `transfer_stock(from_id, to_id, quantity)` that locks both books in primary key order, validates stock, and moves it atomically. Test the rollback when the destination does not exist.
3. Add an idempotency payload check: store a hash of the order lines with the key, and raise a `ConflictError` when a repeated key arrives with different lines. Test both cases.
4. Demonstrate the broken-transaction error: catch an `IntegrityError` inside an `atomic` block without a savepoint, run another query, and record the error. Then fix it with a nested `atomic`.
5. Write a test using `captureOnCommitCallbacks` that proves a callback registered inside a failed `atomic` block is discarded.
6. On PostgreSQL, write a `TransactionTestCase` with two threads that lock the same two books in opposite orders to provoke a deadlock, then change the code to lock in sorted order and show that the deadlock disappears.
7. Implement a job-claiming function for a `Task` model using `select_for_update(skip_locked=True)` so that two workers never claim the same row, and test it with two threads on PostgreSQL.

## Summary

Autocommit makes every statement its own transaction, so multi-step changes need `transaction.atomic` to succeed or fail as a unit. Nested `atomic` blocks provide savepoints, which is the right way to survive a database error inside a transaction. Use `on_commit` for side effects so they happen only for committed data, keep transactions short, and treat `on_commit` as non-durable. Prevent lost updates with `F` expressions, `select_for_update` (locking in a consistent order to avoid deadlocks), or optimistic locking with a version column, and use unique constraints, such as an idempotency key, to close the gap between "check" and "insert". Test concurrency on PostgreSQL, because SQLite hides these problems.
