## Overview

Django's ORM lets you write Python instead of SQL, but every ORM call becomes a SQL statement, and every performance problem, data corruption bug, and concurrency surprise you meet will be visible at the SQL level. A Django developer who cannot read SQL is debugging blind. This note teaches the relational model and the SQL you need: tables, keys, the four basic statements, joins, grouping, indexes, normalization, transactions, the differences between SQLite and PostgreSQL, `psql` basics, and reading a query plan. You will practice with raw SQL and with Python's built-in `sqlite3` module, so the same concepts carry straight into the ORM in the next notes.

The running example for all of Phase 2 is a small bookstore: authors, books, reviews, and orders.

## Core concepts

### Tables, rows, and columns

A relational database stores data in tables. A table has columns (each with a name and a type) and rows (one record each). Every row in a table has the same columns.

### Primary and foreign keys

- A primary key uniquely identifies a row. It is never null and never repeats.
- A foreign key is a column that stores the primary key of a row in another table. The database enforces that the referenced row exists.
- Relationships are expressed through foreign keys: a book row stores `author_id`.

### The four basic statements

- `SELECT` reads rows.
- `INSERT` adds rows.
- `UPDATE` changes existing rows.
- `DELETE` removes rows.

`UPDATE` and `DELETE` without a `WHERE` clause affect every row in the table.

### Filtering and ordering

- `WHERE` filters rows, with comparison operators, `AND`, `OR`, `NOT`, `IN`, `BETWEEN`, `LIKE`, and `IS NULL`.
- `ORDER BY` sorts, ascending by default, with `DESC` for descending.
- `LIMIT` and `OFFSET` restrict the number of rows returned.

### NULL

`NULL` means "unknown or absent". It is not equal to anything, not even another `NULL`. Use `IS NULL` and `IS NOT NULL`. Comparisons with `NULL` yield unknown, which `WHERE` treats as false.

### Joins

A join combines rows from two tables on a condition.

- `INNER JOIN`: only rows that match on both sides.
- `LEFT JOIN`: every row from the left table, with `NULL` for missing matches on the right.
- `RIGHT JOIN` and `FULL JOIN`: the mirror image and the union of both. Less common.

### Grouping and aggregates

`GROUP BY` collapses rows that share values into one row per group, and aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) compute a value for each group. `HAVING` filters groups after aggregation, where `WHERE` filters rows before it.

### Indexes

An index is a separate sorted structure that lets the database find rows without scanning the whole table. Indexes speed up reads on the indexed columns and slow down writes, because every `INSERT`, `UPDATE`, and `DELETE` must also maintain them, and they use disk space. Primary keys and unique constraints are indexed automatically.

### Normalization

Normalization organizes data so each fact is stored once. If a book row stored the author's name and birth date, changing a name would require updating many rows, and the copies could disagree. Instead, store authors in their own table and reference them by key. Normalize by default, and denormalize deliberately, for measured performance reasons.

### Transactions and ACID

A transaction groups statements so they succeed or fail together.

- Atomicity: all or nothing.
- Consistency: the database moves from one valid state to another; constraints hold.
- Isolation: concurrent transactions do not see each other's half-finished work (to a configurable degree).
- Durability: once committed, data survives a crash.

### SQLite versus PostgreSQL

- SQLite is a file-based library inside your process. It is excellent for learning, tests, and small apps. It has loose typing, limited concurrent writes, and fewer features.
- PostgreSQL is a server with strict types, strong concurrency, row-level locking, rich data types (arrays, JSONB, ranges), full-text search, and many extensions. It is the standard choice for production Django, and it is what note 0.10 set up.

### psql basics

`psql` is PostgreSQL's command-line client. Meta-commands start with a backslash and are not SQL.

### Reading a query plan

`EXPLAIN` shows how the database intends to run a query: which indexes it will use, how it joins tables, and estimated costs. `EXPLAIN ANALYZE` runs the query and reports real timings.

### Why a Django developer must know SQL

You will read ORM-generated SQL to find N+1 queries, design indexes, write data migrations, debug locking, use window functions the ORM cannot express, and discuss problems with database administrators.

## Basic syntax examples

### Creating tables

