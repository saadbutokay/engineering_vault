## Overview

You can write Python in any text editor, but a good code editor catches mistakes as you type, runs your code with one click, and lets you pause a running program to inspect it. This roadmap uses **Visual Studio Code** (VS Code), a free editor from Microsoft that is popular in professional Python work.

By the end of this note you will be able to:

- Install VS Code and the Python extension.
- Open a project folder and select the correct Python interpreter.
- Run a file from the editor and from the integrated terminal.
- Set breakpoints and use the debugger to step through code line by line.
- Store shared editor settings inside a project so a whole team gets the same setup.

> [!note] Editor versus IDE
> VS Code is an editor that becomes IDE-like through extensions. Full IDEs such as PyCharm exist and are also good. The ideas in this note (interpreter selection, debugging, integrated terminal) apply to all of them.

## Core concepts

### Why an editor matters

| Feature | What it does for you |
| --- | --- |
| Syntax highlighting | Colors keywords, strings, and comments so mistakes stand out |
| IntelliSense (autocomplete) | Suggests names and shows documentation as you type |
| Linting | Underlines likely errors before you run the code |
| Formatting | Fixes spacing and layout automatically |
| Integrated terminal | A terminal inside the editor, opened in your project folder |
| Debugger | Pauses a running program so you can inspect variables |

### Installing

1. Download VS Code from code.visualstudio.com and install it.
2. Open it and go to the Extensions panel (the square icon on the left, or press Ctrl and Shift and X, or Command and Shift and X on macOS).
3. Install the extension named **Python** published by Microsoft. It also brings **Pylance** (language intelligence) and the debugger.
4. Optional but recommended for later chapters: install **Ruff** (published by Astral) for linting and formatting.

### Open a folder, not a single file

Use **File, then Open Folder** and choose your project folder. VS Code treats that folder as a **workspace**: it finds your virtual environment, stores project settings in a `.vscode` subfolder, and opens the terminal in the right place. Opening a lone file works but loses these benefits.

You can also open a folder from a terminal:

```bash
cd my-project
code .
```

On macOS, if `code` is not found, open the Command Palette and run **Shell Command: Install 'code' command in PATH**.

### The Command Palette

The Command Palette is the search box for every command in VS Code.

- Windows and Linux: Ctrl and Shift and P
- macOS: Command and Shift and P

Most "how do I do X" questions in VS Code are answered by opening the palette and typing a few words.

### Selecting the interpreter

The editor must know which Python to use, because it affects autocomplete, running, and debugging. Look at the bottom-right of the window for the Python version, or run **Python: Select Interpreter** from the Command Palette and choose the one in your project's `.venv` folder. When you create a virtual environment inside the project (Chapter 0.5), VS Code usually detects it and offers it in the list.

### The integrated terminal

Open it with the terminal toggle shortcut (Ctrl and the backtick key) or from the Terminal menu. When the selected interpreter is a virtual environment, VS Code activates it for you in new terminals, so `python` points to the right place. Look for `(.venv)` at the start of the prompt.

### Running code

- Click the play button at the top right of an open `.py` file, or
- Right-click in the file and choose **Run Python File in Terminal**, or
- Type `python filename.py` in the integrated terminal yourself.

The third way is worth practicing because it is exactly what runs on servers and in automation.

### Debugging concepts

A **breakpoint** is a marker on a line that tells the debugger to pause there. When the program pauses you can inspect every variable, then move forward one step at a time.

| Action | Key | Meaning |
| --- | --- | --- |
| Start or continue | F5 | Run until the next breakpoint |
| Toggle breakpoint | F9 | Add or remove a breakpoint on the current line |
| Step over | F10 | Run the current line, do not enter functions |
| Step into | F11 | Enter the function called on this line |
| Step out | Shift and F11 | Finish the current function and return to the caller |
| Stop | Shift and F5 | End the debugging session |

On some laptops you must hold the Fn key to use function keys.

## Basic syntax examples

### A first debugging session

Create a folder, open it in VS Code, and create `average.py`:

```python
def average(numbers):
    total = 0
    for n in numbers:
        total = total + n
    return total / len(numbers)


scores = [70, 85, 90]
result = average(scores)
print("Average score:", result)
```

Follow these steps:

1. Click in the gutter just left of the line number of `total = total + n`. A red dot appears, which is your breakpoint.
2. Press F5 and choose **Python File** if asked.
3. The program pauses on that line. Look at the **Variables** panel on the left. You will see `n`, `total`, and `numbers`.
4. Press F10 several times. Watch `total` change from `0` to `70`, then `155`, then `245`.
5. Press F5 to let the program finish, and read the output in the Debug Console or terminal.

### Debugging an error on purpose

Change the call to `average([])` and run again. The program stops with `ZeroDivisionError`. In debug mode, VS Code pauses at the failing line and shows you the values, which makes the cause obvious: `len(numbers)` is `0`.

### Conditional breakpoints

Right-click a breakpoint and choose **Edit Breakpoint**. You can enter a condition such as `n > 80`, and the debugger will only pause when it is true. This is invaluable inside loops that run thousands of times.

### Editor shortcuts worth learning

| Task | Windows and Linux | macOS |
| --- | --- | --- |
| Command Palette | Ctrl and Shift and P | Command and Shift and P |
| Open file by name | Ctrl and P | Command and P |
| Format document | Shift and Alt and F | Shift and Option and F |
| Comment or uncomment line | Ctrl and / | Command and / |
| Search in all files | Ctrl and Shift and F | Command and Shift and F |
| Rename a symbol everywhere | F2 | F2 |

## Deeper behavior

### The interpreter choice is saved per workspace

VS Code remembers your interpreter choice for each project folder. If code that works in the terminal fails in the editor, or the editor underlines an installed library as "not found," the cause is almost always that the editor is using a different interpreter than your terminal. Check the version and path in the status bar.

### Linting is not running

Underlines in the editor come from a language server (Pylance) or linter (Ruff) that reads your code without executing it. Running code and checking code are separate activities. This is why you can see an error underline for code that has never been run.

### What the `.vscode` folder holds

Project-level settings live in `.vscode/settings.json`, debug configurations in `.vscode/launch.json`, and recommended extensions in `.vscode/extensions.json`. These files let a team share the same setup. User-level settings (your theme, font size) live elsewhere and are not shared.

### The debugger launches your code in a special way

When you press F5, VS Code starts your program under a debug adapter (`debugpy`). The program behaves the same, but the adapter can pause it and inspect memory. Debug configurations let you pass arguments, set environment variables, and choose a module instead of a file.

### Just My Code

By default the debugger skips stepping into library code, so pressing F11 does not dive into Python's internals. This option is called `justMyCode`. Turn it off only when you need to investigate a library.

### Terminals do not share state

Each integrated terminal is its own session. A virtual environment activated in one terminal is not automatically active in another that was opened earlier. New terminals follow the selected interpreter.

## Industry-level snippet

Professional teams commit a small set of editor files so a new developer opens the project and everything works. The example below describes a backend service called `orders`. First, the code to debug:

`src/orders/main.py`

