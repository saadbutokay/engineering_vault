## Overview

A class-based view (CBV) expresses a view as a class instead of a function. Its main benefit is reuse: Django ships generic views that implement common patterns (show a template, list objects, show one object, handle a form), and mixins let you combine behaviors. The cost is indirection: behavior is spread across several parent classes, so you need to know which method to override. This note explains how a CBV turns into a callable, how `dispatch` routes requests, how generic views and mixins are composed, how to type them, and when a plain function is the better choice.

## Core concepts

### View and as_view()

`django.views.View` is the base class. You define methods named after HTTP methods (`get`, `post`, `put`, `delete`). Because URL patterns need a callable, you call `MyView.as_view()`, which returns a function that creates a fresh instance of your class for every request and calls its `dispatch` method.

### dispatch

`dispatch(request, *args, **kwargs)` looks at `request.method` and calls the matching handler. If the class has no method for it, `dispatch` returns a 405 response. Overriding `dispatch` lets you run code before every handler.

### Generic views

- `TemplateView`: render a template with a context.
- `ListView`: render a list of objects, with pagination.
- `DetailView`: render one object looked up from the URL.
- `RedirectView`: redirect to another URL.
- `FormView`, `CreateView`, `UpdateView`, `DeleteView`: forms and editing (Phase 3).

### Key attributes and methods

- `template_name`: which template to render.
- `model` or `queryset`: where objects come from.
- `get_queryset()`: override to filter or order dynamically, for example by query parameter or user.
- `get_context_data(**kwargs)`: override to add values to the template context.
- `paginate_by`: how many items per page in a `ListView`.
- `context_object_name`: the template variable name for the object or list.
- `slug_field` and `slug_url_kwarg` (or `pk_url_kwarg`): how `DetailView` finds its object.

### Mixins and method resolution order

A mixin is a class that adds behavior and is not meant to be used alone. Python resolves methods from left to right across the base classes (the method resolution order, MRO). Mixins go to the left of the generic view so that their methods run first and can call `super()`.

### Function or class

Prefer a class when you want to reuse generic behavior or share logic through mixins. Prefer a function when the logic is unique, procedural, or involves several unrelated steps. A short, readable function beats a class that overrides five methods to bend a generic view out of shape.

## Basic syntax examples

### A View subclass

```python
from django.http import HttpRequest, HttpResponse
from django.views import View


class HelloView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        return HttpResponse("Hello from a class")

    def post(self, request: HttpRequest) -> HttpResponse:
        return HttpResponse("Posted", status=201)
```

```python
from django.urls import path

from .views import HelloView

urlpatterns = [path("hello/", HelloView.as_view(), name="hello")]
```

### The same view as a function

```python
from django.http import HttpRequest, HttpResponse
from django.views.decorators.http import require_http_methods


@require_http_methods(["GET", "POST"])
def hello(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        return HttpResponse("Posted", status=201)
    return HttpResponse("Hello from a function")
```

### TemplateView

```python
from django.views.generic import TemplateView


class AboutView(TemplateView):
    template_name = "pages/about.html"
    extra_context = {"team_size": 12}
```

`extra_context` must only contain static values. Anything that depends on the request belongs in `get_context_data`.

### ListView and DetailView

```python
from django.views.generic import DetailView, ListView

from .models import Article


class ArticleListView(ListView):
    model = Article
    paginate_by = 10


class ArticleDetailView(DetailView):
    model = Article
```

With defaults, `ListView` renders `articles/article_list.html` with the variables `article_list`, `object_list`, `page_obj`, `paginator`, and `is_paginated`. `DetailView` renders `articles/article_detail.html` with `article` and `object`, finding the row by the `slug` or `pk` URL keyword.

### Overriding the queryset and context

```python
from typing import Any

from django.db.models import QuerySet


class PublishedArticleListView(ListView[Article]):
    template_name = "articles/article_list.html"

    def get_queryset(self) -> QuerySet[Article]:
        return Article.objects.filter(status=Article.Status.PUBLISHED)

    def get_context_data(self, **kwargs: Any) -> dict[str, Any]:
        context = super().get_context_data(**kwargs)
        context["heading"] = "Published articles"
        return context
```

The `ListView[Article]` subscript relies on the type-hint setup explained in the next section.

### Type hints for generic views

Django's classes are not generic at runtime, but `django-stubs` declares them as generic so that `mypy` knows which model a view uses. The helper package `django-stubs-ext` makes the subscript syntax work at runtime. It is installed as a dependency of `django-stubs`. In `config/settings.py`, before any views are imported:

```python
import django_stubs_ext

django_stubs_ext.monkeypatch()
```

Then subscript the generic views:

```python
class ArticleListView(ListView[Article]):
    model = Article
```

## Deeper behavior

### One instance per request

`as_view()` creates a new instance for every request, so instance attributes such as `self.request` and `self.kwargs` are safe. Class attributes, however, are shared by all requests. Mutating a list or dictionary defined on the class leaks data between users. Pass configuration through immutable class attributes and keep per-request state on `self`.

