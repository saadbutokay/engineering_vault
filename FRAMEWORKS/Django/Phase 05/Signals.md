## Overview

Django signals let one part of an application announce that something happened, and let other parts react without the announcer knowing who is listening. A model was saved, a user logged in, a request finished: signals are the mechanism behind many built-in behaviors. They are also one of the most overused features in Django. A signal receiver is code that runs somewhere else, in an order you cannot see, triggered by an action that looks innocent. Used for decoupling independent apps, signals are helpful. Used to hide the main flow of your own application, they make behavior impossible to trace and bugs very hard to find. This note covers the built-in signals, custom signals, `@receiver`, why signals hide control flow, why `bulk_create` and `update` do not send them, using `transaction.on_commit` for side effects, and how to decide between a signal and an explicit function call. It ends with a tested example in which a custom signal, sent after the commit, lets a separate notifications app react to orders without the orders code knowing it exists.

## Core concepts

### What a signal is

A signal is an object that keeps a list of receivers: functions to call when the signal is sent. A sender announces an event with `signal.send(...)`, and Django calls each connected receiver with the arguments. The pattern is the publish-subscribe idea inside one process: the publisher does not know the subscribers.

### Built-in model signals

- `pre_save` and `post_save`: before and after `Model.save()`. `post_save` receives `created` (true for a new row) and `update_fields`.
- `pre_delete` and `post_delete`: around deletion of an instance.
- `m2m_changed`: when rows are added to, removed from, or cleared from a many-to-many relation, with an `action` such as `"post_add"`.
- `pre_init` and `post_init`: around model instantiation (rarely a good idea).

### Other built-in signals

- `request_started` and `request_finished`: around each request.
- `user_logged_in`, `user_logged_out`, and `user_login_failed`: from the authentication system.
- `post_migrate`: after migrations run (used to create permissions).
- `setting_changed`: in tests when `override_settings` changes a setting.

### Receivers and @receiver

A receiver is any function accepting `sender` and `**kwargs`. Connect it with the `@receiver(signal, sender=...)` decorator or with `signal.connect(func)`. Receivers must accept `**kwargs`, because Django may add arguments in the future.

### Where to connect receivers

Put receivers in a module such as `receivers.py` in the app that reacts to the event, and import that module from `AppConfig.ready()`, so they connect once, when Django starts.

### dispatch_uid

If a module is imported under two names, a receiver can be connected twice and run twice. Passing a unique `dispatch_uid` to `connect` or `@receiver` makes the connection idempotent.

### Custom signals

Define your own with `Signal()`, send it from your code, and document its arguments. Custom signals are the legitimate way to let an app announce a domain event (an order was placed) to apps that should not be imported by the sender.

### send and send_robust

`signal.send()` calls the receivers in turn and lets any exception propagate to the sender, so a faulty receiver can break the sending code. `signal.send_robust()` catches each receiver's exception and returns it in the result list, so one bad receiver does not stop the others or the sender. For events announced after the work is done, `send_robust` plus logging is the safer choice.

### Signals and bulk operations

Signals are sent from `save()` and `delete()` on instances. They are not sent by `bulk_create()`, `bulk_update()`, or `QuerySet.update()`. (`QuerySet.delete()` does send `pre_delete` and `post_delete` for each instance when receivers exist, which is why it can be slow.) Code that depends on a `post_save` receiver silently stops working when someone switches to a bulk operation for speed.

### Side effects and transactions

A `post_save` receiver runs inside the transaction of the code that saved the object. If it sends an email or enqueues a job, that happens before the commit, and the work is wasted or wrong if the transaction later rolls back. Use `transaction.on_commit` (note 2.9) for side effects, either in the receiver or, better, at the point that announces the event.

### Why signals hide control flow

When you read `order.save()`, nothing tells you that three receivers in other apps will run. A bug can come from code you have never seen, ordering between receivers is not guaranteed, and the call stack at a breakpoint shows the signal machinery rather than a clear sequence. Explicit calls (`place_order()` calls `send_confirmation()`) are visible, searchable, testable, and ordered.

