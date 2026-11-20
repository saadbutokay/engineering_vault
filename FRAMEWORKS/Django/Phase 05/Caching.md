## Overview

Caching means remembering the result of expensive work so the next request can reuse it. A product page that runs six queries and renders a large template can be served from memory in a fraction of a millisecond. Caching is also one of the easiest ways to introduce bugs: users see stale data, one user sees another user's page, a popular key expires and a hundred requests rebuild it at once. Django ships a flexible cache framework with several backends and tools at three levels: low-level key and value access, per-view caching, and template fragment caching. This note covers the cache framework, its backends (local memory, Redis, Memcached, database, file), `cache.get`, `set`, and `get_or_set`, per-view and fragment caching, cache keys and versioning, invalidation strategies, cache stampedes, what must never be cached, and `cache_page` with the `Vary` header. It ends with a tested "cache aside" helper that adds jittered expiry and stampede protection, and its use for a book's rating summary with invalidation after the commit.

## Core concepts

### What a cache is

A cache is a fast key-value store that holds copies of data computed elsewhere, usually with an expiry time. A request first looks for the value in the cache. On a hit it uses it. On a miss it computes the value, stores it, and uses it. The original data (the database) stays the source of truth.

### Backends

Configured in `CACHES`:

- Local memory (`LocMemCache`): the default, one copy per process. Fast and simple, but each worker process has its own cache, so values differ between processes, and nothing is shared. Good for development and tests.
- Redis (`django.core.cache.backends.redis.RedisCache`, built in since Django 4.0): shared, fast, with persistence options. The usual production choice, and the same Redis you may already run for other purposes (note 11.2).
- Memcached (`PyMemcacheCache`): shared and very fast, no persistence, simple semantics.
- Database (`DatabaseCache`): stores entries in a table. Works anywhere, much slower than memory.
- File (`FileBasedCache`): one file per entry. Rarely a good choice.
- Dummy (`DummyCache`): accepts everything and stores nothing, useful to disable caching in development.

You may configure several caches under different aliases (`default`, `sessions`) and fetch them with `caches["alias"]`.

### The low-level API

`cache.get(key, default)`, `cache.set(key, value, timeout)`, `cache.add(key, value, timeout)` (set only if absent, returning whether it stored), `cache.get_or_set(key, default, timeout)`, `cache.delete(key)`, `cache.get_many(keys)`, `cache.set_many(mapping)`, `cache.incr(key)` and `cache.decr(key)`, `cache.touch(key, timeout)`, and `cache.clear()`. Values must be picklable.

### Timeouts

`timeout` is in seconds. `None` means never expire, and `0` means do not cache. The `TIMEOUT` setting is the default. Choose timeouts from how stale the data may be: seconds for volatile data, minutes for typical content, hours for rarely changing data.

### Per-view caching

`@cache_page(60 * 15)` caches an entire view's response for 15 minutes, keyed by the URL (and headers named in `Vary`). It is the quickest speed-up for public pages that are the same for everyone.

### Template fragment caching

`{% cache 600 sidebar book.pk %}...{% endcache %}` caches a piece of rendered HTML. The extra arguments after the name become part of the key, so values that change the output must be included.

### Cache keys and versioning

A key identifies one cached value. Good keys name the thing and everything that varies it: `book:42:rating-summary:v1`. Settings help: `KEY_PREFIX` separates applications sharing one cache server, and `VERSION` lets you invalidate everything by bumping a number. Keys must be strings without spaces or control characters, and Memcached limits them to 250 characters.

### Invalidation

Removing or replacing cached values when the underlying data changes. Strategies:

- Time-based expiry: accept staleness up to the timeout. Simple and robust.
- Explicit deletion on write: delete the key when the data changes, ideally after the commit.
- Versioned keys: include a version or "generation" number in the key and bump it when data changes, so old entries are simply never read again.
- Write-through: update the cache at the same time as the database.

