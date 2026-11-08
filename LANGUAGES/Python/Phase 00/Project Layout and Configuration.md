## Overview

A single file is fine for learning, but real software is made of many files that must be organized predictably. This chapter shows the standard way to lay out a Python project, how `pyproject.toml` describes it, how `.gitignore` keeps junk out of version control, and how `.env` files keep secrets out of your code.

By the end of this note you will be able to:

- Create a project with a `src/` folder, a `tests/` folder, and the files every project needs.
- Read and write a basic `pyproject.toml`.
- Write a `.gitignore` for a Python project.
- Load configuration and secrets from environment variables and explain why secrets never belong in source code.

> [!note] Why this matters early
> Habits formed in your first projects stay with you. A clean layout makes testing, packaging, and teamwork easier later, and leaking a password into a public repository is a mistake that is easy to prevent and painful to fix.

## Core concepts

### The standard layout

Here is a layout used by many professional Python projects:

```text
inventory-service/
  .env.example
  .gitignore
  README.md
  pyproject.toml
  src/
    inventory_service/
      __init__.py
      config.py
      main.py
  tests/
    test_config.py
```

| Item | Purpose |
| --- | --- |
| `src/inventory_service/` | Your actual code, as an importable package |
| `tests/` | Automated tests, kept outside the package |
| `pyproject.toml` | Project metadata, dependencies, and tool settings |
| `README.md` | What the project is and how to run it |
| `.gitignore` | Files Git must never track |
| `.env.example` | A template listing needed settings, with fake values |
| `.env` (not committed) | Your real local settings and secrets |
| `.venv/` (not committed) | Your virtual environment |

### Flat layout versus src layout

| Aspect | Flat layout | src layout |
| --- | --- | --- |
| Package location | Project root | Inside `src/` |
| Risk of importing the wrong copy | Higher, because the working directory is on the import path | Lower, you must install the package to import it |
| Tests reflect real usage | Not always | Yes, tests import the installed package |
| Typical use | Small scripts | Libraries, services, anything you will ship |

With the `src/` layout, you install your own project into the virtual environment in **editable mode**. That makes Python see your package while still letting your edits take effect immediately:

```bash
python -m pip install -e .
```

or

```bash
uv pip install -e .
```

### Package names and folder names

A package folder must be a valid Python name: lowercase letters, digits, and underscores, never starting with a digit and never containing hyphens or spaces. A project called `inventory-service` therefore has a package named `inventory_service`.

### The `__init__.py` file

A folder with an `__init__.py` file is a regular package. The file can be empty, or it can hold package-level information such as a version string. Chapter 4.2 explains packages in depth.

### pyproject.toml

`pyproject.toml` is the single configuration file for modern Python projects. It uses **TOML**, a simple format of `key = value` lines grouped under `[section]` headers. It has three main areas:

| Section | Role |
| --- | --- |
| `[build-system]` | Which tool builds your project, such as hatchling or setuptools |
| `[project]` | Name, version, description, Python requirement, and dependencies |
| `[tool.NAME]` | Settings for individual tools such as pytest, ruff, and mypy |

### .gitignore

A `.gitignore` file lists patterns for files Git should not track. Typical entries for Python projects are the virtual environment, compiled bytecode, caches from tools, build output, and `.env`.

### Environment variables and .env files

An **environment variable** is a named value provided to a program from outside its code. Programs read them at startup to learn things like a database address or an API key.

A **`.env` file** is a plain text file of `NAME=value` lines used during local development to set those variables. Python does not read `.env` files on its own. A small package called `python-dotenv`, your editor, or your tooling loads them. On real servers, the hosting platform sets the variables directly and no `.env` file exists.

### Why secrets never go into code

- Source code is copied everywhere: laptops, backups, code review tools, and repositories.
- Git remembers every version forever. Deleting a password from a file does not delete it from history.
- Public repositories are scanned by automated bots within minutes of a secret appearing.
- Different environments (development, testing, production) need different values, and code should not change between them.

> [!warning] If a secret is ever committed
> Treat it as compromised. Rotate it (generate a new one and revoke the old) immediately. Cleaning history helps, but rotating is the actual fix.

## Basic syntax examples

### Create the layout from the terminal

macOS and Linux:

```bash
mkdir -p inventory-service/src/inventory_service inventory-service/tests
cd inventory-service
touch README.md pyproject.toml .gitignore .env.example
touch src/inventory_service/__init__.py src/inventory_service/main.py
```

Windows PowerShell:

```powershell
mkdir inventory-service\src\inventory_service, inventory-service\tests
cd inventory-service
New-Item README.md, pyproject.toml, .gitignore, .env.example
New-Item src\inventory_service\__init__.py, src\inventory_service\main.py
```

