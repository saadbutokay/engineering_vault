## Overview

Applications talk to people in two ways that are not part of the main page: email, which reaches them outside the browser (order confirmations, password resets, alerts), and one-time on-screen notices after an action ("Your review was saved"). Django provides both: a mail framework with pluggable backends, and the messages framework for flash messages. Email looks simple and hides real problems: slow SMTP servers blocking requests, header injection, HTML that renders differently in every mail client, deliverability, and sending to people who never asked for it. This note covers `send_mail`, `EmailMessage`, and `EmailMultiAlternatives`, the backends (console, file, SMTP, locmem), HTML emails with templates, sending email outside the request cycle, the modern email API (Django 6.0), `django-anymail`, and the messages framework with its levels and storage. It ends with a tested templated email sender and an order confirmation, wired to the `order_placed` signal from note 5.3, that sends exactly one email per order.

## Core concepts

### The mail framework

`django.core.mail` builds and sends messages through a configured backend. Everyday tools:

- `send_mail(subject, message, from_email, recipient_list, html_message=...)`: the simplest call.
- `EmailMessage`: a message object with `subject`, `body`, `from_email`, `to`, `cc`, `bcc`, `reply_to`, `headers`, and `attach()` for attachments, sent with `.send()`.
- `EmailMultiAlternatives`: an `EmailMessage` that also carries alternative bodies, typically a plain text body plus an HTML version added with `attach_alternative(html, "text/html")`.
- `send_mass_mail` and connection reuse for many messages.

### Backends

`EMAIL_BACKEND` chooses what happens to a message:

- Console (`django.core.mail.backends.console.EmailBackend`): prints messages to the terminal. Ideal in development.
- File (`...backends.filebased.EmailBackend`): writes each message to a file in `EMAIL_FILE_PATH`.
- SMTP (`...backends.smtp.EmailBackend`, the default): sends through a mail server using `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `EMAIL_USE_TLS`, and `EMAIL_TIMEOUT`.
- In-memory (`...backends.locmem.EmailBackend`): collects messages in `django.core.mail.outbox`. The test runner selects it automatically.
- Dummy (`...backends.dummy.EmailBackend`): discards everything.

### Addresses and settings

- `DEFAULT_FROM_EMAIL`: the default sender of your messages.
- `SERVER_EMAIL`: the sender for error emails to `ADMINS`.
- `ADMINS` and `MANAGERS`: lists of recipients for error reports and broken-link notices.

### HTML email with templates

Render the body from templates, never by concatenating strings. Send both a plain text part and an HTML part, because some clients and spam filters prefer plain text, and the multipart message lets each client pick. HTML email is its own dialect: use simple table-based layouts and inline styles, because clients ignore much of modern CSS.

### Sending outside the request cycle

SMTP is a network call to another company's server. It can take seconds or fail, and doing it inside a request makes users wait and turns mail outages into site errors. Better options: register the send with `transaction.on_commit` so it happens only for committed data (note 2.9), and hand it to a background worker (Phase 9) with retries. At minimum, set `EMAIL_TIMEOUT` so a dead server cannot hold a worker forever.

### The modern email API (Django 6.0)

Django 6.0 modernizes `django.core.mail` to build messages on Python's modern `email.message.EmailMessage` API in place of the legacy MIME helper classes. Code that uses `send_mail`, `EmailMessage`, and `EmailMultiAlternatives` as shown in this note keeps working. If you subclass the old internals or inspect the generated MIME objects, read the 6.0 release notes before upgrading. On Django 5.2 LTS, use the API shown here.

### django-anymail

`django-anymail` connects Django's mail API to transactional email services (Amazon SES, Mailgun, SendGrid, Postmark, and others) through their HTTP APIs. You keep using `EmailMessage`, and gain provider features: tags, metadata, scheduling, delivery status after sending, and signals for webhooks that report deliveries, bounces, opens, and complaints.

### The messages framework

Flash messages show a one-time notice on the next page: after a redirect, "Your review was saved". Add them with `messages.success(request, text)` and its siblings, and show them in a template by iterating over `messages`. Messages are stored between requests in a cookie or the session, and disappear once displayed.

### Message levels

`DEBUG` (10), `INFO` (20), `SUCCESS` (25), `WARNING` (30), and `ERROR` (40). `MESSAGE_LEVEL` sets the minimum level that is recorded, and `MESSAGE_TAGS` maps levels to CSS class names so you can style them.

## Basic syntax examples

### Settings for development and production

```python
# development
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
DEFAULT_FROM_EMAIL = "Bookstore <shop@example.com>"
```

```python
# production (SMTP)
import os

EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = os.environ["EMAIL_HOST"]
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.environ["EMAIL_HOST_USER"]
EMAIL_HOST_PASSWORD = os.environ["EMAIL_HOST_PASSWORD"]
EMAIL_TIMEOUT = 10
DEFAULT_FROM_EMAIL = "Bookstore <shop@example.com>"
SERVER_EMAIL = "errors@example.com"
```

### send_mail

```python
from django.core.mail import send_mail

sent = send_mail(
    subject="Welcome",
    message="Thanks for joining.",
    from_email=None,                       # uses DEFAULT_FROM_EMAIL
    recipient_list=["ada@example.com"],
    fail_silently=False,
)
```

### EmailMessage with an attachment

```python
from django.core.mail import EmailMessage

message = EmailMessage(
    subject="Your invoice",
    body="The invoice is attached.",
    to=["ada@example.com"],
    reply_to=["support@example.com"],
)
message.attach("invoice.pdf", pdf_bytes, "application/pdf")
message.send()
```

### A text and HTML message

```python
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string

context = {"name": "Ada"}
message = EmailMultiAlternatives(
    subject="Welcome",
    body=render_to_string("emails/welcome.txt", context),
    to=["ada@example.com"],
)
message.attach_alternative(render_to_string("emails/welcome.html", context), "text/html")
message.send()
```

### Sending after the commit

```python
from functools import partial

from django.db import transaction

transaction.on_commit(partial(send_welcome_email, user.pk))
```

### Reusing one connection for many messages

```python
from django.core.mail import get_connection

with get_connection() as connection:
    for message in messages_to_send:
        message.connection = connection
        message.send()
```

### Testing with the in-memory outbox

```python
from django.core import mail

send_welcome_email(user.pk)
assert len(mail.outbox) == 1
assert mail.outbox[0].to == ["ada@example.com"]
```

### django-anymail

```bash
uv add "django-anymail[mailgun]"
```

```python
import os

EMAIL_BACKEND = "anymail.backends.mailgun.EmailBackend"
ANYMAIL = {
    "MAILGUN_API_KEY": os.environ["MAILGUN_API_KEY"],
    "MAILGUN_SENDER_DOMAIN": "mg.example.com",
}
```

```python
message.tags = ["order-confirmation"]
message.metadata = {"order_id": "42"}
message.send()
print(message.anymail_status.message_id)
```

### The messages framework in a view

```python
from django.contrib import messages
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect


def save_profile(request: HttpRequest) -> HttpResponse:
    # ... save ...
    messages.success(request, "Your profile was saved.")
    messages.add_message(request, messages.WARNING, "Verify your email address.", extra_tags="sticky")
    return redirect("accounts:account")
```

### Showing messages in a template

```html
{% if messages %}
  <ul class="messages">
    {% for message in messages %}
      <li class="{{ message.tags }}">{{ message }}</li>
    {% endfor %}
  </ul>
{% endif %}
```

### Message settings

```python
from django.contrib.messages import constants as message_constants

