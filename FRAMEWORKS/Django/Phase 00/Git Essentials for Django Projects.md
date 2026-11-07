## Overview

Git records the history of your project so you can undo mistakes, work on features in isolation, and collaborate. Django projects have specific rules about what belongs in version control: migrations and lock files must be committed, while databases, secrets, and generated files must not. This note covers the everyday Git commands, a correct `.gitignore`, how to push to GitHub, and how to resolve the migration conflicts that appear when two branches both add migrations. It finishes with a tested script that checks a repository for files that should never be tracked.

## Core concepts

### The three areas

- Working directory: the files you edit.
- Staging area (index): the changes you have selected for the next commit.
- Repository: the permanent history of commits.

A commit is a snapshot with a message, an author, and a pointer to its parent.

### Branches and merging

A branch is a movable pointer to a commit. Create one for each feature or fix. Merging combines the history of two branches; when both changed the same lines, Git asks you to resolve a conflict.

### Remotes

A remote is a copy of the repository hosted elsewhere, usually on GitHub. `origin` is the conventional name for the main one. You `push` to send commits and `pull` or `fetch` to receive them.

### What to commit in a Django project

Commit:

- All source code and templates.
- Migration files for every app, because they are part of the schema history.
- `pyproject.toml` and the lock file (`uv.lock`).
- `.env.example` with safe placeholder values.
- Configuration such as `.gitignore`, `ruff` settings, and CI files.

Do not commit:

- `db.sqlite3` and any local database files.
- `.env` and any file containing secrets.
- `staticfiles/` (the output of `collectstatic`).
- `media/` (user uploads).
- `__pycache__/` and `*.pyc`.
- `.venv/` and editor folders that are personal to you.

### Why migrations are committed

Migrations are code that describes how to move the database from one state to another. Every developer and every deployment must apply the same ones in the same order. Never edit a migration that others have already applied; add a new one.

### Migration conflicts

If two branches each create `0003_*.py` from the same `0002`, merging them produces two leaf nodes in the migration graph, and `migrate` fails with a message about conflicting migrations.

## Basic syntax examples

### Initial setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main

git init
git status
```

### The everyday cycle

```bash
git status                 # what changed?
git diff                   # unstaged changes
git add apps/blog/views.py # stage one file
git add -p                 # stage interactively, hunk by hunk
git commit -m "Add post list view"
git log --oneline --graph --decorate -10
```

### Branching and merging

```bash
git switch -c feature/post-publishing
# ... edit, add, commit ...
git switch main
git merge feature/post-publishing
git branch -d feature/post-publishing
```

### Connecting to GitHub

```bash
git remote add origin git@github.com:your-user/mysite.git
git push -u origin main
git pull
```

### A .gitignore for Django

```text
# Python
__pycache__/
*.py[cod]
.venv/
.mypy_cache/
.ruff_cache/
.pytest_cache/
.coverage
htmlcov/

# Django
db.sqlite3
db.sqlite3-journal
staticfiles/
media/

# Secrets and local config
.env
.env.*
!.env.example

