## Overview

Most forms exist to create or edit database rows, and writing every field twice, once on the model and once on the form, is wasteful and error-prone. A `ModelForm` builds its fields from a model, validates through the model's own rules, and saves straight to the database. Formsets handle the other recurring need: editing several related objects on one page, such as the items of an order. Both come with traps that matter in production: a careless field list can let users set columns they should never touch, validation silently skips constraints that involve fields missing from the form, and a select box can quietly cost one query per row. This note covers `ModelForm`, `Meta.fields` and why `"__all__"` is a security risk, `save(commit=False)`, the order of model and form validation, formsets and inline formsets, managing child objects, and `ModelChoiceField` costs. It ends with tested forms for books, reviews, and an order with editable items.

This note builds on the bookstore models from Phase 2 and the forms module from note 3.1.

## Core concepts

### ModelForm

A `ModelForm` is a form whose fields are generated from a model. Its inner `Meta` class names the model and the fields to include.

```python
class BookForm(forms.ModelForm[Book]):
    class Meta:
        model = Book
        fields = ["title", "isbn", "price"]
```

`form.save()` creates or updates the model instance. Passing `instance=book` to the constructor edits an existing row.

### Field generation

Each model field maps to a form field: `CharField` to `forms.CharField`, `ForeignKey` to `ModelChoiceField`, `ManyToManyField` to `ModelMultipleChoiceField`, `DecimalField` to `forms.DecimalField`, and fields with `choices` to `TypedChoiceField`. Options such as `max_length`, `blank`, `help_text`, and `verbose_name` carry over. Model validators do not transfer to the form field. They run later, during model validation.

### Meta.fields and the mass assignment risk

`fields` is an allowlist of model fields the form may read from submitted data. Any model field not listed cannot be set by the user, no matter what they send. Two alternatives are dangerous:

- `fields = "__all__"` includes every editable field, including columns you add later, such as `is_staff`, `owner`, `version`, or `price_override`. A new column silently becomes user-editable.
- `exclude = [...]` is a denylist, so every new column is included unless you remember to exclude it.

Always use an explicit list. This vulnerability class is called mass assignment, and the Ruff `DJ` rules flag both forms.

### save(commit=False)

`form.save(commit=False)` returns an unsaved instance so you can fill in fields the form does not handle, such as the current user or a parent object, and then save it yourself. If the model has many-to-many fields on the form, call `form.save_m2m()` after saving the instance.

### The order of validation

When you call `is_valid()` on a `ModelForm`:

1. Each form field cleans its input, then any `clean_<field>()` runs.
2. The form's `clean()` runs.
3. `_post_clean()` builds an instance from `cleaned_data` and calls the model's `full_clean()`, excluding fields not in the form. This runs model field validators, `clean()`, `validate_unique()`, and `validate_constraints()`.

Errors from the model layer are attached to the matching form fields, or to the form as non-field errors.

### Excluded fields and constraints

Model validation skips any unique check or constraint that involves a field missing from the form. If `Review` has a unique constraint on `(book, reviewer_name)` and the form omits `book`, the form will not report a duplicate review. The database will still refuse it with an `IntegrityError`, but the user sees a server error instead of a friendly message. The fix is an explicit check in the form, shown in the example below.

### Formsets

A formset is a collection of forms of the same type on one page. `formset_factory(MyForm, extra=2)` builds one from a plain form, and `modelformset_factory(Model, ...)` builds one that edits a queryset of model instances.

### The management form

Every formset needs hidden fields (`TOTAL_FORMS`, `INITIAL_FORMS`, `MIN_NUM_FORMS`, `MAX_NUM_FORMS`) that tell Django how many forms were submitted. Render them with `{{ formset.management_form }}`. Without them, validation fails with "ManagementForm data is missing or has been tampered with".

### Inline formsets

`inlineformset_factory(Order, OrderItem, ...)` edits the children of one parent: the items of one order. It manages the foreign key to the parent for you and limits the queryset to that parent's children.

### Managing child objects

