## Overview

Validation in a real application is not one thing in one place. A coupon code has a format (a form concern), a coupon exists and is currently valid (a business rule), an order may use only one coupon (a data integrity rule), and the coupon's redemption limit must hold even when two customers redeem the last use at the same instant (a concurrency rule). Each rule belongs at a specific layer, and putting it in the wrong layer produces bugs that range from duplicated logic to corrupt data. This note describes the layers, shows what each one is for, and covers `ValidationError` with codes and parameters, cross-field validation, business rules in services, how to return errors from APIs, and how to keep messages consistent. It ends with a coupon redemption feature that uses every layer, with tests at each one and a stable error format shared by the HTML form and the JSON endpoint.

This note extends the `Coupon` model from note 2.11.

## Core concepts

### The layers

Think of validation as defense in depth, from the edge of the system to the data:

1. Client-side hints: HTML attributes and JavaScript give instant feedback. They are convenience only and can be bypassed by anyone.
2. Form (or serializer) validation: checks the shape of the input, converts types, normalizes values, and produces user-friendly messages.
3. Model validation: validators and `clean()` on the model describe rules about a single row, and run when forms or your code call `full_clean()`.
4. Service validation: business rules that depend on other data, state, time, or several models, and apply regardless of which interface triggered the operation.
5. Database constraints: the final, unbypassable guarantee of data integrity and the only layer that is safe under concurrency.

### What each layer is for

- Forms answer: "Is this input well-formed, and what should I tell the user?"
- Models answer: "Is this row internally consistent?"
- Services answer: "Is this operation allowed right now?"
- The database answers: "Can this data ever exist?"

No layer replaces another. A form does not protect against a script that writes through the ORM, and a database constraint cannot tell a user how to fix their input.

### Validators

A validator is a callable that takes a value and raises `ValidationError` if it is unacceptable. Validators can be attached to model fields and form fields (`validators=[...]`). They run only on non-empty values (empty values are handled by `required` and `blank`), and they should be pure: no database access, no network, no randomness.

### ValidationError, codes, and params

A `ValidationError` carries a human-readable message, a stable machine-readable `code`, and optional `params` used to fill placeholders in the message:

```python
raise ValidationError(
    "Ensure the value is at most %(max)d characters.",
    code="too_long",
    params={"max": 32},
)
```

The code is what tests, APIs, and client code should depend on. The message is for people and may change or be translated.

### Three shapes of ValidationError

- A single message: `ValidationError("text", code="x")`.
- A list of errors: `ValidationError([error1, error2])`.
- A dictionary mapping field names to errors: `ValidationError({"field": error})`, which attaches errors to specific fields. The key `"__all__"` (the constant `NON_FIELD_ERRORS`) is for errors not tied to one field.

### Cross-field validation

Rules involving several fields belong in the form's `clean()` or the model's `clean()`. Use `add_error` or a dictionary-shaped `ValidationError` to attach the message to the field the user should fix.

### Business rules belong in services

Rules such as "the coupon is active", "the redemption limit is not reached", or "the order is still unpaid" depend on database state and the current time. They are not properties of the input. Put them in a service function that every entry point calls: the HTML form view, the JSON API, a management command, a background task. That way the rule is written once.

### Domain exceptions versus ValidationError

A service layer is clearer when it raises its own exceptions (`CouponInactiveError`) rather than Django's `ValidationError`, because services should not depend on forms and web concerns. The view layer translates the domain exception into a `ValidationError`, an HTML message, or a JSON error payload.

### Returning validation errors from APIs

APIs need a consistent, machine-readable error format: a status code (400 for malformed requests, 422 for well-formed requests that fail validation), field names, stable codes, and messages. Define one shape and produce it from forms, services, and any other source, so clients handle errors in one way.

### Consistent messages

Messages should be specific about what to do, free of internal details, and written once. Keep them in one place (constants, `error_messages`, or translation files) so the same problem always reads the same way, regardless of the layer that detected it.

## Basic syntax examples

### Validator functions

```python
import re

from django.core.exceptions import ValidationError

COUPON_CODE = re.compile(r"[A-Z0-9]{4,32}")


def validate_coupon_code(value: str) -> None:
    if not COUPON_CODE.fullmatch(value):
        raise ValidationError(
            "Coupon codes must be %(min)d to %(max)d letters or digits.",
            code="invalid_coupon_code",
            params={"min": 4, "max": 32},
        )
```

