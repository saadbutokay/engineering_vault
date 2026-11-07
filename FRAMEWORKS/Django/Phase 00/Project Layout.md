## Overview

Django's default layout works for tutorials, but it does not scale gracefully. A deliberate layout makes a project easier to navigate, test, and split into responsibilities as it grows. This note presents a layout used in many professional codebases: a `config/` package for project-level files, an `apps/` folder for applications, and apps that are organized into modules once they outgrow single files. You will also build a small, tested app that follows the structure.

Models are explained properly in Phase 2. Here they appear only to show where files belong.

## Core concepts

### The default layout and its limits

`startproject` and `startapp` give you flat apps with single `models.py`, `views.py`, and `tests.py` files. This is fine at first. Problems appear when a `models.py` reaches thousands of lines, when the project package is named after the project and confuses imports, and when business logic ends up scattered across views.

### A recommended layout

```text
mysite/
    manage.py
    pyproject.toml
    uv.lock
    .gitignore
    .env.example
    config/
        __init__.py
        settings.py
        urls.py
        asgi.py
        wsgi.py
    apps/
        __init__.py
        blog/
            __init__.py
            apps.py
            admin.py
            urls.py
            models/
                __init__.py
                post.py
            views/
                __init__.py
                posts.py
            services/
                __init__.py
                publishing.py
            migrations/
                __init__.py
            templates/
                blog/
                    post_list.html
            static/
                blog/
                    blog.css
            tests/
                __init__.py
                test_services.py
    templates/
        base.html
    static/
```

- `config/` holds only project-wide configuration.
- `apps/` groups every application so imports read `apps.blog.models`.
- Each app can namespace its templates and static files under a folder named after the app to avoid collisions.
- Project-level `templates/` holds shared layouts such as `base.html`.

### AppConfig

Each app has an `AppConfig` class in `apps.py`.

- `name` is the full Python path of the app (for example `apps.blog`).
- `label` is the short unique identifier used in migrations and database table names (for example `blog`).
- `default_auto_field` chooses the primary key type for models that do not declare one. `BigAutoField` is the sensible default.

### Naming apps

Name apps with short, lowercase, singular or plural nouns that describe a domain: `blog`, `billing`, `accounts`. Avoid vague names such as `core` or `utils` for domain logic; they attract unrelated code.

### Splitting an app into modules

When a file grows large, convert it into a package. For `models/`, re-export the public names in `__init__.py` so that `from apps.blog.models import Post` keeps working and Django still discovers every model.

### Where tests go

A `tests/` package inside each app keeps tests next to the code they cover. Name files `test_*.py`.

## Basic syntax examples

### Create the structure

```bash
mkdir apps
touch apps/__init__.py
uv run python manage.py startapp blog apps/blog
```

On Windows PowerShell:

```powershell
mkdir apps
New-Item apps/__init__.py -ItemType File
uv run python manage.py startapp blog apps/blog
```

The `startapp` command requires the target folder to exist; create `apps/blog` first if your Django version reports an error.

### Configure the app

`apps/blog/apps.py`:

```python
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.blog"
    label = "blog"
```

Register it in `config/settings.py`:

```python
INSTALLED_APPS = [
    # ... Django apps ...
    "apps.blog.apps.BlogConfig",
]
```

### Turn models.py into a package

Delete `models.py` and create:

`apps/blog/models/post.py`:

```python
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)

    def __str__(self) -> str:
        return self.title
```

`apps/blog/models/__init__.py`:

```python
from .post import Post

__all__ = ["Post"]
```

### Namespaced templates and static files

```html
{% extends "base.html" %}

{% block content %}
  <h1>Posts</h1>
{% endblock %}
```

Place this at `apps/blog/templates/blog/post_list.html` and refer to it in a view as `"blog/post_list.html"`.

### Settings for project-level templates and static files

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {"context_processors": []},
    }
]

STATICFILES_DIRS = [BASE_DIR / "static"]
STATIC_URL = "static/"
```

## Deeper behavior

### Why `name` and `label` differ

Django uses `label` for database table prefixes (`blog_post`), for migration references, and for string references to models such as `"blog.Post"`. If you move an app from `blog` to `apps/blog` and keep `label = "blog"`, existing tables and migration history stay valid. Changing the label later means renaming tables, which is painful, so choose it deliberately before the first migration.

### Template and static file discovery

With `APP_DIRS = True`, Django searches each installed app's `templates/` folder in `INSTALLED_APPS` order. If two apps both contain `templates/list.html`, the first app wins and the second is silently shadowed. Namespacing as `templates/blog/list.html` removes the ambiguity. Static files behave the same way.

### Why the models package must re-export

Django discovers models by importing the app's `models` module. Models defined in submodules but never imported by `models/__init__.py` are not registered, so `makemigrations` ignores them. This is a classic "my table was not created" bug.

### Circular imports between apps

Apps should depend in one direction. If `orders` imports from `catalog`, `catalog` should not import from `orders`. When two models must reference each other, use string references such as `"orders.Order"` in relation fields. For behavior, move shared logic into a lower-level module that both can import.

### When not to over-structure

A tiny app with a dozen lines of models does not need five sub-packages. Split when a file becomes hard to navigate, not before.

## Industry-level snippet

A typical app separates data (models), business operations (services), and presentation (views). This example keeps publishing logic in a service function that views and commands can both call.

`apps/blog/models/post.py`:

```python
from django.db import models


