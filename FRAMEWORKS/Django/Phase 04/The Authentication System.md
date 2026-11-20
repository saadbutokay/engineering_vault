## Overview

Authentication answers one question: who is making this request? Django ships a complete answer: a user model, password storage with strong hashing, login and logout, views for password changes and resets, decorators and middleware that require a login, and a pluggable backend system. Using it correctly means making several decisions early, and one of them, the user model, is painful to change later. This note covers the `User` model, `authenticate`, `login`, and `logout`, `LoginView` and `LogoutView`, requiring logins (`login_required`, `LoginRequiredMixin`, and `LoginRequiredMiddleware` from Django 5.1), password hashing and validation, the password reset flow, and why every project should define a custom user model before its first migration, using email as the login field. It ends with a complete accounts app: a custom user model, sign-up, login with email, logout, password change and reset, and tests for each flow.

## Core concepts

### Authentication and authorization

Authentication establishes identity (a user proved who they are, usually with a password). Authorization decides what that identity may do (note 4.3). Django keeps them separate: this note covers the first.

### The pieces

- The user model stores identities and password hashes.
- Backends verify credentials. The default `ModelBackend` checks a username and password against the database.
- `authenticate(request, **credentials)` asks each backend to verify credentials and returns a user or `None`.
- `login(request, user)` records the user in the session (rotating the session key).
- `logout(request)` flushes the session.
- `request.user` is the current user on every request, set by `AuthenticationMiddleware`. It is an `AnonymousUser` for visitors who are not logged in.

### Login, logout, and the built-in views

`LoginView` shows a form and logs the user in. `LogoutView` logs out. Since Django 5.0, logging out requires a POST request, because a GET that changes state can be triggered by any image tag on any website. The built-in `django.contrib.auth.views` also include `PasswordChangeView`, `PasswordResetView`, `PasswordResetConfirmView`, and their "done" pages.

### Requiring a login

- `@login_required` on function views.
- `LoginRequiredMixin` on class-based views (listed first among the base classes).
- `LoginRequiredMiddleware` (Django 5.1, available on 5.2 LTS) protects every view by default. You then opt public views out with `@login_not_required`. This reverses the failure mode: a forgotten decorator makes a page private instead of public.

All three redirect anonymous visitors to `settings.LOGIN_URL` with a `next` parameter holding the page they wanted.

### Password hashing

Django never stores passwords. It stores a salted, deliberately slow hash in a self-describing string such as `pbkdf2_sha256$870000$salt$hash`. The first part names the algorithm, so old hashes keep working when you change the default, and Django upgrades a user's hash to the preferred algorithm at their next successful login. The first entry of `PASSWORD_HASHERS` is used for new passwords. PBKDF2 is the default, and Argon2 (`uv add "django[argon2]"`) is the stronger recommendation because it is memory-hard, which makes large-scale cracking on GPUs expensive.

### Password validators

`AUTH_PASSWORD_VALIDATORS` lists checks applied when users choose passwords: similarity to their other attributes, minimum length, a list of common passwords, and all-numeric passwords. They apply in forms and views that call `validate_password()`. They are not enforced by `set_password()` itself, so your own code must call them.

### The password reset flow

1. The user enters an email address on a form.
2. If an account exists, Django emails a link containing the user's id and a token.
3. The link leads to a form for a new password.
4. The token is an HMAC over the user's id, password hash, last login time, and a timestamp, so it works once (using it changes the password hash) and expires after `PASSWORD_RESET_TIMEOUT` seconds.

The first step always responds the same way whether or not the address exists, so the form cannot be used to discover accounts.

### Why a custom user model, before the first migration

Django lets you swap the user model through `AUTH_USER_MODEL`, but only cleanly at the start of a project. After the first migration, other tables reference the built-in model and swapping requires hand-written, risky migrations. Even if the default model looks sufficient today, nearly every project later wants an email login, a different set of profile fields, or a different identifier. Defining your own model on day one costs a few lines and removes that risk.

### AbstractUser versus AbstractBaseUser

- `AbstractUser` keeps Django's fields (`username`, `first_name`, `last_name`, `email`, and flags) and lets you add more. Use it for minimal change.
- `AbstractBaseUser` provides only password handling and `last_login`, and you define everything else. Combined with `PermissionsMixin` it gives groups and permissions. Use it when you want a different identifier such as email.