### A minimal pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "inventory-service"
version = "0.1.0"
description = "Inventory tracking backend"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []
```

Then install it into your active virtual environment and import it:

```bash
python -m pip install -e .
python -c "import inventory_service; print(inventory_service.__file__)"
```

### Adding dependencies

```toml
dependencies = [
  "requests>=2.31,<3",
]
```

### A starter .gitignore

```text
# Virtual environments
.venv/
venv/

# Python bytecode and caches
__pycache__/
*.py[cod]
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Packaging and build output
dist/
build/
*.egg-info/

# Local configuration and secrets
.env

# Editors and operating system files
.idea/
.DS_Store
```

Lines starting with `#` are comments. A trailing slash matches folders. `*` matches any characters.

### .env and .env.example

`.env` (never committed):

```text
DATABASE_URL=postgresql://app:s3cret@localhost:5432/inventory
API_KEY=live-1234567890
DEBUG=true
REQUEST_TIMEOUT=10
```

`.env.example` (committed, with fake values):

```text
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
API_KEY=replace-me
DEBUG=false
REQUEST_TIMEOUT=10
```

A new teammate copies `.env.example` to `.env` and fills in real values.

### Reading an environment variable

```python
import os

api_key = os.environ["API_KEY"]            # raises KeyError if missing
timeout = os.environ.get("REQUEST_TIMEOUT", "10")   # falls back to "10"
print(api_key, timeout)
```

Environment variables are always text. Convert to numbers or booleans yourself.

## Deeper behavior

### Why the src layout catches real bugs

With a flat layout, running tests from the project root puts the root on Python's import path, so `import inventory_service` quietly succeeds by loading the files in front of you, even if packaging is broken. With `src/`, the package is only importable if it is installed, so a missing file or wrong configuration fails in testing rather than in production.

### Editable installs

`pip install -e .` does not copy your code into `site-packages`. It creates a pointer back to your folder. Edit a file, rerun your code, and the change takes effect. You only reinstall when you change dependencies or package structure in `pyproject.toml`.

### Build backends

The `[build-system]` section names the tool that turns your source into an installable package. Hatchling, setuptools, flit, and others all work with the same `[project]` table. If your project name is `inventory-service`, hatchling automatically finds `src/inventory_service/`. Phase 13 covers building and publishing.

### Version specifiers

| Specifier | Meaning |
| --- | --- |
| `==2.32.3` | Exactly this version |
| `>=2.31` | This version or newer |
| `>=2.31,<3` | Newer than 2.31 but below 3 |
| `~=2.31` | Compatible release: 2.31 or newer but below 3.0 |

### Configuration hierarchy

A common priority for settings, from lowest to highest: defaults in code, then a config file, then `.env` for local development, then real environment variables, then command-line options. Higher entries override lower ones. This keeps one codebase working in every environment.

### Validate configuration at startup

Failing immediately with a clear message when a required setting is missing is far better than crashing minutes later with a confusing error. Treat configuration loading as a gate that must pass before the application starts.

### Do not log secrets

Logging a settings object that contains a password writes the password to log files, which are then widely shared. Mark secret fields so they are hidden when printed.

## Industry-level snippet

This module loads settings from environment variables, validates them, converts types, hides secrets from logs, and raises a clear custom error. It belongs at `src/inventory_service/config.py`.

```python
"""Application settings loaded from environment variables."""

import logging
import os
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)

_TRUE_VALUES = {"1", "true", "yes", "on"}
_FALSE_VALUES = {"0", "false", "no", "off"}


class ConfigError(Exception):
    """Raised when required configuration is missing or invalid."""


@dataclass(frozen=True)
class Settings:
    """Validated, immutable application settings."""

    database_url: str = field(repr=False)  # may contain a password
    api_key: str = field(repr=False)
    debug: bool
    request_timeout: float


def _require(name: str) -> str:
    """Return a required environment variable or raise ConfigError."""
    value = os.environ.get(name, "").strip()
    if not value:
        raise ConfigError(f"Environment variable {name} is required but not set")
    return value


def _read_bool(name: str, default: bool) -> bool:
    """Read a boolean environment variable with a default."""
    raw = os.environ.get(name)
    if raw is None:
        return default
    normalized = raw.strip().lower()
    if normalized in _TRUE_VALUES:
        return True
    if normalized in _FALSE_VALUES:
        return False
    raise ConfigError(f"{name} must be a boolean such as true or false, got {raw!r}")


def _read_positive_float(name: str, default: float) -> float:
    """Read a positive number from an environment variable."""
    raw = os.environ.get(name)
    if raw is None:
        return default
    try:
        value = float(raw)
    except ValueError as exc:
        raise ConfigError(f"{name} must be a number, got {raw!r}") from exc
    if value <= 0:
        raise ConfigError(f"{name} must be greater than zero, got {value}")
    return value


def load_settings() -> Settings:
    """Build Settings from the environment, failing fast on any problem."""
    settings = Settings(
        database_url=_require("DATABASE_URL"),
        api_key=_require("API_KEY"),
        debug=_read_bool("DEBUG", default=False),
        request_timeout=_read_positive_float("REQUEST_TIMEOUT", default=10.0),
    )
    logger.info("Loaded settings: %s", settings)
    return settings
```