Combine a short time-based expiry with explicit deletion for the data that matters most. Time-based expiry is the safety net that limits the damage of a missed invalidation.

### Cache stampede

When a popular key expires, many requests miss at once and all rebuild it simultaneously, which can overload the database (also called the thundering herd or dog-piling). Defenses: let one request rebuild while the others wait or serve slightly stale data, add random jitter to expiry times so related keys do not expire together, and refresh before expiry.

### What must never be cached (without care)

- Any response that depends on the user, if cached in a shared cache with a key that does not include the user.
- Data that is permission-sensitive, unless the permission is part of the key.
- Pages containing CSRF tokens or personal data, in shared caches.
- Responses to unsafe methods and responses that set cookies.
- Secrets and credentials.

### cache_page and the Vary header

A response's `Vary` header lists the request headers that change the response. `cache_page` includes those headers in the key. Django adds `Vary: Cookie` automatically when a view touches the session, so personalized pages are cached separately for each cookie value, which is safe and also nearly useless as a shared cache. Use `vary_on_headers("Accept-Language")` and `vary_on_cookie` deliberately, and keep personalized pieces out of cached pages (fetch them separately, or use fragment caching around the public parts only).

### Browser and CDN caching

Besides the server-side cache, HTTP headers control caching in browsers and CDNs: `Cache-Control` (`max-age`, `private`, `no-store`), `ETag`, and `Last-Modified`. Django offers `cache_control`, `never_cache`, and `condition` decorators.

## Basic syntax examples

### Configuring Redis

```bash
uv add redis
```

```python
import os

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": os.environ.get("REDIS_URL", "redis://localhost:6379/1"),
        "KEY_PREFIX": "bookstore",
        "TIMEOUT": 300,
    }
}
```

### Development and test configurations

```python
CACHES = {"default": {"BACKEND": "django.core.cache.backends.locmem.LocMemCache"}}
CACHES = {"default": {"BACKEND": "django.core.cache.backends.dummy.DummyCache"}}   # caching off
```

### The low-level API

```python
from django.core.cache import cache

cache.set("greeting", "hello", timeout=60)
cache.get("greeting")                  # 'hello'
cache.get("missing", "fallback")       # 'fallback'
cache.add("lock", 1, timeout=10)       # True the first time, False while it exists
cache.delete("greeting")
cache.get_many(["a", "b"])             # {'a': ..., 'b': ...} only for keys that exist
cache.set_many({"a": 1, "b": 2}, timeout=60)
cache.incr("counter")                  # raises ValueError if the key does not exist
```

### get_or_set

```python
def expensive() -> int:
    return 42

value = cache.get_or_set("answer", expensive, timeout=300)   # calls expensive() only on a miss
```

### Distinguishing a miss from a cached None

```python
sentinel = object()
value = cache.get("maybe-none", sentinel)
if value is sentinel:
    ...   # a real miss
```

### Per-view caching

```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_headers


@cache_page(60 * 15)
@vary_on_headers("Accept-Language")
def public_catalog(request: HttpRequest) -> HttpResponse:
    ...
```

### Keeping pages out of caches

```python
from django.views.decorators.cache import cache_control, never_cache


@never_cache
def account_page(request: HttpRequest) -> HttpResponse:
    ...


@cache_control(private=True, max_age=60)
def my_orders(request: HttpRequest) -> HttpResponse:
    ...
```

### Template fragment caching

```html
{% load cache %}
{% cache 600 book_sidebar book.pk request.LANGUAGE_CODE %}
  <aside>... expensive sidebar for {{ book.title }} ...</aside>
{% endcache %}
```

### Versioning

```python
cache.set("catalog", data, version=2)
cache.get("catalog", version=2)
cache.incr_version("catalog")
```

### Several caches

```python
from django.core.cache import caches

sessions_cache = caches["sessions"]
```

## Deeper behavior

### Local memory is per process

