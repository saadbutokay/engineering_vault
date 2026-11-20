## Overview

An application that serves people in more than one country meets three problems that look unrelated and all come down to the same discipline: keep the data neutral and adapt at the edge. Time is stored as an exact instant and shown in the viewer's time zone. Text is written in one language in the code and translated when it is shown. Numbers, dates, and currencies are stored as values and formatted according to the reader's conventions. Django supports all three: aware datetimes and time zones, the translation system (internationalization, or i18n), and locale-aware formatting (localization, or l10n). Most bugs in this area come from deciding too late (strings concatenated in code, naive datetimes already in the database) so this note stresses the habits to adopt from the first commit. It covers `USE_TZ` and aware datetimes, storing UTC and converting at the edge, `timezone.now()`, translation with `gettext` and `gettext_lazy`, `makemessages` and `compilemessages`, `LocaleMiddleware`, formats, pluralization, and right-to-left languages. It ends with a tested time zone toolkit (validated per-user time zones, a middleware, and a function that refuses non-existent and ambiguous local times) and a pluralized, translatable label.

## Core concepts

### Aware and naive datetimes

A naive datetime has no time zone: `datetime(2026, 3, 8, 9, 30)` could mean any place on Earth. An aware datetime carries its zone and denotes an exact instant. Mixing the two raises errors in comparisons, and treating naive values as if they were in the right zone is a source of wrong data that is almost impossible to repair afterward.

### USE_TZ

With `USE_TZ = True` (the default for new projects), Django stores datetimes as UTC in the database, returns aware datetimes from the ORM, and converts to the current time zone when rendering templates and parsing forms. Keep it on and never turn it off in a project with real data.

### Store UTC, convert at the edge

UTC is the neutral storage format: it has no daylight saving jumps, and instants compare and sort correctly. Convert to a local zone only when displaying to a person, or when interpreting a local time a person typed. The conversion belongs in templates, forms, and the presentation layer, not in your business logic.

### timezone.now() and friends

- `django.utils.timezone.now()`: the current instant as an aware UTC datetime (use it instead of `datetime.now()`).
- `timezone.localtime(value)`: convert an aware datetime to the current time zone.
- `timezone.localdate()`: today's date in the current time zone.
- `timezone.make_aware(naive, tz)`: attach a zone to a naive datetime.
- `timezone.activate(tz)` and `timezone.deactivate()`: set and clear the current time zone for the running thread or task.

### The default and the current time zone

`TIME_ZONE` is the default time zone: the zone for naive values and for display when no zone is activated. Keep it `"UTC"` in most projects. The current time zone is the one activated for this request, typically from the user's preference, and is what templates and forms use.

### Translation basics

Write user-visible text in one language in the code, wrap it in a translation function, and supply catalogs for the other languages. The functions:

- `gettext("text")`: translate immediately, using the active language.
- `gettext_lazy("text")`: return a lazy object that translates when it is converted to a string. Use it at import time: model field labels, form labels, class attributes, settings.
- `ngettext("singular", "plural", n)`: choose the right plural form for the number.
- `pgettext("context", "text")`: disambiguate identical words that translate differently ("Order" as a noun versus a verb).

### Translation in templates

`{% load i18n %}` then `{% translate "Text" %}` for a short string and `{% blocktranslate %}...{% endblocktranslate %}` for text with variables and plurals.

### The translation workflow

1. Mark strings in code and templates.
2. `django-admin makemessages -l es` extracts them into `locale/es/LC_MESSAGES/django.po`.
3. A translator fills in the `msgstr` entries.
4. `django-admin compilemessages` produces the binary `django.mo` files that Django reads. It needs GNU gettext installed.

Re-run `makemessages` when strings change, and the tooling merges new strings and marks changed ones as "fuzzy".

### Choosing the language

`LocaleMiddleware` selects the language for each request, in this order: a language prefix in the URL (with `i18n_patterns`), the language cookie (`django_language`), the `Accept-Language` header sent by the browser, and finally `LANGUAGE_CODE`. It must come after `SessionMiddleware` and before `CommonMiddleware`. `LANGUAGES` lists the languages you support.

