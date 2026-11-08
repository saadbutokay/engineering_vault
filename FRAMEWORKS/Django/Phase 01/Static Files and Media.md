## Overview

Web pages need more than HTML: stylesheets, scripts, fonts, and images that ship with your code (static files), and files that users upload while the site runs (media files). Django treats the two differently because they have different lifecycles and different security properties. Static files are part of your deployment and can be cached aggressively. Media files are untrusted user content that must be stored and served with care. This note explains the settings involved, how `collectstatic` works, how to serve static files in production, why Django does not serve media in production, and how hashed filenames provide cache busting.

## Core concepts

### Static files versus media files

- Static files: written by developers, committed to Git, identical for every user, safe to cache for a long time. Examples: `site.css`, a logo, a JavaScript bundle.
- Media files: uploaded by users while the site runs, stored outside Git, unique per user, and never trusted. Examples: profile pictures, PDF attachments.

### Static settings

- `STATIC_URL`: the URL prefix for static files, for example `static/`.
- `STATICFILES_DIRS`: extra folders (beyond each app's `static/` folder) where Django looks for static files during development and collection.
- `STATIC_ROOT`: the single folder where `collectstatic` gathers every static file for deployment. It must be empty of your own source files and is not committed to Git.
- `STORAGES["staticfiles"]`: the backend that stores and names collected files.

### Media settings

- `MEDIA_URL`: the URL prefix for uploads.
- `MEDIA_ROOT`: the folder on disk where uploads are stored.
- `STORAGES["default"]`: the backend used by `FileField` and `ImageField`. The default writes to `MEDIA_ROOT`.

### The STORAGES setting

`STORAGES` is a dictionary that names the storage backends. It replaced the older `STATICFILES_STORAGE` and `DEFAULT_FILE_STORAGE` settings, which were removed in Django 5.1.

### collectstatic

`manage.py collectstatic` copies every static file from every app and from `STATICFILES_DIRS` into `STATIC_ROOT`. A web server or WhiteNoise then serves that single folder.

### Serving static files in production

- Development: `runserver` serves static files automatically when `DEBUG` is true.
- Production: Django does not serve them. Use a web server (Nginx, Caddy), a CDN, object storage, or the WhiteNoise middleware, which serves collected files efficiently from inside the application and is a common starting point.

### Why Django does not serve media in production

Django's file serving is slow and not built for load, and uploaded files are untrusted content. Production setups serve media from object storage such as S3 with `django-storages` (Phase 11), or from a web server location, and often from a separate domain so that uploaded HTML or SVG can never run with your site's cookies.

### Cache busting with hashed filenames

Browsers cache static files. If you change `site.css` but keep the name, users may keep the old version. A manifest storage appends a content hash (`site.4f2a9c1b.css`), so a changed file gets a new URL and an unchanged file keeps its cached copy forever.

## Basic syntax examples

### Where files live

```text
mysite/
    static/                     # project-wide, listed in STATICFILES_DIRS
        css/site.css
    apps/blog/
        static/blog/blog.css    # app-level, namespaced by app name
    staticfiles/                # STATIC_ROOT, generated, not committed
    media/                      # MEDIA_ROOT, uploads, not committed
```

### Settings

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

STATIC_URL = "static/"
STATICFILES_DIRS = [BASE_DIR / "static"]
STATIC_ROOT = BASE_DIR / "staticfiles"

MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"

STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage"
    },
}
```

### Using static files in a template

```html
{% load static %}
<link rel="stylesheet" href="{% static 'css/site.css' %}">
<img src="{% static 'blog/logo.png' %}" alt="Logo">
```

Never write `/static/css/site.css` by hand. The `static` tag respects `STATIC_URL` and the storage backend, including hashed names.

### Collecting and finding files

```bash
uv run python manage.py collectstatic --noinput
uv run python manage.py findstatic css/site.css
```

### Serving media during development only

`config/urls.py`:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.urls import path

urlpatterns = [
    # ... your routes
]

urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

The `static()` helper returns an empty list when `DEBUG` is false, so it cannot accidentally serve media in production.

### WhiteNoise for production

```bash
uv add whitenoise
```

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # directly after SecurityMiddleware
    # ... the rest
]

STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"
    },
}
```

`CompressedManifestStaticFilesStorage` compresses files with gzip and Brotli and gives them hashed names. Run `collectstatic` during the build or deploy.

## Deeper behavior

### How Django finds static files

The `staticfiles` app uses finders. `FileSystemFinder` looks in `STATICFILES_DIRS`. `AppDirectoriesFinder` looks in each installed app's `static/` folder. Because all app folders share one namespace, two apps that both contain `static/site.css` collide, and the earlier app in `INSTALLED_APPS` wins. Namespace app files as `static/<app_name>/...`, just as with templates.

### Why `STATIC_ROOT` must be separate

`collectstatic` writes to `STATIC_ROOT`. If that folder were also listed in `STATICFILES_DIRS`, Django would copy files onto themselves. If it overlapped your media folder, deploys could delete or expose uploads. Keep them distinct and not nested.