With four web workers, `LocMemCache` gives you four independent caches. A request may hit a worker that has the value and the next may not, and an invalidation in one process does not reach the others. That is acceptable for data that is cheap to recompute and bad for anything you must invalidate. Use a shared backend (Redis or Memcached) in production.

### Values are pickled

Cached objects are serialized with `pickle`. Anything that can write to your cache server can therefore influence what your application unpickles. Keep the cache server on a private network, protect it with authentication, and never share a cache between systems with different trust levels. Cache plain data (numbers, strings, dictionaries, dataclasses) rather than model instances: instances carry state, can become stale, and may not unpickle after a model changes.

### Keys

Django warns (`CacheKeyWarning`) about keys that would not work on Memcached: over 250 characters or containing whitespace or control characters. Hash long or user-supplied parts of a key (note 4.4 does this for the rate limiter), and never put raw user input into a key, because it lets users flood the cache with distinct keys. Include everything that changes the value: ids, language, a schema version. When you change the shape of a cached value, change a version segment in the key (`:v2`), so old entries are ignored rather than misread.

### Why invalidation is hard

Two problems recur:

- Missed invalidation: you change data in a place that forgot to delete the key. A time-based expiry limits the damage.
- Racing invalidation: a reader misses, loads old data from the database, and then, after a writer changed the data and deleted the key, stores the old value. The stale value now lives until it expires. Delete keys after the commit (not before), keep expiry short for important data, and for critical cases use versioned keys, which cannot be overwritten by stale writers.

### Invalidate after the commit

Deleting a key inside the transaction creates a window in which another request reloads the old, not-yet-committed data and caches it. Register the deletion with `transaction.on_commit` (note 2.9) so it happens when the new data is visible.

### Stampede protection in detail

The helper below applies three techniques. A lock built with `cache.add` ensures one request rebuilds a missing value. The other requests wait briefly and re-check, instead of all hitting the database. A random jitter spreads expiry times of related keys. If the lock holder never fills the cache (it crashed, or the lock expired), waiters give up and compute the value themselves, so a failure of the rebuilder cannot block everyone. For very hot keys, a further technique is to refresh before expiry, serving the old value while one request renews it.

### Negative caching

A miss that finds nothing (a book with no reviews) should be cached too, or each request repeats the query. That is why a sentinel distinguishes "not in cache" from "cached None" or an empty result. Use shorter timeouts for negative results when the data may appear soon.

### cache_page details

`cache_page` stores only successful responses to GET and HEAD requests, builds the key from the URL, the method, and the headers listed in `Vary`, and sets `Cache-Control: max-age` on the response so browsers and CDNs cache it as well. A page that reads `request.user` or `request.session` becomes `Vary: Cookie`, which means a separate cache entry per cookie. If a page mixes public and personal content, cache the public part as a fragment or load the personal part separately.

### CSRF tokens in cached pages

A page with `{% csrf_token %}` is personal because the token is tied to the user's cookie. Django's CSRF middleware adds `Vary: Cookie` in that case, so `cache_page` will not share it, but a fragment cache that includes a token would. Never include tokens inside cached fragments.

### Cache middleware

`UpdateCacheMiddleware` and `FetchFromCacheMiddleware` cache whole-site responses. They are blunt and easy to get wrong with personalized content, so prefer explicit per-view caching for public pages and a CDN for everything static or fully public.

### Measuring a cache

A cache that is rarely hit is a cost, not a benefit. Track the hit ratio, the time to compute misses, memory use, and evictions. Redis exposes `INFO stats` with `keyspace_hits` and `keyspace_misses`. Measure before adding caching, as note 10.6 explains, and cache the data that is expensive to compute and read far more often than it changes.

### When not to use a cache

Do not cache to hide a slow query that an index or a better query would fix (note 2.7). Do not cache data that must be exactly current (stock at checkout, account balances). Do not cache what is cheap to compute. And do not add caching to code you have not yet profiled.

