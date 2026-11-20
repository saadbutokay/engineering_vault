## Overview

Authentication tells you who a user is. Authorization decides what they may do: which pages they can open, which records they can read, and which actions they can perform. Broken access control, where a user can see or change something that is not theirs, tops the OWASP list of the most common serious web vulnerabilities, and it is almost always a missing check rather than a clever attack. Django provides staff and superuser flags, a permission system with groups, decorators and mixins that enforce permissions, and the tools to scope what each user can query. This note covers `is_staff` and `is_superuser`, permissions and groups, `has_perm`, `permission_required`, `UserPassesTestMixin`, object-level permissions and the insecure direct object reference (IDOR) problem, scoping querysets to the current user, `django-guardian` as an overview, policy functions, and role design. It ends with a tested design in which orders are private to their owners, support staff gain access through a role, and refunds require a permission and a valid state.

This note adds two fields to the bookstore models: `Order.customer` (the owning user) and custom permissions on `Order` and `Book`.

## Core concepts

### Flags on the user

- `is_active`: if false, the user cannot log in and has no permissions.
- `is_staff`: may log in to the Django admin. It grants no model permissions by itself.
- `is_superuser`: has every permission automatically. Keep the number of superusers tiny.

### Permissions

A permission is a named right such as `bookstore.change_book`, written as `app_label.codename`. For every model Django creates four default permissions when migrations run: `add_`, `change_`, `delete_`, and `view_` plus the model name. You can declare more in a model's `Meta.permissions`:

```python
class Meta:
    permissions = [("refund_order", "Can refund orders")]
```

### Groups and roles

A group is a named set of permissions. Assign permissions to groups, and users to groups, so that a user's rights come from their roles ("Support", "Catalog manager") rather than from individual grants. Users may belong to several groups, and their permissions add up.

### Checking permissions

- `user.has_perm("bookstore.refund_order")` returns a boolean.
- `user.has_perms([...])` requires all of them.
- `user.get_all_permissions()` returns the full set as strings.
- Templates can use the `perms` variable: `{% if perms.bookstore.refund_order %}`.

### Enforcing in views

- `@login_required` and `@permission_required("app.codename", raise_exception=True)` for function views.
- `LoginRequiredMixin`, `PermissionRequiredMixin` (with `permission_required = ...`), and `UserPassesTestMixin` (with `test_func()`) for class-based views.
- `raise PermissionDenied` anywhere in a view to produce a 403 response.

### Model-level versus object-level permissions

Django's built-in permissions are model-level: "may change books" means all books. They cannot say "may change this book" or "may view only their own orders". Object-level rules need more: scoping the queryset, an explicit policy check, or a package such as `django-guardian`.

### IDOR

An insecure direct object reference happens when a view takes an identifier from the request (`/orders/42/`) and loads the object without checking that the current user may access it. Changing the number in the URL then shows someone else's data. It is the most frequent real-world authorization bug, and it appears in APIs as often as in pages.

### Scoping querysets

The most robust defense against IDOR is to never fetch objects from an unscoped queryset. Start from a queryset that contains only what the user may see, and look up the identifier inside it: `get_object_or_404(orders_visible_to(request.user), pk=pk)`. An object outside the scope simply does not exist for that user.

### 404 or 403

Returning 403 Forbidden for someone else's order confirms that the order exists. Returning 404 Not Found does not. For private resources, prefer 404. Use 403 when the resource's existence is public but the action is not allowed (a refund button for a visible order).

### Policy functions

A policy function answers one question about one user and one object: `can_view_order(user, order)`, `can_refund_order(user, order)`. Keeping them as plain functions in one module makes rules readable, reusable by views, templates, APIs, and tasks, and easy to test without HTTP.

### django-guardian

`django-guardian` adds per-object permissions stored in the database: grant "view" on this order to that user, and query which objects a user may see. It suits sharing models (documents shared with specific people). For rules that follow from relationships (owner, same team, same tenant), a queryset scope and a policy function are simpler and faster.

### Role design

