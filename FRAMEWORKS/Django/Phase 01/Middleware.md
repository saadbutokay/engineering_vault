## Overview

Middleware is code that runs around every request. Each layer receives the request, may act on it, passes it inward, then receives the response on the way out and may act on that too. Sessions, authentication, CSRF protection, security headers, and compression are all middleware. Writing your own is the right tool for cross-cutting concerns: request IDs, timing, maintenance mode, tenant detection, and security rules that apply to everything. This note explains the onion model, the built-in layers and why their order matters, how to write function-based and class-based middleware, short-circuiting, exception handling, `LoginRequiredMiddleware`, and async support.

## Core concepts

### The onion model

The `MIDDLEWARE` setting is an ordered list. A request travels through the list from top to bottom until it reaches the view, then the response travels back from bottom to top.

```text
Request  ->  Security -> Session -> Auth -> View
Response <-  Security <- Session <- Auth <- View
```

Each layer wraps the next one, like layers of an onion.

### A middleware is a callable factory

Django builds the chain once at startup. Each middleware is given `get_response`, which is the next layer (or the view), and returns a callable that takes a request and returns a response.

### Short-circuiting

A layer can return a response without calling `get_response`. The inner layers and the view never run, but the outer layers still see the response on its way out. Use this for rejecting requests early (blocked clients, maintenance mode, failed security checks).

### Built-in middleware and order

A typical default list, with the reason each position matters:

- `SecurityMiddleware`: first, so redirects to HTTPS and security headers apply to everything.
- `WhiteNoiseMiddleware` (if used): directly after security, so static files skip the rest of the stack.
- `SessionMiddleware`: before anything that needs `request.session`.
- `CommonMiddleware`: handles `APPEND_SLASH` and other URL normalization.
- `CsrfViewMiddleware`: CSRF protection for unsafe methods.
- `AuthenticationMiddleware`: needs sessions, adds `request.user`.
- `LoginRequiredMiddleware` (Django 5.1): needs `request.user`, so it comes after authentication.
- `MessageMiddleware`: needs sessions.
- `XFrameOptionsMiddleware`: adds clickjacking protection headers.

### Hooks beyond `__call__`

A class-based middleware may also define:

- `process_view(request, view_func, view_args, view_kwargs)`: runs just before the view, knowing which view will run. Return `None` to continue or a response to short-circuit.
- `process_exception(request, exception)`: runs when a view raises. Return `None` to let other handlers act or a response to handle it.
- `process_template_response(request, response)`: runs for responses that have a `render` method.

### LoginRequiredMiddleware

Since Django 5.1, `django.contrib.auth.middleware.LoginRequiredMiddleware` redirects anonymous users to the login page for every view, so you protect the whole site by default and open specific views with the `login_not_required` decorator. This is available on Django 5.2 LTS. On older versions, use `login_required` on each view or a custom middleware.

### Sync and async

A middleware can declare `sync_capable` and `async_capable`. Django adapts layers that cannot handle the mode in use, but each adaptation switches between threads and costs performance, so production middleware that must run under ASGI should support both modes.

## Basic syntax examples

### Function-based middleware

```python
import time
from collections.abc import Callable

from django.http import HttpRequest, HttpResponse

GetResponse = Callable[[HttpRequest], HttpResponse]


def timing_middleware(get_response: GetResponse) -> GetResponse:
    # One-time configuration and initialization goes here.

    def middleware(request: HttpRequest) -> HttpResponse:
        started = time.perf_counter()
        response = get_response(request)
        elapsed_ms = (time.perf_counter() - started) * 1000
        response.headers["X-Response-Time-ms"] = f"{elapsed_ms:.1f}"
        return response

    return middleware
```

### Class-based middleware

```python
class TimingMiddleware:
    def __init__(self, get_response: GetResponse) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        started = time.perf_counter()
        response = self.get_response(request)
        elapsed_ms = (time.perf_counter() - started) * 1000
        response.headers["X-Response-Time-ms"] = f"{elapsed_ms:.1f}"
        return response
```

### Registering middleware

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "apps.core.middleware.TimingMiddleware",
]
```

### Short-circuiting

```python
from django.http import HttpResponseForbidden


class BlockBadClientsMiddleware:
    blocked_agents = frozenset({"badbot"})

    def __init__(self, get_response: GetResponse) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        agent = request.headers.get("User-Agent", "").lower()
        if any(blocked in agent for blocked in self.blocked_agents):
            return HttpResponseForbidden("Forbidden")
        return self.get_response(request)