## Industry-level snippet

A reusable "cache aside" helper with jittered expiry and stampede protection, a cached rating summary for each book, and invalidation after the commit when a review is added. The tests prove that the second call needs no queries, that stale data disappears after the commit, that a cached empty result is a hit, and that concurrent misses run the loader once.

`apps/core/cache_utils.py`:

```python
"""Helpers for the cache-aside pattern."""

import logging
import random
import time
from collections.abc import Callable
from typing import Final, cast

from django.core.cache import BaseCache, caches

logger = logging.getLogger(__name__)

_MISSING: Final = object()


def jittered_ttl(ttl: int, jitter: float = 0.1) -> int:
    """Return ``ttl`` seconds plus or minus up to ``jitter`` (a fraction), at least 1."""
    if ttl < 1:
        raise ValueError("ttl must be at least 1 second")
    if not 0 <= jitter < 1:
        raise ValueError("jitter must be at least 0 and below 1")
    spread = ttl * jitter
    return max(1, round(ttl + random.uniform(-spread, spread)))  # noqa: S311 - not security related


def get_or_load[T](
    key: str,
    loader: Callable[[], T],
    *,
    ttl: int,
    jitter: float = 0.1,
    cache: BaseCache | None = None,
    lock_seconds: int = 10,
    wait_seconds: float = 0.05,
    wait_attempts: int = 5,
) -> T:
    """Return the cached value for ``key``, computing it with ``loader`` on a miss.

    ``None`` is a cacheable value. Only one caller rebuilds a missing value at
    a time; the others wait briefly and re-check. If no value appears, they
    compute it themselves without caching, so a failed rebuilder cannot block
    anyone for long.
    """
    store = cache if cache is not None else caches["default"]

    cached = store.get(key, _MISSING)
    if cached is not _MISSING:
        return cast(T, cached)

    lock_key = f"{key}:rebuild-lock"
    if store.add(lock_key, 1, timeout=lock_seconds):
        try:
            value = loader()
            store.set(key, value, timeout=jittered_ttl(ttl, jitter))
            return value
        finally:
            store.delete(lock_key)

    for _ in range(wait_attempts):
        time.sleep(wait_seconds)
        cached = store.get(key, _MISSING)
        if cached is not _MISSING:
            return cast(T, cached)

    logger.warning("Cache rebuild for %s did not finish; computing without the cache", key)
    return loader()
```

`apps/bookstore/services/rating_summary.py`:

```python
"""Cached review statistics for a book."""

from dataclasses import dataclass
from functools import partial

from django.core.cache import caches
from django.db import transaction
from django.db.models import Avg, Count

from apps.bookstore.models import Book, Review
from apps.core.cache_utils import get_or_load

SUMMARY_TTL_SECONDS = 300


@dataclass(frozen=True)
class RatingSummary:
    count: int
    average: float | None


def _key(book_id: int) -> str:
    return f"book:{book_id}:rating-summary:v1"


def get_rating_summary(book_id: int) -> RatingSummary:
    def load() -> RatingSummary:
        stats = Review.objects.filter(book_id=book_id).aggregate(
            count=Count("id"), average=Avg("rating")
        )
        average = stats["average"]
        return RatingSummary(
            count=int(stats["count"]),
            average=float(average) if average is not None else None,
        )

    return get_or_load(_key(book_id), load, ttl=SUMMARY_TTL_SECONDS)


def invalidate_rating_summary(book_id: int) -> None:
    caches["default"].delete(_key(book_id))


def add_review(*, book: Book, reviewer_name: str, rating: int, body: str = "") -> Review:
    """Create a review and refresh the cached summary after the commit."""
    if rating not in Review.Rating.values:
        raise ValueError("rating must be between 1 and 5")
    with transaction.atomic():
        review = Review.objects.create(
            book=book, reviewer_name=reviewer_name.strip(), rating=rating, body=body
        )
        transaction.on_commit(partial(invalidate_rating_summary, book.pk))
    return review
```

