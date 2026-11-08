## Overview

Before writing any Python, you need a mental picture of what happens when a program runs and a place to run it from. That place is the terminal. Almost every professional Python workflow (installing packages, running scripts, using Git, deploying to a server) happens there, so this chapter is the foundation for everything else.

By the end of this note you will be able to:

- Explain what a program is and how a computer runs it.
- Explain the difference between a compiler and an interpreter.
- Tell the difference between a terminal and a shell.
- Move around your file system, create folders, and remove them using commands.
- Read and write absolute and relative paths without confusion.

> [!note] If you are brand new
> The terminal looks scary because it has no buttons. It is simply a text conversation: you type a command, the computer answers. Nothing you type in the first half of this note can damage your computer, as long as you follow the warnings about deleting.

## Core concepts

### What a program is

A **program** is a list of instructions stored in a file. A **process** is that program while it is actually running in memory. You can have one program and several processes of it at once, for example three browser windows each running the same browser program.

Your processor (the CPU) only understands **machine code**, which is long sequences of numbers. Humans write **source code** in a language like Python, and something has to translate it into machine instructions.

### Compiler versus interpreter

| Idea | Compiler | Interpreter |
| --- | --- | --- |
| When translation happens | Once, before the program runs | While the program runs |
| What you get | A separate executable file | Nothing extra, the interpreter reads your source |
| Typical languages | C, C++, Go, Rust | Python, Ruby, JavaScript |
| Speed | Usually faster at run time | Usually slower at run time |
| Feedback while learning | Must build before each test | Run immediately, see errors immediately |

Python is called an interpreted language. The truth is slightly more subtle (see Deeper behavior), but for now think of it as: you give Python a file, and the Python interpreter reads and carries out your instructions.

### Terminal, shell, and prompt

These three words are often mixed up.

| Term | Meaning |
| --- | --- |
| Terminal (terminal emulator) | The window that displays text and accepts your typing. Examples: Windows Terminal, macOS Terminal, GNOME Terminal |
| Shell | The program inside the window that reads your command and runs it. Examples: PowerShell, zsh, bash |
| Prompt | The text the shell prints to say it is ready, such as `PS C:\Users\learner>` or `learner@laptop ~ %` |

Different operating systems use different default shells:

- Windows: PowerShell
- macOS: zsh
- Linux: usually bash

This roadmap gives commands for all three. zsh and bash behave almost identically for everything in Phase 0, so one set of commands covers both.

### How to open a terminal

- Windows: press the Windows key, type `Terminal` or `PowerShell`, press Enter.
- macOS: press Command and Space, type `Terminal`, press Enter.
- Linux: press Ctrl, Alt, and T on most distributions, or search for Terminal in the applications menu.

### Files, folders, and paths

Your files live in a tree of folders. A **path** is the address of a file or folder in that tree.

- An **absolute path** starts from the very top of the tree and works from anywhere. Example: `C:\Users\learner\projects` on Windows, `/home/learner/projects` on Linux, `/Users/learner/projects` on macOS.
- A **relative path** starts from wherever you currently are (your **working directory**). Example: `projects\notes` means "the folder called notes inside the folder called projects that is inside where I am now."

Special shortcuts that work on all three systems:

| Symbol | Meaning |
| --- | --- |
| `.` | The current folder |
| `..` | The parent folder (one level up) |
| `~` | Your home folder |

