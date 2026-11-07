## Overview

SQLite is convenient, but production Django applications run on PostgreSQL, often with Redis beside it. Developing against a different database than production hides bugs: SQL dialects differ, concurrency and locking behave differently, and PostgreSQL-only features do not exist in SQLite. Docker Compose lets you run PostgreSQL and Redis as containers with one command, identical on Windows, macOS, and Linux. Your Django code keeps running on your machine, while the services run in containers. This note covers the Compose file, volumes, ports, environment files, health checks, connecting Django, resetting the database, and a management command that waits for the database to be ready.

## Core concepts

### Why match production

- A query that works on SQLite may fail or behave differently on PostgreSQL (case sensitivity, type strictness, locking, `JSONField` operators, full-text search).
- Migrations that succeed on SQLite can fail on PostgreSQL.
- Finding these problems on your laptop costs minutes; finding them in production costs an incident.

### Containers and images

An image is a packaged filesystem and default command. A container is a running instance of an image. Public images such as `postgres` and `redis` are published on Docker Hub, with version tags such as `postgres:17`.

### Docker Compose

Compose describes several containers in one YAML file and starts, stops, and connects them together. Commands use `docker compose` (with a space, the modern form).

### Ports

A port mapping such as `5432:5432` publishes the container's port on your machine, so Django running on your machine reaches PostgreSQL at `localhost:5432`. If you already run PostgreSQL locally, map to a different host port such as `5433:5432`.

### Volumes

Containers are disposable: their filesystem disappears with them. A named volume stores data outside the container so the database survives restarts. Deleting the volume is how you reset the database.

### Environment files

Credentials and settings are supplied through an `.env` file that Compose reads. Do not commit it; commit an `.env.example` instead (note 0.8).

### Health checks

A health check is a command Compose runs periodically to find out whether a service is actually ready. A container can be running while PostgreSQL inside is still initializing, so dependent services should wait for "healthy", not just "started".

### Docker on each platform

- Windows: Docker Desktop with the WSL 2 backend. Keep project files inside the WSL filesystem for much better performance.
- macOS: Docker Desktop, or an alternative such as Colima.
- Linux: Docker Engine and the Compose plugin, and add your user to the `docker` group.

## Basic syntax examples

### The Compose file

`compose.yaml`:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  postgres_data:
```

### The environment file

`.env` (not committed):

```text
POSTGRES_DB=appdb
POSTGRES_USER=app
POSTGRES_PASSWORD=local-dev-password
DATABASE_URL=postgres://app:local-dev-password@localhost:5432/appdb
REDIS_URL=redis://localhost:6379/0
DJANGO_DEBUG=true
DJANGO_SECRET_KEY=local-only-key-change-me-to-something-long-and-random-1234567890
```

Commit a copy named `.env.example` with placeholder values.

### Everyday commands

```bash
docker compose up -d          # start in the background
docker compose ps             # list services and health
docker compose logs -f db     # follow the database logs
docker compose exec db psql -U app -d appdb
docker compose stop           # stop, keep data
docker compose down           # remove containers, keep data
docker compose down -v        # remove containers AND the data volume
```

On Windows PowerShell the commands are identical:

```powershell
docker compose up -d
docker compose ps
docker compose down -v
```

### Connect Django

Install the PostgreSQL driver, then reuse the loader from note 0.5:

```bash
uv add "psycopg[binary]"
```

```python
import os

from .env import build_database_config, load_env_settings

_env = load_env_settings(os.environ)
DATABASES = {"default": build_database_config(_env.database_url)}
```

Run the project with the environment file loaded:

```bash
uv run --env-file .env python manage.py migrate
uv run --env-file .env python manage.py runserver
```

### Inspect the database

```bash
docker compose exec db psql -U app -d appdb -c "\dt"
```

```sql
SELECT version();
SELECT count(*) FROM django_migrations;
```

## Deeper behavior

### Data persistence and resets

`docker compose down` removes containers but leaves the named volume, so data remains. `docker compose down -v` also deletes the volume, which gives you a clean database. This is the fastest way to recover from a broken local schema. Note that changing `POSTGRES_PASSWORD` in `.env` after the volume exists does nothing, because the credentials are applied only when the volume is first initialized. Reset the volume to apply new ones.

### Health versus started

`depends_on` with `condition: service_healthy` makes one service wait for another to pass its health check. For services you run on your host (Django), there is no Compose dependency, so your code or a script must wait. The management command in this note handles that.

### Port conflicts

If `5432` is already in use, Compose fails with a "port is already allocated" error. Either stop the other PostgreSQL or change the host side of the mapping, for example `"5433:5432"`, and update `DATABASE_URL` to port `5433`.

### Pin versions

Use a specific major version such as `postgres:17` rather than `latest`. When production runs PostgreSQL 17, development should too. Upgrade both deliberately.

### Binding to localhost only

`"5432:5432"` publishes the port on all interfaces of your machine. On a shared network, prefer `"127.0.0.1:5432:5432"` so that only your machine can connect.

### Resource usage

Docker Desktop reserves memory and CPUs. If your machine is slow, lower the allocation in its settings and run only the services you need, for example `docker compose up -d db`.

### Containerized Django comes later

This note keeps Django on your machine for fast reloads and easy debugging. Phase 12 builds a production-like image and a full Compose stack that includes the application.

## Industry-level snippet

Scripts, CI jobs, and teammates often start Django before PostgreSQL is accepting connections. A management command that waits for the database, with a timeout and clear errors, solves the race.

Create the folders `apps/core/management/` and `apps/core/management/commands/`, each with an empty `__init__.py`, and register `apps.core` in `INSTALLED_APPS`.

`apps/core/management/commands/wait_for_db.py`:

```python
"""Block until the default database accepts connections."""

