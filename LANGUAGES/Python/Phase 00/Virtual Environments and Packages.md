## Overview

Python becomes powerful through **packages**: code written by other people that you install and use. Web frameworks, data tools, database drivers, and testing tools are all packages. A **virtual environment** is a private, disposable folder where one project's packages live, so projects never interfere with each other or with your operating system.

By the end of this note you will be able to:

- Explain why every project should have its own virtual environment.
- Create, activate, and deactivate a virtual environment on Windows, macOS, and Linux.
- Install, list, upgrade, and remove packages with `pip`.
- Save and restore a project's dependencies with `requirements.txt`.
- Do the same tasks faster with `uv venv` and `uv pip`.

> [!note] Terminology
> A **package** or **dependency** is third-party code you install. **PyPI** (the Python Package Index at pypi.org) is the public website that hosts them. **pip** is the standard tool that downloads from PyPI. A **requirement** is a line stating which package and version a project needs.

## Core concepts

### Why isolation matters

Imagine two projects on one computer.

- Project A was written two years ago and needs version 1 of a library.
- Project B is brand new and needs version 3 of the same library.

Only one version can be installed in a single place at a time. If both projects share one global Python, one of them breaks. A virtual environment gives each project its own space.

Isolation also gives you:

- **Safety.** Experiments cannot damage your operating system's Python.
- **Reproducibility.** You can describe exactly which packages a project needs, and a teammate or server can rebuild the same setup.
- **Cleanliness.** To remove a project's packages, delete one folder.

### What a virtual environment is

It is an ordinary folder, conventionally named `.venv`, inside your project. It contains:

- A link or copy of a Python interpreter.
- An empty `site-packages` folder, where installed packages go.
- Activation scripts.
- A small file `pyvenv.cfg` recording which Python created it.

| Item | Windows | macOS and Linux |
| --- | --- | --- |
| Interpreter and scripts | `.venv\Scripts\` | `.venv/bin/` |
| Installed packages | `.venv\Lib\site-packages\` | `.venv/lib/python3.x/site-packages/` |

### Creating one with venv

`venv` is built into Python.

```bash
python -m venv .venv
```

Use `python3` instead of `python` on macOS and Linux when you are outside an environment.

### Activating and deactivating

Activation tells your current terminal to use the environment's Python and pip.

| Shell | Activate command |
| --- | --- |
| PowerShell (Windows) | `.venv\Scripts\Activate.ps1` |
| Command Prompt (Windows) | `.venv\Scripts\activate.bat` |
| zsh or bash (macOS, Linux) | `source .venv/bin/activate` |
| fish | `source .venv/bin/activate.fish` |

To leave the environment, type `deactivate` in any of them. When active, your prompt shows `(.venv)` at the start.

> [!warning] PowerShell may block the script
> On a fresh Windows machine, running `Activate.ps1` can fail with an error about scripts being disabled. Allow locally created scripts for your user once:
> `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`
> Then open a new terminal and try again.

### Installing packages with pip

Always use `python -m pip` so the package goes into the interpreter you intend.

| Task | Command |
| --- | --- |
| Install a package | `python -m pip install requests` |
| Install an exact version | `python -m pip install requests==2.32.3` |
| Install within a range | `python -m pip install "requests>=2.31,<3"` |
| Upgrade a package | `python -m pip install --upgrade requests` |
| Remove a package | `python -m pip uninstall requests` |
| List installed packages | `python -m pip list` |
| Show details about one | `python -m pip show requests` |
| Upgrade pip itself | `python -m pip install --upgrade pip` |

The quotes around `"requests>=2.31,<3"` are required. Without them the shell treats `>` and `<` as redirection symbols.

### requirements.txt

`requirements.txt` is a plain text file with one requirement per line. It lets anyone recreate your environment.

```text
requests==2.32.3
rich>=13
```

Two commands do the work:

```bash
python -m pip freeze > requirements.txt
python -m pip install -r requirements.txt
```

`freeze` writes every installed package with its exact version. `install -r` reads the file and installs everything in it.

### uv: the faster alternative

`uv` (installed in Chapter 0.2) offers the same functions with much higher speed and a few extras.

| Task | pip way | uv way |
| --- | --- | --- |
| Create environment | `python -m venv .venv` | `uv venv` |
| Create with a Python version | run that version, for example `python3.12 -m venv .venv` | `uv venv --python 3.12` |
| Install a package | `python -m pip install requests` | `uv pip install requests` |
| Install from file | `python -m pip install -r requirements.txt` | `uv pip install -r requirements.txt` |
| List installed | `python -m pip list` | `uv pip list` |
| Freeze | `python -m pip freeze` | `uv pip freeze` |

`uv pip` works against the environment named `.venv` in the current folder, or the active one, even without activating it first.

## Basic syntax examples

### Complete first-time workflow on macOS or Linux

```bash
mkdir demo-env
cd demo-env
python3 -m venv .venv
source .venv/bin/activate
python -m pip install requests
python -m pip list
python -c "import requests; print(requests.__version__)"
python -m pip freeze > requirements.txt
cat requirements.txt
deactivate
```

### Complete first-time workflow on Windows PowerShell

```powershell
mkdir demo-env
cd demo-env
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install requests
python -m pip list
python -c "import requests; print(requests.__version__)"
python -m pip freeze > requirements.txt
Get-Content requirements.txt
deactivate
```

### Prove that isolation works

```bash
python3 -c "import requests"
```

Run this outside the environment, and if `requests` is not installed globally you will see:

```text
ModuleNotFoundError: No module named 'requests'
```

Activate the environment and run the same command, and it succeeds. The package exists only inside `.venv`.

### Recreating an environment from scratch

A teammate clones your project and has no `.venv`. They run:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

### The same with uv

```bash
uv venv
uv pip install requests
uv pip freeze > requirements.txt
uv pip install -r requirements.txt
```

Because uv finds `.venv` automatically, no activation is needed for these commands. You still activate when you want to type plain `python` and have it use the environment.

## Deeper behavior

### What activation really does

Activation does very little. It adds `.venv/bin` (or `.venv\Scripts`) to the front of your `PATH` for that terminal session and sets a variable named `VIRTUAL_ENV`. Because that folder is now first, typing `python` finds the environment's interpreter before any other. You can skip activation completely by running the interpreter by its full path:

```bash
.venv/bin/python script.py
```

```powershell
.venv\Scripts\python.exe script.py
```

Servers and scheduled jobs often do exactly this.

### Detecting an environment from Python

Inside a virtual environment, `sys.prefix` differs from `sys.base_prefix`:

```python
import sys