### Built-in validators

```python
from decimal import Decimal

from django.core.validators import (
    MaxValueValidator,
    MinLengthValidator,
    MinValueValidator,
    RegexValidator,
)

price_validators = [MinValueValidator(Decimal("0"))]
sku_validator = RegexValidator(r"^[A-Z]{3}-\d{4}$", message="Use the format ABC-1234.", code="invalid_sku")
```

### Class-based validators with parameters

```python
from django.utils.deconstruct import deconstructible


@deconstructible
class MaxWordsValidator:
    def __init__(self, limit: int) -> None:
        self.limit = limit

    def __call__(self, value: str) -> None:
        if len(value.split()) > self.limit:
            raise ValidationError(
                "Use at most %(limit)d words.",
                code="too_many_words",
                params={"limit": self.limit},
            )

    def __eq__(self, other: object) -> bool:
        return isinstance(other, MaxWordsValidator) and other.limit == self.limit
```

`@deconstructible` lets migrations serialize the validator when it is used on a model field. The `__eq__` method prevents spurious migrations.

### Attaching validators

```python
from django import forms
from django.db import models


class Coupon(models.Model):
    code = models.CharField(max_length=32, validators=[validate_coupon_code])


class RedeemForm(forms.Form):
    code = forms.CharField(validators=[validate_coupon_code])
```

### The three shapes

```python
raise ValidationError("Something is wrong.", code="wrong")

raise ValidationError(
    [
        ValidationError("First problem.", code="first"),
        ValidationError("Second problem.", code="second"),
    ]
)

raise ValidationError(
    {"end": ValidationError("End must be after start.", code="end_before_start")}
)
```

### Cross-field validation in a form

```python
class DateRangeForm(forms.Form):
    start = forms.DateField()
    end = forms.DateField()

    def clean(self) -> dict[str, Any]:
        cleaned = super().clean()
        start, end = cleaned.get("start"), cleaned.get("end")
        if start and end and end < start:
            self.add_error("end", ValidationError("End must not be before start.", code="end_before_start"))
        return cleaned
```

### Overriding messages by code

```python
class RedeemForm(forms.Form):
    code = forms.CharField(
        error_messages={
            "required": "Enter a coupon code.",
            "max_length": "That code is too long.",
        }
    )
```

### Inspecting errors with their codes

```python
form = RedeemForm({"code": ""})
form.is_valid()
form.errors.as_data()
# {'code': [ValidationError([...'required'...])]}
form.errors.get_json_data()
# {'code': [{'message': 'Enter a coupon code.', 'code': 'required'}]}
```

### Model-level validation

```python
class Coupon(models.Model):
    def clean(self) -> None:
        super().clean()
        if self.valid_from and self.valid_until and self.valid_until <= self.valid_from:
            raise ValidationError({"valid_until": "Must be after the start."})
```

### A database constraint as the last layer

```python
class CouponRedemption(models.Model):
    order = models.OneToOneField("bookstore.Order", on_delete=models.CASCADE)
```

The one-to-one relation carries a unique constraint, so one order can never hold two redemptions, no matter what code runs.

## Deeper behavior

### Why the layers are not redundant

Consider "one coupon per order". The form cannot know whether the order already has one. The service checks first, which gives a good error message in nearly every case. But two requests can pass that check at the same moment, and only the unique constraint can stop the second insert. The service must then translate the resulting `IntegrityError` into the same domain error. Each layer catches a different failure, and the later ones guarantee what the earlier ones can only make likely.

### Validators run only on non-empty values

A blank optional field skips its validators. If a rule must hold for empty values too, make the field required or check it in `clean()`. For form fields, `required=True` is checked before validators, and a missing value stops the chain.

### Model validators and forms

For a `ModelForm`, validators declared on the model field run during model validation (`_post_clean`), after the form field validates. Validators declared on a declared form field run earlier. Choose where to attach a validator by when you need it: on the model if every input path should apply it through `full_clean()`, on the form field if it concerns input shape for that form only.

### Pure validators, impure checks