```

### The extra hooks

```python
import logging
from collections.abc import Callable
from typing import Any

logger = logging.getLogger(__name__)


class AuditMiddleware:
    def __init__(self, get_response: GetResponse) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        return self.get_response(request)

    def process_view(
        self,
        request: HttpRequest,
        view_func: Callable[..., HttpResponse],
        view_args: tuple[Any, ...],
        view_kwargs: dict[str, Any],
    ) -> HttpResponse | None:
        logger.info("Dispatching %s", view_func.__name__)
        return None

    def process_exception(
        self, request: HttpRequest, exception: Exception
    ) -> HttpResponse | None:
        logger.error("View raised %s", type(exception).__name__)
        return None
```

### Protecting the whole site with LoginRequiredMiddleware

```python
MIDDLEWARE = [
    # ... session and authentication middleware first ...
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.auth.middleware.LoginRequiredMiddleware",
]

LOGIN_URL = "accounts:login"
```

Open a public view with the decorator:

```python
from django.contrib.auth.decorators import login_not_required
from django.http import HttpRequest, JsonResponse


@login_not_required
def health(request: HttpRequest) -> JsonResponse:
    return JsonResponse({"status": "ok"})
```

For a class-based view, decorate the `dispatch` method with `method_decorator(login_not_required, name="dispatch")` or wrap the result of `as_view()`.

## Deeper behavior

### What the view's exceptions look like to middleware

Django wraps each middleware layer so that an exception raised below it is converted into a response (404, 403, or 500) before it reaches the layer above. This means a plain `try` around `get_response` in a middleware will not catch a view's exception. Use `process_exception` if you need to see the exception object, and inspect `response.status_code` if you only need to react to failures.

### One instance, many requests

Django creates a middleware instance once and uses it for every request, possibly from many threads. Do not store per-request data on `self`. Use the request object, or a `ContextVar` for values that logging and other code need to read, as the example below does.

### Order is behavior

Putting `CsrfViewMiddleware` before `SessionMiddleware`, or the authentication layer before the session layer, breaks features in confusing ways. When you add a layer, ask what it needs from earlier layers and what later layers need from it. Request-scoped infrastructure such as request IDs belongs at the top, so every other layer and log line can use it.

### Streaming responses

Middleware that reads `response.content` forces the whole body into memory and fails on streaming responses. Check `response.streaming` before touching the body.

### MiddlewareNotUsed

A middleware can raise `MiddlewareNotUsed` from `__init__` to disable itself, for example when a feature flag is off. Django then skips it.

### Async-capable middleware

Under ASGI, a layer that is not async-capable forces Django to run it in a thread and switch context. To support both modes, declare the flags and adapt to what `get_response` is.

```python
from collections.abc import Awaitable, Callable
from typing import cast

from asgiref.sync import iscoroutinefunction, markcoroutinefunction
from django.http import HttpRequest, HttpResponse

SyncGetResponse = Callable[[HttpRequest], HttpResponse]
AsyncGetResponse = Callable[[HttpRequest], Awaitable[HttpResponse]]


class SourceHeaderMiddleware:
    sync_capable = True
    async_capable = True

    def __init__(self, get_response: SyncGetResponse | AsyncGetResponse) -> None:
        self.get_response = get_response
        self._is_async = iscoroutinefunction(get_response)
        if self._is_async:
            markcoroutinefunction(self)

    def __call__(
        self, request: HttpRequest
    ) -> HttpResponse | Awaitable[HttpResponse]:
        if self._is_async:
            return self._acall(request)
        response = cast(SyncGetResponse, self.get_response)(request)
        response.headers["X-Content-Source"] = "app"
        return response

    async def _acall(self, request: HttpRequest) -> HttpResponse:
        response = await cast(AsyncGetResponse, self.get_response)(request)
        response.headers["X-Content-Source"] = "app"
        return response
```

Phase 8 returns to async in depth.

### Legacy style

Older code uses `MiddlewareMixin` with `process_request` and `process_response`. It still works, but the `__call__` style above is clearer and is what new code should use.

## Industry-level snippet

Two layers that nearly every production service has: a request ID middleware that labels every request and every log line with a correlation ID, and a maintenance mode middleware that short-circuits with a proper 503 while still allowing the health check. Both are typed, configurable, and tested.

`apps/core/middleware.py`:

```python
"""Cross-cutting request middleware."""

