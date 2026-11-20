## Overview

The Django admin is a ready-made web interface for browsing and editing the data in your models. It is generated from your model definitions, respects Django's permission system, records every change in an audit log, and takes a few lines of code to set up. For internal staff who maintain catalog data, review orders, or fix records, it is often the fastest way to give people the tools they need. It is also easy to misuse: as a customer-facing interface, as a substitute for proper workflows, or with default settings that issue hundreds of queries per page. This note covers registering models, the main `ModelAdmin` options (`list_display`, `list_filter`, `search_fields`, `readonly_fields`, `fieldsets`), inlines, custom actions, admin permissions, `list_select_related` and avoiding N+1 queries, facets (Django 5.0), who should use the admin, and how to secure it. It ends with a tested admin configuration for the bookstore: books, authors, and read-only orders with a computed total.

## Core concepts

### What the admin is

`django.contrib.admin` is an app. It builds a site from every model you register, with list pages (search, filters, sorting, pagination, bulk actions), add and change forms, delete confirmation, and a history of changes. Its URLs live under the path you mount it on (`path("admin/", admin.site.urls)`), and its pages are protected: a user needs `is_staff` and the right model permissions (note 4.3).

### Registering models

Registering a model with no options gives you a working but bare interface. Registering with a `ModelAdmin` subclass lets you configure it:

```python
@admin.register(Book)
class BookAdmin(admin.ModelAdmin[Book]):
    ...
```

The admin finds `admin.py` modules automatically (`autodiscover`) when the app is installed.

### The list page options

- `list_display`: columns shown. Entries can be fields, methods on the admin, or methods on the model.
- `list_filter`: sidebar filters on fields or custom filter classes.
- `search_fields`: fields searched by the search box, with optional lookups (`^title` for starts-with, `=isbn` for exact).
- `ordering`: default sort.
- `date_hierarchy`: drill-down by date.
- `list_per_page`: page size.
- `list_editable`: fields editable directly in the list.
- `list_select_related`: relations to join, to avoid one query per row.
- `show_facets`: counts next to filter choices (Django 5.0).

### The form page options

- `fields` and `fieldsets`: which fields appear and how they are grouped.
- `readonly_fields`: shown but not editable, and the place to display computed values.
- `autocomplete_fields`: replace a long select with a search-as-you-type widget (the related admin must define `search_fields`).
- `raw_id_fields`: a plain id input for huge related tables.
- `prepopulated_fields`: fill slugs from other fields in the browser.

### Inlines

An inline edits related rows on the parent's page: `TabularInline` (compact table) or `StackedInline`. Typical use: the items of an order.

### Custom display methods

A method on the admin that receives the object and returns a value shown as a column or read-only field. Decorate it with `@admin.display(description="...", ordering="field")` to set the header and make the column sortable.

### Actions

An action is a function that runs on the rows selected in the list, chosen from a dropdown: "mark as sold out", "export". Decorate with `@admin.action(description="...", permissions=[...])` to name it and restrict who may use it.

### Admin permissions

The admin uses the model permissions (`view`, `add`, `change`, `delete`). `ModelAdmin` exposes `has_view_permission`, `has_add_permission`, `has_change_permission`, and `has_delete_permission`, which you can override for rules such as "nobody may delete orders". Custom action permissions use `has_<name>_permission` methods.

### The audit log

Every add, change, and delete through the admin creates a `LogEntry` row recording the user, object, time, and what changed. It is a useful trail for support and compliance. Changes made outside the admin (code, shell, scripts) are not logged there.

### Who should use the admin

The admin is designed for trusted internal staff. It is not a customer interface: it exposes your data model directly, its design is generic, and its security model assumes people who are allowed to see and change data. If a workflow needs steps, rules, or approvals, build a proper view or service and keep the admin for corrections and operations.

### Securing the admin

- Only staff accounts, with narrow model permissions and roles (note 4.3), and very few superusers.
- HTTPS only, strong passwords, and two-factor authentication (note 4.4).
- Rate limits on the login page, and optionally network restrictions at the proxy (an allowlist of office or VPN addresses).
- A changed URL prefix reduces noise from bots but is not security.