### When an explicit call is better

Use a plain function call when the behavior is part of the core logic of the operation, when the caller needs the result or must handle the failure, when the order matters, or when both pieces live in the same app. Use a signal when the sender genuinely should not know about the receivers, such as a reusable app announcing events, or when many unrelated apps may react and the sender should not change when one is added.

## Basic syntax examples

### Receiving a built-in signal

```python
from typing import Any

from django.db.models.signals import post_save
from django.dispatch import receiver

from apps.bookstore.models import Review


@receiver(post_save, sender=Review, dispatch_uid="reviews.log_new_review")
def log_new_review(sender: type[Review], instance: Review, created: bool, **kwargs: Any) -> None:
    if created:
        print(f"New review {instance.pk} for book {instance.book_id}")
```

### Connecting without the decorator

```python
post_save.connect(log_new_review, sender=Review, dispatch_uid="reviews.log_new_review")
```

### Registering receivers in AppConfig.ready

`apps/notifications/apps.py`:

```python
from django.apps import AppConfig


class NotificationsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.notifications"

    def ready(self) -> None:
        from apps.notifications import receivers  # noqa: F401  (imported for its side effect)
```

### Reading created and update_fields

```python
@receiver(post_save, sender=Review)
def on_review_saved(sender: type[Review], instance: Review, created: bool, **kwargs: Any) -> None:
    update_fields = kwargs.get("update_fields")       # None, or a frozenset of field names
    if kwargs.get("raw"):                            # True while loading fixtures
        return
    if not created and update_fields is not None and "rating" not in update_fields:
        return                                       # the rating did not change
    ...
```

### m2m_changed

```python
from django.db.models.signals import m2m_changed

from apps.bookstore.models import Book


@receiver(m2m_changed, sender=Book.genres.through)
def on_genres_changed(
    sender: type[Any], instance: Book, action: str, pk_set: set[int] | None, **kwargs: Any
) -> None:
    if action in {"post_add", "post_remove", "post_clear"}:
        ...
```

### Logging a login

```python
from django.contrib.auth.signals import user_logged_in


@receiver(user_logged_in)
def record_login(sender: type[Any], request: HttpRequest, user: Any, **kwargs: Any) -> None:
    logger.info("User %s logged in", user.pk)
```

### A custom signal

```python
from django.dispatch import Signal

order_placed = Signal()   # arguments: order_id


order_placed.send(sender=Order, order_id=42)
results = order_placed.send_robust(sender=Order, order_id=42)   # [(receiver, response_or_exception), ...]
```

### Side effects after the commit

```python
from functools import partial

from django.db import transaction

transaction.on_commit(partial(order_placed.send_robust, sender=Order, order_id=order.pk))
```

### Disconnecting (mostly in tests)

```python
post_save.disconnect(log_new_review, sender=Review, dispatch_uid="reviews.log_new_review")
```

## Deeper behavior

### How sending works

`Signal.send()` loops over the connected receivers synchronously, in the thread that called it, and returns a list of `(receiver, response)` pairs. There is no queue and no isolation: a slow receiver slows the sender, and a receiver that raises (with plain `send`) aborts the sender's operation after some receivers have already run.

### Order is undefined

Receivers run in connection order, which depends on import order and is not a contract. If receiver B must run after receiver A, they are not independent, and the relationship belongs in an explicit function that calls A and then B.

### Weak references

By default `connect` stores a weak reference to the receiver, so a receiver defined as a local function or a lambda can be garbage collected and silently disappear. Module-level functions and methods of long-lived objects are safe. Pass `weak=False` when connecting a closure, as tests often do.

### Receivers that save the same model

