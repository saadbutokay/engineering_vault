## Overview

**Git** is a version control system. It records the history of your project as a series of snapshots, so you can see what changed, when, and why, undo mistakes, and work on new ideas without risking working code. **GitHub** is a website that hosts Git repositories so you can back them up and collaborate. Git and GitHub are different things: Git is the tool on your machine, GitHub is one of several services built around it.

By the end of this note you will be able to:

- Create a repository and record your work with commits.
- Read the state of your project with `status`, `log`, and `diff`.
- Work on ideas safely in branches and merge them back.
- Connect a local repository to GitHub and push your work.
- Decide what belongs in a repository and what should be ignored.

> [!note] Install check
> Run `git --version` in your terminal. If it is missing, install it from git-scm.com, with Homebrew (`brew install git`), or your Linux package manager (`sudo apt install git` or `sudo dnf install git`). On macOS, typing `git` may offer to install developer tools.

## Core concepts

### Repository, commit, and history

- A **repository** (repo) is a project folder that Git tracks. Its history is stored in a hidden `.git` folder.
- A **commit** is a saved snapshot of your tracked files, with a message, an author, a timestamp, and a unique ID called a **hash**.
- The **history** is the chain of commits, each pointing to the one before it.

### The three areas

Git has an extra step compared with a normal "save" button. You choose exactly which changes go into the next snapshot.

```text
Working directory  --git add-->  Staging area  --git commit-->  Repository
(your files)                      (next snapshot)                (permanent history)
```

| Area | What it holds |
| --- | --- |
| Working directory | Your actual files, including edits you have not recorded |
| Staging area (index) | Changes you have marked to include in the next commit |
| Repository | All commits made so far |

### One-time setup

Tell Git who you are. This information is attached to every commit.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

### The core commands

| Command | What it does |
| --- | --- |
| `git init` | Start tracking the current folder |
| `git status` | Show what changed, what is staged, and what is untracked |
| `git add file` | Stage one file |
| `git add .` | Stage all changes in the current folder |
| `git commit -m "message"` | Create a commit from what is staged |
| `git log` | Show commit history |
| `git log --oneline` | Show history compactly |
| `git diff` | Show unstaged changes line by line |
| `git diff --staged` | Show staged changes |
| `git restore file` | Discard unstaged edits to a file |
| `git restore --staged file` | Unstage a file but keep the edits |

### Branches

A **branch** is a movable label pointing at a commit. Creating a branch is nearly free. It lets you experiment or build a feature without touching the main line of work.

| Command | What it does |
| --- | --- |
| `git branch` | List branches |
| `git switch -c feature-name` | Create a branch and switch to it |
| `git switch main` | Switch back to main |
| `git merge feature-name` | Bring a branch's commits into the current branch |
| `git branch -d feature-name` | Delete a merged branch |

### Remotes and GitHub

A **remote** is another copy of the repository, usually on a server. By convention the main remote is called `origin`.

| Command | What it does |
| --- | --- |
| `git clone URL` | Copy a remote repository to your machine |
| `git remote add origin URL` | Connect your local repository to a remote |
| `git remote -v` | Show configured remotes |
| `git push -u origin main` | Upload your branch and remember the connection |
| `git push` | Upload new commits |
| `git pull` | Download and integrate new commits from the remote |

### Authentication to GitHub

GitHub does not accept your account password for Git operations. Use one of these:

- **Git Credential Manager**, included with Git for Windows and available for macOS and Linux, which opens a browser to sign in.
- **GitHub CLI**: run `gh auth login`.
- **SSH keys**: generate a key, add the public part to your GitHub account, and use a `git@github.com:USER/REPO.git` address.

### What to commit and what to ignore

| Commit | Ignore |
| --- | --- |
| Source code in `src/` | `.venv/` virtual environments |
| Tests in `tests/` | `__pycache__/` and `*.pyc` |
| `pyproject.toml` | `.env` and any file with secrets |
| `requirements.txt` or lock files | `.mypy_cache/`, `.ruff_cache/`, `.pytest_cache/` |
| `README.md` | `dist/`, `build/`, `*.egg-info/` |
| `.gitignore` | Editor folders such as `.idea/` and OS files such as `.DS_Store` |
| `.env.example` | Large generated data files and logs |

Rule of thumb: commit what a teammate needs to rebuild your project, ignore what can be regenerated or is private.

### Writing good commit messages

- Start with a short summary line, around 50 characters, in the imperative mood: "Add login endpoint", not "Added login endpoint".
- Make each commit one logical change.
- If more explanation is needed, leave a blank line and add a paragraph about why the change was made.

## Basic syntax examples

### Your first repository

```bash
mkdir git-practice
cd git-practice
git init
echo "# Git practice" > README.md
git status
git add README.md
git commit -m "Add README"
git log --oneline
```

