## Overview

This note takes you from an empty folder to a running Django site with your own app, view, and URL. You will install Django into an isolated virtual environment, generate a project, read every generated file, and write a first view. By the end you will also have a tested health-check endpoint, a small but realistic piece of backend code.

Target versions: Django 5.2 LTS on Python 3.12 or newer. Everything here also works on Django 6.0.

## Core concepts

### Virtual environments

A virtual environment is an isolated Python installation for one project, so that package versions do not collide between projects. Never install Django into your system Python.

### Package managers

- `uv` is a fast modern tool that creates environments, resolves dependencies, and writes a lock file.
- `pip` is the standard installer that ships with Python. It works well but does not lock dependencies by itself.

This roadmap prefers `uv` and shows `pip` as the alternative.

### The generated project

`django-admin startproject` creates:

- `manage.py`: a thin command-line wrapper that sets `DJANGO_SETTINGS_MODULE` and runs Django commands.
- The configuration package (named `config` in this roadmap) containing:
  - `settings.py`: all configuration.
  - `urls.py`: the root URL configuration.
  - `wsgi.py` and `asgi.py`: entry points for application servers.
  - `__init__.py`: marks the folder as a package.

### The development server

`runserver` starts a lightweight server and an autoreloader that restarts the process when Python files change. It is for development only: it is not hardened, not fast, and not meant for real traffic.

### Apps

`startapp` generates a package with `models.py`, `views.py`, `admin.py`, `apps.py`, `tests.py`, and a `migrations/` folder. An app does nothing until it is listed in `INSTALLED_APPS`.

### Views and URLs

A view is a callable that takes an `HttpRequest` and returns an `HttpResponse`. A URL pattern connects a path to a view. The project `urls.py` usually delegates to each app's `urls.py` with `include()`.

### System checks

`manage.py check` runs Django's built-in validation of settings, models, and URL configuration. It catches many mistakes before the server even starts.

## Basic syntax examples

### Create and activate an environment with uv

```bash
mkdir mysite
cd mysite
uv init --bare
uv add "django>=5.2,<6.0"
uv run django-admin --version
```

`uv run` automatically uses the project's environment, so activation is optional.

### The pip alternative on macOS and Linux

```bash
mkdir mysite
cd mysite
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install "django>=5.2,<6.0"
django-admin --version
```

### The pip alternative on Windows PowerShell

```powershell
mkdir mysite
cd mysite
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install "django>=5.2,<6.0"
django-admin --version
```

If PowerShell blocks the activation script, run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once.

### Generate the project and an app

```bash
uv run django-admin startproject config .
uv run python manage.py startapp pages
```

The trailing dot in the first command matters: it creates the project in the current folder instead of nesting another folder.

The result:

```text
mysite/
    manage.py
    pyproject.toml
    config/
        __init__.py
        asgi.py
        settings.py
        urls.py
        wsgi.py
    pages/
        __init__.py
        admin.py
        apps.py
        migrations/
            __init__.py
        models.py
        tests.py
        views.py
```

### Register the app

In `config/settings.py`:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "pages",
]
```

### Write a first view

`pages/views.py`:

```python
from django.http import HttpRequest, HttpResponse


def home(request: HttpRequest) -> HttpResponse:
    return HttpResponse("Hello, Django")
```

### Wire up the URLs

`pages/urls.py` (create this file):

```python
from django.urls import path

from . import views

app_name = "pages"

urlpatterns = [
    path("", views.home, name="home"),
]
```

`config/urls.py`:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("pages.urls")),
]
```

### Run it

```bash
uv run python manage.py check
uv run python manage.py runserver
```

Open `http://127.0.0.1:8000/` and you should see the greeting. Python 3.12 or newer is required; confirm with `uv run python --version`.

## Deeper behavior

### What `manage.py` actually does

It sets the `DJANGO_SETTINGS_MODULE` environment variable to `config.settings` and calls `execute_from_command_line`. Every command (`runserver`, `migrate`, `check`, `shell`) is a management command discovered from installed apps.

### The autoreloader

`runserver` runs two processes: a parent that watches files and a child that serves requests. When you save a Python file, the child is restarted. Changes to templates are read on the next request, but changes to settings or newly added files in new directories sometimes need a manual restart. Use `--noreload` when debugging with a breakpoint-based debugger that conflicts with the reloader.

### Warning about the unapplied migrations

The first time you run `runserver`, you will see a notice about unapplied migrations. The built-in apps (admin, auth, sessions) have database tables that do not exist yet. Run `uv run python manage.py migrate` to create them. Phase 2 explains migrations properly.

### What registration means

Putting an app in `INSTALLED_APPS` lets Django discover its models, templates in `templates/`, static files in `static/`, management commands, and migrations. A forgotten registration causes confusing errors such as `TemplateDoesNotExist` or models that never migrate.

