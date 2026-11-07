## Overview

Consistent style makes code readable, reviews faster, and onboarding easier. Django has its own coding style guide and naming habits on top of PEP 8, and modern tooling enforces most of it automatically. This note covers the conventions you should follow from day one: formatting with `ruff format`, import ordering, naming for models, fields, related names, URL names, and templates, where logic goes in a first pass, and how `django-upgrade` modernizes code. It ends with a small, tested example that applies everything.

## Core concepts

### PEP 8 and Django's style guide

- PEP 8 is Python's baseline: four-space indentation, snake_case for functions and variables, PascalCase for classes, UPPER_CASE for constants.
- Django's coding style builds on it and adds conventions such as a consistent order inside model classes and preferring explicit over clever code.
- `ruff format` applies a Black-compatible style automatically, so formatting is never a manual discussion.

### Import ordering

Imports are grouped in this order, with a blank line between groups:

1. Standard library.
2. Third-party packages (including Django).
3. Your own project code.

Ruff's `I` rules (isort-compatible) sort and group imports for you.

### Naming conventions

- Models: singular PascalCase nouns. `Article`, not `Articles`.
- Fields: snake_case, descriptive, no repeated model name (`title`, not `article_title`).
- Foreign keys: named after the related thing in the singular, such as `category`, which creates the column `category_id`.
- `related_name`: the plural name used from the other side, such as `articles` so that `category.articles.all()` reads naturally.
- Booleans: start with `is_`, `has_`, or `can_`.
- Timestamps: end with `_at` (`created_at`), and dates with `_on`.
- URL names: lowercase with words separated by underscores, namespaced by app (`articles:detail`).
- Templates: `<app>/<model>_<purpose>.html`, such as `articles/article_detail.html`.
- Views: verbs or purposes, such as `article_detail` for functions and `ArticleDetailView` for classes.

### Model class ordering

A conventional order inside a model:

1. Choices classes.
2. Fields.
3. Custom manager declarations.
4. `class Meta`.
5. `__str__`.
6. `save`, `get_absolute_url`, and other overrides.
7. Custom methods and properties.

### Where logic goes in a first pass

- Validation of one field: the field or a validator.
- Rules about stored data: the model, constraints, or a service function.
- Reading the request and choosing the response: the view.
- Presentation: the template.
- Everything else: a plain function in a module, such as a service.

Later phases refine this, but this default prevents most early mess.

### django-upgrade

`django-upgrade` is a tool that automatically rewrites code to use newer Django APIs when you raise the minimum supported version. It makes upgrades mechanical instead of manual.

## Basic syntax examples

### Naming in practice

```python
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)

    def __str__(self) -> str:
        return self.name


class Article(models.Model):
    title = models.CharField(max_length=200)
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,
        related_name="articles",
    )
    is_featured = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self) -> str:
        return self.title
```

### Import grouping

```python
import logging
from datetime import datetime

from django.http import HttpRequest, HttpResponse
from django.shortcuts import get_object_or_404, render

from apps.articles.models import Article
```

### Running the formatter and linter

```bash
uv run ruff format .
uv run ruff check . --fix
```

### Ruff configuration

In `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # import sorting
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # bugbear
    "DJ",   # flake8-django
]
```

### Using django-upgrade

```bash
uv add --dev django-upgrade
uv run django-upgrade --target-version 5.2 $(git ls-files '*.py')
```

On Windows PowerShell:

```powershell
uv run django-upgrade --target-version 5.2 (git ls-files '*.py')
```

For example, when targeting Django 5.1 or newer, it rewrites the old constraint argument to the new name:

```python
# before
models.CheckConstraint(check=models.Q(price__gte=0), name="price_non_negative")

# after
models.CheckConstraint(condition=models.Q(price__gte=0), name="price_non_negative")
```

## Deeper behavior

### Why formatters end style debates

A formatter has no opinions to argue with: it produces the same output for the same input. Running it on save and in CI means reviews focus on behavior. The `ruff format` command is intentionally compatible with Black's style, so existing Black habits carry over.

### Lint rules that matter for Django

The `DJ` rule set flags common Django mistakes. Examples:

- A model that defines no `__str__`.
- Using `exclude` in a `ModelForm` `Meta` class instead of listing fields.
- Using `locals()` as a template context.
- Wrong ordering of elements inside a model class.

### Naming and `related_name`

If you omit `related_name`, Django generates `article_set` on the other side. That works, but `category.articles` reads better and is self-documenting. When a model has two foreign keys to the same model, `related_name` is mandatory to avoid clashes.

### Singular model names and plural tables

Models are singular because one instance represents one row. Django pluralizes the verbose name in the admin automatically, and the table name is `<app_label>_<model>` in lowercase, such as `articles_article`.

### Explicit is better than implicit

Prefer `choices=Status` with `TextChoices` to magic strings scattered through the code. Prefer `reverse("articles:detail", kwargs={"slug": slug})` to string-built URLs. Prefer named arguments for functions with several parameters of the same type.

### Type hints

Annotate function signatures, especially views (`HttpRequest` in, `HttpResponse` out). Type hints are documentation that tools verify. Note 0.9 shows how `mypy` enforces them.