### Email as the login field

Set `USERNAME_FIELD = "email"`, make the email unique, and write a manager whose `create_user` and `create_superuser` take an email. Treat emails as case-insensitive by storing them in lowercase and looking them up in lowercase, because `Ada@Example.com` and `ada@example.com` must not be two accounts.

### Referencing the user model

Never import the user class directly in reusable code. Use `settings.AUTH_USER_MODEL` (a string) in model relations, and `get_user_model()` in functions. In your own accounts app you may import the class directly.

## Basic syntax examples

### authenticate, login, and logout by hand

```python
from django.contrib.auth import authenticate, login, logout
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect, render


def sign_in(request: HttpRequest) -> HttpResponse:
    error = ""
    if request.method == "POST":
        user = authenticate(
            request,
            username=request.POST.get("username", ""),
            password=request.POST.get("password", ""),
        )
        if user is not None:
            login(request, user)
            return redirect("home")
        error = "Invalid credentials."
    return render(request, "sign_in.html", {"error": error})


def sign_out(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        logout(request)
    return redirect("home")
```

### The built-in views in URLs

```python
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path("login/", auth_views.LoginView.as_view(), name="login"),
    path("logout/", auth_views.LogoutView.as_view(), name="logout"),
]
```

### Logging out with a POST form

```html
<form method="post" action="{% url 'accounts:logout' %}">
  {% csrf_token %}
  <button type="submit">Log out</button>
</form>
```

### Requiring a login

```python
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin
from django.http import HttpRequest, HttpResponse
from django.views.generic import TemplateView


@login_required
def dashboard(request: HttpRequest) -> HttpResponse:
    return HttpResponse(f"Hello, {request.user}")


class AccountView(LoginRequiredMixin, TemplateView):
    template_name = "accounts/account.html"
```

### Protecting the whole site with the middleware

```python
MIDDLEWARE = [
    # ...
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.auth.middleware.LoginRequiredMiddleware",
    # ...
]
```

```python
from django.contrib.auth.decorators import login_not_required


@login_not_required
def health(request: HttpRequest) -> HttpResponse:
    return HttpResponse("ok")
```

### Login-related settings

```python
AUTH_USER_MODEL = "accounts.User"
LOGIN_URL = "accounts:login"
LOGIN_REDIRECT_URL = "bookstore:search"
LOGOUT_REDIRECT_URL = "accounts:login"
```

### Working with passwords

```python
from django.contrib.auth.hashers import check_password, make_password

hashed = make_password("correct-horse-battery-staple")
print(hashed)                                    # 'pbkdf2_sha256$...' or 'argon2$...'
print(check_password("correct-horse-battery-staple", hashed))   # True

user.set_password("new passphrase here")         # hashes, does not save
user.save(update_fields=["password"])
user.set_unusable_password()                     # for accounts that cannot log in with a password
```

### Password hasher and validator settings

```python
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",         # used for new passwords
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",         # still verifies older hashes
    "django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher",
]

AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
        "OPTIONS": {"min_length": 12},
    },
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]
```

### Validating a password in code

```python
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError

try:
    validate_password("12345678", user=user)
except ValidationError as error:
    print(error.messages)
```

### Password reset settings for development

```python
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"   # prints emails in the terminal
PASSWORD_RESET_TIMEOUT = 60 * 60                                   # one hour instead of three days
```

## Deeper behavior

### What authenticate does

`authenticate()` loops over `AUTHENTICATION_BACKENDS`, calls each backend's `authenticate(request, **credentials)`, and returns the first user found, after attaching the backend's dotted path to the user as `user.backend`. If a backend raises `PermissionDenied`, the loop stops and returns `None`. If every backend returns `None`, the credentials are invalid. `ModelBackend` also rejects inactive users (`is_active=False`).

### What login does

`login(request, user, backend=None)` stores the user's primary key, the backend path, and the password hash in the session, rotates the session key (note 4.1), updates `last_login` through the `user_logged_in` signal, and attaches the user to `request.user`. When more than one backend is configured, `login()` needs to know which one to record: pass `backend=` or call it with a user returned by `authenticate()`. For a user you just created, pass the backend explicitly.

### request.user is lazy