Define roles from real jobs, grant each the least it needs, and give permissions to roles instead of people. Separate duties that should not be combined (the person who issues refunds should not be the one who approves their own refund). Keep the definitions in code so changes are reviewed, and review who holds powerful roles regularly.

## Basic syntax examples

### Default and custom permissions

```python
from django.contrib.auth.models import Permission

codenames = Permission.objects.filter(content_type__app_label="bookstore").values_list(
    "codename", flat=True
)
# ['add_book', 'change_book', 'delete_book', 'view_book', 'add_order', ..., 'refund_order']
```

### Checking permissions

```python
user.has_perm("bookstore.change_book")
user.has_perms(["bookstore.view_order", "bookstore.refund_order"])
user.get_all_permissions()     # {'bookstore.view_order', ...}
```

### Groups

```python
from django.contrib.auth.models import Group, Permission

support, _ = Group.objects.get_or_create(name="Support")
support.permissions.add(Permission.objects.get(codename="view_order"))
user.groups.add(support)
```

### Function views

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.http import HttpRequest, HttpResponse
from django.views.decorators.http import require_POST


@login_required
@permission_required("bookstore.manage_inventory", raise_exception=True)
@require_POST
def adjust_stock(request: HttpRequest, pk: int) -> HttpResponse:
    ...
```

With `raise_exception=True` a logged-in user who lacks the permission receives a 403 instead of being redirected to the login page, which would be confusing because they are already logged in.

### Class-based views

```python
from django.contrib.auth.mixins import (
    LoginRequiredMixin,
    PermissionRequiredMixin,
    UserPassesTestMixin,
)
from django.db.models import QuerySet
from django.views.generic import ListView, TemplateView

from apps.bookstore.models import Order


class MyOrdersView(LoginRequiredMixin, ListView[Order]):
    template_name = "bookstore/my_orders.html"

    def get_queryset(self) -> QuerySet[Order]:
        return Order.objects.filter(customer=self.request.user)   # scoped, never all orders


class InventoryView(PermissionRequiredMixin, TemplateView):
    permission_required = "bookstore.manage_inventory"
    template_name = "bookstore/inventory.html"


class StaffOnlyView(UserPassesTestMixin, TemplateView):
    template_name = "bookstore/staff.html"

    def test_func(self) -> bool:
        user = self.request.user
        return user.is_authenticated and user.is_staff
```

Mixins go before the generic view in the base class list, as note 1.3 explained.

### Showing or hiding interface elements

```html
{% if perms.bookstore.refund_order %}
  <form method="post" action="{% url 'bookstore:refund_order' pk=order.pk %}">
    {% csrf_token %}
    <button type="submit">Refund</button>
  </form>
{% endif %}
```

This only affects what is shown. The view must enforce the rule independently.

### Raising PermissionDenied

```python
from django.core.exceptions import PermissionDenied


def can_edit(request: HttpRequest, order: Order) -> None:
    if order.customer_id != request.user.pk:
        raise PermissionDenied("This is not your order.")
```

### Scoping before lookup

```python
from django.shortcuts import get_object_or_404

order = get_object_or_404(Order.objects.filter(customer=request.user), pk=pk)   # 404 for others
```

### django-guardian (overview)

```python
# settings.py
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "guardian.backends.ObjectPermissionBackend",
]
```

```python
from guardian.shortcuts import assign_perm, get_objects_for_user