MESSAGE_LEVEL = message_constants.INFO
MESSAGE_TAGS = {
    message_constants.ERROR: "danger",   # a Bootstrap-style class name
}
```

## Deeper behavior

### What a backend does

`message.send()` asks the backend to deliver the message. The SMTP backend opens a connection, upgrades it with TLS if configured, authenticates, sends, and closes. If anything fails it raises an exception (unless `fail_silently=True`, which hides errors and is dangerous for anything important). The locmem backend appends to `mail.outbox`, which is why tests need no mail server.

### Header injection

Email headers are separated by newlines. If a user-supplied value containing a newline ends up in a header such as the subject or a recipient, an attacker can inject extra headers (for example a `Bcc:` line) and use your server to send spam. Django refuses newlines in headers by raising `BadHeaderError`. Do not rely on that alone: validate recipient addresses, normalize subjects, never let users choose the `From` address, and rate limit any form that sends mail to an address a user types (a "send this to a friend" form is a spam relay).

### Do not log addresses or message bodies

Recipient addresses are personal data and bodies may contain tokens (password reset links). Log the template name and the number of recipients, not the content (note 5.8).

### Plain text and HTML together

Always include a plain text version. The multipart `alternative` structure lets the client choose, and spam filters penalize HTML-only mail. Generate both from the same context, and keep the plain version readable rather than a stripped copy of the HTML.

### Escaping in text templates

Django's template engine escapes HTML characters by default in every template, including `.txt` files, which would turn an ampersand in a book title into `&amp;` in a plain text email. Wrap text templates in `{% autoescape off %}`, and leave autoescaping on for HTML templates, where it protects against injected markup.

### Deliverability

Mail from your domain must prove it is legitimate. Configure SPF (which servers may send for your domain), DKIM (cryptographic signatures), and DMARC (what receivers should do with failures). Use a reputable provider, a consistent `From` address, and a domain you own. Handle bounces and complaints, because sending repeatedly to dead addresses damages your reputation. Separate transactional mail (receipts, resets) from marketing mail, and add unsubscribe headers and links to anything marketing.

### Sensitive content

Email is not a secure channel. Avoid sending passwords or full personal data. Links in mail (password reset, verification) should carry short-lived, single-use tokens, and should use HTTPS. Anyone with access to the mailbox can use them.

### Failures and retries

Mail servers fail temporarily. Sending inside a request means a failure becomes a user-visible error or lost mail. A background worker with retries and backoff (Phase 9) can retry safely, provided the send is idempotent: record that an email was sent (a notification row with a unique constraint, as in note 5.3) so a retry or a duplicate event does not send twice.

### Connection handling

Each `send()` opens a new connection unless you pass one. For many messages, open a connection once and reuse it, as shown above, or send in batches through a provider API with `django-anymail`. Set `EMAIL_TIMEOUT` in every environment.

### Messages storage

The default `FallbackStorage` stores messages in a cookie first and falls back to the session when they do not fit (cookies are limited to about 4 KB). Cookie storage is signed so users cannot forge messages, but the content is readable by the browser, so never put sensitive data in messages. `SessionStorage` keeps everything on the server and requires the session middleware.

### Messages need the middleware and the context processor

`MessageMiddleware` and the `django.contrib.messages.context_processors.messages` context processor are both enabled in the default project. Without the middleware, `messages.add_message()` raises `MessageFailure` unless you pass `fail_silently=True`. Without the context processor, templates see no `messages` variable.

### Messages are consumed when read

Iterating over `messages` in a template marks them used, and they are removed at the end of that response. A response that never renders them (a JSON response, an HTMX fragment) leaves them queued for the next full page. For fragment responses, show feedback inline instead.

### Messages and user content

The text of a message is escaped when rendered, so including a user's input in a message is safe. If you need markup in a message, build it with `format_html` and mark the message safe deliberately with `extra_tags="safe"` and `mark_safe`, which should be rare.

### When not to use messages

Use messages for confirmations and notices after a redirect. Use form errors for validation, an inline success state for HTMX fragments, and a notification system with storage (note 5.3) for things that must persist or be shown to someone who is not currently online.

## Industry-level snippet

A reusable function that sends a templated, multipart email safely, an order confirmation built on it, and the notifications receiver from note 5.3 extended so that exactly one email is sent per order, even if the event is delivered twice. The tests check content, escaping, header safety, and idempotency using the in-memory outbox.

`apps/notifications/emailing.py`:

```python
"""Sending templated multipart emails safely."""

