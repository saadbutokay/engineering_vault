## Overview

HTTP is stateless (note 0.1): each request arrives with no memory of the previous one. Logins, shopping carts, flash messages, and "recently viewed" lists all need that memory, and sessions are how Django provides it. A session stores data on the server and gives the browser a small, random identifier in a cookie. This note explains how sessions work, the available backends, the `request.session` API, expiry and cookie settings, the `HttpOnly`, `Secure`, and `SameSite` attributes, session fixation, why session data is JSON only, and how to log a user out everywhere. It also covers plain and signed cookies. It ends with typed, tested code for session data, hardened cookie settings, and ending every session of one user.

This note uses the built-in `django.contrib.auth.models.User` in its tests. Note 4.2 introduces a custom user model; the session code is identical once you switch to `get_user_model()`.

## Core concepts

### How a session works

1. A request arrives without a session cookie. Django creates an empty session lazily, only if your code uses it.
2. When your code stores something (`request.session["cart"] = [...]`), Django saves the data in the session store under a random key and sends `Set-Cookie: sessionid=<key>`.
3. The browser returns that cookie with every later request.
4. `SessionMiddleware` loads the data for that key into `request.session`.

The cookie contains only an identifier. The data stays on the server, so the user cannot read or alter it, and you can end a session by deleting it.

### Session backends

Selected with `SESSION_ENGINE`:

- Database (`django.contrib.sessions.backends.db`): rows in `django_session`. The default, durable, and simple.
- Cache (`...backends.cache`): fastest, but data is lost if the cache is flushed or restarted.
- Cached database (`...backends.cached_db`): writes to the database and reads through the cache. A good production default.
- File (`...backends.file`): one file per session. Rarely a good choice.
- Signed cookies (`...backends.signed_cookies`): the data itself lives in the cookie, signed with `SECRET_KEY`. No server storage, but the data is visible to the user, limited to about 4 KB, and cannot be revoked.

### The request.session API

`request.session` behaves like a dictionary: `session["key"] = value`, `session.get("key", default)`, `session.pop("key", None)`, `"key" in session`, `session.setdefault(...)`. Additional methods:

- `session.flush()`: delete the data and the cookie, and start fresh.
- `session.cycle_key()`: keep the data but switch to a new session key.
- `session.set_expiry(value)`, `session.get_expiry_age()`, `session.get_expire_at_browser_close()`.
- `session.session_key`: the identifier, or `None` before the first save.

### Expiry

- `SESSION_COOKIE_AGE` (default two weeks): how long a session lasts after its last save.
- `SESSION_EXPIRE_AT_BROWSER_CLOSE`: make the cookie a session cookie that browsers drop when closed. Browsers with "restore tabs" may keep it.
- `SESSION_SAVE_EVERY_REQUEST`: refresh the expiry on every request instead of only when data changes.
- `session.set_expiry(seconds)` overrides one session. `0` means browser close and `None` means the global setting.

Expired rows are not removed automatically. Run `manage.py clearsessions` regularly, for example from cron.

### Cookie settings

- `SESSION_COOKIE_NAME`: the cookie name (default `sessionid`).
- `SESSION_COOKIE_HTTPONLY` (default true): JavaScript cannot read the cookie, which limits the damage of cross-site scripting.
- `SESSION_COOKIE_SECURE`: send the cookie only over HTTPS. Set it to true in production.
- `SESSION_COOKIE_SAMESITE` (default `Lax`): controls whether the cookie accompanies cross-site requests. `Lax` allows top-level navigations and blocks most cross-site subrequests, which helps against cross-site request forgery. `Strict` blocks even links from other sites. `None` allows everything and requires `Secure`.
- `SESSION_COOKIE_DOMAIN`: leave it unset (host-only) unless you must share a session across subdomains. A broad domain widens who can read or overwrite the cookie.
- `SESSION_COOKIE_PATH`: normally `/`.

### Session fixation

An attacker who can make a victim use a known session identifier waits for the victim to log in, then uses the same identifier. The defense is to issue a new identifier at login. Django's `login()` does this for you (it calls `cycle_key()` or `flush()`). Any code that raises a user's privileges must do the same.

### JSON serialization

Session data is stored with a JSON serializer. Only JSON types survive: strings, numbers, booleans, `None`, lists, and dictionaries with string keys. A `datetime`, `Decimal`, or model instance fails. The old pickle serializer was removed in Django 5.0, because unpickling data from a tampered store allows code execution. Store identifiers and plain values, and rebuild objects when you read them.

