## Overview

Not everything an application does happens in response to a web request. Importing a spreadsheet of books, deleting expired API keys, rebuilding a search index, or sending a nightly report are jobs that run from a terminal, a scheduler, or a deployment script. Django's management commands are the standard way to write them: a command is a Python class with its own arguments, access to the full Django environment (settings, models, the ORM), and a conventional way to report success or failure through its exit code. You have already used built-in commands such as `migrate` and `makemigrations`. This note covers writing your own: `BaseCommand`, `add_arguments`, `handle`, output styles, exit codes with `CommandError`, making commands idempotent and safe to re-run, running them from cron and schedulers, data import and maintenance commands, and testing with `call_command`. It ends with two tested commands: a CSV book importer with a dry-run mode and an all-or-nothing strict mode, and a batched cleanup command for stale API keys.

## Core concepts

### What a management command is

A Python module inside an app's `management/commands/` folder defines a class named `Command` that extends `BaseCommand`. The module name is the command name: `apps/bookstore/management/commands/import_books.py` is run with `python manage.py import_books`. Both `management/` and `commands/` need an empty `__init__.py`.

### The structure of a command

- `help`: a class attribute that appears in `manage.py help import_books`.
- `add_arguments(self, parser)`: declares arguments with `argparse`, which Django uses for parsing.
- `handle(self, *args, **options)`: the code that runs, receiving the parsed arguments as keyword options.

### Arguments

Arguments come in two kinds:

- Positional arguments: `parser.add_argument("path")`.
- Options: `parser.add_argument("--dry-run", action="store_true")`, `--batch-size` with `type=int` and a default, and `choices`, `nargs`, `required`, and mutually exclusive groups from `argparse`.

Every command also receives Django's standard options such as `--verbosity`, `--settings`, `--traceback`, and `--no-color`.

### Output

Write through `self.stdout` and `self.stderr`, not `print`, so that tests can capture the output and Django can handle color and encoding. `self.style` provides styles: `SUCCESS`, `WARNING`, `ERROR`, `NOTICE`, and `HTTP_*` helpers.

### Exit codes and CommandError

A command signals failure by raising `CommandError("message")`. When run from the terminal, Django prints the message to standard error and exits with status 1, which is what schedulers, CI jobs, and shell scripts check. An unhandled exception also fails the command but prints a traceback. Raise `CommandError` for expected problems (bad arguments, missing files), and let unexpected bugs propagate.

### Idempotent and re-runnable

An idempotent command can run twice, or be interrupted and run again, without causing harm: the second run does nothing new or finishes the first run's work. Jobs fail, servers restart, and schedulers occasionally run a job twice, so idempotency is the most important property of a command that runs unattended.

### Running commands on a schedule

Commands are the unit of work for cron, systemd timers, Kubernetes CronJobs, and platform schedulers. The scheduler runs `python manage.py purge_api_keys`, checks the exit code, and records the output. For jobs that need retries, queues, or progress tracking, a background task system (Phase 9) is the better fit, and commands often become thin wrappers that enqueue work.

### Data import commands

Importers read files, validate rows, and write to the database. Good ones report what they did, support a dry run, handle bad rows without losing good ones (or all-or-nothing when asked), and are safe to re-run.

### Maintenance commands

Cleanup jobs delete expired sessions, old logs, revoked tokens, orphaned files, and soft-deleted rows past their retention period. They should work in batches so they never lock a large table for long.

### Testing commands

`call_command("name", *args, stdout=..., stderr=...)` runs a command in-process with captured output, raising `CommandError` instead of exiting. That makes commands as easy to test as any function.

## Basic syntax examples

### The smallest command

`apps/bookstore/management/commands/hello.py`:

```python
from typing import Any

from django.core.management.base import BaseCommand


class Command(BaseCommand):
    help = "Print a greeting."

    def handle(self, *args: Any, **options: Any) -> None:
        self.stdout.write("Hello from a management command")
```

```bash
uv run python manage.py hello
uv run python manage.py help hello
```

### Arguments and options

