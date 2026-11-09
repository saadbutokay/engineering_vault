## Overview

Every Django process talks to the database over connections, and connections are a limited, expensive resource. PostgreSQL allocates memory for each one and refuses new ones past `max_connections`. An application that opens a connection per request wastes time on handshakes, and an application that holds a connection per worker thread across many servers can exhaust the database. This note explains how Django manages connections, how persistent connections and the native connection pool introduced in Django 5.1 reduce overhead, how to size connection usage, how to route queries across several databases such as a primary and read replicas, what can go wrong with replication lag, how the test database works, and where PgBouncer fits. It ends with a validated settings builder and a tested read-replica router.

## Core concepts

### The DATABASES setting

`DATABASES` is a dictionary of named database configurations. The alias `"default"` is required. Each entry has `ENGINE`, `NAME`, `USER`, `PASSWORD`, `HOST`, `PORT`, and options such as `CONN_MAX_AGE`, `ATOMIC_REQUESTS`, `OPTIONS`, and `TEST`.

### Connection lifecycle

By default Django opens a connection when a request first needs the database and closes it at the end of the request (`CONN_MAX_AGE = 0`). Connections are per thread: each thread has its own connection to each configured database.

### Persistent connections: CONN_MAX_AGE

`CONN_MAX_AGE = 600` keeps a connection open for up to 600 seconds and reuses it for later requests handled by the same thread, avoiding the cost of reconnecting. `CONN_HEALTH_CHECKS = True` makes Django test a reused connection at the start of a request and reconnect if it has gone stale. `None` means never close, which is rarely wise.

### Native connection pooling (Django 5.1, PostgreSQL)

With `psycopg` version 3 and its pool package, Django can maintain a pool of connections inside each process:

```python
"OPTIONS": {"pool": {"min_size": 2, "max_size": 10, "timeout": 10}}
```

Threads borrow a connection for the duration of a request and return it, so the number of connections is bounded by `max_size` instead of by the number of threads. Pooling cannot be combined with persistent connections (`CONN_MAX_AGE` must be 0). On Django 5.2 LTS this is available; on earlier versions use persistent connections or an external pooler.

### Connection limits

Total connections to the database are roughly the number of processes multiplied by the connections each process may hold, summed over every service: web workers, background workers, scheduled jobs, and admin tools. That total must stay below the server's `max_connections`, with headroom for maintenance and migrations.

### Multiple databases and routers

Django can talk to several databases. A database router decides which alias each read, write, relation, and migration uses. The classic case is a primary database for writes and one or more read replicas for reads.

### Choosing a database explicitly

- `Model.objects.using("replica")` runs a query on a named database.
- `instance.save(using="default")` writes to a named database.
- `transaction.atomic(using="default")` opens a transaction on one database.
- `manage.py migrate --database=default` migrates one database.

### Replication lag

A replica receives changes from the primary after a delay, usually milliseconds and occasionally seconds or more. A request that writes and then immediately reads from a replica may not see its own write. This is the main hazard of read replicas.

### The test database

Django creates a separate database for tests named with a `test_` prefix, runs migrations in it, and destroys it afterward. A test class must declare which databases it may use through the `databases` attribute. By default only `default` is allowed.

### PgBouncer

PgBouncer is an external connection pooler that sits between your application and PostgreSQL, multiplexing many client connections onto few server connections. In transaction pooling mode, which is the efficient one, session-level features break: server-side cursors, session variables, and advisory locks held across transactions.

### psycopg 3

`psycopg` (version 3) is the modern PostgreSQL driver and the one Django's pooling support requires. Install `psycopg[binary,pool]`, or `psycopg[c,pool]` for production images that compile the C extension.

## Basic syntax examples

### A PostgreSQL entry with persistent connections

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "appdb",
        "USER": "app",
        "PASSWORD": "secret",
        "HOST": "localhost",
        "PORT": "5432",
        "CONN_MAX_AGE": 600,
        "CONN_HEALTH_CHECKS": True,
    }
}
```

### The same entry with the native pool

```bash
uv add "psycopg[binary,pool]"
```

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "appdb",
        "USER": "app",
        "PASSWORD": "secret",
        "HOST": "localhost",
        "PORT": "5432",
        "CONN_MAX_AGE": 0,   # required with pooling
        "OPTIONS": {"pool": {"min_size": 2, "max_size": 10, "timeout": 10}},
    }
}
```