import logging
from collections.abc import Mapping, Sequence
from typing import Any

from django.conf import settings
from django.core.exceptions import ValidationError
from django.core.mail import EmailMultiAlternatives
from django.core.mail.backends.base import BaseEmailBackend
from django.core.validators import validate_email
from django.template.loader import render_to_string

logger = logging.getLogger(__name__)

MAX_SUBJECT_LENGTH = 200


def _clean_subject(subject: str) -> str:
    """Collapse whitespace, including newlines, so a subject cannot add headers."""
    cleaned = " ".join(subject.split())
    if not cleaned:
        raise ValueError("The subject must not be blank")
    if len(cleaned) > MAX_SUBJECT_LENGTH:
        raise ValueError(f"The subject must be at most {MAX_SUBJECT_LENGTH} characters")
    return cleaned


def _clean_recipients(to: Sequence[str] | str) -> list[str]:
    recipients = [to] if isinstance(to, str) else list(to)
    if not recipients:
        raise ValueError("At least one recipient is required")
    for address in recipients:
        try:
            validate_email(address)  # also rejects newlines
        except ValidationError as exc:
            raise ValueError(f"Invalid recipient address: {address!r}") from exc
    return recipients


def send_templated_email(
    *,
    to: Sequence[str] | str,
    subject: str,
    template: str,
    context: Mapping[str, Any],
    from_email: str | None = None,
    reply_to: Sequence[str] = (),
    connection: BaseEmailBackend | None = None,
) -> int:
    """Render ``<template>.txt`` and ``<template>.html`` and send them together.

    Returns the number of messages sent (1 on success).

    Raises:
        ValueError: for a blank or overlong subject, or an invalid recipient.
    """
    recipients = _clean_recipients(to)
    message = EmailMultiAlternatives(
        subject=_clean_subject(subject),
        body=render_to_string(f"{template}.txt", dict(context)),
        from_email=from_email or settings.DEFAULT_FROM_EMAIL,
        to=recipients,
        reply_to=list(reply_to) or None,
        connection=connection,
    )
    message.attach_alternative(render_to_string(f"{template}.html", dict(context)), "text/html")
    sent = message.send(fail_silently=False)
    logger.info("Sent %s to %d recipient(s)", template, len(recipients))  # no addresses in logs
    return sent
```

`apps/notifications/order_emails.py`:

```python
"""Order-related emails."""

from dataclasses import dataclass
from decimal import Decimal

from django.db.models import Prefetch

from apps.bookstore.models import Order, OrderItem
from apps.notifications.emailing import send_templated_email


@dataclass(frozen=True)
class ConfirmationLine:
    title: str
    quantity: int
    unit_price: Decimal


def send_order_confirmation(order_id: int) -> int:
    """Email the customer a summary of the order. Raises ``Order.DoesNotExist``."""
    items = OrderItem.objects.select_related("book").order_by("id")
    order = Order.objects.prefetch_related(Prefetch("items", queryset=items)).get(pk=order_id)
    lines = [
        ConfirmationLine(item.book.title, item.quantity, item.unit_price)
        for item in order.items.all()
    ]
    total = sum((line.quantity * line.unit_price for line in lines), start=Decimal("0"))
    return send_templated_email(
        to=order.customer_email,
        subject=f"Your order #{order.pk}",
        template="notifications/order_confirmation",
        context={"order": order, "lines": lines, "total": total},
    )
```

`apps/notifications/templates/notifications/order_confirmation.txt`:

```text
{% autoescape off %}Thanks for your order #{{ order.pk }}.
{% for line in lines %}
{{ line.quantity }} x {{ line.title }} at {{ line.unit_price }}{% endfor %}

Total: {{ total }}
{% endautoescape %}
```

`apps/notifications/templates/notifications/order_confirmation.html`:

```html
<p>Thanks for your order #{{ order.pk }}.</p>
<ul>
  {% for line in lines %}
    <li>{{ line.quantity }} x {{ line.title }} at {{ line.unit_price }}</li>
  {% endfor %}