```python
from typing import Any

from django.core.management.base import BaseCommand, CommandParser


class Command(BaseCommand):
    help = "Greet someone."

    def add_arguments(self, parser: CommandParser) -> None:
        parser.add_argument("name", help="Who to greet.")
        parser.add_argument("--times", type=int, default=1, help="How many times.")
        parser.add_argument("--shout", action="store_true")
        parser.add_argument("--style", choices=["plain", "fancy"], default="plain")

    def handle(self, *args: Any, **options: Any) -> None:
        text = f"Hello, {options['name']}"
        if options["shout"]:
            text = text.upper()
        for _ in range(options["times"]):
            self.stdout.write(text)
```

```bash
uv run python manage.py greet Ada --times 3 --shout
```

### Output styles and verbosity

```python
self.stdout.write(self.style.SUCCESS("Done"))
self.stdout.write(self.style.WARNING("Skipped 3 rows"))
self.stderr.write(self.style.ERROR("Row 7 is invalid"))

if options["verbosity"] >= 2:
    self.stdout.write("Detailed progress ...")
```

```bash
uv run python manage.py import_books books.csv --verbosity 2
```

### Failing with an exit code

```python
from pathlib import Path

from django.core.management.base import CommandError

path = Path(options["path"])
if not path.is_file():
    raise CommandError(f"File not found: {path}")
```

```bash
uv run python manage.py import_books missing.csv
echo $?          # 1
```

### Calling a command from code and from tests

```python
from io import StringIO

from django.core.management import call_command

out = StringIO()
call_command("greet", "Ada", "--times", "2", stdout=out)
print(out.getvalue())
```

### Scheduling with cron

```bash
# Every night at 02:15, skip the run if the previous one is still going.
15 2 * * * cd /srv/app && flock -n /tmp/purge_api_keys.lock .venv/bin/python manage.py purge_api_keys --days 30 >> /var/log/purge.log 2>&1
```

### A systemd timer

```text
# /etc/systemd/system/purge-api-keys.service
[Service]
Type=oneshot
WorkingDirectory=/srv/app
ExecStart=/srv/app/.venv/bin/python manage.py purge_api_keys --days 30

# /etc/systemd/system/purge-api-keys.timer
[Timer]
OnCalendar=*-*-* 02:15:00
Persistent=true

[Install]
WantedBy=timers.target
```

## Deeper behavior

### How a command runs

`manage.py` finds the command module, instantiates `Command`, and calls `run_from_argv()`, which parses the arguments, runs Django's system checks (unless the command disables them), and calls `execute()`, which calls your `handle()`. Whatever `handle()` returns as a string is written to standard output. If it raises `CommandError`, `run_from_argv()` writes the message to standard error and exits with status 1.

### System checks and migrations

By default commands run the system checks first, which catches configuration mistakes before your code runs. A command that must work even when checks fail (diagnostics, repair tools) can set `requires_system_checks = []`. Setting `requires_migrations_checks = True` makes the command warn when migrations are unapplied.

### The full Django environment

A command runs with settings loaded and apps ready, so you can use models, caches, email, and everything else. Be aware that it does not go through the request cycle: no middleware, no `request.user`, no request ID. Log with the standard `logging` module and write structured output so that scheduler logs are searchable.

### Transactions

A command is not wrapped in a transaction by default. Choose the scope deliberately:

- One transaction for the whole run gives all-or-nothing behavior, but holds locks for as long as the command runs. That suits small, fast imports.
- A transaction per batch limits lock time and lets a long job make progress, at the price of partial results after a failure, which idempotency must handle.
- No transaction is fine for single idempotent statements.

For a dry run, run the real code inside `transaction.atomic()` and call `transaction.set_rollback(True)` at the end, so the dry run exercises exactly the same path and leaves no trace.

### Memory and batches

A command that loads a whole table into a list will exhaust memory on production data. Use `queryset.iterator(chunk_size=...)` for reads (note 2.7), `bulk_create` and `bulk_update` with `batch_size` for writes, and for deletes loop over batches of primary keys so no single statement is enormous.

### Overlapping runs

If a nightly job takes longer than expected, the scheduler may start a second copy while the first is still running. Prevent damage with a lock outside the program (`flock -n` in cron), an advisory lock in the database (`pg_try_advisory_lock`), or a design in which concurrent runs are harmless because each batch is idempotent. Kubernetes CronJobs offer `concurrencyPolicy: Forbid`.