### A primary and a replica

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "appdb",
        "HOST": "primary.internal",
        # ...
    },
    "replica": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "appdb",
        "HOST": "replica.internal",
        "TEST": {"MIRROR": "default"},   # in tests, use the primary instead
        # ...
    },
}

DATABASE_ROUTERS = ["config.db.PrimaryReplicaRouter"]
```

### Choosing a database by hand

```python
from apps.bookstore.models import Book

Book.objects.using("replica").filter(stock__gt=0)
book = Book(title="Dune")
book.save(using="default")
```

### Migrating a specific database

```bash
uv run python manage.py migrate --database=default
```

### Allowing a test to use a second database

```python
from django.test import TestCase


class ReportTests(TestCase):
    databases = {"default", "replica"}
```

### Counting server connections

```sql
SELECT count(*) FROM pg_stat_activity WHERE datname = 'appdb';
SHOW max_connections;
```

### A rough sizing check

```text
web:     4 gunicorn workers x 4 threads, no pool, CONN_MAX_AGE=600  ->  up to 16 connections per server
pooled:  4 workers x pool max_size 5                                 ->  up to 20 connections per server
workers: 2 Celery processes x 4 threads                              ->  up to 8 connections
```

Multiply by the number of servers and compare with `max_connections`.

## Deeper behavior

### How persistent connections behave

With `CONN_MAX_AGE`, each thread keeps its connection between requests. Django closes the connection if it is older than the limit at the start or end of a request. If the database restarted or a firewall dropped idle connections, the next request fails unless `CONN_HEALTH_CHECKS` is enabled, which makes Django verify the connection at the start of each request and reopen it if needed. Long-lived threads outside request handling, such as management commands and background tasks, do not get automatic closing, so call `connection.close()` or `close_old_connections()` where appropriate.

### Persistent connections and async

Under ASGI, synchronous code runs in a thread pool, and database connections are tied to those threads. With `CONN_MAX_AGE` greater than zero, connections can accumulate across the pool's threads, so size them and watch totals. The native pool, which hands connections out per request, suits that setup better. Phase 8 covers async database access in depth.

### How the native pool works

Each Django process creates its own pool, holding between `min_size` and `max_size` connections. A request checks out a connection and returns it when it finishes. If all connections are in use, a request waits up to `timeout` seconds and then fails with a pool timeout error. Pool size is therefore a deliberate cap and a source of back-pressure. Choose `max_size` from the number of threads that actually run queries simultaneously, not from the total number of threads. Because each process has its own pool, multiply `max_size` by the number of processes when budgeting against `max_connections`.

### Choosing between the options

- Few processes and plenty of database capacity: default or persistent connections are fine.
- Many threads, serverless-like bursts, or ASGI: prefer the native pool.
- Many application servers and a small database: put PgBouncer in front, and plan for its transaction-mode restrictions.
- Never use both the native pool and `CONN_MAX_AGE` on the same alias.

### PgBouncer caveats

In transaction pooling mode, a client does not own a server connection between transactions. Server-side cursors, which Django uses for `QuerySet.iterator()` on PostgreSQL, rely on a persistent session, so set `DISABLE_SERVER_SIDE_CURSORS = True` in the database settings when using it. Features such as session-level advisory locks, `SET` commands, and `LISTEN` are also unreliable. Use session pooling for those, or connect directly.

### How routers are consulted

For each query Django asks the routers in `DATABASE_ROUTERS`, in order, calling `db_for_read(model, **hints)` or `db_for_write(model, **hints)`. A router returning `None` defers to the next one, and if none answer, Django uses `default`. `allow_relation(obj1, obj2)` decides whether two objects from different databases may be related (return `True` for databases that hold the same data, such as a primary and its replicas). `allow_migrate(db, app_label, model_name, **hints)` decides which database a migration operates on, which for replicas must be `False` because they receive their schema from the primary through replication.

### Replication lag in practice

A typical failure: a user submits a form, the view writes to the primary and redirects, and the next page reads from a replica that has not caught up, so the new record seems missing. Remedies:

- Read from the primary right after a write, in the same request or for a short time afterward.
- Send reads that need to be current (inside transactions, after writes, in checkout flows) to the primary explicitly.
- Keep replicas for reports, search listings, and other reads that tolerate staleness.
- Monitor lag, and stop routing to a replica that falls behind.

The router below supports the first two with a `use_primary()` context manager.

### Transactions are per database

`transaction.atomic()` uses the default database. Writes to different databases are not in one transaction, so a failure between them can leave them inconsistent. If two databases must change together, redesign so one of them is the source of truth, or use a pattern such as the outbox (Phase 9).

### Relations across databases

Foreign keys cannot be enforced between separate databases, and Django does not support joins across them. Keep related models in one database. Replicas hold the same data, so relations among replica objects are fine.

### Test database details

Django creates `test_<NAME>` by default, runs migrations, and drops it at the end of the run. Use `--keepdb` to reuse it between runs for speed, and `--parallel` to run test processes against cloned databases. A replica alias with `"TEST": {"MIRROR": "default"}` shares the primary's test database, so tests that read through the replica see the data written through the primary within the same test.

### psycopg 3 details

Django 4.2 and later support `psycopg` 3. By default Django uses client-side parameter binding with it for compatibility, and the `server_side_binding` option in `OPTIONS` enables server-side binding, which can improve plan caching but changes some behaviors. The binary wheel (`psycopg[binary]`) is convenient for development, and the C implementation is preferred in production images.

### Connection limits and errors

"too many connections" and "remaining connection slots are reserved" errors mean the total across all processes exceeds `max_connections`. Fix them by lowering the per-process maximum, reducing the number of processes, adding PgBouncer, or increasing `max_connections` with care, because each connection costs memory on the server.

## Industry-level snippet

A settings builder that validates connection options and refuses incompatible combinations, and a read-replica router that keeps writes on the primary, spreads reads across replicas, and lets code force the primary for read-your-writes consistency. Both are typed and tested.

`config/db.py`:

```python
"""Database settings builder and primary/replica router."""

