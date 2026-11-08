## Overview

Every developer spends a large share of their time reading errors. The difference between a beginner and an experienced engineer is rarely that the experienced one writes fewer bugs; it is that they read the evidence calmly and narrow the search quickly. This note teaches you to read Django's debug page and Python tracebacks, how Django turns errors into 404 and 500 responses, how to write custom error pages that behave correctly in production, what the most common Django failures mean and how to fix them, when to use `print`, logging, or the debugger, and a systematic procedure for debugging a failing request.

## Core concepts

### The debug error page

When `DEBUG = True` and a view raises an exception, Django shows a detailed page with the exception type and message, the full traceback with source lines and local variables, the request data, and the settings (with sensitive values masked). This page is invaluable in development and dangerous in production, because it reveals code, paths, and configuration. That is why `DEBUG` must be false on any public server.

### How Django handles errors in production

With `DEBUG = False`:

- `Http404` (and `Resolver404` when no URL matches) produces a 404 response using the `404.html` template.
- `PermissionDenied` produces 403 using `403.html`.
- `SuspiciousOperation` and `BadRequest` produce 400 using `400.html`.
- Any other exception produces 500 using `500.html`, is logged to the `django.request` logger, and can email administrators if configured.

### Custom error pages

Create `404.html`, `403.html`, `400.html`, and `500.html` in a template directory at the root of your template search path. To take over the logic, assign handler paths in the root urlconf:

- `handler404 = "path.to.view"` taking `(request, exception)`.
- `handler403` and `handler400` taking `(request, exception)`.
- `handler500` taking `(request)`.

The 500 handler runs when things are already broken, so it must be simple and must not depend on the database, sessions, or complex context processors.

### Reading a traceback

A Python traceback lists calls from the oldest at the top to the most recent at the bottom, and ends with the exception type and message. Read it from the bottom up:

1. The last line is what went wrong.
2. The lines just above it show where it was raised, often inside Django or a library.
3. Walk upward until you reach the first file that belongs to your project. That line is usually where the bug is or where you passed bad input to the library.
4. If you see "The above exception was the direct cause of the following exception" or "During handling of the above exception, another exception occurred", the first exception listed is the root cause.

### The common failures

- `ImproperlyConfigured`: a setting is missing or invalid (for example no `SECRET_KEY`, or a bad `DATABASES` entry).
- `TemplateDoesNotExist`: the template name is wrong, the app is not in `INSTALLED_APPS`, or the file is in the wrong folder.
- `NoReverseMatch`: `reverse()` or `{% url %}` found no route for the given name and arguments.
- `OperationalError`: the database cannot be reached, a table or column is missing, or the database is locked.
- `DisallowedHost`: the request's `Host` header is not in `ALLOWED_HOSTS`.
- `CSRF verification failed`: a POST arrived without a valid CSRF token or from an untrusted origin.
- `InconsistentMigrationHistory`: the migration records in the database disagree with the migration files on disk.

### print, logging, and the debugger

- `print` is quick for a first look, but easy to forget, unstructured, and absent from production logs. Never commit it.
- Logging records events with levels and context, works in production, and can be filtered and shipped to tools.
- The debugger (`breakpoint()` with `pdb`, or your editor) pauses execution so you can inspect live state and step through the code. It is the best tool when you do not yet know what the data looks like.

### A systematic way to debug

1. Reproduce the problem reliably, ideally as a failing test.
2. Read the entire error message and traceback before changing anything.
3. Find the first line of your own code in the traceback.
4. Form one hypothesis about the cause.
5. Test the hypothesis cheaply: inspect values, add a log line, or use the debugger.
6. Fix the cause, not the symptom.
7. Keep the failing test as a regression test.

## Basic syntax examples

### Reading a real traceback

```text
Traceback (most recent call last):
  File ".../django/core/handlers/exception.py", line 55, in inner
    response = get_response(request)
  File ".../django/core/handlers/base.py", line 197, in _get_response
    response = wrapped_callback(request, *callback_args, **callback_kwargs)
  File "/app/apps/articles/views.py", line 14, in article_detail
    return render(request, "articles/article_detail.html", {"article": article})
  File ".../django/shortcuts.py", line 25, in render
    content = loader.render_to_string(template_name, context, request, using=using)
  File ".../django/template/loader.py", line 61, in render_to_string
    template = get_template(template_name, using=using)
  File ".../django/template/loader.py", line 19, in get_template
    raise TemplateDoesNotExist(", ".join(template_name_list), chain=chain)
django.template.exceptions.TemplateDoesNotExist: articles/article_detail.html
```

