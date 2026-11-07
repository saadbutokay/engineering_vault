## Overview

A well-configured editor makes Django development faster and safer: autocomplete that understands models, type checking that catches mistakes before runtime, a debugger you can step through, and an interactive shell with your project already loaded. This note covers Visual Studio Code, PyCharm as an alternative, type checking with `django-stubs` and `mypy`, the Django shell with `django-extensions`, and `django-debug-toolbar` for inspecting requests in the browser.

## Core concepts

### Visual Studio Code

Install these extensions:

- Python (Microsoft): interpreter selection, testing, and debugging.
- Pylance: language server with completion and type information.
- Django (Baptiste Darthenay): syntax highlighting for Django templates.
- Ruff (Astral): linting and formatting on save.

Select the project's interpreter with the command palette entry "Python: Select Interpreter" and pick the `.venv` inside the project. If the wrong interpreter is selected, imports appear unresolved even though the code runs.

### PyCharm

PyCharm Professional has first-class Django support: project structure awareness, template navigation, model and URL completion, a database browser, and a run configuration for Django servers and tests. The Community edition lacks the Django-specific features but still works as a capable Python editor.

### Type checking with django-stubs

Django is dynamically typed in many places (for example, model fields become attributes at runtime). `django-stubs` provides type information, and its `mypy` plugin teaches `mypy` how models, managers, and querysets behave. Together they let `mypy` catch errors such as accessing a field that does not exist.

### Debugging a running server

A debugger lets you pause execution at a breakpoint, inspect variables, and step line by line. In VS Code this requires a launch configuration that starts `manage.py runserver` under the debugger.

### The Django shell

`manage.py shell` opens a Python REPL with Django configured, so you can query models and call code interactively. `django-extensions` adds `shell_plus`, which automatically imports all your models and common utilities.

### django-debug-toolbar

A panel that appears in the browser during development and shows the SQL queries, template rendering time, request and response headers, settings, and more for the current page. It is one of the most useful tools for finding slow or repeated queries.

## Basic syntax examples

### Install the development tools

```bash
uv add --dev mypy "django-stubs[compatible-mypy]" ruff
uv add --dev django-extensions django-debug-toolbar
```

With `pip`:

```bash
python -m pip install mypy "django-stubs[compatible-mypy]" ruff django-extensions django-debug-toolbar
```

### VS Code workspace settings

Create `.vscode/settings.json`:

```text
{
    "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
    "python.testing.unittestEnabled": true,
    "python.testing.unittestArgs": ["-v", "-s", ".", "-p", "test*.py"],
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    },
    "files.associations": {
        "**/templates/**/*.html": "django-html"
    }
}
```

On Windows, the interpreter path is `${workspaceFolder}/.venv/Scripts/python.exe`.

### VS Code debug configuration

Create `.vscode/launch.json`:

```text
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Django: runserver",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/manage.py",
            "args": ["runserver", "--noreload"],
            "django": true,
            "justMyCode": true
        }
    ]
}
```

Press F5, set a breakpoint inside a view, and load the page in the browser. Execution pauses at your breakpoint.

### mypy configuration

In `pyproject.toml`:

```toml
[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["mypy_django_plugin.main"]

[tool.django-stubs]
django_settings_module = "config.settings"
```

Run it:

```bash
uv run mypy .
```

### The Django shell

```bash
uv run python manage.py shell
```

```python
from django.contrib.auth.models import User

print(User.objects.count())
```

### shell_plus

Add `"django_extensions"` to `INSTALLED_APPS`, then:

```bash
uv run python manage.py shell_plus
uv run python manage.py shell_plus --print-sql
```

The `--print-sql` flag prints every SQL query that your commands run, which is excellent for learning how the ORM behaves.

### Debug toolbar setup

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    "debug_toolbar",
]

MIDDLEWARE = [
    "debug_toolbar.middleware.DebugToolbarMiddleware",
    # ... the rest of the middleware
]