Validators should not touch the database. A rule such as "this email is not already registered" depends on data, can change between validation and saving, and slows down every validation call. Put it in a form's `clean_<field>()` for the user-friendly message, in the service for the guarantee, and in a unique constraint for the truth.

### Error codes as a contract

Treat codes as part of your interface. Clients and tests compare codes (`coupon_inactive`) while messages can be edited and translated freely. When you change a rule, keep the code if the meaning is the same and introduce a new code if the meaning changes. Avoid codes that expose internal structure.

### Params keep messages translatable

Build messages with `%(name)s` placeholders and `params`, rather than f-strings, so the template can be translated (note 5.6) and the values stay separate for logging and clients. For formatting a message from a `ValidationError`, use `error.messages`, which applies the parameters.

### Information leakage

Specific errors help honest users and also help attackers. "This coupon does not exist" versus "This coupon has expired" tells a brute-force script which codes are real. For public endpoints, consider one generic message for several failure kinds (while keeping the specific code in your logs), and add rate limiting (note 4.4). For authenticated internal tools, specific messages are usually fine.

### HTTP status codes for validation failures

400 Bad Request fits input that cannot be parsed or is structurally wrong. 422 Unprocessable Content fits input that is well-formed but fails validation or business rules. APIs differ, so pick one convention and apply it everywhere. HTML forms re-render with status 200 (note 3.4).

### Shape of an error payload

A common, simple shape maps each field to a list of objects with a stable code and a message, and uses a reserved key for errors not tied to a field:

```text
{"errors": {"code": [{"code": "coupon_inactive", "message": "This coupon is not active."}],
            "__all__": [{"code": "order_not_payable", "message": "This order can no longer be changed."}]}}
```

Django's `form.errors.get_json_data()` already produces a very similar structure, and Phase 7 shows how DRF and RFC 9457 problem details fit the same idea.

### Domain exceptions carry the information the view needs

A domain exception should carry a stable `code`, a user-safe `message`, and optionally the name of the input it relates to. That is enough for any adapter to produce an HTML message, a field error, or a JSON entry, without the service knowing which interface is calling it.

### Validation and concurrency

Validation answers "is this acceptable now?". It cannot promise it stays acceptable by the time the data is written. Whenever a rule depends on other rows (limits, uniqueness, balances), pair the check with a lock or a constraint (note 2.9) and handle the failure path explicitly.

### Where to test

Test each layer at its own level: validators and forms as fast unit tests without a database, services against a database including race outcomes, constraints by attempting forbidden writes directly, and views for status codes and payload shape. A rule that is tested only through the UI is easy to break.

## Industry-level snippet

Coupon redemption with every layer in place. The format is validated by a pure validator in the form field, which also normalizes the input. The service enforces business rules (the order is open, the coupon exists and is active, the redemption limit has not been reached), serializes access with row locks, and converts the database's uniqueness error into a domain error. An adapter converts domain errors into `ValidationError` objects, and one function renders any mapping of errors into the same JSON shape, used by both the form and the service paths in the JSON endpoint.

Add to `apps/bookstore/models.py`:

```python
# in Coupon
max_redemptions = models.PositiveIntegerField(
    null=True, blank=True, help_text="Leave empty for unlimited use."
)


class CouponRedemption(models.Model):
    coupon = models.ForeignKey(Coupon, on_delete=models.PROTECT, related_name="redemptions")
    order = models.OneToOneField(Order, on_delete=models.CASCADE, related_name="redemption")
    redeemed_at = models.DateTimeField(auto_now_add=True)

    def __str__(self) -> str:
        return f"{self.coupon_id} on order {self.order_id}"
```

`apps/bookstore/validators.py`:

```python
"""Pure validators: no database access, no side effects."""

import re

from django.core.exceptions import ValidationError

COUPON_CODE_MIN = 4
COUPON_CODE_MAX = 32
_COUPON_CODE = re.compile(rf"[A-Z0-9]{{{COUPON_CODE_MIN},{COUPON_CODE_MAX}}}")


def validate_coupon_code(value: str) -> None:
    if not _COUPON_CODE.fullmatch(value):
        raise ValidationError(
            "Coupon codes must be %(min)d to %(max)d letters or digits.",
            code="invalid_coupon_code",
            params={"min": COUPON_CODE_MIN, "max": COUPON_CODE_MAX},
        )
```

