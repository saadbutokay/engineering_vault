## Overview

Every web application accepts input from people: search terms, sign-up details, reviews, payment information. That input is untrusted. It can be missing, malformed, malicious, or simply a string where you expected a number. A Django `Form` is a class that describes the input you expect, converts raw strings into Python values, validates them, remembers what went wrong, and renders HTML for the fields. This note covers the `Form` class, field types and widgets, bound and unbound forms, `is_valid`, `cleaned_data` and `errors`, the `clean_<field>` and `clean` hooks, rendering forms and field groups (Django 5.0), customizing widgets and templates, CSRF protection, and `initial` and `prefix`. It ends with a typed, tested search form that feeds the filter object from note 2.5.

## Core concepts

### What a form is

A form is a Python class whose attributes are fields. It has two jobs: processing submitted data (validation and conversion) and rendering HTML. It is not tied to the database. Forms tied to models are covered in note 3.2.

### Fields

A field converts and validates one value. Common fields:

- Text: `CharField`, `EmailField`, `URLField`, `SlugField`, `RegexField`.
- Numbers: `IntegerField`, `DecimalField`, `FloatField`.
- Dates: `DateField`, `DateTimeField`, `TimeField`.
- Choices: `ChoiceField`, `TypedChoiceField`, `MultipleChoiceField`, `ModelChoiceField`.
- Booleans: `BooleanField`, `NullBooleanField` behavior through `NullBooleanSelect`.
- Files: `FileField`, `ImageField` (note 3.3).

Useful options on every field: `required`, `label`, `help_text`, `initial`, `validators`, `error_messages`, `disabled`, and `widget`.

### Widgets

A widget is the HTML control that renders a field: `TextInput`, `Textarea`, `Select`, `CheckboxInput`, `CheckboxSelectMultiple`, `DateInput`, `PasswordInput`, `HiddenInput`. Widgets decide how a field looks and how its value is read from the submitted data. They do not validate anything.

### Bound and unbound forms

- An unbound form has no data: `ContactForm()`. It renders empty fields (or initial values) and cannot be validated.
- A bound form was given data: `ContactForm(request.POST)`. It can be validated and it renders the submitted values with any errors.

### Validation and results

- `form.is_valid()` runs validation and returns a boolean.
- `form.cleaned_data` is a dictionary of converted values, available only after `is_valid()` has been called, and containing only the fields that passed.
- `form.errors` maps field names to error lists, with form-wide errors under `__all__`.

### clean_<field> and clean

- `clean_<fieldname>()` validates and may transform one field after the field's own validation.
- `clean()` validates the form as a whole and is the place for rules that involve several fields.

### Rendering

`{{ form }}` renders the whole form using a template (a `div`-based layout by default since Django 5.0). `{{ form.as_div }}`, `{{ form.as_p }}`, `{{ form.as_table }}`, and `{{ form.as_ul }}` pick other layouts. You can render each field yourself for full control, and Django 5.0 added `{{ field.as_field_group }}`, which renders the label, widget, help text, and errors of one field in one step.

### CSRF protection

Cross-site request forgery lets another site make a logged-in user's browser submit your form. Django defends with a secret token that must be submitted with unsafe requests. Include `{% csrf_token %}` inside every `<form method="post">`. GET forms (such as search) must not include it and must not change data.

### initial and prefix

- `initial` sets the starting values of an unbound form (`ContactForm(initial={"name": "Ada"})`).
- `prefix` namespaces field names so several forms can share one page (`ContactForm(prefix="support")` renders `support-name`).

## Basic syntax examples

### A first form

```python
from django import forms


class ContactForm(forms.Form):
    TOPICS = [
        ("question", "Question"),
        ("bug", "Report a bug"),
        ("other", "Something else"),
    ]

    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    topic = forms.ChoiceField(choices=TOPICS)
    message = forms.CharField(widget=forms.Textarea(attrs={"rows": 5}), min_length=10)
    subscribe = forms.BooleanField(required=False, label="Send me the newsletter")
```

### Bound versus unbound

```python
unbound = ContactForm()
print(unbound.is_bound)        # False

bound = ContactForm(
    {"name": "Ada", "email": "ada@example.com", "topic": "bug", "message": "It broke."}
)
print(bound.is_bound)          # True
print(bound.is_valid())        # False: the message is shorter than 10 characters
print(bound.errors)            # {'message': ['Ensure this value has at least 10 characters (it has 9).']}
```