`AuthenticationMiddleware` sets `request.user` to a lazy object. The database is queried only when your code touches an attribute, so pages that never look at the user do not pay for it. The lookup is cached for the request.

### Logout and the session

`logout()` calls `session.flush()`, deleting the data and cookie. It sends `user_logged_out`. It does nothing about other devices; see note 4.1 for ending every session.

### Password hash upgrades

When a user logs in with a hash created by a non-preferred algorithm or with fewer iterations than the current default, Django rehashes the password with the preferred settings and saves it. This is why adding Argon2 to the front of `PASSWORD_HASHERS` gradually upgrades active users without a migration. Users who never log in keep their old hash, which Django can still verify. Keep old hashers in the list as long as such hashes exist.

### Why hashes are slow on purpose

A fast hash lets an attacker who steals the database test billions of guesses per second. A deliberately slow, salted hash makes each guess expensive and prevents precomputed tables. The cost is CPU time on every login and in tests, so test settings often use a fast hasher (such as `MD5PasswordHasher`) to speed up the suite, and never production.

### Constant-time checking

`check_password()` compares hashes in constant time, so response timing does not reveal how many characters matched. If you write your own comparisons of secrets, use `hmac.compare_digest`.

### Validators are not automatic

`set_password()` and `create_user()` do not run validators, and neither does the admin's "set password" shortcut in every path. Forms and views must call `validate_password()`, and API endpoints that accept passwords must too. The best place is a single function that every path calls, as in the example below.

### Reset tokens

The token generator hashes the user's primary key, current password hash, last login time, and a timestamp with `SECRET_KEY`. Changing the password or logging in changes the hash input, which invalidates old links, and nothing about the token is stored in the database. Three days is the default lifetime. One hour is a better choice for most applications. The reset view always shows the same "email sent" page, and it sends nothing when no account matches.

### Reset links and the Host header

Reset emails build the link from the request's host, so `ALLOWED_HOSTS` must be correct (note 0.5), otherwise an attacker who can forge the `Host` header could make emails point to a hostile site.

### The custom model, step by step

- `USERNAME_FIELD` names the login identifier, and Django's checks require it to be unique.
- `REQUIRED_FIELDS` lists fields that `createsuperuser` prompts for, besides the identifier and password. Do not include the identifier or password.
- The manager must implement `create_user` and `create_superuser`, because `createsuperuser` calls them.
- `get_by_natural_key()` is how `ModelBackend` finds users by login value. Overriding it to lowercase input makes logins case-insensitive.
- `is_staff` controls admin access, `is_superuser` grants all permissions, and `is_active` disables login without deleting the account.

### Starting over in a practice project

You cannot switch to a custom user model in a database that already contains the default model's migrations (`InconsistentMigrationHistory`). In a project with no data to keep, add the `accounts` app and `AUTH_USER_MODEL`, reset the development database (`docker compose down -v`, then `docker compose up -d`), and run `makemigrations` and `migrate`. Your earlier bookstore migrations do not reference the user model, so they stay as they are. In a real project with production data, follow the careful procedure in the Django documentation or avoid the problem by starting right.

### Open redirects through next

`LoginView` redirects to the `next` parameter after login. It validates the target with `url_has_allowed_host_and_scheme`, accepting only URLs on your own hosts. If you write your own redirect-after-login code, apply the same check.

### Email verification

Anyone can type any address at sign-up. If your application trusts the address (for password resets, for linking social accounts, for sending data), verify it first by emailing a link. The `django-allauth` package provides verification flows, and note 4.4 covers it.

## Industry-level snippet

A complete accounts app: a custom user model with email login and case-insensitive uniqueness, a password strength check shared by the form and the service, a registration service that turns a database uniqueness error into a clear exception, a sign-up view that logs the new user in, the built-in views wired into a namespace with one generic template, and tests for the main flows.

`apps/accounts/models.py`:

```python
from typing import Any

from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models
from django.db.models.functions import Lower
from django.utils import timezone


class UserManager(BaseUserManager["User"]):
    use_in_migrations = True

    def _create_user(self, email: str, password: str | None, **extra_fields: Any) -> "User":
        cleaned = self.normalize_email(email).strip().lower()
        if not cleaned:
            raise ValueError("An email address is required")
        user = self.model(email=cleaned, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_user(
        self, email: str, password: str | None = None, **extra_fields: Any
    ) -> "User":
        extra_fields.setdefault("is_staff", False)
        extra_fields.setdefault("is_superuser", False)
        return self._create_user(email, password, **extra_fields)

    def create_superuser(
        self, email: str, password: str | None = None, **extra_fields: Any
    ) -> "User":
        extra_fields.setdefault("is_staff", True)
        extra_fields.setdefault("is_superuser", True)
        if extra_fields["is_staff"] is not True:
            raise ValueError("Superuser must have is_staff=True")
        if extra_fields["is_superuser"] is not True:
            raise ValueError("Superuser must have is_superuser=True")
        return self._create_user(email, password, **extra_fields)

    def get_by_natural_key(self, username: str | None) -> "User":
        """Look up by email, ignoring case, so logins are case-insensitive."""
        return self.get(email=(username or "").strip().lower())


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField("email address", max_length=254, unique=True)
    full_name = models.CharField(max_length=150, blank=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(default=timezone.now)

    objects = UserManager()

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS: list[str] = []

    class Meta:
        constraints = [
            # Catches mixed-case rows written without going through the manager.
            models.UniqueConstraint(Lower("email"), name="accounts_user_email_ci_unique"),
        ]

    def __str__(self) -> str:
        return self.email

    def clean(self) -> None:
        super().clean()
        self.email = self.email.strip().lower()

    def get_full_name(self) -> str:
        return self.full_name or self.email

    def get_short_name(self) -> str:
        return self.full_name.split(" ")[0] if self.full_name else self.email
```

Settings:

```python
INSTALLED_APPS = [
    # ...
    "apps.accounts",
]

AUTH_USER_MODEL = "accounts.User"
LOGIN_URL = "accounts:login"
LOGIN_REDIRECT_URL = "bookstore:search"
LOGOUT_REDIRECT_URL = "accounts:login"
PASSWORD_RESET_TIMEOUT = 60 * 60
```

`apps/accounts/services.py`:

```python
"""Account registration rules shared by every entry point."""

import logging

from django.contrib.auth import password_validation
from django.db import IntegrityError, transaction

from apps.accounts.models import User

logger = logging.getLogger(__name__)


class EmailAlreadyRegisteredError(Exception):
    """An account with this email address already exists."""


def check_password_strength(password: str, *, email: str, full_name: str = "") -> None:
    """Raise ``ValidationError`` listing every failed password validator."""
    candidate = User(email=email, full_name=full_name)  # unsaved, for similarity checks
    password_validation.validate_password(password, candidate)


def register_user(*, email: str, password: str, full_name: str = "") -> User:
    """Create an account.

    Raises:
        ValidationError: if the password fails the configured validators.
        EmailAlreadyRegisteredError: if the email (ignoring case) is taken.
    """
    cleaned = email.strip().lower()
    check_password_strength(password, email=cleaned, full_name=full_name)
    try:
        with transaction.atomic():
            user = User.objects.create_user(email=cleaned, password=password, full_name=full_name)
    except IntegrityError as exc:
        if User.objects.filter(email=cleaned).exists():
            raise EmailAlreadyRegisteredError(cleaned) from exc
        raise
    logger.info("Registered user %s", user.pk)
    return user
```

`apps/accounts/forms.py`:

```python
from typing import Any

from django import forms
from django.contrib.auth import password_validation
from django.contrib.auth.forms import AuthenticationForm
from django.core.exceptions import ValidationError

from apps.accounts.services import check_password_strength


class EmailAuthenticationForm(AuthenticationForm):
    """The built-in login form, labeled and tuned for email addresses."""

    def __init__(self, *args: Any, **kwargs: Any) -> None:
        super().__init__(*args, **kwargs)
        self.fields["username"].label = "Email"
        self.fields["username"].widget.attrs.update(
            {"autocomplete": "email", "autofocus": True, "inputmode": "email"}
        )


class SignUpForm(forms.Form):
    email = forms.EmailField(max_length=254)
    full_name = forms.CharField(max_length=150, required=False, strip=True)
    password1 = forms.CharField(
        label="Password",
        strip=False,
        widget=forms.PasswordInput(attrs={"autocomplete": "new-password"}),
        help_text=password_validation.password_validators_help_text_html(),
    )
    password2 = forms.CharField(
        label="Password confirmation",
        strip=False,
        widget=forms.PasswordInput(attrs={"autocomplete": "new-password"}),
    )

    def clean_email(self) -> str:
        return str(self.cleaned_data["email"]).strip().lower()

    def clean(self) -> dict[str, Any]:
        cleaned = super().clean()
        password1 = cleaned.get("password1")
        password2 = cleaned.get("password2")
        if password1 and password2 and password1 != password2:
            self.add_error(
                "password2",
                ValidationError("The two password fields didn't match.", code="password_mismatch"),
            )
        elif password1:
            try:
                check_password_strength(
                    password1,
                    email=cleaned.get("email", ""),
                    full_name=cleaned.get("full_name", ""),
                )
            except ValidationError as error:
                self.add_error("password1", error)
        return cleaned
```

