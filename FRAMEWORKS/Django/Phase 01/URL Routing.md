## Overview

URL routing is how Django decides which view handles a request. The mapping lives in a URL configuration (urlconf): a Python list of patterns that Django checks from top to bottom until one matches. Good routing keeps URLs stable and readable, and good habits (names, namespaces, `reverse`) mean you can change a URL in one place without breaking links anywhere else. This note covers `path()`, `re_path()`, converters, `include()`, namespaces, reversing, custom converters, trailing slashes, and the exact matching algorithm.

## Core concepts

### The urlconf

A urlconf is a module that defines a variable named `urlpatterns`, a list of `path()`, `re_path()`, or `include()` entries. The root urlconf is named by the `ROOT_URLCONF` setting, which is `config.urls` in this roadmap.

### path() and re_path()

- `path("articles/<int:year>/", view, name="year_archive")` uses simple route syntax with angle-bracket converters.
- `re_path(r"^articles/(?P<year>[0-9]{4})/$", view)` uses a full regular expression. Reach for it only when `path()` cannot express the pattern.

### Built-in path converters

- `str`: any non-empty string excluding the slash. This is the default when no converter is written.
- `int`: zero or a positive integer, passed to the view as `int`.
- `slug`: letters, digits, hyphens, and underscores.
- `uuid`: a lowercase UUID in the standard format, passed as `uuid.UUID`.
- `path`: any non-empty string including slashes. Useful for file paths.

### include() and app-level urlconfs

`include("apps.blog.urls")` delegates everything after the matched prefix to another urlconf. Each app owns its routes, and the project urlconf only decides the prefix.

### Names and namespaces

- `name="detail"` gives a pattern a name you can reverse.
- `app_name = "blog"` in an app's `urls.py` creates a namespace, so the full name is `blog:detail`. Two apps can both have a `detail` route without clashing.

### Reversing

`reverse("blog:detail", kwargs={"slug": "hello"})` builds the URL from the name. In templates the equivalent is `{% url "blog:detail" slug="hello" %}`. Never hard-code URLs: when a route changes, reversed URLs follow automatically.

### Trailing slashes

Django's convention is to end URLs with a slash. With `APPEND_SLASH = True` (the default) and `CommonMiddleware` enabled, a request to `/about` is redirected to `/about/` if only the slash version matches.

## Basic syntax examples

### A project urlconf

`config/urls.py`:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("articles/", include("apps.articles.urls")),
    path("", include("apps.pages.urls")),
]
```

### An app urlconf with a namespace

`apps/articles/urls.py`:

```python
from django.urls import path

from . import views

app_name = "articles"

urlpatterns = [
    path("", views.article_list, name="list"),
    path("<int:year>/", views.year_archive, name="year"),
    path("<slug:slug>/", views.article_detail, name="detail"),
]
```

### Receiving captured values

```python
from django.http import HttpRequest, HttpResponse


def year_archive(request: HttpRequest, year: int) -> HttpResponse:
    return HttpResponse(f"Articles from {year}")


def article_detail(request: HttpRequest, slug: str) -> HttpResponse:
    return HttpResponse(f"Article {slug}")
```

Captured values arrive as keyword arguments, converted by the converter (`year` is already an `int`).

### Reversing in Python and in templates

```python
from django.urls import reverse

reverse("articles:list")                              # /articles/
reverse("articles:year", kwargs={"year": 2026})       # /articles/2026/
reverse("articles:detail", kwargs={"slug": "hello"})  # /articles/hello/
```

```html
<a href="{% url 'articles:list' %}">All articles</a>
<a href="{% url 'articles:detail' slug=article.slug %}">{{ article.title }}</a>
```

### A regular expression route

```python
from django.urls import re_path

urlpatterns = [
    re_path(r"^archive/(?P<year>[0-9]{4})/(?P<month>0[1-9]|1[0-2])/$", views.month_archive),
]
```

Captured groups from `re_path()` are passed as strings, so convert them yourself.

### A custom converter

```python
from django.urls import path, register_converter


class FourDigitYearConverter:
    regex = "[0-9]{4}"

    def to_python(self, value: str) -> int:
        return int(value)

    def to_url(self, value: int) -> str:
        return f"{value:04d}"


register_converter(FourDigitYearConverter, "yyyy")