`apps/bookstore/services/coupons.py`:

```python
"""Coupon redemption business rules."""

import logging
from datetime import datetime

from django.db import IntegrityError, transaction
from django.utils import timezone

from apps.bookstore.models import Coupon, CouponRedemption, Order

logger = logging.getLogger(__name__)


class CouponError(Exception):
    """Base class for rule violations. Carries what any interface needs."""

    code: str = "coupon_error"
    field: str | None = None  # the input the error belongs to; None for the whole request

    def __init__(self, message: str) -> None:
        super().__init__(message)
        self.message = message


class CouponNotFoundError(CouponError):
    code = "coupon_not_found"
    field = "code"


class CouponInactiveError(CouponError):
    code = "coupon_inactive"
    field = "code"


class CouponExhaustedError(CouponError):
    code = "coupon_exhausted"
    field = "code"


class OrderNotPayableError(CouponError):
    code = "order_not_payable"


class OrderAlreadyDiscountedError(CouponError):
    code = "order_already_discounted"


def redeem_coupon(
    *, order_id: int, code: str, now: datetime | None = None
) -> CouponRedemption:
    """Apply the coupon ``code`` to an order.

    Raises:
        Order.DoesNotExist: if the order does not exist.
        CouponError: (a subclass) if a business rule is violated.
    """
    normalized = code.strip().upper()
    moment = now or timezone.now()

    with transaction.atomic():
        # Lock the coupon first and the order second, always in that order.
        try:
            coupon = Coupon.objects.select_for_update().get(code=normalized)
        except Coupon.DoesNotExist as exc:
            raise CouponNotFoundError("This coupon does not exist.") from exc
        order = Order.objects.select_for_update().get(pk=order_id)

        if order.status != Order.Status.PENDING:
            raise OrderNotPayableError("This order can no longer be changed.")
        if not coupon.is_active(moment):
            raise CouponInactiveError("This coupon is not active.")
        if (
            coupon.max_redemptions is not None
            and coupon.redemptions.count() >= coupon.max_redemptions
        ):
            raise CouponExhaustedError("This coupon has been fully redeemed.")

        try:
            with transaction.atomic():  # savepoint for the constraint check
                redemption = CouponRedemption.objects.create(coupon=coupon, order=order)
        except IntegrityError as exc:
            raise OrderAlreadyDiscountedError(
                "A coupon has already been applied to this order."
            ) from exc

    logger.info("Redeemed coupon %s on order %s", coupon.pk, order_id)
    return redemption
```

`apps/bookstore/errors.py`:

```python
"""Translate errors from any layer into one consistent shape."""

from collections.abc import Mapping, Sequence

from django.core.exceptions import NON_FIELD_ERRORS, ValidationError

from apps.bookstore.services.coupons import CouponError


def to_validation_error(exc: CouponError) -> ValidationError:
    """Convert a domain error to a ``ValidationError`` for the right field."""
    error = ValidationError(exc.message, code=exc.code)
    if exc.field is None:
        return error
    return ValidationError({exc.field: error})


def validation_error_to_mapping(error: ValidationError) -> dict[str, list[ValidationError]]:
    """Normalize any ``ValidationError`` into ``{field: [errors]}``."""
    if hasattr(error, "error_dict"):
        return {field: list(items) for field, items in error.error_dict.items()}
    return {NON_FIELD_ERRORS: list(error.error_list)}


def errors_to_payload(
    errors: Mapping[str, Sequence[ValidationError]],
) -> dict[str, list[dict[str, str]]]:
    """Render errors as ``{field: [{"code": ..., "message": ...}]}``."""
    return {
        field: [
            {"code": error.code or "invalid", "message": error.messages[0]}
            for error in items
        ]
        for field, items in errors.items()
    }
```

`apps/bookstore/forms.py` (addition):

```python
from apps.bookstore.validators import validate_coupon_code


class CouponCodeField(forms.CharField):
    """A text field that normalizes coupon codes before validating them."""

    def to_python(self, value: Any) -> str:
        return super().to_python(value).strip().upper()


class RedeemCouponForm(forms.Form):
    code = CouponCodeField(
        validators=[validate_coupon_code],
        error_messages={"required": "Enter a coupon code."},
    )
```