</ul>
<p><strong>Total: {{ total }}</strong></p>
```

The receiver from note 5.3 now sends the email only when it newly creates the notification row, which makes the whole step idempotent:

```python
@receiver(order_placed, dispatch_uid="notifications.record_order_placed")
def record_order_placed(sender: type[Order], order_id: int, **kwargs: Any) -> None:
    """Record the notification and email the customer, once per order."""
    _, created = Notification.objects.get_or_create(
        order_id=order_id, kind=Notification.Kind.ORDER_PLACED
    )
    if created:
        send_order_confirmation(order_id)
    logger.info("Order %s notification %s", order_id, "sent" if created else "already existed")
```

Add `from apps.notifications.order_emails import send_order_confirmation` to the imports of `receivers.py`.

`apps/notifications/tests/test_email.py`:

```python
from decimal import Decimal

from django.core import mail
from django.test import TestCase, override_settings

from apps.bookstore.models import Order
from apps.bookstore.services.checkout import InsufficientStockError, place_order
from apps.bookstore.services.orders import OrderLine
from apps.bookstore.tests.factories import make_book
from apps.notifications.emailing import send_templated_email
from apps.notifications.models import Notification
from apps.notifications.order_emails import send_order_confirmation
from apps.notifications.receivers import record_order_placed

TEMPLATE = "notifications/order_confirmation"


class SendTemplatedEmailTests(TestCase):
    def send(
        self,
        *,
        to: str | list[str] = "a@example.com",
        subject: str = "Hello",
        reply_to: list[str] | None = None,
    ) -> int:
        order = Order.objects.create(customer_email="a@example.com")
        return send_templated_email(
            to=to,
            subject=subject,
            template=TEMPLATE,
            context={"order": order, "lines": [], "total": Decimal("0")},
            reply_to=reply_to or [],
        )

    @override_settings(DEFAULT_FROM_EMAIL="Shop <shop@example.com>")
    def test_the_message_is_multipart_with_the_default_sender(self) -> None:
        self.assertEqual(self.send(reply_to=["help@example.com"]), 1)
        message = mail.outbox[0]
        self.assertEqual(message.from_email, "Shop <shop@example.com>")
        self.assertEqual(message.reply_to, ["help@example.com"])
        self.assertIn("Thanks for your order", message.body)
        html, mimetype = message.alternatives[0][0], message.alternatives[0][1]
        self.assertEqual(mimetype, "text/html")
        self.assertIn("<strong>Total", str(html))

    def test_newlines_in_the_subject_cannot_inject_headers(self) -> None:
        self.send(subject="Hello\nBcc: evil@example.com")
        message = mail.outbox[0]
        self.assertEqual(message.subject, "Hello Bcc: evil@example.com")
        self.assertEqual(message.bcc, [])
        self.assertNotIn("Bcc", message.extra_headers)

    def test_bad_recipients_are_rejected_before_anything_is_sent(self) -> None:
        for recipient in ("not-an-email", "a@example.com\nbcc:x@example.com", [], ""):
            with self.subTest(recipient=recipient):
                with self.assertRaises(ValueError):
                    self.send(to=recipient)
        self.assertEqual(mail.outbox, [])

    def test_bad_subjects_are_rejected(self) -> None:
        for subject in ("", "   \n ", "x" * 201):
            with self.subTest(subject=subject[:10]):
                with self.assertRaises(ValueError):
                    self.send(subject=subject)