### Localized formats

With localization active (it is always on since Django 5.0), dates, times, and numbers render in the active locale's format: `15/01/2026` or `01/15/2026`, `1.234,56` or `1,234.56`. Templates and forms use these formats automatically. The `localize` and `unlocalize` filters and the `{% localize %}` tag control it explicitly.

### Plural forms

English has two forms (one, other), but many languages have more: Arabic has six, Polish and Russian have three or four. That is why you must never build plurals by adding an "s" or by branching on `n == 1` yourself. `ngettext` and the `.po` plural rules handle the differences.

### Right-to-left languages

Arabic, Hebrew, Persian, and Urdu are written right to left. Django exposes `LANGUAGE_BIDI` in templates, and `get_language_bidi()` in code. Set `dir="rtl"` on the root element and use CSS logical properties (`margin-inline-start`) so layouts mirror correctly.

## Basic syntax examples

### Time zone settings

```python
USE_TZ = True
TIME_ZONE = "UTC"
```

### Working with aware datetimes

```python
from datetime import UTC, datetime, timedelta
from zoneinfo import ZoneInfo

from django.utils import timezone

now = timezone.now()                                  # aware, UTC
deadline = now + timedelta(days=7)

dhaka = ZoneInfo("Asia/Dhaka")
local = now.astimezone(dhaka)                         # same instant, shown in Dhaka
naive = datetime(2026, 1, 15, 9, 30)
aware = timezone.make_aware(naive, dhaka)             # interpret as 09:30 in Dhaka

timezone.localtime(now)                               # in the current time zone
timezone.localdate()                                  # today's date there
```

### Activating a user's time zone

```python
from zoneinfo import ZoneInfo

from django.utils import timezone

timezone.activate(ZoneInfo("Asia/Dhaka"))
try:
    ...
finally:
    timezone.deactivate()
```

### Time zones in templates

```html
{% load tz %}

{{ order.created_at }}                         {# converted to the current time zone #}
{{ order.created_at|utc }}                     {# forced to UTC #}
{{ order.created_at|timezone:"Asia/Dhaka" }}   {# forced to a zone #}

{% timezone "America/New_York" %}
  {{ order.created_at }}
{% endtimezone %}

{% localtime off %}{{ order.created_at }}{% endlocaltime %}
```

### Marking text for translation

```python
from django.utils.translation import gettext, gettext_lazy, ngettext, pgettext_lazy

# at request time
message = gettext("Your order was placed.")

# at import time: lazy
class Book(models.Model):
    title = models.CharField(gettext_lazy("title"), max_length=200)

    class Meta:
        verbose_name = gettext_lazy("book")
        verbose_name_plural = gettext_lazy("books")

# plurals and context
label = ngettext("%(count)d book", "%(count)d books", count) % {"count": count}
verb = pgettext_lazy("action", "Order")
```

### Named placeholders, never f-strings

```python
# wrong: the sentence cannot be reordered or translated as a whole
gettext("Hello, ") + name + gettext("!")

# right: translators may move the placeholder
gettext("Hello, %(name)s!") % {"name": name}
```

### Translation in templates

```html
{% load i18n %}

<h1>{% translate "Your orders" %}</h1>

{% blocktranslate count counter=orders|length %}
  You have {{ counter }} order.
{% plural %}
  You have {{ counter }} orders.
{% endblocktranslate %}

{% blocktranslate with name=user.get_short_name %}Welcome back, {{ name }}.{% endblocktranslate %}
```

### The workflow

```bash
mkdir locale
uv run django-admin makemessages -l es -l ar
# edit locale/es/LC_MESSAGES/django.po
uv run django-admin compilemessages
```

### An entry in a .po file

```text
#: apps/bookstore/labels.py:14
#, python-format
msgid "%(count)d copy in stock"
msgid_plural "%(count)d copies in stock"
msgstr[0] "%(count)d ejemplar disponible"
msgstr[1] "%(count)d ejemplares disponibles"
```

### Settings for languages