`apps/bookstore/views.py` (addition):

```python
from django.http import JsonResponse
from django.views.decorators.http import require_POST

from apps.bookstore.errors import (
    errors_to_payload,
    to_validation_error,
    validation_error_to_mapping,
)
from apps.bookstore.forms import RedeemCouponForm
from apps.bookstore.models import Order
from apps.bookstore.services.coupons import CouponError, redeem_coupon


@require_POST
def redeem_coupon_view(request: HttpRequest, order_id: int) -> JsonResponse:
    order = get_object_or_404(Order, pk=order_id)

    form = RedeemCouponForm(request.POST)
    if not form.is_valid():  # layer 2: input shape
        return JsonResponse({"errors": errors_to_payload(form.errors.as_data())}, status=422)

    try:
        redemption = redeem_coupon(order_id=order.pk, code=form.cleaned_data["code"])  # layers 4 and 5
    except CouponError as exc:
        mapping = validation_error_to_mapping(to_validation_error(exc))
        return JsonResponse({"errors": errors_to_payload(mapping)}, status=422)

    return JsonResponse(
        {
            "order": order.pk,
            "coupon": redemption.coupon.code,
            "percent_off": redemption.coupon.percent_off,
        },
        status=201,
    )
```

Add `path("orders/<int:order_id>/coupon/", views.redeem_coupon_view, name="redeem_coupon")` to the app's `urlpatterns`.

`apps/bookstore/tests/test_validation_layers.py`:

```python
from datetime import timedelta

from django.core.exceptions import ValidationError
from django.db import IntegrityError, transaction
from django.test import SimpleTestCase, TestCase
from django.urls import reverse
from django.utils import timezone

from apps.bookstore.errors import (
    errors_to_payload,
    to_validation_error,
    validation_error_to_mapping,
)
from apps.bookstore.forms import RedeemCouponForm
from apps.bookstore.models import Coupon, CouponRedemption, Order
from apps.bookstore.services.coupons import (
    CouponExhaustedError,
    CouponInactiveError,
    CouponNotFoundError,
    OrderAlreadyDiscountedError,
    OrderNotPayableError,
    redeem_coupon,
)
from apps.bookstore.services.orders import OrderLine, create_order
from apps.bookstore.validators import validate_coupon_code

from .factories import make_book


class ValidatorTests(SimpleTestCase):
    def test_valid_codes_pass(self) -> None:
        for code in ("SPRING20", "ABCD", "A1B2C3D4"):
            validate_coupon_code(code)

    def test_invalid_codes_report_a_stable_code_and_a_formatted_message(self) -> None:
        for code in ("", "ABC", "spring20", "SPRING-20", "A" * 33):
            with self.subTest(code=code):
                with self.assertRaises(ValidationError) as context:
                    validate_coupon_code(code)
                self.assertEqual(context.exception.code, "invalid_coupon_code")
                self.assertEqual(
                    context.exception.messages,
                    ["Coupon codes must be 4 to 32 letters or digits."],
                )


class FormTests(SimpleTestCase):
    def test_input_is_normalized_before_validation(self) -> None:
        form = RedeemCouponForm({"code": "  spring20 "})
        self.assertTrue(form.is_valid(), form.errors)
        self.assertEqual(form.cleaned_data["code"], "SPRING20")

    def test_bad_format_and_missing_value_are_reported_with_codes(self) -> None:
        bad = RedeemCouponForm({"code": "no!"})
        self.assertFalse(bad.is_valid())
        self.assertEqual(bad.errors.as_data()["code"][0].code, "invalid_coupon_code")
        missing = RedeemCouponForm({})
        self.assertFalse(missing.is_valid())
        self.assertEqual(missing.errors.as_data()["code"][0].code, "required")


class ServiceTestCase(TestCase):
    def make_coupon(self, **overrides: object) -> Coupon:
        now = timezone.now()
        values: dict[str, object] = {
            "code": "SPRING20",
            "percent_off": 20,
            "valid_from": now - timedelta(days=1),
            "valid_until": now + timedelta(days=7),
        }
        values.update(overrides)
        return Coupon.objects.create(**values)

    def make_order(self) -> Order:
        book = make_book(stock=5)
        return create_order(customer_email="a@example.com", lines=[OrderLine(book.pk, 1)])


class RedeemServiceTests(ServiceTestCase):
    def test_redeeming_creates_one_redemption(self) -> None:
        coupon = self.make_coupon()
        order = self.make_order()
        redemption = redeem_coupon(order_id=order.pk, code=" spring20 ")
        self.assertEqual((redemption.coupon, redemption.order), (coupon, order))

    def test_unknown_coupon(self) -> None:
        order = self.make_order()
        with self.assertRaises(CouponNotFoundError):
            redeem_coupon(order_id=order.pk, code="NOSUCH1")

    def test_coupon_outside_its_window_is_inactive(self) -> None:
        coupon = self.make_coupon()
        order = self.make_order()
        for moment in (coupon.valid_from - timedelta(seconds=1), coupon.valid_until):
            with self.subTest(moment=moment):
                with self.assertRaises(CouponInactiveError):
                    redeem_coupon(order_id=order.pk, code="SPRING20", now=moment)

    def test_redemption_limit_is_enforced(self) -> None:
        self.make_coupon(max_redemptions=1)
        redeem_coupon(order_id=self.make_order().pk, code="SPRING20")
        with self.assertRaises(CouponExhaustedError):
            redeem_coupon(order_id=self.make_order().pk, code="SPRING20")

    def test_only_pending_orders_can_be_discounted(self) -> None:
        self.make_coupon()
        order = self.make_order()
        Order.objects.filter(pk=order.pk).update(status=Order.Status.PAID)
        with self.assertRaises(OrderNotPayableError):
            redeem_coupon(order_id=order.pk, code="SPRING20")

    def test_an_order_cannot_be_discounted_twice(self) -> None:
        self.make_coupon()
        order = self.make_order()
        redeem_coupon(order_id=order.pk, code="SPRING20")
        with self.assertRaises(OrderAlreadyDiscountedError):
            redeem_coupon(order_id=order.pk, code="SPRING20")

    def test_unknown_order_is_not_a_coupon_error(self) -> None:
        self.make_coupon()
        with self.assertRaises(Order.DoesNotExist):
            redeem_coupon(order_id=999_999, code="SPRING20")


class DatabaseLayerTests(ServiceTestCase):
    def test_the_constraint_holds_even_when_the_service_is_bypassed(self) -> None:
        coupon = self.make_coupon()
        order = self.make_order()
        CouponRedemption.objects.create(coupon=coupon, order=order)
        with self.assertRaises(IntegrityError), transaction.atomic():
            CouponRedemption.objects.create(coupon=coupon, order=order)


class ErrorTranslationTests(SimpleTestCase):
    def test_field_errors_and_request_errors_land_in_different_keys(self) -> None:
        inactive = validation_error_to_mapping(to_validation_error(CouponInactiveError("Not active.")))
        closed = validation_error_to_mapping(to_validation_error(OrderNotPayableError("Closed.")))
        self.assertEqual(list(inactive), ["code"])
        self.assertEqual(list(closed), ["__all__"])

    def test_payload_has_stable_codes_and_formatted_messages(self) -> None:
        try:
            validate_coupon_code("x")
        except ValidationError as error:
            payload = errors_to_payload({"code": [error]})
        self.assertEqual(
            payload,
            {
                "code": [
                    {
                        "code": "invalid_coupon_code",
                        "message": "Coupon codes must be 4 to 32 letters or digits.",
                    }
                ]
            },
        )


class RedeemViewTests(ServiceTestCase):
    def post(self, order: Order, code: str) -> "HttpResponse":
        url = reverse("bookstore:redeem_coupon", kwargs={"order_id": order.pk})
        return self.client.post(url, {"code": code})

    def test_success_returns_201(self) -> None:
        self.make_coupon()
        response = self.post(self.make_order(), "spring20")
        self.assertEqual(response.status_code, 201)
        self.assertEqual(response.json()["percent_off"], 20)

    def test_bad_format_is_a_422_from_the_form_layer(self) -> None:
        response = self.post(self.make_order(), "no!")
        self.assertEqual(response.status_code, 422)
        self.assertEqual(response.json()["errors"]["code"][0]["code"], "invalid_coupon_code")

    def test_rule_violations_use_the_same_payload_shape(self) -> None:
        order = self.make_order()
        response = self.post(order, "NOSUCH1")
        self.assertEqual(response.status_code, 422)
        self.assertEqual(response.json()["errors"]["code"][0]["code"], "coupon_not_found")

        self.make_coupon()
        Order.objects.filter(pk=order.pk).update(status=Order.Status.CANCELLED)
        closed = self.post(order, "SPRING20")
        self.assertEqual(closed.json()["errors"]["__all__"][0]["code"], "order_not_payable")

    def test_unknown_order_is_404_and_get_is_405(self) -> None:
        missing = reverse("bookstore:redeem_coupon", kwargs={"order_id": 999_999})
        self.assertEqual(self.client.post(missing, {"code": "SPRING20"}).status_code, 404)
        existing = reverse("bookstore:redeem_coupon", kwargs={"order_id": self.make_order().pk})
        self.assertEqual(self.client.get(existing).status_code, 405)
```