### Logging out everywhere

Logging out ends one session. "Log out of all devices" must end every session belonging to the user. Options: delete all sessions that contain the user's id, or change something that all sessions check. Django already does the second one when the password changes: each session stores a hash derived from the password, and `AuthenticationMiddleware` logs the user out of any session whose hash no longer matches.

### Plain, signed, and encrypted cookies

You can set cookies directly with `response.set_cookie(name, value, max_age=..., secure=..., httponly=..., samesite=...)` and read them from `request.COOKIES`. A signed cookie (`response.set_signed_cookie`, `request.get_signed_cookie`) cannot be altered without detection, but its content is visible to the user. Never put secrets in cookies. Use sessions for anything the user must not see or change.

## Basic syntax examples

### Using the session in a view

```python
from django.http import HttpRequest, HttpResponse


def add_to_wishlist(request: HttpRequest, book_id: int) -> HttpResponse:
    wishlist: list[int] = request.session.get("wishlist", [])
    if book_id not in wishlist:
        wishlist.append(book_id)
    request.session["wishlist"] = wishlist     # assign, so Django sees the change
    return HttpResponse(f"{len(wishlist)} books on your wishlist")
```

### In-place changes are not detected

```python
request.session["wishlist"].append(7)    # wrong: Django does not notice
request.session.modified = True          # tell Django you changed it, or reassign the key
```

### Expiry

```python
request.session.set_expiry(300)          # five minutes from now
request.session.set_expiry(0)            # until the browser closes
request.session.set_expiry(None)         # use SESSION_COOKIE_AGE
print(request.session.get_expiry_age())  # seconds remaining
```

### Settings for a production-like setup

```python
SESSION_ENGINE = "django.contrib.sessions.backends.cached_db"
SESSION_COOKIE_AGE = 60 * 60 * 24 * 14
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_SAMESITE = "Lax"
SESSION_SAVE_EVERY_REQUEST = False
```

### Cleaning up expired sessions

```bash
uv run python manage.py clearsessions
```

### Rotating the key and flushing

```python
request.session.cycle_key()   # same data, new identifier (after a privilege change)
request.session.flush()       # remove everything (logout)
```

### Setting and reading plain cookies

```python
def set_banner_dismissed(request: HttpRequest) -> HttpResponse:
    response = HttpResponse("ok")
    response.set_cookie(
        "banner_dismissed",
        "1",
        max_age=60 * 60 * 24 * 30,
        secure=True,
        httponly=True,
        samesite="Lax",
    )
    return response


dismissed = request.COOKIES.get("banner_dismissed") == "1"
```

### Signed cookies

```python
response.set_signed_cookie("theme", "dark", max_age=31_536_000, samesite="Lax")

theme = request.get_signed_cookie("theme", default="light", max_age=31_536_000)
```

With a `default`, a missing or tampered cookie returns the default instead of raising `BadSignature`.

### Deleting a cookie

```python
response.delete_cookie("theme")
```

### Inspecting a session in the shell

```python
from django.contrib.sessions.models import Session

session = Session.objects.get(session_key="abc123...")
print(session.get_decoded())   # {'_auth_user_id': '7', 'wishlist': [3, 8], ...}
print(session.expire_date)
```

## Deeper behavior

### Sessions are lazy

Django does not create or save a session unless you touch `request.session`. When you read it, Django adds `Vary: Cookie` to the response so caches do not share the page between users. If you read the session in a page that should be cached publicly, you will defeat the cache. When you modify it, Django saves it and sends the cookie at the end of the request. Reading alone does not refresh the expiry unless `SESSION_SAVE_EVERY_REQUEST` is on.

### What is in a session after login