assign_perm("view_order", user, order)                 # grant on one object
user.has_perm("bookstore.view_order", order)           # object-level check now works
visible = get_objects_for_user(user, "bookstore.view_order")
```

## Deeper behavior

### How has_perm works

`user.has_perm(perm, obj=None)` returns true immediately for an active superuser. Otherwise it asks each authentication backend. `ModelBackend` returns false for inactive users, collects the permissions granted directly to the user and through their groups, and checks membership. Inactive users and anonymous users never have permissions.

### has_perm with an object returns false

`ModelBackend` does not implement object-level permissions. If you call `user.has_perm("bookstore.view_order", order)`, it returns an empty set of permissions for the object and the answer is false, even for a user who holds `bookstore.view_order` model-wide. Because of this, a policy function that wants model-wide permission as a fallback must call `has_perm` without the object, as the example below does. Passing an object only works with a backend that supports it, such as guardian's.

### Permissions are cached on the user instance

The first `has_perm` call loads the user's permissions and stores them on the instance. Changing the user's groups afterward does not affect that instance. In a request, the user is loaded fresh every time, so this matters mainly in tests and long-running code. Fetch the user again (`User.objects.get(pk=...)`) to see new permissions.

### Where default permissions come from

Permissions are rows created by the `post_migrate` signal after migrations run. A new model's permissions appear after you migrate. Code that looks up a permission before then will not find it. Tests get them automatically because the test database runs the migrations.

### Superusers bypass everything

An active superuser passes every permission check and every `PermissionRequiredMixin`. It is convenient for operations and dangerous in daily use. Give people the narrow roles they need, and keep superuser accounts for emergencies and setup.

### 404 versus 403 in practice

`get_object_or_404` on a scoped queryset returns 404 for objects the user cannot see, which hides their existence and simplifies the code. When a user can see an object but may not perform an action on it, raise `PermissionDenied` for a 403. Keep the policy consistent, because inconsistent responses leak information (one URL answers 404 for missing and 403 for forbidden, revealing which exist).

### Mixins and decorators run before your code

`LoginRequiredMixin` and `PermissionRequiredMixin` check in `dispatch()`, before the handler runs. If you override `dispatch()` or put the mixin after the view class, the check can be skipped. For decorators, order matters too: the outermost decorator runs first.

### The default should be deny

Add checks to everything and prefer mechanisms that fail closed: `LoginRequiredMiddleware` (note 4.2) makes login the default, a scoped base queryset makes "nothing" the default for unknown users, and a policy function that returns `False` unless a rule matches gives "no" for forgotten cases. Lists of "forbidden" cases fail open.

### Enforce on every path

A rule must hold wherever the operation can be triggered: HTML views, JSON endpoints, the admin, management commands, background tasks, and bulk operations. Put the rule in a policy function or service and call it from each path. A rule checked only in the UI protects nothing.

### Admin permissions

The admin applies the model permissions to its own views. For object-level rules in the admin, override `get_queryset()` and the `has_view_permission`, `has_change_permission`, and `has_delete_permission` methods on `ModelAdmin` (note 5.1).

### Roles in code or in the database

Defining roles in code (a mapping from role name to permission strings, applied by a command or a migration) makes changes reviewable and reproducible across environments. Letting administrators edit groups in the admin is flexible but invisible to code review. A common compromise is to define the standard roles in code and use the admin for exceptions with an audit trail (note 11.6).

### Multi-tenant scoping

If your application serves several organizations, every queryset must be scoped to the user's organization, and the scope should be applied by default in a manager or base queryset so developers cannot forget it (note 11.6).

## Industry-level snippet

Orders belong to customers and are private. Support staff can see all orders and, if the order is paid or shipped, cancel it. Catalog managers adjust stock. The design uses a scoped queryset to prevent IDOR, policy functions for decisions, roles defined in code and synchronized by a command, a coarse permission check plus a fine object rule on the refund view, and tests that prove each guarantee, including the permission-cache and `has_perm(obj)` surprises.

Changes to `apps/bookstore/models.py`:

```python
from django.conf import settings

# in Book.Meta (add to the existing class)
permissions = [("manage_inventory", "Can adjust book stock")]

# in Order
customer = models.ForeignKey(
    settings.AUTH_USER_MODEL,
    null=True,
    blank=True,
    on_delete=models.SET_NULL,
    related_name="orders",
)

class Meta:
    permissions = [("refund_order", "Can refund orders")]
```

Run `makemigrations` and `migrate`.

`apps/bookstore/policies.py`:

```python
"""Authorization rules for orders. Plain functions, easy to test."""

from django.contrib.auth.models import AnonymousUser
from django.db.models import QuerySet

from apps.accounts.models import User
from apps.bookstore.models import Order