import logging
import time
from typing import Any

from django.core.management.base import BaseCommand, CommandError, CommandParser
from django.db import OperationalError, connections

logger = logging.getLogger(__name__)


class Command(BaseCommand):
    help = "Wait until the default database accepts connections."

    def add_arguments(self, parser: CommandParser) -> None:
        parser.add_argument(
            "--timeout",
            type=float,
            default=30.0,
            help="Maximum seconds to wait (default: 30).",
        )
        parser.add_argument(
            "--interval",
            type=float,
            default=1.0,
            help="Seconds between attempts (default: 1).",
        )

    def handle(self, *args: Any, **options: Any) -> None:
        timeout = float(options["timeout"])
        interval = float(options["interval"])
        if timeout <= 0 or interval <= 0:
            raise CommandError("--timeout and --interval must be positive")

        deadline = time.monotonic() + timeout
        attempt = 0
        while True:
            attempt += 1
            try:
                connections["default"].ensure_connection()
            except OperationalError as exc:
                if time.monotonic() >= deadline:
                    raise CommandError(
                        f"Database unavailable after {timeout:g} seconds "
                        f"({attempt} attempts)"
                    ) from exc
                logger.info("Database not ready (attempt %d); retrying", attempt)
                time.sleep(interval)
            else:
                self.stdout.write(self.style.SUCCESS("Database is available"))
                return
```

`apps/core/tests/test_wait_for_db.py` (with an empty `apps/core/tests/__init__.py`):

```python
from io import StringIO
from unittest import mock

from django.core.management import call_command
from django.core.management.base import CommandError
from django.db import OperationalError, connections
from django.test import TestCase


class WaitForDbTests(TestCase):
    def test_succeeds_immediately_when_database_is_up(self) -> None:
        out = StringIO()
        call_command("wait_for_db", stdout=out)
        self.assertIn("Database is available", out.getvalue())

    def test_retries_until_database_comes_up(self) -> None:
        out = StringIO()
        with mock.patch.object(
            connections["default"],
            "ensure_connection",
            side_effect=[OperationalError("down"), OperationalError("down"), None],
        ) as ensure:
            call_command("wait_for_db", "--interval", "0.001", stdout=out)
        self.assertEqual(ensure.call_count, 3)
        self.assertIn("Database is available", out.getvalue())

    def test_gives_up_after_timeout(self) -> None:
        with mock.patch.object(
            connections["default"],
            "ensure_connection",
            side_effect=OperationalError("down"),
        ):
            with self.assertRaisesMessage(CommandError, "unavailable"):
                call_command(
                    "wait_for_db", "--timeout", "0.05", "--interval", "0.001"
                )

    def test_rejects_non_positive_values(self) -> None:
        with self.assertRaises(CommandError):
            call_command("wait_for_db", "--timeout", "0")
```

Typical use in a script or CI step:

```bash
docker compose up -d
uv run --env-file .env python manage.py wait_for_db --timeout 60
uv run --env-file .env python manage.py migrate
```

## Common mistakes

- Developing on SQLite and discovering PostgreSQL differences only at deploy time.
- Using the `latest` image tag, which changes under you and diverges from production.
- Committing `.env` with real credentials. Commit only `.env.example`.
- Changing the database password in `.env` and expecting an existing volume to update.
- Running `docker compose down -v` without realizing it deletes all local data.
- Using `localhost` as the database host from inside another container. Inside a Compose network, services reach each other by service name (`db`), not `localhost`.
- Publishing the database port on all network interfaces on an untrusted network.
- Storing the project on the Windows filesystem while running Docker through WSL 2, which makes file access slow.
- Starting Django immediately after `docker compose up -d` and seeing connection errors because PostgreSQL is still initializing.

## Practice exercises

1. Write a `compose.yaml` with PostgreSQL 17 and Redis 7, start it, and confirm both services report healthy with `docker compose ps`.
2. Connect with `psql` through `docker compose exec`, create a table, restart the stack with `down` and `up -d`, and confirm the data persisted. Then use `down -v` and confirm the data is gone.
3. Change the host port mapping to `5433` and update `DATABASE_URL` so Django still connects. Explain which side of the mapping each number refers to.
4. Run `migrate` against PostgreSQL, then run `\dt` in `psql` and list the tables Django created.
5. Write a `.env.example` for this stack and a short README paragraph telling a new teammate how to start the services and run the tests.
6. Extend `wait_for_db` with a `--database` option that selects a database alias, validating that the alias exists in `DATABASES` and raising `CommandError` otherwise. Add tests.
7. Add a `mailpit` or similar local mail-catching service to the Compose file, point Django's email settings at it, and send a test message from the shell.

## Summary

Developing against the same database engine you deploy on removes a whole class of surprises. Docker Compose runs PostgreSQL and Redis locally with one file: ports expose the services to Django on your machine, named volumes keep data across restarts, `.env` files supply credentials, and health checks tell you when a service is truly ready. Pin image versions, keep `.env` out of Git, and use `down -v` to reset the database deliberately. A small `wait_for_db` management command makes scripts and CI robust against start-up timing, and Phase 12 extends the same approach to a full containerized stack.