in_venv = sys.prefix != sys.base_prefix
print("Inside a virtual environment:", in_venv)
```

### Direct dependencies versus transitive dependencies

You install `requests`. But `requests` needs other packages (for example `urllib3` and `certifi`). Those are **transitive dependencies**. `pip freeze` lists everything, direct and transitive, which is good for exact reproduction and noisy for humans. Many teams keep a short hand-written list of direct dependencies and generate the full pinned file from it:

```bash
uv pip compile requirements.in -o requirements.txt
```

Phase 4 returns to lock files and version pinning.

### Environments are not portable

Do not copy a `.venv` folder to another computer or move it to another location. It stores absolute paths. Recreate it from `requirements.txt` instead. Likewise, never commit `.venv` to Git.

### Externally managed environments

On Debian, Ubuntu, and Homebrew Python, pip may refuse to install into the system interpreter:

```text
error: externally-managed-environment
```

This is intentional. Create a virtual environment and install there.

### Why you do not use sudo with pip

Running `sudo pip install` changes files owned by the operating system and can break system tools. If a command seems to need `sudo`, you almost certainly want a virtual environment instead.

### uv environments do not include pip by default

`uv venv` creates an environment without `pip` inside it. Use `uv pip install ...` instead of `python -m pip install ...`, or create the environment with `uv venv --seed` if you want pip included.

### Where pip downloads from

By default pip downloads from PyPI. Packages can contain arbitrary code that runs on your machine. Check names carefully, because attackers publish packages with names similar to popular ones. Chapter 4.4 covers how to evaluate package quality.

## Industry-level snippet

New developers and CI machines need a one-command setup. This script creates a virtual environment if it does not exist and installs the project's dependencies. It works on every operating system, uses logging, validates its inputs, and distinguishes different failures with exit codes.

```python
"""Create a virtual environment and install the project's dependencies.

Usage:
    python bootstrap_env.py
"""

import logging
import subprocess
import sys
import venv
from pathlib import Path

logger = logging.getLogger("bootstrap")

PROJECT_ROOT = Path(__file__).resolve().parent
VENV_DIR = PROJECT_ROOT / ".venv"
REQUIREMENTS_FILE = PROJECT_ROOT / "requirements.txt"

EXIT_OK = 0
EXIT_INSTALL_FAILED = 1
EXIT_MISSING_FILE = 2