Reading it from the bottom: the exception is `TemplateDoesNotExist` for `articles/article_detail.html`. Walking up, the first line in project code is `apps/articles/views.py`, line 14. So the view asked for a template that Django cannot find. Check the file's location (`apps/articles/templates/articles/article_detail.html`), the spelling, and that the app is installed.

### Simple custom error templates

`templates/404.html`:

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>Page not found</title></head>
  <body>
    <h1>Page not found</h1>
    <p>We could not find what you were looking for.</p>
  </body>
</html>
```

`templates/500.html` should be static and self-contained:

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>Server error</title></head>
  <body>
    <h1>Server error</h1>
    <p>Something went wrong on our side. Please try again later.</p>
  </body>
</html>
```

Test them locally by running with `DEBUG = False` and `ALLOWED_HOSTS = ["localhost", "127.0.0.1"]`. Static files are then not served by `runserver` unless you pass `--insecure`.

### Using the debugger

```python
from django.http import HttpRequest, HttpResponse


def article_detail(request: HttpRequest, slug: str) -> HttpResponse:
    breakpoint()  # execution stops here, in the terminal running runserver
    ...
```

Useful `pdb` commands:

- `n`: run the next line.
- `s`: step into a function call.
- `c`: continue until the next breakpoint.
- `p expression` or `pp expression`: print a value.
- `l`: list the source around the current line.
- `w`: show the call stack.
- `q`: quit.

Remove every `breakpoint()` before committing. Ruff's `T100` rule catches forgotten breakpoints, and `T20` catches `print`.

### Logging instead of printing

```python
import logging

logger = logging.getLogger(__name__)


def import_rows(rows: list[dict[str, str]]) -> int:
    imported = 0
    for index, row in enumerate(rows):
        try:
            validate_row(row)
        except ValueError:
            logger.warning("Skipping row %d: invalid data", index, exc_info=True)
            continue
        imported += 1
    logger.info("Imported %d of %d rows", imported, len(rows))
    return imported


def validate_row(row: dict[str, str]) -> None:
    if not row.get("email"):
        raise ValueError("email is required")
```

Pass values as arguments (`%d`, `%s`) rather than building the string with an f-string, so that formatting happens only when the message is actually emitted. Use `logger.exception(...)` inside an `except` block to record the traceback.

### Catching the error where it happens

```python
from django.test import Client


def reproduce() -> None:
    client = Client(raise_request_exception=True)
    client.get("/articles/does-not-exist/")
```

The test client re-raises view exceptions by default, so a failing test shows the real traceback instead of a 500 page.

## Deeper behavior

### What each common error usually means

- `ImproperlyConfigured`: read the message, which names the setting. If it happens only on a server, the environment variable is probably missing there. Check that `DJANGO_SETTINGS_MODULE` points at the module you expect.
- `TemplateDoesNotExist`: the debug page lists every location Django searched. If your file is not among them, the app is not installed or `DIRS` is wrong. If a template inherits from a missing parent, the error names the parent.
- `NoReverseMatch`: the message lists the patterns that were tried. Compare the name (including the namespace), the argument names, and whether the converter accepts the value you passed. A template that uses `{% url %}` with a variable that is empty is a frequent cause.
- `OperationalError: no such table`: migrations have not been applied to this database. `OperationalError: could not connect to server`: the database is down, the host or port is wrong, or the container is not ready (see note 0.10).
- `DisallowedHost`: add the host to `ALLOWED_HOSTS`. If the host looks strange, it may be a scanner, which you can ignore.
- `CSRF verification failed`: missing `{% csrf_token %}`, a cookie blocked or expired, a request from an origin not listed in `CSRF_TRUSTED_ORIGINS`, or a misconfigured proxy that hides HTTPS from Django.
- `InconsistentMigrationHistory`: typically caused by merging branches, deleting migrations that were already applied, or changing a custom user model after the first migration. Do not edit the database history casually; compare `showmigrations` output with the files and fix the cause.

### Error messages inside templates

Errors in templates appear as exceptions in the traceback, with the template line highlighted on the debug page. Missing variables do not raise errors, they render nothing, so empty output is a clue that a variable name or context key is wrong.

### The 500 handler is special

Django calls the 500 handler when something has already failed, perhaps the database or a context processor. If the handler itself raises, the user sees a bare error. Keep it independent: render a static template without context processors and fall back to a plain string if even that fails.