## Basic syntax examples

### The simplest registration

```python
from django.contrib import admin

from apps.bookstore.models import Genre

admin.site.register(Genre)
```

### A configured ModelAdmin

```python
from django.contrib import admin

from apps.bookstore.models import Author, Book


@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin[Author]):
    list_display = ("name", "born_on")
    search_fields = ("name",)          # required for autocomplete from BookAdmin


@admin.register(Book)
class BookAdmin(admin.ModelAdmin[Book]):
    list_display = ("title", "author", "isbn", "price", "stock", "published_on")
    list_filter = ("format", "published_on")
    search_fields = ("title", "isbn", "author__name")
    list_select_related = ("author",)
    ordering = ("title",)
    readonly_fields = ("public_id", "created_at", "version")
    autocomplete_fields = ("author",)
    date_hierarchy = "published_on"
    list_per_page = 50
```

### Search field prefixes

```python
search_fields = (
    "title",          # icontains
    "=isbn",          # exact match
    "^author__name",  # starts with
)
```

### Grouping form fields

```python
fieldsets = (
    (None, {"fields": ("title", "author", "isbn")}),
    ("Pricing and stock", {"fields": ("price", "stock", "format")}),
    ("Metadata", {"fields": ("public_id", "version", "created_at"), "classes": ("collapse",)}),
)
```

### A computed column

```python
from django.contrib import admin


class BookAdmin(admin.ModelAdmin[Book]):
    list_display = ("title", "stock_status")

    @admin.display(description="Availability", ordering="stock")
    def stock_status(self, obj: Book) -> str:
        if obj.stock == 0:
            return "Sold out"
        return "Low" if obj.stock <= 5 else "In stock"
```

### An inline

```python
class OrderItemInline(admin.TabularInline[OrderItem, Order]):
    model = OrderItem
    extra = 0
    can_delete = False
    readonly_fields = ("book", "quantity", "unit_price")
```

### A custom list filter

```python
from django.contrib import admin
from django.db.models import QuerySet
from django.http import HttpRequest


class LowStockFilter(admin.SimpleListFilter):
    title = "stock level"
    parameter_name = "stock"

    def lookups(self, request: HttpRequest, model_admin: admin.ModelAdmin[Book]) -> list[tuple[str, str]]:
        return [("out", "Sold out"), ("low", "Low (5 or fewer)")]

    def queryset(self, request: HttpRequest, queryset: QuerySet[Book]) -> QuerySet[Book] | None:
        if self.value() == "out":
            return queryset.filter(stock=0)
        if self.value() == "low":
            return queryset.filter(stock__gt=0, stock__lte=5)
        return None
```

### An action

```python
from django.contrib import admin, messages
from django.db.models import F, QuerySet
from django.http import HttpRequest


class BookAdmin(admin.ModelAdmin[Book]):
    actions = ["restock_ten"]

    @admin.action(description="Add 10 units to selected books", permissions=["manage_inventory"])
    def restock_ten(self, request: HttpRequest, queryset: QuerySet[Book]) -> None:
        updated = queryset.update(stock=F("stock") + 10)
        self.message_user(request, f"Restocked {updated} books.", messages.SUCCESS)

    def has_manage_inventory_permission(self, request: HttpRequest) -> bool:
        return request.user.has_perm("bookstore.manage_inventory")
```

### Overriding permissions

```python
class OrderAdmin(admin.ModelAdmin[Order]):
    def has_add_permission(self, request: HttpRequest) -> bool:
        return False        # orders are created by checkout, never by hand

    def has_delete_permission(self, request: HttpRequest, obj: Order | None = None) -> bool:
        return False        # orders are records, never deleted
```

### Mounting the admin and branding it

```python
from django.contrib import admin
from django.urls import path

admin.site.site_header = "Bookstore administration"
admin.site.site_title = "Bookstore admin"
admin.site.index_title = "Operations"

urlpatterns = [
    path("backoffice/", admin.site.urls),
]
```

