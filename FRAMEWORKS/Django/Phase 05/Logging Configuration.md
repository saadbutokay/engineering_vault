## Overview

When something goes wrong in production, logs are usually the only evidence you have. You cannot attach a debugger to a server handling thousands of requests, and the user who hit the bug has long since gone. Good logging answers four questions after the fact: what happened, to whom, in which request, and why. Bad logging is either silent (nothing recorded when it matters), noisy (so much that nothing can be found), or dangerous (passwords, tokens, and personal data written to files that many people can read). Python's `logging` module and Django's `LOGGING` setting give you the tools. This note covers the `LOGGING` dictionary, loggers, handlers, formatters, and filters, Django's built-in loggers (`django`, `django.request`, and `django.db.backends`), JSON logging, an overview of `structlog`, request ID propagation, `mail_admins` and its limits, never logging secrets or personal data, and why `print` is not used in production. It ends with a tested configuration builder that produces structured JSON logs with request IDs, redacts secrets before they reach any handler, and validates its own options.

## Core concepts

### The four parts of Python logging

- A logger is the object your code writes to (`logger.info("...")`). Loggers have names that form a hierarchy with dots: `apps.bookstore.services` is a child of `apps.bookstore`, which is a child of `apps`.
- A handler decides where records go: the console, a file, an email, a network service.
- A formatter turns a record into text: a plain line or a JSON object.
- A filter can reject records or modify them before they are handled.

### Levels

From least to most severe: `DEBUG` (detail for developers), `INFO` (normal business events), `WARNING` (unexpected but handled), `ERROR` (an operation failed), `CRITICAL` (the system may be unusable). A logger emits records at or above its level, and each handler can apply its own level as well.

### Propagation

A record sent to a logger is handled by that logger's handlers, then passed to its parent's handlers, and so on up to the root logger, unless a logger sets `propagate: False`. This is why configuring handlers on the root logger or on a top-level logger such as `apps` covers everything beneath it.

### Using loggers in code

Create one logger per module with `logger = logging.getLogger(__name__)`. The module name places it correctly in the hierarchy, and the name appears in every record so you can see where it came from.

### The LOGGING setting

Django applies `LOGGING`, a dictionary in the `logging.config.dictConfig` schema, at startup. It names formatters, filters, handlers, and loggers declaratively. Leaving `disable_existing_loggers` at its default of `True` silently disables loggers created before configuration, so set it to `False`.

### Django's loggers

- `django`: the parent of all of Django's loggers.
- `django.request`: records for responses with status 4xx (as warnings) and 5xx (as errors), including the traceback for server errors.
- `django.server`: the development server's request lines.
- `django.template`: template engine problems, for example missing variables at `DEBUG` level.
- `django.db.backends`: every SQL statement, at `DEBUG` level and only when `DEBUG = True`.
- `django.security.*`: security-related events, such as `django.security.DisallowedHost` for requests with an invalid `Host` header, which scanners generate constantly.

### Structured (JSON) logging

A log line is for machines first and people second. JSON lines (one JSON object per line) carry fields such as `level`, `logger`, `message`, `request_id`, and `order_id`. Log platforms index them, so you can search for `request_id = X` or count errors per endpoint without parsing prose.

### structlog

`structlog` is a library that makes structured logging the default style: you log an event name plus key-value pairs, and processors add timestamps, levels, and bound context (such as the request ID) to every record. It can sit on top of the standard `logging` module so that Django and third-party libraries log through the same pipeline.

### Request IDs

A request ID is a short identifier assigned to each request and attached to every log record produced while handling it. With it you can collect all the lines for one failing request, across several modules and even services. The middleware from note 1.6 stores the ID in a context variable, and a log filter copies it onto each record.

### mail_admins

`AdminEmailHandler` emails each error record to the addresses in `ADMINS`. It is easy to enable and easy to regret: it sends one email per error with no grouping or rate limit, so an outage floods the inbox and may exceed your email quota, and the emails contain tracebacks and request data. Error tracking services (Sentry, note 11.7) group duplicates and are the right tool for production.

### What never to log

Passwords, tokens, API keys, session identifiers, full payment details, and personal data beyond what you need (email addresses, phone numbers, addresses, request bodies). Logs are copied to many systems, kept for months, and readable by many people, and they are a favorite target in breaches. Where an identifier is needed, log an internal ID. Where a value must appear, redact it.

