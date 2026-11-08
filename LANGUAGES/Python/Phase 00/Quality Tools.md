## Overview

Humans are good at design and poor at noticing a missing import or a stray space on line 212. Quality tools are programs that check your code automatically, so people can focus on the parts that need judgment. In this chapter you will meet three tools that almost every modern Python project uses:

- **ruff**: a very fast linter and formatter. A **linter** finds likely bugs and style problems. A **formatter** rewrites code into a consistent layout.
- **mypy**: a **static type checker**. It reads your type hints and warns when values do not match, without running the program.
- **pre-commit**: a manager for **Git hooks**, small checks that run automatically each time you commit.

By the end of this note you will be able to:

- Install and run ruff and mypy.
- Configure all three tools in `pyproject.toml` and `.pre-commit-config.yaml`.
- Read and fix typical warnings.
- Set up pre-commit so bad code never reaches your repository.

> [!note] Why automate
> Automated checks are impartial and instant. They end style debates, catch whole categories of bugs before a reviewer sees them, and give you confidence to change code.

## Core concepts

### Linting, formatting, and type checking

| Activity | Question it answers | Tool | Changes your files? |
| --- | --- | --- | --- |
| Linting | Is anything here probably wrong or risky? | `ruff check` | Only with `--fix`, for safe fixes |
| Formatting | Is the layout consistent? | `ruff format` | Yes |
| Type checking | Do the values match the declared types? | `mypy` | No |

These are complementary. Passing one does not replace the others.

### Installing the tools

Inside your activated virtual environment:

```bash
python -m pip install ruff mypy pre-commit
```

or with uv:

```bash
uv pip install ruff mypy pre-commit
```

Check:

```bash
ruff --version
mypy --version
pre-commit --version
```

In a real project you list these as **development dependencies**, separate from what the application needs at run time. Phase 13 covers that structure.

### Ruff commands

| Command | What it does |
| --- | --- |
| `ruff check` | Lint the current folder and report problems |
| `ruff check --fix` | Lint and automatically apply safe fixes |
| `ruff format` | Reformat files in place |
| `ruff format --check` | Report files that would change, but change nothing |
| `ruff format --diff` | Show what formatting would change |

### Ruff rule families

Ruff organizes lint rules into families, each identified by a letter code. You choose which families to enable.

| Code | Family | Catches |
| --- | --- | --- |
| `E` | pycodestyle errors | Style violations such as bad indentation |
| `F` | pyflakes | Unused imports, unused variables, undefined names |
| `I` | isort | Unsorted or badly grouped imports |
| `N` | pep8-naming | Names that break naming conventions |
| `UP` | pyupgrade | Old syntax that has a modern replacement |
| `B` | bugbear | Common bug patterns |
| `SIM` | simplify | Code that can be written more simply |
| `S` | bandit | Security risks such as unsafe temporary files |

### mypy basics

mypy reads annotations such as `name: str` and `-> int`. It then checks that your code uses values consistently. It never executes your program.

```bash
mypy src
```

Strict mode turns on many extra checks, for example requiring annotations on every function. It is the right target for new projects.

### pre-commit

Git hooks are scripts Git runs at certain moments. The `pre-commit` tool installs a hook that runs your chosen checks whenever you try to commit. If a check fails, the commit stops until you fix the problem. You describe the checks in `.pre-commit-config.yaml`.

| Command | What it does |
| --- | --- |
| `pre-commit install` | Install the hook into this repository |
| `pre-commit run --all-files` | Run all hooks on every file now |
| `pre-commit autoupdate` | Update hook versions to the latest releases |

### Where configuration lives

Ruff and mypy both read settings from `pyproject.toml`, the file you met in Chapter 0.6. pre-commit uses its own file because it manages tools that may not even be Python tools.

## Basic syntax examples

### A messy file for ruff

Save this as `app.py`:

```python
import os
import sys
import json

def  load(path):
    data=json.load(open(path))
    unused = 42
    return dat
```

Run:

```bash
ruff check app.py
```

You will see findings like these (this is real output from a recent ruff release, and the exact list depends on your version and settings):

```text
app.py:1:1: I001 [*] Import block is un-sorted or un-formatted
app.py:1:8: F401 [*] `os` imported but unused
app.py:2:8: F401 [*] `sys` imported but unused
app.py:6:5: F841 Local variable `data` is assigned to but never used
app.py:6:20: SIM115 Use a context manager for opening files
app.py:7:5: F841 Local variable `unused` is assigned to but never used
app.py:8:12: F821 Undefined name `dat`
Found 7 errors.
[*] 3 fixable with `--fix` (2 unsafe fixes available with `--fix --unsafe-fixes`).
```

How to read a line: file name, line number, column number, rule code, then a message. A `[*]` means ruff can fix that finding automatically.

Notice that `F821` points at a real bug. The function would crash with a `NameError` the moment it ran, which is exactly the kind of mistake linting catches before execution. It is also the reason `data` is reported as unused: the typo `dat` means the real variable is never returned.

Apply the safe automatic fixes:

```bash
ruff check --fix app.py
```