Separators differ: Windows uses a backslash `\`, while macOS and Linux use a forward slash `/`. PowerShell accepts both.

### Essential commands

| Task | PowerShell (Windows) | zsh or bash (macOS, Linux) |
| --- | --- | --- |
| Print working directory | `pwd` | `pwd` |
| List files | `ls` | `ls` or `ls -la` (include hidden files) |
| Change directory | `cd folder` | `cd folder` |
| Go up one level | `cd ..` | `cd ..` |
| Go to home folder | `cd ~` | `cd ~` or just `cd` |
| Make a folder | `mkdir name` | `mkdir name` |
| Make nested folders | `mkdir a\b\c` | `mkdir -p a/b/c` |
| Create an empty file | `New-Item notes.txt` | `touch notes.txt` |
| Show file contents | `Get-Content notes.txt` or `cat notes.txt` | `cat notes.txt` |
| Copy a file | `Copy-Item a.txt b.txt` or `cp a.txt b.txt` | `cp a.txt b.txt` |
| Move or rename | `Move-Item a.txt b.txt` or `mv a.txt b.txt` | `mv a.txt b.txt` |
| Remove a file | `Remove-Item a.txt` | `rm a.txt` |
| Remove an empty folder | `rmdir name` | `rmdir name` |
| Remove a folder and everything in it | `Remove-Item -Recurse name` | `rm -r name` |
| Clear the screen | `cls` or `clear` | `clear` |

> [!warning] Deleting is permanent
> Commands like `rm` and `Remove-Item` do not use the recycle bin or trash. The file is gone. Always run `pwd` and `ls` first to confirm where you are and what is there.

## Basic syntax examples

### Windows (PowerShell)

```powershell
pwd
cd ~
mkdir python-roadmap
cd python-roadmap
mkdir notes
New-Item hello.txt
ls
cd notes
pwd
cd ..
cd ..
Remove-Item -Recurse python-roadmap
```

What each line does, in order: show where you are, jump to your home folder, create a folder, enter it, create a subfolder, create an empty file, list the contents, enter the subfolder, confirm the new location, go up twice, then delete the practice folder and everything inside it.

### macOS and Linux (zsh or bash)

```bash
pwd
cd ~
mkdir python-roadmap
cd python-roadmap
mkdir notes
touch hello.txt
ls -la
cd notes
pwd
cd ../..
rm -r python-roadmap
```

Example of what `pwd` prints after `cd notes`:

```text
/home/learner/python-roadmap/notes
```

### Absolute versus relative, side by side

Assume you are in your home folder and want to enter `python-roadmap/notes`.

```bash
# Relative path: starts from where you are now
cd python-roadmap/notes

# Absolute path: works no matter where you are
cd /home/learner/python-roadmap/notes
```

### Quoting paths that contain spaces

```bash
cd "My Documents/python notes"
```

Without the quotes, the shell sees three separate words and fails.

### Tips that save hours

- Press Tab to auto-complete file and folder names.
- Press the Up arrow to bring back previous commands.
- Press Ctrl and C to stop a command that is stuck or running too long.
- Paste commands from this note without the leading `$` or `PS>` prompt symbols if you ever see them in other tutorials. Those are the prompt, not part of the command.

## Deeper behavior

### Python is compiled and interpreted

When you run a `.py` file, Python first compiles your source into **bytecode**, a compact set of instructions for Python's own virtual machine. Then the interpreter (the Python virtual machine) executes that bytecode. You may notice a `__pycache__` folder appear next to your code. It stores the compiled bytecode so later runs can start faster. Phase 1 returns to this in detail.

### The working directory matters

Every process has a **current working directory**. When your Python script opens a file called `data.csv`, it looks relative to the working directory of the terminal where you launched it, not relative to where the script file is saved. A large share of "file not found" errors in beginner projects come from this single fact.

### Where does the shell find commands?

When you type `python`, the shell does not search your whole disk. It looks through a list of folders stored in an environment variable called `PATH`, in order, until it finds a program with that name. Installing Python correctly (next chapter) is largely about making sure Python is on this list.

### Exit codes

Every program, when it finishes, hands back a number called an **exit code** (or exit status). By convention, `0` means success and any other number means some kind of failure. Scripts, build tools, and deployment pipelines rely on this to decide whether to continue.

```bash
ls /this/folder/does/not/exist
echo $?
```

```powershell
Get-Item C:\this\folder\does\not\exist
$LASTEXITCODE
$?
```

In bash and zsh, `$?` holds the last exit code. In PowerShell, `$LASTEXITCODE` holds the exit code of the last external program and `$?` is True or False for the last command.

### Standard streams, redirection, and pipes

Every process has three channels: standard input (`stdin`), standard output (`stdout`), and standard error (`stderr`). The shell can redirect them.

```bash
ls > files.txt          # send output into a file (overwrite)
ls >> files.txt         # append instead of overwrite
ls | sort               # send output of ls into sort
```

### Case sensitivity

Linux treats `Notes.txt` and `notes.txt` as different files. Windows and (by default) macOS treat them as the same. Code written on a case-insensitive system can break on a Linux server, so always match capitalization exactly.

## Industry-level snippet

Backend engineers regularly write small diagnostic scripts to answer "where is this program running and in what conditions?" This one reports on the environment, validates that a directory is writable, and uses exit codes properly. You do not need to understand every line yet, read the comments and run it to see the output.

```python
"""Print a diagnostic report about the environment a script is running in."""

import logging
import os
import platform
import sys
from dataclasses import dataclass
from pathlib import Path

logger = logging.getLogger("env_report")

EXIT_OK = 0
EXIT_PROBLEM = 1