And a tiny entry point, `src/inventory_service/main.py`:

```python
"""Entry point for the inventory service."""

import logging
import sys

from inventory_service.config import ConfigError, load_settings

logger = logging.getLogger(__name__)


def main() -> int:
    """Start the service and return a process exit code."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")
    try:
        settings = load_settings()
    except ConfigError as exc:
        logger.error("Configuration problem: %s", exc)
        return 2
    logger.info("Starting with a request timeout of %s seconds", settings.request_timeout)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Run it, after installing the project in editable mode:

```bash
python -m inventory_service.main
```

With no variables set you get a clear message and exit code 2. If you set `DATABASE_URL` and `API_KEY` in your shell, it starts.

Why this is production-style:

- `frozen=True` makes the settings object read-only after creation, preventing accidental changes.
- `field(repr=False)` ensures secrets do not appear when the object is printed or logged.
- Every conversion has explicit validation and a specific error message.
- A custom `ConfigError` lets the caller handle configuration problems separately from other failures.
- The `from exc` clause keeps the original error attached for debugging.

To load a `.env` file automatically in local development you could add the `python-dotenv` package and call `load_dotenv()` before `load_settings()`. Keep that call out of production code paths, where the platform provides the real variables.

## Common mistakes

1. **Putting secrets directly in code.** Even in a private repository, this is a habit that leaks eventually.
2. **Committing `.env`.** Add it to `.gitignore` before the first commit, not after.
3. **Creating `.gitignore` after files were already committed.** Ignoring does not untrack files already in history. Remove them from tracking (Chapter 0.8) and rotate any leaked secret.
4. **Hyphens in package folder names.** `inventory-service` cannot be imported. Use `inventory_service`.
5. **Forgetting `__init__.py`.** Without it some tools will not treat the folder as a regular package.
6. **Skipping the editable install in a `src/` project.** You will get `ModuleNotFoundError` when importing your own package.
7. **Treating environment variables as numbers or booleans.** `os.environ["DEBUG"]` returns text, and the text `"false"` is truthy in Python.
8. **Committing the virtual environment or `__pycache__`.** These are large, machine-specific, and regenerable.
9. **Having no `.env.example`.** New developers cannot tell which settings are needed.
10. **Reading configuration from many places across the code.** Load it once at startup into one object and pass that around.

## Practice exercises

1. Create the layout shown in this note for a project named `library-api`, using commands only. Verify it with a directory listing.
2. Write a minimal `pyproject.toml` for it that requires Python 3.12 or newer and has no dependencies.
3. Install the project in editable mode into a fresh virtual environment, then import the package from a different folder to prove the install worked.
4. Write a `.gitignore` that covers a virtual environment, caches, build output, secrets, and your editor's files. Explain each line you wrote.
5. Create `.env` and `.env.example` for a project that needs a database URL, an API key, and a log level. Decide which file belongs in version control and explain why.
6. Write a short script that reads `LOG_LEVEL` from the environment, defaults to `INFO`, and refuses anything outside the standard level names.
7. Explain why the text `"false"` is not a good way to store a boolean if you only use `if os.environ["DEBUG"]:`. Write the correct conversion.
8. Add a new setting, `MAX_CONNECTIONS`, to the industry-level snippet. It must be an integer between 1 and 100 and default to 10.
9. Run the entry point with a missing `API_KEY` and with an invalid `REQUEST_TIMEOUT`. Compare the two error messages and exit codes.
10. Add a test file in `tests/` that checks `load_settings` raises `ConfigError` when `DATABASE_URL` is missing, without using real secrets.

## Summary

- Use a `src/` layout with a `tests/` folder next to it. Name the package folder with underscores.
- `pyproject.toml` holds project metadata, dependencies, and tool settings in TOML format.
- Install your own project with `pip install -e .` so tests and tools import the real package.
- `.gitignore` keeps virtual environments, caches, build output, and secrets out of version control.
- Configuration comes from the environment. `.env` files are a local convenience, `.env.example` documents what is needed.
- Never put secrets in code. If one leaks, rotate it.
- Load and validate configuration once at startup, fail early with clear errors, and keep secrets out of logs.
