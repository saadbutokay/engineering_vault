## Overview

A view is where your application decides what to do with a request. A function-based view is the simplest form: a plain Python function that takes an `HttpRequest` and returns an `HttpResponse`. Understanding views as ordinary functions removes a great deal of the mystery from Django. This note covers the request object, the response classes, the shortcut functions, method restriction, status codes, and how to structure a view that validates input and reports errors consistently.

## Core concepts

### What a view is

A view is any callable that accepts an `HttpRequest` as its first argument, plus any values captured from the URL, and returns an `HttpResponse` (or raises an exception such as `Http404`). Nothing else is required.

### The HttpRequest object

- `request.method`: the HTTP method in uppercase (`"GET"`, `"POST"`).
- `request.GET`: query string parameters, a `QueryDict`.
- `request.POST`: form-encoded body parameters, a `QueryDict`.
- `request.body`: the raw request body as bytes.
- `request.headers`: a case-insensitive mapping of HTTP headers.
- `request.META`: a dictionary of server and header values, with headers prefixed by `HTTP_` and uppercased (`HTTP_USER_AGENT`).
- `request.COOKIES`: cookies sent by the browser.
- `request.FILES`: uploaded files (Phase 3).
- `request.path` and `request.get_full_path()`: the requested path, with or without the query string.
- `request.user` and `request.session`: attached by middleware once authentication and session middleware are enabled.
- `request.content_type`: the media type from the `Content-Type` header, without parameters such as charset.

### HttpResponse and its subclasses

- `HttpResponse(content, status=200, content_type=...)`: the base class.
- `JsonResponse(data, status=...)`: serializes a dictionary to JSON with the right content type.
- `HttpResponseRedirect` (302) and `HttpResponsePermanentRedirect` (301).
- `HttpResponseNotFound` (404), `HttpResponseBadRequest` (400), `HttpResponseNotAllowed` (405), `HttpResponseForbidden` (403), `HttpResponseServerError` (500).
- `FileResponse` and `StreamingHttpResponse` for large content (Phase 8).

### Shortcut functions

- `render(request, template_name, context)`: render a template into an `HttpResponse`.
- `redirect(to, *args, **kwargs)`: build a redirect from a URL name, a model with `get_absolute_url`, or a URL string.
- `get_object_or_404(Model, **lookups)`: fetch one object or raise `Http404`.
- `get_list_or_404(...)`: the same for a queryset that must not be empty.

### Restricting methods

Decorators from `django.views.decorators.http`:

- `require_GET`, `require_POST`, `require_safe` (GET and HEAD).
- `require_http_methods(["GET", "POST"])` for custom combinations. Disallowed methods receive a 405 response with an `Allow` header.

### Status codes

Return the status that describes the outcome: 200 for success, 201 for a created resource, 204 for success with no body, 400 for malformed input, 404 for missing resources, 405 for wrong method, 415 for an unsupported content type. The standard library `http.HTTPStatus` enum gives readable names.

## Basic syntax examples

### The smallest view

```python
from django.http import HttpRequest, HttpResponse


def hello(request: HttpRequest) -> HttpResponse:
    return HttpResponse("Hello")
```

### Reading query parameters

```python
def greet(request: HttpRequest) -> HttpResponse:
    name = request.GET.get("name", "world")
    return HttpResponse(f"Hello, {name}")
```

A request to `/greet/?name=Ada` returns `Hello, Ada`. `request.GET.get` returns the default when the key is missing, while `request.GET["name"]` raises an error.

### Reading form data

```python
from django.http import HttpRequest, HttpResponse, HttpResponseBadRequest
from django.views.decorators.http import require_POST


@require_POST
def subscribe(request: HttpRequest) -> HttpResponse:
    email = request.POST.get("email", "").strip()
    if "@" not in email:
        return HttpResponseBadRequest("A valid email address is required")
    return HttpResponse(f"Subscribed {email}", status=201)
```

Phase 3 replaces this manual validation with forms.

### Returning JSON

```python
from django.http import HttpRequest, JsonResponse


def status(request: HttpRequest) -> JsonResponse:
    return JsonResponse({"status": "ok", "version": 1})
```

### Rendering a template, redirecting, and 404