### Why print is not used in production

`print` has no level, no timestamp, no logger name, no structure, and no filtering. It cannot be routed differently for different parts of the code, silenced selectively, or enriched with a request ID. Output may be buffered and appear late or not at all. Libraries use `logging`, so mixing `print` splits your output into two unrelated streams.

### Where logs should go

For containers and platforms that collect output, write logs to standard output or standard error, one record per line, and let the platform ship them (the twelve-factor approach). Writing and rotating files yourself is for traditional servers, and needs rotation and permissions handled with care.

## Basic syntax examples

### A logger per module

```python
import logging

logger = logging.getLogger(__name__)


def charge(order_id: int, amount_cents: int) -> None:
    logger.info("Charging order %s for %d cents", order_id, amount_cents)
    try:
        call_payment_provider(order_id, amount_cents)
    except TimeoutError:
        logger.warning("Payment provider timed out for order %s; will retry", order_id)
        raise
    except Exception:
        logger.exception("Payment failed for order %s", order_id)   # includes the traceback
        raise
```

Pass values as arguments (`%s`) rather than building the message with an f-string. The formatting then happens only when a handler actually emits the record, and log aggregators can group identical messages.

### Structured extras

```python
logger.info("Order placed", extra={"order_id": order.pk, "items": 3})
```

The extra keys become attributes on the record, and a JSON formatter can output them as fields. Do not use names that already exist on a record (`name`, `args`, `message`), because logging raises an error.

### A minimal LOGGING setting

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {"format": "%(asctime)s %(levelname)s %(name)s: %(message)s"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "standard"},
    },
    "loggers": {
        "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "apps": {"handlers": ["console"], "level": "INFO", "propagate": False},
    },
    "root": {"handlers": ["console"], "level": "WARNING"},
}
```

### Seeing SQL in development

```python
LOGGING["loggers"]["django.db.backends"] = {
    "handlers": ["console"],
    "level": "DEBUG",
    "propagate": False,
}
```

### Quieting a noisy logger

```python
LOGGING["handlers"]["null"] = {"class": "logging.NullHandler"}
LOGGING["loggers"]["django.security.DisallowedHost"] = {
    "handlers": ["null"],
    "propagate": False,
}
```

### Mailing errors to administrators

```python
ADMINS = [("Ops", "ops@example.com")]
SERVER_EMAIL = "errors@example.com"

LOGGING["filters"] = {"require_debug_false": {"()": "django.utils.log.RequireDebugFalse"}}
LOGGING["handlers"]["mail_admins"] = {
    "class": "django.utils.log.AdminEmailHandler",
    "level": "ERROR",
    "filters": ["require_debug_false"],
    "include_html": False,
}
LOGGING["loggers"]["django.request"] = {"handlers": ["mail_admins"], "level": "ERROR", "propagate": True}
```

### structlog on top of the standard library

```bash
uv add structlog
```

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.JSONRenderer(),
    ],
)

log = structlog.get_logger()
structlog.contextvars.bind_contextvars(request_id="abc123")
log.info("order_placed", order_id=42, items=3)
```

### Keeping sensitive values out of error reports

```python
from django.views.decorators.debug import sensitive_post_parameters, sensitive_variables


@sensitive_variables("card_number", "cvv")
def charge_card(card_number: str, cvv: str) -> None:
    ...


@sensitive_post_parameters("password", "new_password")
def change_password(request: HttpRequest) -> HttpResponse:
    ...
```

These hide the named values in error pages and emails. They do not affect your own `logger` calls.

## Deeper behavior

### How a record travels

`logger.info()` creates a `LogRecord`, checks the logger's effective level (its own, or the nearest ancestor with one), runs the logger's filters, hands the record to each of its handlers, and then to the parent's handlers if `propagate` is true. Each handler checks its own level and filters, formats the record with its formatter, and emits it. A record can therefore appear twice if both a child logger and its parent have a handler to the same destination, which is a common mistake. Attach handlers at one level of the hierarchy and let propagation do the rest.

### Handler filters versus logger filters