Unused imports are removed and the import block is tidied. The undefined name needs you to correct the typo yourself, and the two "unsafe" fixes are not applied unless you ask for them.

> [!note] Default rules change between versions
> Older ruff releases enabled only a small default rule set, while recent ones enable more. This is the reason real projects set `select` explicitly in `pyproject.toml`, as the industry-level snippet below does, so that every developer and every machine runs the same rules.

### Formatting

Before:

```python
def  add( a,b ):
    return a+b
x = add(1,2)
```

Run:

```bash
ruff format app.py
```

After:

```python
def add(a, b):
    return a + b


x = add(1, 2)
```

### A first mypy error

```python
def greet(name: str) -> str:
    return "Hello, " + name


greet(42)
```

```bash
mypy greet.py
```

```text
greet.py:5: error: Argument 1 to "greet" has incompatible type "int"; expected "str"  [arg-type]
Found 1 error in 1 file (checked 1 source file)
```

Python itself would only fail later, at run time, inside the function. mypy reports it before you ever run the code.

### Returning `None` by accident

```python
def find_user_name(user_id: int) -> str:
    if user_id == 1:
        return "admin"
```

mypy reports a missing return, because for any other `user_id` the function returns `None`, which is not a `str`. The fix is either to return something in every path or to change the type to say `None` is possible, which Chapter 8.1 explains.

### Setting up pre-commit

```bash
pre-commit install
git add .
git commit -m "Add feature"
```

If ruff or another hook fails or modifies a file, the commit is aborted. Review the changes, stage them again, and repeat the commit:

```bash
git add .
git commit -m "Add feature"
```

Run all hooks on the whole project at any time with:

```bash
pre-commit run --all-files
```

## Deeper behavior

### Why ruff is fast and why it matters

Ruff is written in Rust and replaces several older tools at once (flake8, isort, pyupgrade, and parts of Black). Because it runs in milliseconds, you can run it on every save and every commit without noticing. Slow tools get skipped, fast tools get used.

### Linter findings are not equal

- Some findings are bugs (`F821` undefined name).
- Some are risks (`B006` mutable default arguments).
- Some are taste (`SIM` suggestions).

Start with a modest set of rules, fix everything, then add more. Turning on every rule on day one produces noise that teams learn to ignore.

### Safe fixes and unsafe fixes

`ruff check --fix` only applies fixes that cannot change behavior. Ruff separates out "unsafe" fixes that might, and you must request those explicitly. Always review changes with Git before committing.

### Formatter versus linter

Run both. The formatter decides layout (quotes, line breaks, spacing), and the linter decides correctness and quality. Because the formatter is deterministic, two developers formatting the same code get identical results.

### Suppressing a warning on purpose

Sometimes a warning is wrong for one line. You can silence it locally and explain why:

```python
value = some_call()  # noqa: S603  (arguments are validated above)
```

For mypy:

```python
result = legacy_function()  # type: ignore[no-untyped-call]
```

Use these sparingly, always target a specific code, and add a reason. A blanket ignore hides real problems.

### Types are checked statically, not at run time

Type hints do not stop wrong values from running. Python ignores them during execution. Only the type checker, and your discipline, enforce them. Chapter 8.6 distinguishes runtime and static typing in depth.

### Gradual typing

mypy lets you adopt types gradually. Unannotated code is treated loosely. Strict mode removes that looseness. For a new project, begin strict. For an old one, adopt strictness module by module.

### pre-commit runs tools in isolated environments

Each hook gets its own private environment, created and cached by pre-commit. This keeps tool versions consistent for everyone, but it means a mypy hook cannot see packages installed in your project's virtual environment. If your code imports third-party libraries, list the type stubs mypy needs under `additional_dependencies`, or run mypy as a separate step in your continuous integration system.

### Pin tool versions

Hook versions are pinned with a `rev` value. Everyone on the team then gets the same results, and upgrades happen deliberately through `pre-commit autoupdate`.

### Hooks can be skipped, CI cannot

A developer can bypass hooks with `git commit --no-verify`. That is why teams also run the same checks in continuous integration, as a final gate that no one can skip.

## Industry-level snippet

A typical backend repository has configuration for all three tools plus a typed module that passes them. First, `pyproject.toml`:

```toml
[tool.ruff]
line-length = 88
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
  "E",    # pycodestyle errors
  "F",    # pyflakes
  "I",    # import sorting
  "N",    # naming conventions
  "UP",   # modern syntax
  "B",    # likely bugs
  "SIM",  # simplifications
  "S",    # security
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]  # assert statements are normal in tests

[tool.ruff.format]
quote-style = "double"

[tool.mypy]
python_version = "3.12"
strict = true
files = ["src"]
```

Next, `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.17.4
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v2.4.1
    hooks:
      - id: mypy
        files: ^src/
        additional_dependencies: []
```

Version numbers move quickly. After copying this file, run `pre-commit autoupdate` to refresh every `rev` to the latest release, then commit the result.

Activate everything:

```bash
pre-commit install
pre-commit run --all-files
```

Finally, a module that is strict-mypy clean and ruff clean. It lives at `src/accounts/emails.py`:

```python
"""Helpers for cleaning user-provided account data."""

import logging
import re

logger = logging.getLogger(__name__)

_EMAIL_PATTERN = re.compile(r"[^@\s]+@[^@\s]+\.[^@\s]+")


class InvalidEmailError(ValueError):
    """Raised when text does not look like an email address."""


def normalize_email(raw: str) -> str:
    """Return a lowercase, trimmed email address.

    Args:
        raw: Email text exactly as the user typed it.

    Returns:
        The cleaned email address.

    Raises:
        InvalidEmailError: If the text does not look like an email address.
    """
    cleaned = raw.strip().lower()
    if not _EMAIL_PATTERN.fullmatch(cleaned):
        # Never log the raw value: email addresses are personal data.
        logger.warning("Rejected malformed email input of length %d", len(raw))
        raise InvalidEmailError("invalid email address")
    return cleaned


def normalize_all(raw_values: list[str]) -> tuple[list[str], list[int]]:
    """Normalize many emails, collecting the positions of invalid ones.

    Args:
        raw_values: Raw email strings.

    Returns:
        A pair of lists. The first holds the cleaned emails. The second holds
        the zero-based positions of the inputs that were invalid.
    """
    valid: list[str] = []
    invalid_positions: list[int] = []
    for position, raw in enumerate(raw_values):
        try:
            valid.append(normalize_email(raw))
        except InvalidEmailError:
            invalid_positions.append(position)
    return valid, invalid_positions
```

What this setup gives you:

- Formatting, imports, and common bugs are fixed or flagged before a commit exists.
- Strict typing means every function signature is documented and checked.
- `src` and `tests` are told to ruff, so import sorting recognizes your own package as first-party.
- A rule is relaxed only where it makes sense (`assert` in tests), not globally.
- The log message avoids personal data, which is both good security practice and a privacy requirement in many jurisdictions.

> [!tip] Rollout order for an existing project
> Run `ruff format` first and commit that alone. Then enable a small rule set and fix findings. Then add mypy in non-strict mode, then raise strictness. Doing everything in one commit makes the history unreadable.

## Common mistakes

1. **Installing the tools globally.** Install them in the project's virtual environment, or the versions will differ between machines.
2. **Enabling every rule at once.** The wall of warnings gets ignored. Grow the rule set gradually.
3. **Using `# noqa` or `# type: ignore` without a code or a reason.** Always name the rule and explain why.
4. **Not committing the configuration.** `pyproject.toml` and `.pre-commit-config.yaml` must be in the repository so everyone shares the same rules.
5. **Forgetting `pre-commit install`.** The config file alone does nothing. The hook must be installed in each clone.
6. **Being confused when a commit "fails" after files were auto-fixed.** Hooks that modify files abort the commit by design. Stage the changes and commit again.
7. **Expecting mypy to see packages in your virtual environment from the pre-commit hook.** Hooks run in isolated environments, so configure `additional_dependencies` or run mypy separately.
8. **Treating passing checks as proof the code is correct.** Tools find certain classes of mistakes. You still need tests (Phase 11).
9. **Fixing warnings by deleting type hints or silencing everything.** The goal is clearer code, not a quiet terminal.
10. **Using old, pinned tool versions for years.** Update regularly with `pre-commit autoupdate`.

## Practice exercises

1. Install ruff, mypy, and pre-commit in a fresh virtual environment and print each version.
2. Save the messy `app.py` from this note, run `ruff check`, and write down what each reported code means. Then fix every finding.
3. Reformat a deliberately ugly file with `ruff format`. Use `ruff format --diff` first to preview changes without applying them.
4. Add a `[tool.ruff]` section to `pyproject.toml` with a line length of 100 and enable the `E`, `F`, and `I` rule families. Show that a line of 95 characters is now accepted.
5. Write a function with a wrong type hint on purpose, such as returning an `int` where `str` is declared, and read mypy's message.
6. Write a function that has a path with no return statement but is annotated to return `str`. Predict mypy's complaint, then verify.
7. Enable `strict = true` for mypy. Take an unannotated function and make it pass.
8. Create `.pre-commit-config.yaml` with ruff hooks only. Run `pre-commit install`, introduce an unused import, try to commit, and describe exactly what happens.
9. Add a `# noqa` comment that silences one real warning with a specific code and a reason. Then explain why a blanket `# noqa` would be worse.
10. Take the `normalize_all` function from this note and add a new function `domain_of(email: str) -> str` that returns the part after the `@`. Keep ruff and mypy strict mode completely clean.

## Summary

- A linter finds likely bugs, a formatter makes layout consistent, a type checker verifies that values match declared types. Use all three.
- ruff handles both linting (`ruff check`) and formatting (`ruff format`) quickly.
- mypy reads type hints and checks them without running the code. Start strict on new projects.
- pre-commit runs your checks automatically on every commit once you run `pre-commit install`.
- Ruff and mypy are configured in `pyproject.toml`. pre-commit is configured in `.pre-commit-config.yaml`.
- Pin tool versions, update them deliberately, and mirror the same checks in continuous integration.
- Suppress warnings only with a specific code and a reason, and treat green checks as necessary but not sufficient.