# Editors and OS
.vscode/
.idea/
.DS_Store
```

### Undoing things safely

```bash
git restore apps/blog/views.py        # discard unstaged edits to a file
git restore --staged apps/blog/views.py  # unstage a file
git revert <commit>                   # create a commit that undoes another
git stash                             # shelve changes temporarily
git stash pop                         # bring them back
```

## Deeper behavior

### Good commit messages

Write the subject in the imperative mood and keep it under about 70 characters ("Add post publishing service"). Add a body explaining why, not what, when the reason is not obvious. Small, focused commits are easier to review, revert, and bisect.

### Ignoring files that are already tracked

`.gitignore` only affects untracked files. If `db.sqlite3` was committed by mistake, adding it to `.gitignore` does nothing until you remove it from the index:

```bash
git rm --cached db.sqlite3
git commit -m "Stop tracking the local database"
```

A secret that was committed is still in history. Treat it as leaked, rotate it, and only then consider rewriting history.

### Resolving a migration conflict

Suppose `main` and your branch both added a migration after `0002_initial`. After merging, Django reports conflicting migrations. Check the situation:

```bash
uv run python manage.py showmigrations blog
uv run python manage.py makemigrations --check --dry-run
```

There are two clean fixes.

First option, when the migrations touch unrelated things: generate a merge migration.

```bash
uv run python manage.py makemigrations --merge
```

Second option, when your migration has not been shared yet: delete your branch's migration, merge `main`, and regenerate it so it depends on the latest one.

```bash
git rm apps/blog/migrations/0003_add_tag.py
git merge main
uv run python manage.py makemigrations blog
```

Never solve the conflict by deleting migrations that already ran on a shared or production database.

### Branching workflows

- Trunk-based: short-lived branches merged to `main` frequently. Works well with strong CI.
- Feature branches with pull requests: each change is reviewed before merging.

Protect `main` on GitHub so changes arrive only through reviewed pull requests whose CI checks pass.

### Line endings and Windows

Windows uses CRLF line endings and Linux and macOS use LF. Set `git config --global core.autocrlf input` on macOS and Linux, and `true` on Windows, or add a `.gitattributes` with `* text=auto` so files are normalized.

## Industry-level snippet

Mistakes such as committing a database or an `.env` file happen to everyone. A small script run in CI and in a pre-commit hook can catch them. It lists tracked files with Git, applies explicit rules, reports every violation, and returns a proper exit code.

`scripts/check_repo_hygiene.py`:

```python
"""Fail if files that must never be tracked are present in the repository."""

import logging
import subprocess
import sys
from collections.abc import Iterable, Sequence
from pathlib import Path, PurePosixPath

logger = logging.getLogger(__name__)

FORBIDDEN_FILE_NAMES = frozenset({".env", "db.sqlite3", "db.sqlite3-journal"})
FORBIDDEN_DIRECTORIES = frozenset({"staticfiles", "media", "__pycache__", ".venv"})
FORBIDDEN_SUFFIXES = frozenset({".pyc", ".pyo"})


class GitError(RuntimeError):
    """Git could not be run or reported a failure."""


def list_tracked_files(repo_root: Path) -> list[str]:
    """Return every path tracked by Git under ``repo_root``."""
    if not repo_root.is_dir():
        raise GitError(f"Not a directory: {repo_root}")
    try:
        result = subprocess.run(  # noqa: S603
            ["git", "-C", str(repo_root), "ls-files", "-z"],  # noqa: S607
            check=True,
            capture_output=True,
            text=True,
        )
    except (OSError, subprocess.CalledProcessError) as exc:
        raise GitError(f"Could not list tracked files: {exc}") from exc
    return [path for path in result.stdout.split("\0") if path]


def find_violations(paths: Iterable[str]) -> list[str]:
    """Return the paths that break the repository rules, in sorted order."""
    violations: set[str] = set()
    for raw in paths:
        path = PurePosixPath(raw)
        in_forbidden_directory = any(
            part in FORBIDDEN_DIRECTORIES for part in path.parts[:-1]
        )
        if (
            path.name in FORBIDDEN_FILE_NAMES
            or path.suffix in FORBIDDEN_SUFFIXES
            or in_forbidden_directory
        ):
            violations.add(raw)
    return sorted(violations)


def main(argv: Sequence[str] | None = None) -> int:
    args = list(sys.argv[1:] if argv is None else argv)
    repo_root = Path(args[0]) if args else Path.cwd()

    try:
        tracked = list_tracked_files(repo_root)
    except GitError as exc:
        logger.error("%s", exc)
        return 2

    violations = find_violations(tracked)
    if violations:
        for violation in violations:
            logger.error("Must not be tracked: %s", violation)
        return 1

    logger.info("Repository hygiene check passed (%d files)", len(tracked))
    return 0


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    raise SystemExit(main())
```

`scripts/test_check_repo_hygiene.py`:

```python
from pathlib import Path

import pytest

from check_repo_hygiene import GitError, find_violations, list_tracked_files, main