Actor = User | AnonymousUser

VIEW_ANY_ORDER = "bookstore.view_order"
REFUND_ORDER = "bookstore.refund_order"
REFUNDABLE_STATUSES = frozenset({Order.Status.PAID, Order.Status.SHIPPED})


def orders_visible_to(user: Actor) -> QuerySet[Order]:
    """The orders this user may see: everything, their own, or nothing."""
    if not user.is_authenticated or not user.is_active:
        return Order.objects.none()
    if user.has_perm(VIEW_ANY_ORDER):  # no object argument, see the notes above
        return Order.objects.all()
    return Order.objects.filter(customer=user)


def can_view_order(user: Actor, order: Order) -> bool:
    if not user.is_authenticated or not user.is_active:
        return False
    return order.customer_id == user.pk or user.has_perm(VIEW_ANY_ORDER)


def can_refund_order(user: Actor, order: Order) -> bool:
    """Needs the permission and an order that has actually been paid."""
    if not user.is_authenticated or not user.is_active:
        return False
    return user.has_perm(REFUND_ORDER) and order.status in REFUNDABLE_STATUSES
```

`apps/bookstore/order_views.py`:

```python
import logging

from django.contrib.auth.decorators import login_required, permission_required
from django.core.exceptions import PermissionDenied
from django.http import HttpRequest, HttpResponse, JsonResponse
from django.shortcuts import get_object_or_404
from django.views.decorators.http import require_POST

from apps.bookstore.models import Order
from apps.bookstore.policies import can_refund_order, orders_visible_to

logger = logging.getLogger(__name__)


@login_required
def my_orders(request: HttpRequest) -> HttpResponse:
    orders = Order.objects.filter(customer=request.user).order_by("-created_at", "-id")
    return JsonResponse({"orders": [order.pk for order in orders]})


@login_required
def order_detail(request: HttpRequest, pk: int) -> HttpResponse:
    # Scope first, then look up: other people's orders do not exist for this user.
    order = get_object_or_404(orders_visible_to(request.user), pk=pk)
    return JsonResponse(
        {"id": order.pk, "status": order.status, "customer_email": order.customer_email}
    )


@login_required
@permission_required("bookstore.refund_order", raise_exception=True)   # coarse gate
@require_POST
def refund_order(request: HttpRequest, pk: int) -> HttpResponse:
    order = get_object_or_404(orders_visible_to(request.user), pk=pk)
    if not can_refund_order(request.user, order):                     # fine, per-object rule
        raise PermissionDenied("This order cannot be refunded.")
    Order.objects.filter(pk=order.pk).update(status=Order.Status.CANCELLED)
    logger.info("User %s refunded order %s", request.user.pk, order.pk)
    return JsonResponse({"id": order.pk, "status": Order.Status.CANCELLED.value})
```

Add to the bookstore `urlpatterns`:

```python
path("my/orders/", order_views.my_orders, name="my_orders"),
path("orders/<int:pk>/", order_views.order_detail, name="order_detail"),
path("orders/<int:pk>/refund/", order_views.refund_order, name="refund_order"),
```

`apps/accounts/roles.py`:

```python
"""Roles defined in code and synchronized to groups."""

import logging
from collections.abc import Mapping, Sequence

from django.contrib.auth.models import Group, Permission
from django.db import transaction

logger = logging.getLogger(__name__)

ROLES: dict[str, tuple[str, ...]] = {
    "Support": ("bookstore.view_order", "bookstore.refund_order"),
    "Catalog manager": (
        "bookstore.view_book",
        "bookstore.add_book",
        "bookstore.change_book",
        "bookstore.manage_inventory",
    ),
}


class UnknownPermissionError(LookupError):
    """A role refers to a permission that does not exist."""


def _resolve(label: str) -> Permission:
    app_label, _, codename = label.partition(".")
    try:
        return Permission.objects.get(content_type__app_label=app_label, codename=codename)
    except Permission.DoesNotExist as exc:
        raise UnknownPermissionError(label) from exc


