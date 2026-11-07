## Overview

Django is a Python web framework that supplies the pieces most web applications need: URL routing, request and response objects, templates, an ORM, forms, authentication, an admin interface, security protections, and a test framework. This note explains how those pieces fit together, how a request travels through them, and how Django releases and documentation work. To make the lifecycle concrete, you will build a tiny framework of your own on top of WSGI, which is the interface Django itself uses.

## Core concepts

### Model-Template-View (MTV) and MVC

Django describes its structure as Model-Template-View. The mapping to the classic MVC pattern is:

- Model (MVC) corresponds to Model (MTV): data and rules, stored in the database through the ORM.
- View (MVC) corresponds to Template (MTV): presentation, how data looks.
- Controller (MVC) corresponds to View (MTV): the Python function or class that receives a request and decides the response.
- Django itself, with its URL dispatcher, plays part of the controller role by choosing which view runs.

The names differ, but the idea is the same: separate data, logic, and presentation.

### The request lifecycle

1. A client opens a connection to a web server (Nginx, Caddy) or application server (Gunicorn, Uvicorn).
2. The application server translates the raw HTTP request into a Python call using WSGI or ASGI.
3. Django builds an `HttpRequest` object.
4. The request passes through middleware, top to bottom.
5. The URL resolver matches the path to a view.
6. The view runs. It may query models and render a template.
7. The view returns an `HttpResponse`.
8. The response passes back through middleware, bottom to top.
9. The application server writes the response to the socket.

### Projects versus apps

- A project is the whole website: settings, root URL configuration, and the WSGI and ASGI entry points.
- An app is a focused Python package that does one thing (blog, billing, accounts). It owns its models, views, URLs, and tests.
- One project contains many apps. A well-designed app can be reused in another project.

### Batteries included

In practice this means authentication, sessions, admin, forms, migrations, caching, email, internationalization, and security middleware are all in the box, and they are designed to work together. You can replace most of them, but the defaults are good.

### WSGI and ASGI

- WSGI (Web Server Gateway Interface) is the synchronous standard: one callable, `app(environ, start_response)`, handles one request.
- ASGI (Asynchronous Server Gateway Interface) is the successor that supports async code, WebSockets, and long-lived connections.
- Django ships both entry points: `config/wsgi.py` and `config/asgi.py`.

### Releases, LTS, and deprecation

- Django releases a feature version roughly every eight months. Versions ending in `.2` are Long-Term Support (LTS) releases with about three years of security fixes.
- Django 5.2 is an LTS release. Django 6.0 is the next feature release; features that need it are flagged throughout this roadmap.
- Deprecation policy: a feature is deprecated in one release, warns for at least two feature releases, then is removed. If your code runs without `DeprecationWarning` on version N, it will run on version N+1.

### Reading the documentation

The official docs at `docs.djangoproject.com` are organized into tutorials, topic guides, reference, and how-to guides. Use topic guides to learn a concept, the reference for exact signatures, and the release notes before every upgrade. Always check that the version selector matches your installed version.

## Basic syntax examples

### The WSGI contract

```python
from collections.abc import Iterable
from wsgiref.simple_server import make_server
from wsgiref.types import StartResponse, WSGIEnvironment


def app(environ: WSGIEnvironment, start_response: StartResponse) -> Iterable[bytes]:
    body = b"Hello from WSGI"
    start_response(
        "200 OK",
        [
            ("Content-Type", "text/plain; charset=utf-8"),
            ("Content-Length", str(len(body))),
        ],
    )
    return [body]


if __name__ == "__main__":
    with make_server("127.0.0.1", 8000, app) as server:
        server.serve_forever()
```

Django's `get_wsgi_application()` returns a callable with exactly this shape.

### The ASGI contract

```python
from collections.abc import Awaitable, Callable, MutableMapping
from typing import Any

Scope = MutableMapping[str, Any]
Message = MutableMapping[str, Any]
Receive = Callable[[], Awaitable[Message]]
Send = Callable[[Message], Awaitable[None]]


async def asgi_app(scope: Scope, receive: Receive, send: Send) -> None:
    if scope["type"] != "http":
        raise RuntimeError(f"Unsupported scope type: {scope['type']}")
    await send(
        {
            "type": "http.response.start",
            "status": 200,
            "headers": [(b"content-type", b"text/plain")],
        }
    )
    await send({"type": "http.response.body", "body": b"Hello from ASGI"})
```

### A first look at the Django pieces

These snippets show the shape only. Set-up is covered in note 0.3.

```python
# a view: request in, response out
from django.http import HttpRequest, HttpResponse


def home(request: HttpRequest) -> HttpResponse:
    return HttpResponse("Hello")
```

```python
# a URL pattern: path to view
from django.urls import path

from . import views

urlpatterns = [path("", views.home, name="home")]
```

```html
<!-- a template: presentation -->
<h1>Hello, {{ name }}</h1>
```

## Deeper behavior