@dataclass(frozen=True)
class EnvironmentReport:
    """A snapshot of the environment the interpreter is running in."""

    os_name: str
    python_version: str
    interpreter_path: str
    working_directory: Path
    home_directory: Path
    path_entries: list[str]


def build_report() -> EnvironmentReport:
    """Collect facts about the current environment."""
    raw_path = os.environ.get("PATH", "")
    return EnvironmentReport(
        os_name=f"{platform.system()} {platform.release()}",
        python_version=platform.python_version(),
        interpreter_path=sys.executable,
        working_directory=Path.cwd(),
        home_directory=Path.home(),
        path_entries=[entry for entry in raw_path.split(os.pathsep) if entry],
    )


def is_writable(directory: Path) -> bool:
    """Return True if the current user can create files inside directory."""
    return directory.is_dir() and os.access(directory, os.W_OK)


def main() -> int:
    """Print the report and return a process exit code."""
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    )
    try:
        report = build_report()
    except OSError as exc:
        logger.error("Could not read the environment: %s", exc)
        return EXIT_PROBLEM

    logger.info("Operating system : %s", report.os_name)
    logger.info("Python version   : %s", report.python_version)
    logger.info("Interpreter      : %s", report.interpreter_path)
    logger.info("Working directory: %s", report.working_directory)
    logger.info("PATH has %d folders", len(report.path_entries))

    if not is_writable(report.working_directory):
        logger.error("Working directory is not writable")
        return EXIT_PROBLEM
    return EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Save it as `env_report.py`, then run it from the terminal with `python env_report.py` (Windows) or `python3 env_report.py` (macOS and Linux). Afterwards check its exit code with `echo $?` or `$LASTEXITCODE`.

What to notice:

- `Path` objects represent file paths in a way that works on every operating system.
- `os.pathsep` is `;` on Windows and `:` on macOS and Linux, which is why the code does not hard-code either one.
- `main()` returns a number instead of calling `exit` in the middle, and `sys.exit(main())` hands that number to the shell as the exit code.

## Common mistakes

1. **Running a command in the wrong folder.** Always check with `pwd` before creating, running, or deleting anything.
2. **Forgetting quotes around paths with spaces.** `cd My Documents` fails, `cd "My Documents"` works.
3. **Copying the prompt symbol.** Pasting `$ ls` or `PS C:\> ls` includes characters that are not part of the command.
4. **Using the wrong slash style in bash or zsh.** Backslashes mean something else there. Use `/` on macOS and Linux.
5. **Assuming a script's folder is the working directory.** They are often different.
6. **Running destructive commands without looking.** `rm -r` and `Remove-Item -Recurse` skip the trash. Never run one you do not fully understand.
7. **Confusing terminal and shell.** If a tutorial says "open PowerShell" and you are on a Mac, translate to "open Terminal and use zsh."
8. **Closing the terminal to stop a program.** Press Ctrl and C first. Closing the window can leave processes behind.

## Practice exercises

1. Open your terminal. Write down which shell you are using and how you found out.
2. Print your working directory. Then navigate to your home folder using a command that works from any location.
3. List all files in your home folder, including hidden ones. Identify two hidden items and guess what they might be for.
4. Create a folder named `python-roadmap` containing three subfolders named `phase-0`, `phase-1`, and `phase-2`, using as few commands as you can.
5. Inside `phase-0`, create an empty file named `notes.txt`. Then rename it to `terminal-notes.txt` without opening any editor.
6. From inside `phase-2`, write a single relative path that refers to `terminal-notes.txt` in `phase-0`. Then write the absolute path to the same file.
7. Delete `phase-2` using one command, and confirm it is gone.
8. Run a command that fails on purpose (for example, listing a folder that does not exist). Read its exit code and explain what the number means.
9. Run `env_report.py` from two different folders. Explain exactly which line of the output changed and why.
10. Modify the industry-level snippet so it also reports the number of files in the working directory. Handle the case where the directory cannot be read.

## Summary

- A program is stored instructions, a process is a running program, and the CPU only understands machine code.
- A compiler translates before running, an interpreter translates while running. Python compiles to bytecode and then interprets it.
- The terminal is the window, the shell is the program that understands your commands, and the prompt shows it is ready.
- The commands you need daily are `pwd`, `ls`, `cd`, `mkdir`, `rm` or `Remove-Item`, `cat` or `Get-Content`, `cp`, and `mv`.
- Absolute paths start from the top of the file system, relative paths start from your working directory. `.`, `..`, and `~` are shortcuts.
- The working directory, `PATH`, and exit codes are three hidden concepts that explain many beginner errors.
- Deleting from the terminal is permanent, so check your location before every destructive command.
