## Overview

Python is not built into your computer in a form you can rely on. You install it, verify it, and learn how to manage more than one version, because real projects often need different versions on the same machine.

This roadmap targets **Python 3.12 or newer**. At the time of writing, Python 3.15 is the newest release, with 3.12, 3.13, and 3.14 all still widely used. Any of those is fine for learning. If you want the safest choice, pick a version one or two releases behind the newest, because third-party libraries sometimes take a few months to support a brand-new release.

By the end of this note you will be able to:

- Install Python on Windows, macOS, or Linux.
- Verify the installation and find out which Python your terminal is using.
- Explain what `PATH` is and why a wrong `PATH` causes "command not found" errors.
- Install `uv` and use it to download and switch between Python versions.

## Core concepts

### What "installing Python" means

Installing Python puts three things on your machine:

1. The **interpreter**, the program that runs your code.
2. The **standard library**, a large collection of ready-made modules (`json`, `pathlib`, `sqlite3`, and many more).
3. Tools like `pip`, the package installer, and `venv`, the virtual environment creator.

It also makes the interpreter reachable from your terminal by adding it to `PATH`.

### PATH

`PATH` is a list of folders the shell searches, in order, when you type a command. If Python's folder is not in that list, the shell answers with an error such as:

```text
'python' is not recognized as an internal or external command
```

or

```text
command not found: python
```

Many installation problems are really PATH problems.

### The names of the command

| System | Command you will usually type |
| --- | --- |
| Windows | `python` or `py` |
| macOS | `python3` |
| Linux | `python3` |

On macOS and Linux, plain `python` often does not exist at all, or points to something old. Using `python3` avoids ambiguity. Inside a virtual environment (next chapters), `python` always works.

### Installing on Windows

There are two official routes from python.org.

**Route 1: Python install manager (recommended for new installs).** This is the current official way. It provides the `py` and `python` commands and can install and switch between many versions. You can get it from the Microsoft Store (search for "Python install manager"), through `winget`, or from the download page at python.org. After installing it, open a new PowerShell window.

```powershell
py install 3.13
py list
python --version
```

**Route 2: The traditional installer (.exe).** Still available for current releases from python.org. During setup:

1. Tick the box **Add python.exe to PATH** on the first screen. This is the single most important click.
2. Choose Install Now.
3. Open a new terminal window afterwards. Terminals opened before the install do not see the new `PATH`.

The traditional installer includes the **`py` launcher**, a small program that lets you choose a version: `py -3.12 script.py` runs a script with Python 3.12 even if 3.13 is also installed.

> [!warning] A common Windows trap
> If Python is not installed, typing `python` may open the Microsoft Store instead of showing an error. That is Windows redirecting you, not Python running. Install Python, then open a fresh terminal.

### Installing on macOS

Two good options.

**Option A: the python.org installer.** Download the macOS installer package from python.org and double-click it. It installs into `/Library/Frameworks/Python.framework` and gives you the `python3` command. If the installation folder shows a script named `Install Certificates.command`, run it once so Python can verify HTTPS certificates.

**Option B: Homebrew.** If you use the Homebrew package manager:

```bash
brew install python@3.13
```

Do not use or modify the Python that macOS itself might ship with. Treat it as belonging to the operating system.

### Installing on Linux

Most distributions already include Python 3 because system tools depend on it. You usually need to add the extra pieces.

| Distribution family | Command |
| --- | --- |
| Debian, Ubuntu | `sudo apt update` then `sudo apt install python3 python3-venv python3-pip` |
| Fedora, RHEL, Rocky | `sudo dnf install python3 python3-pip` |
| Arch | `sudo pacman -S python python-pip` |

The version you get depends on the distribution release. An older Ubuntu release may ship Python older than 3.12. In that case, use `uv` (below) to install the version you need without touching the system Python.

> [!note] Why `python3-venv` is a separate package on Debian and Ubuntu
> Those distributions split `venv` out of the main package. If `python3 -m venv .venv` fails later with a message about `ensurepip`, installing `python3-venv` fixes it.

### Verifying the installation

```powershell
python --version
py --version
Get-Command python
```

```bash
python3 --version
which python3
```

Both of these answer two questions: which version, and which file on disk. If the version is not what you expected, you are probably running a different Python than the one you just installed. A universal check that works anywhere:

```bash
python3 -c "import sys; print(sys.version); print(sys.executable)"
```

On Windows, use `python` or `py` in place of `python3`.

### Installing uv

`uv` is a fast, modern tool for installing Python versions, creating virtual environments, and installing packages. It is written in Rust and replaces several older tools at once.

macOS and Linux:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Windows (PowerShell):

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Alternatives: `brew install uv` on macOS, or `pip install uv` into any Python. Open a new terminal and check:

```bash
uv --version
```

### Managing multiple Python versions

With uv:

```bash
uv python list              # show available and installed versions
uv python install 3.13      # download and install 3.13
uv python install 3.12 3.14 # install several at once
uv run --python 3.12 python --version
```

Other ways to pick a version:

| Method | Example |
| --- | --- |
| Windows launcher or install manager | `py -3.12 script.py` |
| Versioned executable (macOS, Linux) | `python3.12 script.py` |
| Project pin file | `uv python pin 3.12` writes a `.python-version` file |
| Virtual environment | `uv venv --python 3.12` creates an environment using that version |

## Basic syntax examples

### Check your install in under a minute

```bash
python3 --version
python3 -c "print('Python is working')"
```

Expected output (your version number will differ):

```text
Python 3.13.5
Python is working
```

### See which interpreter a command uses

```powershell
Get-Command python | Select-Object Source
```

```bash
which python3
```

On Windows, if you also have `where.exe` available, `where.exe python` lists every `python` found on `PATH`. Note the `.exe`: plain `where` in PowerShell is a different command.

### Install and use a second Python version with uv

```bash
uv python install 3.12
uv run --python 3.12 python -c "import sys; print(sys.version_info[:2])"
```

```text
(3, 12)
```

### Pin a version for a project

```bash
mkdir demo-project
cd demo-project
uv python pin 3.12
cat .python-version
```

```text
3.12
```

## Deeper behavior

### Why PATH order matters

If three Pythons are installed, the first one found on `PATH` wins. When you activate a virtual environment (Chapter 0.5), its folder is placed at the front of `PATH` for that terminal session, which is why `python` suddenly means "the environment's Python."

### Why you should never modify the system Python

On macOS and Linux, parts of the operating system run on the system Python. Installing or upgrading packages into it can break those tools. Modern Linux distributions enforce this with a rule from PEP 668. If you try `pip install something` against the system Python, you see:

```text
error: externally-managed-environment
```

That message is protecting you. The correct fix is a virtual environment, not forcing the install.

### Python versions and the release cycle

Python releases a new feature version every year in October. Each version receives bug fixes for a couple of years and security fixes for about five years in total. Projects usually declare a **minimum** supported version, which is why this roadmap says 3.12 or newer. Newer syntax, such as the `type` statement and generic function syntax like `def f[T](x: T)`, requires 3.12.

### What the `py` command is on Windows

The `py` command (from the install manager or the older launcher) is not Python itself. It finds installed Pythons and starts the right one. It also reads the first line of a script (the "shebang") to choose a version, mirroring behavior on Linux and macOS.

### Why open a new terminal after installing

Each terminal copies `PATH` when it starts. Changes made afterwards are invisible to it. Close and reopen the terminal after installing Python or `uv`.

## Industry-level snippet

Teams often add a check script that runs on a new developer's laptop and in CI to confirm the machine meets the project's requirements. This one verifies the version, the presence of key standard library modules, and whether `uv` is available. It reports problems clearly and returns a proper exit code.