### What the manifest does

With a manifest storage, `collectstatic` writes `staticfiles.json`, a map from original names to hashed names. The `{% static %}` tag consults the map. CSS files are post-processed so that `url(...)` references inside them also point at hashed files. Because the hashed URL changes whenever the content changes, you can serve these files with `Cache-Control: max-age=31536000, immutable`.

### A common test failure

With a manifest storage and `DEBUG` false (the test runner's default), `{% static 'x.css' %}` raises `ValueError: Missing staticfiles manifest entry` if `collectstatic` has not run. Use the plain `StaticFilesStorage` in test settings, or override `STORAGES` for the tests that render templates.

### Handling uploads safely

Treat every uploaded file as hostile: check its real type, limit its size, generate your own file names, and never execute it. Phase 3 covers these protections in detail. Serving uploads from a different domain prevents uploaded HTML or SVG from reading your site's cookies.

### Private files

Files that only certain users may see (invoices, reports) must not sit in a public media folder. Serve them through a view that checks permissions, or through short-lived pre-signed URLs from object storage (Phase 11).

### CDNs

A content delivery network caches your static files at locations near users. Because hashed filenames never change content, they work perfectly with long CDN cache lifetimes. Set `STATIC_URL` to the CDN address when you use one.

## Industry-level snippet

Misconfigured static settings are a frequent source of broken deployments: a missing trailing slash, a relative root, or nested folders. A small validator, called at startup, turns those mistakes into immediate, specific errors. A second function builds the `STORAGES` dictionary for the environment.

`config/static_config.py`:

```python
"""Validated static and media configuration."""

import logging
from pathlib import Path
from typing import Any

from django.core.exceptions import ImproperlyConfigured

logger = logging.getLogger(__name__)

PLAIN_STATIC_BACKEND = "django.contrib.staticfiles.storage.StaticFilesStorage"
HASHED_STATIC_BACKEND = "whitenoise.storage.CompressedManifestStaticFilesStorage"
MEDIA_BACKEND = "django.core.files.storage.FileSystemStorage"


def build_storages(*, debug: bool, hashed: bool | None = None) -> dict[str, dict[str, Any]]:
    """Return the ``STORAGES`` setting.

    Hashed, compressed static files are used unless ``debug`` is true. Pass
    ``hashed`` explicitly to override this, for example in test settings.
    """
    use_hashed = (not debug) if hashed is None else hashed
    return {
        "default": {"BACKEND": MEDIA_BACKEND},
        "staticfiles": {
            "BACKEND": HASHED_STATIC_BACKEND if use_hashed else PLAIN_STATIC_BACKEND
        },
    }


def validate_static_settings(
    *,
    debug: bool,
    static_url: str,
    static_root: Path | None,
    media_url: str,
    media_root: Path | None,
) -> list[str]:
    """Return a list of human-readable problems, empty when everything is valid."""
    problems: list[str] = []

    for name, url in (("STATIC_URL", static_url), ("MEDIA_URL", media_url)):
        if not url:
            problems.append(f"{name} must not be empty")
        elif not url.endswith("/"):
            problems.append(f"{name} must end with a slash")

    if static_url and static_url == media_url:
        problems.append("STATIC_URL and MEDIA_URL must be different")

    if not debug and static_root is None:
        problems.append("STATIC_ROOT is required when DEBUG is false")

    for name, root in (("STATIC_ROOT", static_root), ("MEDIA_ROOT", media_root)):
        if root is not None and not root.is_absolute():
            problems.append(f"{name} must be an absolute path")

    if static_root is not None and media_root is not None:
        overlapping = (
            static_root == media_root
            or static_root.is_relative_to(media_root)
            or media_root.is_relative_to(static_root)
        )
        if overlapping:
            problems.append("STATIC_ROOT and MEDIA_ROOT must not be equal or nested")

    return problems


def check_static_settings(
    *,
    debug: bool,
    static_url: str,
    static_root: Path | None,
    media_url: str,
    media_root: Path | None,
) -> None:
    """Raise ``ImproperlyConfigured`` listing every problem found."""
    problems = validate_static_settings(
        debug=debug,
        static_url=static_url,
        static_root=static_root,
        media_url=media_url,
        media_root=media_root,
    )
    if problems:
        for problem in problems:
            logger.error("Static configuration problem: %s", problem)
        raise ImproperlyConfigured("; ".join(problems))
```

Using it at the bottom of `config/settings.py`:

```python
from .static_config import build_storages, check_static_settings

STORAGES = build_storages(debug=DEBUG)

check_static_settings(
    debug=DEBUG,
    static_url=STATIC_URL,
    static_root=STATIC_ROOT,
    media_url=MEDIA_URL,
    media_root=MEDIA_ROOT,
)
```

`config/test_static_config.py`:

```python
from pathlib import Path
from typing import Any

import pytest
from django.core.exceptions import ImproperlyConfigured

from config.static_config import (
    HASHED_STATIC_BACKEND,
    PLAIN_STATIC_BACKEND,
    build_storages,
    check_static_settings,
    validate_static_settings,
)

GOOD: dict[str, Any] = {
    "debug": False,
    "static_url": "static/",
    "static_root": Path("/srv/app/staticfiles"),
    "media_url": "media/",
    "media_root": Path("/srv/app/media"),
}


def test_valid_configuration_has_no_problems() -> None:
    assert validate_static_settings(**GOOD) == []


def test_missing_trailing_slash_is_reported() -> None:
    problems = validate_static_settings(**{**GOOD, "static_url": "static"})
    assert problems == ["STATIC_URL must end with a slash"]


def test_static_root_required_in_production() -> None:
    problems = validate_static_settings(**{**GOOD, "static_root": None})
    assert "STATIC_ROOT is required when DEBUG is false" in problems


def test_static_root_not_required_in_debug() -> None:
    config = {**GOOD, "debug": True, "static_root": None}
    assert validate_static_settings(**config) == []


def test_relative_roots_are_rejected() -> None:
    config = {**GOOD, "media_root": Path("media")}
    problems = validate_static_settings(**config)
    assert problems == ["MEDIA_ROOT must be an absolute path"]


def test_nested_roots_are_rejected() -> None:
    config = {**GOOD, "media_root": Path("/srv/app/staticfiles/uploads")}
    problems = validate_static_settings(**config)
    assert problems == ["STATIC_ROOT and MEDIA_ROOT must not be equal or nested"]


def test_same_urls_are_rejected() -> None:
    config = {**GOOD, "media_url": "static/"}
    problems = validate_static_settings(**config)
    assert "STATIC_URL and MEDIA_URL must be different" in problems


def test_check_raises_with_all_problems() -> None:
    config = {**GOOD, "static_url": "static", "media_root": Path("media")}
    with pytest.raises(ImproperlyConfigured, match="STATIC_URL.*MEDIA_ROOT"):
        check_static_settings(**config)


@pytest.mark.parametrize(
    ("debug", "hashed", "backend"),
    [
        (True, None, PLAIN_STATIC_BACKEND),
        (False, None, HASHED_STATIC_BACKEND),
        (False, False, PLAIN_STATIC_BACKEND),
    ],
)
def test_build_storages_selects_backend(
    debug: bool, hashed: bool | None, backend: str
) -> None:
    storages = build_storages(debug=debug, hashed=hashed)
    assert storages["staticfiles"]["BACKEND"] == backend
```

## Common mistakes

- Writing static URLs by hand instead of using `{% static %}`.
- Forgetting `collectstatic` during deployment, which leaves the site without CSS.
- Setting `STATIC_ROOT` to a folder that is also in `STATICFILES_DIRS`.
- Committing `staticfiles/` or `media/` to Git.
- Serving media through Django in production.
- Leaving `DEBUG = True` on a public server because `runserver` served static files so conveniently.
- Putting `WhiteNoiseMiddleware` before `SecurityMiddleware` or far down the list.
- Using a manifest storage in tests without running `collectstatic`.
- Using old settings (`STATICFILES_STORAGE`, `DEFAULT_FILE_STORAGE`) that no longer exist in Django 5.1 and later.
- Naming app static files without the app folder prefix, so two apps overwrite each other.
- Trusting uploaded file names and types.

## Practice exercises

1. Create a project-level `static/css/site.css` and an app-level `static/blog/blog.css`. Load both in a template and verify them with `findstatic`.
2. Run `collectstatic` into a `staticfiles/` folder and list what it copied, including files from the admin.
3. Install WhiteNoise, switch to `CompressedManifestStaticFilesStorage`, run `collectstatic`, and inspect `staticfiles.json`. Change `site.css` and rerun to see the new hash.
4. Reproduce the "Missing staticfiles manifest entry" error by rendering a template with a manifest storage before running `collectstatic`. Fix it for tests without changing production settings.
5. Configure media serving for development, upload a file through the admin (after adding an `ImageField` in a scratch model), and confirm that `static()` returns an empty list when `DEBUG` is false.
6. Refactor `validate_static_settings` to accept a frozen dataclass `StaticConfig` instead of five keyword arguments, and update the tests to build instances of it rather than spreading a dictionary.
7. Write a short deployment checklist for static and media files covering collection, storage, caching headers, and a separate domain for uploads.

## Summary

Static files ship with your code and are collected into `STATIC_ROOT` with `collectstatic`; media files are untrusted uploads stored under `MEDIA_ROOT` or in object storage. The `STORAGES` setting selects the backends, and a manifest storage adds content hashes that enable permanent caching. In production, serve static files with WhiteNoise, a web server, or a CDN, and serve media from object storage or a dedicated location, never from Django itself. Reference files only through `{% static %}`, keep roots separate and absolute, validate the configuration at startup, and treat every upload as hostile.
