## Overview

Python's standard library is large, but it is not everything. For web frameworks, HTTP clients, data analysis, testing tools, database drivers, and thousands of other jobs, you use code written and shared by other people: **third-party packages**. Using them well is a core professional skill, and it is as much about judgment as technique. A good dependency saves months of work. A bad one brings bugs, security holes, and upgrade pain that last for years.

This chapter covers how to find and install packages, how versions and version specifiers work, how to pin and lock dependencies so that builds are reproducible, how to read package documentation, and how to decide whether a package deserves a place in your project.

By the end of this note you will be able to:

- Install, inspect, upgrade, and remove packages with `pip` and `uv`.
- Read and write version specifiers such as `>=2.31,<3`.
- Explain the difference between declared dependencies, pinned versions, and a lock file.
- Use `uv` to add dependencies and produce a lock file.
- Find your way around a package's documentation.
- Evaluate whether a package is trustworthy and worth depending on.

> [!warning] Installing a package runs someone else's code
> A package executes on your machine when you install it (in some cases) and every time you import it. It has the same permissions you do. Treat adding a dependency as a decision, not a convenience.

## Core concepts

### Packages, PyPI, and distributions

**PyPI**, the Python Package Index at pypi.org, is the public repository where people publish packages. When you run `pip install requests`, pip looks the name up on PyPI, downloads a file, and installs it into the current environment (Chapter 0.5). The thing you install is called a **distribution**. It may contain one or several importable packages (Chapter 4.2).

### Wheels and source distributions

A package is published in one or both of these forms:

| Form | File | What installing does | Notes |
| --- | --- | --- | --- |
| **Wheel** | `.whl` | Unpacks ready-made files | Fast and predictable. Runs no build code |
| **Source distribution** | `.tar.gz` | Builds the package on your machine first | Slower, needs build tools, and executes the package's build code |

pip and uv prefer a wheel when one matches your Python version and platform.

### The install name is not always the import name

The name you give to `pip install` is the **distribution name**. The name you write in `import` is the **module name**. They are often the same, and often not:

| `pip install` | `import` |
| --- | --- |
| `requests` | `requests` |
| `pillow` | `PIL` |
| `beautifulsoup4` | `bs4` |
| `python-dateutil` | `dateutil` |
| `PyYAML` | `yaml` |
| `scikit-learn` | `sklearn` |

Always check the project's installation instructions rather than guessing.

### Installing packages

Always install into a virtual environment (Chapter 0.5), never into the system Python.

| Task | pip | uv |
| --- | --- | --- |
| Install a package | `python -m pip install requests` | `uv pip install requests` |
| Install from a requirements file | `python -m pip install -r requirements.txt` | `uv pip install -r requirements.txt` |
| Upgrade a package | `python -m pip install --upgrade requests` | `uv pip install --upgrade requests` |
| Uninstall | `python -m pip uninstall requests` | `uv pip uninstall requests` |
| List installed | `python -m pip list` | `uv pip list` |
| Show details | `python -m pip show requests` | `uv pip show requests` |
| Record exact versions | `python -m pip freeze` | `uv pip freeze` |
| Check consistency | `python -m pip check` | `uv pip check` |

`uv` also has a **project mode** that manages dependencies for you in `pyproject.toml` and a lock file: `uv add requests`, `uv remove requests`, `uv sync`, `uv run`, `uv lock`, and `uv tree`.

### Versions and specifiers

Package versions look like `2.34.2`. Most projects follow **semantic versioning**: `MAJOR.MINOR.PATCH`.

| Part | Increases when | Safe to upgrade? |
| --- | --- | --- |
| Patch (`2.34.1` to `2.34.2`) | Bug fixes only | Usually |
| Minor (`2.33.0` to `2.34.0`) | New features, backward compatible | Usually |
| Major (`2.x` to `3.0`) | Breaking changes allowed | Read the changelog first |

Not every project follows the rules perfectly, so tests are your real safety net.

A **version specifier** says which versions you accept:

| Specifier | Meaning |
| --- | --- |
| `requests==2.34.2` | Exactly this version |
| `requests>=2.31` | This or anything newer |
| `requests>=2.31,<3` | At least 2.31 and below 3 |
| `requests~=2.31` | Compatible release: at least 2.31, below 3.0 |
| `requests!=2.32.0` | Anything except this version |

Quote specifiers on the command line, because `>` and `<` are shell redirection symbols: `pip install "requests>=2.31,<3"`.

Other features you will see:

- **Extras** install optional dependencies: `requests[socks]`.
- **Environment markers** restrict a line to some situations: `tomli>=2; python_version < "3.11"`.
- **Pre-releases** (`3.0.0rc1`) are skipped unless you ask for them.
- A maintainer can **yank** a release that turned out broken. New installs avoid it, but existing pins still work.

### Direct and transitive dependencies

The packages you ask for are your **direct dependencies**. The packages **they** need are **transitive dependencies**. Installing `requests` also installs `certifi`, `charset-normalizer`, `idna`, and `urllib3`. A real project may have a handful of direct dependencies and a hundred transitive ones, and any of them can break or be compromised.

### Declared, pinned, and locked

Three different things are easy to confuse:

| Term | Where | Example | Purpose |
| --- | --- | --- | --- |
| **Declared** | `pyproject.toml` | `requests>=2.34` | What your project is **compatible with** |
| **Pinned** | `requirements.txt` | `requests==2.34.2` | One exact version of each package |
| **Locked** | `uv.lock` and similar | Every package, exact version, and file hashes, for all platforms | Exactly what was tested, rebuilt identically anywhere |

Declaring ranges lets your project coexist with other software. Locking makes sure that your tests, your teammates, and your production servers all use **exactly the same** code.

### Lock files

A **lock file** is generated by a tool that **resolves** your declared dependencies into one complete, exact set (every direct and transitive package, with versions and checksums), and records it. Installing from the lock file then reproduces that set precisely. Without one, `pip install requests` today and next month can produce different environments, and "it worked on my machine" becomes a real problem.

| Tool | Lock file |
| --- | --- |
| `uv` | `uv.lock` |
| Poetry | `poetry.lock` |
| PDM | `pdm.lock` |
| `pip-tools` | A compiled, fully pinned `requirements.txt` |

The usual rule: **applications** (services you deploy) commit their lock file. **Libraries** (code others install) declare reasonable ranges and should not force exact versions on their users, though they may keep a lock file for their own development.

### Development dependencies

Tools you need for development but not at run time (a test runner, a linter, a type checker) belong in a separate **dependency group** in `pyproject.toml`, so they are not installed in production:

```toml
[dependency-groups]
dev = ["pytest>=9", "ruff", "mypy"]
```

### Reading documentation

Well-documented packages follow a pattern:

| Section | Use it to |
| --- | --- |
| Quickstart or tutorial | See the shortest path to a working example |
| User guide | Understand concepts and common tasks |
| API reference | Look up exact function and class signatures |
| Changelog or release notes | Learn what changed, especially across major versions |
| Issue tracker | See known bugs and how responsive the maintainers are |

Always check that the **documentation version matches the version you installed**. Many sites let you pick a version. Tutorials found on the web may describe an older or newer API than yours. In the Python prompt, `help(module.function)`, `dir(module)`, and the editor's hover text show the real signatures of what is installed.

### Evaluating a package

Before adding a dependency, check:

| Question | What to look at |
| --- | --- |
| Do I need it at all? | Is there a standard library solution? Is the feature small enough to write yourself? |
| Is it maintained? | Date of the latest release, recent commits, whether issues and pull requests get answers |
| Is it used? | Download counts and who depends on it (a weak signal alone, but useful) |
| Who stands behind it? | An organization or well-known maintainers, and more than one maintainer |
| Is it documented? | A clear quickstart, a reference, a changelog |
| Is it tested? | A visible test suite and continuous integration |
| Is the licence acceptable? | MIT, BSD, and Apache-2.0 are permissive. Others may impose obligations |
| Does it support my Python? | The `requires-python` setting and the classifiers |
| Is it typed? | A `py.typed` marker, or type hints in the code |
| How heavy is it? | How many transitive dependencies it pulls in |
| Has it had security problems? | Known advisories, and how fast they were fixed |
| Is it stable? | A version 1.0 or later and a sensible release history |