### Exit codes and monitoring

Schedulers treat exit status 0 as success. A command that catches every exception and exits normally hides failures from monitoring. Let failures propagate or raise `CommandError`, and send a heartbeat or alert when a critical job does not complete (note 11.7).

### Arguments are strings until you convert them

`argparse` converts values with `type=`: `int`, `float`, `pathlib.Path`, or your own function that raises `argparse.ArgumentTypeError` for invalid input. Validate ranges inside `handle()` too, and raise `CommandError` with a clear message.

### Naming and discoverability

Use `verb_noun` names (`import_books`, `purge_api_keys`, `sync_roles`). Give every command a `help` string and every argument a `help` string, because `manage.py help <command>` is the documentation your teammates will read at 3 a.m.

### Commands and secrets

Pass secrets through the environment, never through arguments, because arguments appear in process listings and shell history. Do not log them either (note 5.8).

### Commands versus other tools

- Use a command for operational jobs run by a person or scheduler.
- Use a data migration (note 2.3) for one-time changes tied to a schema version.
- Use a background task (Phase 9) for work triggered by users or events that needs retries and visibility.
- Use `shell` for one-off exploration, never for repeatable production changes.

## Industry-level snippet

Two commands. The first imports books from a CSV file: the parsing is a pure module that reports problems with line numbers, the command supports `--dry-run` and an all-or-nothing `--strict` mode, and re-running it is harmless. The second deletes stale API keys in batches and can be scheduled safely.

`apps/bookstore/csv_import.py`:

```python
"""Parsing book rows from CSV text, with line numbers for every problem."""

import csv
from collections.abc import Iterable, Mapping
from dataclasses import dataclass, field
from decimal import Decimal, InvalidOperation
from typing import Any

from apps.bookstore.services.importing import BookRow

REQUIRED_COLUMNS = ("isbn", "title", "author", "price")
MAX_ROWS = 100_000


class CsvFormatError(ValueError):
    """The file as a whole cannot be used."""


@dataclass(frozen=True)
class ParseError:
    line: int
    message: str


@dataclass
class ParsedFile:
    rows: list[BookRow] = field(default_factory=list)
    line_numbers: list[int] = field(default_factory=list)  # line of each parsed row
    errors: list[ParseError] = field(default_factory=list)


def _parse_row(raw: Mapping[str, Any]) -> BookRow:
    def text(name: str) -> str:
        value = str(raw.get(name) or "").strip()
        if not value:
            raise ValueError(f"{name} is required")
        return value

    try:
        price = Decimal(text("price"))
    except InvalidOperation as exc:
        raise ValueError("price is not a number") from exc
    if not price.is_finite():
        raise ValueError("price is not a finite number")

    stock_text = str(raw.get("stock") or "").strip()
    try:
        stock = int(stock_text) if stock_text else 0
    except ValueError as exc:
        raise ValueError("stock is not a whole number") from exc
    if stock < 0:
        raise ValueError("stock must not be negative")

    return BookRow(
        isbn=text("isbn"),
        title=text("title"),
        author_name=text("author"),
        price=price,
        stock=stock,
    )


def parse_book_rows(lines: Iterable[str]) -> ParsedFile:
    """Parse CSV ``lines``. Row problems are collected; file problems raise."""
    reader = csv.DictReader(lines)
    if reader.fieldnames is None:
        raise CsvFormatError("The file is empty")
    missing = [column for column in REQUIRED_COLUMNS if column not in reader.fieldnames]
    if missing:
        raise CsvFormatError(f"Missing required columns: {', '.join(missing)}")

    parsed = ParsedFile()
    for line_number, raw in enumerate(reader, start=2):  # the header is line 1
        if len(parsed.rows) + len(parsed.errors) >= MAX_ROWS:
            raise CsvFormatError(f"The file has more than {MAX_ROWS} rows")
        try:
            parsed.rows.append(_parse_row(raw))
            parsed.line_numbers.append(line_number)
        except ValueError as exc:
            parsed.errors.append(ParseError(line_number, str(exc)))
    return parsed
```

`apps/bookstore/management/commands/import_books.py` (with `__init__.py` in `management/` and `commands/`):