import logging
import re
import uuid
from collections.abc import Callable
from contextvars import ContextVar

from django.conf import settings
from django.http import HttpRequest, HttpResponse

logger = logging.getLogger(__name__)

GetResponse = Callable[[HttpRequest], HttpResponse]

REQUEST_ID_HEADER = "X-Request-ID"
_VALID_REQUEST_ID = re.compile(r"[A-Za-z0-9._-]{1,64}")

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")


def get_or_create_request_id(incoming: str | None) -> str:
    """Reuse a well-formed incoming ID, otherwise generate a new one.

    Incoming values are validated because they end up in logs and headers.
    """
    if incoming and _VALID_REQUEST_ID.fullmatch(incoming):
        return incoming
    return uuid.uuid4().hex


class RequestIDFilter(logging.Filter):
    """Add the current request ID to every log record."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.__dict__["request_id"] = request_id_var.get()
        return True


class RequestIDMiddleware:
    def __init__(self, get_response: GetResponse) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        request_id = get_or_create_request_id(request.headers.get(REQUEST_ID_HEADER))
        token = request_id_var.set(request_id)
        try:
            response = self.get_response(request)
            logger.info(
                "%s %s -> %d", request.method, request.path, response.status_code
            )
        finally:
            request_id_var.reset(token)
        response.headers[REQUEST_ID_HEADER] = request_id
        return response


class MaintenanceModeMiddleware:
    """Return 503 for every request while ``settings.MAINTENANCE_MODE`` is true."""

    exempt_paths = frozenset({"/health/"})
    retry_after_seconds = 120

    def __init__(self, get_response: GetResponse) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        # Read the setting per request so it can be changed without a restart
        # of the middleware chain (and overridden in tests).
        if (
            getattr(settings, "MAINTENANCE_MODE", False)
            and request.path not in self.exempt_paths
        ):
            logger.warning("Maintenance mode: rejecting %s %s", request.method, request.path)
            return HttpResponse(
                "Service temporarily unavailable",
                status=503,
                content_type="text/plain",
                headers={"Retry-After": str(self.retry_after_seconds)},
            )
        return self.get_response(request)
```

Configuration in `config/settings.py`:

```python
MIDDLEWARE = [
    "apps.core.middleware.RequestIDMiddleware",
    "apps.core.middleware.MaintenanceModeMiddleware",
    "django.middleware.security.SecurityMiddleware",
    # ... the rest of the stack
]

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {"request_id": {"()": "apps.core.middleware.RequestIDFilter"}},
    "formatters": {
        "standard": {
            "format": "%(asctime)s %(levelname)s [%(request_id)s] %(name)s: %(message)s"
        }
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "filters": ["request_id"],
            "formatter": "standard",
        }
    },
    "root": {"handlers": ["console"], "level": "INFO"},
}
```

`apps/core/tests/test_middleware.py`:

```python
import logging

from django.http import HttpRequest, HttpResponse
from django.test import RequestFactory, SimpleTestCase, override_settings

from apps.core.middleware import (
    MaintenanceModeMiddleware,
    RequestIDFilter,
    RequestIDMiddleware,
    request_id_var,
)


def ok_view(request: HttpRequest) -> HttpResponse:
    return HttpResponse("ok")


class RequestIDMiddlewareTests(SimpleTestCase):
    def setUp(self) -> None:
        self.factory = RequestFactory()

    def test_generates_an_id_when_none_is_sent(self) -> None:
        response = RequestIDMiddleware(ok_view)(self.factory.get("/"))
        self.assertRegex(response.headers["X-Request-ID"], r"^[0-9a-f]{32}$")

    def test_reuses_a_valid_incoming_id(self) -> None:
        request = self.factory.get("/", headers={"X-Request-ID": "abc-123"})
        response = RequestIDMiddleware(ok_view)(request)
        self.assertEqual(response.headers["X-Request-ID"], "abc-123")

    def test_replaces_malformed_incoming_ids(self) -> None:
        for bad in ["has spaces", "x" * 65, "semi;colon"]:
            with self.subTest(value=bad):
                request = self.factory.get("/", headers={"X-Request-ID": bad})
                response = RequestIDMiddleware(ok_view)(request)
                self.assertNotEqual(response.headers["X-Request-ID"], bad)

    def test_id_is_visible_to_the_view_and_reset_afterwards(self) -> None:
        seen: list[str] = []

        def view(request: HttpRequest) -> HttpResponse:
            seen.append(request_id_var.get())
            return HttpResponse()

        request = self.factory.get("/", headers={"X-Request-ID": "trace-1"})
        RequestIDMiddleware(view)(request)
        self.assertEqual(seen, ["trace-1"])
        self.assertEqual(request_id_var.get(), "-")

    def test_id_is_reset_when_the_view_raises(self) -> None:
        def view(request: HttpRequest) -> HttpResponse:
            raise RuntimeError("boom")

        with self.assertRaises(RuntimeError):
            RequestIDMiddleware(view)(self.factory.get("/"))
        self.assertEqual(request_id_var.get(), "-")

    def test_log_filter_adds_the_current_id(self) -> None:
        record = logging.LogRecord("name", logging.INFO, "path", 1, "msg", None, None)
        token = request_id_var.set("abc")
        try:
            self.assertTrue(RequestIDFilter().filter(record))
        finally:
            request_id_var.reset(token)
        self.assertEqual(record.__dict__["request_id"], "abc")


class MaintenanceModeMiddlewareTests(SimpleTestCase):
    def setUp(self) -> None:
        self.factory = RequestFactory()
        self.middleware = MaintenanceModeMiddleware(ok_view)

    def test_passes_through_when_disabled(self) -> None:
        response = self.middleware(self.factory.get("/anything/"))
        self.assertEqual(response.status_code, 200)

    @override_settings(MAINTENANCE_MODE=True)
    def test_returns_503_with_retry_after_when_enabled(self) -> None:
        response = self.middleware(self.factory.get("/anything/"))
        self.assertEqual(response.status_code, 503)
        self.assertEqual(response.headers["Retry-After"], "120")

    @override_settings(MAINTENANCE_MODE=True)
    def test_health_check_stays_available(self) -> None:
        response = self.middleware(self.factory.get("/health/"))
        self.assertEqual(response.status_code, 200)
```

## Common mistakes

- Wrong order: for example, placing `AuthenticationMiddleware` before `SessionMiddleware`, so `request.session` is missing.
- Storing per-request data on `self`, which leaks between concurrent requests. Use the request or a `ContextVar`.
- Putting expensive work (database queries, remote calls) in a middleware that runs on every request, including static files and health checks.
- Wrapping `get_response` in `try` and expecting to catch view exceptions. Use `process_exception`.
- Reading `response.content` on streaming responses.
- Trusting incoming headers such as `X-Request-ID` or `X-Forwarded-For` without validation.
- Forgetting that a short-circuiting layer skips everything inside it, including security and session layers that were meant to run.
- Enabling `LoginRequiredMiddleware` without marking the login page, health checks, and webhooks as public.
- Using legacy `MiddlewareMixin` in new code, or forgetting to declare async capability and silently paying a thread-switch cost.
- Adding a middleware to a local settings file and forgetting that production uses a different list.

## Practice exercises

1. Write a function-based middleware that adds `X-Frame-Options: DENY` to every response unless the response already has the header. Test both cases.
2. Write a class-based middleware that rejects requests whose body is larger than one megabyte, using the `Content-Length` header, with a 413 response. Handle a missing or non-numeric header.
3. Add a `process_view` hook that logs the name of the view function for each request and test it with `RequestFactory`.
4. Add a `process_exception` hook that converts a custom `PaymentRequiredError` into a 402 JSON response, and prove with a test that other exceptions are not handled.
5. Reorder the default middleware list in a scratch project so that `AuthenticationMiddleware` comes before `SessionMiddleware`, describe the resulting error, and explain it.
6. Enable `LoginRequiredMiddleware`, mark a health view as `login_not_required`, and write tests showing that anonymous users are redirected from other views.
7. Make `RequestIDMiddleware` async-capable following the pattern in this note, and write one test that calls it through the async test client.

## Summary

Middleware wraps every request in layers: the request travels inward through `MIDDLEWARE` from top to bottom and the response travels back out. Each layer can modify the request or response, short-circuit by returning early, or use the `process_view` and `process_exception` hooks. Order determines behavior because layers depend on each other, so sessions precede authentication and security sits at the top. Instances are shared across requests, so keep per-request data in the request or a `ContextVar`. `LoginRequiredMiddleware` (Django 5.1) protects the whole site by default, and middleware that must run under ASGI should declare and support async mode.