### Namespacing URL names

Setting `app_name = "pages"` lets you refer to the URL as `pages:home`. Namespaces prevent two apps that both define a `home` route from colliding. Always use `reverse("pages:home")` or `{% url "pages:home" %}` instead of hard-coding paths.

### Lock files

With `uv`, `uv.lock` records the exact version of every dependency. Commit it. With `pip`, produce a similar record with a requirements file. Reproducible installs avoid the classic "works on my machine" problem.

## Industry-level snippet

Real deployments need a health endpoint that load balancers and orchestrators call to ask "is this instance alive and able to serve?" A good one checks the database, returns the right status codes, rejects the wrong methods, logs failures, and is tested.

`pages/views.py`:

```python
import logging

from django.db import DatabaseError, connection
from django.http import HttpRequest, HttpResponse, JsonResponse
from django.views.decorators.http import require_GET

logger = logging.getLogger(__name__)


def home(request: HttpRequest) -> HttpResponse:
    return HttpResponse("Hello, Django")


def check_database() -> bool:
    """Return True if a trivial query succeeds."""
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
            cursor.fetchone()
    except DatabaseError:
        logger.exception("Database health check failed")
        return False
    return True


@require_GET
def health(request: HttpRequest) -> JsonResponse:
    database_ok = check_database()
    return JsonResponse(
        {
            "status": "ok" if database_ok else "degraded",
            "database": database_ok,
        },
        status=200 if database_ok else 503,
    )
```

`pages/urls.py`:

```python
from django.urls import path

from . import views

app_name = "pages"

urlpatterns = [
    path("", views.home, name="home"),
    path("health/", views.health, name="health"),
]
```

`pages/tests.py`:

```python
from unittest import mock

from django.test import TestCase
from django.urls import reverse


class HealthViewTests(TestCase):
    def test_returns_200_when_database_is_available(self) -> None:
        response = self.client.get(reverse("pages:health"))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json(), {"status": "ok", "database": True})

    def test_returns_503_when_database_check_fails(self) -> None:
        with mock.patch("pages.views.check_database", return_value=False):
            response = self.client.get(reverse("pages:health"))
        self.assertEqual(response.status_code, 503)
        self.assertEqual(response.json()["status"], "degraded")

    def test_rejects_non_get_methods(self) -> None:
        response = self.client.post(reverse("pages:health"))
        self.assertEqual(response.status_code, 405)

    def test_home_page_greets(self) -> None:
        response = self.client.get(reverse("pages:home"))
        self.assertContains(response, "Hello, Django")
```

Run the tests:

```bash
uv run python manage.py test
```

The health view demonstrates several habits to keep: small functions that do one thing, explicit status codes, a decorator that restricts methods, logging with a traceback on failure, and a test for the failure path as well as the happy path.

## Common mistakes

- Running `startproject config` without the trailing dot and ending up with `config/config/`.
- Forgetting to register the app in `INSTALLED_APPS`.
- Forgetting to create `urls.py` in the app, or forgetting `include()` in the project.
- Installing Django globally and then wondering which version is running. Check with `django-admin --version` inside the environment.
- Committing the virtual environment folder. Add `.venv/` to `.gitignore`.
- Ignoring the output of `manage.py check`.
- Using `runserver` behind a real domain.
- Hard-coding URLs in code and templates instead of using named routes.

## Practice exercises

1. Create a new project named `library` with an app named `catalog`. Do it once with `uv` and once with `pip` in a separate folder, and compare the files each produces.
2. Add a view `about` at `/about/` that returns a short text response, and give it a URL name. Use `reverse` in a test to check the status code.
3. Add a view `time_now` that returns the current UTC time as JSON with a key `utc` in ISO format. Write a test that mocks the clock so the result is deterministic.
4. Make a view `echo` that reads a `message` query parameter and returns it as plain text, returning 400 with a clear message when the parameter is missing or longer than 200 characters.
5. Break the project on purpose in three ways (remove an app from `INSTALLED_APPS`, misspell a URL name, delete a comma in settings). Record each error message and what fixed it.
6. Add a `version` key to the health endpoint response that reports the installed Django version, using `django.get_version()`.
7. Read `config/wsgi.py` and `config/asgi.py` and explain in two sentences what each exposes and who calls it.

## Summary

A Django project starts with a virtual environment and `startproject`, which gives you `manage.py` and a configuration package. Functionality lives in apps, which must be registered in `INSTALLED_APPS`. A view takes a request and returns a response, and URL patterns, namespaced with `app_name`, connect paths to views. `runserver` and its autoreloader are for development only. `manage.py check` catches configuration mistakes early. Writing even a small health endpoint with tests builds the habits of explicit status codes, logging, and test coverage that the rest of this roadmap relies on.