def venv_python(venv_dir: Path) -> Path:
    """Return the path to the interpreter inside a virtual environment."""
    if sys.platform == "win32":
        return venv_dir / "Scripts" / "python.exe"
    return venv_dir / "bin" / "python"


def create_venv(venv_dir: Path) -> None:
    """Create the virtual environment unless one already exists."""
    if venv_python(venv_dir).exists():
        logger.info("Virtual environment already exists at %s", venv_dir)
        return
    logger.info("Creating virtual environment at %s", venv_dir)
    venv.EnvBuilder(with_pip=True).create(venv_dir)


def install_requirements(python: Path, requirements: Path) -> None:
    """Install every package listed in the requirements file."""
    if not requirements.is_file():
        raise FileNotFoundError(f"Missing requirements file: {requirements}")
    command = [str(python), "-m", "pip", "install", "--requirement", str(requirements)]
    logger.info("Running: %s", " ".join(command))
    subprocess.run(command, check=True)


def main() -> int:
    """Set up the environment and return a process exit code."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    try:
        create_venv(VENV_DIR)
        install_requirements(venv_python(VENV_DIR), REQUIREMENTS_FILE)
    except FileNotFoundError as exc:
        logger.error("%s", exc)
        return EXIT_MISSING_FILE
    except subprocess.CalledProcessError as exc:
        logger.error("A setup command failed with exit code %s", exc.returncode)
        return EXIT_INSTALL_FAILED

    logger.info("Setup complete")
    return EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Notes on the design:

- `Path(__file__).resolve().parent` anchors every path to the script's own folder, so it works no matter where the user runs it from.
- `subprocess.run(..., check=True)` raises an exception when pip fails, instead of silently continuing.
- Running pip through the environment's own interpreter means activation is not needed.
- A real project would also pin Python's version and print the activation command at the end.

## Common mistakes

1. **Installing packages globally by accident.** Always look for `(.venv)` in your prompt, or use `python -m pip` with the environment's Python.
2. **Forgetting to activate after opening a new terminal.** Activation lasts for one terminal session only.
3. **Committing `.venv` to Git.** Add it to `.gitignore` (Chapter 0.6) and rely on `requirements.txt`.
4. **Skipping quotes around version ranges.** `pip install requests>=2` writes a file named `=2` instead of installing a range.
5. **Using bare `pip`.** It may belong to a different Python. Use `python -m pip`.
6. **Editing `requirements.txt` by hand and breaking it.** Prefer regenerating it after installing, and verify by installing from it in a fresh environment.
7. **Moving or renaming the project folder with `.venv` inside.** Delete `.venv` and recreate it.
8. **Using `sudo pip`.** Never. Create a virtual environment.
9. **Naming the folder inconsistently.** Use `.venv` so editors and tools find it.
10. **Running `uv pip` and `pip` interchangeably without understanding that they are different tools.** Pick one workflow per project and stay with it.

## Practice exercises

1. Create a folder named `env-practice` and a virtual environment inside it using `venv`. List the contents of `.venv` and identify where the interpreter lives on your system.
2. Activate the environment. Prove it is active in two different ways.
3. Install the `requests` package. Print its version from Python, then show where pip says it is located.
4. Deactivate and try importing `requests` again. Explain the result.
5. Create `requirements.txt` from your environment. Open it and explain why it contains more packages than the one you installed.
6. Delete `.venv`, create a new one, and restore your dependencies from `requirements.txt` only.
7. Install a package at a specific older version, then upgrade it to the latest, then remove it. Check `pip list` after each step.
8. Write a short Python script that prints whether it is running inside a virtual environment, and run it both inside and outside.
9. Repeat exercises 1 to 6 using `uv venv` and `uv pip`. Note one difference you observe.
10. Extend the industry-level snippet so that it accepts an optional path to a different requirements file as a command-line argument and falls back to the default when none is given.

## Summary

- A virtual environment is a per-project folder, usually `.venv`, holding its own interpreter link and packages.
- Isolation prevents version conflicts, protects the system Python, and makes setups reproducible.
- Create with `python -m venv .venv` or `uv venv`, activate with the script for your shell, and leave with `deactivate`.
- Install with `python -m pip install` or `uv pip install`, and always quote version ranges.
- `pip freeze > requirements.txt` records a setup and `pip install -r requirements.txt` rebuilds it.
- Activation only changes `PATH`, so you can also run `.venv/bin/python` directly.
- Never copy, move, or commit `.venv`, and never use `sudo pip`.