A `post_save` receiver that saves the instance again triggers `post_save` again, which can recurse. Guard with a flag, use `update_fields`, or write with `QuerySet.update()`, which does not send signals. Recursion through signals is a classic source of infinite loops and duplicated work.

### raw saves and fixtures

When `loaddata` saves objects it sets `raw=True` on `pre_save` and `post_save`. Receivers that read related objects or create new rows must skip raw saves, because the database may be only partly loaded.

### Signals and transactions

`post_save` is sent immediately after the row is written, inside the surrounding transaction, before it commits. A receiver that enqueues a task for the new row can start a worker that cannot yet see the row. Register `transaction.on_commit` callbacks for anything that leaves the process (email, tasks, webhooks). Announce domain events after the commit, as the example does, so receivers always see committed data and are never run for rolled-back work.

### Testing code that uses signals

Receivers are active in tests, so tests that create objects trigger them. That is correct for integration tests and a nuisance for unit tests. Either let them run and assert on the result, or disconnect them for the test and restore them afterward. Always test the receiver as a plain function too, by calling it directly with the arguments Django would pass.

### Idempotent receivers

Because receivers may run more than once (retries, re-saves, duplicate connections), write them so that running twice has the same effect as running once: `get_or_create` on a unique constraint, or an update that sets a value rather than increments it.

### Cross-app dependencies

The healthy direction is: the notifications app imports the orders app's signal and reacts to it, and the orders app never imports notifications. The dependency arrow points from the listener to the announcer. When both apps must know each other anyway, a signal only hides a real coupling, and a direct call is more honest.

### Debugging signals

To find out what is connected, inspect `signal.receivers` in a shell, search the code for `@receiver` and `.connect(`, and keep all receivers in files named `receivers.py` so there is one place to look. Log in each receiver while developing.

### Alternatives

- A service function that does the work and then calls the other functions explicitly.
- `transaction.on_commit` for side effects of one operation.
- The transactional outbox pattern (Phase 9) for events that must not be lost, because a signal receiver that crashes loses the event.
- Background tasks for slow or retried work.

## Industry-level snippet

A domain event announced after the commit: the order service sends a custom `order_placed` signal, and a separate notifications app records a notification. Receivers are idempotent, a failing receiver is logged without hurting the order, and nothing is sent when the order fails.

`apps/bookstore/signals.py`:

```python
"""Domain events announced by the bookstore app."""

import logging

from django.dispatch import Signal

from apps.bookstore.models import Order

logger = logging.getLogger(__name__)

# Sent after an order has been committed. Argument: ``order_id`` (int).
order_placed = Signal()


def announce_order_placed(order_id: int) -> None:
    """Send ``order_placed`` and log any receiver that fails.

    Uses ``send_robust`` so one broken receiver cannot affect the others or
    turn a committed order into an error for the customer.
    """
    results = order_placed.send_robust(sender=Order, order_id=order_id)
    for receiver, outcome in results:
        if isinstance(outcome, Exception):
            logger.error(
                "Receiver %s failed for order %s",
                getattr(receiver, "__qualname__", repr(receiver)),
                order_id,
                exc_info=outcome,
            )
```

In `apps/bookstore/services/checkout.py` (from note 2.9), replace the `notify_order_placed` function and its use:

```python
from apps.bookstore.signals import announce_order_placed

# inside place_order, after the items are created:
transaction.on_commit(partial(announce_order_placed, order.pk), robust=True)
```

`apps/notifications/models.py`:

```python
from django.db import models


class Notification(models.Model):
    class Kind(models.TextChoices):
        ORDER_PLACED = "order_placed", "Order placed"

    order = models.ForeignKey(
        "bookstore.Order", on_delete=models.CASCADE, related_name="notifications"
    )
    kind = models.CharField(max_length=20, choices=Kind)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["order", "kind"], name="notification_one_per_order_kind"),
        ]

    def __str__(self) -> str:
        return f"{self.kind} for order {self.order_id}"
```

`apps/notifications/receivers.py`:

```python
import logging
from typing import Any

from django.dispatch import receiver

from apps.bookstore.models import Order
from apps.bookstore.signals import order_placed
from apps.notifications.models import Notification

logger = logging.getLogger(__name__)


@receiver(order_placed, dispatch_uid="notifications.record_order_placed")
def record_order_placed(sender: type[Order], order_id: int, **kwargs: Any) -> None:
    """Record that the customer should be told about the order. Idempotent."""
    _, created = Notification.objects.get_or_create(
        order_id=order_id, kind=Notification.Kind.ORDER_PLACED
    )
    logger.info("Notification for order %s %s", order_id, "created" if created else "already existed")
```

Register `apps.notifications` in `INSTALLED_APPS` and add the `apps.py` shown earlier.

`apps/notifications/tests/test_signals.py` (with an empty `__init__.py` in `tests/`):

```python
from typing import Any

from django.db.models.signals import post_save
from django.test import TestCase

from apps.bookstore.models import Order, Review
from apps.bookstore.services.checkout import InsufficientStockError, place_order
from apps.bookstore.services.orders import OrderLine
from apps.bookstore.signals import order_placed
from apps.bookstore.tests.factories import make_book
from apps.notifications.models import Notification
from apps.notifications.receivers import record_order_placed


class OrderPlacedSignalTests(TestCase):
    def place(self, stock: int = 5, quantity: int = 1) -> Order:
        book = make_book(stock=stock)
        return place_order(
            customer_email="a@example.com", lines=[OrderLine(book.pk, quantity)]
        )

    def test_the_notification_is_recorded_only_after_the_commit(self) -> None:
        with self.captureOnCommitCallbacks(execute=False) as callbacks:
            self.place()
        self.assertEqual(Notification.objects.count(), 0)   # nothing ran yet
        for callback in callbacks:
            callback()
        self.assertEqual(Notification.objects.count(), 1)

    def test_a_failed_order_sends_nothing(self) -> None:
        with self.captureOnCommitCallbacks(execute=True) as callbacks:
            with self.assertRaises(InsufficientStockError):
                self.place(stock=0)
        self.assertEqual(callbacks, [])
        self.assertEqual(Notification.objects.count(), 0)

    def test_a_failing_receiver_is_logged_and_does_not_stop_the_others(self) -> None:
        def exploding(sender: object, order_id: int, **kwargs: Any) -> None:
            raise RuntimeError("boom")

        order_placed.connect(exploding, dispatch_uid="test.exploding", weak=False)
        self.addCleanup(order_placed.disconnect, dispatch_uid="test.exploding")

        with self.assertLogs("apps.bookstore.signals", level="ERROR") as logs:
            with self.captureOnCommitCallbacks(execute=True):
                order = self.place()

        self.assertTrue(Order.objects.filter(pk=order.pk).exists())
        self.assertEqual(Notification.objects.count(), 1)    # the healthy receiver still ran
        self.assertTrue(any("exploding" in line for line in logs.output))

    def test_the_receiver_is_idempotent(self) -> None:
        order = self.place()
        record_order_placed(sender=Order, order_id=order.pk)
        record_order_placed(sender=Order, order_id=order.pk)
        self.assertEqual(Notification.objects.filter(order=order).count(), 1)

    def test_dispatch_uid_prevents_duplicate_connections(self) -> None:
        def handler(sender: object, **kwargs: Any) -> None:
            return None

        before = len(order_placed.receivers)
        order_placed.connect(handler, dispatch_uid="test.once", weak=False)
        order_placed.connect(handler, dispatch_uid="test.once", weak=False)
        self.addCleanup(order_placed.disconnect, dispatch_uid="test.once")
        self.assertEqual(len(order_placed.receivers), before + 1)


class BulkOperationTests(TestCase):
    def test_only_instance_saves_send_post_save(self) -> None:
        calls: list[frozenset[str] | None] = []

        def spy(sender: type[Review], instance: Review, created: bool, **kwargs: Any) -> None:
            calls.append(kwargs.get("update_fields"))

        post_save.connect(spy, sender=Review, weak=False, dispatch_uid="test.spy")
        self.addCleanup(post_save.disconnect, sender=Review, dispatch_uid="test.spy")

        book = make_book()
        review = Review.objects.create(book=book, reviewer_name="A", rating=5)
        self.assertEqual(len(calls), 1)

        Review.objects.bulk_create([Review(book=book, reviewer_name="B", rating=4)])
        Review.objects.filter(pk=review.pk).update(rating=3)
        self.assertEqual(len(calls), 1)            # neither bulk_create nor update sent it

        review.body = "changed"
        review.save(update_fields=["body"])
        self.assertEqual(calls[-1], frozenset({"body"}))
```