### Warning signs

- A name that is **almost** a popular package (`reqeusts`, `python-requests-2`). This is **typosquatting**, a common attack.
- A brand-new package with few downloads that claims to do what a popular one already does.
- No source repository, or a repository link that does not match the package.
- An abandoned project with no release for years and many unanswered issues.
- Requests for unusual installation steps, such as running a downloaded script.
- A huge number of dependencies for a small job.

### Keeping dependencies safe

| Practice | Why |
| --- | --- |
| Lock and commit exact versions | You control when changes arrive |
| Install from a lock file with hashes | A tampered file will not match |
| Scan regularly with a tool such as `pip-audit` | Finds known vulnerabilities in what you use |
| Update small and often | Smaller upgrades are easier to test and review |
| Review new dependencies and big upgrades | Supply-chain attacks arrive through updates |
| Use one trusted package index | Mixing public and private indexes can let an attacker publish a package with your private package's name (**dependency confusion**) |
| Prefer wheels | They do not run build code on your machine |

## Basic syntax examples

The versions below are the ones available at the time of writing, and yours will differ.

### Install and inspect

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\Activate.ps1
python -m pip install requests
```

```bash
python -m pip show requests
```

```text
Name: requests
Version: 2.34.2
Summary: Python HTTP for Humans.
Author-email: Kenneth Reitz <me@kennethreitz.org>
License: Apache-2.0
Location: /home/learner/project/.venv/lib/python3.12/site-packages
Requires: certifi, charset_normalizer, idna, urllib3
Required-by:
```

`Requires` shows the direct dependencies of this package, and `Required-by` shows which installed packages depend on it.

### Freeze, list, and check

```bash
python -m pip freeze
```

```text
certifi==2026.10.23
charset-normalizer==3.5.2
idna==3.20
requests==2.34.2
urllib3==2.8.0
```

```bash
python -m pip check
```

```text
No broken requirements found.
```

`pip freeze` lists **every** installed package with an exact version, including the four transitive ones. Save it with `python -m pip freeze > requirements.txt` to describe this environment. `pip check` verifies that the installed packages' requirements are all satisfied.

### Reading what a package requires

```python
from importlib.metadata import requires, version

print(version("requests"))
print(requires("requests"))
```

```text
2.34.2
['charset_normalizer<4,>=2', 'idna<4,>=2.5', 'urllib3<3,>=1.26', 'certifi>=2023.5.7', 'PySocks!=1.5.7,>=1.5.6; extra == "socks"', 'chardet<8,>=3.0.2; extra == "use-chardet-on-py3"']
```

These are the specifiers the package itself declares. Notice the ranges, the extras (`extra == "socks"`), and the markers after the semicolons.

### Asking for a version that does not exist

```bash
python -m pip install "requests==0.0.1"
```

```text
ERROR: Could not find a version that satisfies the requirement requests==0.0.1 (from versions: 0.2.0, 0.2.1, ...)
ERROR: No matching distribution found for requests==0.0.1
```

The long list of available versions (shortened here) helps you choose a valid one.

### Using an installed package

```python
import requests

request = requests.Request(
    "GET",
    "https://api.example.com/users",
    params={"page": 2, "q": "ada lovelace"},
).prepare()
print(request.url)
```

```text
https://api.example.com/users?page=2&q=ada+lovelace
```

This builds a request without sending it, so it works offline. The library took care of encoding the space in the query.

### A project with uv

```bash
uv init myproject
cd myproject
uv add requests
uv add --dev pytest
```

`uv add` changes `pyproject.toml`, resolves all the dependencies, writes the lock file, and installs everything into the project's `.venv`. The relevant part of `pyproject.toml` afterwards:

```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.34.2",
]