## Deeper behavior

### How the admin builds a page

For each registered model the admin creates a set of URLs (`admin:bookstore_book_changelist`, `..._add`, `..._change`, `..._delete`, `..._history`). The change list builds a queryset (`get_queryset()`), applies filters, search, and ordering, paginates it, and renders each row from `list_display`. The form pages build a `ModelForm` from the fields you configure, so everything about model forms in note 3.2 applies, including validation, `Meta.fields` allowlists, and unique checks that skip excluded fields.

### The admin and N+1 queries

The admin is a very common source of N+1 problems (note 2.7):

- `list_display` that includes a foreign key (`author`) renders `str(obj.author)` per row, one query per row unless you add `list_select_related`.
- A model's `__str__` that touches a relation causes queries wherever the object is displayed, including in every select box of a form.
- Display methods that query related rows run once per row.
- `autocomplete_fields` and `raw_id_fields` avoid loading every related row into a select box for large tables.

Fix them with `list_select_related`, annotations in `get_queryset()` for computed columns, and by keeping display methods free of queries. Test the number of queries of the list page with different row counts.

### Counting and pagination

The change list runs `COUNT(*)` queries (one for the filtered result and one for the unfiltered total). On very large tables these are slow. Set `show_full_result_count = False` to skip the unfiltered count, keep `list_per_page` modest, and make sure filters and search fields are indexed (note 2.10).

### Facets (Django 5.0)

With `show_facets = admin.ShowFacets.ALLOW` (the default), the admin shows a "Show counts" toggle that adds the number of matching rows to each filter choice, computed with extra queries only when the user turns it on. `ShowFacets.ALWAYS` computes them always, which costs queries on every page view, and `NEVER` hides the toggle.

### Annotated columns

Computed columns that need aggregates should be annotated once in `get_queryset()` and then read in a display method, and made sortable with `ordering="annotation_name"`. Computing the aggregate in the display method itself would be one query per row.

### Hooks you will use

- `get_queryset(request)`: restrict or annotate what the list shows, for example hiding soft-deleted rows or scoping to the user's organization.
- `save_model(request, obj, form, change)`: run code when saving, such as recording who changed something. Keep business rules in services and call them here rather than duplicating them.
- `get_readonly_fields(request, obj)`: make fields read-only after creation or for some users.
- `formfield_for_foreignkey(...)`: limit the choices of a foreign key.
- `has_*_permission` methods: per-object and per-user rules.

### Admin actions run on the queryset

An action receives a queryset of the selected rows. `queryset.update()` is fast and skips `save()` and signals. If the action must run per-object logic (side effects, validation), loop and call your service function, and handle partial failure with messages that say what succeeded and what did not.

### Permissions in the admin

A staff user needs at least `view` permission on a model to see it, `change` to edit, `add` to create, and `delete` to remove. Superusers have everything. A user with `change` but not `view` may still be able to see the form in some versions, so grant them together. Custom action permissions must exist as permissions on the model (declared in `Meta.permissions`) and as `has_<name>_permission` methods on the admin.

### Output in the admin is HTML

Values returned from display methods are escaped. If you need HTML (a link, a colored badge), build it with `format_html()`, which escapes its arguments. Using `mark_safe()` on text that includes data creates a stored cross-site scripting hole visible to your own staff, who are the most valuable targets.

### The admin is not a product

The more behavior you add (custom views, complex actions, workflow state machines), the more you rebuild a worse version of your application inside a generic tool. Use the admin for the 20 percent of internal tasks it does very well, build real interfaces for the rest, and keep the admin read-only for data that must change only through validated workflows.

### Hardening checklist

- Serve over HTTPS and set the secure cookie settings (note 4.1).
- Require two-factor authentication for staff (note 4.4).
- Add rate limiting for the login page (note 4.4), and consider an IP allowlist at the proxy.
- Give staff narrow roles, never superuser by default.
- Review `LogEntry` rows or ship them to your logging system.
- Do not expose the admin on the same domain as untrusted user content (note 3.3).

## Industry-level snippet