### Using cleaned_data

```python
data = {
    "name": " Ada ",
    "email": "ada@example.com",
    "topic": "question",
    "message": "How do I reset my password?",
    "subscribe": "on",
}
form = ContactForm(data)
if form.is_valid():
    print(form.cleaned_data["name"])        # 'Ada' (surrounding whitespace stripped)
    print(form.cleaned_data["subscribe"])   # True
```

### Widgets and attributes

```python
class SearchForm(forms.Form):
    q = forms.CharField(
        required=False,
        widget=forms.TextInput(attrs={"placeholder": "Title or author", "autofocus": True}),
    )
    published_after = forms.DateField(
        required=False,
        widget=forms.DateInput(attrs={"type": "date"}),
    )
    formats = forms.MultipleChoiceField(
        required=False,
        choices=[("hardcover", "Hardcover"), ("paperback", "Paperback")],
        widget=forms.CheckboxSelectMultiple,
    )
```

### clean_<field> and clean

```python
from typing import Any

from django.core.exceptions import ValidationError


class ContactForm(forms.Form):
    # ... fields as above ...

    def clean_name(self) -> str:
        name: str = self.cleaned_data["name"]
        if name.lower() in {"admin", "root"}:
            raise ValidationError("Please use your real name.", code="reserved_name")
        return name

    def clean(self) -> dict[str, Any]:
        cleaned = super().clean()
        if cleaned.get("topic") == "bug" and len(cleaned.get("message", "")) < 30:
            self.add_error("message", "Please describe the bug in at least 30 characters.")
        return cleaned
```

### A view that handles the form

```python
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect, render


def contact(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        form = ContactForm(request.POST)
        if form.is_valid():
            # do something with form.cleaned_data
            return redirect("pages:contact_thanks")
    else:
        form = ContactForm()
    return render(request, "pages/contact.html", {"form": form})
```

### Rendering in a template

```html
<form method="post" novalidate>
  {% csrf_token %}
  {{ form }}
  <button type="submit">Send</button>
</form>
```

Manual rendering with full control:

```html
<form method="post" novalidate>
  {% csrf_token %}
  {{ form.non_field_errors }}
  {% for field in form %}
    <div class="field {% if field.errors %}has-error{% endif %}">
      {{ field.label_tag }}
      {{ field }}
      {% if field.help_text %}<small>{{ field.help_text }}</small>{% endif %}
      {{ field.errors }}
    </div>
  {% endfor %}
  <button type="submit">Send</button>
</form>
```

Django 5.0 and later can render one field with its label, help text, and errors in a single call:

```html
{% for field in form %}
  {{ field.as_field_group }}
{% endfor %}
```

### initial and prefix

```python
form = ContactForm(initial={"topic": "bug"})
support_form = ContactForm(prefix="support")
sales_form = ContactForm(prefix="sales")   # both can live in one HTML page
```

### Custom error messages and CSS classes

```python
class ContactForm(forms.Form):
    required_css_class = "required"
    error_css_class = "error"

    email = forms.EmailField(
        error_messages={
            "required": "We need an email address to reply.",
            "invalid": "That does not look like an email address.",
        }
    )
```

## Deeper behavior

### What happens inside is_valid

Validation runs in a fixed order:

1. For each field, in declaration order, the widget extracts the raw value from the submitted data (`value_from_datadict`).
2. The field's `clean()` converts it (`to_python`), checks it (`validate`, which enforces `required`), and runs its validators (`run_validators`).
3. If the field passed, the form calls `clean_<fieldname>()` if it exists.
4. After all fields, the form calls `clean()` for whole-form rules.

An error at any step records the error for that field and removes the field from `cleaned_data`. Because of this, `clean()` must tolerate missing keys: use `cleaned.get("topic")`, not `cleaned["topic"]`.

### Empty values

Optional fields do not produce `None` for every type. A `CharField` yields an empty string, while numeric, date, and choice-like fields yield `None` or `""` depending on the type. A `BooleanField(required=False)` yields `False` when the checkbox is absent. Check each field type's empty value rather than assuming.

### cleaned_data only after validation