def sync_roles(roles: Mapping[str, Sequence[str]] = ROLES) -> dict[str, int]:
    """Create or update groups so they match ``roles`` exactly.

    All changes happen in one transaction, so a typo in a permission name
    leaves nothing half-applied. Returns the number of permissions per role.
    """
    result: dict[str, int] = {}
    with transaction.atomic():
        for name, labels in roles.items():
            permissions = [_resolve(label) for label in labels]
            group, _ = Group.objects.get_or_create(name=name)
            group.permissions.set(permissions)
            result[name] = len(permissions)
    logger.info("Synchronized roles: %s", ", ".join(sorted(result)))
    return result
```

`apps/accounts/management/commands/sync_roles.py` (with `__init__.py` files in `management/` and `commands/`):

```python
from typing import Any

from django.core.management.base import BaseCommand, CommandError

from apps.accounts.roles import UnknownPermissionError, sync_roles


class Command(BaseCommand):
    help = "Create or update the permission groups defined in apps.accounts.roles."

    def handle(self, *args: Any, **options: Any) -> None:
        try:
            result = sync_roles()
        except UnknownPermissionError as exc:
            raise CommandError(f"Unknown permission in a role: {exc}") from exc
        for name, count in sorted(result.items()):
            self.stdout.write(f"{name}: {count} permissions")
```

Run `uv run python manage.py sync_roles` as part of each deployment, after `migrate`.

`apps/bookstore/tests/test_authorization.py`:

```python
from io import StringIO
from unittest import mock

from django.contrib.auth.models import AnonymousUser, Group
from django.core.management import call_command
from django.core.management.base import CommandError
from django.test import TestCase
from django.urls import reverse

from apps.accounts.models import User
from apps.accounts.roles import UnknownPermissionError, sync_roles
from apps.bookstore.models import Order
from apps.bookstore.policies import can_refund_order, can_view_order, orders_visible_to

PASSWORD = "correct-horse-battery-staple"


class AuthorizationTestCase(TestCase):
    def setUp(self) -> None:
        sync_roles()
        self.alice = User.objects.create_user("alice@example.com", PASSWORD)
        self.bob = User.objects.create_user("bob@example.com", PASSWORD)
        self.support = User.objects.create_user("support@example.com", PASSWORD)
        self.support.groups.add(Group.objects.get(name="Support"))
        self.catalog = User.objects.create_user("catalog@example.com", PASSWORD)
        self.catalog.groups.add(Group.objects.get(name="Catalog manager"))
        self.admin = User.objects.create_superuser("root@example.com", PASSWORD)
        self.alice_order = self.make_order(self.alice, Order.Status.PAID)
        self.bob_order = self.make_order(self.bob, Order.Status.PENDING)

    def make_order(self, customer: User, status: str) -> Order:
        return Order.objects.create(
            customer_email=customer.email, customer=customer, status=status
        )


class PolicyTests(AuthorizationTestCase):
    def test_owners_staff_with_the_permission_and_superusers_can_view(self) -> None:
        for user in (self.alice, self.support, self.admin):
            with self.subTest(user=user.email):
                self.assertTrue(can_view_order(user, self.alice_order))

    def test_others_cannot_view(self) -> None:
        for user in (self.bob, self.catalog):
            with self.subTest(user=user.email):
                self.assertFalse(can_view_order(user, self.alice_order))

    def test_anonymous_and_inactive_users_have_no_access(self) -> None:
        self.assertFalse(can_view_order(AnonymousUser(), self.alice_order))
        self.support.is_active = False
        self.support.save(update_fields=["is_active"])
        support = User.objects.get(pk=self.support.pk)
        self.assertFalse(can_view_order(support, self.alice_order))
        self.assertEqual(orders_visible_to(support).count(), 0)

    def test_visible_orders_are_scoped(self) -> None:
        self.assertEqual(set(orders_visible_to(self.alice)), {self.alice_order})
        self.assertEqual(set(orders_visible_to(self.support)), {self.alice_order, self.bob_order})

    def test_refunds_need_the_permission_and_a_paid_order(self) -> None:
        self.assertTrue(can_refund_order(self.support, self.alice_order))
        self.assertFalse(can_refund_order(self.support, self.bob_order))   # still pending
        self.assertFalse(can_refund_order(self.alice, self.alice_order))   # owner, no permission

    def test_model_backend_ignores_permissions_when_an_object_is_passed(self) -> None:
        self.assertTrue(self.support.has_perm("bookstore.view_order"))
        self.assertFalse(self.support.has_perm("bookstore.view_order", self.alice_order))

    def test_permissions_are_cached_on_the_user_instance(self) -> None:
        user = User.objects.create_user("new@example.com", PASSWORD)
        self.assertFalse(user.has_perm("bookstore.refund_order"))
        user.groups.add(Group.objects.get(name="Support"))
        self.assertFalse(user.has_perm("bookstore.refund_order"))          # cached
        self.assertTrue(User.objects.get(pk=user.pk).has_perm("bookstore.refund_order"))


