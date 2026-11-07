## Overview

Everything that can differ between your laptop, a test run, and production lives in settings: secrets, database addresses, debug mode, allowed domains. Django reads these from a Python module. This note explains the important settings, shows how to load values from environment variables safely, and builds a validated, tested configuration loader that fails fast with clear errors instead of misbehaving silently in production.

## Core concepts

### The settings module

`settings.py` is an ordinary Python module. Django reads its uppercase names. The module that Django uses is named by the `DJANGO_SETTINGS_MODULE` environment variable, for example `config.settings`. `manage.py` sets a default for it.

### Key settings

- `BASE_DIR`: a `pathlib.Path` pointing at the project root, used to build other paths.
- `SECRET_KEY`: a long random string used for signing sessions, password reset tokens, and more. If it leaks, attackers can forge them.
- `DEBUG`: when true, Django shows detailed error pages and relaxes some checks. It must be false in production.
- `ALLOWED_HOSTS`: the host names this site may serve. With `DEBUG` off and an empty list, every request fails with `DisallowedHost`.
- `INSTALLED_APPS`: the apps Django loads.
- `DATABASES`: connection details for one or more databases.
- `TEMPLATES`: template engines and their options.

### Configuration from the environment

The twelve-factor approach stores configuration in environment variables, not in code. The same code then runs everywhere, and only the environment changes. Libraries such as `django-environ` and `pydantic-settings` help parse and validate values, and this note also shows how to do it with only the standard library.

### Settings per environment

Common approaches:

- One settings module that reads everything from the environment (simplest, recommended to start).
- A settings package with `base.py`, `local.py`, `production.py`, and `test.py`, selected through `DJANGO_SETTINGS_MODULE`.

Either way, the rule is the same: no environment-specific values hard-coded.

### Secrets never go in the repository

Anything committed to Git stays in its history, even after deletion. Keep `.env` files out of version control, commit only a `.env.example` with fake values, and rotate any secret that has ever been committed.

### Inspection commands

- `manage.py diffsettings` shows how your settings differ from Django's defaults.
- `manage.py check --deploy` audits settings for production safety.

## Basic syntax examples

### The anatomy of settings.py

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = "django-insecure-change-me"  # development only
DEBUG = True
ALLOWED_HOSTS: list[str] = []

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    }
]
```

### Reading from the environment

```python
import os

DEBUG = os.environ.get("DJANGO_DEBUG", "false").lower() == "true"
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]  # KeyError if missing: intentional
ALLOWED_HOSTS = [h for h in os.environ.get("DJANGO_ALLOWED_HOSTS", "").split(",") if h]
```

`os.environ["NAME"]` raises `KeyError` when missing, which is what you want for required secrets. `os.environ.get` supplies a default for optional values.

### A .env file for local development

```text
DJANGO_DEBUG=true
DJANGO_SECRET_KEY=local-only-key-change-me-to-something-long-and-random-1234567890
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgres://app:app@localhost:5432/app
```

A `.env` file is not read automatically by Python. Either export the variables in your shell, use `uv run --env-file .env ...`, or load the file with `django-environ` (`uv add django-environ`).

### Choosing a settings module

```bash
export DJANGO_SETTINGS_MODULE=config.settings
uv run python manage.py check
```

```powershell
$env:DJANGO_SETTINGS_MODULE = "config.settings"
uv run python manage.py check
```

### Generating a secret key

```bash
uv run python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

### Inspecting settings

```bash
uv run python manage.py diffsettings
uv run python manage.py check --deploy
```

## Deeper behavior

### Settings are read lazily

`django.conf.settings` is a lazy object. It loads the settings module the first time any setting is accessed. Always import settings as `from django.conf import settings`, never `from config import settings`, because the former respects overrides in tests and the latter bypasses them.

### Why the `DEBUG` default must be false

If an environment variable is missing, the safe behavior is to fall back to the safest value. A missing `DJANGO_DEBUG` should mean production-safe `False`, never `True`. Reverse defaults cause real incidents: debug pages expose settings, source code, and SQL.

### `ALLOWED_HOSTS` and the Host header

The `Host` header is client-supplied, and an attacker can forge it to poison password reset links or caches. Django validates it against `ALLOWED_HOSTS`. With `DEBUG = True` and an empty list, Django permits only `localhost`, `127.0.0.1`, and `[::1]`.

### Type conversion is your job

Environment variables are always strings. The string `"False"` is truthy in Python, so `bool(os.environ["DEBUG"])` is wrong whenever the variable is set at all. Parse booleans explicitly, and parse integers and lists explicitly with clear error messages.

### Fail at startup, not at 3 a.m.

Validate all configuration when the process starts. A missing database setting should stop the deploy immediately, not appear as a `KeyError` in the middle of a request days later.

### Key rotation