```sql
CREATE TABLE author (
    id          INTEGER PRIMARY KEY,
    name        TEXT NOT NULL,
    born_on     DATE
);

CREATE TABLE book (
    id          INTEGER PRIMARY KEY,
    author_id   INTEGER NOT NULL REFERENCES author (id),
    title       TEXT NOT NULL,
    price_cents INTEGER NOT NULL CHECK (price_cents >= 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0)
);
```

On PostgreSQL, use `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` for the key column.

### Inserting, updating, and deleting

```sql
INSERT INTO author (name, born_on) VALUES ('Ursula K. Le Guin', '1929-10-21');
INSERT INTO book (author_id, title, price_cents, stock)
VALUES (1, 'A Wizard of Earthsea', 1299, 10),
       (1, 'The Dispossessed', 1499, 4);

UPDATE book SET stock = stock - 1 WHERE id = 2;

DELETE FROM book WHERE stock = 0;
```

### Selecting

```sql
SELECT id, title, price_cents
FROM book
WHERE stock > 0 AND title LIKE 'The%'
ORDER BY price_cents DESC
LIMIT 10;

SELECT * FROM author WHERE born_on IS NULL;
SELECT * FROM book WHERE id IN (1, 2, 3);
```

### Joins

```sql
SELECT b.title, a.name AS author
FROM book AS b
INNER JOIN author AS a ON a.id = b.author_id;

-- Every author, even those with no books
SELECT a.name, b.title
FROM author AS a
LEFT JOIN book AS b ON b.author_id = a.id;
```

### Grouping

```sql
SELECT a.name, COUNT(b.id) AS book_count, COALESCE(SUM(b.stock), 0) AS total_stock
FROM author AS a
LEFT JOIN book AS b ON b.author_id = a.id
GROUP BY a.id, a.name
HAVING COUNT(b.id) >= 1
ORDER BY book_count DESC;
```

### Indexes

```sql
CREATE INDEX book_author_idx ON book (author_id);
CREATE UNIQUE INDEX author_name_uniq ON author (name);
```

### A transaction

```sql
BEGIN;
UPDATE book SET stock = stock - 1 WHERE id = 1;
INSERT INTO book (author_id, title, price_cents) VALUES (1, 'Lavinia', 1599);
COMMIT;   -- or ROLLBACK; to undo everything since BEGIN
```

### psql in practice

```bash
psql -h localhost -U app -d appdb
```

```text
\l            list databases
\dt           list tables
\d book       describe a table, including indexes and constraints
\x            toggle expanded output
\timing       show query durations
\q            quit
```

### Reading a plan

```sql
EXPLAIN SELECT * FROM book WHERE author_id = 1;
```

```text
Index Scan using book_author_idx on book  (cost=0.15..8.17 rows=1 width=40)
  Index Cond: (author_id = 1)
```

Without the index the plan would say `Seq Scan on book`, meaning every row is read. On tiny tables PostgreSQL prefers sequential scans because they are cheaper; indexes pay off as tables grow.

### Parameters, never string building

```python
import sqlite3

connection = sqlite3.connect(":memory:")
connection.execute("CREATE TABLE author (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")

name = "O'Brien"

# Wrong: breaks on the apostrophe and is open to SQL injection.
# connection.execute(f"INSERT INTO author (name) VALUES ('{name}')")

# Right: the driver sends the value separately from the SQL text.
connection.execute("INSERT INTO author (name) VALUES (?)", (name,))
```

## Deeper behavior

### How a query runs, logically

SQL is declarative: you describe the result, not the steps. Logically the database processes `FROM` and `JOIN`, then `WHERE`, then `GROUP BY`, then `HAVING`, then `SELECT`, then `ORDER BY`, then `LIMIT`. This is why you cannot use a `SELECT` alias inside `WHERE` but can in `ORDER BY`.

### NULL surprises

- `WHERE x = NULL` never matches. Write `WHERE x IS NULL`.
- `NOT IN (subquery)` returns nothing if the subquery yields any `NULL`. Prefer `NOT EXISTS`.
- `COUNT(column)` ignores nulls while `COUNT(*)` counts rows.
- `SUM` over zero rows is `NULL`, so wrap in `COALESCE(SUM(x), 0)`.

### Joins can multiply rows

Joining authors to books and then to reviews yields one row per review. Summing a book-level value across that result counts it once per review. This "join multiplication" is the classic cause of wrong totals, and note 2.8 shows how to avoid it in the ORM.

### What indexes cost and when they help