[dependency-groups]
dev = [
    "pytest>=9.2.0",
]
```

`uv add` wrote a lower bound, not an exact version. The exact versions live in the lock file.

### The lock file

An excerpt of `uv.lock`, with the long web addresses and file checksums shortened:

```text
[[package]]
name = "requests"
version = "2.34.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "charset-normalizer" },
    { name = "idna" },
    { name = "urllib3" },
]
sdist = { url = "https://files.pythonhosted.org/packages/...", hash = "sha256:..." }
wheels = [
    { url = "https://files.pythonhosted.org/packages/...", hash = "sha256:..." },
]
```

You do not edit this file by hand. Every package has an exact version and a **hash** of each file, so an installation can confirm that it received exactly what was locked.

### The dependency tree

```bash
uv tree
```

```text
Resolved 12 packages in 38ms
myproject v0.1.0
├── requests v2.34.2
│   ├── certifi v2026.10.23
│   ├── charset-normalizer v3.5.2
│   ├── idna v3.20
│   └── urllib3 v2.8.0
└── pytest v9.2.0 (group: dev)
    ├── iniconfig v2.3.1
    ├── packaging v26.3
    ├── pluggy v1.7.0
    └── pygments v2.21.0
```

Two direct dependencies, eight transitive ones. This view answers "why is this package here?".

### Rebuilding an environment from the lock file

```bash
uv sync          # make .venv match uv.lock exactly
uv run python -c "import requests; print(requests.__version__)"
uv lock --upgrade-package requests    # allow only requests to move to a newer version
uv lock --check                       # verify the lock file matches pyproject.toml
```

A teammate who clones the project needs only `uv sync`. Your continuous integration system and your production build do the same, so every environment runs the code you tested.

## Deeper behavior

### How the resolver works

When you ask for several packages, each with its own constraints on others, an installer must find **one version of each** that satisfies everybody. This is the **dependency resolution** problem, and tools solve it by trying combinations and backing up when they hit a conflict. The result is deterministic for a given set of inputs and a given state of the index, which is exactly what a lock file captures.

### Dependency conflicts

If two packages need incompatible versions of a third, no valid solution exists, and the tool reports it. For example, one library needs `urllib3<2` while another needs `urllib3>=2`. Ways out: upgrade or downgrade one of the two so their ranges overlap, look for a newer release that relaxes the constraint, replace one package, or isolate the two in different environments or processes. Do not use options that skip the resolver, because they produce environments that import fine and fail strangely later.

### Why ranges in `pyproject.toml` and exact versions in the lock

A library that demanded `requests==2.34.2` would clash with every other package that wanted a different version. So projects declare **what they accept** (`requests>=2.34`) and let the **lock file** record what they actually **tested with**. Declaring an upper bound is a judgment call: it prevents a future major release from breaking you, but it also blocks users who need to upgrade that dependency. Many libraries leave out upper bounds unless a known break exists, and rely on tests and quick fixes. Applications are different: they pin completely through the lock file.

### Pinning in `requirements.txt` versus a real lock

`pip freeze` captures one environment on one platform, and does not record **why** each package is present, nor hashes by default. A proper lock file resolves for **all** supported platforms and Python versions at once and includes hashes. If you must use `requirements.txt`, generate it with a tool (`uv pip compile`, `pip-compile`) from a short list of direct dependencies, and consider installing with `--require-hashes`.

### Environment markers and extras

A single requirements line can adapt to the environment: `tomli>=2; python_version < "3.11"` installs only on older Pythons, and `requests[socks]` pulls in the optional extra dependencies of that package. Lock files record the result for every combination.

### Installing from other sources

pip can also install from a local folder, a built wheel file, or a Git repository (`pip install git+https://...`). These bypass the safety of an index and make reproducibility harder. Use them for development, not for production dependencies, unless you pin an exact commit and trust the source.

### Updating dependencies

A sensible routine is to update **small and often**:

1. See what is behind: `python -m pip list --outdated`, or let a bot such as Dependabot or Renovate open pull requests.
2. Update one package (or one related group) at a time with `uv lock --upgrade-package name`.
3. Read the changelog, especially for major versions and anything that touches security.
4. Run the tests and the type checker.
5. Commit the changed lock file.

