## The modern stack

|Job|Tool|Replaces|
|---|---|---|
|Lint + format + import sort|**Ruff**|flake8, black, isort, pyupgrade, many plugins|
|Type checking|**mypy** or **pyright**||
|Testing|**pytest**|unittest|
|Coverage|**pytest-cov** / coverage.py||
|Git hooks|**pre-commit**||
|Security|**bandit** (or Ruff `S` rules), **pip-audit**||

**Quick setup with uv:**
```bash
uv add --dev ruff mypy pytest pytest-cov pre-commit
```

---
## `Ruff` (lint + format)
```bash
ruff check .                     # lint
ruff check --fix .               # auto-fix safe issues
ruff check --fix --unsafe-fixes .
ruff check --select I --fix .    # sort imports only
ruff check --statistics .        # summary of violations by rule
ruff check --add-noqa .          # silence all existing violations (for adoption)
ruff rule B006                   # explain a rule
ruff format .                    # format (Black-compatible)
ruff format --check .            # CI mode: fail if changes needed
ruff format --diff .             # show what would change
```
With uv, without installing: `uvx ruff check .`
### `pyproject.toml`
```toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = [
  "E", "W",   # pycodestyle
  "F",        # pyflakes
  "I",        # isort
  "B",        # bugbear (likely bugs)
  "UP",       # pyupgrade (modern syntax)
  "SIM",      # simplifications
  "C4",       # comprehensions
  "RUF",      # ruff-specific
  # "S",      # security (bandit)
  # "N",      # naming
  # "D",      # docstrings (noisy)
]
ignore = ["E501"]                # line length handled by formatter

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]             # allow assert in tests
"__init__.py" = ["F401"]         # allow unused imports

[tool.ruff.format]
quote-style = "double"
docstring-code-format = true
```

### Suppressing
```python
x = 1  # noqa: E501
import os  # noqa
# ruff: noqa: F401          (whole file, put at top)
# fmt: off
# fmt: on
# fmt: skip
```

---
## Type checking

### mypy
```bash
mypy .
mypy src/
mypy --strict src/
mypy --install-types             # install missing stub packages
mypy --html-report report/       # needs lxml
dmypy run -- .                   # daemon mode, much faster on re-runs
```

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unused_ignores = true
exclude = ["build/", "migrations/"]

[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

### pyright
```bash
brew install pyright             # or: uv add --dev pyright
pyright
pyright src/
```

```toml
[tool.pyright]
typeCheckingMode = "standard"    # "basic" | "standard" | "strict"
include = ["src"]
```

### Modern typing syntax (3.10+)
```python
def f(x: int | None, items: list[str]) -> dict[str, int]: ...

from typing import TypedDict, Protocol, Literal, Final, Self, TypeAlias
```

### Suppressing
```python
x: int = "a"  # type: ignore[assignment]
# mypy: ignore-errors            (top of file)
```
Newer, faster checkers (Astral's **ty**, Meta's **pyrefly**) are in preview and worth watching.

---
## pytest
```bash
pytest                           # run all
pytest tests/test_x.py           # one file
pytest tests/test_x.py::test_fn  # one test
pytest -k "login and not slow"   # filter by name
pytest -x                        # stop at first failure
pytest --lf                      # rerun last failed
pytest --ff                      # failed first
pytest -v / -q                   # verbose / quiet
pytest -s                        # show print output
pytest --pdb                     # debugger on failure
pytest -m "not slow"             # by marker
pytest -n auto                   # parallel (pytest-xdist)
pytest --durations=10            # slowest 10 tests
```

### Coverage
```bash
pytest --cov=src --cov-report=term-missing
pytest --cov=src --cov-report=html          # open htmlcov/index.html
pytest --cov=src --cov-fail-under=80        # fail if below 80%
```

### Config
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"
markers = ["slow: slow tests"]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
show_missing = true
exclude_lines = ["pragma: no cover", "if TYPE_CHECKING:", "raise NotImplementedError"]
```

### Patterns
```python
import pytest

@pytest.fixture
def user():
    return {"name": "Ada"}

@pytest.mark.parametrize("a,b,expected", [(1, 2, 3), (0, 0, 0)])
def test_add(a, b, expected):
    assert a + b == expected

def test_raises():
    with pytest.raises(ValueError, match="invalid"):
        int("x")

@pytest.mark.skip(reason="todo")
@pytest.mark.xfail
def test_later(): ...

def test_env(monkeypatch, tmp_path):
    monkeypatch.setenv("API_KEY", "test")
    (tmp_path / "f.txt").write_text("hi")
```
Built-in fixtures: `tmp_path`, `monkeypatch`, `capsys`, `caplog`, `request`.

---
## pre-commit
```bash
pre-commit install               # enable git hook
pre-commit run --all-files       # run on everything
pre-commit autoupdate            # bump hook versions
pre-commit run ruff --all-files  # one hook
git commit --no-verify           # skip hooks (emergency only)
```

### `.pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0          # run `pre-commit autoupdate` to get the latest
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: []   # add type stubs here
```

---
## Security and hygiene
```bash
pip-audit                        # known vulnerabilities in dependencies
bandit -r src/                   # static security analysis
gitleaks detect                  # scan repo for secrets (brew install gitleaks)
vulture src/                     # find dead code
deptry .                         # unused/missing dependencies
radon cc src/ -a -nc             # cyclomatic complexity (C grade or worse)
```

---
## One-command checks

### Makefile
```makefile
.PHONY: check fix test

check:
	ruff check .
	ruff format --check .
	mypy src
	pytest

fix:
	ruff check --fix .
	ruff format .
```

### Or with uv
```bash
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest
```

---
## GitHub Actions (CI)
```yaml
name: CI
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --all-extras --dev
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy src
      - run: uv run pytest --cov=src --cov-fail-under=80
```

---
## VS Code
Install the extensions **Ruff** (`charliermarsh.ruff`) and **Pylance** or **Mypy Type Checker**.
```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"]
}
```

---
## Adopting on an existing codebase
1. `ruff format .` and commit it alone.
2. Add the formatting commit to `.git-blame-ignore-revs`, then run `git config blame.ignoreRevsFile .git-blame-ignore-revs`.
3. Start with a small `select` list (`E`, `F`, `I`), run `ruff check --fix .`, and expand gradually.
4. Use `ruff check --add-noqa .` to baseline the old violations and enforce clean code going forward.
5. Start mypy non-strict, then enable strict per module with overrides.

---
## Good habits
- Pin tool versions in `pyproject.toml` and the lockfile so CI matches local runs.
- Run the same commands locally, in pre-commit, and in CI.
- Prefer fixing issues over blanket `noqa` and `type: ignore`. Add the specific rule code when you must suppress.
- Test behavior, not implementation. Keep tests fast and isolated.

---