`apps/bookstore/tests/test_caching.py`:

```python
import threading
import time

from django.core.cache import cache
from django.core.cache.backends.locmem import LocMemCache
from django.test import SimpleTestCase, TestCase

from apps.bookstore.models import Review
from apps.bookstore.services.rating_summary import (
    RatingSummary,
    add_review,
    get_rating_summary,
)
from apps.core.cache_utils import get_or_load, jittered_ttl

from .factories import make_book


class RatingSummaryTests(TestCase):
    def setUp(self) -> None:
        cache.clear()
        self.book = make_book()

    def test_the_second_call_is_served_from_the_cache(self) -> None:
        Review.objects.create(book=self.book, reviewer_name="A", rating=5)
        first = get_rating_summary(self.book.pk)
        with self.assertNumQueries(0):
            second = get_rating_summary(self.book.pk)
        self.assertEqual(first, second)
        self.assertEqual(first, RatingSummary(count=1, average=5.0))

    def test_a_book_without_reviews_is_cached_too(self) -> None:
        self.assertEqual(get_rating_summary(self.book.pk), RatingSummary(0, None))
        with self.assertNumQueries(0):
            get_rating_summary(self.book.pk)

    def test_adding_a_review_refreshes_the_summary_after_the_commit(self) -> None:
        add_review(book=self.book, reviewer_name="A", rating=5)
        self.assertEqual(get_rating_summary(self.book.pk).count, 1)

        with self.captureOnCommitCallbacks(execute=False) as callbacks:
            add_review(book=self.book, reviewer_name="B", rating=3)
        self.assertEqual(get_rating_summary(self.book.pk).count, 1)    # still the old value
        for callback in callbacks:
            callback()
        self.assertEqual(get_rating_summary(self.book.pk), RatingSummary(2, 4.0))

    def test_invalid_ratings_are_rejected_before_writing(self) -> None:
        with self.assertRaises(ValueError):
            add_review(book=self.book, reviewer_name="A", rating=9)
        self.assertEqual(Review.objects.count(), 0)


class CacheAsideTests(SimpleTestCase):
    def setUp(self) -> None:
        self.store = LocMemCache("cache-aside-tests", {})
        self.store.clear()

    def test_the_loader_runs_once_per_key(self) -> None:
        calls: list[int] = []

        def load() -> str:
            calls.append(1)
            return "value"

        for _ in range(3):
            self.assertEqual(get_or_load("k", load, ttl=60, cache=self.store), "value")
        self.assertEqual(len(calls), 1)

    def test_none_is_a_cacheable_value(self) -> None:
        calls: list[int] = []

        def load() -> None:
            calls.append(1)

        get_or_load("none", load, ttl=60, cache=self.store)
        get_or_load("none", load, ttl=60, cache=self.store)
        self.assertEqual(len(calls), 1)

    def test_a_failing_loader_releases_the_lock(self) -> None:
        def broken() -> str:
            raise RuntimeError("boom")

        with self.assertRaises(RuntimeError):
            get_or_load("k", broken, ttl=60, cache=self.store)
        self.assertEqual(get_or_load("k", lambda: "ok", ttl=60, cache=self.store), "ok")

    def test_concurrent_misses_run_the_loader_once(self) -> None:
        calls: list[int] = []
        results: list[str] = []

        def slow_loader() -> str:
            calls.append(1)
            time.sleep(0.2)
            return "value"

        def worker() -> None:
            results.append(
                get_or_load(
                    "hot", slow_loader, ttl=60, cache=self.store,
                    wait_seconds=0.05, wait_attempts=40,
                )
            )

        threads = [threading.Thread(target=worker) for _ in range(5)]
        for thread in threads:
            thread.start()
        for thread in threads:
            thread.join()

        self.assertEqual(len(calls), 1)
        self.assertEqual(results, ["value"] * 5)

    def test_waiters_fall_back_when_the_rebuilder_never_finishes(self) -> None:
        self.store.add("stuck:rebuild-lock", 1, timeout=60)     # someone else holds the lock
        value = get_or_load(
            "stuck", lambda: "computed", ttl=60, cache=self.store,
            wait_seconds=0.01, wait_attempts=2,
        )
        self.assertEqual(value, "computed")
        self.assertIsNone(self.store.get("stuck"))              # and it did not cache the result

    def test_jittered_expiry_stays_within_bounds(self) -> None:
        values = {jittered_ttl(100, 0.1) for _ in range(300)}
        self.assertTrue(all(90 <= value <= 110 for value in values))
        self.assertGreater(len(values), 1)

    def test_invalid_expiry_arguments_are_rejected(self) -> None:
        with self.assertRaises(ValueError):
            jittered_ttl(0)
        with self.assertRaises(ValueError):
            jittered_ttl(10, jitter=1.0)
```