## Industry-level snippet

A small catalog feature following all conventions: singular models, descriptive related names, namespaced URL names, a typed view, a template, and tests.

`apps/articles/models.py`:

```python
from django.db import models
from django.urls import reverse


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        ordering = ["name"]
        verbose_name_plural = "categories"

    def __str__(self) -> str:
        return self.name


class Article(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    body = models.TextField()
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,
        related_name="articles",
    )
    status = models.CharField(
        max_length=10,
        choices=Status,
        default=Status.DRAFT,
    )
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self) -> str:
        return self.title

    def get_absolute_url(self) -> str:
        return reverse("articles:detail", kwargs={"slug": self.slug})
```

`apps/articles/views.py`:

```python
import logging

from django.http import Http404, HttpRequest, HttpResponse
from django.shortcuts import get_object_or_404, render

from .models import Article

logger = logging.getLogger(__name__)


def article_detail(request: HttpRequest, slug: str) -> HttpResponse:
    """Show a published article, hiding drafts behind a 404."""
    article = get_object_or_404(
        Article.objects.select_related("category"),
        slug=slug,
    )
    if article.status != Article.Status.PUBLISHED:
        logger.info("Draft article %s requested publicly", slug)
        raise Http404("Article not found")
    return render(request, "articles/article_detail.html", {"article": article})
```

`apps/articles/urls.py`:

```python
from django.urls import path

from . import views

app_name = "articles"

urlpatterns = [
    path("<slug:slug>/", views.article_detail, name="detail"),
]
```

`apps/articles/templates/articles/article_detail.html`:

```html
<article>
  <h1>{{ article.title }}</h1>
  <p>Category: {{ article.category.name }}</p>
  <div>{{ article.body|linebreaks }}</div>
</article>
```

`apps/articles/tests.py`:

```python
from django.test import TestCase
from django.urls import reverse

from .models import Article, Category


class ArticleDetailViewTests(TestCase):
    def setUp(self) -> None:
        self.category = Category.objects.create(name="News", slug="news")

    def _create(self, slug: str, status: str) -> Article:
        return Article.objects.create(
            title=f"Title {slug}",
            slug=slug,
            body="Body text",
            category=self.category,
            status=status,
        )

    def test_published_article_is_shown(self) -> None:
        article = self._create("hello", Article.Status.PUBLISHED)
        response = self.client.get(article.get_absolute_url())
        self.assertContains(response, "Title hello")
        self.assertContains(response, "News")

    def test_draft_article_returns_404(self) -> None:
        self._create("secret", Article.Status.DRAFT)
        response = self.client.get(reverse("articles:detail", kwargs={"slug": "secret"}))
        self.assertEqual(response.status_code, 404)

    def test_unknown_slug_returns_404(self) -> None:
        response = self.client.get(reverse("articles:detail", kwargs={"slug": "nope"}))
        self.assertEqual(response.status_code, 404)

    def test_related_name_gives_readable_access(self) -> None:
        self._create("one", Article.Status.PUBLISHED)
        self.assertEqual(self.category.articles.count(), 1)
```

Notice how the code reads: `category.articles`, `articles:detail`, `Article.Status.PUBLISHED`. None of it needs a comment to explain what the names mean.

## Common mistakes

- Plural model names such as `Articles`, which produce confusing code like `Articles.objects.get(...)`.
- Prefixing every field with the model name (`article_title`).
- Skipping `related_name` and living with `article_set` forever.
- Hard-coding URLs like `"/articles/hello/"` instead of using named routes.
- Mixing import styles and orders because the formatter was never configured.
- Using wildcard imports (`from module import *`), which hide where names come from.
- Defining magic strings such as `"published"` in many places instead of one `TextChoices` class.
- Treating lint warnings as noise and disabling the rules instead of reading them.

## Practice exercises

1. Rename the following to follow conventions: model `Customers`, field `customer_name`, boolean `active`, timestamp `creation_date`, URL name `ShowOrder`. Explain each change.
2. Take a Python file with unsorted imports and inconsistent formatting, run Ruff on it, and describe every change made.
3. Add the `DJ` rule set to a project and fix every violation it reports in a model you wrote earlier.
4. Create a `Tag` model with a many-to-many relationship to `Article`, choosing a `related_name`, and write a test that proves `tag.articles` works.
5. Write a view function with full type hints that returns a `JsonResponse` listing the titles of published articles in a category, returning 404 for an unknown category slug. Add tests.
6. Introduce a deprecated argument (the old `check=` form of `CheckConstraint`) in a scratch file, run `django-upgrade`, and explain the rewrite.
7. Write a one-page personal style checklist for pull requests, covering naming, imports, type hints, and where logic goes.

## Summary

Follow PEP 8 as applied by `ruff format`, group imports as standard library, third party, and local, and let tooling handle the mechanics. Name models in the singular, fields descriptively, foreign keys after the related thing, `related_name` as the plural from the other side, and URL names with an app namespace. Order the contents of model classes predictably, keep first-pass logic in the right layer, annotate functions with types, and use `django-upgrade` to modernize code whenever you raise the minimum Django version.