INTERNAL_IPS = ["127.0.0.1"]
```

```python
# config/urls.py
from django.conf import settings
from django.urls import include, path

urlpatterns = [
    # ... your routes
]

if settings.DEBUG:
    urlpatterns = [path("__debug__/", include("debug_toolbar.urls")), *urlpatterns]
```

## Deeper behavior

### Why the interpreter choice matters

The editor runs its language server, linter, and test discovery using whichever interpreter you select. If it points to the wrong environment, the editor reports missing imports for packages you have installed, or silently checks against a different Django version. When something looks wrong in the editor but works in the terminal, check the interpreter first.

### What the mypy plugin adds

Without the plugin, `Post.objects.filter(...)` is typed loosely and attribute access on model instances cannot be verified. With it, `mypy` knows that `Post.objects` is a manager returning `QuerySet[Post]`, that `post.title` is a `str`, and that a misspelled lookup name in a method call is suspicious. The plugin loads your settings module, so it must be importable when `mypy` runs.

### Debugging and the reloader

The autoreloader starts a child process, and debuggers attach to the parent, which is why the launch configuration uses `--noreload`. After editing code you must restart the debug session manually.

### Breakpoints without an IDE

Python's built-in `breakpoint()` function drops you into `pdb` in the terminal that runs `runserver`. Useful commands: `n` (next line), `s` (step into), `c` (continue), `p expression` (print), `l` (list source), and `q` (quit).

### Debug toolbar conditions

The toolbar shows only when `DEBUG` is true, the request comes from an address in `INTERNAL_IPS`, and the response is HTML with a closing body tag. A common surprise is that it does not appear inside Docker, because the request arrives from the container network address rather than `127.0.0.1`. The tests also run with `DEBUG` forced off, so the toolbar does not interfere.

### Keep development tools out of production

`django-extensions` and `django-debug-toolbar` are development dependencies. The toolbar in particular exposes SQL and settings. Install them in a development dependency group and load them conditionally.

## Industry-level snippet

Hard-coding development apps into `INSTALLED_APPS` breaks production, where those packages are not installed. A reliable pattern is a small, testable helper that adds development tools only when appropriate.

`config/devtools.py`:

```python
"""Conditionally enable development-only apps and middleware."""

import logging
from dataclasses import dataclass
from importlib.util import find_spec

logger = logging.getLogger(__name__)

TOOLBAR_APP = "debug_toolbar"
TOOLBAR_MIDDLEWARE = "debug_toolbar.middleware.DebugToolbarMiddleware"
EXTENSIONS_APP = "django_extensions"


@dataclass(frozen=True)
class DevTools:
    installed_apps: list[str]
    middleware: list[str]


def is_installed(module_name: str) -> bool:
    """Return True if ``module_name`` can be imported."""
    return find_spec(module_name) is not None


def with_dev_tools(
    installed_apps: list[str],
    middleware: list[str],
    *,
    debug: bool,
    testing: bool = False,
) -> DevTools:
    """Return app and middleware lists with development tools added.

    Nothing is added when ``debug`` is false, when running tests, or when
    the package is not installed. The inputs are never mutated.
    """
    apps = list(installed_apps)
    layers = list(middleware)

    if not debug or testing:
        return DevTools(apps, layers)

    if is_installed(EXTENSIONS_APP) and EXTENSIONS_APP not in apps:
        apps.append(EXTENSIONS_APP)

    if is_installed(TOOLBAR_APP) and TOOLBAR_APP not in apps:
        apps.append(TOOLBAR_APP)
        layers.insert(0, TOOLBAR_MIDDLEWARE)
        logger.info("django-debug-toolbar enabled")

    return DevTools(apps, layers)
```

Using it in `config/settings.py`:

```python
import sys

from .devtools import with_dev_tools

DEBUG = True
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