urlpatterns = [
    path("archive/<yyyy:year>/", views.year_archive, name="archive"),
]
```

## Deeper behavior

### How Django matches a URL

1. Django takes the request path and removes the leading slash and the script prefix. The query string and fragment play no part.
2. It walks `urlpatterns` from top to bottom.
3. For `include()`, it strips the matched prefix and continues matching the remainder against the included urlconf.
4. The first pattern that matches wins. Django calls the view with the captured values.
5. If nothing matches, Django raises `Resolver404`, which becomes a 404 response.

Because the first match wins, order matters. Put specific routes before general ones:

```python
urlpatterns = [
    path("new/", views.article_create, name="create"),    # must come first
    path("<slug:slug>/", views.article_detail, name="detail"),
]
```

If the order were reversed, `/new/` would be treated as an article with the slug `new`.

### Converter failures mean no match

If `to_python` raises `ValueError`, Django treats it as "this pattern does not match" and continues to the next pattern, eventually producing a 404. This makes converters a clean place for validation of the URL shape.

### Reversing failures

`reverse()` raises `NoReverseMatch` when no pattern fits the name and arguments. Common causes: a misspelled name, a missing namespace, wrong argument names, or a value that does not match the converter. In `to_url`, raising `ValueError` makes the reverse fail with `NoReverseMatch`.

You may pass `args` or `kwargs` to `reverse`, but not both.

### reverse_lazy

Class attributes and module-level constants are evaluated at import time, before URLs are loaded. Use `reverse_lazy` there:

```python
from django.urls import reverse_lazy

LOGIN_URL = reverse_lazy("accounts:login")
```

### Inspecting a match

```python
from django.urls import resolve

match = resolve("/articles/2026/")
print(match.view_name)  # articles:year
print(match.kwargs)     # {'year': 2026}
print(match.func)       # the view callable
```

### Trailing slash behavior

`APPEND_SLASH` redirects only when the URL without a slash fails to match and the same URL with a slash would match. The redirect is a GET-style redirect, so a POST to a URL without the slash loses its body. With `DEBUG = True`, Django raises a `RuntimeError` for that case to warn you. Always post to the canonical URL, which is why you should reverse URLs rather than type them.

### Namespaces in nested includes

Namespaces nest. If the project includes `blog.urls` under the namespace `blog`, and that urlconf includes another with `app_name = "comments"`, the name is `blog:comments:detail`. Instance namespaces (`include(..., namespace="x")`) allow the same app to be mounted twice, which is rare.

## Industry-level snippet

A realistic example: order URLs like `/orders/ORD-00000042/`, where a custom converter validates the format, converts to an integer, and refuses to build URLs for impossible numbers. Data comes from an in-memory stand-in for the database, which Phase 2 replaces with the ORM.

`apps/orders/converters.py`:

```python
"""URL converters for the orders app."""

ORDER_PREFIX = "ORD-"
MAX_ORDER_NUMBER = 99_999_999


class OrderNumberConverter:
    """Match ``ORD-`` followed by eight digits and expose the number as an int."""

    regex = r"ORD-[0-9]{8}"

    def to_python(self, value: str) -> int:
        return int(value.removeprefix(ORDER_PREFIX))

    def to_url(self, value: int) -> str:
        if not 0 <= value <= MAX_ORDER_NUMBER:
            raise ValueError(f"Order number out of range: {value}")
        return f"{ORDER_PREFIX}{value:08d}"
```

`apps/orders/data.py`:

```python
"""In-memory stand-in for the database, replaced by the ORM in Phase 2."""

from dataclasses import dataclass


@dataclass(frozen=True)
class Order:
    number: int
    customer: str
    items: tuple[str, ...]


ORDERS: dict[int, Order] = {
    1: Order(1, "Ada", ("keyboard", "mouse")),
    42: Order(42, "Grace", ("monitor",)),
}
```

`apps/orders/views.py`:

```python
import logging

from django.http import Http404, HttpRequest, JsonResponse
from django.urls import reverse

from .data import ORDERS

logger = logging.getLogger(__name__)


def order_list(request: HttpRequest) -> JsonResponse:
    results = [
        {
            "number": order.number,
            "url": reverse("orders:detail", kwargs={"number": order.number}),
        }
        for order in ORDERS.values()
    ]
    return JsonResponse({"results": results})


def order_detail(request: HttpRequest, number: int) -> JsonResponse:
    order = ORDERS.get(number)
    if order is None:
        logger.info("Order %s not found", number)
        raise Http404("Order not found")
    return JsonResponse(
        {"number": order.number, "customer": order.customer, "items": order.items}
    )