Postponing all updates for a year makes the eventual jump large and risky.

### Vulnerability scanning

Tools such as `pip-audit` compare the exact versions you use against databases of known vulnerabilities and report the affected packages and fixed versions. Run one in your continuous integration system. A lock file makes the scan precise, because it knows exactly which versions are deployed.

### Supply-chain attacks

Attackers publish malicious packages with names close to popular ones, take over abandoned accounts, or slip code into an update of a legitimate package. Defenses are the habits above: few dependencies, locked and hashed installs, deliberate updates, scanning, and reading what a new dependency is before adding it. Installing only wheels (`--only-binary :all:`) avoids running a package's build code, though the package's own code still runs when you import it.

### Licences

Every package has a licence, and some impose obligations on you or your users. Permissive licences (MIT, BSD, Apache-2.0) are generally easy to use. Others, such as the GPL family, require more thought if you distribute software that includes them. Your organization may have a policy, and tools can list the licences of everything in your environment.

### Vendoring and alternatives

Sometimes copying a small, stable piece of code into your project (with its licence) is better than adding a dependency. And sometimes the standard library does the job: `urllib.request` is enough for a trivial request, `argparse` for command-line options, `sqlite3` for a small database, `zoneinfo` for time zones. Each avoided dependency is one less thing that can break or be attacked.

### Removing what you no longer use

Dependencies accumulate. Remove ones you stopped using (`uv remove name`, then rerun the tests), and check from time to time that every direct dependency is still imported somewhere.

## Industry-level snippet

A deployment pipeline should refuse to start a service when the environment does not match what was tested. This module checks the installed packages against a file of exact pins, such as the output of `pip freeze` or a compiled requirements file. It parses the pins carefully, normalizes package names the way the packaging standards require, accepts a replaceable lookup function so it can be tested without installing anything, and returns exit codes that a deployment script can act on.

```python
"""Verify that installed packages match the exact pins in a requirements file.

Usage:
    python check_requirements.py requirements.txt

Exit codes:
    0  every pinned package is installed at the pinned version
    1  at least one package is missing or has a different version
    2  the file cannot be read or contains a line that is not an exact pin
"""

import logging
import sys
from collections.abc import Callable
from dataclasses import dataclass
from importlib.metadata import PackageNotFoundError, version
from pathlib import Path

__all__ = ["Pin", "check_pins", "main", "normalize_name", "parse_pins"]

logger = logging.getLogger(__name__)

EXIT_OK = 0
EXIT_MISMATCH = 1
EXIT_USAGE = 2

INVALID_VERSION_CHARS = frozenset("<>=!~, ")  # characters that mean "not an exact pin"


@dataclass(frozen=True)
class Pin:
    """A requirement pinned to one exact version."""

    name: str
    version: str


def normalize_name(name: str) -> str:
    """Return the canonical form of a package name: lowercase, with dashes.

    Names such as "Charset_Normalizer" and "charset.normalizer" both become
    "charset-normalizer", as the packaging standards require.
    """
    cleaned = name.strip().lower().replace("_", "-").replace(".", "-")
    while "--" in cleaned:
        cleaned = cleaned.replace("--", "-")
    return cleaned


def parse_pins(text: str) -> list[Pin]:
    """Read "name==version" lines, skipping comments, blank lines, and options.

    Extras such as requests[socks] and environment markers after a semicolon
    are ignored, because only the name and exact version matter here.

    Raises:
        ValueError: If a line is not a simple exact pin.
    """
    pins: list[Pin] = []
    for number, raw_line in enumerate(text.splitlines(), start=1):
        content = raw_line.split("#", 1)[0].split(";", 1)[0]  # drop comment, marker
        line = content.rstrip().rstrip("\\").strip()  # drop a trailing continuation
        if not line or line.startswith("-"):
            continue  # blank, comment-only, or an option such as -r or --hash
        name, separator, pinned = line.partition("==")
        name = name.split("[", 1)[0].strip()
        pinned = pinned.strip()
        if not (separator and name and pinned) or INVALID_VERSION_CHARS & set(pinned):
            raise ValueError(
                f"line {number}: expected an exact pin like name==1.2.3, "
                f"got {raw_line.strip()!r}"
            )
        pins.append(Pin(normalize_name(name), pinned))
    return pins


def check_pins(pins: list[Pin], lookup: Callable[[str], str] = version) -> list[str]:
    """Compare pins with installed versions and describe every difference.

    Args:
        pins: The exact requirements to check.
        lookup: Returns the installed version of a package, or raises
            PackageNotFoundError. Replace it in tests.

    Returns:
        One message per problem. An empty list means everything matches.
    """
    problems: list[str] = []
    for pin in pins:
        try:
            installed = lookup(pin.name)
        except PackageNotFoundError:
            problems.append(f"{pin.name}: not installed (pinned {pin.version})")
            continue
        if installed != pin.version:
            problems.append(f"{pin.name}: installed {installed}, pinned {pin.version}")
    return problems


def main(argv: list[str]) -> int:
    """Check the requirements file named on the command line."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    if len(argv) != 2:
        logger.error("usage: check_requirements.py REQUIREMENTS_FILE")
        return EXIT_USAGE
    try:
        pins = parse_pins(Path(argv[1]).read_text(encoding="utf-8"))
    except (OSError, ValueError) as exc:
        logger.error("%s", exc)
        return EXIT_USAGE
    problems = check_pins(pins)
    for problem in problems:
        logger.error("%s", problem)
    if problems:
        return EXIT_MISMATCH
    logger.info("all %d pinned packages match", len(pins))
    return EXIT_OK


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Try it with a made-up set of installed packages, so the example runs anywhere:

```python
from importlib.metadata import PackageNotFoundError