class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"

    title = models.CharField(max_length=200)
    body = models.TextField()
    status = models.CharField(
        max_length=10,
        choices=Status,
        default=Status.DRAFT,
    )
    published_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self) -> str:
        return self.title
```

`apps/blog/models/__init__.py`:

```python
from .post import Post

__all__ = ["Post"]
```

`apps/blog/services/publishing.py`:

```python
"""Business operations for publishing posts."""

import logging
from datetime import datetime

from django.utils import timezone

from apps.blog.models import Post

logger = logging.getLogger(__name__)


class PostAlreadyPublishedError(Exception):
    """Raised when publishing a post that is already public."""


def publish_post(*, post_id: int, now: datetime | None = None) -> Post:
    """Publish the post with ``post_id`` and return it.

    Raises:
        Post.DoesNotExist: if no such post exists.
        PostAlreadyPublishedError: if the post is already published.
        ValueError: if the title or body is blank.
    """
    post = Post.objects.get(pk=post_id)

    if post.status == Post.Status.PUBLISHED:
        raise PostAlreadyPublishedError(f"Post {post_id} is already published")
    if not post.title.strip() or not post.body.strip():
        raise ValueError("A post needs a title and a body before publishing")

    post.status = Post.Status.PUBLISHED
    post.published_at = now or timezone.now()
    post.save(update_fields=["status", "published_at"])

    logger.info("Published post %s", post_id)
    return post
```

`apps/blog/services/__init__.py`:

```python
from .publishing import PostAlreadyPublishedError, publish_post

__all__ = ["PostAlreadyPublishedError", "publish_post"]
```

`apps/blog/tests/test_services.py`:

```python
from datetime import UTC, datetime

from django.test import TestCase

from apps.blog.models import Post
from apps.blog.services import PostAlreadyPublishedError, publish_post


class PublishPostTests(TestCase):
    def setUp(self) -> None:
        self.post = Post.objects.create(title="Hello", body="World")

    def test_publishes_draft_and_sets_timestamp(self) -> None:
        moment = datetime(2026, 1, 15, 9, 30, tzinfo=UTC)

        result = publish_post(post_id=self.post.pk, now=moment)

        self.assertEqual(result.status, Post.Status.PUBLISHED)
        self.assertEqual(result.published_at, moment)
        self.post.refresh_from_db()
        self.assertEqual(self.post.status, Post.Status.PUBLISHED)

    def test_publishing_twice_raises(self) -> None:
        publish_post(post_id=self.post.pk)
        with self.assertRaises(PostAlreadyPublishedError):
            publish_post(post_id=self.post.pk)

    def test_blank_body_is_rejected(self) -> None:
        empty = Post.objects.create(title="Title", body="   ")
        with self.assertRaises(ValueError):
            publish_post(post_id=empty.pk)

    def test_missing_post_raises_does_not_exist(self) -> None:
        with self.assertRaises(Post.DoesNotExist):
            publish_post(post_id=999_999)
```

The view layer stays thin: it parses input, calls `publish_post`, and translates exceptions into responses. Later phases build that layer in full.

## Common mistakes

- Defining models in a package but forgetting to import them in `models/__init__.py`.
- Reusing generic template names such as `index.html` across apps so that one silently hides the other.
- Naming the project package after the project and then importing from it ambiguously.
- Creating a catch-all `core` or `utils` app that becomes a dumping ground.
- Changing an app's `label` after migrations exist.
- Putting business logic in views, so it cannot be reused by commands, tasks, and tests.
- Circular imports between apps because two apps import each other's models at module level.
- Forgetting `__init__.py` in new packages and wondering why imports fail or tests are not discovered.

## Practice exercises

1. Create the recommended layout for a project called `shop` with apps `catalog` and `orders`. Verify with `manage.py check` that both load.
2. Convert a flat `models.py` containing three models into a `models/` package with one file per model and verify `makemigrations` still detects all three.
3. Create two apps that both have a template named `list.html` without namespacing. Demonstrate the shadowing problem, then fix it with namespaced folders.
4. Write a service function `unpublish_post(*, post_id: int) -> Post` that mirrors `publish_post`, clearing `published_at`, raising a specific error for drafts, and logging. Write tests for each branch.
5. Add `label` and `name` to an app config, run `makemigrations`, and inspect the generated migration to see where the label appears.
6. Draw the dependency direction among your apps and identify any import cycles. Propose how to break one using a string reference or a lower-level module.
7. Write a short rule in your own words for when an app should be split into two.

## Summary

A deliberate layout pays off as projects grow: `config/` for project-wide files, `apps/` for applications, namespaced templates and static files, and sub-packages such as `models/` and `services/` once files get large. Each app's `AppConfig` sets its path, short label, and default primary key type, and the label is hard to change later. Models must be re-exported from `models/__init__.py` so Django sees them. Keep dependencies between apps one-directional, and keep business logic in services so that views, commands, and tasks can share it.