def order_item(request: HttpRequest, number: int, item_id: int) -> JsonResponse:
    order = ORDERS.get(number)
    if order is None or not 0 <= item_id < len(order.items):
        raise Http404("Order item not found")
    return JsonResponse({"order": number, "item": order.items[item_id]})
```

`apps/orders/urls.py`:

```python
from django.urls import path, register_converter

from . import views
from .converters import OrderNumberConverter

register_converter(OrderNumberConverter, "order_number")

app_name = "orders"

urlpatterns = [
    path("", views.order_list, name="list"),
    path("<order_number:number>/", views.order_detail, name="detail"),
    path(
        "<order_number:number>/items/<int:item_id>/",
        views.order_item,
        name="item",
    ),
]
```

Mount it in `config/urls.py` with `path("orders/", include("apps.orders.urls"))`.

`apps/orders/tests.py`:

```python
from django.test import SimpleTestCase
from django.urls import NoReverseMatch, resolve, reverse


class OrderUrlTests(SimpleTestCase):
    def test_reverse_formats_the_number(self) -> None:
        url = reverse("orders:detail", kwargs={"number": 42})
        self.assertEqual(url, "/orders/ORD-00000042/")

    def test_resolve_converts_to_int(self) -> None:
        match = resolve("/orders/ORD-00000042/")
        self.assertEqual(match.view_name, "orders:detail")
        self.assertEqual(match.kwargs, {"number": 42})

    def test_out_of_range_number_cannot_be_reversed(self) -> None:
        with self.assertRaises(NoReverseMatch):
            reverse("orders:detail", kwargs={"number": 10**9})

    def test_malformed_number_is_a_404(self) -> None:
        for path in ["/orders/42/", "/orders/ORD-1/", "/orders/ord-00000042/"]:
            with self.subTest(path=path):
                self.assertEqual(self.client.get(path).status_code, 404)

    def test_detail_returns_order(self) -> None:
        response = self.client.get(reverse("orders:detail", kwargs={"number": 42}))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["customer"], "Grace")

    def test_unknown_order_is_a_404(self) -> None:
        response = self.client.get("/orders/ORD-00000099/")
        self.assertEqual(response.status_code, 404)

    def test_item_route_uses_both_converters(self) -> None:
        url = reverse("orders:item", kwargs={"number": 1, "item_id": 1})
        self.assertEqual(url, "/orders/ORD-00000001/items/1/")
        self.assertEqual(self.client.get(url).json(), {"order": 1, "item": "mouse"})
```

## Common mistakes

- Hard-coding URLs in views and templates instead of using `reverse` and `{% url %}`.
- Putting a catch-all pattern such as `<slug:slug>/` above more specific patterns.
- Forgetting `app_name`, then wondering why `reverse("detail")` finds the wrong route or fails.
- Reversing with `args` and `kwargs` together.
- Using `re_path` for patterns that `path` with a converter handles more readably.
- Forgetting that `re_path` captures arrive as strings.
- Omitting the trailing slash in a route and fighting redirects.
- Calling `reverse()` at import time instead of `reverse_lazy()`.
- Trying to match the query string in a pattern. Query parameters are read from `request.GET`.

## Practice exercises

1. Create a `library` app with routes for a book list, a book detail by slug, and an author detail by integer ID. Give each a name under the `library` namespace and verify each with `reverse`.
2. Write a test that fails if `/books/new/` is shadowed by a slug pattern, and fix the order.
3. Write a custom converter `IsbnConverter` that matches exactly thirteen digits, returns a `str`, and refuses to reverse anything that is not thirteen digits.
4. Use `resolve()` in the Django shell for five different paths and record `view_name`, `args`, and `kwargs` for each.
5. Add a `re_path` for `/archive/2026-03/` capturing year and month, then rewrite it with a `path()` and two converters. Compare readability.
6. Cause a `NoReverseMatch` three different ways (wrong name, missing namespace, wrong kwarg name) and write down the message Django gives for each.
7. Mount the same urlconf twice with different instance namespaces and write a test showing that `reverse` with `current_app` picks the right one.

## Summary

A urlconf is an ordered list of patterns, and the first match wins, so specific routes go first. `path()` with converters handles most cases, `re_path()` handles the rest, and `include()` lets each app own its routes. Name every route, give each app a namespace, and always build URLs with `reverse()` or `{% url %}`. Converters can validate and transform captured values, and a `ValueError` in `to_python` means "no match" while in `to_url` it means "cannot reverse". `APPEND_SLASH` redirects missing trailing slashes for GET requests only.