```python
import logging
from pathlib import Path
from typing import Any

from django.core.management.base import BaseCommand, CommandError, CommandParser
from django.db import transaction

from apps.bookstore.csv_import import CsvFormatError, parse_book_rows
from apps.bookstore.services.importing import import_books

logger = logging.getLogger(__name__)


class Command(BaseCommand):
    help = "Import books from a CSV file with the columns isbn, title, author, price and optionally stock."

    def add_arguments(self, parser: CommandParser) -> None:
        parser.add_argument("path", type=Path, help="Path to the CSV file (UTF-8).")
        parser.add_argument(
            "--dry-run",
            action="store_true",
            help="Validate and report, but roll back every change.",
        )
        parser.add_argument(
            "--strict",
            action="store_true",
            help="All or nothing: if any row has a problem, import nothing and exit with an error.",
        )
        parser.add_argument("--batch-size", type=int, default=500, help="Rows per insert batch.")

    def handle(self, *args: Any, **options: Any) -> None:
        path: Path = options["path"]
        dry_run: bool = options["dry_run"]
        strict: bool = options["strict"]
        batch_size: int = options["batch_size"]

        if batch_size <= 0:
            raise CommandError("--batch-size must be positive")
        if not path.is_file():
            raise CommandError(f"File not found: {path}")

        try:
            with path.open(newline="", encoding="utf-8-sig") as handle:  # also accepts a BOM
                parsed = parse_book_rows(handle)
        except CsvFormatError as exc:
            raise CommandError(str(exc)) from exc
        except UnicodeDecodeError as exc:
            raise CommandError("The file is not valid UTF-8") from exc

        problems = [(error.line, error.message) for error in parsed.errors]

        with transaction.atomic():
            result = import_books(parsed.rows, batch_size=batch_size)
            problems.extend(
                (parsed.line_numbers[error.index], error.message) for error in result.errors
            )
            if dry_run or (strict and problems):
                transaction.set_rollback(True)

        for line, message in sorted(problems):
            self.stderr.write(self.style.WARNING(f"line {line}: {message}"))

        if strict and problems:
            raise CommandError(f"{len(problems)} rows have problems; nothing was imported")

        prefix = "[dry run] " if dry_run else ""
        self.stdout.write(
            self.style.SUCCESS(
                f"{prefix}Created {result.created} books; "
                f"{result.already_present} already present; {len(problems)} rejected."
            )
        )
        logger.info(
            "import_books finished: created=%d present=%d rejected=%d dry_run=%s",
            result.created,
            result.already_present,
            len(problems),
            dry_run,
        )
```

`apps/accounts/management/commands/purge_api_keys.py`:

```python
import logging
from datetime import timedelta
from typing import Any

from django.core.management.base import BaseCommand, CommandError, CommandParser
from django.db.models import Q
from django.utils import timezone

from apps.accounts.models import ApiKey

logger = logging.getLogger(__name__)


class Command(BaseCommand):
    help = "Delete API keys that expired or were revoked more than --days days ago."

    def add_arguments(self, parser: CommandParser) -> None:
        parser.add_argument("--days", type=int, default=30, help="Grace period in days.")
        parser.add_argument("--batch-size", type=int, default=1000)

    def handle(self, *args: Any, **options: Any) -> None:
        days: int = options["days"]
        batch_size: int = options["batch_size"]
        if days < 1:
            raise CommandError("--days must be at least 1")
        if batch_size < 1:
            raise CommandError("--batch-size must be at least 1")

        cutoff = timezone.now() - timedelta(days=days)
        stale = ApiKey.objects.filter(Q(revoked_at__lt=cutoff) | Q(expires_at__lt=cutoff))

        total = 0
        while True:  # small batches keep each statement and lock short
            ids = list(stale.values_list("pk", flat=True)[:batch_size])
            if not ids:
                break
            deleted, _ = ApiKey.objects.filter(pk__in=ids).delete()
            total += deleted

        self.stdout.write(self.style.SUCCESS(f"Deleted {total} stale API keys."))
        logger.info("purge_api_keys deleted %d keys older than %d days", total, days)
```

`apps/bookstore/tests/test_commands.py`:

```python
import tempfile
from datetime import timedelta
from io import StringIO
from pathlib import Path

from django.core.management import call_command
from django.core.management.base import CommandError
from django.test import TestCase
from django.utils import timezone

from apps.accounts.api_keys import create_api_key, revoke_api_key
from apps.accounts.models import ApiKey, User
from apps.bookstore.models import Book

GOOD_CSV = """isbn,title,author,price,stock
9780306406157,Dune,Frank Herbert,12.50,3
9780131103627,The C Programming Language,Brian Kernighan,49.00,
"""

MIXED_CSV = """isbn,title,author,price,stock
9780306406157,Dune,Frank Herbert,12.50,3
9780131103627,Bad price,Someone,abc,
9780201633611,Bad ISBN,Someone,10,
"""


class CsvCommandTestCase(TestCase):
    def setUp(self) -> None:
        directory = tempfile.TemporaryDirectory()
        self.addCleanup(directory.cleanup)
        self.directory = Path(directory.name)

    def write(self, text: str, name: str = "books.csv", encoding: str = "utf-8") -> Path:
        path = self.directory / name
        path.write_text(text, encoding=encoding)
        return path

    def run_import(self, *args: str) -> tuple[str, str]:
        out, err = StringIO(), StringIO()
        call_command("import_books", *args, stdout=out, stderr=err)
        return out.getvalue(), err.getvalue()


class ImportBooksCommandTests(CsvCommandTestCase):
    def test_valid_rows_are_imported(self) -> None:
        out, err = self.run_import(str(self.write(GOOD_CSV)))
        self.assertIn("Created 2 books", out)
        self.assertEqual(err, "")
        self.assertEqual(Book.objects.count(), 2)

    def test_running_it_twice_is_harmless(self) -> None:
        path = str(self.write(GOOD_CSV))
        self.run_import(path)
        out, _ = self.run_import(path)
        self.assertIn("Created 0 books; 2 already present", out)
        self.assertEqual(Book.objects.count(), 2)

    def test_problem_rows_are_reported_with_line_numbers_and_good_rows_still_import(self) -> None:
        out, err = self.run_import(str(self.write(MIXED_CSV)))
        self.assertIn("line 3: price is not a number", err)
        self.assertIn("line 4:", err)                      # the bad check digit, found at import
        self.assertIn("Created 1 books", out)
        self.assertEqual(list(Book.objects.values_list("title", flat=True)), ["Dune"])

    def test_dry_run_reports_but_changes_nothing(self) -> None:
        out, _ = self.run_import(str(self.write(GOOD_CSV)), "--dry-run")
        self.assertIn("[dry run] Created 2 books", out)
        self.assertEqual(Book.objects.count(), 0)

    def test_strict_mode_is_all_or_nothing(self) -> None:
        with self.assertRaises(CommandError):
            self.run_import(str(self.write(MIXED_CSV)), "--strict")
        self.assertEqual(Book.objects.count(), 0)

    def test_a_byte_order_mark_is_accepted(self) -> None:
        path = self.write(GOOD_CSV, encoding="utf-8-sig")
        out, _ = self.run_import(str(path))
        self.assertIn("Created 2 books", out)

    def test_bad_input_files_fail_with_clear_messages(self) -> None:
        cases = {
            "missing file": str(self.directory / "nope.csv"),
            "empty file": str(self.write("", name="empty.csv")),
            "missing column": str(self.write("isbn,title\n1,2\n", name="cols.csv")),
        }
        for label, path in cases.items():
            with self.subTest(case=label):
                with self.assertRaises(CommandError):
                    self.run_import(path)

    def test_invalid_batch_size_is_rejected(self) -> None:
        with self.assertRaises(CommandError):
            self.run_import(str(self.write(GOOD_CSV)), "--batch-size", "0")


class PurgeApiKeysCommandTests(TestCase):
    def setUp(self) -> None:
        self.user = User.objects.create_user("ada@example.com", "correct-horse-battery-staple")

    def purge(self, *args: str) -> str:
        out = StringIO()
        call_command("purge_api_keys", *args, stdout=out)
        return out.getvalue()

    def test_only_keys_past_the_grace_period_are_deleted(self) -> None:
        now = timezone.now()
        old_revoked, _ = create_api_key(user=self.user, name="old revoked")
        recent_revoked, _ = create_api_key(user=self.user, name="recent revoked")
        old_expired, _ = create_api_key(user=self.user, name="old expired")
        active, _ = create_api_key(user=self.user, name="active")
        ApiKey.objects.filter(pk=old_revoked.pk).update(revoked_at=now - timedelta(days=40))
        ApiKey.objects.filter(pk=old_expired.pk).update(expires_at=now - timedelta(days=40))
        revoke_api_key(recent_revoked)

        output = self.purge("--days", "30", "--batch-size", "1")

        self.assertIn("Deleted 2 stale API keys", output)
        remaining = set(ApiKey.objects.values_list("name", flat=True))
        self.assertEqual(remaining, {"recent revoked", "active"})

    def test_a_second_run_finds_nothing(self) -> None:
        key, _ = create_api_key(user=self.user, name="old")
        ApiKey.objects.filter(pk=key.pk).update(revoked_at=timezone.now() - timedelta(days=90))
        self.purge()
        self.assertIn("Deleted 0 stale API keys", self.purge())

    def test_invalid_options_are_rejected(self) -> None:
        with self.assertRaises(CommandError):
            self.purge("--days", "0")
        with self.assertRaises(CommandError):
            self.purge("--batch-size", "0")
```

