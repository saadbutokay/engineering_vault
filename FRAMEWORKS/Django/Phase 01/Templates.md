## Overview

Templates turn data into HTML. Django's template language (DTL) is deliberately limited: it can output values, loop, branch, include other templates, and apply filters, but it cannot run arbitrary Python. That restriction is a feature, because it keeps logic in views and services where it can be tested. This note covers variables, tags, filters, inheritance, includes, context and context processors, automatic escaping, the common built-in tags, writing your own tags and filters, template partials, and the Jinja2 option.

## Core concepts

### The template language

- Variables: `{{ article.title }}` outputs a value from the context.
- Tags: `{% if %}`, `{% for %}`, `{% url %}` perform logic and loading.
- Filters: `{{ value|lower }}` transform a value, and filters can take an argument (`{{ text|truncatewords:20 }}`) and be chained.
- Comments: `{# one line #}` and `{% comment %}...{% endcomment %}` for blocks.

### Template inheritance

A base template defines the page skeleton and named blocks. Child templates declare `{% extends "base.html" %}` and fill in the blocks. Inheritance keeps repeated markup (head, navigation, footer) in one place.

### Includes

`{% include "partials/card.html" with item=article %}` renders another template inside the current one. Use it for reusable fragments.

### Context and context processors

The context is the dictionary of variables a template can see. A view passes values explicitly (`render(request, name, context)`). Context processors are functions that add variables to every template rendered with a request, such as `request`, `user`, and `messages`.

### Automatic escaping

Django escapes `<`, `>`, `&`, `'`, and `"` in every variable output by default, which prevents most cross-site scripting (XSS). A value is only left alone if it is explicitly marked safe.

### Built-in tags you will use constantly

- `{% url 'app:name' arg=value %}`: reverse a URL.
- `{% load static %}` and `{% static 'path' %}`: reference static files.
- `{% csrf_token %}`: insert the CSRF token inside every POST form.
- `{% querystring %}`: build a query string while changing some parameters (Django 5.1).
- `{% with %}`, `{% firstof %}`, `{% cycle %}`, `{% now %}`, `{% spaceless %}`.

### Commonly used filters

- Text: `lower`, `upper`, `title`, `truncatechars`, `truncatewords`, `linebreaks`, `slugify`.
- Defaults and logic: `default`, `default_if_none`, `yesno`, `length`.
- Dates and numbers: `date`, `time`, `timesince`, `floatformat`, `intcomma` (from `humanize`).
- Safety: `safe`, `escape`, `force_escape`.

### Custom tags and filters

Create a `templatetags/` package inside an app (with `__init__.py`), define a `register = template.Library()`, and decorate functions. Load them in a template with `{% load module_name %}`. The app must be in `INSTALLED_APPS`.

### Template partials (Django 6.0)

Django 6.0 adds named, reusable fragments inside a single template file. On Django 5.2 LTS, use the `django-template-partials` package, which provides the same syntax, or split the fragment into its own file and use `{% include %}`.

### Jinja2

Django can use Jinja2 as an alternative template backend. It permits more Python-like expressions and can be faster, but it is a different language, and Django's built-in tags and filters, admin, and many third-party packages assume DTL. Keep DTL unless you have a clear reason.

### Why logic stays out of templates

Template logic cannot be unit-tested as easily, hides bugs, and can issue unexpected database queries. Compute values in the view or a model method and let the template display them.

## Basic syntax examples

### Variables, filters, and conditions

```html
<h1>{{ article.title|title }}</h1>
<p>Posted {{ article.created_at|date:"j F Y" }} by {{ article.author_name|default:"anonymous" }}</p>

{% if article.is_featured %}
  <span class="badge">Featured</span>
{% elif article.is_new %}
  <span class="badge">New</span>
{% else %}
  <span class="badge">Regular</span>
{% endif %}
```

### Loops

```html
<ul>
  {% for article in articles %}
    <li class="{% cycle 'odd' 'even' %}">
      {{ forloop.counter }}. {{ article.title }}
    </li>
  {% empty %}
    <li>No articles.</li>
  {% endfor %}
</ul>
```

Useful loop variables: `forloop.counter` (from 1), `forloop.counter0`, `forloop.first`, `forloop.last`, `forloop.parentloop`.

### Inheritance

`templates/base.html`:

```html
{% load static %}
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>{% block title %}My Site{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'css/site.css' %}">
  </head>
  <body>
    <main>
      {% block content %}{% endblock %}
    </main>
    {% block scripts %}{% endblock %}
  </body>
</html>
```

`apps/articles/templates/articles/article_list.html`:

```html
{% extends "base.html" %}

{% block title %}Articles - {{ block.super }}{% endblock %}

{% block content %}
  <h1>Articles</h1>
  {% include "articles/_article_card.html" with article=featured only %}
{% endblock %}
```

`{{ block.super }}` outputs the parent's block content. `extends` must be the first tag in the file. `only` in `include` prevents the fragment from seeing the rest of the context.

### A form with CSRF protection and a URL