class OrderConfirmationTests(TestCase):
    def place(self, title: str = "Dune") -> Order:
        book = make_book(title=title, stock=5)
        return place_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 2)])

    def test_content_and_escaping(self) -> None:
        order = self.place(title="<b>Bold</b> & Co")
        send_order_confirmation(order.pk)
        message = mail.outbox[0]
        self.assertEqual(message.to, ["a@example.com"])
        self.assertIn(f"#{order.pk}", message.subject)
        self.assertIn("2 x <b>Bold</b> & Co", message.body)           # plain text: no HTML escaping
        html = str(message.alternatives[0][0])
        self.assertIn("&lt;b&gt;Bold&lt;/b&gt; &amp; Co", html)        # HTML: escaped
        self.assertIn("Total: 20.00", message.body)

    def test_a_confirmation_is_sent_once_per_order_after_the_commit(self) -> None:
        with self.captureOnCommitCallbacks(execute=True):
            order = self.place()
        self.assertEqual(len(mail.outbox), 1)

        record_order_placed(sender=Order, order_id=order.pk)           # a duplicate event
        self.assertEqual(len(mail.outbox), 1)
        self.assertEqual(Notification.objects.filter(order=order).count(), 1)

    def test_no_email_for_a_failed_order(self) -> None:
        book = make_book(stock=0)
        with self.captureOnCommitCallbacks(execute=True):
            with self.assertRaises(InsufficientStockError):
                place_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        self.assertEqual(mail.outbox, [])
```

## Common mistakes

- Sending email inside the request cycle with no timeout, so a slow mail server slows or breaks pages.
- Sending email in `post_save` or inside a transaction, before the data is committed.
- Letting users control the `From`, `Reply-To`, or recipient fields without validation, or exposing a form that sends mail to any address without rate limiting.
- Concatenating HTML strings instead of using templates, or leaving autoescaping off in HTML templates.
- Leaving autoescaping on in plain text templates and sending `&amp;` to readers.
- Sending HTML only, with no plain text alternative.
- Using `fail_silently=True` for mail that matters.
- Logging message bodies or recipient addresses, which may contain tokens and personal data.
- Forgetting SPF, DKIM, and DMARC, then wondering why mail lands in spam.
- Sending the same email twice because a retry or duplicate event was not guarded by a unique record.
- Putting sensitive data, passwords, or long-lived links in emails.
- Adding messages in a view that returns JSON or an HTMX fragment, which leaves them queued for an unrelated later page.

## Practice exercises

1. Configure the console backend, send the welcome email from a sign-up flow, and read the multipart output in the terminal. Identify the boundary lines and the two content types.
2. Write `send_password_changed_notice(user_id)` that emails the user when their password changes, using a template, and test it through `mail.outbox`. Decide what the email must not contain.
3. Add an `Attachment` to the order confirmation (a plain text receipt generated in memory) and test the attachment name, content type, and content.
4. Implement a "send this book to a friend" feature that emails a link to an address the user types. Add per-user and per-address rate limits from note 4.4, and write tests for the limits and for header injection attempts in every field.
5. Use `get_connection()` to send a batch of 50 messages over a single connection, and prove with a mock backend that the connection was opened once.
6. Add an optional `headers` argument to `send_templated_email` that accepts a `List-Unsubscribe` value for marketing mail, rejects values containing newlines, and test both a valid header and an injection attempt.
7. Replace `messages.success` in a view with a version that works for both normal requests and HTMX fragment requests: normal requests add a flash message and redirect, and fragment requests render the confirmation inline. Test both.

## Summary

Django's mail framework builds messages with `EmailMessage` and `EmailMultiAlternatives`, sends them through a backend chosen by `EMAIL_BACKEND` (console in development, SMTP or a `django-anymail` provider in production, in-memory in tests), and exposes `mail.outbox` for assertions. Send both text and HTML from templates, with autoescaping on in HTML and off in plain text, validate recipients and normalize subjects to prevent header injection, and keep addresses and bodies out of logs. Do not send mail inside the request or before the commit: register the send with `transaction.on_commit`, hand it to a worker with retries when you can, set `EMAIL_TIMEOUT`, and make the send idempotent with a unique record. Configure SPF, DKIM, and DMARC for deliverability. The messages framework shows one-time notices after redirects, with levels and tags, stored in a signed cookie or the session, and consumed when displayed; use inline feedback for fragment and JSON responses.