### The setup sequence

For each request, Django creates the instance with the arguments given to `as_view(**initkwargs)`, then calls `setup()` (which assigns `self.request`, `self.args`, and `self.kwargs`), then `dispatch()`. If you override `setup()`, call `super().setup()` first.

### http_method_names

The class attribute `http_method_names` lists the methods the view accepts. To make a view read-only, set `http_method_names = ["get", "head", "options"]`.

### MRO in practice

```python
class A:
    def hello(self) -> str:
        return "A"


class B(A):
    def hello(self) -> str:
        return "B" + super().hello()


class C(A):
    def hello(self) -> str:
        return "C" + super().hello()


class D(B, C):
    pass


print(D().hello())    # BCA
print(D.__mro__)      # D, B, C, A, object
```

`super()` follows the MRO, not just the parent class. That is how several mixins cooperate: each calls `super()` and the chain continues. Always put mixins before the generic view, as in `class MyView(MyMixin, ListView)`.

### Which method to override

- Which objects? `get_queryset()`.
- Extra template variables? `get_context_data()`.
- Which template? `get_template_names()`.
- What to do before any handler? `dispatch()` or `setup()`.
- How to find the object? `get_object()`.

Reading the source on the Classy Class-Based Views website, or directly in Django's `django/views/generic/`, shows the full call order for any view.

### When generic views hide too much

If a page needs several unrelated models, side effects on GET, or conditional flows, you may find yourself overriding `get`, `get_context_data`, and `get_queryset` all at once. At that point a function is clearer, because the control flow is visible in one place.

## Industry-level snippet

A public article listing with category filtering, pagination, query-efficient fetching, only published articles, and a reusable mixin. It uses the `Article` and `Category` models from note 0.7.

`apps/articles/views.py`:

```python
import logging
from typing import Any

from django.db.models import QuerySet
from django.views.generic import DetailView, ListView
from django.views.generic.base import ContextMixin

from .models import Article

logger = logging.getLogger(__name__)


class PageTitleMixin(ContextMixin):
    """Add a ``page_title`` variable to the template context."""

    page_title: str = ""

    def get_page_title(self) -> str:
        return self.page_title

    def get_context_data(self, **kwargs: Any) -> dict[str, Any]:
        context = super().get_context_data(**kwargs)
        context["page_title"] = self.get_page_title()
        return context


def published_articles() -> QuerySet[Article]:
    return Article.objects.filter(status=Article.Status.PUBLISHED).select_related(
        "category"
    )


class ArticleListView(PageTitleMixin, ListView[Article]):
    paginate_by = 10
    page_title = "Articles"

    def get_selected_category(self) -> str:
        return self.request.GET.get("category", "").strip()

    def get_queryset(self) -> QuerySet[Article]:
        queryset = published_articles()
        category_slug = self.get_selected_category()
        if category_slug:
            queryset = queryset.filter(category__slug=category_slug)
        return queryset

    def get_context_data(self, **kwargs: Any) -> dict[str, Any]:
        context = super().get_context_data(**kwargs)
        context["selected_category"] = self.get_selected_category()
        return context


class ArticleDetailView(DetailView[Article]):
    queryset = published_articles()

    def get_context_data(self, **kwargs: Any) -> dict[str, Any]:
        context = super().get_context_data(**kwargs)
        article: Article = context["article"]
        context["page_title"] = article.title
        return context
```

`apps/articles/urls.py`:

```python
from django.urls import path

from . import views

app_name = "articles"

urlpatterns = [
    path("", views.ArticleListView.as_view(), name="list"),
    path("<slug:slug>/", views.ArticleDetailView.as_view(), name="detail"),
]
```

`apps/articles/templates/articles/article_list.html`:

```html
<h1>{{ page_title }}</h1>
<ul>
  {% for article in article_list %}
    <li><a href="{{ article.get_absolute_url }}">{{ article.title }}</a> ({{ article.category.name }})</li>
  {% empty %}
    <li>No articles yet.</li>
  {% endfor %}
</ul>
{% if is_paginated %}
  <nav>
    {% if page_obj.has_previous %}
      <a href="{% querystring page=page_obj.previous_page_number %}">Previous</a>
    {% endif %}
    <span>Page {{ page_obj.number }} of {{ paginator.num_pages }}</span>
    {% if page_obj.has_next %}
      <a href="{% querystring page=page_obj.next_page_number %}">Next</a>
    {% endif %}
  </nav>
{% endif %}
```

The `{% querystring %}` tag was added in Django 5.1 and keeps existing query parameters such as `category` when changing the page. On Django 5.0 or earlier you would build the query string by hand.

`apps/articles/templates/articles/article_detail.html`:

```html
<article>
  <h1>{{ page_title }}</h1>
  <p>Category: {{ article.category.name }}</p>
  <div>{{ article.body|linebreaks }}</div>
</article>
```

`apps/articles/tests.py`:

```python
from typing import ClassVar

from django.test import TestCase
from django.urls import reverse

from .models import Article, Category


class ArticleViewTests(TestCase):
    news: ClassVar[Category]
    tech: ClassVar[Category]

    @classmethod
    def setUpTestData(cls) -> None:
        cls.news = Category.objects.create(name="News", slug="news")
        cls.tech = Category.objects.create(name="Tech", slug="tech")

    def make_article(
        self,
        slug: str,
        category: Category,
        status: str = Article.Status.PUBLISHED,
    ) -> Article:
        return Article.objects.create(
            title=f"Title {slug}",
            slug=slug,
            body="Body",
            category=category,
            status=status,
        )

    def test_list_shows_only_published_articles(self) -> None:
        self.make_article("live", self.news)
        self.make_article("hidden", self.news, Article.Status.DRAFT)
        response = self.client.get(reverse("articles:list"))
        self.assertContains(response, "Title live")
        self.assertNotContains(response, "Title hidden")

    def test_category_filter(self) -> None:
        self.make_article("a", self.news)
        self.make_article("b", self.tech)
        response = self.client.get(reverse("articles:list"), {"category": "tech"})
        self.assertContains(response, "Title b")
        self.assertNotContains(response, "Title a")

    def test_pagination_splits_results(self) -> None:
        for index in range(12):
            self.make_article(f"item-{index}", self.news)
        url = reverse("articles:list")
        self.assertContains(self.client.get(url), "<li>", count=10)
        self.assertContains(self.client.get(url, {"page": 2}), "<li>", count=2)

    def test_page_out_of_range_is_404(self) -> None:
        self.make_article("one", self.news)
        response = self.client.get(reverse("articles:list"), {"page": 999})
        self.assertEqual(response.status_code, 404)

    def test_list_uses_a_fixed_number_of_queries(self) -> None:
        for index in range(5):
            self.make_article(f"q-{index}", self.news)
        with self.assertNumQueries(2):  # one COUNT for pagination, one SELECT
            self.client.get(reverse("articles:list"))

    def test_detail_shows_published_and_hides_drafts(self) -> None:
        self.make_article("live", self.news)
        self.make_article("hidden", self.news, Article.Status.DRAFT)
        live = self.client.get(reverse("articles:detail", kwargs={"slug": "live"}))
        hidden = self.client.get(reverse("articles:detail", kwargs={"slug": "hidden"}))
        self.assertContains(live, "Title live")
        self.assertEqual(hidden.status_code, 404)
```

Notice the pieces: the mixin adds a feature once, `get_queryset` centralizes filtering, `select_related` avoids one query per row, and the draft rule lives in a single function (`published_articles`) shared by both views.

## Common mistakes

- Forgetting `.as_view()` in the URL pattern, which gives the error that the class is not callable.
- Placing mixins to the right of the generic view, so the generic view's method runs first and the mixin never takes effect.
- Mutating mutable class attributes, which leaks state between requests.
- Overriding `get_queryset` and forgetting to apply the filtering again in `get_object` or other methods that bypass it.
- Forgetting to call `super().get_context_data(**kwargs)` and losing the context the parent builds.
- Using `extra_context` for request-dependent values.
- Overriding many methods of a generic view when a short function would be clearer.
- Not optimizing queries, so a list view issues one extra query per row. Use `select_related` and `prefetch_related` (Phase 2).
- Setting `model = Article` and then overriding `get_queryset` with a different model, creating confusing behavior.

## Practice exercises

1. Convert the `hello` function view in this note to a `View` subclass and the `HelloView` class to a function. Verify both with the test client.
2. Write `TemplateView` subclass `HomeView` that adds the current year to the context in `get_context_data`. Write a test that controls the year.
3. Add `paginate_by` and a `search` query parameter to `ArticleListView` that filters titles case-insensitively. Test blank and non-blank searches.
4. Print `ArticleListView.__mro__` in the shell and list the classes in order. Identify which class supplies `get_context_data`, `get_queryset`, and `paginate_queryset`.
5. Create a mixin `JsonOnlyMixin` that overrides `dispatch` to return 406 unless the `Accept` header allows JSON. Combine it with a `View` subclass and test it.
6. Demonstrate the shared class attribute bug: write a view with a class-level list that appends on each request, show the leak, and fix it.
7. Re-implement `ArticleDetailView` as a function using `get_object_or_404` and compare the amount of code and the clarity of the logic.

## Summary

A class-based view is turned into a callable by `as_view()`, which builds a new instance per request and routes through `dispatch` to a method named after the HTTP verb. Generic views such as `TemplateView`, `ListView`, and `DetailView` implement common patterns, customized by overriding `get_queryset`, `get_context_data`, and a handful of attributes. Mixins add shared behavior and must precede the generic view in the base list because of the MRO. Add type parameters with `django-stubs-ext`. Choose classes for reuse of generic behavior and functions when the logic is specific, because readable control flow is worth more than saved lines.