```html
<form method="post" action="{% url 'tasks:collection' %}">
  {% csrf_token %}
  <input type="text" name="title" required>
  <button type="submit">Add</button>
</form>
```

### Keeping the query string while paging (Django 5.1)

```html
<a href="{% querystring page=page_obj.next_page_number %}">Next</a>
```

### A context processor

```python
from typing import Any

from django.conf import settings
from django.http import HttpRequest


def site_info(request: HttpRequest) -> dict[str, Any]:
    return {"site_name": getattr(settings, "SITE_NAME", "My Site")}
```

Register it under `TEMPLATES[0]["OPTIONS"]["context_processors"]` as `"apps.pages.context_processors.site_info"`. Then `{{ site_name }}` works in every template rendered with a request.

### Template partials in Django 6.0

```html
{% partialdef article_card %}
  <article class="card">
    <h2>{{ article.title }}</h2>
  </article>
{% endpartialdef %}

{% for article in articles %}
  {% partial article_card %}
{% endfor %}
```

Add the keyword `inline` after the name (`{% partialdef article_card inline %}`) to render the fragment where it is defined as well as making it reusable. To render only one partial from a view, name it after a hash: `render(request, "articles/list.html#article_card", context)`. This is useful for HTMX responses (Phase 3).

### Jinja2 backend configuration

```python
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.jinja2.Jinja2",
        "DIRS": [BASE_DIR / "jinja2"],
        "APP_DIRS": True,
        "OPTIONS": {"environment": "config.jinja2.environment"},
    },
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {"context_processors": []},
    },
]
```

Jinja2 requires `uv add jinja2`.

## Deeper behavior

### How a variable is resolved

For `{{ foo.bar }}`, Django tries, in order: dictionary lookup (`foo["bar"]`), attribute lookup (`foo.bar`), list-index lookup (`foo[0]` when `bar` is an integer). If the result is callable, Django calls it with no arguments. If a variable does not exist, the output is an empty string by default (the setting `string_if_invalid` changes this). Methods that change data should be marked `alters_data = True` so templates cannot call them.

### Missing variables fail silently

A typo such as `{{ artcle.title }}` produces nothing and no error. To catch mistakes, test your templates, and in development consider setting `string_if_invalid` to a visible marker.

### Escaping in different contexts

Autoescape protects HTML text and quoted attribute values. It does not make unquoted attributes safe, and it is not enough inside `<script>` blocks or URLs. Always quote attributes, use `{{ value|json_script:"id" }}` to pass data to JavaScript, and never build `href` values from untrusted strings without validating the scheme.

### Marking content safe

- `{{ html|safe }}` and `{% autoescape off %}` disable escaping for that output. Use them only for HTML you generated and trust.
- In Python, `mark_safe(string)` does the same and is dangerous if the string contains user input.
- `format_html("<b>{}</b>", user_value)` escapes the arguments and is the safe way to build small HTML fragments in code.

### Template loading and caching

With `APP_DIRS = True`, Django looks in the directories listed in `DIRS` first, then in each installed app's `templates/` folder, in `INSTALLED_APPS` order. Since Django 4.1 the cached template loader is enabled automatically when `DEBUG` is false, so templates are compiled once per process. This is why a template change in production needs a restart or deployment.

### Includes versus inheritance

Inheritance defines the shape of whole pages. Includes insert small fragments. An include with many variables is a sign that the fragment may be better as an inclusion tag or, in Django 6.0, a partial.

### Performance traps

Calling a method or following a relationship inside a loop can run a database query per iteration. Fetch what the template needs in the view with `select_related` and `prefetch_related` (Phase 2), and check query counts in tests.

## Industry-level snippet

Reusable display helpers belong in a template tag library: a `money` filter that formats decimals safely and an `active_class` tag for navigation. Both are typed, defensive, and tested. They live in `apps/pages/templatetags/ui_extras.py` (with an empty `__init__.py` in the `templatetags` folder, and `apps.pages` in `INSTALLED_APPS`).

`apps/pages/templatetags/ui_extras.py`:

```python
"""Display helpers shared by templates."""

import logging
from decimal import ROUND_HALF_EVEN, Decimal, InvalidOperation

from django import template
from django.template import Context

logger = logging.getLogger(__name__)
register = template.Library()

_CENT = Decimal("0.01")


@register.filter
def money(value: object, currency: str = "USD") -> str:
    """Format ``value`` as ``USD 1,234.50``.

    Filters should not raise, so invalid input logs a warning and renders
    an empty string.
    """
    code = currency.strip().upper()
    if len(code) != 3 or not code.isalpha():
        logger.warning("Invalid currency code %r", currency)
        return ""
    try:
        amount = Decimal(str(value))
    except InvalidOperation:
        logger.warning("Cannot format %r as money", value)
        return ""
    if not amount.is_finite():
        return ""
    rounded = amount.quantize(_CENT, rounding=ROUND_HALF_EVEN)
    return f"{code} {rounded:,.2f}"


@register.simple_tag(takes_context=True)
def active_class(context: Context, view_name: str, css_class: str = "active") -> str:
    """Return ``css_class`` when the current request resolved to ``view_name``."""
    request = context.get("request")
    match = getattr(request, "resolver_match", None)
    if match is not None and match.view_name == view_name:
        return css_class
    return ""
```