### Logging configuration for debugging

Django logs requests and errors through named loggers. `django.request` records 4xx and 5xx responses, and `django.db.backends` (at `DEBUG` level) prints every SQL query, which is handy for investigating performance problems. Phase 5 covers the `LOGGING` setting in detail.

### Warnings as errors

Run tests with `python -W error::DeprecationWarning manage.py test` to turn deprecation warnings into failures, and `python -X dev` to enable extra runtime checks. Both catch problems before an upgrade does.

### Bisecting a regression

When something worked last week and fails now, `git bisect` finds the commit that introduced the bug by repeatedly testing the midpoint between a known good and a known bad commit. A failing test makes this fully automatic.

### Production debugging without the debug page

Rely on logs with request IDs (note 1.6), error tracking tools such as Sentry (Phase 11), and reproducing the issue locally with similar data. Never enable `DEBUG` on a live site to investigate.

## Industry-level snippet

Production error pages should be safe and consistent. This module returns JSON to API clients and HTML to browsers, never leaks details, escapes the requested path, and keeps the 500 handler independent of everything that could be broken. It uses the status codes and request handling from the earlier notes.

`apps/core/error_views.py`:

```python
"""Custom error handlers that are safe to use in production."""

import logging
from http import HTTPStatus

from django.http import HttpRequest, HttpResponse, JsonResponse
from django.template import TemplateDoesNotExist, TemplateSyntaxError, loader

logger = logging.getLogger(__name__)

API_PREFIX = "/api/"


def wants_json(request: HttpRequest) -> bool:
    """Return True for API paths and for clients that prefer JSON over HTML."""
    if request.path.startswith(API_PREFIX):
        return True
    accept = request.headers.get("Accept", "")
    return "application/json" in accept and "text/html" not in accept


def error_response(
    request: HttpRequest,
    status: HTTPStatus,
    code: str,
    template_name: str,
) -> HttpResponse:
    """Build an error response without depending on request-scoped machinery."""
    if wants_json(request):
        return JsonResponse({"error": code, "status": status.value}, status=status)

    # Render without the request, so no context processors can fail here.
    try:
        body = loader.render_to_string(template_name, {"request_path": request.path})
    except (TemplateDoesNotExist, TemplateSyntaxError):
        logger.exception("Could not render %s", template_name)
        body = f"<h1>{status.phrase}</h1>"
    return HttpResponse(body, status=status)


def handler400(request: HttpRequest, exception: Exception) -> HttpResponse:
    return error_response(request, HTTPStatus.BAD_REQUEST, "bad_request", "errors/400.html")


def handler403(request: HttpRequest, exception: Exception) -> HttpResponse:
    return error_response(request, HTTPStatus.FORBIDDEN, "forbidden", "errors/403.html")


def handler404(request: HttpRequest, exception: Exception) -> HttpResponse:
    return error_response(request, HTTPStatus.NOT_FOUND, "not_found", "errors/404.html")


def handler500(request: HttpRequest) -> HttpResponse:
    return error_response(
        request,
        HTTPStatus.INTERNAL_SERVER_ERROR,
        "server_error",
        "errors/500.html",
    )
```

Registration in `config/urls.py`:

```python
handler400 = "apps.core.error_views.handler400"
handler403 = "apps.core.error_views.handler403"
handler404 = "apps.core.error_views.handler404"
handler500 = "apps.core.error_views.handler500"
```

`templates/errors/404.html` (the others follow the same pattern, and `500.html` should use no variables):

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>Page not found</title></head>
  <body>
    <h1>Page not found</h1>
    <p>The address <code>{{ request_path }}</code> does not exist.</p>
  </body>
</html>
```

`templates/errors/500.html`:

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>Server error</title></head>
  <body>
    <h1>Server error</h1>
    <p>Something went wrong on our side. Please try again later.</p>
  </body>
</html>
```

`apps/core/tests/test_error_views.py`:

```python
from unittest import mock

from django.template import TemplateDoesNotExist
from django.test import RequestFactory, SimpleTestCase

from apps.core.error_views import handler500, wants_json


class NotFoundTests(SimpleTestCase):
    def test_browsers_get_the_html_page(self) -> None:
        response = self.client.get("/definitely-missing/")
        self.assertContains(response, "Page not found", status_code=404)

    def test_json_clients_get_json(self) -> None:
        response = self.client.get(
            "/definitely-missing/", headers={"Accept": "application/json"}
        )
        self.assertEqual(response.status_code, 404)
        self.assertEqual(response.json(), {"error": "not_found", "status": 404})

    def test_api_paths_always_get_json(self) -> None:
        response = self.client.get("/api/missing/")
        self.assertEqual(response.json()["error"], "not_found")

    def test_requested_path_is_escaped(self) -> None:
        response = self.client.get("/%3Cscript%3Ealert(1)%3C/script%3E/")
        self.assertNotContains(response, "<script>", status_code=404)


class ServerErrorTests(SimpleTestCase):
    def setUp(self) -> None:
        self.factory = RequestFactory()

    def test_html_response_is_a_500(self) -> None:
        response = handler500(self.factory.get("/boom/"))
        self.assertEqual(response.status_code, 500)
        self.assertIn(b"Server error", response.content)

    def test_falls_back_when_template_is_missing(self) -> None:
        with mock.patch(
            "apps.core.error_views.loader.render_to_string",
            side_effect=TemplateDoesNotExist("errors/500.html"),
        ):
            response = handler500(self.factory.get("/boom/"))
        self.assertEqual(response.status_code, 500)
        self.assertIn(b"Internal Server Error", response.content)

    def test_json_clients_get_json(self) -> None:
        request = self.factory.get("/boom/", headers={"Accept": "application/json"})
        response = handler500(request)
        self.assertEqual(response.status_code, 500)
        self.assertEqual(response.content, b'{"error": "server_error", "status": 500}')


class WantsJsonTests(SimpleTestCase):
    def test_prefers_html_when_both_are_accepted(self) -> None:
        request = RequestFactory().get(
            "/page/", headers={"Accept": "text/html,application/json"}
        )
        self.assertFalse(wants_json(request))

    def test_no_accept_header_means_html(self) -> None:
        self.assertFalse(wants_json(RequestFactory().get("/page/")))
```

The handlers never touch the database, sessions, or the authenticated user, so they keep working when those are the things that failed.

## Common mistakes

- Leaving `DEBUG = True` in production, which exposes settings, code, and data on every error.
- Reading only the last line of the traceback, or only the first, instead of finding the first line of your own code.
- Changing code at random hoping the error disappears, instead of forming and testing a hypothesis.
- Fixing a symptom, such as adding a `try` that swallows an exception, instead of the cause.
- Committing `print` and `breakpoint()` calls.
- Catching `Exception` and returning a generic success response, which hides bugs.
- Writing a 500 handler that uses the database, the user, or complex context processors.
- Testing custom error pages only with `DEBUG = True`, where Django shows the debug page instead.
- Pasting an error into a search engine without reading which line of your project triggered it.
- Deleting migration files to "fix" `InconsistentMigrationHistory`.
- Logging secrets, tokens, or personal data while debugging.

## Practice exercises

1. Introduce each of the seven common errors on purpose in a scratch project. For every one, record the message, the first line of project code in the traceback, and the fix.
2. Take the traceback shown in this note and write out, in your own words, what happened at each line from the bottom to the first project line.
3. Add a `breakpoint()` to a view and use `pdb` to inspect `request.GET`, step into a helper function, and print a local variable before continuing. List the commands you used.
4. Run the project with `DEBUG = False`, request a missing page, and compare it with the same request under `DEBUG = True`. Then add custom `404.html` and `500.html` templates.
5. Replace three `print` statements in a function with appropriately leveled log calls, and write a test using `assertLogs` that proves a warning is logged for invalid input.
6. Add handlers for 403 and 400 to the error views module and write tests for them using the test client and `RequestFactory`. For 403, create a view that raises `PermissionDenied`.
7. Write a one-page debugging checklist you can follow next time something breaks, including the first three commands you would run and the three questions you would ask.

## Summary

Errors are evidence. Read a traceback from the bottom to find the exception, then upward to the first line of your own code, and look for chained exceptions to locate the root cause. Django shows a detailed debug page only when `DEBUG` is true; in production it renders `404.html`, `403.html`, `400.html`, and `500.html` or the handlers you register, and the 500 handler must stay independent of the database and sessions. Learn the meaning of the usual failures: `ImproperlyConfigured`, `TemplateDoesNotExist`, `NoReverseMatch`, `OperationalError`, `DisallowedHost`, CSRF failures, and `InconsistentMigrationHistory`. Use logging for lasting information, `breakpoint()` for live inspection, and `print` only for throwaway checks. Debug systematically: reproduce, read, hypothesize, test, fix the cause, and keep a regression test.
