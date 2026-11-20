## Overview

Django includes a collection of small, ready-made tools that solve recurring problems well enough that you should know them before writing your own: a serialization framework that converts model data to JSON and back, commands for moving data in and out of a database as fixtures, apps that generate sitemaps and RSS feeds, humanized number and date formatting, URL redirect management, a sites framework, PostgreSQL-specific features, and a system check framework that catches configuration mistakes before the server starts. This note covers `dumpdata` and `loaddata`, why fixtures are poor for tests at scale (and where they are the right tool), the serialization framework and natural keys, `django.contrib.sitemaps`, `syndication`, `humanize`, `redirects`, `sites`, and `django.contrib.postgres`, and writing a custom system check. It ends with a sitemap and a feed for the bookstore, reference data loaded from a fixture, an export function, and custom checks for the database configuration from note 2.12, all tested.

## Core concepts

### Serialization

The serialization framework turns querysets into text and back. `django.core.serializers` supports JSON, JSON Lines (`jsonl`), XML, and YAML (with PyYAML installed). Each serialized object records the model label, the primary key, and a dictionary of field values:

```text
{"model": "bookstore.genre", "pk": 1, "fields": {"name": "Fantasy", "parent": null}}
```

It is meant for Django-to-Django data transfer, not for building public APIs (Phase 7 covers those).

### Fixtures

A fixture is a file of serialized objects. `manage.py loaddata name` reads fixtures from each app's `fixtures/` directory, the `FIXTURE_DIRS` setting, or a path, and saves the objects. `manage.py dumpdata app.Model` writes them. Loading a fixture updates rows that already have the same primary key and creates the rest, so loading is repeatable.

### What fixtures are good for

- Small, stable reference data that the application needs to run: countries, genres, initial roles, plan definitions.
- Seeding a development database with example content.
- One-off data transfers between development environments.

### Why fixtures are poor for tests at scale

- They are static snapshots, so every schema change risks breaking them, and updating large JSON files by hand is error-prone.
- They hide the data a test depends on in a distant file, which makes tests hard to read.
- They tie tests to specific primary keys and to each other.
- They load more than most tests need and slow the suite.

Factory functions (note 10.2) create exactly the data a test needs, in the test itself, and follow schema changes automatically. Use fixtures in tests only for small reference data.

### Natural keys

A natural key identifies an object by its content, such as a genre's name, instead of its primary key. Fixtures that use natural keys survive being loaded into databases where primary keys differ. To support them, the model defines `natural_key()` and its manager defines `get_by_natural_key()`. Use `dumpdata --natural-foreign` and `--natural-primary` to produce them.

### Contrib apps

Django ships optional apps under `django.contrib`. The ones covered here:

- `sitemaps`: generate `sitemap.xml` for search engines.
- `syndication`: generate RSS and Atom feeds.
- `humanize`: template filters for friendlier numbers and dates.
- `redirects`: store URL redirects in the database, applied when a page is not found.
- `sites`: a `Site` model that represents the domain your project serves, used by several other contrib apps.
- `postgres`: PostgreSQL-specific fields, indexes, constraints, search tools, and aggregates.

### The system check framework

Django runs checks on startup and with `manage.py check`: it validates models, settings, URL configuration, and installed apps. Checks return messages at a level: `Debug`, `Info`, `Warning`, `Error`, or `Critical`. Errors and critical messages stop `runserver` and `migrate`. You can register your own checks to protect your project's conventions and configuration.

### Deployment checks

Checks registered with `deploy=True` run only with `manage.py check --deploy` and audit production readiness (HTTPS settings, secret key strength, secure cookies). Running it in CI (`--fail-level WARNING`) keeps unsafe settings out of releases.

## Basic syntax examples

### dumpdata and loaddata

```bash
uv run python manage.py dumpdata bookstore.Genre --indent 2 > genres.json
uv run python manage.py dumpdata bookstore --natural-foreign --natural-primary --indent 2 > bookstore.json
uv run python manage.py dumpdata --exclude contenttypes --exclude auth.permission > everything.json

uv run python manage.py loaddata genres
uv run python manage.py loaddata path/to/genres.json
```

### A fixture file

`apps/bookstore/fixtures/genres.json`:

```text
[
  {"model": "bookstore.genre", "pk": 1, "fields": {"name": "Fantasy", "parent": null}},
  {"model": "bookstore.genre", "pk": 2, "fields": {"name": "Science fiction", "parent": null}},
  {"model": "bookstore.genre", "pk": 3, "fields": {"name": "Epic fantasy", "parent": 1}}
]
```