import logging
import random
from collections.abc import Callable, Iterator, Sequence
from contextlib import contextmanager
from contextvars import ContextVar
from dataclasses import dataclass
from typing import Any

from django.conf import settings
from django.db.models import Model

from config.env import ConfigError

logger = logging.getLogger(__name__)

PRIMARY_ALIAS = "default"


@dataclass(frozen=True)
class PoolSettings:
    min_size: int = 2
    max_size: int = 10
    timeout: float = 10.0

    def __post_init__(self) -> None:
        if self.min_size < 1:
            raise ConfigError("Pool min_size must be at least 1")
        if self.max_size < self.min_size:
            raise ConfigError("Pool max_size must not be smaller than min_size")
        if self.timeout <= 0:
            raise ConfigError("Pool timeout must be positive")

    def as_options(self) -> dict[str, int | float]:
        return {
            "min_size": self.min_size,
            "max_size": self.max_size,
            "timeout": self.timeout,
        }


def build_postgres_database(
    *,
    name: str,
    user: str,
    password: str,
    host: str,
    port: int = 5432,
    conn_max_age: int = 0,
    pool: PoolSettings | None = None,
    atomic_requests: bool = False,
    connect_timeout: int = 5,
) -> dict[str, Any]:
    """Return a validated PostgreSQL ``DATABASES`` entry."""
    if not name or not host:
        raise ConfigError("Database name and host are required")
    if not 1 <= port <= 65535:
        raise ConfigError(f"Invalid database port: {port}")
    if conn_max_age < 0:
        raise ConfigError("conn_max_age must not be negative")
    if pool is not None and conn_max_age > 0:
        raise ConfigError("Connection pooling cannot be combined with conn_max_age")
    if connect_timeout < 1:
        raise ConfigError("connect_timeout must be at least 1 second")

    options: dict[str, Any] = {"connect_timeout": connect_timeout}
    if pool is not None:
        options["pool"] = pool.as_options()

    return {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": name,
        "USER": user,
        "PASSWORD": password,
        "HOST": host,
        "PORT": str(port),
        "CONN_MAX_AGE": conn_max_age,
        "CONN_HEALTH_CHECKS": conn_max_age > 0,
        "ATOMIC_REQUESTS": atomic_requests,
        "OPTIONS": options,
    }


_force_primary: ContextVar[bool] = ContextVar("force_primary", default=False)


@contextmanager
def use_primary() -> Iterator[None]:
    """Send every read in this block to the primary (read-your-writes)."""
    token = _force_primary.set(True)
    try:
        yield
    finally:
        _force_primary.reset(token)