The most useful test here is the one that shows the stale value surviving until the commit callback runs. It makes the race visible, and it proves the invalidation is tied to the commit rather than to the code path that happened to write the data.

## Common mistakes

- Caching per-user data under a key that does not include the user, and showing one person's data to another.
- Using `cache_page` on views that read the session, which creates one entry per visitor and no benefit.
- Caching pages or fragments that contain CSRF tokens or personal information in a shared cache.
- Forgetting that `LocMemCache` is per process and then wondering why invalidation does not work across workers.
- Invalidating inside the transaction instead of after the commit.
- Relying only on explicit invalidation with no expiry, so a missed delete makes data stale forever.
- Using one long fixed expiry for many keys that are created together, causing a synchronized stampede.
- Caching model instances instead of plain data.
- Building keys from raw user input, enabling cache flooding.
- Treating a cached `None` or empty value as a miss and re-querying every time.
- Adding a cache before measuring, or to hide a query that needs an index.
- Not monitoring the hit ratio, so a useless cache keeps costing memory and complexity.

## Practice exercises

1. Cache the genre list used in a navigation bar for five minutes with `get_or_load`, invalidate it from a genre save through `transaction.on_commit`, and write a test with `assertNumQueries` for both the hit and the invalidation.
2. Add `@cache_page` to a public catalog view, then make it read `request.user` and observe the added `Vary: Cookie` header with the test client. Explain what happened to the cache.
3. Cache a fragment of the book detail page with `{% cache %}`, including the book id and the language in the key, and write a test that proves a second request does not run the fragment's queries.
4. Implement versioned keys: add a `catalog_generation` counter in the cache, include it in the catalog's key, bump it with `cache.incr` when any book changes, and test that old entries are never read.
5. Write a test that demonstrates the racing-invalidation problem with threads or by interleaving calls (reader loads old data, writer commits and deletes, reader stores old data), then fix it with a versioned key or a short TTL and show the corrected result.
6. Add a `refresh_ahead` option to `get_or_load` that, when the remaining lifetime is below a threshold, lets one caller recompute in the background while others use the existing value. Describe the extra state you needed to store.
7. Configure Redis from note 0.10, point the cache at it, and write a small script that measures `keyspace_hits` and `keyspace_misses` before and after loading the book page 100 times.

## Summary

Django's cache framework stores computed values in a fast key-value store, with local memory for development, and Redis or Memcached for production. Use the low-level API for data, `cache_page` for public pages, and `{% cache %}` for fragments, and always include everything that varies the value in the key. Choose expiry from how stale data may be, add jitter so keys do not expire together, and protect hot keys from stampedes with a rebuild lock. Invalidate after the commit and keep a short expiry as a safety net. Cache plain data rather than model instances, treat a cached empty result as a hit, never share personal or permission-dependent responses through a shared cache, and measure the hit ratio so you know the cache is earning its keep.