Reading `form.cleaned_data` before `is_valid()` raises an `AttributeError`. After a failed validation it holds only the valid fields. Never use data from `form.data` (the raw input) in your logic. Always use `cleaned_data`.

### Adding errors in clean

`self.add_error("field", message)` attaches an error to a field and removes that field from `cleaned_data`. `self.add_error(None, message)` or raising `ValidationError` from `clean()` attaches a non-field error under `__all__`. Raising a `ValidationError` with a dictionary (`ValidationError({"field": "message"})`) also attaches errors to fields.

### Widgets and fields are separate concerns

Changing a widget changes the HTML and how data is read, not the validation. Setting `type="date"` on a `DateInput` gives a browser date picker, but the server still validates and parses the submitted string against the field's `input_formats`. The browser's own checks (the `required` and `maxlength` attributes Django adds) are conveniences, not security: a malicious client can send anything, so server-side validation is always required.

### The required attribute and novalidate

Django adds the HTML `required` attribute to required fields. Browsers then block submission and show their own messages, bypassing your custom error messages. Add `novalidate` to the `<form>` tag if you want server-rendered messages to be the only ones, or set `use_required_attribute = False` on the form.

### Form rendering and templates

Forms render through the template engine's form renderer. Django 5.0 changed the default layout from table to `div`. You can set `template_name` on a form class (or `template_name_div` and friends) to use your own template, and field-level templates can be overridden through `FORM_RENDERER = "django.forms.renderers.TemplatesSetting"` with `"django.forms"` in `INSTALLED_APPS`, which lets your project templates override `django/forms/field.html` and the widget templates. A common practice is to customize these templates once to match your CSS framework.

### Accessibility

Render a `<label>` for every control (`{{ field.label_tag }}`), show errors near the field, and keep `help_text` associated with the field. From Django 5.0, the default field group template links help text to the input with `aria-describedby`. Group related checkboxes and radios in a `<fieldset>` with a `<legend>` (Django 5.0 widgets support `use_fieldset`).

### CSRF in detail

For POST, PUT, PATCH, and DELETE requests, `CsrfViewMiddleware` compares the token submitted with the request (the hidden `csrfmiddlewaretoken` input, or the `X-CSRFToken` header for JavaScript requests) with the secret stored in a cookie or session. A missing or wrong token yields a 403. Do not exempt views with `@csrf_exempt` to "fix" this error; add the token. Never put `{% csrf_token %}` in a form that posts to another site, since it would leak the token.

### GET forms

Search and filter forms should use `method="get"`. They change no data, so CSRF does not apply, and the resulting URL can be bookmarked and shared. Bind them to `request.GET`, treating an empty query string as an unbound form so that the first page load shows no errors.

### Type hints and cleaned_data

`cleaned_data` is typed as `dict[str, Any]`. Annotate your `clean_<field>` methods with their return type and keep conversions near the form. When data flows onward, convert it to a typed object (a dataclass) at the boundary, as the example below does.

## Industry-level snippet

A book search form: it validates optional text, price range, formats, and availability, rejects inverted ranges with a field-level error, and converts valid input into the `BookFilters` object from note 2.5. The view treats an empty query string as unbound, and the tests cover validation, conversion, and rendering.

`apps/bookstore/forms.py`:

```python
"""Forms for the bookstore app."""

from decimal import Decimal
from typing import Any

from django import forms

from apps.bookstore.models import Book
from apps.bookstore.selectors import BookFilters

AVAILABILITY_CHOICES = [
    ("", "Any availability"),
    ("in_stock", "In stock"),
    ("sold_out", "Sold out"),
]


class BookSearchForm(forms.Form):
    q = forms.CharField(
        label="Search",
        required=False,
        max_length=100,
        strip=True,
        widget=forms.TextInput(attrs={"placeholder": "Title or author"}),
    )
    min_price = forms.DecimalField(
        required=False,
        min_value=Decimal("0"),
        max_digits=8,
        decimal_places=2,
    )
    max_price = forms.DecimalField(
        required=False,
        min_value=Decimal("0"),
        max_digits=8,
        decimal_places=2,
    )
    formats = forms.MultipleChoiceField(
        required=False,
        choices=Book.Format.choices,
        widget=forms.CheckboxSelectMultiple,
    )
    availability = forms.ChoiceField(required=False, choices=AVAILABILITY_CHOICES)

    def clean(self) -> dict[str, Any]:
        cleaned = super().clean()
        minimum = cleaned.get("min_price")
        maximum = cleaned.get("max_price")
        if minimum is not None and maximum is not None and minimum > maximum:
            self.add_error(
                "max_price", "Maximum price must not be below the minimum price."
            )
        return cleaned

    def to_filters(self) -> BookFilters:
        """Convert valid input into the typed filter object used by selectors."""
        if not self.is_bound or not self.is_valid():
            raise ValueError("Cannot build filters from an invalid or unbound form")
        data = self.cleaned_data
        availability = data["availability"]
        in_stock: bool | None
        if availability == "in_stock":
            in_stock = True
        elif availability == "sold_out":
            in_stock = False
        else:
            in_stock = None
        return BookFilters(
            text=data["q"],
            min_price=data["min_price"],
            max_price=data["max_price"],
            formats=tuple(data["formats"]),
            in_stock=in_stock,
        )
```