class PrimaryReplicaRouter:
    """Writes go to the primary; reads are spread across configured replicas.

    Replica aliases are listed in ``settings.REPLICA_DATABASES``.
    """

    def __init__(
        self, chooser: Callable[[Sequence[str]], str] | None = None
    ) -> None:
        self._chooser = chooser or random.choice  # noqa: S311 - not security related

    @staticmethod
    def _replicas() -> tuple[str, ...]:
        return tuple(getattr(settings, "REPLICA_DATABASES", ()))

    def db_for_read(self, model: type[Model], **hints: Any) -> str:
        replicas = self._replicas()
        if _force_primary.get() or not replicas:
            return PRIMARY_ALIAS
        return self._chooser(replicas)

    def db_for_write(self, model: type[Model], **hints: Any) -> str:
        return PRIMARY_ALIAS

    def allow_relation(self, obj1: Model, obj2: Model, **hints: Any) -> bool:
        pool = {PRIMARY_ALIAS, *self._replicas()}
        return obj1._state.db in pool and obj2._state.db in pool

    def allow_migrate(
        self,
        db: str,
        app_label: str,
        model_name: str | None = None,
        **hints: Any,
    ) -> bool:
        return db == PRIMARY_ALIAS
```

Using it in `config/settings.py`:

```python
import os

from config.db import PoolSettings, build_postgres_database

DATABASES = {
    "default": build_postgres_database(
        name="appdb",
        user="app",
        password=os.environ["POSTGRES_PASSWORD"],
        host=os.environ.get("POSTGRES_HOST", "localhost"),
        pool=PoolSettings(min_size=2, max_size=8),
    ),
}
DATABASE_ROUTERS = ["config.db.PrimaryReplicaRouter"]
REPLICA_DATABASES: tuple[str, ...] = ()   # add aliases such as ("replica",) when you have them
```

`config/test_db.py`:

```python
from typing import Any

import pytest
from django.test import SimpleTestCase, override_settings

from apps.bookstore.models import Author, Book
from config.db import (
    PoolSettings,
    PrimaryReplicaRouter,
    build_postgres_database,
    use_primary,
)
from config.env import ConfigError

GOOD: dict[str, Any] = {
    "name": "appdb",
    "user": "app",
    "password": "secret",
    "host": "db.internal",
}


def test_builds_a_valid_entry() -> None:
    config = build_postgres_database(**GOOD)
    assert config["HOST"] == "db.internal"
    assert config["PORT"] == "5432"
    assert config["CONN_HEALTH_CHECKS"] is False
    assert config["OPTIONS"] == {"connect_timeout": 5}


def test_persistent_connections_enable_health_checks() -> None:
    config = build_postgres_database(**GOOD, conn_max_age=600)
    assert config["CONN_MAX_AGE"] == 600
    assert config["CONN_HEALTH_CHECKS"] is True


def test_pool_is_added_to_options() -> None:
    config = build_postgres_database(**GOOD, pool=PoolSettings(min_size=2, max_size=6))
    assert config["OPTIONS"]["pool"] == {"min_size": 2, "max_size": 6, "timeout": 10.0}


def test_pool_and_persistent_connections_are_incompatible() -> None:
    with pytest.raises(ConfigError, match="pooling"):
        build_postgres_database(**GOOD, conn_max_age=60, pool=PoolSettings())


@pytest.mark.parametrize(
    "overrides",
    [{"port": 0}, {"port": 70000}, {"conn_max_age": -1}, {"host": ""}, {"connect_timeout": 0}],
)
def test_invalid_values_are_rejected(overrides: dict[str, int | str]) -> None:
    with pytest.raises(ConfigError):
        build_postgres_database(**{**GOOD, **overrides})


@pytest.mark.parametrize(
    ("min_size", "max_size", "timeout"),
    [(0, 5, 10.0), (5, 2, 10.0), (1, 5, 0.0)],
)
def test_invalid_pool_settings_are_rejected(
    min_size: int, max_size: int, timeout: float
) -> None:
    with pytest.raises(ConfigError):
        PoolSettings(min_size=min_size, max_size=max_size, timeout=timeout)