`SECRET_KEY_FALLBACKS` lets you rotate `SECRET_KEY` without immediately invalidating sessions: set the new key as `SECRET_KEY` and put the old one in the fallbacks list. Remove it after the old sessions have expired.

## Industry-level snippet

The loader below reads configuration from any mapping (so tests do not need to touch the real environment), parses and validates each value, raises a specific error naming the offending variable, never logs secrets, and produces the `DATABASES` entry from a URL.

`config/env.py`:

```python
"""Typed, validated configuration loaded from environment variables."""

import logging
from collections.abc import Mapping
from dataclasses import dataclass
from typing import Any
from urllib.parse import unquote, urlsplit

from django.core.exceptions import ImproperlyConfigured

logger = logging.getLogger(__name__)

_TRUE_VALUES = frozenset({"1", "true", "yes", "on"})
_FALSE_VALUES = frozenset({"0", "false", "no", "off", ""})
_MIN_SECRET_KEY_LENGTH = 50


class ConfigError(ImproperlyConfigured):
    """A configuration value is missing or invalid."""


@dataclass(frozen=True, slots=True)
class EnvSettings:
    secret_key: str
    debug: bool
    allowed_hosts: tuple[str, ...]
    database_url: str


def parse_bool(name: str, raw: str) -> bool:
    value = raw.strip().lower()
    if value in _TRUE_VALUES:
        return True
    if value in _FALSE_VALUES:
        return False
    raise ConfigError(f"{name} must be a boolean, got {raw!r}")


def parse_csv(raw: str) -> tuple[str, ...]:
    return tuple(item.strip() for item in raw.split(",") if item.strip())


def _require(environ: Mapping[str, str], name: str) -> str:
    value = environ.get(name, "").strip()
    if not value:
        raise ConfigError(f"Environment variable {name} is required")
    return value


def load_env_settings(environ: Mapping[str, str]) -> EnvSettings:
    """Build validated settings from ``environ``.

    Raises:
        ConfigError: if a required value is missing or any value is invalid.
    """
    debug = parse_bool("DJANGO_DEBUG", environ.get("DJANGO_DEBUG", "false"))
    secret_key = _require(environ, "DJANGO_SECRET_KEY")
    allowed_hosts = parse_csv(environ.get("DJANGO_ALLOWED_HOSTS", ""))
    database_url = _require(environ, "DATABASE_URL")

    if not debug:
        if len(secret_key) < _MIN_SECRET_KEY_LENGTH:
            raise ConfigError(
                f"DJANGO_SECRET_KEY must be at least {_MIN_SECRET_KEY_LENGTH} "
                "characters when DJANGO_DEBUG is false"
            )
        if secret_key.startswith("django-insecure"):
            raise ConfigError("DJANGO_SECRET_KEY must not be an insecure placeholder")
        if not allowed_hosts:
            raise ConfigError(
                "DJANGO_ALLOWED_HOSTS must list at least one host "
                "when DJANGO_DEBUG is false"
            )
    else:
        logger.warning("DJANGO_DEBUG is enabled; never use this in production")

    return EnvSettings(
        secret_key=secret_key,
        debug=debug,
        allowed_hosts=allowed_hosts,
        database_url=database_url,
    )


def build_database_config(url: str) -> dict[str, Any]:
    """Convert a database URL into a Django ``DATABASES`` entry."""
    parts = urlsplit(url)

    if parts.scheme in {"postgres", "postgresql"}:
        name = parts.path.lstrip("/")
        if not name or not parts.hostname:
            raise ConfigError("DATABASE_URL must include a host and a database name")
        return {
            "ENGINE": "django.db.backends.postgresql",
            "NAME": name,
            "USER": unquote(parts.username or ""),
            "PASSWORD": unquote(parts.password or ""),
            "HOST": parts.hostname,
            "PORT": str(parts.port or 5432),
        }

    if parts.scheme == "sqlite":
        return {
            "ENGINE": "django.db.backends.sqlite3",
            "NAME": parts.path[1:] or ":memory:",
        }

    raise ConfigError(f"Unsupported DATABASE_URL scheme: {parts.scheme!r}")
```

Using it in `config/settings.py`:

```python
import os
from pathlib import Path

from .env import build_database_config, load_env_settings

BASE_DIR = Path(__file__).resolve().parent.parent

_env = load_env_settings(os.environ)

SECRET_KEY = _env.secret_key
DEBUG = _env.debug
ALLOWED_HOSTS = list(_env.allowed_hosts)
DATABASES = {"default": build_database_config(_env.database_url)}
```

`config/test_env.py`:

```python
import pytest

from config.env import (
    ConfigError,
    build_database_config,
    load_env_settings,
    parse_bool,
)

GOOD_KEY = "x" * 60

PRODUCTION_ENV = {
    "DJANGO_DEBUG": "false",
    "DJANGO_SECRET_KEY": GOOD_KEY,
    "DJANGO_ALLOWED_HOSTS": "example.com, www.example.com",
    "DATABASE_URL": "postgres://app:s3cret@db:5432/appdb",
}


def test_loads_valid_production_settings() -> None:
    settings = load_env_settings(PRODUCTION_ENV)
    assert settings.debug is False
    assert settings.allowed_hosts == ("example.com", "www.example.com")


def test_debug_defaults_to_false() -> None:
    env = {k: v for k, v in PRODUCTION_ENV.items() if k != "DJANGO_DEBUG"}
    assert load_env_settings(env).debug is False


@pytest.mark.parametrize("missing", ["DJANGO_SECRET_KEY", "DATABASE_URL"])
def test_missing_required_variable_is_named(missing: str) -> None:
    env = {k: v for k, v in PRODUCTION_ENV.items() if k != missing}
    with pytest.raises(ConfigError, match=missing):
        load_env_settings(env)


def test_short_secret_key_rejected_in_production() -> None:
    with pytest.raises(ConfigError, match="at least"):
        load_env_settings({**PRODUCTION_ENV, "DJANGO_SECRET_KEY": "short"})


def test_empty_allowed_hosts_rejected_in_production() -> None:
    with pytest.raises(ConfigError, match="ALLOWED_HOSTS"):
        load_env_settings({**PRODUCTION_ENV, "DJANGO_ALLOWED_HOSTS": ""})


def test_debug_mode_relaxes_checks() -> None:
    env = {
        "DJANGO_DEBUG": "true",
        "DJANGO_SECRET_KEY": "dev",
        "DATABASE_URL": "sqlite:///db.sqlite3",
    }
    assert load_env_settings(env).debug is True


@pytest.mark.parametrize(
    ("raw", "expected"),
    [("true", True), ("1", True), ("Off", False), ("", False)],
)
def test_parse_bool(raw: str, expected: bool) -> None:
    assert parse_bool("X", raw) is expected


def test_parse_bool_rejects_garbage() -> None:
    with pytest.raises(ConfigError, match="boolean"):
        parse_bool("X", "maybe")


def test_postgres_url_is_converted() -> None:
    config = build_database_config("postgres://app:p%40ss@db:6543/appdb")
    assert config["NAME"] == "appdb"
    assert config["PASSWORD"] == "p@ss"
    assert config["PORT"] == "6543"


def test_postgres_port_defaults_to_5432() -> None:
    assert build_database_config("postgres://u:p@db/appdb")["PORT"] == "5432"


def test_unsupported_scheme_rejected() -> None:
    with pytest.raises(ConfigError, match="scheme"):
        build_database_config("mysql://u:p@db/appdb")
```

Notice that the error messages name the variable but never print the secret itself.

## Common mistakes

- Committing `.env` or a real `SECRET_KEY`. If it happened, rotate the secret; deleting the file is not enough.
- Using `bool(os.environ["DEBUG"])`, which is true for any non-empty string including `"False"`.
- Leaving `DEBUG = True` as the default and relying on production to override it.
- Setting `ALLOWED_HOSTS = ["*"]` in production, which disables host validation.
- Importing settings with `from config import settings` instead of `django.conf.settings`.
- Hard-coding absolute paths like `/home/me/project` instead of deriving them from `BASE_DIR`.
- Reading environment variables deep inside application code instead of once at startup.
- Logging the whole settings object during debugging and leaking secrets into log storage.

## Practice exercises

1. Run `diffsettings` in a fresh project and list five settings that differ from the defaults, explaining what each does.
2. Write `parse_int(name: str, raw: str, *, minimum: int, maximum: int) -> int` that raises `ConfigError` for non-numeric or out-of-range values, and test it with parametrized cases.
3. Add a `DJANGO_LOG_LEVEL` setting to `EnvSettings` that accepts only the standard logging level names, defaulting to `INFO`.
4. Run `check --deploy` against a project with default settings and write down each warning with the setting that fixes it.
5. Create a `.env.example` file for the loader above with safe placeholder values, and explain which lines must be changed before real use.
6. Extend `build_database_config` to accept a `sslmode` query parameter (`?sslmode=require`) and place it in the `OPTIONS` dictionary, rejecting unknown values.
7. Write a test proving that no exception message raised by `load_env_settings` contains the secret key value.

## Summary

Settings are a Python module chosen by `DJANGO_SETTINGS_MODULE`. The critical ones are `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`, `INSTALLED_APPS`, `DATABASES`, and `TEMPLATES`. Configuration that differs between environments belongs in environment variables, parsed and validated once at startup with explicit, secret-free errors. Defaults must be the safe ones, secrets must stay out of the repository, and `diffsettings` and `check --deploy` are the commands that verify your configuration.