`apps/bookstore/views.py`:

```python
from django.db.models import QuerySet
from django.http import HttpRequest, HttpResponse
from django.shortcuts import render

from apps.bookstore.forms import BookSearchForm
from apps.bookstore.models import Book
from apps.bookstore.selectors import BookFilters, search_books


def book_search(request: HttpRequest) -> HttpResponse:
    # An empty query string means "first visit": do not show errors.
    form = BookSearchForm(request.GET or None)
    books: QuerySet[Book]
    if form.is_bound and form.is_valid():
        books = search_books(form.to_filters())
    elif form.is_bound:
        books = Book.objects.none()
    else:
        books = search_books(BookFilters())
    return render(request, "bookstore/book_search.html", {"form": form, "books": books})
```

`apps/bookstore/urls.py`:

```python
from django.urls import path

from . import views

app_name = "bookstore"

urlpatterns = [
    path("search/", views.book_search, name="search"),
]
```

Mount it in `config/urls.py` with `path("books/", include("apps.bookstore.urls"))`.

`apps/bookstore/templates/bookstore/book_search.html`:

```html
<h1>Find a book</h1>
<form method="get" action="{% url 'bookstore:search' %}">
  {{ form }}
  <button type="submit">Search</button>
</form>
<ul>
  {% for book in books %}
    <li>{{ book.title }} by {{ book.author.name }} ({{ book.availability }})</li>
  {% empty %}
    <li>No books found.</li>
  {% endfor %}
</ul>
```

`apps/bookstore/tests/test_search_form.py`:

```python
from decimal import Decimal

from django.test import SimpleTestCase, TestCase
from django.urls import reverse

from apps.bookstore.forms import BookSearchForm
from apps.bookstore.selectors import BookFilters

from .factories import make_author, make_book


class BookSearchFormTests(SimpleTestCase):
    def test_empty_form_is_valid_and_matches_everything(self) -> None:
        form = BookSearchForm({})
        self.assertTrue(form.is_valid())
        self.assertEqual(form.to_filters(), BookFilters())

    def test_input_is_stripped_and_converted(self) -> None:
        form = BookSearchForm(
            {
                "q": "  dune ",
                "min_price": "5.50",
                "max_price": "20",
                "formats": ["hardcover", "ebook"],
                "availability": "in_stock",
            }
        )
        self.assertTrue(form.is_valid(), form.errors)
        self.assertEqual(
            form.to_filters(),
            BookFilters(
                text="dune",
                min_price=Decimal("5.50"),
                max_price=Decimal("20"),
                formats=("hardcover", "ebook"),
                in_stock=True,
            ),
        )

    def test_sold_out_maps_to_false(self) -> None:
        form = BookSearchForm({"availability": "sold_out"})
        self.assertTrue(form.is_valid())
        self.assertIs(form.to_filters().in_stock, False)

    def test_inverted_price_range_is_reported_on_max_price(self) -> None:
        form = BookSearchForm({"min_price": "20", "max_price": "5"})
        self.assertFalse(form.is_valid())
        self.assertIn("max_price", form.errors)

    def test_invalid_values_are_rejected_field_by_field(self) -> None:
        cases = {
            "min_price": "-1",
            "max_price": "abc",
            "formats": "scroll",
            "availability": "maybe",
            "q": "x" * 101,
        }
        for field, value in cases.items():
            with self.subTest(field=field):
                form = BookSearchForm({field: value})
                self.assertFalse(form.is_valid())
                self.assertIn(field, form.errors)

    def test_to_filters_refuses_invalid_or_unbound_forms(self) -> None:
        with self.assertRaises(ValueError):
            BookSearchForm().to_filters()
        with self.assertRaises(ValueError):
            BookSearchForm({"min_price": "x"}).to_filters()


class BookSearchViewTests(TestCase):
    def test_first_visit_lists_books_without_errors(self) -> None:
        make_book(title="Dune")
        response = self.client.get(reverse("bookstore:search"))
        self.assertContains(response, "Dune")
        self.assertNotContains(response, "errorlist")

    def test_search_filters_results(self) -> None:
        make_book(title="Dune", author=make_author("Frank Herbert"))
        make_book(title="Earthsea")
        response = self.client.get(reverse("bookstore:search"), {"q": "herbert"})
        self.assertContains(response, "Dune")
        self.assertNotContains(response, "Earthsea")

    def test_invalid_input_shows_the_error_and_no_results(self) -> None:
        make_book(title="Dune")
        response = self.client.get(
            reverse("bookstore:search"), {"min_price": "20", "max_price": "5"}
        )
        self.assertContains(response, "Maximum price must not be below the minimum price.")
        self.assertNotContains(response, "Dune")
```