class RouterTests(SimpleTestCase):
    def setUp(self) -> None:
        self.router = PrimaryReplicaRouter(chooser=lambda replicas: replicas[-1])

    @override_settings(REPLICA_DATABASES=("replica1", "replica2"))
    def test_reads_use_a_replica_and_writes_use_the_primary(self) -> None:
        self.assertEqual(self.router.db_for_read(Book), "replica2")
        self.assertEqual(self.router.db_for_write(Book), "default")

    @override_settings(REPLICA_DATABASES=())
    def test_reads_fall_back_to_the_primary_without_replicas(self) -> None:
        self.assertEqual(self.router.db_for_read(Book), "default")

    @override_settings(REPLICA_DATABASES=("replica1",))
    def test_use_primary_forces_reads_to_the_primary(self) -> None:
        with use_primary():
            self.assertEqual(self.router.db_for_read(Book), "default")
        self.assertEqual(self.router.db_for_read(Book), "replica1")

    @override_settings(REPLICA_DATABASES=("replica1",))
    def test_use_primary_is_reset_even_when_the_block_fails(self) -> None:
        with self.assertRaises(RuntimeError):
            with use_primary():
                raise RuntimeError("boom")
        self.assertEqual(self.router.db_for_read(Book), "replica1")

    @override_settings(REPLICA_DATABASES=("replica1",))
    def test_migrations_run_only_on_the_primary(self) -> None:
        self.assertTrue(self.router.allow_migrate("default", "bookstore"))
        self.assertFalse(self.router.allow_migrate("replica1", "bookstore"))

    @override_settings(REPLICA_DATABASES=("replica1",))
    def test_relations_are_allowed_inside_the_primary_replica_pool(self) -> None:
        first = Author(name="A")
        second = Author(name="B")
        first._state.db = "default"
        second._state.db = "replica1"
        self.assertTrue(self.router.allow_relation(first, second))
        second._state.db = "analytics"
        self.assertFalse(self.router.allow_relation(first, second))
```

The mix of pytest-style functions and Django test cases is deliberate: pure functions are simplest as plain functions, and anything needing `override_settings` fits `SimpleTestCase`. If your project runs only one runner, convert accordingly.

## Common mistakes

- Setting `CONN_MAX_AGE` high without `CONN_HEALTH_CHECKS`, then seeing intermittent "connection closed" errors after a database restart.
- Combining the native pool with `CONN_MAX_AGE > 0`.
- Sizing connections per process and forgetting to multiply by the number of processes and servers, then hitting `max_connections`.
- Forgetting that background workers, scheduled jobs, and migration tasks also consume connections.
- Reading from a replica immediately after writing and concluding that data was lost.
- Running migrations against a replica, or letting a router send them there.
- Using `QuerySet.iterator()` through PgBouncer in transaction mode without disabling server-side cursors.
- Expecting a `transaction.atomic()` block to cover writes to two different databases.
- Creating foreign keys between models that live in different databases.
- Forgetting to declare `databases = {...}` on test classes that touch a second alias.
- Setting `CONN_MAX_AGE = None` on servers with many threads, holding connections forever.
- Treating pool timeouts as a bug to remove by raising the limit, rather than a signal that the application holds connections too long.

## Practice exercises

1. Start PostgreSQL from note 0.10, open several concurrent `psql` sessions, and watch `pg_stat_activity` while running Django with `CONN_MAX_AGE = 0` and then `600`. Record the difference in connection counts and behavior.
2. Switch the project to the native pool with `max_size = 4`, write a script with ten threads that each run a slow query, and observe the queueing and the timeout behavior when you lower `timeout`.
3. Write a function `connection_budget(servers: int, processes_per_server: int, per_process: int, other_clients: int, max_connections: int) -> int` that returns the headroom (possibly negative) and raises on non-positive inputs. Add tests.
4. Configure a second database alias for a local PostgreSQL instance, route reads through `PrimaryReplicaRouter` with `REPLICA_DATABASES`, and prove with query logging that reads go to the replica and writes go to the primary.
5. Write a middleware that calls `use_primary()` for the remainder of a request after any non-safe HTTP method (POST, PUT, PATCH, DELETE), and tests that the router returns `default` for reads inside such requests.
6. Simulate replication lag by writing through `default` and reading through a second alias that points at a separate, stale copy of the data, and show how `use_primary()` removes the inconsistency.
7. Explain, with a concrete bookstore example, why a bank-transfer style operation across two separate databases cannot be made atomic with `transaction.atomic()`, and sketch an outbox-based alternative.

## Summary

Django keeps one connection per thread per database alias, closes it after each request by default, and can instead reuse connections (`CONN_MAX_AGE` with `CONN_HEALTH_CHECKS`) or, since Django 5.1 on PostgreSQL with `psycopg` 3, draw them from a bounded in-process pool, but never both. Total connections equal processes times per-process limits across every service and must stay under the server's `max_connections`; PgBouncer helps at scale but breaks session-level features such as server-side cursors in transaction mode. Routers send reads and writes to different databases, migrations only to the primary, and replication lag means code that needs fresh data must read from the primary. Transactions and foreign keys do not span databases. Validate connection settings at startup, size pools deliberately, and give test classes the `databases` they use.