An admin configuration for the bookstore that is fast (no per-row queries), safe (orders are read-only records, a dangerous action requires a dedicated permission), and tested. It uses an annotation for the order total, a read-only inline for order items, and a custom stock filter.

`apps/bookstore/admin.py`:

```python
from decimal import Decimal

from django.contrib import admin, messages
from django.db.models import DecimalField, F, QuerySet, Sum
from django.http import HttpRequest

from apps.bookstore.models import Author, Book, Order, OrderItem


class LowStockFilter(admin.SimpleListFilter):
    title = "stock level"
    parameter_name = "stock"

    def lookups(
        self, request: HttpRequest, model_admin: admin.ModelAdmin[Book]
    ) -> list[tuple[str, str]]:
        return [("out", "Sold out"), ("low", "Low (5 or fewer)")]

    def queryset(
        self, request: HttpRequest, queryset: QuerySet[Book]
    ) -> QuerySet[Book] | None:
        if self.value() == "out":
            return queryset.filter(stock=0)
        if self.value() == "low":
            return queryset.filter(stock__gt=0, stock__lte=5)
        return None


@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin[Author]):
    list_display = ("name", "born_on")
    search_fields = ("name",)


@admin.register(Book)
class BookAdmin(admin.ModelAdmin[Book]):
    list_display = ("title", "author", "isbn", "price", "stock", "stock_status")
    list_filter = ("format", LowStockFilter)
    search_fields = ("title", "=isbn", "author__name")
    list_select_related = ("author",)
    ordering = ("title",)
    readonly_fields = ("public_id", "created_at", "version")
    autocomplete_fields = ("author",)
    show_facets = admin.ShowFacets.ALLOW
    list_per_page = 50
    actions = ["restock_ten"]
    fieldsets = (
        (None, {"fields": ("title", "author", "isbn", "format", "published_on")}),
        ("Pricing and stock", {"fields": ("price", "stock")}),
        ("Metadata", {"fields": ("public_id", "version", "created_at"), "classes": ("collapse",)}),
    )

    def get_queryset(self, request: HttpRequest) -> QuerySet[Book]:
        return super().get_queryset(request).filter(deleted_at__isnull=True)

    @admin.display(description="Availability", ordering="stock")
    def stock_status(self, obj: Book) -> str:
        if obj.stock == 0:
            return "Sold out"
        return "Low" if obj.stock <= 5 else "In stock"

    @admin.action(description="Add 10 units to selected books", permissions=["manage_inventory"])
    def restock_ten(self, request: HttpRequest, queryset: QuerySet[Book]) -> None:
        updated = queryset.update(stock=F("stock") + 10)
        self.message_user(request, f"Restocked {updated} books.", messages.SUCCESS)

    def has_manage_inventory_permission(self, request: HttpRequest) -> bool:
        return request.user.has_perm("bookstore.manage_inventory")


class OrderItemInline(admin.TabularInline[OrderItem, Order]):
    model = OrderItem
    extra = 0
    can_delete = False
    readonly_fields = ("book", "quantity", "unit_price")

    def has_add_permission(self, request: HttpRequest, obj: Order | None = None) -> bool:
        return False


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin[Order]):
    list_display = ("id", "customer_email", "customer", "status", "created_at", "order_total")
    list_filter = ("status",)
    search_fields = ("customer_email", "=id")
    list_select_related = ("customer",)
    readonly_fields = ("customer_email", "customer", "created_at", "idempotency_key")
    inlines = [OrderItemInline]
    show_full_result_count = False

    def get_queryset(self, request: HttpRequest) -> QuerySet[Order]:
        return super().get_queryset(request).annotate(
            total=Sum(
                F("items__quantity") * F("items__unit_price"),
                output_field=DecimalField(max_digits=12, decimal_places=2),
            )
        )

    @admin.display(description="Total", ordering="total")
    def order_total(self, obj: Order) -> Decimal | None:
        return getattr(obj, "total", None)

    def has_add_permission(self, request: HttpRequest) -> bool:
        return False  # orders are created by checkout

    def has_delete_permission(self, request: HttpRequest, obj: Order | None = None) -> bool:
        return False  # orders are records, never deleted
```