```text
Initialized empty Git repository in /home/learner/git-practice/.git/
On branch main
No commits yet
Untracked files:
  README.md
...
[main (root-commit) 3f2a9c1] Add README
 1 file changed, 1 insertion(+)
3f2a9c1 Add README
```

(The exact wording of Git's output varies a little by version.)

### Seeing changes before committing

Edit `README.md` to add a line, then:

```bash
git status
git diff
git add README.md
git diff --staged
git commit -m "Describe the project"
```

### Undoing small mistakes

```bash
git restore README.md              # throw away unstaged edits
git restore --staged README.md     # unstage but keep edits
```

> [!warning] `git restore file` is permanent
> Discarding unstaged edits cannot be undone, because Git never recorded them.

### A feature branch workflow

```bash
git switch -c add-greeting
echo "print('hello')" > hello.py
git add hello.py
git commit -m "Add greeting script"
git switch main
git merge add-greeting
git branch -d add-greeting
git log --oneline
```

### Merge conflicts

If two branches change the same lines, Git cannot decide for you and marks the file:

```text
<<<<<<< HEAD
print("hello from main")
=======
print("hello from the branch")
>>>>>>> add-greeting
```

To resolve it: edit the file to keep what you want and delete the three marker lines, then run `git add` on the file and `git commit`.

### Connecting to GitHub

1. On GitHub, create a new **empty** repository (do not add a README or license there, or the histories will differ).
2. Copy its address, then in your local project:

```bash
git remote add origin https://github.com/YOUR-USER/git-practice.git
git push -u origin main
```

Later pushes are just `git push`.

### Starting from an existing repository

```bash
git clone https://github.com/YOUR-USER/git-practice.git
cd git-practice
git pull
```

### Stopping Git from tracking a file you committed by mistake

```bash
git rm --cached .env
git commit -m "Stop tracking .env"
```

The file stays on your disk but leaves version control. Add it to `.gitignore` as well.

## Deeper behavior

### Commits are snapshots, not differences

Git stores the complete state of every tracked file in each commit, and it is smart about sharing unchanged content. Each commit has a hash computed from its contents and its parent. This is why history is tamper-evident: changing an old commit would change every hash after it.

### Branches are just pointers

A branch is a small file containing a commit hash. `HEAD` is a pointer to the branch (or commit) you are currently on. When you commit, the current branch's pointer moves forward. This is why branching and switching are fast.

### Fast-forward and merge commits

If `main` has not changed since you created your branch, merging simply moves the `main` pointer forward (**fast-forward**). If both sides have new commits, Git creates a **merge commit** with two parents.

### Ignoring does not untrack

`.gitignore` only affects untracked files. If a file was committed before you added it to `.gitignore`, Git keeps tracking it until you run `git rm --cached`.

### History is permanent, so secrets are too

If you commit a password and then delete it in the next commit, the password remains visible in history. Rotate the secret. Rewriting history is possible but difficult once the repository is shared.

### Never rewrite shared history

Commands that rewrite commits (such as `git reset --hard` after sharing, `git push --force`, or rebasing a public branch) can destroy teammates' work. Learn them later and avoid them on shared branches.

### The reflog: a safety net

Even after "losing" commits, Git usually keeps references in the **reflog** for a while. `git reflog` lists where `HEAD` has been, which often lets you recover.

### Line endings and operating systems

Windows ends lines with two characters, macOS and Linux with one. Git can normalize this. On Windows the installer's default setting handles it, and a `.gitattributes` file can enforce a team rule.

### Pull requests are a GitHub feature

A **pull request** asks teammates to review and merge your branch. The usual team workflow is: create a branch, commit, push it, open a pull request, get review, merge, delete the branch. Phase 13 covers collaboration workflows.

## Industry-level snippet

Release and deployment scripts commonly refuse to build from a dirty or wrong branch, and they stamp the build with the commit hash. This script uses Git through `subprocess`, reports problems clearly, and returns exit codes for automation.

```python
"""Pre-release guard: refuse to build unless the git state is clean.

Usage:
    python release_check.py

Exit codes:
    0  repository is on the release branch and has no uncommitted changes
    1  the repository state is not acceptable for a release
    2  git could not be run or this is not a repository
"""

import logging
import subprocess
import sys
from dataclasses import dataclass

logger = logging.getLogger("release_check")

RELEASE_BRANCH = "main"
GIT_TIMEOUT_SECONDS = 30

EXIT_OK = 0
EXIT_BAD_STATE = 1
EXIT_GIT_FAILURE = 2


class GitError(Exception):
    """Raised when a git command cannot be run or fails."""


@dataclass(frozen=True)
class RepoState:
    """The facts about a repository that matter for a release."""

    branch: str
    commit: str
    is_clean: bool


def run_git(*args: str) -> str:
    """Run a git command and return its standard output without trailing spaces."""
    command = ["git", *args]
    try:
        result = subprocess.run(
            command,
            check=True,
            capture_output=True,
            text=True,
            timeout=GIT_TIMEOUT_SECONDS,
        )
    except FileNotFoundError as exc:
        raise GitError("git is not installed or is not on PATH") from exc
    except subprocess.TimeoutExpired as exc:
        raise GitError(f"git timed out after {GIT_TIMEOUT_SECONDS} seconds") from exc
    except subprocess.CalledProcessError as exc:
        details = exc.stderr.strip() or "no error message"
        raise GitError(f"{' '.join(command)} failed: {details}") from exc
    return result.stdout.strip()


def inspect_repository() -> RepoState:
    """Read the current branch, short commit hash, and cleanliness."""
    branch = run_git("rev-parse", "--abbrev-ref", "HEAD")
    commit = run_git("rev-parse", "--short", "HEAD")
    is_clean = run_git("status", "--porcelain") == ""
    return RepoState(branch=branch, commit=commit, is_clean=is_clean)


def find_problems(state: RepoState) -> list[str]:
    """Return a list of reasons the repository is not ready to release."""
    problems: list[str] = []
    if state.branch != RELEASE_BRANCH:
        problems.append(
            f"on branch {state.branch!r}, releases must come from {RELEASE_BRANCH!r}"
        )
    if not state.is_clean:
        problems.append("there are uncommitted changes in the working directory")
    return problems


def main() -> int:
    """Run the release check and return a process exit code."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    try:
        state = inspect_repository()
    except GitError as exc:
        logger.error("%s", exc)
        return EXIT_GIT_FAILURE

    problems = find_problems(state)
    if problems:
        for problem in problems:
            logger.error("Release blocked: %s", problem)
        return EXIT_BAD_STATE

    logger.info("Release check passed at commit %s", state.commit)
    return EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Why it is written this way:

- `git status --porcelain` produces stable, script-friendly output. It prints nothing when the working directory is clean.
- Passing the command as a list (`["git", ...]`) rather than one string avoids shell-quoting problems and injection risks.
- Every way the command can fail (missing git, a timeout, a non-zero exit) becomes one clear `GitError`.
- `find_problems` is a pure function with no side effects, so it is easy to test.
- A real pipeline would also confirm the branch is up to date with the remote and embed `state.commit` in the build metadata.

## Common mistakes

1. **Running `git init` inside another repository or in your home folder.** Check with `pwd` first.
2. **Committing everything with `git add .` without reading `git status`.** You may add secrets, caches, or large files.
3. **Writing vague messages such as "fix" or "update".** Future you will not know what changed.
4. **Committing `.env`, `.venv`, or `__pycache__`.** Create `.gitignore` before the first commit.
5. **Assuming `.gitignore` removes already-tracked files.** Use `git rm --cached`.
6. **Working directly on `main` for everything.** Use branches for features and fixes.
7. **Creating a GitHub repository with a README, then pushing an existing local project.** The histories differ and the push is rejected. Start with an empty remote.
8. **Using `git push --force` to make an error disappear.** It can overwrite teammates' commits.
9. **Not committing often enough.** Small, focused commits are easier to understand and undo.
10. **Panicking at a merge conflict.** It is simply Git asking you to choose. Read the markers, edit, add, commit.
11. **Believing deleting a committed secret removes it.** It stays in history, so rotate it.

## Practice exercises

1. Install Git, set your name, email, and default branch, and show your configuration with `git config --list`.
2. Create a new folder, initialize a repository, and make three separate commits (a README, a Python file, and a `.gitignore`). View the history in compact form.
3. Edit a file, then use `git diff` before staging and `git diff --staged` after staging. Describe the difference.
4. Make an edit, stage it, then unstage it without losing the edit. Then discard the edit.
5. Create a branch called `add-validation`, make two commits on it, switch back to `main`, and merge it. Delete the branch.
6. Create a merge conflict on purpose by changing the same line on two branches. Resolve it and complete the merge.
7. Add a `.env` file, commit it by mistake, then remove it from tracking without deleting it from disk. Explain what else you would need to do if it contained a real secret.
8. Create an empty repository on GitHub, connect your local project, and push. Then edit a file on GitHub's website and use `git pull` to bring the change down.
9. Write three commit messages for three different changes, following the conventions in this note. Explain why each is a good message.
10. Run the industry-level snippet in a clean repository on `main`, then again after modifying a file, then again from a different branch. Compare exit codes.
11. Extend the snippet to also fail the release if the current commit does not match the remote `origin/main`.

## Summary

- Git records project history as commits. GitHub hosts repositories but is a separate service.
- Changes move from working directory to staging area to repository through `git add` and `git commit`.
- Use `status`, `log --oneline`, and `diff` constantly to know where you stand.
- Branches are cheap pointers. Use them for features and fixes, then merge back.
- Connect to GitHub with `git remote add origin` and `git push -u origin main`. Authenticate with a credential manager, `gh`, or SSH, not your account password.
- Commit source, tests, configuration, and documentation. Ignore virtual environments, caches, build output, and secrets.
- History is permanent. Never commit secrets, and never rewrite history that others have.