The receiver is registered in `ready()`, listens to an event owned by a different app, and the orders code never imports the notifications app. Removing the notifications app leaves the orders code untouched. That is the situation where a signal earns its keep.

## Common mistakes

- Putting core business logic in signal receivers, so the main flow cannot be read from the code that triggers it.
- Assuming `bulk_create`, `bulk_update`, or `update()` send `post_save`.
- Sending email or enqueueing jobs in `post_save` before the transaction commits.
- Receivers that save the same model and trigger themselves recursively.
- Forgetting `**kwargs` in a receiver, which breaks when Django adds arguments.
- Connecting receivers at import time in random modules, or twice, without a `dispatch_uid`.
- Defining a receiver as a local function or lambda with the default weak reference, so it vanishes.
- Using `send` for post-commit announcements, so one failing receiver turns a successful operation into an error.
- Ignoring `raw=True` and breaking `loaddata`.
- Depending on the order in which receivers run.
- Letting two apps import each other through signals, creating a hidden circular dependency.
- Writing receivers that are not idempotent, so a duplicate send creates duplicate rows or emails.

## Practice exercises

1. Write a `post_save` receiver that logs every new `Book`, then show with a test that it does not run for `bulk_create` and does run for `create`. Decide whether the receiver should exist at all or whether a service call is better.
2. Replace the notifications receiver with an explicit call from `place_order` to a function in the notifications app. List what you gain (visibility, ordering, error handling) and what you lose (decoupling), and write the test for the new version.
3. Add a second receiver to `order_placed` in another app that increments a daily order counter, using `F` expressions on a unique date row so it is idempotent per event and safe for concurrent orders. Test two receivers together.
4. Demonstrate the recursion trap: write a `post_save` receiver on `Review` that saves the review again, show the infinite loop guarded by a recursion limit in a test, and fix it with `QuerySet.update()`.
5. Write a `m2m_changed` receiver that logs when genres are added to a book and test it with `book.genres.add(...)`, `set(...)`, and `clear()`. Record the `action` values you see for each.
6. Make `announce_order_placed` robust against a receiver that hangs by documenting the limitation, then change the design to enqueue the event to a table (an outbox) and process it from a command (note 5.2). List the trade-offs.
7. Write a test helper (a context manager) that temporarily disconnects all receivers of a signal and restores them afterward, and use it to unit-test a service without notifications running.

## Summary

Signals let one component announce an event and others react without the announcer knowing them. Django sends model signals around `save()` and `delete()`, plus signals for requests, logins, migrations, and many-to-many changes, and you can define your own. Connect receivers from `AppConfig.ready()` with a `dispatch_uid`, accept `**kwargs`, and make them idempotent. Remember that `bulk_create`, `bulk_update`, and `update()` do not send model signals, that receivers run synchronously inside the sender's transaction, that their order is undefined, and that weak references can make local functions vanish. Do side effects after the commit with `transaction.on_commit` and `send_robust`. Prefer explicit function calls for the core flow of your own app, and use signals for decoupling genuinely independent apps, where the listener depends on the announcer and never the reverse.