`apps/bookstore/tests/test_admin.py`:

```python
from decimal import Decimal

from django.contrib.auth.models import Permission
from django.db import connection
from django.test import TestCase
from django.test.utils import CaptureQueriesContext
from django.urls import reverse
from django.utils import timezone

from apps.accounts.models import User
from apps.bookstore.models import Order
from apps.bookstore.services.orders import OrderLine, create_order

from .factories import make_author, make_book

PASSWORD = "correct-horse-battery-staple"


def give(user: User, *codenames: str) -> None:
    permissions = Permission.objects.filter(
        content_type__app_label="bookstore", codename__in=codenames
    )
    user.user_permissions.add(*permissions)


class AdminTestCase(TestCase):
    def setUp(self) -> None:
        self.editor = User.objects.create_user("editor@example.com", PASSWORD, is_staff=True)
        give(self.editor, "view_book", "change_book", "view_author")
        self.manager = User.objects.create_user("manager@example.com", PASSWORD, is_staff=True)
        give(self.manager, "view_book", "change_book", "manage_inventory")
        self.customer = User.objects.create_user("customer@example.com", PASSWORD)
        self.root = User.objects.create_superuser("root@example.com", PASSWORD)
        self.changelist = reverse("admin:bookstore_book_changelist")


class BookAdminTests(AdminTestCase):
    def test_non_staff_users_are_sent_to_the_admin_login(self) -> None:
        self.client.force_login(self.customer)
        response = self.client.get(self.changelist)
        self.assertEqual(response.status_code, 302)
        self.assertIn("/login/", response["Location"])

    def test_staff_can_search_and_filter(self) -> None:
        make_book(title="Dune", stock=0)
        make_book(title="Earthsea", stock=3)
        self.client.force_login(self.editor)
        found = self.client.get(self.changelist, {"q": "dune"})
        self.assertContains(found, "Dune")
        self.assertNotContains(found, "Earthsea")
        low = self.client.get(self.changelist, {"stock": "low"})
        self.assertContains(low, "Earthsea")
        self.assertNotContains(low, "Dune")

    def test_soft_deleted_books_are_hidden(self) -> None:
        book = make_book(title="Hidden")
        book.deleted_at = timezone.now()
        book.save(update_fields=["deleted_at"])
        self.client.force_login(self.editor)
        self.assertNotContains(self.client.get(self.changelist), "Hidden")

    def test_model_permissions_are_enforced(self) -> None:
        self.client.force_login(self.editor)
        self.assertEqual(self.client.get(reverse("admin:bookstore_book_add")).status_code, 403)

    def test_read_only_fields_have_no_input(self) -> None:
        book = make_book()
        self.client.force_login(self.editor)
        response = self.client.get(reverse("admin:bookstore_book_change", args=[book.pk]))
        self.assertContains(response, str(book.public_id))
        self.assertNotContains(response, 'name="public_id"')

    def test_the_list_page_does_not_run_a_query_per_row(self) -> None:
        self.client.force_login(self.editor)

        def queries_with(count: int) -> int:
            for index in range(count):
                make_book(author=make_author(f"Author {count}-{index}"))
            with CaptureQueriesContext(connection) as context:
                self.assertEqual(self.client.get(self.changelist).status_code, 200)
            return len(context)

        few = queries_with(3)
        many = queries_with(10)
        self.assertEqual(few, many)


class ActionTests(AdminTestCase):
    def post_action(self, user: User, action: str, *pks: int) -> None:
        self.client.force_login(user)
        self.client.post(
            self.changelist,
            {"action": action, "_selected_action": [str(pk) for pk in pks]},
        )

    def test_the_action_needs_its_own_permission(self) -> None:
        book = make_book(stock=1)
        self.post_action(self.editor, "restock_ten", book.pk)
        book.refresh_from_db()
        self.assertEqual(book.stock, 1)

    def test_a_permitted_user_can_run_the_action(self) -> None:
        book = make_book(stock=1)
        self.post_action(self.manager, "restock_ten", book.pk)
        book.refresh_from_db()
        self.assertEqual(book.stock, 11)


class OrderAdminTests(AdminTestCase):
    def test_the_list_shows_the_computed_total(self) -> None:
        book = make_book(price=Decimal("10.00"), stock=10)
        create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 3)])
        self.client.force_login(self.root)
        response = self.client.get(reverse("admin:bookstore_order_changelist"))
        self.assertContains(response, "30.00")

    def test_orders_cannot_be_added_or_deleted_even_by_superusers(self) -> None:
        book = make_book(stock=5)
        order = create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])
        self.client.force_login(self.root)
        self.assertEqual(self.client.get(reverse("admin:bookstore_order_add")).status_code, 403)
        delete_url = reverse("admin:bookstore_order_delete", args=[order.pk])
        self.assertEqual(self.client.get(delete_url).status_code, 403)
        self.assertTrue(Order.objects.filter(pk=order.pk).exists())
```