class OrderViewTests(AuthorizationTestCase):
    def detail_url(self, order: Order) -> str:
        return reverse("bookstore:order_detail", kwargs={"pk": order.pk})

    def refund_url(self, order: Order) -> str:
        return reverse("bookstore:refund_order", kwargs={"pk": order.pk})

    def test_owners_see_their_own_orders(self) -> None:
        self.client.force_login(self.alice)
        self.assertEqual(self.client.get(self.detail_url(self.alice_order)).status_code, 200)

    def test_changing_the_id_in_the_url_does_not_expose_other_orders(self) -> None:
        self.client.force_login(self.alice)
        self.assertEqual(self.client.get(self.detail_url(self.bob_order)).status_code, 404)

    def test_support_can_see_any_order(self) -> None:
        self.client.force_login(self.support)
        self.assertEqual(self.client.get(self.detail_url(self.bob_order)).status_code, 200)

    def test_anonymous_visitors_are_sent_to_the_login_page(self) -> None:
        response = self.client.get(self.detail_url(self.alice_order))
        self.assertEqual(response.status_code, 302)
        self.assertIn(reverse("accounts:login"), response["Location"])

    def test_my_orders_lists_only_the_users_orders(self) -> None:
        self.client.force_login(self.alice)
        self.assertEqual(self.client.get(reverse("bookstore:my_orders")).json(), {"orders": [self.alice_order.pk]})

    def test_refund_requires_the_permission(self) -> None:
        self.client.force_login(self.alice)
        self.assertEqual(self.client.post(self.refund_url(self.alice_order)).status_code, 403)

    def test_support_can_refund_a_paid_order(self) -> None:
        self.client.force_login(self.support)
        response = self.client.post(self.refund_url(self.alice_order))
        self.assertEqual(response.status_code, 200)
        self.alice_order.refresh_from_db()
        self.assertEqual(self.alice_order.status, Order.Status.CANCELLED)

    def test_support_cannot_refund_an_order_that_is_not_paid(self) -> None:
        self.client.force_login(self.support)
        self.assertEqual(self.client.post(self.refund_url(self.bob_order)).status_code, 403)
        self.bob_order.refresh_from_db()
        self.assertEqual(self.bob_order.status, Order.Status.PENDING)

    def test_refund_endpoint_rejects_get(self) -> None:
        self.client.force_login(self.support)
        self.assertEqual(self.client.get(self.refund_url(self.alice_order)).status_code, 405)