Using them:

```html
{% load ui_extras %}
<nav>
  <a class="{% active_class 'articles:list' %}" href="{% url 'articles:list' %}">Articles</a>
</nav>
<p>Price: {{ product.price|money:"EUR" }}</p>
```

`apps/pages/tests/test_ui_extras.py` (with an empty `__init__.py` in the `tests` folder):

```python
from decimal import Decimal

from django.http import HttpRequest
from django.template import Context, Template
from django.test import RequestFactory, SimpleTestCase
from django.urls import resolve


def render(source: str, **context: object) -> str:
    return Template("{% load ui_extras %}" + source).render(Context(context))


class MoneyFilterTests(SimpleTestCase):
    def test_formats_decimal_with_grouping(self) -> None:
        self.assertEqual(render("{{ p|money }}", p=Decimal("1234.5")), "USD 1,234.50")

    def test_accepts_other_currency(self) -> None:
        self.assertEqual(render("{{ p|money:'eur' }}", p=2), "EUR 2.00")

    def test_rounds_half_even(self) -> None:
        self.assertEqual(render("{{ p|money }}", p="0.125"), "USD 0.12")

    def test_invalid_amount_renders_empty(self) -> None:
        for value in ["abc", None, "NaN", "Infinity"]:
            with self.subTest(value=value):
                self.assertEqual(render("{{ p|money }}", p=value), "")

    def test_invalid_currency_renders_empty(self) -> None:
        self.assertEqual(render("{{ p|money:'EURO' }}", p=1), "")


class ActiveClassTests(SimpleTestCase):
    def _request(self, path: str) -> HttpRequest:
        request = RequestFactory().get(path)
        request.resolver_match = resolve(path)
        return request

    def test_returns_class_for_current_view(self) -> None:
        request = self._request("/articles/")
        result = render("{% active_class 'articles:list' %}", request=request)
        self.assertEqual(result, "active")

    def test_returns_empty_for_other_views(self) -> None:
        request = self._request("/articles/")
        result = render("{% active_class 'articles:detail' %}", request=request)
        self.assertEqual(result, "")

    def test_missing_request_is_safe(self) -> None:
        self.assertEqual(render("{% active_class 'articles:list' %}"), "")


class EscapingTests(SimpleTestCase):
    def test_variables_are_escaped_by_default(self) -> None:
        output = Template("{{ comment }}").render(Context({"comment": "<script>x</script>"}))
        self.assertEqual(output, "&lt;script&gt;x&lt;/script&gt;")
```

## Common mistakes

- Putting `{% extends %}` anywhere but the first line of the template.
- Disabling escaping with `|safe` on user-supplied text, which creates an XSS hole.
- Forgetting `{% csrf_token %}` in a POST form and getting a 403 error.
- Forgetting `{% load static %}` before using `{% static %}`.
- Putting business logic, queries, or heavy computation in templates.
- Calling relationships in loops and triggering one query per row.
- Using unquoted attributes such as `<a href={{ url }}>`, where escaping cannot protect you.
- Creating a `templatetags` folder without `__init__.py`, or forgetting to restart the server after adding a tag library.
- Generic template file names such as `list.html` that collide across apps.
- Assuming template partials exist on Django 5.2 without installing the compatibility package.

## Practice exercises

1. Build a `base.html` with title, content, and scripts blocks, and two child pages that extend it. Use `{{ block.super }}` in one of them.
2. Write an include `_pagination.html` that takes `page_obj` and renders previous and next links with `{% querystring %}`. Use it in two list templates.
3. Write a filter `initials` that turns `"Ada Lovelace"` into `"AL"`, handling empty strings and extra whitespace, with tests for each case.
4. Write a simple tag `now_iso` that returns the current UTC time in ISO format, and a test that controls the clock.
5. Pass a dictionary from a view to JavaScript safely using `json_script`, and demonstrate that a value containing `</script>` stays harmless.
6. Create a deliberate N+1 problem in a template by accessing `article.category.name` in a loop, count queries with `assertNumQueries`, and fix it with `select_related`.
7. Write a context processor that adds the number of unread messages for the current user (return zero for anonymous users) and explain the performance cost of running a query in a context processor.

## Summary

Django templates display data with variables, tags, and filters, and compose pages through inheritance and includes. Context processors supply values to every template, and automatic escaping protects against XSS unless you opt out with `safe` or `mark_safe`. Custom filters and tags live in `templatetags/` and should be typed, defensive, and tested. Partials arrive in Django 6.0, with the `django-template-partials` package or `{% include %}` as the 5.2 alternatives. Jinja2 is available but is a separate language. Keep logic out of templates, prefetch what you display, and always quote attributes.