```python
"""Verify that this machine meets the Python requirements of the project."""

import importlib.util
import logging
import shutil
import sys
from dataclasses import dataclass

MINIMUM_VERSION: tuple[int, int] = (3, 12)
REQUIRED_MODULES: tuple[str, ...] = ("venv", "sqlite3", "ssl")

logger = logging.getLogger("check_python")


@dataclass(frozen=True)
class CheckResult:
    """The outcome of one environment check."""

    name: str
    passed: bool
    detail: str
    required: bool = True


def check_version() -> CheckResult:
    """Check that the running interpreter is new enough."""
    current = sys.version_info[:2]
    passed = current >= MINIMUM_VERSION
    detail = (
        f"running {current[0]}.{current[1]}, "
        f"need {MINIMUM_VERSION[0]}.{MINIMUM_VERSION[1]} or newer"
    )
    return CheckResult("python version", passed, detail)


def check_modules() -> list[CheckResult]:
    """Check that important standard library modules can be imported."""
    results: list[CheckResult] = []
    for module_name in REQUIRED_MODULES:
        found = importlib.util.find_spec(module_name) is not None
        detail = "available" if found else "missing, reinstall or add the OS package"
        results.append(CheckResult(f"module {module_name}", found, detail))
    return results


def check_uv() -> CheckResult:
    """Check whether uv is on PATH. This one is optional."""
    location = shutil.which("uv")
    detail = location if location else "not found on PATH (optional)"
    return CheckResult("uv", location is not None, detail, required=False)


def run_checks() -> list[CheckResult]:
    """Run every check and return the results."""
    return [check_version(), *check_modules(), check_uv()]


def main() -> int:
    """Run all checks, log a report, and return an exit code."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    logger.info("Interpreter: %s", sys.executable)

    failed_required = 0
    for result in run_checks():
        status = "PASS" if result.passed else "FAIL"
        logger.info("%s  %-16s %s", status, result.name, result.detail)
        if not result.passed and result.required:
            failed_required += 1

    if failed_required:
        logger.error("%d required check(s) failed", failed_required)
        return 1
    logger.info("Environment looks good")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Things to notice:

- `MINIMUM_VERSION` is a constant written in capitals so it is easy to find and change.
- `sys.version_info[:2]` is a tuple like `(3, 12)`, and tuples compare naturally, so `(3, 13) >= (3, 12)` is true.
- Optional and required checks are separated. A missing optional tool does not fail the script.
- The script prints a report, and the exit code lets automation decide whether to continue.

## Common mistakes

1. **Skipping "Add python.exe to PATH" with the traditional Windows installer.** Then `python` is not recognized. Re-run the installer and choose Modify, or use the install manager instead.
2. **Not opening a new terminal after installing.** The old window still has the old `PATH`.
3. **Using `python` on macOS or Linux and getting an error or an old version.** Use `python3`.
4. **Using `sudo pip install` on Linux or macOS.** This can break system tools. Use a virtual environment.
5. **Installing a version older than 3.12 and then following a roadmap that needs newer features.** Check `--version` first.
6. **Not knowing which Python is running.** Always check with `which python3`, `Get-Command python`, or `sys.executable`.
7. **Installing Python from random websites.** Use python.org, your OS package manager, Homebrew, the Microsoft Store listing for the install manager, or `uv`.
8. **Mixing up `pip` and `python -m pip`.** The first may belong to a different Python than the second. Prefer `python -m pip` (Chapter 0.5).

## Practice exercises

1. Install Python 3.12 or newer on your machine, then print its exact version from the terminal.
2. Find the full path to the interpreter your terminal uses when you type `python` (Windows) or `python3` (macOS and Linux).
3. Explain in your own words what `PATH` is, then print your own `PATH` entries one per line using a command or one-line Python expression.
4. Install `uv`, verify it, and list the Python versions it knows about.
5. Use `uv` to install a second Python version that differs from your main one. Run a one-line command that proves it is a different version.
6. Create an empty folder, pin it to a specific Python version with `uv`, and read the file that was created.
7. Run the industry-level snippet. Change `MINIMUM_VERSION` to one version higher than yours and observe the failure and the exit code.
8. Add a new required module of your choice to `REQUIRED_MODULES` and run the script again. Then invent a module name that does not exist and observe what the report says.
9. Write a short script that prints the Python version, operating system, and the full interpreter path on three separate lines using `sys` and `platform`.
10. Deliberately try a command that Python is not installed for (for example `python3.99 --version`). Read the error and explain which part of the shell produced it.

## Summary

- Installing Python adds the interpreter, the standard library, and tools like `pip` and `venv`.
- The shell finds programs through `PATH`. Wrong `PATH` means "command not found."
- On Windows use the Python install manager or the traditional installer with the PATH box ticked. On macOS use python.org or Homebrew. On Linux use `apt` or `dnf` and add `python3-venv` on Debian-family systems.
- Use `python3` on macOS and Linux, and `python` or `py` on Windows.
- Verify every install with `--version` and find the interpreter path with `which`, `Get-Command`, or `sys.executable`.
- Never alter the operating system's Python. Use virtual environments instead.
- `uv` installs and switches Python versions quickly, and `uv python pin` records a project's version.