An index helps `WHERE`, `JOIN`, and `ORDER BY` on its columns, particularly with selective conditions (few matching rows). It does not help when a query reads most of the table, when a function wraps the column (`WHERE lower(title) = 'x'` needs an index on `lower(title)`), or for a pattern with a leading wildcard (`LIKE '%word'`). Every extra index slows writes and uses space, so index for the queries you actually run, and verify with `EXPLAIN`.

### Foreign keys and referential integrity

The database refuses to insert a book with a nonexistent `author_id` and, by default, refuses to delete an author who still has books. Django manages these rules as `on_delete` options (note 2.6). SQLite does not enforce foreign keys unless you run `PRAGMA foreign_keys = ON` on each connection; Django does this for you.

### Isolation and concurrent work

Two transactions running at once can interfere: reading data another transaction is about to change, or both updating the same row based on stale reads. Databases prevent the worst problems with locks and isolation levels. Note 2.9 covers what this means for Django code, including `SELECT ... FOR UPDATE`.

### Dialect differences that bite

- SQLite lets you store a string in an integer column, while PostgreSQL rejects it.
- PostgreSQL string comparison is case-sensitive, and SQLite's `LIKE` is case-insensitive for ASCII.
- Date arithmetic, `RETURNING`, locking, and full-text search differ widely.

Develop against the same database you deploy to (note 0.10).

### The planner uses statistics

PostgreSQL chooses plans from statistics about table contents. After large data changes, `ANALYZE` refreshes them. A plan that looks wrong on an empty development database can be right on production data, so judge plans against realistic volumes.

## Industry-level snippet

A small data-access layer written directly against SQL, using only the standard library. It shows what the ORM does for you: schema constraints, parameterized queries, input validation, explicit errors, atomic multi-statement changes, a grouped report that includes authors with no books, and tests against an in-memory database.

`catalog_sql.py`:

```python
"""A small SQL data-access layer for the bookstore, using only sqlite3."""

import logging
import sqlite3
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA = """
CREATE TABLE IF NOT EXISTS author (
    id   INTEGER PRIMARY KEY,
    name TEXT NOT NULL CHECK (length(trim(name)) > 0)
);
CREATE TABLE IF NOT EXISTS book (
    id          INTEGER PRIMARY KEY,
    author_id   INTEGER NOT NULL REFERENCES author (id),
    title       TEXT NOT NULL CHECK (length(trim(title)) > 0),
    price_cents INTEGER NOT NULL CHECK (price_cents >= 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0)
);
CREATE INDEX IF NOT EXISTS book_author_idx ON book (author_id);
"""


class CatalogError(Exception):
    """Base class for catalog errors."""


class UnknownAuthorError(CatalogError):
    pass


class UnknownBookError(CatalogError):
    pass


class StockError(CatalogError):
    pass


@dataclass(frozen=True)
class AuthorSummary:
    name: str
    book_count: int
    total_stock: int


def _last_id(cursor: sqlite3.Cursor) -> int:
    if cursor.lastrowid is None:
        raise RuntimeError("INSERT did not return a row id")
    return cursor.lastrowid


def _escape_like(term: str) -> str:
    return term.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_")


class Catalog:
    def __init__(self, connection: sqlite3.Connection) -> None:
        connection.row_factory = sqlite3.Row
        connection.execute("PRAGMA foreign_keys = ON")
        connection.executescript(SCHEMA)
        self._conn = connection

    def add_author(self, name: str) -> int:
        cleaned = name.strip()
        if not cleaned:
            raise ValueError("Author name must not be blank")
        with self._conn:  # commits on success, rolls back on error
            cursor = self._conn.execute(
                "INSERT INTO author (name) VALUES (?)", (cleaned,)
            )
        logger.info("Added author %r", cleaned)
        return _last_id(cursor)

    def add_book(
        self, *, author_id: int, title: str, price_cents: int, stock: int = 0
    ) -> int:
        cleaned = title.strip()
        if not cleaned:
            raise ValueError("Book title must not be blank")
        if price_cents < 0 or stock < 0:
            raise ValueError("price_cents and stock must not be negative")
        try:
            with self._conn:
                cursor = self._conn.execute(
                    "INSERT INTO book (author_id, title, price_cents, stock) "
                    "VALUES (?, ?, ?, ?)",
                    (author_id, cleaned, price_cents, stock),
                )
        except sqlite3.IntegrityError as exc:
            raise UnknownAuthorError(f"Author {author_id} does not exist") from exc
        return _last_id(cursor)

    def get_stock(self, book_id: int) -> int:
        row = self._conn.execute(
            "SELECT stock FROM book WHERE id = ?", (book_id,)
        ).fetchone()
        if row is None:
            raise UnknownBookError(f"Book {book_id} does not exist")
        return int(row["stock"])

    def move_stock(
        self, *, from_book_id: int, to_book_id: int, quantity: int
    ) -> None:
        """Move ``quantity`` units between two books, atomically."""
        if quantity <= 0:
            raise ValueError("quantity must be positive")
        if from_book_id == to_book_id:
            raise ValueError("source and destination must differ")
        with self._conn:
            taken = self._conn.execute(
                "UPDATE book SET stock = stock - ? WHERE id = ? AND stock >= ?",
                (quantity, from_book_id, quantity),
            )
            if taken.rowcount == 0:
                raise StockError(
                    f"Book {from_book_id} does not exist or has fewer "
                    f"than {quantity} in stock"
                )
            added = self._conn.execute(
                "UPDATE book SET stock = stock + ? WHERE id = ?",
                (quantity, to_book_id),
            )
            if added.rowcount == 0:
                raise UnknownBookError(f"Book {to_book_id} does not exist")
        logger.info(
            "Moved %d units from book %d to book %d",
            quantity,
            from_book_id,
            to_book_id,
        )

    def search_titles(self, term: str) -> list[str]:
        pattern = f"%{_escape_like(term.strip())}%"
        rows = self._conn.execute(
            "SELECT title FROM book WHERE title LIKE ? ESCAPE '\\' ORDER BY title",
            (pattern,),
        ).fetchall()
        return [str(row["title"]) for row in rows]

    def author_summaries(self) -> list[AuthorSummary]:
        rows = self._conn.execute(
            """
            SELECT a.name AS name,
                   COUNT(b.id) AS book_count,
                   COALESCE(SUM(b.stock), 0) AS total_stock
            FROM author AS a
            LEFT JOIN book AS b ON b.author_id = a.id
            GROUP BY a.id, a.name
            ORDER BY book_count DESC, a.name
            """
        ).fetchall()
        return [
            AuthorSummary(str(r["name"]), int(r["book_count"]), int(r["total_stock"]))
            for r in rows
        ]
```