The form never reaches for `request.GET` values directly. Everything flows through `cleaned_data` and ends as a typed `BookFilters`, so the selector from note 2.5 never sees unvalidated input.

## Common mistakes

- Using `request.POST["field"]` or `form.data["field"]` instead of `form.cleaned_data`.
- Reading `cleaned_data` before calling `is_valid()`.
- Writing `clean()` with `cleaned_data["field"]` instead of `.get()`, which raises `KeyError` when that field failed validation.
- Forgetting `{% csrf_token %}` in a POST form, or "fixing" the resulting 403 with `@csrf_exempt`.
- Putting `{% csrf_token %}` in GET forms, or using GET for forms that change data.
- Relying on HTML attributes (`required`, `maxlength`, `type="number"`) as validation.
- Treating an empty GET query string as a bound form, so users see errors on their first visit.
- Returning data from `clean_<field>` incorrectly: forgetting to return the value sets the field to `None`.
- Re-rendering an invalid form with a redirect, which discards the errors.
- Putting expensive queries or external calls inside field validation without thinking about repeated submissions.
- Giving two forms on one page the same field names without a `prefix`.

## Practice exercises

1. Write a `NewsletterForm` with an email field, a frequency choice (`daily`, `weekly`, `monthly`), and a checkbox that must be ticked to accept the terms. Test valid input, a missing checkbox, and an invalid email.
2. Add a `clean_email` method that rejects addresses from a short list of disposable domains and normalizes the email to lowercase. Test both behaviors.
3. Build a `DateRangeForm` with `start` and `end` dates and a `clean()` that requires `end` to be on or after `start`, reporting the error on `end`. Test the equal, earlier, and missing-date cases.
4. Render the contact form three ways (`as_div`, a manual loop, and `as_field_group`) and compare the HTML each produces for a field with an error and a help text.
5. Put two forms with the same fields on one page using prefixes, and write a test that posts to one of them and checks the other stays unbound.
6. Remove `novalidate` from a form with required fields and observe what the browser does. Explain why server-side validation is still needed.
7. Extend `BookSearchForm` with a `published_after` date field wired into `BookFilters` (adding the field to the filters from note 2.5 if needed), and add tests for a valid date, an invalid date, and an empty value.

## Summary

A Django form declares the input you expect as fields, converts and validates submitted strings into Python values, and renders HTML through widgets and templates. A form is unbound until given data; `is_valid()` runs each field's cleaning, then `clean_<field>`, then `clean()`, and fills `cleaned_data` and `errors`. Always use `cleaned_data`, never raw input, and write `clean()` to tolerate fields that failed. Widgets and HTML attributes only shape the browser's behavior, so validation on the server is mandatory. Include `{% csrf_token %}` in every POST form, use GET for read-only search forms, treat an empty query string as an unbound form, and convert validated data into typed objects at the boundary.