```python
from django.utils.translation import gettext_lazy as _

LANGUAGE_CODE = "en"
LANGUAGES = [
    ("en", _("English")),
    ("es", _("Spanish")),
    ("ar", _("Arabic")),
]
LOCALE_PATHS = [BASE_DIR / "locale"]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.locale.LocaleMiddleware",      # after sessions, before common
    "django.middleware.common.CommonMiddleware",
    # ...
]
```

### Language-prefixed URLs

```python
from django.conf.urls.i18n import i18n_patterns
from django.urls import include, path

urlpatterns = [
    path("i18n/", include("django.conf.urls.i18n")),     # provides the set_language view
]
urlpatterns += i18n_patterns(
    path("books/", include("apps.bookstore.urls")),
    prefix_default_language=False,
)
```

### Right-to-left layout

```html
{% load i18n %}
<html lang="{{ LANGUAGE_CODE }}" dir="{% if LANGUAGE_BIDI %}rtl{% else %}ltr{% endif %}">
```

### Formatting values

```python
from django.utils import formats

formats.number_format(1234567.891, decimal_pos=2, use_l10n=True)
formats.date_format(timezone.localdate(), "SHORT_DATE_FORMAT")
```

## Deeper behavior

### What USE_TZ does at the database

PostgreSQL stores `timestamp with time zone` values in UTC and converts on the way in and out. SQLite stores naive UTC text, and Django converts. Either way, your Python code sees aware datetimes. With `USE_TZ = False`, Django stores and returns naive datetimes in `TIME_ZONE`, which breaks the moment servers, users, or daylight saving time differ.

### The naive datetime warning

When Django receives a naive datetime for a `DateTimeField` with `USE_TZ = True`, it warns (`RuntimeWarning: DateTimeField received a naive datetime`) and interprets it in the default time zone. Treat that warning as a bug: find where the naive value came from (`datetime.now()`, `datetime.utcnow()`, or a parsed string without an offset) and fix it at the source. Use `timezone.now()`.

### Dates are not instants

A `DateField` has no time zone. "The book was published on 1968-03-01" is the same date everywhere. A `DateTimeField` is an instant. When a rule says "today", decide whose today it is: the customer's, the store's, or UTC's, and use `timezone.localdate()` with the matching zone activated.

### Daylight saving time pitfalls

Twice a year, local clocks jump. In the spring a local time such as 02:30 does not exist, and in the autumn 01:30 occurs twice. Python's `zoneinfo` resolves both silently: a nonexistent time is interpreted using the offset before the jump, and an ambiguous time uses the `fold` attribute (0 for the first occurrence, 1 for the second). If a user types a local time, such as the time of an appointment, you must decide what to do in those two cases. The helper below detects them and lets you choose to reject or pick an occurrence.

### Recurring local events

"Every day at 09:00 in Berlin" is not "09:00 UTC every day". Store local wall-clock rules (time, zone name, recurrence) and compute the instants when needed, because the UTC time of 09:00 in Berlin changes with daylight saving. Store the zone name (`"Europe/Berlin"`), not a fixed offset like `+01:00`.

### The zoneinfo database

Python's `zoneinfo` reads the IANA time zone database from the operating system, or from the `tzdata` package where the system has none (Windows, minimal containers). Add `tzdata` as a dependency for those cases. Zone names are validated against `zoneinfo.available_timezones()`. Never pass user input to `ZoneInfo` without validation: odd values raise `ValueError`, `ZoneInfoNotFoundError`, or even file system errors.

### Per-request activation

A time zone is activated for the current thread or async task, so it must be reset at the end of the request, or the next request handled by the same thread inherits it. A middleware with a `try`/`finally` is the standard way, reading the zone from a cookie or the user profile. The default `TIME_ZONE` applies when nothing is activated.

### Lazy versus immediate translation

`gettext` translates when called, using the language active at that moment. At import time no request exists, so the active language is the default, and the text would be stuck in that language forever. Anything defined at import time (model fields, form fields, class attributes, module constants) must use `gettext_lazy`. A lazy string translates every time it is converted to text. Be careful that lazy strings are not plain strings: `len()`, concatenation, and JSON encoding need `str()` first.

### Writing translatable strings