In the test module, add `from django.http import HttpResponse` and annotate `post` as `-> HttpResponse` without the quotes.

Notice how the codes `invalid_coupon_code`, `coupon_not_found`, and `order_not_payable` appear in the same JSON structure whether the problem was detected by a form, a service rule, or an adapter. Clients branch on codes and display messages. The database constraint test proves the last layer works even when everything above it is skipped.

## Common mistakes

- Putting every rule in the form, so the JSON API, the admin, and scripts skip it.
- Putting database-dependent checks in validators, making them slow, impure, and untestable without a database.
- Relying on a check without a constraint for rules that must hold under concurrency.
- Catching the `IntegrityError` and returning a generic 500 or a vague message instead of mapping it to the same domain error as the explicit check.
- Comparing error messages instead of error codes in tests and clients.
- Building messages with f-strings, which prevents translation and loses the parameters.
- Returning different error shapes from different endpoints, forcing clients to special-case each.
- Raising Django's `ValidationError` from deep inside a service, coupling business logic to form machinery.
- Revealing too much in public error messages, such as which coupon codes exist.
- Letting validators have side effects or randomness.
- Writing a validator that crashes on non-string input instead of raising `ValidationError`.
- Testing only through the UI so that individual layers can silently regress.

## Practice exercises

1. Write a `validate_future_date` validator that accepts a clock argument for testing, with a stable code and a message with a `%(date)s` parameter. Test it as a plain function, as a form field validator, and on a model field.
2. Write a reusable `MaxWordsValidator` (as in this note) and show that using it on a model field does not generate a new migration on every `makemigrations` run. Then remove `__eq__` and observe what changes.
3. Add a rule "an order can have a coupon only if its total is at least 20" to `redeem_coupon`. Decide which layer owns it, add a domain exception with its own code, and test it.
4. Make `redeem_coupon` safe when two requests redeem the last use of a single-use coupon on different orders at the same time. Write a PostgreSQL-only threaded test that proves exactly one succeeds.
5. Add a second adapter that turns any `CouponError` into a message for the Django messages framework (for an HTML flow), reusing the same codes and messages. Write a test that the HTML and JSON paths report the same code.
6. Rewrite `errors_to_payload` so that it can also produce RFC 9457 problem-details documents (`type`, `title`, `status`, `errors`), keeping the current function as the default. Write tests for both outputs.
7. Choose three rules from your own project and write a table in prose that says, for each rule, which layer enforces it, which layer only makes it user-friendly, and what happens if the layer above it is bypassed.

## Summary

Validation is layered: client hints give convenience, forms check the shape of input and talk to the user, models describe single-row consistency, services enforce business rules for every entry point, and database constraints guarantee integrity under concurrency. No layer replaces another, and each later layer guarantees what the earlier ones can only make likely. Use pure validators, raise `ValidationError` with stable codes and parameterized messages, and keep database-dependent rules in services and constraints. Let services raise domain exceptions and translate them at the edge into form errors, messages, or JSON. Define one error shape for the whole application, test each layer on its own, and always pair a business check with a lock or constraint when concurrency can break it.