`test_catalog_sql.py`:

```python
import sqlite3
from collections.abc import Iterator

import pytest

from catalog_sql import (
    AuthorSummary,
    Catalog,
    StockError,
    UnknownAuthorError,
    UnknownBookError,
)


@pytest.fixture
def catalog() -> Iterator[Catalog]:
    connection = sqlite3.connect(":memory:")
    try:
        yield Catalog(connection)
    finally:
        connection.close()


def test_blank_author_name_is_rejected(catalog: Catalog) -> None:
    with pytest.raises(ValueError):
        catalog.add_author("   ")


def test_book_requires_an_existing_author(catalog: Catalog) -> None:
    with pytest.raises(UnknownAuthorError):
        catalog.add_book(author_id=99, title="Orphan", price_cents=100)


def test_negative_price_is_rejected(catalog: Catalog) -> None:
    author_id = catalog.add_author("Ursula")
    with pytest.raises(ValueError):
        catalog.add_book(author_id=author_id, title="Bad", price_cents=-1)


def test_move_stock_changes_both_books(catalog: Catalog) -> None:
    author_id = catalog.add_author("Ursula")
    first = catalog.add_book(author_id=author_id, title="A", price_cents=100, stock=5)
    second = catalog.add_book(author_id=author_id, title="B", price_cents=100, stock=1)

    catalog.move_stock(from_book_id=first, to_book_id=second, quantity=3)

    assert catalog.get_stock(first) == 2
    assert catalog.get_stock(second) == 4


def test_move_stock_fails_without_changes_when_stock_is_short(
    catalog: Catalog,
) -> None:
    author_id = catalog.add_author("Ursula")
    first = catalog.add_book(author_id=author_id, title="A", price_cents=100, stock=2)
    second = catalog.add_book(author_id=author_id, title="B", price_cents=100, stock=0)

    with pytest.raises(StockError):
        catalog.move_stock(from_book_id=first, to_book_id=second, quantity=3)

    assert catalog.get_stock(first) == 2
    assert catalog.get_stock(second) == 0


def test_move_stock_rolls_back_when_destination_is_missing(catalog: Catalog) -> None:
    author_id = catalog.add_author("Ursula")
    first = catalog.add_book(author_id=author_id, title="A", price_cents=100, stock=5)

    with pytest.raises(UnknownBookError):
        catalog.move_stock(from_book_id=first, to_book_id=999, quantity=2)

    assert catalog.get_stock(first) == 5  # the first UPDATE was undone


def test_search_treats_input_as_data_not_sql(catalog: Catalog) -> None:
    author_id = catalog.add_author("Ursula")
    catalog.add_book(author_id=author_id, title="Earthsea", price_cents=100)

    assert catalog.search_titles("'; DROP TABLE book; --") == []
    assert catalog.search_titles("Earth") == ["Earthsea"]


def test_search_wildcards_are_escaped(catalog: Catalog) -> None:
    author_id = catalog.add_author("Ursula")
    catalog.add_book(author_id=author_id, title="Earthsea", price_cents=100)
    assert catalog.search_titles("%") == []


def test_author_summaries_include_authors_without_books(catalog: Catalog) -> None:
    busy = catalog.add_author("Busy")
    catalog.add_author("Idle")
    catalog.add_book(author_id=busy, title="A", price_cents=100, stock=3)
    catalog.add_book(author_id=busy, title="B", price_cents=100, stock=4)

    assert catalog.author_summaries() == [
        AuthorSummary("Busy", 2, 7),
        AuthorSummary("Idle", 0, 0),
    ]
```