- Use whole sentences, not fragments joined in code. Word order differs between languages.
- Use named placeholders (`%(name)s`) so translators can reorder them.
- Never put translated text in a format string with user data through an f-string.
- Add context with `pgettext` when a short word is ambiguous.
- Add translator comments with `# Translators:` directly above the marked string.
- Do not translate identifiers, log messages, or exception messages meant for developers.

### Plural handling

`ngettext` takes the number and returns the proper form. The `.po` file's header declares the plural rule for each language, and the entries have one `msgstr[n]` per form. Do not guess forms in code. Test plurals for at least one language with more than two forms.

### The language of a request

The language is available as `request.LANGUAGE_CODE` and through `translation.get_language()`. Background tasks and management commands have no request, so activate the language explicitly (for example the user's preferred language when sending an email) with `translation.override(code)`.

### Translating data in the database

`gettext` translates the text in your code, not rows in your tables. Book descriptions that must exist in several languages need a model design for it: a translations table, or a package such as `django-modeltranslation` or `django-parler`. Decide early, because retrofitting it is hard.

### Formatting and localization details

Formats come from the active locale (`django/conf/locale/<lang>/formats.py`) and can be overridden with `FORMAT_MODULE_PATH`. Template output is localized by default, which surprises people when `{{ number }}` shows `1.234,5`. Use `|unlocalize` for values destined for JavaScript, HTML attributes, or APIs, and use the `json_script` filter for data handed to scripts. Sorting text alphabetically also differs by language (database collation matters), and so do name and address formats, currencies, and calendars.

### Testing

Use `translation.override("es")` as a context manager or decorator, and assert on translations that exist in Django's own catalogs when you do not have yours yet (for example, the word "Yes"). Test plural forms, lazy strings, and the middleware's language choice with `Accept-Language` headers. Tests must run with `compilemessages` already done, because Django reads `.mo` files.

## Industry-level snippet

A time zone toolkit: validated zone names, a middleware that activates the viewer's zone from a cookie and always resets it, a view that stores the preference safely, a formatter, and a function that converts a local wall-clock time to an aware datetime while rejecting nonexistent times and handling ambiguous ones explicitly. A pluralized, translatable stock label rounds it out.

`apps/core/timezones.py`:

```python
"""Time zone helpers: validation, per-request activation, and safe localization."""

import zoneinfo
from collections.abc import Callable
from datetime import UTC, datetime, tzinfo
from functools import cache
from typing import Literal

from django.conf import settings
from django.http import HttpRequest, HttpResponse, JsonResponse
from django.utils import timezone
from django.views.decorators.http import require_POST

TIMEZONE_COOKIE = "tz"
TIMEZONE_COOKIE_MAX_AGE = 365 * 24 * 3600

Ambiguity = Literal["earlier", "later", "error"]
GetResponse = Callable[[HttpRequest], HttpResponse]


@cache
def _known_zones() -> frozenset[str]:
    return frozenset(zoneinfo.available_timezones())


def is_valid_timezone(name: str) -> bool:
    """True only for real IANA zone names. Never builds a path from user input."""
    return name in _known_zones()


def get_zone(name: str) -> zoneinfo.ZoneInfo:
    if not is_valid_timezone(name):
        raise ValueError(f"Unknown time zone: {name!r}")
    return zoneinfo.ZoneInfo(name)


def localize(
    naive: datetime, zone: zoneinfo.ZoneInfo, *, ambiguous: Ambiguity = "error"
) -> datetime:
    """Attach ``zone`` to a wall-clock time, refusing nonexistent times.

    A time that falls in the spring-forward gap raises ``ValueError``. A time that
    occurs twice in the autumn is resolved by ``ambiguous``: ``"earlier"`` (the
    first occurrence), ``"later"``, or ``"error"``.
    """
    if naive.tzinfo is not None:
        raise ValueError("A naive datetime is required")
    first = naive.replace(tzinfo=zone, fold=0)
    second = naive.replace(tzinfo=zone, fold=1)

    round_trip = first.astimezone(UTC).astimezone(zone).replace(tzinfo=None)
    if round_trip != naive:
        raise ValueError(f"{naive.isoformat()} does not exist in {zone.key}")

    if first.utcoffset() != second.utcoffset():
        if ambiguous == "error":
            raise ValueError(f"{naive.isoformat()} is ambiguous in {zone.key}")
        return first if ambiguous == "earlier" else second
    return first


def format_local(
    moment: datetime,
    zone: tzinfo | None = None,
    *,
    pattern: str = "%Y-%m-%d %H:%M",
) -> str:
    """Format an aware datetime in ``zone`` or in the current time zone."""
    if timezone.is_naive(moment):
        raise ValueError("An aware datetime is required")
    target = zone if zone is not None else timezone.get_current_timezone()
    return moment.astimezone(target).strftime(pattern)


class TimezoneMiddleware:
    """Activate the viewer's time zone for the duration of the request."""

    def __init__(self, get_response: GetResponse) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        name = request.COOKIES.get(TIMEZONE_COOKIE, "")
        if name and is_valid_timezone(name):
            timezone.activate(zoneinfo.ZoneInfo(name))
        else:
            timezone.deactivate()
        try:
            return self.get_response(request)
        finally:
            timezone.deactivate()  # never leak into the next request on this thread


@require_POST
def set_timezone(request: HttpRequest) -> HttpResponse:
    name = request.POST.get("timezone", "")
    if not is_valid_timezone(name):
        return JsonResponse({"error": "Unknown time zone."}, status=400)
    response = JsonResponse({"timezone": name})
    response.set_cookie(
        TIMEZONE_COOKIE,
        name,
        max_age=TIMEZONE_COOKIE_MAX_AGE,
        secure=not settings.DEBUG,
        samesite="Lax",
    )
    return response
```

Browsers know their own zone. A small script posts it once:

```html
<script>
  const zone = Intl.DateTimeFormat().resolvedOptions().timeZone;
  if (zone && document.cookie.indexOf("tz=") === -1) {
    fetch("/timezone/", {
      method: "POST",
      headers: {"X-CSRFToken": "{{ csrf_token }}", "Content-Type": "application/x-www-form-urlencoded"},
      body: "timezone=" + encodeURIComponent(zone),
    });
  }
</script>
```

`apps/bookstore/labels.py`:

```python
from django.utils.translation import gettext, ngettext


def stock_label(count: int) -> str:
    """A translated, correctly pluralized availability label."""
    if count < 0:
        raise ValueError("count must not be negative")
    if count == 0:
        return gettext("Sold out")
    return ngettext("%(count)d copy in stock", "%(count)d copies in stock", count) % {
        "count": count
    }
```

`apps/core/tests/test_timezones.py`:

```python
import zoneinfo
from datetime import UTC, datetime, timedelta

from django.conf import settings
from django.http import HttpRequest, HttpResponse
from django.test import RequestFactory, SimpleTestCase
from django.utils import timezone
from django.utils.translation import gettext_lazy, override

from apps.bookstore.labels import stock_label
from apps.core.timezones import (
    TIMEZONE_COOKIE,
    TimezoneMiddleware,
    format_local,
    get_zone,
    is_valid_timezone,
    localize,
    set_timezone,
)

NEW_YORK = zoneinfo.ZoneInfo("America/New_York")


def current_zone_view(request: HttpRequest) -> HttpResponse:
    return HttpResponse(timezone.get_current_timezone_name())


class ZoneValidationTests(SimpleTestCase):
    def test_real_zone_names_are_accepted(self) -> None:
        for name in ("UTC", "Asia/Dhaka", "America/New_York"):
            self.assertTrue(is_valid_timezone(name))
        self.assertEqual(get_zone("Asia/Dhaka").key, "Asia/Dhaka")

    def test_junk_and_path_tricks_are_rejected(self) -> None:
        for name in ("", "Not/AZone", "../../etc/passwd", "America", "UTC\x00", " UTC"):
            with self.subTest(name=name):
                self.assertFalse(is_valid_timezone(name))
        with self.assertRaises(ValueError):
            get_zone("Not/AZone")


class LocalizeTests(SimpleTestCase):
    def test_an_ordinary_time_gets_the_right_offset(self) -> None:
        moment = localize(datetime(2026, 1, 15, 9, 30), NEW_YORK)
        self.assertEqual(moment.utcoffset(), timedelta(hours=-5))
        self.assertEqual(moment.astimezone(UTC).hour, 14)

    def test_a_time_skipped_by_daylight_saving_does_not_exist(self) -> None:
        with self.assertRaises(ValueError) as context:
            localize(datetime(2026, 3, 8, 2, 30), NEW_YORK)    # clocks jump from 02:00 to 03:00
        self.assertIn("does not exist", str(context.exception))

    def test_a_repeated_time_must_be_resolved_explicitly(self) -> None:
        repeated = datetime(2026, 11, 1, 1, 30)                  # clocks go back from 02:00 to 01:00
        with self.assertRaises(ValueError):
            localize(repeated, NEW_YORK)
        earlier = localize(repeated, NEW_YORK, ambiguous="earlier")
        later = localize(repeated, NEW_YORK, ambiguous="later")
        self.assertEqual(earlier.utcoffset(), timedelta(hours=-4))
        self.assertEqual(later.utcoffset(), timedelta(hours=-5))
        self.assertEqual(later - earlier, timedelta(hours=1))

    def test_aware_input_is_refused(self) -> None:
        with self.assertRaises(ValueError):
            localize(datetime(2026, 1, 1, tzinfo=UTC), NEW_YORK)


class FormatLocalTests(SimpleTestCase):
    def test_formats_in_the_given_zone(self) -> None:
        moment = datetime(2026, 1, 15, 12, 0, tzinfo=UTC)
        self.assertEqual(format_local(moment, get_zone("Asia/Dhaka")), "2026-01-15 18:00")

    def test_uses_the_current_zone_when_none_is_given(self) -> None:
        moment = datetime(2026, 1, 15, 12, 0, tzinfo=UTC)
        with timezone.override(get_zone("Asia/Dhaka")):
            self.assertEqual(format_local(moment), "2026-01-15 18:00")

    def test_naive_datetimes_are_refused(self) -> None:
        with self.assertRaises(ValueError):
            format_local(datetime(2026, 1, 15, 12, 0))


class MiddlewareTests(SimpleTestCase):
    def call(self, cookie: str | None) -> str:
        request = RequestFactory().get("/")
        if cookie is not None:
            request.COOKIES[TIMEZONE_COOKIE] = cookie
        response = TimezoneMiddleware(current_zone_view)(request)
        return response.content.decode()

    def test_a_valid_cookie_activates_the_zone_during_the_request(self) -> None:
        self.assertEqual(self.call("Asia/Dhaka"), "Asia/Dhaka")

    def test_missing_or_invalid_cookies_fall_back_to_the_default(self) -> None:
        for cookie in (None, "", "Not/AZone", "../../etc/passwd", "America"):
            with self.subTest(cookie=cookie):
                self.assertEqual(self.call(cookie), settings.TIME_ZONE)

    def test_the_zone_is_reset_after_the_request_even_when_the_view_fails(self) -> None:
        def failing_view(request: HttpRequest) -> HttpResponse:
            raise RuntimeError("boom")

        request = RequestFactory().get("/")
        request.COOKIES[TIMEZONE_COOKIE] = "Asia/Dhaka"
        with self.assertRaises(RuntimeError):
            TimezoneMiddleware(failing_view)(request)
        self.assertEqual(timezone.get_current_timezone_name(), settings.TIME_ZONE)


class SetTimezoneViewTests(SimpleTestCase):
    def test_a_valid_zone_is_stored_in_a_lax_cookie(self) -> None:
        response = set_timezone(RequestFactory().post("/", {"timezone": "Asia/Dhaka"}))
        self.assertEqual(response.status_code, 200)
        cookie = response.cookies[TIMEZONE_COOKIE]
        self.assertEqual((cookie.value, cookie["samesite"]), ("Asia/Dhaka", "Lax"))

    def test_an_invalid_zone_is_rejected_without_a_cookie(self) -> None:
        response = set_timezone(RequestFactory().post("/", {"timezone": "Mars/Base"}))
        self.assertEqual(response.status_code, 400)
        self.assertNotIn(TIMEZONE_COOKIE, response.cookies)


class TranslationTests(SimpleTestCase):
    def test_plural_forms(self) -> None:
        self.assertEqual(stock_label(0), "Sold out")
        self.assertEqual(stock_label(1), "1 copy in stock")
        self.assertEqual(stock_label(5), "5 copies in stock")
        with self.assertRaises(ValueError):
            stock_label(-1)

    def test_lazy_strings_translate_when_they_are_used_not_when_they_are_created(self) -> None:
        label = gettext_lazy("Yes")                 # created with no language chosen
        self.assertEqual(str(label), "Yes")
        with override("es"):
            self.assertEqual(str(label), "Sí")       # Django ships a Spanish catalog
        self.assertEqual(str(label), "Yes")
```

The translation test uses a word from Django's own Spanish catalog so it works before you have written any translations of your own. Once your `locale/es` catalog exists and is compiled, add tests for your strings in the same way.

## Common mistakes

- Using `datetime.now()` or `datetime.utcnow()` and storing naive values.
- Turning `USE_TZ` off to make an error go away.
- Storing local times as if they were UTC, or storing a fixed offset instead of a zone name for recurring events.
- Not resetting an activated time zone at the end of the request.
- Passing unvalidated user input to `ZoneInfo`.
- Ignoring nonexistent and ambiguous local times around daylight saving changes.
- Using `gettext` instead of `gettext_lazy` at import time, freezing text in the default language.
- Concatenating translated fragments or building sentences with f-strings, which prevents correct translation.
- Building plurals with `if n == 1` instead of `ngettext`.
- Placing `LocaleMiddleware` before the session middleware or after `CommonMiddleware`.
- Forgetting `compilemessages`, so translations never appear.
- Letting localized number formatting leak into JavaScript, HTML attributes, or API output.
- Assuming translation covers the data in your database.
- Designing layouts that break in right-to-left languages or with longer translated text.

## Practice exercises

1. Write a function `next_occurrence(local_time: time, zone_name: str, after: datetime) -> datetime` that returns the next UTC instant at which a daily event at the given local time occurs after `after`, using `localize` with sensible handling of nonexistent and ambiguous times. Test across both daylight saving changes in 2026 for New York.
2. Add a `preferred_timezone` field to the custom user model (validated against `available_timezones()`), change the middleware to prefer it over the cookie for logged-in users, and test both sources.
3. Mark all user-visible strings in one app for translation, run `makemessages -l es`, translate five of them including one plural and one with `pgettext`, compile, and write a test that renders them under `override("es")`.
4. Add `LocaleMiddleware` and `i18n_patterns` to the project, and write tests that `Accept-Language: es` produces `Content-Language: es`, that `/es/...` URLs resolve, and that an unsupported language falls back to the default.
5. Write a template tag `local_datetime` that renders an aware datetime in a given zone name with a localized format, and a test that shows the same instant in two zones and two languages.
6. Make the HTML layout of one page work right to left: add `dir`, replace left and right CSS properties with logical ones, and write a test that the root element has `dir="rtl"` under `override("ar")`.
7. Audit your models for fields that store dates and datetimes. For each, write down whether it is an instant or a calendar date, whose time zone applies to "today", and what the business rule is for a customer in another zone.

## Summary

Keep time and text neutral inside the system and adapt at the edge. Store aware datetimes in UTC (`USE_TZ = True`, `timezone.now()`), convert to the viewer's zone when displaying and when interpreting a typed local time, validate zone names before using them, activate a zone per request and always reset it, and handle the nonexistent and ambiguous local times that daylight saving creates. For text, mark strings with `gettext`, use `gettext_lazy` for anything defined at import time, write whole sentences with named placeholders, use `ngettext` for plurals, and run `makemessages` and `compilemessages`. Put `LocaleMiddleware` after the session middleware and before `CommonMiddleware`, let localized formats do their work (and unlocalize values for machines), support right-to-left languages with `dir` and logical CSS, and remember that translating code does not translate data in your database.