_tools = with_dev_tools(
    INSTALLED_APPS,
    MIDDLEWARE,
    debug=DEBUG,
    testing="test" in sys.argv or "pytest" in sys.modules,
)
INSTALLED_APPS = _tools.installed_apps
MIDDLEWARE = _tools.middleware
INTERNAL_IPS = ["127.0.0.1"]
```

`config/test_devtools.py`:

```python
from unittest import mock

from config.devtools import (
    EXTENSIONS_APP,
    TOOLBAR_APP,
    TOOLBAR_MIDDLEWARE,
    with_dev_tools,
)

BASE_APPS = ["django.contrib.auth"]
BASE_MIDDLEWARE = ["django.middleware.common.CommonMiddleware"]


def test_nothing_added_when_debug_is_off() -> None:
    result = with_dev_tools(BASE_APPS, BASE_MIDDLEWARE, debug=False)
    assert result.installed_apps == BASE_APPS
    assert result.middleware == BASE_MIDDLEWARE


def test_nothing_added_during_tests() -> None:
    result = with_dev_tools(BASE_APPS, BASE_MIDDLEWARE, debug=True, testing=True)
    assert result.installed_apps == BASE_APPS


def test_tools_added_when_installed() -> None:
    with mock.patch("config.devtools.is_installed", return_value=True):
        result = with_dev_tools(BASE_APPS, BASE_MIDDLEWARE, debug=True)
    assert TOOLBAR_APP in result.installed_apps
    assert EXTENSIONS_APP in result.installed_apps
    assert result.middleware[0] == TOOLBAR_MIDDLEWARE


def test_missing_packages_are_skipped() -> None:
    with mock.patch("config.devtools.is_installed", return_value=False):
        result = with_dev_tools(BASE_APPS, BASE_MIDDLEWARE, debug=True)
    assert result.installed_apps == BASE_APPS


def test_inputs_are_not_mutated() -> None:
    apps = list(BASE_APPS)
    with mock.patch("config.devtools.is_installed", return_value=True):
        with_dev_tools(apps, BASE_MIDDLEWARE, debug=True)
    assert apps == BASE_APPS
```

## Common mistakes

- Selecting the wrong interpreter, then installing packages that the editor cannot see.
- Debugging with the autoreloader on and wondering why breakpoints are skipped or the process restarts.
- Adding `debug_toolbar` to production settings, which crashes the app when the package is absent or leaks information when it is present.
- Placing `DebugToolbarMiddleware` after middleware that modifies the response body, which stops the toolbar from appearing.
- Forgetting `INTERNAL_IPS`, then deciding the toolbar is broken.
- Running `mypy` without `django_settings_module`, which makes the plugin fail on import.
- Disabling type checking on whole files the first time `mypy` reports a problem instead of reading the errors.
- Leaving `print` statements in code after debugging instead of using logging or the debugger.

## Practice exercises

1. Configure your editor from scratch on a fresh project: interpreter, Ruff on save, Django template association. Verify each by introducing a deliberate fault.
2. Create the launch configuration, set a breakpoint in the `home` view from note 0.3, and step through until the response is returned. Record the values of `request.method` and `request.path`.
3. Use `shell_plus --print-sql` to create a `User` and then fetch it by username. Note how many queries each operation runs.
4. Write a deliberately mistyped attribute access on a model instance and confirm that `mypy` reports it. Then fix it.
5. Install the debug toolbar, load a page, and open the SQL and Headers panels. List what each reveals about the request.
6. Extend `with_dev_tools` so that it also accepts an extra list of module names that should be enabled in debug mode, validating that none are empty strings. Add tests.
7. Explain why the toolbar might not appear inside a Docker container and propose two fixes.

## Summary

A productive Django editor setup combines the right interpreter, Ruff for formatting and linting, a Django-aware template mode, and a debugger configured for `manage.py`. `mypy` with `django-stubs` and its plugin checks model and queryset usage. `shell_plus` speeds up experimentation, and the debug toolbar makes queries and request details visible. Development-only packages must be added conditionally and tested so that production never depends on them.