Filters attached to a logger apply only to records created directly on that logger, not to those propagated from its children. Filters attached to a handler see every record that reaches it. Attach redaction and request ID filters to handlers, so that no record can reach an output without them.

### lastResort

If no handler is found anywhere in the hierarchy, Python uses a fallback handler that prints WARNING and above to standard error. A logger configured with `propagate: False` and no handlers triggers this. To really discard records, attach a `NullHandler`.

### Django's defaults

Without a `LOGGING` setting, Django logs `django` messages at INFO and above to the console only when `DEBUG` is true, and mails errors to `ADMINS` only when `DEBUG` is false. When you provide your own `LOGGING`, you must restate what you need, because your dictionary is applied on top of Django's defaults (with `disable_existing_loggers` controlling the rest).

### The reserved names of a record

Attributes such as `name`, `msg`, `args`, `levelname`, `pathname`, `lineno`, `exc_info`, `created`, and `message` exist on every record, and passing one of them in `extra` raises `KeyError`. A JSON formatter that copies extras must skip those names, and decide what to do about sensitive keys.

### Exceptions in logs

`logger.exception(...)` logs at ERROR and attaches the current exception's traceback. Use it only inside an `except` block. Log an exception once, at the layer that handles it, rather than at every layer it passes through, or one failure produces five identical tracebacks. Tracebacks can contain secrets in exception messages, so run them through redaction too.

### Cost and volume

Logging has a price: CPU for formatting, I/O, and storage and indexing fees. Avoid `DEBUG` logging in hot loops, log at `INFO` for business events rather than every function call, and sample high-volume events. A log line nobody ever reads is waste, and a log line that would have explained last night's outage is priceless, so log decisions, failures, and state changes at boundaries (requests, tasks, external calls).

### Blocking logging

A handler that writes to a slow destination (a network service, a mail server) blocks the code that logged. `logging.handlers.QueueHandler` with a `QueueListener` moves the slow work to a background thread. For standard output in containers this is rarely needed.

### Request IDs across services

Pass the request ID to other services in a header (`X-Request-ID`) and accept an incoming one, validated, as note 1.6's middleware does. With the ID in every service's logs, one search follows a request across the whole system. Background tasks should receive the ID as an argument and bind it when they start.

### Redaction strategy

Redact in layers. Do not log secrets in the first place. Add a handler-level filter that scrubs common patterns (authorization headers, `password=...`, your own token format) as a safety net. Redact structured fields by key name. Keep the patterns conservative: a filter cannot recognize every secret, which is why the first layer matters most.

### Access logs

The application server (Gunicorn, Uvicorn) writes its own access log of every request with status and timing, configured separately from Django (note 12.2). Use it for traffic analysis, and use application logs for events, failures, and context.

### Testing logging

`assertLogs` captures records from a logger in a test, and a test can attach a handler to an in-memory stream to check the final text. When a test applies a whole `LOGGING` configuration, it changes global state, so restore the loggers afterward.

## Industry-level snippet

A configuration module with a JSON formatter, a redaction filter, and a validated builder for the `LOGGING` dictionary. It produces one JSON object per line with a UTC timestamp and the request ID, scrubs secrets from messages, extras, and tracebacks, silences `DisallowedHost` noise, and enables SQL logging only in debug mode. The tests include an end-to-end check that applies the configuration and reads what a real logger writes.

`config/logging_config.py`:

```python
"""Logging configuration: JSON output, redaction, and a validated builder."""

import json
import logging
import re
from datetime import UTC, datetime
from typing import Any

from config.env import ConfigError

VALID_LEVELS = frozenset({"DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"})

# Attributes every LogRecord has, so they are not copied as "extra" fields.
_STANDARD_ATTRIBUTES = frozenset(
    logging.LogRecord("", 0, "", 0, "", (), None).__dict__
) | {"message", "asctime", "request_id"}

_SENSITIVE_KEYS = frozenset({"password", "passwd", "secret", "token", "authorization", "api_key"})

_REDACTIONS: tuple[tuple[re.Pattern[str], str], ...] = (
    (re.compile(r"(?i)(authorization\s*[:=]\s*bearer\s+)[^\s,;]+"), r"\1[REDACTED]"),
    (
        re.compile(r"(?i)\b(password|passwd|secret|token|api[_-]?key)\b(\s*[=:]\s*)[^\s,;&]+"),
        r"\1\2[REDACTED]",
    ),
    (re.compile(r"\bbk_[0-9a-f]{8}_[A-Za-z0-9_-]+"), "[REDACTED_API_KEY]"),
)


def redact(text: str) -> str:
    """Replace common secret patterns in ``text``."""
    for pattern, replacement in _REDACTIONS:
        text = pattern.sub(replacement, text)
    return text


class RedactingFilter(logging.Filter):
    """Scrub secrets from the final message before any handler formats it."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.msg = redact(record.getMessage())
        record.args = ()
        return True


class JsonFormatter(logging.Formatter):
    """Format each record as one JSON object on one line."""

    def format(self, record: logging.LogRecord) -> str:
        payload: dict[str, Any] = {
            "time": datetime.fromtimestamp(record.created, tz=UTC).isoformat(timespec="milliseconds"),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        request_id = record.__dict__.get("request_id")
        if request_id and request_id != "-":
            payload["request_id"] = request_id
        if record.exc_info and record.exc_info[0] is not None:
            payload["exception"] = redact(self.formatException(record.exc_info))
        for key, value in record.__dict__.items():
            if key in _STANDARD_ATTRIBUTES or key.startswith("_") or key in payload:
                continue
            payload[key] = "[REDACTED]" if key.lower() in _SENSITIVE_KEYS else value
        return json.dumps(payload, default=str, ensure_ascii=False)


def build_logging(
    *,
    debug: bool,
    level: str = "INFO",
    json_logs: bool = True,
    mail_admins: bool = False,
    log_sql: bool = False,
) -> dict[str, Any]:
    """Return a validated ``LOGGING`` dictionary."""
    level_name = level.upper()
    if level_name not in VALID_LEVELS:
        raise ConfigError(f"Unknown log level: {level!r}")
    if log_sql and not debug:
        raise ConfigError("SQL logging only works when DEBUG is true")

    handlers: dict[str, dict[str, Any]] = {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "json" if json_logs else "text",
            "filters": ["request_id", "redact"],
        },
        "null": {"class": "logging.NullHandler"},
    }
    loggers: dict[str, dict[str, Any]] = {
        "django": {"handlers": ["console"], "level": level_name, "propagate": False},
        "apps": {"handlers": ["console"], "level": level_name, "propagate": False},
        # Scanners probe with random Host headers; the noise hides real problems.
        "django.security.DisallowedHost": {"handlers": ["null"], "propagate": False},
        "django.db.backends": {
            "handlers": ["console"],
            "level": "DEBUG" if log_sql else "WARNING",
            "propagate": False,
        },
    }
    if mail_admins:
        handlers["mail_admins"] = {
            "class": "django.utils.log.AdminEmailHandler",
            "level": "ERROR",
            "filters": ["require_debug_false", "redact"],
            "include_html": False,
        }
        loggers["django.request"] = {
            "handlers": ["mail_admins"],
            "level": "ERROR",
            "propagate": True,  # also reaches the console through "django"
        }

    return {
        "version": 1,
        "disable_existing_loggers": False,
        "filters": {
            "request_id": {"()": "apps.core.middleware.RequestIDFilter"},
            "redact": {"()": "config.logging_config.RedactingFilter"},
            "require_debug_false": {"()": "django.utils.log.RequireDebugFalse"},
        },
        "formatters": {
            "json": {"()": "config.logging_config.JsonFormatter"},
            "text": {
                "format": "%(asctime)s %(levelname)s [%(request_id)s] %(name)s: %(message)s"
            },
        },
        "handlers": handlers,
        "loggers": loggers,
        "root": {"handlers": ["console"], "level": "WARNING"},
    }
```

In `config/settings.py`:

```python
import os

from config.logging_config import build_logging

LOGGING = build_logging(
    debug=DEBUG,
    level=os.environ.get("LOG_LEVEL", "INFO"),
    json_logs=not DEBUG,
)
```

`config/test_logging_config.py`:

```python
import io
import json
import logging
import logging.config
import sys
from collections.abc import Iterator
from contextlib import contextmanager, redirect_stderr
from typing import Any

import pytest

from apps.core.middleware import request_id_var
from config.env import ConfigError
from config.logging_config import JsonFormatter, RedactingFilter, build_logging, redact

TOUCHED_LOGGERS = ("", "django", "apps", "django.request", "django.db.backends", "django.security.DisallowedHost")


@contextmanager
def preserved_logging() -> Iterator[None]:
    """Restore the global logging state that ``dictConfig`` changes."""
    saved = {
        name: (logger.handlers[:], logger.level, logger.propagate)
        for name in TOUCHED_LOGGERS
        for logger in [logging.getLogger(name)]
    }
    try:
        yield
    finally:
        for name, (handlers, level, propagate) in saved.items():
            logger = logging.getLogger(name)
            logger.handlers[:] = handlers
            logger.setLevel(level)
            logger.propagate = propagate


def make_record(message: str = "hello", *args: Any, **extras: Any) -> logging.LogRecord:
    record = logging.LogRecord("apps.test", logging.INFO, __file__, 1, message, args, None)
    record.__dict__.update(extras)
    return record


def format_json(record: logging.LogRecord) -> dict[str, Any]:
    parsed: dict[str, Any] = json.loads(JsonFormatter().format(record))
    return parsed


def test_the_basic_fields_are_present() -> None:
    payload = format_json(make_record("hello %s", "world"))
    assert payload["message"] == "hello world"
    assert payload["level"] == "INFO"
    assert payload["logger"] == "apps.test"
    assert payload["time"].endswith("+00:00")
    assert "request_id" not in payload


def test_request_ids_and_extras_become_fields() -> None:
    payload = format_json(make_record(request_id="trace-1", order_id=7))
    assert payload["request_id"] == "trace-1"
    assert payload["order_id"] == 7
    assert "request_id" not in format_json(make_record(request_id="-"))


def test_sensitive_extras_are_redacted_by_key() -> None:
    payload = format_json(make_record(password="hunter2", Token="abc", note="fine"))
    assert payload["password"] == "[REDACTED]"
    assert payload["Token"] == "[REDACTED]"
    assert payload["note"] == "fine"


def test_exceptions_are_included_and_scrubbed() -> None:
    try:
        raise ValueError("bad token=abc123")
    except ValueError:
        record = logging.LogRecord("apps.test", logging.ERROR, __file__, 1, "failed", (), sys.exc_info())
    payload = format_json(record)
    assert "ValueError" in payload["exception"]
    assert "abc123" not in payload["exception"]


def test_unicode_is_not_escaped() -> None:
    assert "café" in JsonFormatter().format(make_record("café"))


@pytest.mark.parametrize(
    ("text", "expected"),
    [
        ("Authorization: Bearer abc123", "Authorization: Bearer [REDACTED]"),
        ("login failed password=hunter2 user=ada", "login failed password=[REDACTED] user=ada"),
        ("api_key: sk_live_123", "api_key: [REDACTED]"),
        ("created key bk_1a2b3c4d_secretPART-xyz_9", "created key [REDACTED_API_KEY]"),
        ("nothing sensitive here", "nothing sensitive here"),
    ],
)
def test_redaction_patterns(text: str, expected: str) -> None:
    assert redact(text) == expected


def test_the_filter_scrubs_values_passed_as_arguments() -> None:
    record = make_record("token=%s", "abc123")
    assert RedactingFilter().filter(record) is True
    assert record.getMessage() == "token=[REDACTED]"


def test_the_default_configuration_is_structured_and_complete() -> None:
    config = build_logging(debug=False)
    assert config["disable_existing_loggers"] is False
    assert config["handlers"]["console"]["formatter"] == "json"
    assert config["handlers"]["console"]["filters"] == ["request_id", "redact"]
    assert config["loggers"]["django.db.backends"]["level"] == "WARNING"
    assert "mail_admins" not in config["handlers"]
    assert config["loggers"]["django.security.DisallowedHost"]["handlers"] == ["null"]


def test_options_change_the_configuration() -> None:
    text = build_logging(debug=True, json_logs=False, log_sql=True)
    assert text["handlers"]["console"]["formatter"] == "text"
    assert text["loggers"]["django.db.backends"]["level"] == "DEBUG"
    mailing = build_logging(debug=False, mail_admins=True)
    assert mailing["loggers"]["django.request"]["handlers"] == ["mail_admins"]
    assert "redact" in mailing["handlers"]["mail_admins"]["filters"]


@pytest.mark.parametrize(
    "kwargs",
    [{"level": "LOUD"}, {"debug": False, "log_sql": True}],
)
def test_invalid_options_are_rejected(kwargs: dict[str, Any]) -> None:
    options: dict[str, Any] = {"debug": True}
    options.update(kwargs)
    with pytest.raises(ConfigError):
        build_logging(**options)


def test_end_to_end_json_logging_with_redaction_and_a_request_id() -> None:
    buffer = io.StringIO()
    with preserved_logging(), redirect_stderr(buffer):
        logging.config.dictConfig(build_logging(debug=False, level="INFO"))
        token = request_id_var.set("trace-42")
        try:
            logging.getLogger("apps.demo").info("login password=hunter2 for %s", "ada")
        finally:
            request_id_var.reset(token)

    line = buffer.getvalue().strip()
    payload = json.loads(line)
    assert payload["request_id"] == "trace-42"
    assert payload["logger"] == "apps.demo"
    assert "hunter2" not in line
    assert payload["message"] == "login password=[REDACTED] for ada"
```

