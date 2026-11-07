## Overview

Quality tools catch mistakes before they reach users: formatters keep style consistent, linters find suspicious code, type checkers verify that functions are called correctly, and system checks validate Django configuration. Run automatically in pre-commit hooks and continuous integration (CI), they turn good habits into guarantees. This note covers `ruff` with the Django rule set, `mypy` with `django-stubs`, `djlint` for templates, `pre-commit`, `manage.py check`, `makemigrations --check`, and a minimal CI pipeline. It ends with a tested script that runs the whole quality gate with one command.

## Core concepts

### Ruff

Ruff is a very fast linter and formatter. It replaces several older tools (flake8, isort, pyupgrade, Black) with one. The `DJ` rule set adds Django-specific checks, and the `S` rules add security checks from Bandit.

### mypy and django-stubs

`mypy` checks type hints statically. In strict mode it requires annotations everywhere and flags unsafe patterns. `django-stubs` supplies type information for Django, and its plugin understands models and querysets.

### djlint

`djlint` lints and formats Django templates: it checks indentation, unclosed tags, and common template mistakes.

### Django's own checks

- `manage.py check` validates settings, models, admin, and URL configuration.
- `manage.py check --deploy` adds security-oriented warnings suited to production.
- `manage.py makemigrations --check --dry-run` fails if a model change has no migration. Run it in CI so schema drift cannot be merged.

### pre-commit

`pre-commit` runs a configured list of tools on staged files each time you commit, blocking commits that fail. It gives fast feedback locally.

### Continuous integration

CI runs the same checks on every push and pull request on a clean machine. It is the final authority: local hooks can be skipped, but a required CI check cannot.

## Basic syntax examples

### Installing the tools

```bash
uv add --dev ruff mypy "django-stubs[compatible-mypy]" djlint pre-commit
```

### Ruff, mypy, and djlint configuration in pyproject.toml

```toml
[tool.ruff]
target-version = "py312"
line-length = 88
extend-exclude = ["migrations"]

[tool.ruff.lint]
select = [
    "E", "F", "W",   # pycodestyle and pyflakes
    "I",             # import sorting
    "N",             # naming
    "UP",            # modern syntax
    "B",             # bugbear
    "S",             # security
    "DJ",            # Django
    "RUF",           # Ruff-specific rules
]

[tool.ruff.lint.per-file-ignores]
"**/tests/*.py" = ["S101"]
"**/test_*.py" = ["S101"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["mypy_django_plugin.main"]
exclude = ["migrations/"]

[tool.django-stubs]
django_settings_module = "config.settings"

[tool.djlint]
profile = "django"
indent = 2
```

### Running each tool

```bash
uv run ruff format --check .
uv run ruff check .
uv run mypy .
uv run djlint templates apps --check
uv run python manage.py check
uv run python manage.py makemigrations --check --dry-run
```

To fix what can be fixed automatically:

```bash
uv run ruff format .
uv run ruff check . --fix
uv run djlint templates apps --reformat
```

### pre-commit configuration

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/djlint/djLint
    rev: v1.36.0
    hooks:
      - id: djlint-django
```

The `rev` values here are examples; pin the current release of each tool when you set this up. Activate the hooks once per clone:

```bash
uv run pre-commit install
uv run pre-commit run --all-files
```

### A minimal CI pipeline

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv python install 3.12
      - run: uv sync --frozen
      - run: uv run ruff format --check .
      - run: uv run ruff check .
      - run: uv run mypy .
      - run: uv run djlint templates apps --check
      - run: uv run python manage.py check
      - run: uv run python manage.py makemigrations --check --dry-run
      - run: uv run python manage.py test
```

Your project settings need environment variables in CI. Provide safe test values under `env:` in the workflow, as note 0.5 explains.

## Deeper behavior

### What strict mode demands

`strict = true` enables a group of flags, including disallowing untyped function definitions, untyped calls, implicit `Any`, and unused ignore comments. Typical fixes:

```python
# error: Function is missing a return type annotation
def total(prices):
    return sum(prices)


# fixed
def total(prices: list[float]) -> float:
    return sum(prices)
```

Prefer fixing the type to adding `# type: ignore`. When an ignore is unavoidable, name the error code (`# type: ignore[attr-defined]`) and keep it narrow.

### Why `makemigrations --check` matters

Forgetting to commit a migration is easy: the code works on your machine because you ran `makemigrations` locally. In production, the model and the database then disagree. The check makes the pipeline fail whenever models and migrations are out of sync.

### Excluding migrations

Generated migration files rarely follow your style and are not worth linting or type checking. Exclude the `migrations/` folders in Ruff and mypy, as shown above.

### Rule selection strategy

Start with a modest rule set and widen it. Turning on everything at once produces hundreds of warnings and teaches people to ignore the tool. Add a rule family, fix the violations in one commit, and keep the pipeline green.

### Pre-commit versus CI

Pre-commit is for speed: it runs only on changed files, usually just formatting and linting. CI runs everything on the whole project, including slower steps such as type checking and the test suite. Both should run the same Ruff configuration so they never disagree.

### Caching

CI is faster with caching of the package manager and tool caches. `setup-uv` supports caching, and Ruff and mypy store their caches in `.ruff_cache` and `.mypy_cache`, which are ignored by Git.

## Industry-level snippet

Remembering six commands is error-prone. A single script defines the quality gate once, runs every check, reports all failures rather than stopping at the first, and returns a meaningful exit code. Developers and CI both call it.

`scripts/quality.py`:

```python
"""Run the project's quality gate and report every failing check."""

import logging
import subprocess
import sys
from collections.abc import Callable, Sequence
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class Check:
    name: str
    command: tuple[str, ...]


CHECKS: tuple[Check, ...] = (
    Check("format", ("ruff", "format", "--check", ".")),
    Check("lint", ("ruff", "check", ".")),
    Check("types", ("mypy", ".")),
    Check("templates", ("djlint", "templates", "apps", "--check")),
    Check("django-check", ("python", "manage.py", "check")),
    Check(
        "migrations",
        ("python", "manage.py", "makemigrations", "--check", "--dry-run"),
    ),
)

Runner = Callable[[Sequence[str]], int]


def run_command(command: Sequence[str]) -> int:
    """Run ``command`` and return its exit code, or 127 if it is not found."""
    try:
        return subprocess.run(command, check=False).returncode  # noqa: S603
    except FileNotFoundError:
        logger.error("Command not found: %s", command[0])
        return 127


def run_checks(
    checks: Sequence[Check],
    runner: Runner = run_command,
    *,
    fail_fast: bool = False,
) -> list[str]:
    """Run each check and return the names of those that failed."""
    if not checks:
        raise ValueError("At least one check is required")

    failed: list[str] = []
    for check in checks:
        logger.info("Running %s: %s", check.name, " ".join(check.command))
        exit_code = runner(check.command)
        if exit_code != 0:
            logger.error("%s failed with exit code %d", check.name, exit_code)
            failed.append(check.name)
            if fail_fast:
                break
    return failed


def main(argv: Sequence[str] | None = None) -> int:
    args = list(sys.argv[1:] if argv is None else argv)
    unknown = [arg for arg in args if arg != "--fail-fast"]
    if unknown:
        logger.error("Unknown arguments: %s", " ".join(unknown))
        return 2

    failed = run_checks(CHECKS, fail_fast="--fail-fast" in args)
    if failed:
        logger.error("Quality gate failed: %s", ", ".join(failed))
        return 1
    logger.info("Quality gate passed")
    return 0


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    raise SystemExit(main())
```

`scripts/test_quality.py`:

```python
from collections.abc import Callable, Sequence

import pytest

from quality import Check, main, run_checks, run_command


def make_runner(
    failing: set[str],
) -> tuple[list[str], Callable[[Sequence[str]], int]]:
    seen: list[str] = []

    def runner(command: Sequence[str]) -> int:
        seen.append(command[0])
        return 1 if command[0] in failing else 0

    return seen, runner


CHECKS = (
    Check("one", ("tool-a",)),
    Check("two", ("tool-b",)),
    Check("three", ("tool-c",)),
)


def test_all_checks_pass() -> None:
    seen, runner = make_runner(set())
    assert run_checks(CHECKS, runner) == []
    assert seen == ["tool-a", "tool-b", "tool-c"]


def test_reports_every_failure_by_default() -> None:
    _, runner = make_runner({"tool-a", "tool-c"})
    assert run_checks(CHECKS, runner) == ["one", "three"]


def test_fail_fast_stops_at_first_failure() -> None:
    seen, runner = make_runner({"tool-a"})
    assert run_checks(CHECKS, runner, fail_fast=True) == ["one"]
    assert seen == ["tool-a"]


def test_empty_check_list_is_rejected() -> None:
    with pytest.raises(ValueError):
        run_checks([])


def test_missing_command_returns_127() -> None:
    assert run_command(["definitely-not-a-real-command-xyz"]) == 127


def test_main_rejects_unknown_arguments() -> None:
    assert main(["--bogus"]) == 2
```

The helper `make_runner` returns a fake runner together with a list that records which commands were called, so the tests never launch a real tool.

## Common mistakes

- Treating warnings as optional until hundreds accumulate.
- Using blanket `# type: ignore` or disabling rules globally to silence the tools.
- Linting generated migration files and then fighting the formatter over them.
- Running checks only locally. Without a required CI job, a skipped hook means unchecked code in `main`.
- Letting local and CI tool versions drift apart so results differ. Pin versions in the lock file and in pre-commit.
- Forgetting `makemigrations --check`, which lets models and migrations diverge.
- Running `mypy` without the Django plugin and then ignoring every model-related error.
- Making the pipeline so slow that people stop waiting for it. Cache dependencies and keep the quality job fast.

## Practice exercises

1. Install Ruff, mypy, djlint, and pre-commit in a project and run each once. Record the first problem each tool reports.
2. Turn on the `DJ` rule set and introduce one violation deliberately (a model without `__str__`). Confirm Ruff flags it, then fix it.
3. Write three functions with missing or wrong type hints, run `mypy --strict`, and fix each reported error without using `# type: ignore`.
4. Add a model field without creating a migration and show that `makemigrations --check` fails. Then create the migration and show that it passes.
5. Set up pre-commit hooks, make a commit with badly formatted code, and describe what happens.
6. Add a test to `test_quality.py` proving that `run_checks` passes the exact command tuple of each check to the runner, in order, and that a runner returning a non-zero code other than 1 (for example 127) still counts as a failure.
7. Extend the quality script with a `--only NAME` option that runs a single named check, and report an error for unknown names. Add tests.

## Summary

Quality tools turn style and correctness into automatic checks. Ruff formats and lints (with Django and security rules), `mypy` with `django-stubs` verifies types, `djlint` checks templates, and Django's own `check` and `makemigrations --check` validate configuration and schema consistency. Pre-commit gives fast local feedback, while CI is the enforced gate. Exclude migrations from linting, widen rule sets gradually, pin tool versions, and consolidate the checks into one script so that everyone runs the same gate.