- `extra` controls how many blank forms appear.
- `can_delete=True` adds a delete checkbox to each form.
- `min_num`, `max_num`, and `validate_min` or `validate_max` bound the count.
- `formset.save()` creates, updates, and deletes in one call. With `commit=False` it returns the new and changed objects and leaves deletions for you to apply.
- A formset's `clean()` is the place for rules across forms, such as "no duplicate books".

### ModelChoiceField and its query cost

A foreign key renders as a `<select>` filled by evaluating a queryset, and validating a submitted choice runs a query to find the object. In a formset with twenty rows, each form has its own field instance, so rendering runs twenty queries for the choices and validation runs another twenty lookups. Limit the queryset, select only the columns you display, and consider a different widget for large tables.

## Basic syntax examples

### A minimal ModelForm

```python
from django import forms

from apps.bookstore.models import Book


class BookForm(forms.ModelForm[Book]):
    class Meta:
        model = Book
        fields = ["title", "isbn", "author", "price", "stock", "format", "published_on"]
```

### Creating and editing

```python
form = BookForm(request.POST)               # create
form = BookForm(request.POST, instance=book)  # edit an existing row

if form.is_valid():
    book = form.save()
```

### Customizing labels, help text, and widgets in Meta

```python
class BookForm(forms.ModelForm[Book]):
    class Meta:
        model = Book
        fields = ["title", "price", "published_on"]
        labels = {"price": "Price (USD)"}
        help_texts = {"title": "As printed on the cover."}
        widgets = {"published_on": forms.DateInput(attrs={"type": "date"}, format="%Y-%m-%d")}
        error_messages = {"title": {"max_length": "That title is too long."}}
```

### Filling in the fields the form does not handle

```python
review = form.save(commit=False)
review.book = book            # set by the view, never by the user
review.save()
form.save_m2m()               # only needed when the form has many-to-many fields
```

### Restricting a ModelChoiceField

```python
from typing import Any, cast

from apps.bookstore.models import Author


class BookForm(forms.ModelForm[Book]):
    def __init__(self, *args: Any, **kwargs: Any) -> None:
        super().__init__(*args, **kwargs)
        author_field = cast("forms.ModelChoiceField[Author]", self.fields["author"])
        author_field.queryset = Author.objects.order_by("name")
```

### Changing how choices are labeled

```python
class AuthorChoiceField(forms.ModelChoiceField[Author]):
    def label_from_instance(self, obj: Author) -> str:
        return f"{obj.name} (born {obj.born_on.year})" if obj.born_on else obj.name
```

### A plain formset

```python
from django.forms import formset_factory

TagFormSet = formset_factory(TagForm, extra=2, max_num=10, validate_max=True)
formset = TagFormSet(request.POST or None)
```

### A model formset

```python
from django.forms import modelformset_factory

GenreFormSet = modelformset_factory(
    Genre, fields=["name"], extra=1, can_delete=True
)
formset = GenreFormSet(request.POST or None, queryset=Genre.objects.order_by("name"))
```

### An inline formset

```python
from django.forms import inlineformset_factory

ItemFormSet = inlineformset_factory(
    Order,
    OrderItem,
    fields=["book", "quantity"],
    extra=1,
    can_delete=True,
)
formset = ItemFormSet(request.POST or None, instance=order)
```

### Rendering a formset

```html
<form method="post">
  {% csrf_token %}
  {{ order_form }}
  {{ formset.management_form }}
  {{ formset.non_form_errors }}
  {% for form in formset %}
    <div class="item">{{ form }}</div>
  {% endfor %}
  <button type="submit">Save</button>
</form>
```

For rows added with JavaScript, render `{{ formset.empty_form }}` inside a template element. Its field names contain `__prefix__`, which your script replaces with the next index and then increases `TOTAL_FORMS`.

### A view for a formset

```python
def edit_items(request: HttpRequest, order_id: int) -> HttpResponse:
    order = get_object_or_404(Order, pk=order_id)
    formset = ItemFormSet(request.POST or None, instance=order)
    if request.method == "POST" and formset.is_valid():
        formset.save()
        return redirect("orders:detail", pk=order.pk)
    return render(request, "orders/edit_items.html", {"formset": formset, "order": order})
```

## Deeper behavior

### What ModelForm does with an invalid form