`apps/accounts/views.py`:

```python
from django.conf import settings
from django.contrib.auth import login
from django.contrib.auth.decorators import login_required
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect, render

from apps.accounts.forms import SignUpForm
from apps.accounts.services import EmailAlreadyRegisteredError, register_user

MODEL_BACKEND = "django.contrib.auth.backends.ModelBackend"


def signup(request: HttpRequest) -> HttpResponse:
    if request.user.is_authenticated:
        return redirect(settings.LOGIN_REDIRECT_URL)
    form = SignUpForm(request.POST or None)
    if request.method == "POST" and form.is_valid():
        try:
            user = register_user(
                email=form.cleaned_data["email"],
                password=form.cleaned_data["password1"],
                full_name=form.cleaned_data["full_name"],
            )
        except EmailAlreadyRegisteredError:
            form.add_error("email", "An account with this email already exists.")
        else:
            login(request, user, backend=MODEL_BACKEND)
            return redirect(settings.LOGIN_REDIRECT_URL)
    return render(request, "accounts/form_page.html", {"form": form, "title": "Create an account"})


@login_required
def account(request: HttpRequest) -> HttpResponse:
    return HttpResponse(f"Signed in as {request.user.email}")
```

`apps/accounts/urls.py`:

```python
from django.contrib.auth import views as auth_views
from django.urls import path, reverse_lazy

from apps.accounts import views
from apps.accounts.forms import EmailAuthenticationForm

app_name = "accounts"

FORM_PAGE = "accounts/form_page.html"
MESSAGE_PAGE = "accounts/message_page.html"

urlpatterns = [
    path("signup/", views.signup, name="signup"),
    path("account/", views.account, name="account"),
    path(
        "login/",
        auth_views.LoginView.as_view(
            template_name=FORM_PAGE,
            authentication_form=EmailAuthenticationForm,
            redirect_authenticated_user=True,
        ),
        name="login",
    ),
    path("logout/", auth_views.LogoutView.as_view(), name="logout"),
    path(
        "password-change/",
        auth_views.PasswordChangeView.as_view(
            template_name=FORM_PAGE,
            success_url=reverse_lazy("accounts:password_change_done"),
        ),
        name="password_change",
    ),
    path(
        "password-change/done/",
        auth_views.PasswordChangeDoneView.as_view(template_name=MESSAGE_PAGE),
        name="password_change_done",
    ),
    path(
        "password-reset/",
        auth_views.PasswordResetView.as_view(
            template_name=FORM_PAGE,
            email_template_name="accounts/password_reset_email.txt",
            subject_template_name="accounts/password_reset_subject.txt",
            success_url=reverse_lazy("accounts:password_reset_done"),
        ),
        name="password_reset",
    ),
    path(
        "password-reset/done/",
        auth_views.PasswordResetDoneView.as_view(template_name=MESSAGE_PAGE),
        name="password_reset_done",
    ),
    path(
        "reset/<uidb64>/<token>/",
        auth_views.PasswordResetConfirmView.as_view(
            template_name="accounts/password_reset_confirm.html",
            success_url=reverse_lazy("accounts:password_reset_complete"),
        ),
        name="password_reset_confirm",
    ),
    path(
        "reset/done/",
        auth_views.PasswordResetCompleteView.as_view(template_name=MESSAGE_PAGE),
        name="password_reset_complete",
    ),
]
```

Mount with `path("accounts/", include("apps.accounts.urls"))` in `config/urls.py`.