### What Django adds on top of WSGI

The WSGI callable receives a raw `environ` dictionary. Django wraps it into `HttpRequest` with parsed GET and POST data, cookies, headers, uploaded files, the session, and the authenticated user. It then builds the middleware chain once at startup and calls it for each request. When a view raises `Http404`, Django catches it and renders the 404 response; when it raises any other exception, Django returns a 500 and logs the error.

### The onion model

Middleware wraps the view like layers of an onion. Each layer can act before the view, act after it, or short-circuit and return its own response without calling the view at all. The order in `MIDDLEWARE` therefore matters: the security middleware belongs near the top, and session middleware must come before authentication middleware, which depends on it.

### Where to look when something breaks

- Wrong page or 404: URL configuration.
- 500 error: the view or code below it; read the traceback bottom up.
- Wrong data: the model or queryset.
- Wrong HTML: the template or the context passed to it.
- Works locally, fails in production: settings, static files, or environment variables.

### Sync and async under the hood

Django can run under both WSGI and ASGI. Under ASGI, a synchronous view is executed in a thread so it does not block the event loop. Async only helps when views spend time waiting on I/O; for ordinary database-heavy pages, a sync view is usually fine. Phase 8 covers this in depth.

## Industry-level snippet

The best way to understand a framework is to build a miniature one. This module implements routing, middleware, error mapping, and logging on top of WSGI, mirroring Django's structure with full type hints and tests.

File `mini_framework.py`:

```python
"""A miniature WSGI framework that mirrors Django's request lifecycle."""

import logging
import time
from collections.abc import Callable, Iterable, Sequence
from dataclasses import dataclass, field
from http import HTTPStatus
from urllib.parse import parse_qs
from wsgiref.types import StartResponse, WSGIEnvironment

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class Request:
    method: str
    path: str
    query: dict[str, list[str]] = field(default_factory=dict)


@dataclass(frozen=True)
class Response:
    status: HTTPStatus
    body: bytes
    content_type: str = "text/plain; charset=utf-8"


View = Callable[[Request], Response]
Middleware = Callable[[View], View]


class HttpError(Exception):
    """Raise from a view to produce a specific HTTP error response."""

    def __init__(self, status: HTTPStatus) -> None:
        super().__init__(status.phrase)
        self.status = status


class Router:
    """Map (path, method) pairs to views."""

    def __init__(self) -> None:
        self._routes: dict[str, dict[str, View]] = {}

    def add(self, path: str, method: str, view: View) -> None:
        if not path.startswith("/"):
            raise ValueError(f"Route path must start with '/': {path!r}")
        self._routes.setdefault(path, {})[method.upper()] = view

    def dispatch(self, request: Request) -> Response:
        methods = self._routes.get(request.path)
        if methods is None:
            raise HttpError(HTTPStatus.NOT_FOUND)
        view = methods.get(request.method)
        if view is None:
            raise HttpError(HTTPStatus.METHOD_NOT_ALLOWED)
        return view(request)


def timing_middleware(get_response: View) -> View:
    """Log how long each request takes, like a request-timing middleware."""

    def middleware(request: Request) -> Response:
        started = time.perf_counter()
        response = get_response(request)
        elapsed_ms = (time.perf_counter() - started) * 1000
        logger.info(
            "%s %s -> %d in %.1f ms",
            request.method,
            request.path,
            response.status.value,
            elapsed_ms,
        )
        return response

    return middleware


class Application:
    """The WSGI callable: builds requests, runs middleware, maps errors."""

    def __init__(
        self, router: Router, middleware: Sequence[Middleware] = ()
    ) -> None:
        handler: View = router.dispatch
        # Wrap in reverse so the first item in the list is the outermost layer.
        for layer in reversed(middleware):
            handler = layer(handler)
        self._handler = handler

    def __call__(
        self, environ: WSGIEnvironment, start_response: StartResponse
    ) -> Iterable[bytes]:
        request = Request(
            method=str(environ["REQUEST_METHOD"]).upper(),
            path=str(environ.get("PATH_INFO", "/")) or "/",
            query=parse_qs(str(environ.get("QUERY_STRING", ""))),
        )
        try:
            response = self._handler(request)
        except HttpError as exc:
            response = Response(exc.status, exc.status.phrase.encode())
        except Exception:
            logger.exception("Unhandled error for %s %s", request.method, request.path)
            response = Response(
                HTTPStatus.INTERNAL_SERVER_ERROR, b"Internal Server Error"
            )

        status_line = f"{response.status.value} {response.status.phrase}"
        start_response(
            status_line,
            [
                ("Content-Type", response.content_type),
                ("Content-Length", str(len(response.body))),
            ],
        )
        return [response.body]
```

File `test_mini_framework.py`:

```python
from collections.abc import Callable
from wsgiref.util import setup_testing_defaults
from wsgiref.types import WSGIEnvironment

import pytest

from mini_framework import (
    Application,
    HttpError,
    Request,
    Response,
    Router,
    View,
)
from http import HTTPStatus


def call(app: Application, method: str, path: str) -> tuple[str, bytes]:
    environ: WSGIEnvironment = {"REQUEST_METHOD": method, "PATH_INFO": path}
    setup_testing_defaults(environ)
    captured: dict[str, str] = {}

    def start_response(
        status: str,
        headers: list[tuple[str, str]],
        exc_info: object = None,
    ) -> Callable[[bytes], object]:
        captured["status"] = status
        return lambda data: None

    body = b"".join(app(environ, start_response))
    return captured["status"], body


def hello(request: Request) -> Response:
    name = request.query.get("name", ["world"])[0]
    return Response(HTTPStatus.OK, f"Hello, {name}".encode())


def broken(request: Request) -> Response:
    raise RuntimeError("boom")


def forbidden(request: Request) -> Response:
    raise HttpError(HTTPStatus.FORBIDDEN)


@pytest.fixture
def app() -> Application:
    router = Router()
    router.add("/hello", "GET", hello)
    router.add("/broken", "GET", broken)
    router.add("/forbidden", "GET", forbidden)
    return Application(router)


def test_routes_to_view(app: Application) -> None:
    assert call(app, "GET", "/hello") == ("200 OK", b"Hello, world")


def test_unknown_path_is_404(app: Application) -> None:
    status, _ = call(app, "GET", "/nope")
    assert status == "404 Not Found"


def test_wrong_method_is_405(app: Application) -> None:
    status, _ = call(app, "POST", "/hello")
    assert status == "405 Method Not Allowed"


def test_http_error_maps_to_its_status(app: Application) -> None:
    status, _ = call(app, "GET", "/forbidden")
    assert status == "403 Forbidden"


def test_unhandled_exception_becomes_500(app: Application) -> None:
    status, body = call(app, "GET", "/broken")
    assert status == "500 Internal Server Error"
    assert b"boom" not in body  # never leak internals


def test_middleware_order_is_outermost_first() -> None:
    calls: list[str] = []

    def make(name: str) -> Callable[[View], View]:
        def layer(get_response: View) -> View:
            def middleware(request: Request) -> Response:
                calls.append(f"{name}:before")
                response = get_response(request)
                calls.append(f"{name}:after")
                return response

            return middleware

        return layer

    router = Router()
    router.add("/hello", "GET", hello)
    app = Application(router, [make("outer"), make("inner")])
    call(app, "GET", "/hello")
    assert calls == ["outer:before", "inner:before", "inner:after", "outer:after"]


def test_router_rejects_relative_paths() -> None:
    with pytest.raises(ValueError):
        Router().add("hello", "GET", hello)
```

Notice that the `Application` class maps almost one-to-one onto Django's `WSGIHandler`, `Router` onto the URL resolver, and `Middleware` onto `MIDDLEWARE`. In this miniature, the error handling sits outside the middleware, so an exception raised in a view is not seen by the timing layer. Django solves this by converting exceptions into responses inside each middleware layer.

## Common mistakes

- Believing the framework is magic. It is ordinary Python callables, and reading its source is allowed and useful.
- Putting everything in one giant app. Split by responsibility early; it is cheap then and expensive later.
- Using `runserver` in production. It is a development tool only.
- Reading documentation for the wrong version. Always confirm the version in the docs sidebar.
- Ignoring `DeprecationWarning` output until an upgrade breaks the project.
- Assuming async is automatically faster. For CPU-bound or database-bound work it often changes nothing.
- Confusing "project" and "app", then creating a new project for every feature.

## Practice exercises

1. Draw the request lifecycle from memory, listing nine steps from socket to socket. Compare with the list in this note.
2. Add a `/health` route to the mini framework that returns the plain text `ok`, and write a test for it.
3. Write a middleware that adds a request counter and logs it every ten requests. Decide where shared state should live and why a module-level global is risky with multiple worker processes.
4. Add a `Response` header mechanism to the mini framework so views can set custom headers. Update `Application.__call__` and the tests.
5. Extend `Router` to support one path parameter such as `/users/<id>` using a regular expression, passing the captured value to the view. Reject invalid patterns with a clear error.
6. Run the WSGI example with `wsgiref` and then open the Django release notes for the latest version. List three features that were deprecated and the version in which each will be removed.
7. Write a one-paragraph explanation, in your own words, of why a project contains apps and what makes a good app boundary.

## Summary

Django is a batteries-included framework built on WSGI and ASGI. A request enters through an application server, becomes an `HttpRequest`, passes through middleware, is matched by the URL resolver to a view, and returns as an `HttpResponse` through the same middleware in reverse. MTV is Django's version of MVC. A project holds configuration; apps hold focused functionality. Releases follow a predictable schedule with LTS versions every few cycles, and the deprecation policy makes upgrades manageable if you watch warnings. Building a miniature framework shows that every Django concept is plain Python at heart.