### Using a fixture in a test

```python
from django.test import TestCase


class GenreTests(TestCase):
    fixtures = ["genres.json"]

    def test_the_reference_data_is_loaded(self) -> None:
        ...
```

### Serializing a queryset

```python
from django.core import serializers

from apps.bookstore.models import Book

text = serializers.serialize("json", Book.objects.all(), fields=("title", "price"), indent=2)

for deserialized in serializers.deserialize("json", text):
    deserialized.save()     # only for trusted data
```

### Natural keys

```python
from django.db import models


class GenreManager(models.Manager["Genre"]):
    def get_by_natural_key(self, name: str) -> "Genre":
        return self.get(name=name)


class Genre(models.Model):
    name = models.CharField(max_length=100, unique=True)
    objects = GenreManager()

    def natural_key(self) -> tuple[str]:
        return (self.name,)
```

### A sitemap

```python
from django.contrib.sitemaps import Sitemap


class BookSitemap(Sitemap):
    changefreq = "weekly"
    priority = 0.6

    def items(self):
        return Book.objects.all()

    def lastmod(self, obj):
        return obj.created_at
```

```python
from django.contrib.sitemaps.views import sitemap

urlpatterns = [
    path("sitemap.xml", sitemap, {"sitemaps": {"books": BookSitemap}}, name="django.contrib.sitemaps.views.sitemap"),
]
```

Add `"django.contrib.sitemaps"` to `INSTALLED_APPS`, and each model needs a `get_absolute_url()` method.

### A feed

```python
from django.contrib.syndication.views import Feed


class NewBooksFeed(Feed):
    title = "New books"
    link = "/books/search/"
    description = "The newest books in the catalog."

    def items(self):
        return Book.objects.order_by("-created_at")[:20]

    def item_title(self, item):
        return item.title
```

```python
path("feeds/new-books/", NewBooksFeed(), name="new_books_feed"),
```

### humanize

```python
INSTALLED_APPS += ["django.contrib.humanize"]
```

```html
{% load humanize %}
{{ 1234567|intcomma }}            {# 1,234,567 #}
{{ order.created_at|naturaltime }}  {# 3 hours ago #}
{{ event.date|naturalday }}         {# today, tomorrow, yesterday, or the date #}
{{ 3|ordinal }}                     {# 3rd #}
```

### redirects and sites

```python
INSTALLED_APPS += ["django.contrib.sites", "django.contrib.redirects"]
SITE_ID = 1
MIDDLEWARE += ["django.contrib.redirects.middleware.RedirectFallbackMiddleware"]
```

Create `Redirect` rows in the admin (`old_path="/old/dune/"`, `new_path="/books/7/"`). When a request would otherwise be a 404, the middleware looks for a matching redirect and answers with a 301.

### A custom system check

```python
from collections.abc import Sequence
from typing import Any

from django.apps import AppConfig
from django.core.checks import CheckMessage, Error, register


@register("config")
def check_something(app_configs: Sequence[AppConfig] | None, **kwargs: Any) -> list[CheckMessage]:
    errors: list[CheckMessage] = []
    if not settings.SOMETHING:
        errors.append(Error("SOMETHING must be set.", hint="Set SOMETHING in the environment.", id="config.E100"))
    return errors
```

```bash
uv run python manage.py check
uv run python manage.py check --deploy --fail-level WARNING
```

## Deeper behavior

### How loaddata works

`loaddata` finds the named fixture files, deserializes the objects, and saves each one inside a transaction, with constraint checks deferred until the end so that objects can refer to each other in any order. Each object is saved with `raw=True`, which is also passed to `pre_save` and `post_save` receivers. Receivers that create related objects or call other code must skip raw saves (note 5.3). A fixture whose primary keys collide with unrelated existing rows overwrites them, which is why fixtures with fixed keys are safe for reference data you own and dangerous for anything else.

### Fixtures in migrations

Reference data that must exist with a schema version belongs in a data migration (note 2.3), because migrations run in order and see the model state of their time. A fixture loaded later may not match an older schema. A common approach is a `RunPython` migration that creates the rows with `get_or_create`, or a management command (note 5.2) that is part of deployment and is idempotent.

### dumpdata is not a backup tool