Templates (`apps/accounts/templates/accounts/`):

`form_page.html`, one template for every form page, because the auth views supply a `title`:

```html
<h1>{{ title }}</h1>
<form method="post">
  {% csrf_token %}
  {% if next %}<input type="hidden" name="next" value="{{ next }}">{% endif %}
  {{ form }}
  <button type="submit">Continue</button>
</form>
```

`message_page.html`:

```html
<h1>{{ title }}</h1>
<p><a href="{% url 'accounts:login' %}">Back to log in</a></p>
```

`password_reset_confirm.html`:

```html
{% if validlink %}
  <h1>Choose a new password</h1>
  <form method="post">
    {% csrf_token %}
    {{ form }}
    <button type="submit">Change password</button>
  </form>
{% else %}
  <h1>Link not valid</h1>
  <p>This reset link is invalid or has expired. <a href="{% url 'accounts:password_reset' %}">Request a new one</a>.</p>
{% endif %}
```

`password_reset_subject.txt` (one line only):

```text
Reset your password
```

`password_reset_email.txt`:

```text
{% autoescape off %}You asked to reset your password. Open this link to choose a new one:

{{ protocol }}://{{ domain }}{% url 'accounts:password_reset_confirm' uidb64=uid token=token %}

If you did not ask for this, you can ignore this message.
{% endautoescape %}
```

`apps/accounts/tests/test_authentication.py`:

```python
import re

from django.contrib.auth import authenticate
from django.core import mail
from django.db import IntegrityError, transaction
from django.http import HttpResponse
from django.test import Client, TestCase, override_settings
from django.urls import reverse

from apps.accounts.models import User

PASSWORD = "correct-horse-battery-staple"
NEW_PASSWORD = "another-long-passphrase-9"
FAST_HASHER = ["django.contrib.auth.hashers.MD5PasswordHasher"]  # tests only


@override_settings(PASSWORD_HASHERS=FAST_HASHER)
class UserModelTests(TestCase):
    def test_email_is_normalized_and_the_password_is_hashed(self) -> None:
        with override_settings(
            PASSWORD_HASHERS=["django.contrib.auth.hashers.PBKDF2PasswordHasher"]
        ):
            user = User.objects.create_user("  Ada@Example.COM ", PASSWORD)
        self.assertEqual(user.email, "ada@example.com")
        self.assertNotEqual(user.password, PASSWORD)
        self.assertTrue(user.password.startswith("pbkdf2_sha256$"))
        self.assertTrue(user.check_password(PASSWORD))

    def test_a_blank_email_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            User.objects.create_user("   ", PASSWORD)

    def test_emails_are_unique_ignoring_case_at_the_database_level(self) -> None:
        User.objects.create_user("ada@example.com", PASSWORD)
        with self.assertRaises(IntegrityError), transaction.atomic():
            User.objects.create(email="ADA@example.com")   # bypasses the manager

    def test_superuser_flags(self) -> None:
        admin = User.objects.create_superuser("root@example.com", PASSWORD)
        self.assertTrue(admin.is_staff and admin.is_superuser)
        with self.assertRaises(ValueError):
            User.objects.create_superuser("x@example.com", PASSWORD, is_staff=False)

    def test_login_lookup_ignores_case(self) -> None:
        user = User.objects.create_user("ada@example.com", PASSWORD)
        self.assertEqual(authenticate(username="ADA@Example.com", password=PASSWORD), user)

    def test_wrong_password_and_inactive_accounts_do_not_authenticate(self) -> None:
        user = User.objects.create_user("ada@example.com", PASSWORD)
        self.assertIsNone(authenticate(username="ada@example.com", password="nope"))
        user.is_active = False
        user.save(update_fields=["is_active"])
        self.assertIsNone(authenticate(username="ada@example.com", password=PASSWORD))


@override_settings(PASSWORD_HASHERS=FAST_HASHER)
class SignUpTests(TestCase):
    def post(self, **overrides: str) -> HttpResponse:
        data = {
            "email": "Ada@Example.com",
            "full_name": "Ada Lovelace",
            "password1": PASSWORD,
            "password2": PASSWORD,
        }
        data.update(overrides)
        return self.client.post(reverse("accounts:signup"), data)

    def test_successful_signup_creates_the_user_and_logs_in(self) -> None:
        response = self.post()
        self.assertRedirects(response, reverse("bookstore:search"), fetch_redirect_response=False)
        user = User.objects.get()
        self.assertEqual(user.email, "ada@example.com")
        self.assertEqual(self.client.session["_auth_user_id"], str(user.pk))

    def test_weak_and_mismatched_passwords_are_rejected(self) -> None:
        weak = self.post(password1="123456789012", password2="123456789012")
        self.assertContains(weak, "entirely numeric")
        mismatch = self.post(password2="different-passphrase-1")
        self.assertContains(mismatch, "didn")
        self.assertEqual(User.objects.count(), 0)

    def test_duplicate_emails_are_rejected_ignoring_case(self) -> None:
        User.objects.create_user("ada@example.com", PASSWORD)
        response = self.post(email="ADA@example.com")
        self.assertContains(response, "already exists")
        self.assertEqual(User.objects.count(), 1)

    def test_logged_in_users_are_sent_away_from_signup(self) -> None:
        self.post()
        response = self.client.get(reverse("accounts:signup"))
        self.assertEqual(response.status_code, 302)


@override_settings(PASSWORD_HASHERS=FAST_HASHER)
class LoginLogoutTests(TestCase):
    def setUp(self) -> None:
        self.user = User.objects.create_user("ada@example.com", PASSWORD)
        self.login_url = reverse("accounts:login")

    def test_login_with_the_email_in_any_case(self) -> None:
        response = self.client.post(self.login_url, {"username": "ADA@example.com", "password": PASSWORD})
        self.assertEqual(response.status_code, 302)
        self.assertEqual(self.client.session["_auth_user_id"], str(self.user.pk))

    def test_unknown_email_and_wrong_password_give_the_same_error(self) -> None:
        unknown = self.client.post(self.login_url, {"username": "nobody@example.com", "password": PASSWORD})
        wrong = self.client.post(self.login_url, {"username": "ada@example.com", "password": "bad"})
        self.assertEqual(
            unknown.context["form"].non_field_errors(),
            wrong.context["form"].non_field_errors(),
        )

    def test_next_is_honored_only_for_local_urls(self) -> None:
        credentials = {"username": "ada@example.com", "password": PASSWORD}
        local = self.client.post(f"{self.login_url}?next=/accounts/account/", credentials)
        self.assertEqual(local["Location"], "/accounts/account/")
        self.client.logout()
        outside = self.client.post(f"{self.login_url}?next=https://evil.example/", credentials)
        self.assertEqual(outside["Location"], reverse("bookstore:search"))

    def test_logout_requires_post(self) -> None:
        self.client.force_login(self.user)
        self.assertEqual(self.client.get(reverse("accounts:logout")).status_code, 405)
        self.assertIn("_auth_user_id", self.client.session)
        response = self.client.post(reverse("accounts:logout"))
        self.assertEqual(response.status_code, 302)
        self.assertNotIn("_auth_user_id", self.client.session)

    def test_protected_pages_redirect_anonymous_users_with_next(self) -> None:
        url = reverse("accounts:account")
        response = self.client.get(url)
        self.assertRedirects(response, f"{self.login_url}?next={url}")


@override_settings(PASSWORD_HASHERS=FAST_HASHER)
class PasswordFlowTests(TestCase):
    def setUp(self) -> None:
        self.user = User.objects.create_user("ada@example.com", PASSWORD)

    def test_changing_the_password_keeps_this_session_and_ends_others(self) -> None:
        current, other = Client(), Client()
        current.force_login(self.user)
        other.force_login(self.user)

        response = current.post(
            reverse("accounts:password_change"),
            {"old_password": PASSWORD, "new_password1": NEW_PASSWORD, "new_password2": NEW_PASSWORD},
        )
        self.assertEqual(response.status_code, 302)

        self.assertTrue(current.get("/").wsgi_request.user.is_authenticated)
        self.assertFalse(other.get("/").wsgi_request.user.is_authenticated)

    def test_reset_flow_end_to_end(self) -> None:
        response = self.client.post(reverse("accounts:password_reset"), {"email": "ADA@example.com"})
        self.assertRedirects(response, reverse("accounts:password_reset_done"))
        self.assertEqual(len(mail.outbox), 1)

        match = re.search(r"http://testserver\S+", mail.outbox[0].body)
        assert match is not None
        landing = self.client.get(match.group(0), follow=True)
        set_password_url = landing.redirect_chain[-1][0]

        done = self.client.post(
            set_password_url,
            {"new_password1": NEW_PASSWORD, "new_password2": NEW_PASSWORD},
        )
        self.assertRedirects(done, reverse("accounts:password_reset_complete"))
        self.assertTrue(self.client.login(email="ada@example.com", password=NEW_PASSWORD))

    def test_unknown_emails_get_the_same_response_and_no_mail(self) -> None:
        response = self.client.post(reverse("accounts:password_reset"), {"email": "ghost@example.com"})
        self.assertRedirects(response, reverse("accounts:password_reset_done"))
        self.assertEqual(mail.outbox, [])

    def test_a_garbled_reset_link_shows_the_invalid_page(self) -> None:
        url = reverse("accounts:password_reset_confirm", kwargs={"uidb64": "xx", "token": "bad-token"})
        self.assertContains(self.client.get(url), "Link not valid")
```