```python
"""Entry point for the orders service."""

import logging
import os
import sys
from decimal import Decimal, InvalidOperation

logger = logging.getLogger(__name__)

DEFAULT_TAX_RATE = "0.08"


def calculate_total(prices: list[Decimal], tax_rate: Decimal) -> Decimal:
    """Return the order total including tax, rounded to two decimal places."""
    if tax_rate < 0:
        raise ValueError("tax_rate must not be negative")
    subtotal = sum(prices, start=Decimal("0"))
    return (subtotal * (1 + tax_rate)).quantize(Decimal("0.01"))


def main() -> int:
    """Read configuration, compute a sample order, and return an exit code."""
    logging.basicConfig(
        level=os.environ.get("LOG_LEVEL", "INFO").upper(),
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    )
    raw_rate = os.environ.get("TAX_RATE", DEFAULT_TAX_RATE)
    try:
        tax_rate = Decimal(raw_rate)
    except InvalidOperation:
        logger.error("TAX_RATE must be a number, got %r", raw_rate)
        return 1

    try:
        total = calculate_total([Decimal("19.99"), Decimal("5.01")], tax_rate)
    except ValueError as exc:
        logger.error("Cannot calculate the order total: %s", exc)
        return 1

    logger.info("Order total with tax: %s", total)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

`Decimal` is used for money because ordinary decimal numbers in Python (floats) cannot represent values like 0.1 exactly. Chapter 1.3 explains why.

`.vscode/settings.json`

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.terminal.activateEnvironment": true,
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"],
  "editor.rulers": [88],
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

On Windows the interpreter path is `${workspaceFolder}\\.venv\\Scripts\\python.exe`. If your team mixes operating systems, omit `python.defaultInterpreterPath` and let each developer select the interpreter once, since VS Code detects `.venv` anyway.

`.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug current file",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "justMyCode": true
    },
    {
      "name": "Run orders service",
      "type": "debugpy",
      "request": "launch",
      "module": "orders.main",
      "cwd": "${workspaceFolder}",
      "env": {
        "PYTHONPATH": "${workspaceFolder}/src",
        "LOG_LEVEL": "DEBUG"
      },
      "envFile": "${workspaceFolder}/.env",
      "console": "integratedTerminal",
      "justMyCode": true
    }
  ]
}
```

`.vscode/extensions.json`

```json
{
  "recommendations": ["ms-python.python", "charliermarsh.ruff"]
}
```

How the pieces fit together:

- `module` runs `python -m orders.main`, matching how the service will be started in production.
- `envFile` loads variables from `.env` so secrets never appear in `launch.json`.
- `PYTHONPATH` lets Python find the package inside `src/`. Chapter 0.6 explains the `src/` layout.
- Format-on-save with Ruff keeps every developer's code identical without arguments about spacing.

> [!warning] Do not commit secrets
> `.vscode/` may be committed, but `.env` must not be. Only reference the file by path, as `launch.json` does.

## Common mistakes

1. **Opening a single file instead of a folder.** You lose the interpreter, terminal, and settings tied to the workspace.
2. **Wrong interpreter selected.** Symptoms are "module not found" in the editor while the terminal works, or the other way around.
3. **Installing the Python extension but never installing Python.** The extension needs an interpreter on your machine.
4. **Running code only with the play button forever.** Learn the terminal command too, because servers and CI use it.
5. **Debugging by adding `print` everywhere.** Breakpoints show every variable without editing code.
6. **Setting a breakpoint on a blank line or comment.** The debugger will not pause there. Use a line with code.
7. **Editing settings in the wrong scope.** User settings apply to every project, workspace settings apply to one. Team rules belong in `.vscode/settings.json`.
8. **Ignoring red and yellow underlines.** They are often real bugs, and reading them is a skill (Chapter 1.7).
9. **Committing `.env` while committing `.vscode/`.** Check `git status` before every commit.

## Practice exercises

1. Install VS Code and the Python extension. Open a new empty folder using **Open Folder**.
2. Create `hello.py` and run it three ways: the play button, the right-click menu, and the integrated terminal.
3. Open the Command Palette and select the Python interpreter. State which version and path you chose.
4. Open the integrated terminal and find out whether it shows an activated environment. Explain what you see.
5. Recreate `average.py` from this note. Set a breakpoint inside the loop and record the value of `total` after each iteration.
6. Add a conditional breakpoint that only pauses when `total` is greater than 100.
7. Cause a `ZeroDivisionError` deliberately and use the debugger to confirm which variable caused it.
8. Create a `.vscode/launch.json` that runs your file with `LOG_LEVEL` set to `DEBUG` through an environment variable, and print that variable from your script.
9. Add `.vscode/settings.json` with a ruler at column 88 and trailing whitespace trimming. Verify both work.
10. Run the `orders` service from the industry-level snippet in the debugger. Change the `TAX_RATE` value in a `.env` file to `abc` and observe the error handling. Then change it to `-0.1`.

## Summary

- Use VS Code with the Microsoft Python extension. Add Ruff when you reach the quality tools chapter.
- Open a project as a folder so VS Code can use workspace settings and your `.venv`.
- The Command Palette (Ctrl or Command and Shift and P) finds every command.
- Select the interpreter explicitly. Most "it works in the terminal but not in the editor" problems are interpreter mismatches.
- The integrated terminal opens in your project and activates the virtual environment.
- Debugging means breakpoints plus F5, F10, F11, and the Variables panel, and is far better than scattering `print` calls.
- Shared files in `.vscode/` give a whole team the same setup, but secrets never go into them.