During `_post_clean()`, the form copies cleaned values onto `form.instance` before model validation runs. When validation then fails, `form.instance` may already hold the invalid values. Do not use `form.instance` after a failed validation, and never save it. This matters when you edit an object and then re-render the page: render the object from the database (or the form's data), not from a mutated instance.

### Why unique checks skip excluded fields

`BaseModelForm._get_validation_exclusions()` builds the list of fields not in the form (and fields that failed validation), and passes it to `full_clean(exclude=...)`. A `UniqueConstraint` that uses an excluded field cannot be checked reliably because the instance may not have a value for it. Django therefore skips it. This also applies to conditional unique constraints whose condition mentions an excluded field, as with the active-ISBN constraint from note 2.10, where `deleted_at` is not on the form. In both cases, add an explicit check in the form or service and keep the database constraint as the last line of defense.

### Declared fields override generated ones

If you declare a field with the same name as a model field (`isbn = forms.CharField(max_length=20)`), your declaration wins, and the generated one is discarded. This is the normal way to change a field's type, length, or normalization. Remember that model validators then run on the cleaned value during `_post_clean()`, not on the form field.

### Model validators, form fields, and normalization

The form field validates the raw submitted text first. If the model's `max_length` is 13 and you want to accept hyphenated ISBNs of 17 characters, the generated form field would reject them before `clean_isbn()` can strip the hyphens. Declare the form field with a larger `max_length`, strip in `clean_isbn()`, and let the model validators check the normalized value.

### commit=False details

`save(commit=False)` does not save the instance or many-to-many data. For forms with `ManyToManyField`s, call `form.save_m2m()` after `instance.save()`. For formsets, `formset.save(commit=False)` returns the new and changed instances, and records deletions in `formset.deleted_objects` without applying them, so you must delete those yourself.

### Formset validation order and non-form errors

Each form validates on its own, then the formset's `clean()` runs. Errors raised there appear in `formset.non_form_errors()`. Because `clean()` reads each form's `cleaned_data`, check `any(self.errors)` first and return early if any form already failed, otherwise `cleaned_data` may be missing keys. Forms that are empty and unchanged (the extra blank forms) have an empty `cleaned_data` and should be skipped.

### Inline formset prefixes

The prefix for an inline formset defaults to the related name of the foreign key (`items` for `Order.items`), so field names look like `items-0-book`, and the management form fields are `items-TOTAL_FORMS` and so on. Posted data, tests, and JavaScript must use the same prefix. Pass `prefix="..."` to change it.

### Editing existing children

Forms for existing rows include a hidden `id` field and, for inline formsets, a hidden foreign key field. Both are rendered by `{{ form }}` automatically. If you render fields by hand, include `{{ form.id }}`, or Django cannot match submitted rows to existing ones.

### Querying cost of choice fields

Evaluating `ModelChoiceField.queryset` happens per field instance, and each form in a formset owns its own copy. Use `only("pk", "title")` to avoid loading unneeded columns, keep the queryset small (for example only active books), and for very large tables use a text input with autocomplete and validate the chosen id in `clean()` instead of rendering every row as an option. An approach for formsets is to evaluate the choices once and share them, which exercise 7 asks you to implement.

### Snapshots in child rows

When child rows snapshot data from another table (the unit price copied from the book), set it only when the row is created. Overwriting it on every edit rewrites history. The service below applies the snapshot only to new items.

## Industry-level snippet

Forms for the bookstore: a `BookForm` with a strict field list, normalized ISBN, future-date check, and explicit duplicate checks that cover the constraints model validation skips; a `ReviewForm` that takes its book from the view; and an order form with an inline item formset that rejects duplicates and empty orders. A service saves the order and its items atomically and snapshots prices only for new items.

`apps/bookstore/forms.py` (additions to the module from note 3.1):

```python
from datetime import date
from typing import Any, cast

from django import forms
from django.core.exceptions import ValidationError
from django.forms import BaseInlineFormSet, inlineformset_factory
from django.utils import timezone

from apps.bookstore.models import Author, Book, Order, OrderItem, Review

MAX_ITEM_QUANTITY = 100


class BookForm(forms.ModelForm[Book]):
    # Declared here so hyphenated input is not rejected by the model's max_length.
    isbn = forms.CharField(
        max_length=20,
        label="ISBN",
        help_text="13 digits. Hyphens and spaces are allowed.",
    )

    class Meta:
        model = Book
        fields = ["title", "isbn", "author", "price", "stock", "format", "published_on"]
        widgets = {
            "published_on": forms.DateInput(attrs={"type": "date"}, format="%Y-%m-%d"),
        }

    def __init__(self, *args: Any, **kwargs: Any) -> None:
        super().__init__(*args, **kwargs)
        author_field = cast("forms.ModelChoiceField[Author]", self.fields["author"])
        author_field.queryset = Author.objects.order_by("name")

    def clean_isbn(self) -> str:
        isbn = str(self.cleaned_data["isbn"]).replace("-", "").replace(" ", "")
        # Model validation skips the conditional unique constraint because
        # ``deleted_at`` is not a form field, so check it here.
        duplicates = Book.objects.alive().filter(isbn=isbn).exclude(pk=self.instance.pk)
        if duplicates.exists():
            raise ValidationError(
                "An active book with this ISBN already exists.", code="duplicate_isbn"
            )
        return isbn

    def clean_published_on(self) -> date | None:
        value: date | None = self.cleaned_data["published_on"]
        if value is not None and value > timezone.localdate():
            raise ValidationError(
                "The publication date cannot be in the future.", code="future_date"
            )
        return value


class ReviewForm(forms.ModelForm[Review]):
    """Create a review for a book that the view supplies."""

    class Meta:
        model = Review
        fields = ["reviewer_name", "rating", "body"]

    def __init__(self, *args: Any, book: Book, **kwargs: Any) -> None:
        super().__init__(*args, **kwargs)
        self.book = book
        self.instance.book = book

    def clean_reviewer_name(self) -> str:
        name = str(self.cleaned_data["reviewer_name"]).strip()
        # ``book`` is not a form field, so the unique constraint is not
        # validated automatically.
        if Review.objects.filter(book=self.book, reviewer_name__iexact=name).exists():
            raise ValidationError(
                "You have already reviewed this book.", code="duplicate_review"
            )
        return name


class OrderForm(forms.ModelForm[Order]):
    class Meta:
        model = Order
        fields = ["customer_email"]

    def clean_customer_email(self) -> str:
        return str(self.cleaned_data["customer_email"]).strip().lower()


class OrderItemForm(forms.ModelForm[OrderItem]):
    class Meta:
        model = OrderItem
        fields = ["book", "quantity"]

    def __init__(self, *args: Any, **kwargs: Any) -> None:
        super().__init__(*args, **kwargs)
        book_field = cast("forms.ModelChoiceField[Book]", self.fields["book"])
        book_field.queryset = Book.objects.alive().only("id", "title", "price").order_by("title")

    def clean_quantity(self) -> int:
        quantity: int = self.cleaned_data["quantity"]
        if not 1 <= quantity <= MAX_ITEM_QUANTITY:
            raise ValidationError(
                f"Quantity must be between 1 and {MAX_ITEM_QUANTITY}.",
                code="invalid_quantity",
            )
        return quantity


class BaseOrderItemFormSet(BaseInlineFormSet[OrderItem, Order, OrderItemForm]):
    def clean(self) -> None:
        super().clean()
        if any(self.errors):
            return  # let the per-form errors be shown first
        seen: set[int] = set()
        for form in self.forms:
            cleaned = form.cleaned_data
            if not cleaned or cleaned.get("DELETE"):
                continue  # blank extra forms and rows marked for deletion
            book: Book = cleaned["book"]
            if book.pk in seen:
                raise ValidationError(
                    "Each book may appear only once.", code="duplicate_book"
                )
            seen.add(book.pk)
        if not seen:
            raise ValidationError("An order needs at least one item.", code="empty_order")


OrderItemFormSet = inlineformset_factory(
    Order,
    OrderItem,
    form=OrderItemForm,
    formset=BaseOrderItemFormSet,
    extra=1,
    can_delete=True,
)
```

If your version of `django-stubs-ext` does not make a class subscriptable at runtime, quote the type arguments in the base class list, for example `BaseInlineFormSet["OrderItem", "Order", "OrderItemForm"]`.

`apps/bookstore/services/order_editing.py`:

```python
"""Saving an order together with its items."""

import logging

from django.db import transaction

from apps.bookstore.forms import BaseOrderItemFormSet, OrderForm
from apps.bookstore.models import Order

logger = logging.getLogger(__name__)


@transaction.atomic
def save_order_with_items(*, order_form: OrderForm, formset: BaseOrderItemFormSet) -> Order:
    """Save a valid order form and item formset in one transaction.

    New items snapshot the book's current price. Existing items keep the
    price they were sold at, even if the book's price has changed since.
    """
    if not (order_form.is_valid() and formset.is_valid()):
        raise ValueError("Both forms must be valid before saving")

    order = order_form.save()
    formset.instance = order
    for item in formset.save(commit=False):
        if item.pk is None:
            item.unit_price = item.book.price
        item.order = order
        item.save()
    for deleted in formset.deleted_objects:
        deleted.delete()

    logger.info("Saved order %s", order.pk)
    return order
```

`apps/bookstore/tests/test_model_forms.py`:

```python
from datetime import timedelta
from decimal import Decimal

from django.test import TestCase
from django.utils import timezone

from apps.bookstore.forms import BookForm, OrderForm, OrderItemFormSet, ReviewForm
from apps.bookstore.models import Author, Book, Order
from apps.bookstore.services.order_editing import save_order_with_items

from .factories import make_author, make_book


def book_data(author: Author, **overrides: str) -> dict[str, str]:
    data = {
        "title": "Dune",
        "isbn": "978-0-306-40615-7",
        "author": str(author.pk),
        "price": "12.50",
        "stock": "3",
        "format": "paperback",
        "published_on": "1965-08-01",
    }
    data.update(overrides)
    return data


def formset_data(rows: list[dict[str, str]], *, initial: int = 0) -> dict[str, str]:
    data = {
        "items-TOTAL_FORMS": str(len(rows)),
        "items-INITIAL_FORMS": str(initial),
        "items-MIN_NUM_FORMS": "0",
        "items-MAX_NUM_FORMS": "1000",
    }
    for index, row in enumerate(rows):
        for key, value in row.items():
            data[f"items-{index}-{key}"] = value
    return data


class BookFormTests(TestCase):
    def setUp(self) -> None:
        self.author = make_author()

    def test_valid_data_saves_a_book_with_a_normalized_isbn(self) -> None:
        form = BookForm(book_data(self.author))
        self.assertTrue(form.is_valid(), form.errors)
        book = form.save()
        self.assertEqual(book.isbn, "9780306406157")

    def test_fields_outside_the_allowlist_cannot_be_set(self) -> None:
        data = book_data(self.author, deleted_at="2020-01-01 00:00:00", version="99")
        form = BookForm(data)
        self.assertTrue(form.is_valid(), form.errors)
        book = form.save()
        book.refresh_from_db()
        self.assertIsNone(book.deleted_at)
        self.assertEqual(book.version, 1)

    def test_future_publication_date_is_rejected(self) -> None:
        tomorrow = (timezone.localdate() + timedelta(days=1)).isoformat()
        form = BookForm(book_data(self.author, published_on=tomorrow))
        self.assertFalse(form.is_valid())
        self.assertEqual(form.errors.as_data()["published_on"][0].code, "future_date")

    def test_model_validators_check_the_normalized_isbn(self) -> None:
        form = BookForm(book_data(self.author, isbn="978-0-306-40615-8"))
        self.assertFalse(form.is_valid())
        self.assertIn("isbn", form.errors)

    def test_expression_constraint_is_reported_as_a_form_error(self) -> None:
        make_book(title="Dune", author=self.author)
        form = BookForm(book_data(self.author, title="DUNE"))
        self.assertFalse(form.is_valid())
        self.assertIn("already has a book with that title", form.errors.as_text())

    def test_duplicate_active_isbn_is_caught_by_the_explicit_check(self) -> None:
        existing = make_book(author=self.author)
        form = BookForm(book_data(self.author, isbn=existing.isbn, title="Other"))
        self.assertFalse(form.is_valid())
        self.assertEqual(form.errors.as_data()["isbn"][0].code, "duplicate_isbn")

    def test_soft_deleted_isbn_may_be_reused(self) -> None:
        existing = make_book(author=self.author)
        existing.deleted_at = timezone.now()
        existing.save(update_fields=["deleted_at"])
        form = BookForm(book_data(self.author, isbn=existing.isbn, title="Reprint"))
        self.assertTrue(form.is_valid(), form.errors)

    def test_editing_a_book_does_not_collide_with_itself(self) -> None:
        book = make_book(title="Dune", author=self.author)
        form = BookForm(
            book_data(self.author, isbn=book.isbn, title="Dune", price="15.00"),
            instance=book,
        )
        self.assertTrue(form.is_valid(), form.errors)
        self.assertEqual(form.save().price, Decimal("15.00"))


class ReviewFormTests(TestCase):
    def setUp(self) -> None:
        self.book = make_book()

    def test_book_is_not_a_form_field_and_is_attached_on_save(self) -> None:
        form = ReviewForm({"reviewer_name": " Ada ", "rating": "5", "body": "Great"}, book=self.book)
        self.assertNotIn("book", form.fields)
        self.assertTrue(form.is_valid(), form.errors)
        review = form.save()
        self.assertEqual((review.book, review.reviewer_name, review.rating), (self.book, "Ada", 5))

    def test_duplicate_review_gets_a_friendly_error(self) -> None:
        ReviewForm({"reviewer_name": "Ada", "rating": "5"}, book=self.book).save()
        form = ReviewForm({"reviewer_name": "ada", "rating": "4"}, book=self.book)
        self.assertFalse(form.is_valid())
        self.assertIn("already reviewed", form.errors.as_text())

    def test_rating_must_be_a_valid_choice(self) -> None:
        form = ReviewForm({"reviewer_name": "Ada", "rating": "9"}, book=self.book)
        self.assertFalse(form.is_valid())
        self.assertIn("rating", form.errors)


class OrderFormsetTests(TestCase):
    def setUp(self) -> None:
        self.first = make_book(title="First", price=Decimal("10.00"))
        self.second = make_book(title="Second", price=Decimal("20.00"))

    def create_order(self) -> Order:
        formset = OrderItemFormSet(
            formset_data(
                [
                    {"book": str(self.first.pk), "quantity": "2"},
                    {"book": str(self.second.pk), "quantity": "1"},
                ]
            ),
            instance=Order(),
        )
        return save_order_with_items(
            order_form=OrderForm({"customer_email": "A@Example.com"}),
            formset=formset,
        )

    def test_creating_an_order_snapshots_prices(self) -> None:
        order = self.create_order()
        self.assertEqual(order.customer_email, "a@example.com")
        prices = {item.book_id: item.unit_price for item in order.items.all()}
        self.assertEqual(
            prices, {self.first.pk: Decimal("10.00"), self.second.pk: Decimal("20.00")}
        )

    def test_duplicate_books_are_rejected_by_the_formset(self) -> None:
        formset = OrderItemFormSet(
            formset_data(
                [
                    {"book": str(self.first.pk), "quantity": "1"},
                    {"book": str(self.first.pk), "quantity": "2"},
                ]
            ),
            instance=Order(),
        )
        self.assertFalse(formset.is_valid())
        self.assertEqual(formset.non_form_errors().as_data()[0].code, "duplicate_book")

    def test_an_order_without_items_is_rejected(self) -> None:
        formset = OrderItemFormSet(formset_data([{}]), instance=Order())
        self.assertFalse(formset.is_valid())
        self.assertEqual(formset.non_form_errors().as_data()[0].code, "empty_order")

    def test_quantity_limits_are_reported_on_the_form(self) -> None:
        formset = OrderItemFormSet(
            formset_data([{"book": str(self.first.pk), "quantity": "101"}]),
            instance=Order(),
        )
        self.assertFalse(formset.is_valid())
        self.assertIn("quantity", formset.errors[0])

    def test_editing_keeps_the_original_price_and_can_delete_rows(self) -> None:
        order = self.create_order()
        first_item = order.items.get(book=self.first)
        second_item = order.items.get(book=self.second)
        self.first.price = Decimal("99.00")
        self.first.save(update_fields=["price"])

        formset = OrderItemFormSet(
            formset_data(
                [
                    {
                        "id": str(first_item.pk),
                        "order": str(order.pk),
                        "book": str(self.first.pk),
                        "quantity": "5",
                    },
                    {
                        "id": str(second_item.pk),
                        "order": str(order.pk),
                        "book": str(self.second.pk),
                        "quantity": "1",
                        "DELETE": "on",
                    },
                ],
                initial=2,
            ),
            instance=order,
        )
        save_order_with_items(
            order_form=OrderForm({"customer_email": "a@example.com"}, instance=order),
            formset=formset,
        )

        first_item.refresh_from_db()
        self.assertEqual((first_item.quantity, first_item.unit_price), (5, Decimal("10.00")))
        self.assertFalse(order.items.filter(pk=second_item.pk).exists())

    def test_saving_invalid_forms_is_refused(self) -> None:
        formset = OrderItemFormSet(formset_data([{}]), instance=Order())
        with self.assertRaises(ValueError):
            save_order_with_items(
                order_form=OrderForm({"customer_email": "a@example.com"}),
                formset=formset,
            )
        self.assertEqual(Order.objects.count(), 0)
```

Notice what the tests lock in: the field allowlist, the explicit checks that fill the gaps model validation leaves, the price snapshot that survives edits, and the atomic save.

## Common mistakes

- Using `fields = "__all__"` or `exclude`, so new model fields become user-editable without anyone deciding that.
- Assuming a unique constraint is validated by the form when one of its fields is not on the form.
- Using `form.instance` after a failed validation. It may contain invalid values.
- Forgetting `form.save_m2m()` after `save(commit=False)` on forms with many-to-many fields.
- Forgetting that `formset.save(commit=False)` does not apply deletions.
- Omitting `{{ formset.management_form }}`, which produces the "ManagementForm data is missing" error.
- Rendering formset forms by hand without the hidden `id` and parent foreign key fields.
- Writing a formset `clean()` that reads `cleaned_data` without first checking `any(self.errors)`, or that counts the blank extra forms.
- Letting a select box load every row of a large table, or one query per form in a formset.
- Overwriting a snapshotted value (a price) every time a child row is edited.
- Setting a model field's `max_length` and expecting the form to accept longer normalized input.

## Practice exercises

1. Write a `GenreForm` with an explicit field list for `name` and `parent`, excluding the genre itself from the parent choices when editing, and test that a genre cannot be its own parent.
2. Add `Publisher` to the bookstore (if you have not already) and write a `PublisherForm` that rejects names that differ only by case from an existing publisher, with a code on the error. Test the edit case too.
3. Demonstrate the mass assignment bug: create a form with `fields = "__all__"` on a scratch model that has an `is_featured` field, post that field, and show it is saved. Then switch to an explicit list and show it is ignored.
4. Write a `BookCopiesForm` using `formset_factory` (not a model formset) that accepts up to five barcodes, rejects duplicates across forms, and ignores blank rows. Test blank, duplicate, and valid inputs.
5. Use a model formset to edit the stock of several books at once, restricting the queryset to books with stock below five. Write a test that posts changes and checks which rows were updated.
6. Write a test using `assertNumQueries` to measure the queries needed to render an inline formset of five items, then reduce the number with `only()`, `select_related()`, or a smaller queryset, and record the difference.
7. Implement shared book choices for the item formset: evaluate the active book list once in the formset and give each form the same list of choices, then show that rendering uses one query for books instead of one per form. Explain why validation still needs per-row handling.

## Summary

A `ModelForm` builds its fields from a model, validates through model validation, and saves to the database. Always give it an explicit `fields` list: `"__all__"` and `exclude` make every future column user-editable. Use `save(commit=False)` to fill in values the view controls, and remember `save_m2m()`. Model validation skips unique checks and constraints that involve fields not on the form, so add explicit checks and keep database constraints as the final guard. Formsets manage many forms on one page, need their management form, and validate across rows in `clean()`, while inline formsets manage the children of one parent. Watch the cost of `ModelChoiceField` in formsets, apply snapshots only when rows are created, and save parents and children together in one transaction.