```python
from django.http import HttpRequest, HttpResponse
from django.shortcuts import get_object_or_404, redirect, render

from .models import Task


def task_page(request: HttpRequest, task_id: int) -> HttpResponse:
    task = get_object_or_404(Task, pk=task_id)
    return render(request, "tasks/task_page.html", {"task": task})


def home(request: HttpRequest) -> HttpResponse:
    return redirect("tasks:list")
```

### Restricting methods

```python
from django.views.decorators.http import require_http_methods


@require_http_methods(["GET", "POST"])
def collection(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        return HttpResponse(status=201)
    return HttpResponse("list")
```

## Deeper behavior

### QueryDict and repeated keys

`request.GET` and `request.POST` are `QueryDict` objects, which allow several values per key. `get()` returns the last value; `getlist()` returns all of them.

```python
# /search/?tag=python&tag=django
tags = request.GET.getlist("tag")  # ['python', 'django']
```

### request.body and request.POST

For form-encoded bodies, Django parses the body into `request.POST` when you access it. For JSON, `request.POST` is empty and you must read and parse `request.body` yourself. Accessing `request.body` after the stream has been consumed for a multipart upload raises `RawPostDataException`, so read it before touching `request.POST` or `request.FILES` if you need the raw bytes.

### Headers

`request.headers["Accept"]` is case-insensitive and is the modern way to read headers. The older `request.META["HTTP_ACCEPT"]` form is still needed for server-level values such as `REMOTE_ADDR`. Never trust headers such as `X-Forwarded-For` unless a proxy you control sets them.

### Response objects behave like dictionaries for headers

```python
response = HttpResponse("created", status=201)
response["Location"] = "/tasks/7/"
response.headers["Cache-Control"] = "no-store"
response.set_cookie("seen", "1", max_age=3600, httponly=True, samesite="Lax")
```

You can also pass headers at construction: `HttpResponse(status=503, headers={"Retry-After": "120"})`.

### JsonResponse and lists

By default `JsonResponse` accepts only dictionaries, as a protection against an old JSON hijacking technique. To return a top-level list pass `safe=False`, but wrapping lists in an object such as `{"results": [...]}` is the better design because you can add fields later without breaking clients.

### Redirect details

`redirect()` returns a 302 by default and a 301 with `permanent=True`. After a successful form POST, redirect so that a browser refresh does not resubmit the form (Post/Redirect/Get, Phase 3).

### CSRF and POST

For browser sessions, Django's CSRF middleware rejects POST, PUT, PATCH, and DELETE requests without a valid token. The test client skips this check by default so that tests stay simple. APIs that use session authentication must send the token; APIs that use token headers are covered in Phase 7.

### Exceptions in views

Raising `Http404` produces the 404 page. Raising `PermissionDenied` produces a 403. Any other uncaught exception produces a 500 and is logged. Catch exceptions you can handle meaningfully and let the rest propagate.

## Industry-level snippet

A small JSON API for tasks: it validates the content type and JSON body, reports errors in one consistent shape, bounds the `limit` parameter, returns `201` with a `Location` header on creation, restricts methods, logs important events, and is tested.

`apps/tasks/models.py`:

```python
from django.db import models


class Task(models.Model):
    title = models.CharField(max_length=200)
    is_done = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at", "-id"]

    def __str__(self) -> str:
        return self.title
```

`apps/tasks/views.py`:

```python
import json
import logging
from collections.abc import Callable
from functools import wraps
from http import HTTPStatus
from typing import Any, ParamSpec

from django.http import HttpRequest, HttpResponse, JsonResponse
from django.shortcuts import get_object_or_404
from django.urls import reverse
from django.views.decorators.http import require_http_methods

from .models import Task

logger = logging.getLogger(__name__)

P = ParamSpec("P")

MAX_TITLE_LENGTH = 200
DEFAULT_LIMIT = 20
MAX_LIMIT = 100


class ApiError(Exception):
    """An error that maps directly to a JSON error response."""

    def __init__(self, status: HTTPStatus, message: str) -> None:
        super().__init__(message)
        self.status = status
        self.message = message


def handle_api_errors(view: Callable[P, HttpResponse]) -> Callable[P, HttpResponse]:
    """Convert ``ApiError`` into a JSON response with the right status."""

    @wraps(view)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> HttpResponse:
        try:
            return view(*args, **kwargs)
        except ApiError as exc:
            return JsonResponse({"error": exc.message}, status=exc.status)

    return wrapper


def parse_json_object(request: HttpRequest) -> dict[str, Any]:
    if request.content_type != "application/json":
        raise ApiError(
            HTTPStatus.UNSUPPORTED_MEDIA_TYPE,
            "Content-Type must be application/json",
        )
    try:
        payload = json.loads(request.body)
    except json.JSONDecodeError as exc:
        raise ApiError(HTTPStatus.BAD_REQUEST, "Body is not valid JSON") from exc
    if not isinstance(payload, dict):
        raise ApiError(HTTPStatus.BAD_REQUEST, "Body must be a JSON object")
    return payload


def validate_title(payload: dict[str, Any]) -> str:
    title = payload.get("title")
    if not isinstance(title, str) or not title.strip():
        raise ApiError(HTTPStatus.BAD_REQUEST, "title must be a non-empty string")
    title = title.strip()
    if len(title) > MAX_TITLE_LENGTH:
        raise ApiError(
            HTTPStatus.BAD_REQUEST,
            f"title must be at most {MAX_TITLE_LENGTH} characters",
        )
    return title


def parse_limit(raw: str | None) -> int:
    if raw is None:
        return DEFAULT_LIMIT
    try:
        limit = int(raw)
    except ValueError as exc:
        raise ApiError(HTTPStatus.BAD_REQUEST, "limit must be an integer") from exc
    if not 1 <= limit <= MAX_LIMIT:
        raise ApiError(
            HTTPStatus.BAD_REQUEST,
            f"limit must be between 1 and {MAX_LIMIT}",
        )
    return limit


def serialize_task(task: Task) -> dict[str, Any]:
    return {"id": task.pk, "title": task.title, "is_done": task.is_done}


def create_task(request: HttpRequest) -> HttpResponse:
    title = validate_title(parse_json_object(request))
    task = Task.objects.create(title=title)
    logger.info("Created task %s", task.pk)
    response = JsonResponse(serialize_task(task), status=HTTPStatus.CREATED)
    response["Location"] = reverse("tasks:detail", kwargs={"task_id": task.pk})
    return response


def list_tasks(request: HttpRequest) -> HttpResponse:
    limit = parse_limit(request.GET.get("limit"))
    tasks = Task.objects.all()[:limit]
    return JsonResponse({"results": [serialize_task(task) for task in tasks]})


@require_http_methods(["GET", "POST"])
@handle_api_errors
def task_collection(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        return create_task(request)
    return list_tasks(request)


@require_http_methods(["GET", "DELETE"])
def task_detail(request: HttpRequest, task_id: int) -> HttpResponse:
    task = get_object_or_404(Task, pk=task_id)
    if request.method == "DELETE":
        task.delete()
        logger.info("Deleted task %s", task_id)
        return HttpResponse(status=HTTPStatus.NO_CONTENT)
    return JsonResponse(serialize_task(task))
```

`apps/tasks/urls.py`:

```python
from django.urls import path

from . import views

app_name = "tasks"

urlpatterns = [
    path("", views.task_collection, name="collection"),
    path("<int:task_id>/", views.task_detail, name="detail"),
]
```

`apps/tasks/tests.py`:

```python
import json

from django.http import HttpResponse
from django.test import TestCase
from django.urls import reverse

from .models import Task


class TaskApiTests(TestCase):
    def setUp(self) -> None:
        self.url = reverse("tasks:collection")

    def _post(self, payload: object) -> HttpResponse:
        return self.client.post(
            self.url, data=json.dumps(payload), content_type="application/json"
        )

    def test_create_returns_201_and_location(self) -> None:
        response = self._post({"title": "  Write notes  "})
        self.assertEqual(response.status_code, 201)
        task = Task.objects.get()
        self.assertEqual(task.title, "Write notes")
        self.assertEqual(
            response["Location"], reverse("tasks:detail", kwargs={"task_id": task.pk})
        )

    def test_wrong_content_type_is_415(self) -> None:
        response = self.client.post(self.url, data={"title": "x"})
        self.assertEqual(response.status_code, 415)

    def test_invalid_json_is_400(self) -> None:
        response = self.client.post(
            self.url, data="{not json", content_type="application/json"
        )
        self.assertEqual(response.status_code, 400)
        self.assertEqual(response.json(), {"error": "Body is not valid JSON"})

    def test_non_object_json_is_400(self) -> None:
        self.assertEqual(self._post(["a"]).status_code, 400)

    def test_invalid_titles_are_400(self) -> None:
        for title in ["", "   ", 5, None, "x" * 201]:
            with self.subTest(title=title):
                self.assertEqual(self._post({"title": title}).status_code, 400)
        self.assertEqual(Task.objects.count(), 0)

    def test_list_respects_limit_and_order(self) -> None:
        for index in range(3):
            Task.objects.create(title=f"task {index}")
        response = self.client.get(self.url, {"limit": 2})
        titles = [item["title"] for item in response.json()["results"]]
        self.assertEqual(titles, ["task 2", "task 1"])

    def test_invalid_limit_is_400(self) -> None:
        for limit in ["abc", "0", "101"]:
            with self.subTest(limit=limit):
                self.assertEqual(self.client.get(self.url, {"limit": limit}).status_code, 400)

    def test_disallowed_method_is_405_with_allow_header(self) -> None:
        response = self.client.put(self.url)
        self.assertEqual(response.status_code, 405)
        self.assertIn("POST", response["Allow"])

    def test_detail_get_and_delete(self) -> None:
        task = Task.objects.create(title="Keep")
        url = reverse("tasks:detail", kwargs={"task_id": task.pk})
        self.assertEqual(self.client.get(url).json()["title"], "Keep")
        self.assertEqual(self.client.delete(url).status_code, 204)
        self.assertEqual(self.client.get(url).status_code, 404)
```

The view functions stay short because validation, parsing, and serialization live in small helpers. The decorator guarantees that every `ApiError` becomes a consistent JSON body, and the unit under test is easy to reason about.

## Common mistakes

- Forgetting to return a response, so the view returns `None` and Django raises `ValueError`.
- Using `request.POST["key"]` and crashing with a 400-level error when the key is missing, instead of using `.get()` and validating.
- Parsing JSON from `request.POST`, which is empty for JSON bodies.
- Trusting `request.GET` values without converting and bounding them.
- Returning 200 with an error message in the body, which hides failures from clients and monitoring.
- Changing state in a view that allows GET.
- Reading `request.body` after `request.POST` on a multipart request.
- Doing heavy business logic directly in the view instead of calling a service function.
- Catching `Exception` broadly and returning a generic 200, which swallows real bugs.
- Hard-coding the `Location` URL instead of using `reverse`.

## Practice exercises

1. Write a view `add(request)` that reads integers `a` and `b` from the query string and returns their sum as JSON, returning 400 with a clear message for missing or non-integer values.
2. Add a `PATCH` method to `task_detail` that accepts `{"is_done": true}` or `{"title": "..."}`, validates the fields, and returns the updated task. Write tests for each branch.
3. Use `require_safe` on a view and verify with the test client that HEAD works and POST returns 405.
4. Write a view that sets a cookie named `theme` from a validated query parameter (`light` or `dark`) and reads it on the next request. Set `HttpOnly` and `SameSite` appropriately and explain your choice.
5. Return a top-level JSON list with `safe=False`, then redesign the response as an object with a `results` key and explain the advantage.
6. Add an `ETag` header to the task detail response and return 304 when the client sends a matching `If-None-Match` header. Test both paths.
7. Rewrite `serialize_task` so it also returns the creation time in ISO 8601 format, and write a test that freezes or controls the clock.

## Summary

A function-based view is a function from `HttpRequest` to `HttpResponse`. The request exposes the method, query parameters, body, headers, cookies, and (through middleware) the user and session. Responses come in several flavors, with `JsonResponse`, `render`, `redirect`, and `get_object_or_404` covering most needs. Restrict methods with decorators, return honest status codes, validate and bound every input, and keep views thin by moving parsing, validation, and business operations into small functions. Consistent error handling, through exceptions mapped by a decorator, keeps APIs predictable.