def test_clean_repository_has_no_violations() -> None:
    paths = ["manage.py", "apps/blog/models.py", ".env.example", "uv.lock"]
    assert find_violations(paths) == []


@pytest.mark.parametrize(
    "path",
    [
        ".env",
        "db.sqlite3",
        "config/.env",
        "staticfiles/admin/css/base.css",
        "media/uploads/photo.jpg",
        "apps/blog/__pycache__/views.cpython-312.pyc",
        "apps/blog/views.pyc",
    ],
)
def test_forbidden_paths_are_reported(path: str) -> None:
    assert find_violations([path]) == [path]


def test_violations_are_unique_and_sorted() -> None:
    result = find_violations(["media/b.jpg", ".env", "media/b.jpg"])
    assert result == [".env", "media/b.jpg"]


def test_list_tracked_files_rejects_missing_directory(tmp_path: Path) -> None:
    with pytest.raises(GitError):
        list_tracked_files(tmp_path / "missing")


def test_main_returns_one_when_violations_exist(
    monkeypatch: pytest.MonkeyPatch, tmp_path: Path
) -> None:
    monkeypatch.setattr(
        "check_repo_hygiene.list_tracked_files",
        lambda root: ["manage.py", ".env"],
    )
    assert main([str(tmp_path)]) == 1


def test_main_returns_zero_for_clean_repository(
    monkeypatch: pytest.MonkeyPatch, tmp_path: Path
) -> None:
    monkeypatch.setattr(
        "check_repo_hygiene.list_tracked_files",
        lambda root: ["manage.py"],
    )
    assert main([str(tmp_path)]) == 0


def test_main_returns_two_when_git_fails(
    monkeypatch: pytest.MonkeyPatch, tmp_path: Path
) -> None:
    def fail(root: Path) -> list[str]:
        raise GitError("git is not installed")

    monkeypatch.setattr("check_repo_hygiene.list_tracked_files", fail)
    assert main([str(tmp_path)]) == 2
```

Run it locally and in CI:

```bash
uv run python scripts/check_repo_hygiene.py
```

## Common mistakes

- Committing `db.sqlite3`, `.env`, `media/`, or `staticfiles/`.
- Adding a file to `.gitignore` after it was committed and assuming it is now untracked.
- Adding `migrations/` to `.gitignore`. Migrations belong in the repository.
- Editing or deleting a migration that has already been applied elsewhere.
- Resolving a migration conflict by hand-editing dependencies without understanding the graph.
- Making giant commits with vague messages such as "fix stuff".
- Working directly on `main` with no review or CI.
- Using `git add .` without checking `git status`, which stages files you did not intend to commit.
- Rewriting shared history with force pushes.

## Practice exercises

1. Initialize a repository for your Django project, write a `.gitignore`, and make a first commit. Use `git status` to confirm that `db.sqlite3` and `.venv/` do not appear.
2. Create a branch, add a model and its migration, commit both, merge to `main`, and print the history with `git log --oneline --graph`.
3. Deliberately commit `db.sqlite3`, then remove it from tracking without deleting the local file. Describe each command you used.
4. Simulate a migration conflict: create two branches that each add a different model to the same app, merge them, and resolve the conflict with `makemigrations --merge`. Explain what the generated merge migration contains.
5. Add a rule to `find_violations` that flags any tracked file whose name ends in `.sqlite3` or `.log`, and update the tests.
6. Write a commit message for a change that adds an index to a large table. Include a body that explains the reason and the deployment risk.
7. Configure a branch protection rule on GitHub for `main`, requiring a pull request and a passing status check, and list what each setting prevents.

## Summary

Git tracks history through commits, branches, and remotes. In a Django project, commit source, templates, migrations, `pyproject.toml`, and the lock file, and never commit databases, secrets, static output, uploads, or caches. `.gitignore` does not untrack files that are already committed. Migration conflicts after merges are resolved with `makemigrations --merge` or by regenerating an unshared migration, and applied migrations must never be edited. Automate the rules with a repository hygiene script and protect `main` with reviews and CI.