The importer's `--strict` mode works because the real import runs inside a transaction that is rolled back when problems exist, so strict mode exercises exactly the code that a normal run does.

## Common mistakes

- Using `print` instead of `self.stdout` and `self.stderr`, which breaks output capture in tests.
- Catching every exception and returning normally, so the scheduler reports success for a failed job.
- Writing commands that are not idempotent, so a restart or a duplicate run corrupts data.
- Loading whole tables into memory instead of using `iterator()` and batches.
- Deleting millions of rows in one statement, locking the table.
- Running one transaction for a multi-hour job, or no transaction for a job that needs all-or-nothing behavior.
- Passing secrets as command-line arguments.
- Forgetting `__init__.py` in `management/` and `management/commands/`, so Django cannot find the command.
- Letting two scheduled runs overlap without a lock.
- Using a management command for work that needs retries, progress tracking, and visibility, where a background task fits better.
- Skipping the dry run for destructive commands, or implementing a dry run as separate code that can drift from the real one.
- Parsing arguments by hand from `sys.argv` instead of `add_arguments`.

## Practice exercises

1. Write a command `list_low_stock --threshold N --format table|csv` that prints low-stock books, validates the threshold, and returns exit code 1 when any book has zero stock. Test both formats.
2. Add an `--only-new` option to `import_books` that skips rows whose ISBN exists before the import starts, and test that the output counts differ from the default.
3. Write `export_orders --since DATE --output PATH` that streams orders to a CSV file using `iterator()` and `values_list`, refuses to overwrite an existing file unless `--force` is given, and test it with a temporary directory.
4. Add a database advisory lock (PostgreSQL) to `purge_api_keys` so two concurrent runs cannot overlap. Write a PostgreSQL-only test that runs two copies in threads and proves only one deletes.
5. Write a command `check_data_integrity` that runs a handful of queries (orphaned order items, negative stock, duplicate active ISBNs), prints one line per problem, and exits non-zero if it finds any. Test it with deliberately broken data created through raw SQL.
6. Convert `purge_api_keys` to take `--dry-run` and use the rollback technique from the importer so the count printed is exactly what a real run would delete.
7. Write the Kubernetes CronJob manifest (YAML) that runs `purge_api_keys` nightly with `concurrencyPolicy: Forbid`, a deadline, and a history limit, and explain how a failed run becomes visible to your monitoring.

## Summary

A management command is a `Command` class in an app's `management/commands/` folder, with arguments declared in `add_arguments` and work done in `handle`. Write output through `self.stdout` and `self.stderr`, signal failure with `CommandError` so schedulers see a non-zero exit status, and keep commands idempotent so retries and duplicate runs are harmless. Choose transaction scope deliberately: one for all-or-nothing imports (and dry runs by rolling back), batches for long jobs. Stream and batch large reads and writes, prevent overlapping runs with an external or advisory lock, pass secrets through the environment, and test commands with `call_command` and temporary files like any other code.