`login()` stores three entries: `_auth_user_id` (the primary key as a string), `_auth_user_backend` (the dotted path of the authentication backend), and `_auth_user_hash` (an HMAC derived from the user's password hash). On each request, `AuthenticationMiddleware` loads the user, recomputes the hash, and treats the session as anonymous if it differs. That is why a password change ends every other session, and why changing `SECRET_KEY` logs everyone out (unless you list the old key in `SECRET_KEY_FALLBACKS`).

### Keeping the current user logged in after a password change

After changing a password in a view, call `update_session_auth_hash(request, user)`. Without it, the change invalidates the current session as well and the user is logged out immediately. Django's built-in `PasswordChangeView` does this for you.

### Fixation, in detail

`login()` creates a new key when the old session was anonymous (keeping its data, so an anonymous cart survives), and flushes the old data if the session already belonged to a different user. Explicitly call `request.session.cycle_key()` when you elevate privileges in other ways, for example after a second-factor check.

### Why data must be JSON

JSON cannot represent code, so a tampered session store cannot execute anything when loaded. The cost is that you must convert values: store a `datetime` as an ISO string and a `Decimal` as a string. Do not put large objects in sessions either. Every request loads the entire session, so store identifiers and look up the rest.

### The cookie is a bearer token

Whoever holds the session cookie is the user. Keep it out of reach: `HttpOnly` (against scripts), `Secure` (against network eavesdropping), HTTPS everywhere, and short lifetimes for sensitive applications. Browsers also support the `__Host-` cookie name prefix, which makes a browser accept the cookie only if it is `Secure`, has `Path=/`, and has no `Domain` attribute. That blocks a vulnerable subdomain from planting or overwriting your cookie.

### SameSite and CSRF

`SameSite=Lax` is a strong second layer of protection against cross-site request forgery. It does not replace Django's CSRF tokens (note 6.1): older browsers ignore it, same-site attackers (a compromised subdomain) are not blocked by it, and `Lax` still sends the cookie on top-level GET navigations, so state must never change on GET.

### Behind a proxy

When TLS ends at a proxy, Django sees plain HTTP unless you set `SECURE_PROXY_SSL_HEADER`. Secure cookies still work (Django sets the flag from the setting, not from the connection), but `request.is_secure()` and redirects need the header (note 6.2). Set it only if your proxy strips and sets that header itself; otherwise clients can forge it.

### Choosing a backend

- Database: simplest, persistent, queryable (you can find a user's sessions), but a query on every request.
- Cached database: the same durability with far fewer queries. Recommended.
- Cache only: fast, but a cache flush or eviction logs users out, so use it when that is acceptable.
- Signed cookies: no storage, but no revocation, visible content, and replay is possible until expiry. Suitable for small, non-sensitive data.

### Session size and concurrency

Two simultaneous requests from one browser both load the session, change it, and save it, and the last save wins. A request that adds item A and another that adds item B can lose one of them. Treat the session as a place for small, mostly-read data, and keep contested state (carts that matter, counters) in the database.

### Ending all of a user's sessions

For the database and cached database backends, find sessions that contain the user's id and delete them. Scanning every session is linear in the table size, which is acceptable for modest sites and wrong for very large ones. At scale, keep a table that maps users to session keys, or rely on the password-hash check by rotating a per-user "session version" that you mix into the session auth hash. The example below uses the scan.

### Cookies and privacy law

A session cookie that is strictly necessary for a service the user asked for (login, cart) is generally exempt from consent requirements, while analytics and advertising cookies are not. Check the rules that apply to your users.

## Industry-level snippet

Three small pieces that most applications need: typed access to a "recently viewed" list kept in the session (never trusting the stored shape), a validated settings builder for hardened session cookies, and a function that ends every session of one user. A signed cookie helper for a user preference rounds it out.

`apps/accounts/session_data.py`:

```python
"""Typed, defensive access to values stored in the session."""

from django.contrib.sessions.backends.base import SessionBase

RECENT_KEY = "recently_viewed"
MAX_RECENT = 10


class RecentlyViewed:
    """A most-recent-first list of book ids stored in the session."""

    def __init__(self, session: SessionBase, *, limit: int = MAX_RECENT) -> None:
        if limit <= 0:
            raise ValueError("limit must be positive")
        self._session = session
        self._limit = limit

    def ids(self) -> list[int]:
        """Return the stored ids, ignoring anything that is not a positive int."""
        raw = self._session.get(RECENT_KEY, [])
        if not isinstance(raw, list):
            return []
        cleaned = [
            item
            for item in raw
            if isinstance(item, int) and not isinstance(item, bool) and item > 0
        ]
        return cleaned[: self._limit]

    def add(self, book_id: int) -> None:
        if book_id <= 0:
            raise ValueError("book_id must be positive")
        remaining = [item for item in self.ids() if item != book_id]
        self._session[RECENT_KEY] = [book_id, *remaining][: self._limit]

    def clear(self) -> None:
        self._session.pop(RECENT_KEY, None)
```

`config/session_settings.py`:

```python
"""Validated session cookie settings."""

from typing import Any

from config.env import ConfigError

MIN_AGE_SECONDS = 300
MAX_AGE_SECONDS = 90 * 24 * 3600


def build_session_settings(
    *,
    debug: bool,
    cookie_age_seconds: int = 14 * 24 * 3600,
    host_prefixed_cookie: bool = True,
) -> dict[str, Any]:
    """Return session settings that are safe by default.

    The ``__Host-`` cookie prefix is used outside debug mode, because browsers
    only accept it for Secure cookies without a Domain attribute.
    """
    if not MIN_AGE_SECONDS <= cookie_age_seconds <= MAX_AGE_SECONDS:
        raise ConfigError(
            f"cookie_age_seconds must be between {MIN_AGE_SECONDS} and {MAX_AGE_SECONDS}"
        )
    secure = not debug
    name = "__Host-sessionid" if secure and host_prefixed_cookie else "sessionid"
    return {
        "SESSION_ENGINE": "django.contrib.sessions.backends.cached_db",
        "SESSION_COOKIE_NAME": name,
        "SESSION_COOKIE_AGE": cookie_age_seconds,
        "SESSION_COOKIE_HTTPONLY": True,
        "SESSION_COOKIE_SECURE": secure,
        "SESSION_COOKIE_SAMESITE": "Lax",
        "SESSION_COOKIE_DOMAIN": None,
        "SESSION_COOKIE_PATH": "/",
        "SESSION_EXPIRE_AT_BROWSER_CLOSE": False,
        "SESSION_SAVE_EVERY_REQUEST": False,
    }
```

In `config/settings.py`:

```python
from config.session_settings import build_session_settings

globals().update(build_session_settings(debug=DEBUG))
```

`apps/accounts/sessions.py`:

```python
"""Operations on stored sessions."""

import logging

from django.contrib.auth import SESSION_KEY
from django.contrib.sessions.models import Session
from django.utils import timezone

logger = logging.getLogger(__name__)


def terminate_user_sessions(user_id: int, *, keep_session_key: str | None = None) -> int:
    """Delete every live session that belongs to ``user_id``.

    Pass ``keep_session_key`` to spare the caller's current session. Returns
    the number of sessions ended. This scans all live sessions, so it suits
    moderate traffic; very large sites should index sessions by user.
    """
    wanted = str(user_id)  # Django stores the primary key as a string
    ended = 0
    live = Session.objects.filter(expire_date__gt=timezone.now())
    for session in live.iterator():
        if session.session_key == keep_session_key:
            continue
        if session.get_decoded().get(SESSION_KEY) == wanted:
            session.delete()
            ended += 1
    logger.info("Ended %d sessions for user %s", ended, user_id)
    return ended
```

`apps/accounts/preferences.py`:

```python
"""A tamper-proof (but not secret) user preference cookie."""

from django.conf import settings
from django.http import HttpRequest, HttpResponse

THEME_COOKIE = "theme"
THEME_MAX_AGE = 365 * 24 * 3600
DEFAULT_THEME = "light"
ALLOWED_THEMES = frozenset({"light", "dark"})


def read_theme(request: HttpRequest) -> str:
    """Return the saved theme, falling back for missing or tampered cookies."""
    value = request.get_signed_cookie(THEME_COOKIE, default=DEFAULT_THEME, max_age=THEME_MAX_AGE)
    return value if value in ALLOWED_THEMES else DEFAULT_THEME


def save_theme(response: HttpResponse, theme: str) -> None:
    if theme not in ALLOWED_THEMES:
        raise ValueError(f"Unknown theme: {theme!r}")
    response.set_signed_cookie(
        THEME_COOKIE,
        theme,
        max_age=THEME_MAX_AGE,
        secure=not settings.DEBUG,
        httponly=True,
        samesite="Lax",
    )
```

`apps/accounts/tests/test_sessions.py` (with an empty `__init__.py` in `tests/`):

```python
from django.contrib.auth.models import User
from django.contrib.sessions.backends.signed_cookies import SessionStore
from django.contrib.sessions.models import Session
from django.http import HttpRequest, HttpResponse
from django.test import Client, RequestFactory, SimpleTestCase, TestCase

from apps.accounts.preferences import THEME_COOKIE, read_theme, save_theme
from apps.accounts.session_data import RECENT_KEY, RecentlyViewed
from apps.accounts.sessions import terminate_user_sessions
from config.env import ConfigError
from config.session_settings import build_session_settings

PASSWORD = "correct-horse-battery-staple"


class RecentlyViewedTests(SimpleTestCase):
    def setUp(self) -> None:
        self.session = SessionStore()
        self.recent = RecentlyViewed(self.session, limit=3)

    def test_most_recent_first_without_duplicates(self) -> None:
        for book_id in (1, 2, 3, 2):
            self.recent.add(book_id)
        self.assertEqual(self.recent.ids(), [2, 3, 1])

    def test_the_list_is_capped(self) -> None:
        for book_id in range(1, 6):
            self.recent.add(book_id)
        self.assertEqual(self.recent.ids(), [5, 4, 3])

    def test_corrupted_session_data_is_ignored(self) -> None:
        self.session[RECENT_KEY] = "garbage"
        self.assertEqual(self.recent.ids(), [])
        self.session[RECENT_KEY] = [1, "x", True, -5, None, 2.5, 2]
        self.assertEqual(self.recent.ids(), [1, 2])

    def test_invalid_input_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            self.recent.add(0)
        with self.assertRaises(ValueError):
            RecentlyViewed(self.session, limit=0)

    def test_clear_removes_the_key(self) -> None:
        self.recent.add(1)
        self.recent.clear()
        self.assertNotIn(RECENT_KEY, self.session)


class SessionSettingsTests(SimpleTestCase):
    def test_production_settings_are_secure(self) -> None:
        config = build_session_settings(debug=False)
        self.assertTrue(config["SESSION_COOKIE_SECURE"])
        self.assertTrue(config["SESSION_COOKIE_HTTPONLY"])
        self.assertEqual(config["SESSION_COOKIE_SAMESITE"], "Lax")
        self.assertEqual(config["SESSION_COOKIE_NAME"], "__Host-sessionid")
        self.assertIsNone(config["SESSION_COOKIE_DOMAIN"])

    def test_debug_settings_work_over_plain_http(self) -> None:
        config = build_session_settings(debug=True)
        self.assertFalse(config["SESSION_COOKIE_SECURE"])
        self.assertEqual(config["SESSION_COOKIE_NAME"], "sessionid")

    def test_cookie_lifetime_is_bounded(self) -> None:
        for seconds in (10, 100 * 24 * 3600):
            with self.subTest(seconds=seconds):
                with self.assertRaises(ConfigError):
                    build_session_settings(debug=False, cookie_age_seconds=seconds)


class ThemeCookieTests(SimpleTestCase):
    def request_with(self, cookies: dict[str, str]) -> HttpRequest:
        request = RequestFactory().get("/")
        request.COOKIES.update(cookies)
        return request

    def test_round_trip(self) -> None:
        response = HttpResponse()
        save_theme(response, "dark")
        morsel = response.cookies[THEME_COOKIE]
        self.assertTrue(morsel["httponly"])
        self.assertEqual(morsel["samesite"], "Lax")
        request = self.request_with({THEME_COOKIE: morsel.value})
        self.assertEqual(read_theme(request), "dark")

    def test_tampered_or_missing_cookies_fall_back(self) -> None:
        self.assertEqual(read_theme(self.request_with({THEME_COOKIE: "dark"})), "light")
        self.assertEqual(read_theme(self.request_with({})), "light")

    def test_unknown_theme_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            save_theme(HttpResponse(), "neon")


class SessionBehaviorTests(TestCase):
    def setUp(self) -> None:
        self.user = User.objects.create_user("ada", password=PASSWORD)

    def test_login_rotates_the_key_and_keeps_anonymous_data(self) -> None:
        client = Client()
        session = client.session
        session["wishlist"] = [1, 2]
        session.save()
        before = session.session_key

        self.assertTrue(client.login(username="ada", password=PASSWORD))

        after = client.session.session_key
        self.assertNotEqual(before, after)               # fixation defense
        self.assertEqual(client.session["wishlist"], [1, 2])

    def test_expiry_can_be_set_per_session(self) -> None:
        client = Client()
        session = client.session
        session.set_expiry(300)
        self.assertAlmostEqual(session.get_expiry_age(), 300, delta=5)


class TerminateSessionsTests(TestCase):
    def setUp(self) -> None:
        self.ada = User.objects.create_user("ada", password=PASSWORD)
        self.grace = User.objects.create_user("grace", password=PASSWORD)

    def login(self, user: User) -> Client:
        client = Client()
        client.force_login(user)
        return client

    def test_all_sessions_of_one_user_are_ended(self) -> None:
        ada_phone, ada_laptop = self.login(self.ada), self.login(self.ada)
        grace = self.login(self.grace)

        ended = terminate_user_sessions(self.ada.pk)

        self.assertEqual(ended, 2)
        self.assertFalse(Session.objects.filter(session_key=ada_phone.session.session_key).exists())
        self.assertFalse(Session.objects.filter(session_key=ada_laptop.session.session_key).exists())
        self.assertTrue(Session.objects.filter(session_key=grace.session.session_key).exists())

    def test_the_current_session_can_be_spared(self) -> None:
        current, other = self.login(self.ada), self.login(self.ada)
        keep = current.session.session_key
        self.assertEqual(terminate_user_sessions(self.ada.pk, keep_session_key=keep), 1)
        self.assertTrue(Session.objects.filter(session_key=keep).exists())
        self.assertFalse(Session.objects.filter(session_key=other.session.session_key).exists())

    def test_a_user_with_no_sessions_ends_zero(self) -> None:
        self.assertEqual(terminate_user_sessions(self.grace.pk), 0)

    def test_changing_the_password_invalidates_other_sessions(self) -> None:
        client = self.login(self.ada)
        before = client.get("/")
        self.assertTrue(before.wsgi_request.user.is_authenticated)

        self.ada.set_password("a-completely-new-passphrase")
        self.ada.save()

        # The session row still exists, but its stored password hash no longer
        # matches, so the middleware treats the request as anonymous.
        after = client.get("/")
        self.assertFalse(after.wsgi_request.user.is_authenticated)
```

## Common mistakes

- Putting `datetime`, `Decimal`, or model instances in the session and getting serialization errors.
- Mutating a list or dictionary inside the session in place and not marking it modified.
- Storing large objects or whole carts of full records in the session, loading them on every request.
- Leaving `SESSION_COOKIE_SECURE` off in production, or forgetting that `HttpOnly` protects the cookie only from scripts.
- Setting `SESSION_COOKIE_DOMAIN` to a parent domain for convenience, exposing the session to every subdomain.
- Using the signed cookie backend for data the user must not see or for state that must be revocable.
- Never running `clearsessions`, so the session table grows without bound.
- Reading the session in views that should be publicly cacheable, which adds `Vary: Cookie` and defeats caching.
- Forgetting `update_session_auth_hash` after a password change, which logs the user out.
- Not rotating the session key when privileges change.
- Trusting values read back from the session without validating their shape.
- Treating `SameSite` as a replacement for CSRF tokens.

## Practice exercises

1. Build a wishlist stored in the session with `add`, `remove`, and `list` operations, bounded to 50 items, and write tests with `SessionStore` that cover duplicates, the bound, and corrupted data.
2. In the shell, log in through the test client and print `Session.objects.get(...).get_decoded()`. Identify the three authentication keys and explain what each is for.
3. Write a middleware that calls `request.session.set_expiry(...)` to give staff users a shorter session than customers, and test both cases.
4. Demonstrate session fixation: write a test that sets a known session key before login (with `cycle_key` removed from a scratch copy of the login code) and shows the key survives, then show that Django's real `login()` changes it.
5. Add a "log out everywhere else" view that calls `terminate_user_sessions` with the current session spared and redirects. Write tests for it with two clients.
6. Write a scratch view that changes the logged-in user's password. Show with the test client that without `update_session_auth_hash` the next request is anonymous, and that with it the same session stays logged in.
7. Write a management command `prune_sessions` that reports how many expired sessions exist and deletes them in batches with a `--batch-size` option, and test it with sessions whose `expire_date` you set in the past.

## Summary

A session keeps server-side state for a browser that holds only a random identifier in a cookie. The database or cached database backend is the right default, signed cookies suit small, non-secret data, and data must be JSON because pickle was removed for safety. Assign new values to session keys or set `modified`, keep sessions small, and validate what you read back. Protect the cookie with `HttpOnly`, `Secure`, `SameSite=Lax`, no broad `Domain`, and optionally the `__Host-` prefix. Django's `login()` rotates the key to defeat session fixation, and the stored password hash ends other sessions when a password changes. To log a user out everywhere, delete their sessions or invalidate a value every session checks, and run `clearsessions` to remove expired rows.