`dumpdata` serializes through the ORM, skips database features Django does not know about, loads everything through Python, and can run out of memory on large tables. For backups use the database's own tools (`pg_dump`, note 11.1) and test restores. Use `dumpdata` for small, deliberate exports.

### Serialization pitfalls

- Serialized data contains only the concrete columns of each model. It has no behavior, and it does not enforce your service-layer rules when loaded.
- Many-to-many data is serialized as lists of keys, and foreign keys as raw keys or natural keys.
- Loading serialized data from untrusted sources creates arbitrary rows with arbitrary primary keys. Only deserialize data you trust, and never use the serializers as an import format for user-uploaded files (note 5.2's CSV import validates each row through your own code).
- The XML and YAML serializers have a larger attack surface than JSON. Prefer JSON.

### Natural keys in practice

A natural key must be unique and stable. Define `natural_key()` on the model and `get_by_natural_key()` on its default manager. If the natural key refers to a related object, set a `natural_key.dependencies` attribute so Django serializes the dependencies first. Natural keys make fixtures portable between databases, at the cost of a lookup per object when loading.

### Sitemaps in detail

- `items()` returns the objects, `location(obj)` returns the URL path (defaulting to `obj.get_absolute_url()`), and `lastmod`, `changefreq`, and `priority` give hints to search engines.
- The sitemap protocol limits one file to 50,000 URLs and 50 MB. Django paginates automatically with `limit`, and the `index` and `sitemap` views produce a sitemap index with sections.
- Include only canonical, public, indexable pages. Do not list drafts, soft-deleted rows, private pages, or pages that redirect.
- Reference the sitemap from `robots.txt`, and cache the view, because generating it can touch many rows.
- The site's domain comes from the `sites` framework if installed, or from the request.

### Feeds in detail

The `Feed` class builds RSS 2.0 by default, and Atom through `feed_type = Atom1Feed`. Provide `item_guid` or rely on `item_link` for stable identifiers, set `item_pubdate` for ordering, and call `select_related` in `items()` when `item_description` touches relations, or the feed issues one query per item (note 2.7). Feeds are public pages, so apply the same visibility rules as the site, and consider conditional GET headers or caching for busy feeds.

### humanize caveats

The filters produce strings in the active language, so test them with translations active. `naturaltime` depends on the current time and time zone, so tests must control the clock. For data shown in tables or exported, use exact values.

### Redirects and sites in detail

`RedirectFallbackMiddleware` runs only when a response would be a 404, so valid URLs are never affected. It stores redirects per `Site`, which is why `django.contrib.sites` and `SITE_ID` are required. Redirects are valuable when you restructure URLs, because old links from search engines and other sites keep working. For a few well-known redirects, URL patterns with `RedirectView` are simpler and faster, because they need no database query.

### Is the sites framework needed?

Only when one project serves several domains with different data, or when a contrib app that you use requires it (redirects). Many projects never need it. Sitemaps and feeds fall back to the request's host when `sites` is not installed.

### django.contrib.postgres

Beyond fields like `ArrayField` and `DateRangeField` (note 2.10), the package provides full-text search (`SearchVector`, `SearchQuery`, `SearchRank`), trigram similarity (`TrigramSimilarity`), aggregates such as `ArrayAgg` and `StringAgg`, extension operations (`TrigramExtension`, `BtreeGistExtension`), and exclusion constraints. These tie your schema to PostgreSQL, which is a reasonable decision for most production Django projects (note 11.1 goes deeper).

### Writing checks

A check is a function that receives `app_configs` and keyword arguments and returns a list of `CheckMessage` objects. Registered with `@register(...)`, it runs with the others. Good checks are fast, do not touch the database, and give an `id` (an app prefix plus `E`, `W`, or `I` and a number), a clear message, and a `hint` that says how to fix the problem. Tags select checks by category (`"security"`, `"database"`, or your own), and `deploy=True` limits a check to `check --deploy`. Reserve errors for problems that make the project unsafe or broken, because errors block `runserver` and `migrate`.

### Silencing checks

`SILENCED_SYSTEM_CHECKS = ["config.W001"]` hides specific messages in settings, for the rare case where a warning does not apply. Record why next to the setting.

### Checks in CI

Run `manage.py check --deploy --fail-level WARNING` with production-like settings in your pipeline (note 10.4). It catches insecure defaults and missing settings before they reach a server.

## Industry-level snippet

A sitemap and a feed for books, reference data from a fixture, a JSON export function that limits fields, and custom checks that validate the replica configuration from note 2.12 (an error) and the session cookie name in production (a deploy warning). The checks are split into a pure function that is easy to test and a thin registered wrapper that reads settings.

Add to `apps/bookstore/models.py`:

```python
from django.urls import reverse

# in Book
def get_absolute_url(self) -> str:
    return reverse("bookstore:book_detail", kwargs={"pk": self.pk})
```

`apps/bookstore/sitemaps.py`:

```python
from datetime import datetime

from django.contrib.sitemaps import Sitemap
from django.db.models import QuerySet

from apps.bookstore.models import Book


class BookSitemap(Sitemap):
    changefreq = "weekly"
    priority = 0.6
    limit = 5000

    def items(self) -> QuerySet[Book]:
        # Only public, canonical pages: soft-deleted books are not listed.
        return Book.objects.alive().order_by("pk")

    def lastmod(self, obj: Book) -> datetime:
        return obj.created_at
```

`apps/bookstore/feeds.py`:

```python
from datetime import datetime

from django.contrib.syndication.views import Feed
from django.urls import reverse_lazy

from apps.bookstore.models import Book

FEED_LENGTH = 20


class NewBooksFeed(Feed):
    title = "New books"
    link = reverse_lazy("bookstore:search")
    description = "The newest books in the catalog."

    def items(self) -> list[Book]:
        books = Book.objects.alive().select_related("author")
        return list(books.order_by("-created_at", "-pk")[:FEED_LENGTH])

    def item_title(self, item: Book) -> str:
        return item.title

    def item_description(self, item: Book) -> str:
        return f"{item.title} by {item.author.name}"

    def item_pubdate(self, item: Book) -> datetime:
        return item.created_at
```

In `config/urls.py`:

```python
from django.contrib.sitemaps.views import sitemap

from apps.bookstore.feeds import NewBooksFeed
from apps.bookstore.sitemaps import BookSitemap

urlpatterns += [
    path(
        "sitemap.xml",
        sitemap,
        {"sitemaps": {"books": BookSitemap}},
        name="django.contrib.sitemaps.views.sitemap",
    ),
    path("feeds/new-books/", NewBooksFeed(), name="new_books_feed"),
]
```

Add `"django.contrib.sitemaps"` to `INSTALLED_APPS`.

`apps/bookstore/export.py`:

```python
"""Small, deliberate data exports."""

import json
from collections.abc import Sequence
from typing import Any

from django.core import serializers
from django.db.models import QuerySet

from apps.bookstore.models import Book

EXPORTABLE_FIELDS = frozenset({"title", "isbn", "price", "stock", "format", "published_on"})


def export_books_json(
    queryset: QuerySet[Book],
    *,
    fields: Sequence[str] = ("title", "isbn", "price", "stock"),
) -> list[dict[str, Any]]:
    """Serialize books to Python data, limited to an allowlist of fields."""
    unknown = set(fields) - EXPORTABLE_FIELDS
    if unknown:
        raise ValueError(f"Fields cannot be exported: {sorted(unknown)}")
    text = serializers.serialize("json", queryset, fields=tuple(fields))
    data: list[dict[str, Any]] = json.loads(text)
    return data
```

`apps/core/apps.py`:

```python
from django.apps import AppConfig


class CoreConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.core"

    def ready(self) -> None:
        from apps.core import checks  # noqa: F401  (registers the checks)
```

`apps/core/checks.py`:

```python
"""Project-specific system checks."""

from collections.abc import Collection, Sequence
from typing import Any

from django.apps import AppConfig
from django.conf import settings
from django.core.checks import CheckMessage, Error, Tags, Warning, register

HOST_PREFIX = "__Host-"


def find_replica_problems(configured: Collection[str], replicas: Sequence[str]) -> list[CheckMessage]:
    """Pure function: compare the replica list with the configured databases."""
    problems: list[CheckMessage] = []
    for alias in replicas:
        if alias == "default":
            problems.append(
                Error(
                    "REPLICA_DATABASES must not contain 'default'.",
                    hint="'default' is the primary; list only read replicas.",
                    id="config.E002",
                )
            )
        elif alias not in configured:
            problems.append(
                Error(
                    f"REPLICA_DATABASES lists {alias!r}, but DATABASES has no such alias.",
                    hint="Add the alias to DATABASES or remove it from REPLICA_DATABASES.",
                    id="config.E001",
                )
            )
    return problems


@register("config")
def check_replica_aliases(
    app_configs: Sequence[AppConfig] | None, **kwargs: Any
) -> list[CheckMessage]:
    return find_replica_problems(
        set(settings.DATABASES), tuple(getattr(settings, "REPLICA_DATABASES", ()))
    )


@register(Tags.security, deploy=True)
def check_session_cookie_name(
    app_configs: Sequence[AppConfig] | None, **kwargs: Any
) -> list[CheckMessage]:
    if settings.SESSION_COOKIE_SECURE and not settings.SESSION_COOKIE_NAME.startswith(HOST_PREFIX):
        return [
            Warning(
                "The session cookie does not use the __Host- prefix.",
                hint="Set SESSION_COOKIE_NAME to '__Host-sessionid' (see note 4.1).",
                id="config.W001",
            )
        ]
    return []
```

`apps/bookstore/tests/test_contrib.py`:

```python
import json
from decimal import Decimal

from django.core.checks import Error
from django.core.management import call_command
from django.core.management.base import SystemCheckError
from django.db import connection
from django.test import SimpleTestCase, TestCase, override_settings
from django.test.utils import CaptureQueriesContext
from django.urls import reverse
from django.utils import timezone

from apps.bookstore.export import export_books_json
from apps.bookstore.models import Book, Genre
from apps.core.checks import (
    check_replica_aliases,
    check_session_cookie_name,
    find_replica_problems,
)

from .factories import make_author, make_book


class FixtureTests(TestCase):
    fixtures = ["genres.json"]

    def test_reference_data_is_loaded_with_its_relations(self) -> None:
        self.assertEqual(Genre.objects.count(), 3)
        epic = Genre.objects.get(name="Epic fantasy")
        self.assertEqual(epic.parent, Genre.objects.get(name="Fantasy"))

    def test_loading_the_fixture_again_is_repeatable(self) -> None:
        call_command("loaddata", "genres", verbosity=0)
        self.assertEqual(Genre.objects.count(), 3)


class ExportTests(TestCase):
    def test_only_the_requested_fields_are_exported(self) -> None:
        make_book(title="Dune", price=Decimal("12.50"), stock=3)
        data = export_books_json(Book.objects.all(), fields=("title", "price"))
        self.assertEqual(len(data), 1)
        self.assertEqual(data[0]["model"], "bookstore.book")
        self.assertEqual(data[0]["fields"], {"title": "Dune", "price": "12.50"})

    def test_fields_outside_the_allowlist_are_refused(self) -> None:
        with self.assertRaises(ValueError):
            export_books_json(Book.objects.all(), fields=("title", "deleted_at"))


class SitemapAndFeedTests(TestCase):
    def test_the_sitemap_lists_public_books_only(self) -> None:
        live = make_book(title="Live")
        hidden = make_book(title="Hidden")
        hidden.deleted_at = timezone.now()
        hidden.save(update_fields=["deleted_at"])

        response = self.client.get(reverse("django.contrib.sitemaps.views.sitemap"))

        self.assertEqual(response.status_code, 200)
        body = response.content.decode()
        self.assertIn(live.get_absolute_url(), body)
        self.assertNotIn(hidden.get_absolute_url(), body)

    def test_the_feed_is_rss_with_the_newest_books_and_a_fixed_length(self) -> None:
        for index in range(25):
            make_book(title=f"Book {index:02d}")
        response = self.client.get(reverse("new_books_feed"))
        self.assertIn("application/rss+xml", response["Content-Type"])
        body = response.content.decode()
        self.assertEqual(body.count("<item>"), 20)
        self.assertIn("Book 24", body)
        self.assertNotIn("Book 00", body)

    def test_the_feed_does_not_query_per_item(self) -> None:
        def queries_for_feed() -> int:
            with CaptureQueriesContext(connection) as context:
                self.client.get(reverse("new_books_feed"))
            return len(context)

        for index in range(3):
            make_book(author=make_author(f"Few {index}"))
        few = queries_for_feed()
        for index in range(10):
            make_book(author=make_author(f"Many {index}"))
        many = queries_for_feed()
        self.assertEqual(few, many)


class SystemCheckTests(SimpleTestCase):
    def test_unknown_and_primary_aliases_are_errors(self) -> None:
        problems = find_replica_problems({"default", "replica"}, ["replica", "ghost", "default"])
        self.assertEqual(sorted(problem.id or "" for problem in problems), ["config.E001", "config.E002"])
        self.assertTrue(all(isinstance(problem, Error) for problem in problems))

    def test_valid_configuration_has_no_problems(self) -> None:
        self.assertEqual(find_replica_problems({"default", "replica"}, ["replica"]), [])

    @override_settings(REPLICA_DATABASES=("ghost",))
    def test_the_registered_check_reads_the_settings(self) -> None:
        self.assertEqual([m.id for m in check_replica_aliases(None)], ["config.E001"])

    @override_settings(REPLICA_DATABASES=("ghost",))
    def test_manage_check_fails_for_a_bad_configuration(self) -> None:
        with self.assertRaises(SystemCheckError):
            call_command("check", "config")

    @override_settings(SESSION_COOKIE_SECURE=True, SESSION_COOKIE_NAME="sessionid")
    def test_the_deploy_check_warns_about_an_unprefixed_cookie(self) -> None:
        self.assertEqual([m.id for m in check_session_cookie_name(None)], ["config.W001"])

    @override_settings(SESSION_COOKIE_SECURE=True, SESSION_COOKIE_NAME="__Host-sessionid")
    def test_a_prefixed_cookie_passes(self) -> None:
        self.assertEqual(check_session_cookie_name(None), [])
```

The check tests show the useful pattern: logic in a pure function that takes plain values, and a one-line registered wrapper that reads settings. The `call_command("check", "config")` test proves the whole chain, from registration to a failing command.

## Common mistakes

- Using fixtures as the main source of test data, then fighting stale JSON after every schema change.
- Using `dumpdata` and `loaddata` as a backup and restore strategy for production.
- Loading fixtures with fixed primary keys over tables that already contain unrelated data.
- Writing `post_save` receivers that break on `raw=True` and make `loaddata` fail.
- Deserializing data from untrusted sources, or using the serializers as a user-facing import format.
- Listing private, draft, duplicate, or redirecting URLs in a sitemap.
- Building a feed whose `items()` queries the database once per item.
- Forgetting `get_absolute_url()` on models used by sitemaps.
- Installing the sites framework and redirects middleware when a few `RedirectView` routes would do.
- Writing system checks that query the database or take seconds to run.
- Using `Error` for problems that are only advice, which blocks `runserver` and `migrate`.
- Ignoring `check --deploy` output until the first production incident.
- Silencing a check without writing down the reason.

## Practice exercises

1. Add natural keys to `Genre` (and its manager), dump the genres with `--natural-foreign --natural-primary`, load them into a fresh database where primary keys differ, and confirm the parent relation survives. Write the test.
2. Write a data migration that creates the standard genres with `get_or_create`, and a test with `MigrationExecutor` (note 2.3) that it is idempotent. Compare it with the fixture approach for the same data.
3. Add `django.contrib.humanize` and show order totals with `intcomma` and creation times with `naturaltime`. Write a test that controls the clock and checks the output.
4. Replace the single sitemap with a sitemap index containing a section for books and a section for static pages (home, search), using the `index` and `sitemap` views, and test both URLs.
5. Add `django.contrib.redirects` and the middleware, create a redirect in a test, and show that a missing URL redirects with a 301 while a valid URL is unaffected. Then replace it with a `RedirectView` route and compare the query counts.
6. Write an `export_orders_json` function in the style of `export_books_json`, with its own field allowlist that excludes `idempotency_key` and the customer's email, and tests that prove excluded fields cannot be requested.
7. Write a deploy check `config.W002` that warns when `ALLOWED_HOSTS` contains a wildcard (`"*"`) while `DEBUG` is false, as a pure function plus a registered wrapper, and test both layers.

## Summary

Django's serialization framework converts model data to JSON and back, and `dumpdata` and `loaddata` use it to move small, trusted sets of data, with natural keys for portability. Fixtures suit stable reference data and are a poor fit for tests, where factories keep the data next to the test and follow the schema. They are not a backup tool, and `loaddata` saves objects with `raw=True`, so receivers must handle it. The contrib apps cover common needs: sitemaps and feeds (list only public, canonical content and avoid per-item queries), humanize filters, URL redirects and the sites framework when you actually need them, and PostgreSQL features once you commit to that database. System checks protect your configuration: write them as fast, database-free functions with ids and hints, split pure logic from the registered wrapper, reserve errors for real breakage, tag production audits with `deploy=True`, and run `check --deploy` in CI.