The end-to-end test is the one that matters: it proves that the pieces connect in the real configuration, so a secret in a message cannot reach the output even though each part was also tested alone.

## Common mistakes

- Using `print` for diagnostics that must survive in production.
- Building log messages with f-strings, which format eagerly and defeat grouping.
- Leaving `disable_existing_loggers` at `True`, so loggers created before configuration go silent.
- Attaching the same handler to a logger and its parent, so every line appears twice.
- Logging passwords, tokens, session IDs, authorization headers, full request bodies, or personal data.
- Logging the same exception at several layers.
- Using `logger.exception` outside an `except` block, or `logger.error` inside one and losing the traceback.
- Passing reserved names in `extra`, which raises an error at the worst moment.
- Enabling `mail_admins` in production and being flooded during an outage.
- Logging at `DEBUG` or logging SQL in production, drowning real signals and inflating cost.
- Putting redaction on the logger instead of the handler, so propagated records escape it.
- Writing log files from several processes without rotation and permissions planned.
- Forgetting to carry the request ID into background tasks and outgoing calls.

## Practice exercises

1. Add a logger and sensible log calls to `place_order` (note 2.9): an `INFO` record when an order is placed with the order ID and item count, a `WARNING` when stock is insufficient, and no personal data. Write tests with `assertLogs` for both.
2. Add a `sampled_fraction` option to the JSON setup that keeps only a given fraction of `DEBUG` records (a filter), and write a test with a fixed random seed.
3. Extend `RedactingFilter` so it also scrubs email addresses to `a***@example.com` form, and write tests that show valid emails are masked while other text is untouched. Discuss when masking is not enough.
4. Write middleware that logs one JSON line per request with the method, path (without the query string), status, duration in milliseconds, and user ID (not email), using the request ID filter. Test the fields and that a failing view still produces a line.
5. Bind the request ID in a Celery-style task wrapper (a decorator is enough, with no Celery needed): it takes the ID as an argument, sets `request_id_var` for the duration of the task, and resets it. Test that log records inside the task carry the ID.
6. Replace the standard-library JSON formatter with `structlog`'s `ProcessorFormatter` so that both `structlog` and standard-library records are rendered the same way, and write a test for a record from each.
7. Use `QueueHandler` and `QueueListener` to move a slow handler (simulated with a handler that sleeps) off the request thread, and measure the difference in time spent in a function that logs 100 records.

## Summary

Python logging routes records from named loggers through handlers, formatters, and filters, and Django configures it with the `LOGGING` dictionary. Create one logger per module with `getLogger(__name__)`, pass values as arguments, use the right level, and log an exception once with its traceback. Attach handlers at one level of the hierarchy, set `disable_existing_loggers` to `False`, silence known noise such as `DisallowedHost` with a `NullHandler`, and enable SQL logging only in development. Emit structured JSON to standard output with a request ID on every record, and put redaction filters on handlers so that secrets cannot reach any output. Never log passwords, tokens, or personal data, avoid `mail_admins` in production in favor of an error tracking service, and never rely on `print`. Build the configuration in code with validated options, and test it end to end by applying it and reading what a real logger writes.