class RoleTests(TestCase):
    def test_sync_is_idempotent(self) -> None:
        first = sync_roles()
        second = sync_roles()
        self.assertEqual(first, second)
        self.assertEqual(Group.objects.count(), 2)
        support = Group.objects.get(name="Support")
        self.assertEqual(
            set(support.permissions.values_list("codename", flat=True)),
            {"view_order", "refund_order"},
        )

    def test_sync_removes_permissions_no_longer_listed(self) -> None:
        sync_roles()
        sync_roles({"Support": ("bookstore.view_order",)})
        support = Group.objects.get(name="Support")
        self.assertEqual(list(support.permissions.values_list("codename", flat=True)), ["view_order"])

    def test_an_unknown_permission_changes_nothing(self) -> None:
        with self.assertRaises(UnknownPermissionError):
            sync_roles({"Ghost": ("bookstore.view_order", "bookstore.does_not_exist")})
        self.assertFalse(Group.objects.filter(name="Ghost").exists())

    def test_the_command_reports_each_role(self) -> None:
        out = StringIO()
        call_command("sync_roles", stdout=out)
        self.assertIn("Support: 2 permissions", out.getvalue())

    def test_the_command_reports_configuration_errors(self) -> None:
        with mock.patch(
            "apps.accounts.management.commands.sync_roles.sync_roles",
            side_effect=UnknownPermissionError("bookstore.nope"),
        ):
            with self.assertRaises(CommandError):
                call_command("sync_roles")
```

The tests show the authorization guarantees one at a time: a user cannot reach another user's order by editing the URL, a role grants exactly what it lists, a refund needs both the permission and a valid state, and the surprising behaviors of Django's permission checks are pinned down so nobody relies on them by accident.

## Common mistakes

- Loading an object by id from an unscoped queryset (`Order.objects.get(pk=pk)`), then relying on the interface to hide the link.
- Checking permissions only in templates (`{% if perms... %}`) and not in the view.
- Using `has_perm(perm, obj)` with the default backend and getting false for everyone, or assuming it returns true for the owner.
- Giving permissions to individual users instead of roles, which makes access impossible to audit.
- Making many users superusers or staff "for convenience".
- Returning 403 for private resources and revealing that they exist.
- Forgetting that permissions are cached on the user instance.
- Protecting the HTML view but not the JSON endpoint, the export, or the admin for the same data.
- Putting the mixin after the view class in the base list, so it never runs.
- Writing rules as scattered `if` statements instead of reusable policy functions.
- Forgetting to scope bulk operations, such as an export that dumps every order.
- Letting a user escalate by editing their own role, groups, or `is_staff` through a profile form (use an explicit field list, note 3.2).

## Practice exercises

1. Add a `Catalog manager` check to a view that adjusts stock, using `@permission_required("bookstore.manage_inventory", raise_exception=True)`, and write tests for a manager, a customer, and an anonymous visitor.
2. Write `orders_visible_to` in a second form as a method on a custom `OrderQuerySet` (`Order.objects.visible_to(user)`), keep the policy functions, and show with a test that both give the same result.
3. Demonstrate an IDOR: write a deliberately vulnerable view that uses `Order.objects.get(pk=pk)`, show with the test client that another user can read the order, then fix it by scoping and show the 404.
4. Add a "Support lead" role that includes the Support permissions plus `bookstore.delete_order`, and write a test proving that plain Support cannot delete while the lead can. Decide whether deleting orders should exist at all, and defend the answer.
5. Write a test that scans the project's URL patterns and fails if a view that is not on an allowlist of public views is reachable by an anonymous user without a redirect or a 403.
6. Install `django-guardian`, allow customers to share a read-only link to one order with a named user, and compare the code and the query count with the scoped-queryset approach for the same feature.
7. Record every refund in an `AuditEntry` model (actor, order, time, previous status) written in the same transaction as the status change. Write tests proving that a denied refund creates no entry and that a failure while saving the entry leaves the order status unchanged.

## Summary

Authorization decides what an authenticated user may do. Django offers staff and superuser flags, model-level permissions (`app.codename`) collected into groups that act as roles, and enforcement through `permission_required`, `PermissionRequiredMixin`, and `UserPassesTestMixin`. Model-level permissions cannot express "only your own orders", so scope querysets to what the user may see before looking up an id, which both prevents IDOR and makes unknown objects return 404. Put decisions in policy functions, enforce them on every path that can trigger the action, and treat template checks as cosmetic. Remember that `has_perm` with an object returns false under the default backend, that permissions are cached per user instance, and that superusers bypass everything. Define roles in code, give each the least privilege it needs, and test the denials as carefully as the grants.