The query-count test is the important one for the admin: it fails the moment someone adds a column that triggers a query per row, which is how admin pages quietly become slow.

## Common mistakes

- Adding a foreign key to `list_display` without `list_select_related`, causing one query per row.
- Writing a `__str__` that touches related objects, multiplying queries everywhere the object is displayed.
- Putting computed columns that query in display methods instead of annotating the queryset.
- Giving customers or untrusted users `is_staff` so they can reach "just one" admin page.
- Making many staff accounts superusers instead of granting narrow permissions.
- Using `mark_safe` on text that includes data in a display method.
- Using actions with `queryset.update()` when per-object business rules or signals are needed.
- Letting the admin edit data that must change only through validated workflows, such as order status or stock levels with side effects.
- Exposing the admin on the public internet without HTTPS, rate limits, and two-factor authentication.
- Assuming that renaming the admin URL secures it.
- Forgetting that `search_fields` on large tables with `icontains` scans the table, and not indexing or restricting them.
- Registering sensitive models (password hashes, tokens) with editable fields.

## Practice exercises

1. Register `Genre` with a self-referencing `parent` in `list_display`, then use the debug toolbar to count the queries of its list page. Fix any per-row queries with `list_select_related`.
2. Add an inline for `Review` on the book change page that is read-only, ordered newest first, and limited to the latest 20 reviews. Explain how you limited the rows.
3. Add a "Mark selected books as discontinued" action that sets `deleted_at` through `BookQuerySet.soft_delete()`, requires the `change_book` permission, and reports how many rows changed. Test it for a user with and without the permission.
4. Add `ApiKey` to the admin as a read-only list (name, user, prefix, created, last used, status) with no add or change permission and a "Revoke selected keys" action that uses `revoke_api_key`. Make sure the hash is never shown.
5. Write a test that visits every admin changelist for every registered model as a superuser and asserts a 200 response, so broken `list_display` entries are caught.
6. Use `LogEntry` to build a read-only "Recent changes" admin page that lists the last 50 changes with the user, object, and action. Decide whether it should be visible to every staff user and defend the answer.
7. Add a `ReviewAdmin` with a custom filter for ratings of two stars or fewer, `list_select_related` for the book, and a read-only body preview column built with `format_html` that truncates long text. Test that a review body containing a script tag is escaped in the list.

## Summary

The Django admin generates a permission-aware interface for your models, with search, filters, bulk actions, inlines, and an audit log. Configure it with `list_display`, `list_filter`, `search_fields`, `readonly_fields`, `fieldsets`, and custom display methods, and make it fast with `list_select_related`, annotations in `get_queryset()`, `autocomplete_fields`, and by avoiding queries in display methods. Control access with model permissions, `has_*_permission` overrides (no adds or deletes for records that must be preserved), and dedicated permissions for dangerous actions. Escape output with `format_html`, keep business rules in services, use the admin for internal operations rather than customers, and secure it with HTTPS, two-factor authentication, rate limits, and narrow roles. Test the permissions and the query count, because those are the parts that regress silently.