Everything the Django ORM later does for you, you have now seen done by hand: constraints in the schema, parameter binding, transactions through `with`, and a join with grouping.

## Common mistakes

- Building SQL with string formatting or f-strings, which causes bugs and SQL injection. Always pass parameters separately.
- Running `UPDATE` or `DELETE` without `WHERE`.
- Comparing to `NULL` with `=`, or using `NOT IN` with a subquery that may return `NULL`.
- Joining several one-to-many tables and then summing, which multiplies rows and inflates totals.
- Adding indexes everywhere "to be fast" instead of indexing for specific queries and checking the plan.
- Wrapping an indexed column in a function in the `WHERE` clause, which prevents the index from being used.
- Testing only on SQLite and discovering PostgreSQL differences in production.
- Storing the same fact in many places (denormalizing by accident) and letting the copies drift apart.
- Doing a read, then a write based on it, outside a transaction or lock, which creates race conditions.
- Judging a query plan on an almost empty table.

## Practice exercises

1. Write the `CREATE TABLE` statements for `review` (with a foreign key to `book`, a rating that must be between 1 and 5, and a text body), then insert five reviews.
2. Write a query listing each book with its author's name and its average review rating, including books with no reviews, sorted by average rating descending with unrated books last.
3. Write a query returning authors who have at least two books whose total stock is below ten. Explain why this needs `HAVING` and not `WHERE`.
4. Demonstrate join multiplication: join authors, books, and reviews, sum book stock per author, and compare with the correct total computed from the book table alone. Explain the difference.
5. Create an index on `book (author_id)` in PostgreSQL, insert 100,000 generated rows, and compare `EXPLAIN ANALYZE` output for a lookup by `author_id` with and without the index.
6. Extend `Catalog` with `delete_author(author_id)` that refuses to delete an author who still has books by raising a clear error, and write tests for both cases.
7. Write a transaction in `psql` that moves stock between two books, then deliberately violates the `CHECK (stock >= 0)` constraint halfway through and observe what `ROLLBACK` restores.

## Summary

A relational database stores facts in tables linked by foreign keys, and SQL reads and changes them with `SELECT`, `INSERT`, `UPDATE`, and `DELETE`, combined through joins and grouped with aggregates. Indexes speed up reads at the cost of writes and space, normalization keeps each fact in one place, and transactions make multi-step changes atomic. SQLite is a convenient learning tool, but PostgreSQL is the production target. Always pass parameters instead of building SQL strings, expect `NULL` to behave unusually, watch for join multiplication, and use `EXPLAIN` to see what the database actually does. Everything the ORM does in the coming notes is these concepts wearing Python syntax.