The tests cover the decisions that matter: emails are case-insensitive in the manager and at the database level, passwords are hashed, the login error does not reveal whether an account exists, `next` cannot redirect off-site, logout needs POST, and the reset flow works end to end without disclosing which addresses are registered.

## Common mistakes

- Starting a project with the default user model and trying to switch later.
- Using `from django.contrib.auth.models import User` in reusable code instead of `get_user_model()` and `settings.AUTH_USER_MODEL`.
- Treating `set_password()` or `create_user()` as validating the password. They do not.
- Storing passwords in plain text, or using `MD5`, `SHA1`, or an unsalted hash in production.
- Leaving `PASSWORD_RESET_TIMEOUT` at three days, or not shortening it for sensitive applications.
- Letting emails differ only by case, so one person owns several accounts.
- Logging out with a GET link, which any page can trigger.
- Writing `login(request, user)` for a freshly created user when several backends are configured, without passing the backend.
- Telling attackers whether an email exists, in login errors, sign-up forms, or reset forms. (Sign-up forms often must, so protect them with rate limits, note 4.4.)
- Adding `@login_required` to some views and forgetting others. Consider `LoginRequiredMiddleware` so that the default is private.
- Using a fast hasher outside tests.
- Trusting an unverified email address for anything important.

## Practice exercises

1. Create the accounts app from scratch in a fresh project, reset the development database, run `createsuperuser`, and log in to the admin with the email. List every file you had to touch.
2. Add `Argon2PasswordHasher` to the front of `PASSWORD_HASHERS`, log in as a user created with PBKDF2, and show that the stored hash was upgraded. Write a test that checks the algorithm prefix before and after.
3. Add a custom validator to `AUTH_PASSWORD_VALIDATORS` that rejects passwords containing the site name, and test it through the sign-up form and through `check_password_strength`.
4. Enable `LoginRequiredMiddleware`, mark the login, signup, password reset, and a health view as `login_not_required` (using the decorator on function views and `method_decorator` on class-based views), and write a test that fails if a new view is accidentally public.
5. Write a custom authentication backend that accepts either an email or an "employee number" as the login value, returns `None` for anything else, and test it together with `ModelBackend`.
6. Implement a "delete my account" view that requires the password again, logs the user out, and deletes the user. Decide what should happen to the user's orders (note 2.6's `on_delete` choices) and test your decision.
7. Time `check_password` and the login view with `PBKDF2` and `Argon2` for the same password using `time.perf_counter`, and explain in a few sentences why tests use a fast hasher and production must not.

## Summary

Django's authentication system provides a user model, slow salted password hashes that upgrade themselves, `authenticate`, `login`, and `logout`, ready-made login, logout, password change, and password reset views, and three ways to require a login (`login_required`, `LoginRequiredMixin`, and `LoginRequiredMiddleware`). Define a custom user model before the first migration, preferably with email as the unique, case-insensitive login field, and always refer to it through `settings.AUTH_USER_MODEL` and `get_user_model()`. Prefer Argon2, keep old hashers for existing users, apply password validators in every path that sets a password, shorten the reset timeout, require POST for logout, and never reveal which emails have accounts. Test the flows, including the failure paths.
