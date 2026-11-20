## Overview

Browser sessions with a password work well for people using your own website. Real systems also need other ways to prove identity: "Sign in with Google" so users do not manage another password, API keys and tokens for scripts and mobile apps, second factors for sensitive accounts, and defenses against the constant background noise of password-guessing bots. This note covers `django-allauth` for social login and email verification, the concepts of OAuth 2.0 and OpenID Connect, token authentication, JWT and its trade-offs compared with sessions, two-factor authentication, passkeys, brute-force protection with `django-axes` and rate limiting, and account enumeration. It ends with working, tested code for the parts you should understand rather than outsource: hashed and revocable API keys with a bearer-token decorator, and a cache-based rate limiter applied to the login view.

## Core concepts

### Why more than passwords

Passwords are reused, phished, and guessed. Each alternative shifts a risk:

- Social login delegates the password and its defenses to a provider that is usually better at it than you are.
- Second factors make a stolen password insufficient.
- Tokens give programs credentials that can be limited, rotated, and revoked independently of a person's password.
- Rate limits and lockouts make guessing expensive.

### django-allauth

`django-allauth` is the standard package for registration, login, email verification, password management, and social login with many providers (Google, GitHub, Microsoft, Apple, and others). It also supports multi-factor authentication and, in recent versions, passkeys. It works with a custom user model such as the one from note 4.2.

### OAuth 2.0 in one paragraph

OAuth 2.0 lets an application act on a user's behalf at another service without learning the user's password. The user is sent to the provider, approves specific scopes (such as "read your email address"), and the provider gives your application a token. Your application never sees the provider password. OAuth 2.0 is an authorization protocol: it answers "may this application do X for this user?"

### The authorization code flow with PKCE

The flow for web and mobile applications:

1. Your application redirects the browser to the provider's authorization endpoint with a client id, a redirect URI, requested scopes, a random `state` value, and a PKCE challenge (a hash of a secret the application keeps).
2. The user logs in at the provider and approves the scopes.
3. The provider redirects back to your redirect URI with a one-time `code` and your `state`.
4. Your server checks that `state` matches what it sent (this blocks cross-site request forgery on the login), then exchanges the `code`, together with the PKCE verifier and the client secret, at the provider's token endpoint.
5. The provider returns an access token (and often a refresh token and an ID token).

The older implicit flow, which returned tokens in the browser's URL, is obsolete.

### OpenID Connect

OpenID Connect (OIDC) is a thin identity layer on top of OAuth 2.0. It adds a signed ID token (a JWT) that states who the user is (`sub`, `email`, `email_verified`), a standard "userinfo" endpoint, and discovery documents. "Sign in with Google" is OIDC. Use OIDC when you want to know who someone is, and plain OAuth when you want to call an API for them.

### Token authentication

For programs, send a secret token in each request, normally in the header `Authorization: Bearer <token>`. Two common designs:

- Opaque tokens: random strings that are meaningless by themselves. The server looks them up. They are easy to revoke and reveal nothing.
- Self-contained tokens (JWT): signed documents holding claims (subject, expiry, scopes) that the server verifies without a lookup.

### JWT trade-offs compared with sessions

JSON Web Tokens are popular and frequently misused. Compared with server-side sessions:

- They let services verify identity without a shared session store, which helps when many independent services accept the same token.
- They are hard to revoke before they expire, because nothing on the server has to know about them. The usual answer is short lifetimes plus refresh tokens, which reintroduces server state.
- They tend to grow large, and their claims are readable by anyone holding the token.
- They fail in specific ways: accepting the `none` algorithm, confusing a public key with a shared secret, skipping audience and issuer checks.
- Storing them in browser storage exposes them to cross-site scripting, and storing them in cookies reintroduces CSRF concerns.

For a browser application served from your own domain, ordinary sessions are simpler and safer. Reach for tokens when programs, mobile apps, or other services are the callers.

### Two-factor authentication

A second factor proves possession of something in addition to knowing the password:

- TOTP codes from an authenticator app (via `django-otp` or allauth's MFA support), with recovery codes.
- Hardware keys and passkeys (WebAuthn), which are phishing-resistant because the credential works only on the real site.
- SMS codes, the weakest option because phone numbers can be hijacked, but better than nothing for many users.

### Passkeys

A passkey is a WebAuthn credential: a key pair created on the user's device, where the private key never leaves it, and the server stores only the public key. Logging in is a signed challenge confirmed by a fingerprint, face, or device PIN. There is no secret to phish or reuse. Recent `django-allauth` versions support passkeys, and the usual deployment keeps passwords as a fallback during the transition.

### Brute-force protection

Attackers try common passwords against many accounts (password spraying), or many passwords against one account (brute force), or leaked credentials from other sites (credential stuffing). Defenses:

- Rate limits per client and per account.
- Temporary lockouts, applied carefully so attackers cannot lock real users out at will.
- `django-axes`, which records failed logins and enforces lockouts by IP address, username, or both.
- Limits at the reverse proxy, before requests reach Django.
- Strong password rules and breached-password checks.
- Second factors, which make guessed passwords useless.

### Account enumeration

If your application reveals whether an email address has an account (in login errors, password reset responses, sign-up errors, or response timing), attackers can build lists of valid accounts for targeted attacks. Use identical messages, identical status codes, and similar timing for known and unknown accounts wherever you can.

## Basic syntax examples

### django-allauth settings

Follow the installation guide for the version you install, because setting names change between releases. A representative configuration for email login with Google:

```bash
uv add "django-allauth[socialaccount]"
```

```python
INSTALLED_APPS = [
    # ...
    "django.contrib.sites",
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
]

SITE_ID = 1

MIDDLEWARE = [
    # ... after AuthenticationMiddleware
    "allauth.account.middleware.AccountMiddleware",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

ACCOUNT_LOGIN_METHODS = {"email"}
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]
ACCOUNT_USER_MODEL_USERNAME_FIELD = None      # our user model has no username
ACCOUNT_EMAIL_VERIFICATION = "mandatory"

SOCIALACCOUNT_PROVIDERS = {
    "google": {
        "SCOPE": ["profile", "email"],
        "APP": {
            "client_id": os.environ["GOOGLE_CLIENT_ID"],
            "secret": os.environ["GOOGLE_CLIENT_SECRET"],
        },
    }
}
```

```python
# config/urls.py
urlpatterns = [
    path("accounts/", include("allauth.urls")),
]
```

Register `https://yourdomain.example/accounts/google/login/callback/` as an authorized redirect URI in the provider's console. Keep client secrets in environment variables (note 0.5).

### Verifying a JWT with PyJWT

```python
from typing import Any

import jwt

def verify_access_token(token: str, public_key: str) -> dict[str, Any]:
    return jwt.decode(
        token,
        public_key,
        algorithms=["RS256"],          # never read the algorithm from the token
        audience="https://api.example.com",
        issuer="https://auth.example.com/",
        options={"require": ["exp", "iat", "sub"]},
    )
```

### Calling an API with a bearer token

```bash
curl -H "Authorization: Bearer bk_1a2b3c4d_the-secret-part" https://shop.example.com/api/whoami/
```

### TOTP with django-otp

```bash
uv add django-otp qrcode
```

```python
INSTALLED_APPS += ["django_otp", "django_otp.plugins.otp_totp"]
MIDDLEWARE += ["django_otp.middleware.OTPMiddleware"]    # after AuthenticationMiddleware
```

```python
from django_otp import login as otp_login
from django_otp.plugins.otp_totp.models import TOTPDevice

device = TOTPDevice.objects.create(user=user, name="phone", confirmed=False)
device.config_url          # otpauth:// URL to show as a QR code

if device.verify_token(code_from_user):
    device.confirmed = True
    device.save()
    otp_login(request, device)       # marks this session as verified
```

```python
from django_otp.decorators import otp_required


@otp_required
def sensitive_view(request: HttpRequest) -> HttpResponse:
    ...
```

### django-axes

```bash
uv add django-axes
```

```python
from datetime import timedelta

INSTALLED_APPS += ["axes"]
AUTHENTICATION_BACKENDS = [
    "axes.backends.AxesStandaloneBackend",        # must come first
    "django.contrib.auth.backends.ModelBackend",
]
MIDDLEWARE += ["axes.middleware.AxesMiddleware"]

AXES_FAILURE_LIMIT = 5
AXES_COOLOFF_TIME = timedelta(minutes=15)
AXES_LOCKOUT_PARAMETERS = [["username", "ip_address"]]   # lock the pair, not just the account
AXES_RESET_ON_SUCCESS = True
```

### Rate limiting at the proxy

```nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    location /accounts/login/ {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://app;
    }
}
```

## Deeper behavior

### What can go wrong with social login

- Trusting an unverified email address from a provider and attaching the login to an existing local account with that email lets an attacker take it over. Link accounts automatically only when the provider asserts that the email is verified, or ask the user to log in locally first. Allauth's default behavior errs on the side of caution.
- Skipping the `state` check enables login CSRF. Use a library that handles it.
- Misconfigured redirect URIs (wildcards, open redirects) let attackers capture authorization codes. Register exact URIs.
- Requesting more scopes than needed increases harm if the token leaks.

### Email verification

An email address is not proof of identity until the user clicks a link sent to it. Require verification before you trust the address for password resets, notifications of sensitive actions, or linking accounts. `ACCOUNT_EMAIL_VERIFICATION = "mandatory"` in allauth blocks login until verified.

### Opaque tokens, done safely

A good API key design:

- Generate at least 128 bits of randomness with `secrets` (never `random`).
- Show the token once, at creation. Store only a hash. If the database leaks, the keys do not.
- Because the token is long and random, a fast hash such as SHA-256 is enough. (Slow password hashes protect low-entropy human passwords, and are wasted on random tokens.)
- Include a short non-secret prefix in the token so you can find the row without scanning, and so support staff and users can identify a key without seeing the secret.
- Compare hashes with `hmac.compare_digest`.
- Support expiry and revocation, record `last_used_at` (not on every request, to avoid a write per call), and scope tokens to the minimum permissions.
- Rotate by creating a new key, switching clients, and revoking the old one.

### Bearer tokens and CSRF

Browsers attach cookies automatically, and that is what CSRF abuses. A bearer token in an `Authorization` header is attached only by code that holds the token, so requests authenticated purely by that header do not need CSRF protection. That is why token-authenticated API views may be exempt from CSRF checks. The exemption is safe only if the view ignores session cookies, which the code below guarantees by authenticating from the header alone.

### Where JWTs belong

Good fits: short-lived access tokens issued by a central identity provider and verified by many services, and ID tokens from OIDC providers verified once at login. Poor fits: long-lived browser sessions for your own site. If you accept JWTs, pin the allowed algorithms, check `iss`, `aud`, `exp`, and `nbf`, fetch signing keys from the provider's published key set (and cache them), and keep lifetimes to minutes.

### Refresh tokens

Short access tokens are paired with refresh tokens that obtain new ones. A refresh token is a long-lived credential, so it must be stored securely, rotated on use (the old one is invalidated), and revocable. Detecting reuse of an already-rotated refresh token signals theft and should revoke the whole chain.

### Why rate limits belong in several places

A proxy limit protects the whole application cheaply and early. An application limit knows the account and can use the cache or database. `django-axes` adds history and lockouts. Combine layers, keep thresholds high enough for real users (shared office addresses, mobile networks), and think about denial of service: a lockout keyed only on the username lets an attacker lock a real user out by failing logins on purpose. Keying on username plus address, or using delays rather than hard locks, limits that.

### Fixed windows and their weakness

A fixed-window limiter counts requests in a time bucket. It is simple and cheap, but a client can send the limit at the end of one window and again at the start of the next, briefly doubling the rate. Sliding windows and token buckets smooth that at the cost of complexity. For login protection, the simple version is usually enough.

### Client addresses behind proxies

`REMOTE_ADDR` is the address of the connection, which is the proxy's when you run behind one, so every user would share one limit. Read the client address from a header such as `X-Forwarded-For` only if your proxy sets it and strips any client-supplied value, and only trust the entries your own infrastructure added. Using a forged header lets an attacker pick their own identity and bypass the limiter.

### Enumeration beyond messages

- Timing: Django's `ModelBackend` hashes the supplied password even for nonexistent users so that responses take about the same time. Do the same in your own code, as the API key authenticator below does with a dummy comparison.
- Password reset: always answer "if an account exists, we sent an email" (Django does).
- Sign-up: telling the user "this email is already registered" is useful and leaks existence. A privacy-preserving alternative is to accept every sign-up, send an email either way (a verification link for new users, a "you already have an account" note for existing ones), and show the same page.
- Rate limit the endpoints that can still be probed.

### Multi-factor details

TOTP secrets are shared secrets stored on the server, so protect them like passwords. Provide recovery codes (single use, hashed), and a process for lost devices that does not defeat the purpose. Require a second factor for sensitive actions (changing email, adding API keys), not only at login. WebAuthn and passkeys avoid shared secrets entirely.

### Logging and notifications

Record logins, failures, password changes, factor changes, and key creation, without logging passwords or tokens. Send an email when sensitive settings change, so account takeovers are noticed.

## Industry-level snippet

Hashed, expiring, revocable API keys for programs, a view decorator that authenticates a bearer token and returns consistent 401 responses, and a cache-based rate limiter applied to the login view. Raw tokens are returned once and never stored; unknown and wrong tokens take similar time and produce the same error; and a session cookie alone can never authenticate an API call.

Add to `apps/accounts/models.py`:

```python
from django.conf import settings


class ApiKey(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="api_keys")
    name = models.CharField(max_length=100)
    prefix = models.CharField(max_length=8, unique=True)       # public, used to find the row
    secret_hash = models.CharField(max_length=64)              # SHA-256 of the secret part
    created_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField(null=True, blank=True)
    revoked_at = models.DateTimeField(null=True, blank=True)
    last_used_at = models.DateTimeField(null=True, blank=True)

    def __str__(self) -> str:
        return f"{self.name} (bk_{self.prefix}_...)"
```

`apps/accounts/api_keys.py`:

```python
"""Hashed, expiring, revocable API keys."""

import hashlib
import hmac
import logging
import secrets
from datetime import datetime, timedelta

from django.db import IntegrityError, transaction
from django.utils import timezone

from apps.accounts.models import ApiKey, User

logger = logging.getLogger(__name__)

TOKEN_LABEL = "bk"
PREFIX_LENGTH = 8
MAX_NAME_LENGTH = 100
LAST_USED_RESOLUTION = timedelta(seconds=60)


def hash_secret(secret: str) -> str:
    """SHA-256 is appropriate here because the secret is long and random."""
    return hashlib.sha256(secret.encode()).hexdigest()


_DUMMY_HASH = hash_secret("dummy-secret-for-constant-work")


def create_api_key(
    *,
    user: User,
    name: str,
    lifetime: timedelta | None = None,
    now: datetime | None = None,
) -> tuple[ApiKey, str]:
    """Create a key and return ``(row, raw_token)``. The raw token is never stored."""
    cleaned = name.strip()
    if not 1 <= len(cleaned) <= MAX_NAME_LENGTH:
        raise ValueError(f"name must be 1 to {MAX_NAME_LENGTH} characters")
    if lifetime is not None and lifetime <= timedelta(0):
        raise ValueError("lifetime must be positive")

    moment = now or timezone.now()
    expires_at = moment + lifetime if lifetime is not None else None

    for _ in range(3):  # a prefix collision is possible but very unlikely
        prefix = secrets.token_hex(PREFIX_LENGTH // 2)
        secret = secrets.token_urlsafe(32)
        try:
            with transaction.atomic():
                key = ApiKey.objects.create(
                    user=user,
                    name=cleaned,
                    prefix=prefix,
                    secret_hash=hash_secret(secret),
                    expires_at=expires_at,
                )
        except IntegrityError:
            continue
        logger.info("Created API key %s for user %s", key.pk, user.pk)  # never log the token
        return key, f"{TOKEN_LABEL}_{prefix}_{secret}"
    raise RuntimeError("Could not generate a unique API key prefix")


def authenticate_api_key(raw: str, *, now: datetime | None = None) -> ApiKey | None:
    """Return the key row for a valid token, or ``None`` for any problem."""
    parts = raw.split("_", 2)
    if len(parts) != 3 or parts[0] != TOKEN_LABEL:
        return None
    _, prefix, secret = parts
    if len(prefix) != PREFIX_LENGTH or not secret:
        return None

    try:
        key = ApiKey.objects.select_related("user").get(prefix=prefix)
    except ApiKey.DoesNotExist:
        hmac.compare_digest(hash_secret(secret), _DUMMY_HASH)  # similar work either way
        return None

    if not hmac.compare_digest(key.secret_hash, hash_secret(secret)):
        return None

    moment = now or timezone.now()
    if (
        key.revoked_at is not None
        or (key.expires_at is not None and key.expires_at <= moment)
        or not key.user.is_active
    ):
        return None

    if key.last_used_at is None or moment - key.last_used_at >= LAST_USED_RESOLUTION:
        ApiKey.objects.filter(pk=key.pk).update(last_used_at=moment)
        key.last_used_at = moment
    return key


def revoke_api_key(key: ApiKey, *, now: datetime | None = None) -> None:
    if key.revoked_at is None:
        key.revoked_at = now or timezone.now()
        key.save(update_fields=["revoked_at"])
        logger.info("Revoked API key %s", key.pk)
```

`apps/accounts/api_auth.py`:

```python
"""Bearer-token authentication for API views."""

from collections.abc import Callable
from functools import wraps
from typing import Concatenate, ParamSpec

from django.http import HttpRequest, HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt

from apps.accounts.api_keys import authenticate_api_key

P = ParamSpec("P")


def unauthorized(code: str, message: str) -> JsonResponse:
    response = JsonResponse({"error": {"code": code, "message": message}}, status=401)
    response["WWW-Authenticate"] = 'Bearer realm="api"'
    return response


def api_key_required(
    view: Callable[Concatenate[HttpRequest, P], HttpResponse],
) -> Callable[Concatenate[HttpRequest, P], HttpResponse]:
    """Authenticate from the ``Authorization: Bearer`` header only.

    Session cookies are ignored, so the view is not exposed to CSRF and can
    safely be exempt from the CSRF check.
    """

    @wraps(view)
    def wrapper(request: HttpRequest, /, *args: P.args, **kwargs: P.kwargs) -> HttpResponse:
        scheme, _, token = request.headers.get("Authorization", "").partition(" ")
        if scheme.lower() != "bearer" or not token.strip():
            return unauthorized(
                "missing_token", "Send the header 'Authorization: Bearer <token>'."
            )
        key = authenticate_api_key(token.strip())
        if key is None:
            # One message for unknown, wrong, expired, and revoked tokens.
            return unauthorized("invalid_token", "The token is invalid, expired, or revoked.")
        request.user = key.user
        return view(request, *args, **kwargs)

    return csrf_exempt(wrapper)
```

`apps/accounts/ratelimit.py`:

```python
"""A small fixed-window rate limiter on top of Django's cache."""

import hashlib
from dataclasses import dataclass

from django.core.cache import BaseCache, caches


@dataclass(frozen=True)
class RateRule:
    limit: int
    window_seconds: int

    def __post_init__(self) -> None:
        if self.limit < 1 or self.window_seconds < 1:
            raise ValueError("limit and window_seconds must be at least 1")


class RateLimiter:
    def __init__(self, name: str, rule: RateRule, *, cache: BaseCache | None = None) -> None:
        self._name = name
        self.rule = rule
        self._cache = cache if cache is not None else caches["default"]

    def _key(self, identifier: str) -> str:
        # Hash the identifier: bounded key length and no personal data in the cache.
        digest = hashlib.sha256(identifier.encode()).hexdigest()[:32]
        return f"ratelimit:{self._name}:{digest}"

    def hit(self, identifier: str) -> bool:
        """Record an attempt. Return ``True`` if it is within the limit."""
        key = self._key(identifier)
        if self._cache.add(key, 1, timeout=self.rule.window_seconds):
            return True  # first attempt in a new window
        try:
            count = self._cache.incr(key)
        except ValueError:  # the key expired between add() and incr()
            self._cache.set(key, 1, timeout=self.rule.window_seconds)
            return True
        return count <= self.rule.limit
```

`apps/accounts/views.py` (additions):

```python
from typing import Any

from django.contrib.auth.views import LoginView
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods

from apps.accounts.api_auth import api_key_required, unauthorized
from apps.accounts.models import User
from apps.accounts.ratelimit import RateLimiter, RateRule

LOGIN_LIMITER = RateLimiter("login", RateRule(limit=5, window_seconds=300))


class RateLimitedLoginView(LoginView):
    """The standard login view with a limit on attempts per client and email."""

    def post(self, request: HttpRequest, *args: Any, **kwargs: Any) -> HttpResponse:
        email = request.POST.get("username", "").strip().lower()
        client = request.META.get("REMOTE_ADDR", "unknown")  # see the notes on proxies
        if not LOGIN_LIMITER.hit(f"{client}:{email}"):
            response = HttpResponse(
                "Too many login attempts. Try again later.",
                status=429,
                content_type="text/plain",
            )
            response["Retry-After"] = str(LOGIN_LIMITER.rule.window_seconds)
            return response
        return super().post(request, *args, **kwargs)


@require_http_methods(["GET", "POST"])
@api_key_required
def whoami(request: HttpRequest) -> HttpResponse:
    user = request.user
    if not isinstance(user, User):
        return unauthorized("invalid_token", "The token is invalid, expired, or revoked.")
    return JsonResponse({"email": user.email})
```

In `apps/accounts/urls.py`, use `views.RateLimitedLoginView.as_view(...)` instead of `auth_views.LoginView.as_view(...)` for the `login/` route (keeping the same arguments), and add:

```python
path("api/whoami/", views.whoami, name="whoami"),
```

Because the limiter uses the shared cache, call `cache.clear()` in the `setUp` of any test that logs in through the form, otherwise unrelated tests will count against each other's limits.

`apps/accounts/tests/test_api_auth.py`:

```python
import time
from datetime import timedelta
from unittest import mock

from django.core.cache import cache
from django.core.cache.backends.locmem import LocMemCache
from django.test import Client, SimpleTestCase, TestCase, override_settings
from django.urls import reverse
from django.utils import timezone

from apps.accounts.api_keys import (
    authenticate_api_key,
    create_api_key,
    hash_secret,
    revoke_api_key,
)
from apps.accounts.models import ApiKey, User
from apps.accounts.ratelimit import RateLimiter, RateRule

PASSWORD = "correct-horse-battery-staple"
FAST_HASHER = ["django.contrib.auth.hashers.MD5PasswordHasher"]


def bearer(token: str) -> dict[str, str]:
    return {"Authorization": f"Bearer {token}"}


class ApiKeyTests(TestCase):
    def setUp(self) -> None:
        self.user = User.objects.create_user("ada@example.com", PASSWORD)

    def test_only_a_hash_is_stored(self) -> None:
        key, token = create_api_key(user=self.user, name="deploy script")
        _, _, secret = token.split("_", 2)
        self.assertTrue(token.startswith(f"bk_{key.prefix}_"))
        self.assertEqual(key.secret_hash, hash_secret(secret))
        self.assertNotIn(secret, key.secret_hash)
        self.assertFalse(ApiKey.objects.filter(secret_hash=secret).exists())

    def test_a_valid_token_authenticates_and_records_use(self) -> None:
        key, token = create_api_key(user=self.user, name="script")
        now = timezone.now()
        found = authenticate_api_key(token, now=now)
        self.assertEqual(found, key)
        key.refresh_from_db()
        self.assertEqual(key.last_used_at, now)

    def test_last_used_is_not_rewritten_on_every_request(self) -> None:
        key, token = create_api_key(user=self.user, name="script")
        first = timezone.now()
        authenticate_api_key(token, now=first)
        authenticate_api_key(token, now=first + timedelta(seconds=10))
        key.refresh_from_db()
        self.assertEqual(key.last_used_at, first)
        authenticate_api_key(token, now=first + timedelta(seconds=61))
        key.refresh_from_db()
        self.assertEqual(key.last_used_at, first + timedelta(seconds=61))

    def test_malformed_wrong_and_unknown_tokens_are_rejected(self) -> None:
        key, token = create_api_key(user=self.user, name="script")
        wrong_secret = f"bk_{key.prefix}_not-the-secret"
        for bad in ("", "bk", "bk_x", "xx_abcdef12_secret", f"bk_{key.prefix}_", wrong_secret, "bk_00000000_secret"):
            with self.subTest(token=bad):
                self.assertIsNone(authenticate_api_key(bad))
        self.assertIsNotNone(authenticate_api_key(token))

    def test_expired_revoked_and_inactive_keys_are_rejected(self) -> None:
        now = timezone.now()
        key, token = create_api_key(user=self.user, name="short", lifetime=timedelta(hours=1), now=now)
        self.assertIsNotNone(authenticate_api_key(token, now=now + timedelta(minutes=59)))
        self.assertIsNone(authenticate_api_key(token, now=now + timedelta(hours=1)))   # exactly at expiry
        self.assertIsNone(authenticate_api_key(token, now=now + timedelta(hours=2)))

        revoked, revoked_token = create_api_key(user=self.user, name="revoked")
        revoke_api_key(revoked)
        self.assertIsNone(authenticate_api_key(revoked_token))

        _, inactive_token = create_api_key(user=self.user, name="inactive")
        self.user.is_active = False
        self.user.save(update_fields=["is_active"])
        self.assertIsNone(authenticate_api_key(inactive_token))

    def test_creation_is_validated(self) -> None:
        with self.assertRaises(ValueError):
            create_api_key(user=self.user, name="   ")
        with self.assertRaises(ValueError):
            create_api_key(user=self.user, name="x" * 101)
        with self.assertRaises(ValueError):
            create_api_key(user=self.user, name="ok", lifetime=timedelta(0))

    def test_a_prefix_collision_is_retried(self) -> None:
        with mock.patch(
            "apps.accounts.api_keys.secrets.token_hex",
            side_effect=["aaaaaaaa", "aaaaaaaa", "bbbbbbbb"],
        ):
            first, _ = create_api_key(user=self.user, name="one")
            second, _ = create_api_key(user=self.user, name="two")
        self.assertEqual((first.prefix, second.prefix), ("aaaaaaaa", "bbbbbbbb"))

    def test_giving_up_after_repeated_collisions(self) -> None:
        with mock.patch("apps.accounts.api_keys.secrets.token_hex", return_value="aaaaaaaa"):
            create_api_key(user=self.user, name="one")
            with self.assertRaises(RuntimeError):
                create_api_key(user=self.user, name="two")


@override_settings(PASSWORD_HASHERS=FAST_HASHER)
class ApiViewTests(TestCase):
    def setUp(self) -> None:
        self.user = User.objects.create_user("ada@example.com", PASSWORD)
        self.url = reverse("accounts:whoami")

    def test_a_valid_token_returns_the_user(self) -> None:
        _, token = create_api_key(user=self.user, name="script")
        response = self.client.get(self.url, headers=bearer(token))
        self.assertEqual(response.json(), {"email": "ada@example.com"})

    def test_missing_headers_and_other_schemes_are_401(self) -> None:
        for headers in ({}, {"Authorization": "Basic abc"}, {"Authorization": "Bearer"}):
            with self.subTest(headers=headers):
                response = self.client.get(self.url, headers=headers)
                self.assertEqual(response.status_code, 401)
                self.assertEqual(response.json()["error"]["code"], "missing_token")
                self.assertEqual(response["WWW-Authenticate"], 'Bearer realm="api"')

    def test_all_token_problems_look_identical_to_the_caller(self) -> None:
        revoked, revoked_token = create_api_key(user=self.user, name="revoked")
        revoke_api_key(revoked)
        responses = [
            self.client.get(self.url, headers=bearer(token))
            for token in (revoked_token, "bk_00000000_unknown", "garbage")
        ]
        self.assertEqual({response.status_code for response in responses}, {401})
        self.assertEqual(len({response.content for response in responses}), 1)

    def test_session_cookies_do_not_authenticate_api_calls(self) -> None:
        self.client.force_login(self.user)
        self.assertEqual(self.client.get(self.url).status_code, 401)

    def test_token_requests_do_not_need_a_csrf_token(self) -> None:
        _, token = create_api_key(user=self.user, name="script")
        strict = Client(enforce_csrf_checks=True)
        self.assertEqual(strict.post(self.url, headers=bearer(token)).status_code, 200)


class RateLimiterTests(SimpleTestCase):
    def setUp(self) -> None:
        self.cache = LocMemCache("ratelimit-tests", {})
        self.cache.clear()
        self.limiter = RateLimiter("test", RateRule(limit=2, window_seconds=60), cache=self.cache)

    def test_attempts_beyond_the_limit_are_refused(self) -> None:
        self.assertEqual([self.limiter.hit("a") for _ in range(4)], [True, True, False, False])

    def test_identifiers_are_independent(self) -> None:
        self.limiter.hit("a")
        self.limiter.hit("a")
        self.assertFalse(self.limiter.hit("a"))
        self.assertTrue(self.limiter.hit("b"))

    def test_the_window_resets_after_it_expires(self) -> None:
        self.limiter.hit("a")
        self.limiter.hit("a")
        self.assertFalse(self.limiter.hit("a"))
        later = time.time() + 120
        with mock.patch("time.time", return_value=later):
            self.assertTrue(self.limiter.hit("a"))

    def test_the_rule_is_validated(self) -> None:
        with self.assertRaises(ValueError):
            RateRule(limit=0, window_seconds=60)


@override_settings(PASSWORD_HASHERS=FAST_HASHER)
class LoginRateLimitTests(TestCase):
    def setUp(self) -> None:
        cache.clear()
        User.objects.create_user("ada@example.com", PASSWORD)
        self.url = reverse("accounts:login")

    def attempt(self, email: str, password: str) -> int:
        return self.client.post(self.url, {"username": email, "password": password}).status_code

    def test_the_sixth_attempt_is_blocked_even_with_the_right_password(self) -> None:
        for _ in range(5):
            self.assertEqual(self.attempt("ada@example.com", "wrong"), 200)
        blocked = self.client.post(self.url, {"username": "ada@example.com", "password": PASSWORD})
        self.assertEqual(blocked.status_code, 429)
        self.assertEqual(blocked["Retry-After"], "300")

    def test_other_accounts_are_not_affected(self) -> None:
        for _ in range(6):
            self.attempt("ada@example.com", "wrong")
        self.assertEqual(self.attempt("someone@example.com", "wrong"), 200)

    def test_a_correct_login_within_the_limit_works(self) -> None:
        self.assertEqual(self.attempt("ada@example.com", PASSWORD), 302)
```

The tests pin down the properties that matter for credentials: only hashes are stored, expiry and revocation work, every failure looks the same to the caller, cookies cannot authenticate the API, and the login limiter blocks guessing without harming other users.

## Common mistakes

- Storing API tokens in plain text, or showing them again after creation.
- Using `random` instead of `secrets` for tokens.
- Comparing tokens or hashes with `==` instead of a constant-time comparison.
- Telling the caller why a token failed (expired versus revoked versus unknown), which helps attackers.
- Accepting JWTs without pinning algorithms and checking issuer, audience, and expiry.
- Using long-lived JWTs with no way to revoke them.
- Storing tokens in `localStorage` where any injected script can read them.
- Automatically linking a social login to an existing account by an unverified email address.
- Skipping the `state` parameter or registering wildcard redirect URIs.
- Locking accounts out purely by username so attackers can lock real users out.
- Reading client addresses from headers that clients can forge.
- Rate limiting only the web form and leaving the API login or password reset open.
- Writing a database row (`last_used_at`) on every API request at high volume.
- Treating SMS as a strong second factor, or offering a second factor with no recovery path.

## Practice exercises

1. Add a `scopes` field to `ApiKey` (a list of strings), a `scopes=` argument to `api_key_required`, and a test that a key without the needed scope receives a 403 with a stable error code.
2. Write a management command `create_api_key <email> <name> --days N` that prints the raw token exactly once and refuses to run for inactive users, with tests using `call_command`.
3. Write a view that lists the current user's API keys (name, prefix, created, last used, status) and a POST endpoint to revoke one, making sure users can revoke only their own keys. Test the case where a user tries to revoke someone else's key.
4. Replace the fixed-window limiter with a sliding-window version that stores timestamps in the cache, and write a test showing the double-rate burst at a window boundary that the fixed window allows and the sliding window prevents.
5. Install `django-axes`, configure it as in this note, and write a test that five failed logins from one address for one username lock that pair while another username from the same address can still log in.
6. Add a `RateLimiter.remaining(identifier)` method that returns how many attempts are left in the current window without counting a new attempt, and a `retry_after(identifier)` estimate. Test both, including after the window expires.
7. Design and implement an email-based "magic link" login: a signed, single-use, 15-minute token sent by email, with the same response whether or not the address has an account, and a rate limit per address. Write tests for success, reuse, expiry, and enumeration.

## Summary

Beyond passwords, applications use social login, tokens, second factors, and rate limits. `django-allauth` handles registration, email verification, and OAuth 2.0 and OpenID Connect providers: the authorization code flow with PKCE and a checked `state`, exact redirect URIs, and cautious account linking. For programs, opaque API tokens that are generated with `secrets`, shown once, stored only as hashes, compared in constant time, expiring, and revocable are simple and safe, and requests that authenticate only by a bearer header are not exposed to CSRF. JWTs fit short-lived, centrally issued tokens verified with pinned algorithms and strict claim checks, while sessions remain the better choice for a browser application on your own domain. Add second factors (TOTP, hardware keys, passkeys), combine proxy limits, application rate limits, and `django-axes` against brute force, and keep responses and timing identical for known and unknown accounts so your application does not reveal who has an account.