from check_requirements import check_pins, parse_pins

requirements = """
# direct dependencies
requests==2.34.2
urllib3==2.8.0           # transitive, pinned for safety
Charset_Normalizer==3.5.2
pytest==9.2.0 ; python_version >= "3.9"
-r other.txt
"""

pins = parse_pins(requirements)
print(pins)

installed = {"requests": "2.34.2", "urllib3": "2.7.0", "charset-normalizer": "3.5.2"}


def fake_lookup(name: str) -> str:
    try:
        return installed[name]
    except KeyError:
        raise PackageNotFoundError(name) from None


for problem in check_pins(pins, fake_lookup):
    print(problem)

try:
    parse_pins("requests>=2\n")
except ValueError as exc:
    print("ValueError:", exc)
```

```text
[Pin(name='requests', version='2.34.2'), Pin(name='urllib3', version='2.8.0'), Pin(name='charset-normalizer', version='3.5.2'), Pin(name='pytest', version='9.2.0')]
urllib3: installed 2.7.0, pinned 2.8.0
pytest: not installed (pinned 9.2.0)
ValueError: line 1: expected an exact pin like name==1.2.3, got 'requests>=2'
```

Run against a real environment, inside the virtual environment where `requests` was installed, with the output of `pip freeze` saved as `requirements.txt`:

```bash
python check_requirements.py requirements.txt
echo $?
```

```text
INFO all 5 pinned packages match
0
```

After changing a pin in the file to `requests==2.0.0`:

```text
ERROR requests: installed 2.34.2, pinned 2.0.0
```

and the exit code becomes `1`.

Why it is written this way:

- `parse_pins` accepts only **exact** pins and rejects anything else with the line number and the offending text, because a check that silently skips lines it does not understand gives a false sense of safety.
- `normalize_name` applies the standard naming rule, so `Charset_Normalizer` in a file matches `charset-normalizer` in the environment.
- `check_pins` returns descriptions rather than printing or exiting, so it is easy to test and to reuse. The `lookup` parameter is the seam that lets a test supply a fake environment.
- `importlib.metadata.version` reads the installed distribution's metadata without importing the package, so the check is fast and has no side effects.
- The exit codes distinguish "environment does not match" (`1`) from "the file is unusable" (`2`), which a pipeline can treat differently.

## Common mistakes

1. **Installing packages globally** instead of in a virtual environment.
2. **Guessing the install name from the import name,** or the reverse. `pip install PIL` does not install Pillow.
3. **Leaving the quotes off a version range** such as `requests>=2` in the shell, which creates a file called `=2`.
4. **Not locking an application's dependencies,** so that builds change from week to week without any change to your code.
5. **Pinning a library's dependencies exactly,** which makes it impossible to install alongside other packages.
6. **Hand-editing a lock file.** Let the tool regenerate it.
7. **Adding a dependency without checking it,** including its maintenance, licence, and its own dependencies.
8. **Installing look-alike package names** from a typo, or from an unfamiliar tutorial.
9. **Ignoring `pip check` and resolver errors,** or forcing the installation past them.
10. **Never updating,** then facing a huge, risky upgrade or an unpatched vulnerability.
11. **Following documentation for the wrong version** of a package.
12. **Mixing a public index and a private index carelessly,** which allows dependency confusion.
13. **Putting development tools in the run-time dependencies,** bloating and widening the attack surface of the production environment.
14. **Committing the `.venv` folder.** Commit `pyproject.toml` and the lock file instead.

## Practice exercises

1. Create a virtual environment, install `requests`, and run `pip show`, `pip list`, `pip freeze`, and `pip check`. Explain each line of `Requires` and `Required-by`.
2. Find the import name for each of these distribution names, using the packages' documentation: `pillow`, `pyyaml`, `beautifulsoup4`, `python-dateutil`, `scikit-learn`.
3. Write down what each specifier accepts: `>=2.31,<3`, `~=2.31`, `==2.34.*`, `!=2.32.0`. Then test your answers with `pip install --dry-run`.
4. Make a small project with `uv init`, add one runtime dependency and one development dependency, and read `pyproject.toml` and `uv.lock`. Identify the direct and the transitive dependencies with `uv tree`.
5. Delete your `.venv` folder and recreate it from the lock file. Confirm that `pip freeze` (or `uv pip freeze`) shows the same versions as before.
6. Use `uv lock --upgrade-package` to move one package to its newest version. Compare the lock file before and after using `git diff`.
7. Create two requirement lists that conflict (find two packages whose declared versions of a shared dependency do not overlap, or invent the situation with a small local example). Read the resolver's message and describe two ways out.
8. Pick a package you do not know from PyPI, and evaluate it using the checklist in this note. Write a short paragraph recommending for or against adding it to a project.
9. Find three names that are one typo away from a very popular package, and explain why each is a risk. Do not install them.
10. Use `importlib.metadata.requires` to list what two different installed packages need, and explain the specifiers you see, including any extras and markers.
11. Run `pip-audit` (install it into a throwaway environment) against a requirements file that pins an old version of a popular package, and read the report.
12. Extend `check_requirements.py` so that it can also verify **hashes**: parse `--hash=sha256:...` options, compare them with the digest of a wheel file you provide, and add a `--strict` flag that makes missing hashes an error. Write tests for the parsing and for each outcome.

## Summary

- **PyPI** hosts third-party packages. `pip` and `uv` install them into the current environment, always a virtual environment. Prefer **wheels**, which run no build code.
- The distribution name you install can differ from the module name you import.
- Versions use `MAJOR.MINOR.PATCH`. Specifiers such as `==`, `>=`, `<`, `~=`, and `!=` express which versions you accept. Quote them in the shell.
- **Direct** dependencies are what you ask for. **Transitive** dependencies are what those need.
- **Declared** ranges (`pyproject.toml`) say what you are compatible with. **Pinned** and **locked** versions say exactly what you tested. **Applications** commit a lock file. **Libraries** declare sensible ranges.
- `uv add`, `uv lock`, `uv sync`, `uv run`, and `uv tree` manage a project and its lock file. Keep development tools in a separate dependency group.
- Read documentation for **the version you installed**, and use `help()` and `dir()` to see what is really there.
- Evaluate a package before adopting it: need, maintenance, ownership, documentation, tests, licence, Python support, typing, weight, and security record. Watch for typosquatting.
- Reduce risk with locked, hashed installs, regular small updates, vulnerability scanning, review of new dependencies, and as few dependencies as the job needs.
