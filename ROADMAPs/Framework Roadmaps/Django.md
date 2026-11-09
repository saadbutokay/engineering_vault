**Django is a high-level, open-source Python web framework designed for rapid development, secure design, and clean architectural paradigms**. Maintained by the [Django Software Foundation](https://www.djangoproject.com/foundation/), it adheres to a **"batteries-included" philosophy**, meaning it provides nearly everything a developer needs to build an application out of the box without relying heavily on third-party plugins.

**Target version:** Django 5.2 LTS or newer, with Django 6.0 features flagged. Each note that uses a version-specific feature names the minimum Django version and the alternative on 5.2 LTS.

**Python version:** 3.12 or newer, which both Django 5.2 and Django 6.0 support.

---
## Phase 0: Environment, Tooling, and Foundations

1. [[How the Web Works]] - Clients and servers, URLs and their parts, HTTP request and response structure, methods and their meanings (GET, POST, PUT, PATCH, DELETE), idempotency and safety, status code families, headers, cookies, HTTPS and TLS at a conceptual level, request and response bodies, `curl` and browser developer tools, what a web framework does and why one is used.

2. [[What Django Is and How It Is Organized]] The Model-Template-View pattern and how it maps to MVC, the request lifecycle from socket to response, projects vs apps, what "batteries included" means in practice, WSGI and ASGI introduced, the release cycle, LTS policy, the deprecation policy, and how to read the official documentation and release notes.

3. [[Creating a Project]] - Installing Django with `uv` and `pip` into a virtual environment, `django-admin startproject`, the generated files and what each one does, `manage.py`, `runserver` and the autoreloader, `startapp`, registering an app, writing a first view and URL, `manage.py check`.

4. [[Editor Setup]] - Visual Studio Code with the Python and Django template extensions, PyCharm as an alternative, `django-stubs` and `mypy` configuration, debugging a running Django server from the editor, the Django shell, `django-extensions` and `shell_plus`, `django-debug-toolbar` installation.

5. [[Settings and Configuration]] - Anatomy of `settings.py`, `BASE_DIR`, `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`, `INSTALLED_APPS`, `DATABASES`, `TEMPLATES`, reading configuration from environment variables, `django-environ` or `pydantic-settings`, settings per environment, `DJANGO_SETTINGS_MODULE`, `manage.py diffsettings`, `check --deploy`, why secrets never go into the repository.

6. [[Project Layout]] - The default layout and its limits, a `config/` package for settings and a `apps/` folder for applications, `AppConfig` and `default_auto_field`, naming apps, splitting an app into modules (`models/`, `views/`, `services/`), `tests/` placement, `templates/` and `static/` placement, `.gitignore` for Django.

7. [[Code Style and Conventions]] - The Django coding style, PEP 8 as applied by `ruff format`, import ordering, naming of models (singular), fields, related names, URL names, and templates, where logic goes in a first pass, `django-upgrade` for modernizing code.

8. [[Git Essentials for Django Projects]] - `init`, `status`, `add`, `commit`, `log`, `diff`, branches, merging, remotes, pushing to GitHub, what to commit (migrations, `pyproject.toml`, lock file) and what to ignore (`db.sqlite3`, `.env`, `staticfiles/`, `media/`, `__pycache__/`), resolving migration conflicts after a merge.

9. [[Django Quality Tools]] - `ruff` for linting and formatting with the Django-specific rule set (`DJ`), `mypy` with `django-stubs`, `djlint` for templates, `pre-commit` hooks, `manage.py check`, `makemigrations --check` in CI, a minimal CI pipeline.

10. [[Local Development Services with Docker Compose]] - Why development should use the same database as production, running PostgreSQL and Redis in containers, volumes and ports, environment files, health checks, connecting Django to the containerized database, resetting the development database, Docker on Windows, macOS, and Linux.

---
## Phase 1: Requests, Responses, and Templates

1. [[URL Routing]] - The `urlconf`, `path()` and `re_path()`, path converters, `include()`, app-level `urls.py`, `app_name` and namespaces, `reverse()` and `{% url %}`, never hard-coding URLs, custom path converters, trailing slashes and `APPEND_SLASH`, how Django matches a URL.

2. [[Function-Based Views]] - The `HttpRequest` object (`method`, `GET`, `POST`, `body`, `headers`, `META`, `user`, `session`), `HttpResponse` and its subclasses, `JsonResponse`, `render`, `redirect`, `get_object_or_404`, `Http404`, `require_http_methods`, returning proper status codes, views as plain functions.

3. [[Class-Based Views]] - The `View` class and `as_view()`, `TemplateView`, `ListView`, `DetailView`, how `dispatch` works, mixins and method resolution order, `get_context_data`, `get_queryset`, deciding between function-based and class-based views, why generic views save code and when they hide too much.

4. [[Templates]] - The Django Template Language (variables, tags, filters), template inheritance with `extends` and `block`, `include`, context and context processors, automatic HTML escaping, `url`, `static`, `csrf_token`, custom template tags and filters, template partials (Django 6.0), the Jinja2 backend as an option, why logic stays out of templates.

5. [[Static Files and Media]] - `STATIC_URL`, `STATICFILES_DIRS`, `collectstatic`, the `STORAGES` setting, serving static files in production with WhiteNoise or a web server, user-uploaded media and `MEDIA_ROOT`, why Django does not serve media in production, cache-busting with hashed filenames.

6. [[Middleware]] - The middleware chain and its onion model, built-in middleware and why order matters, writing function-based and class-based middleware, short-circuiting a request, exception handling in middleware, `LoginRequiredMiddleware` (Django 5.1), request timing and request ID middleware, async-capable middleware.

7. [[Reading Errors and Debugging Basics]] - The debug error page, `404` and `500` handling, custom error templates, reading a Django traceback from the bottom up, common failures (`ImproperlyConfigured`, `TemplateDoesNotExist`, `NoReverseMatch`, `OperationalError`, `DisallowedHost`, `CSRF verification failed`, `InconsistentMigrationHistory`), `print` vs logging vs the debugger, a systematic way to debug a failing request.

---
## Phase 2: Models and the ORM

1. [[Relational Databases and SQL Essentials]] - Tables, rows, columns, primary and foreign keys, `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `WHERE`, `ORDER BY`, `JOIN` types, `GROUP BY`, indexes and what they cost, normalization, transactions and ACID, SQLite vs PostgreSQL, `psql` basics, reading a query plan with `EXPLAIN`, why a Django developer must know SQL.

2. [[Models and Fields]] - Defining a model, field types and their options (`null` vs `blank`, `default`, `db_default` in Django 5.0, `unique`, `choices`), `TextChoices` and `IntegerChoices`, `Meta` options, `__str__`, `GeneratedField` (Django 5.0), primary keys and `BigAutoField`, UUID primary keys, how models map to tables, `inspectdb`.

3. [[Migrations]] - `makemigrations`, `migrate`, `sqlmigrate`, `showmigrations`, the migration graph and dependencies, reading a migration file, `RunPython` and `RunSQL` for data migrations, reversible migrations, squashing, merge migrations, why migration files are committed, never editing an applied migration.

4. [[QuerySet Basics]] - Creating, retrieving, updating, and deleting objects, `all`, `get`, `filter`, `exclude`, `order_by`, slicing and `LIMIT`, lazy evaluation and when SQL runs, the result cache, `exists`, `count`, `first`, `bulk_create`, `bulk_update`, `update`, `delete`, `get_or_create`, `update_or_create`, `DoesNotExist` and `MultipleObjectsReturned`.

5. [[Lookups, Q, F, and Expressions]] - Field lookups (`exact`, `icontains`, `in`, `gt`, `range`, `isnull`, `startswith`), traversing relations with double underscores, `Q` objects for OR and NOT, `F` expressions for database-side arithmetic and race-free updates, `Value`, `Func`, `Case` and `When`, database functions, `Coalesce`, `Lower`, `Concat`, `Cast`.

6. [[Relationships]] - `ForeignKey`, `OneToOneField`, `ManyToManyField` and explicit `through` models, `on_delete` options and their consequences, `related_name`, forward and reverse access, self-referencing relations, relations across apps with string references, generic relations (overview), composite primary keys (Django 5.2).

7. [[The N+1 Problem and Query Optimization]] - How lazy relation access causes N+1 queries, `select_related` and `prefetch_related`, `Prefetch` objects, `only`, `defer`, `values`, `values_list`, `iterator`, counting queries with `connection.queries`, `django-debug-toolbar`, and `assertNumQueries`, `QuerySet.explain()`, finding accidental quadratic behavior in templates and serializers.

8. [[Aggregation, Annotation, and Subqueries]] - `aggregate` vs `annotate`, `Count`, `Sum`, `Avg`, `Min`, `Max`, grouping with `values().annotate()`, filtered aggregates, `Subquery`, `OuterRef`, `Exists`, window functions, `alias`, avoiding Python-side aggregation, the join multiplication pitfall with `Count`.

9. [[Transactions and Concurrency]] - Autocommit behavior, `transaction.atomic` as decorator and context manager, `ATOMIC_REQUESTS` and its trade-offs, savepoints, `transaction.on_commit`, `select_for_update` and row locking, lost updates and race conditions, `F` expressions as the first defense, optimistic locking with a version field, `unique` constraints as idempotency guards, isolation levels overview.

10. [[Constraints, Indexes, and Schema Design]] - `CheckConstraint`, `UniqueConstraint` with conditions and expressions, `Meta.indexes`, composite and partial indexes, covering indexes, deciding what to index, soft delete and its costs, `JSONField` and when it is the wrong choice, database-level defaults, PostgreSQL-specific fields (`ArrayField`, `HStoreField`, range fields), schema design for evolution.

11. [[Managers, Custom QuerySets, and Model Design]] - Default and custom managers, `QuerySet` subclasses with chainable methods, `as_manager`, model methods and properties, `clean`, `full_clean`, and why `save` does not call them, abstract base classes, proxy models, multi-table inheritance and its costs, where business logic belongs (fat models vs service layer introduced), raw SQL with `raw()` and `connection.cursor()` and when it is justified.

12. [[Database Connections, Multiple Databases, and Pooling]] - The `DATABASES` setting, `CONN_MAX_AGE` and persistent connections, native connection pooling for PostgreSQL (Django 5.1), database routers and read replicas, `using()`, replication lag pitfalls, the test database, PgBouncer overview, `psycopg` 3, connection limits in production.

---
## Phase 3: Forms and Validation

1. [[Forms]] - The `Form` class, field types and widgets, `is_valid`, `cleaned_data` and `errors`, bound vs unbound forms, `clean_<field>` and `clean`, rendering forms and field groups (Django 5.0), customizing widgets and templates, CSRF protection in forms, `initial` and `prefix`.

2. [[ModelForm and Formsets]] - `ModelForm`, `Meta.fields` and why `"__all__"` is a security risk, `save(commit=False)`, model and form validation order, formsets and inline formsets, managing child objects, `ModelChoiceField` and its query cost.

3. [[File Uploads]] - `FileField` and `ImageField`, `request.FILES`, upload handlers and size limits, validating file type by content and not by extension, generated filenames and path traversal, storage backends, `django-storages` with S3, serving private files, security of user uploads.

4. [[Form Handling Patterns]] - The Post/Redirect/Get pattern, `CreateView`, `UpdateView`, `DeleteView`, `FormView`, handling form errors, pagination with `Paginator`, search and filter forms with GET, HTMX for partial page updates, progressive enhancement, when a form should be an API instead.

5. [[Validation Architecture]] - Validators on fields, form validation vs model validation vs database constraints, three layers of defense and what each one is for, `ValidationError` with codes and params, cross-field validation, business rules that belong in services, returning validation errors from APIs, consistent error messages.

---
## Phase 4: Authentication, Authorization, and Sessions

- **4.1 Sessions and cookies:** How sessions work, session backends (database, cache, signed cookies), `request.session`, session expiry and `SESSION_COOKIE_*` settings, `HttpOnly`, `Secure`, and `SameSite`, session fixation, why `pickle`-based session serialization is gone, logging out everywhere.
- **4.2 The authentication system:** `User`, `authenticate`, `login`, `logout`, `LoginView` and `LogoutView`, `login_required` and `LoginRequiredMixin`, `LoginRequiredMiddleware` (Django 5.1), password hashing (PBKDF2, Argon2) and password validators, password reset flow, why every project defines a custom user model before the first migration, `AbstractUser` vs `AbstractBaseUser`, email as the login field.
- **4.3 Authorization:** `is_staff`, `is_superuser`, permissions and groups, `has_perm`, `permission_required`, `UserPassesTestMixin`, object-level permissions and the IDOR problem, scoping querysets to the current user, `django-guardian` overview, policy functions, role design for real applications.
- **4.4 Third-party and API authentication:** `django-allauth` for social login and email verification, OAuth 2.0 and OpenID Connect concepts, token authentication, JWT and its trade-offs vs sessions, two-factor authentication, passkeys overview, brute-force protection with `django-axes`, rate limiting login attempts, account enumeration.

---

## Phase 5: Built-in Framework Features

- **5.1 The Django admin:** Registering models, `ModelAdmin` options (`list_display`, `list_filter`, `search_fields`, `readonly_fields`, `fieldsets`), inlines, custom actions, admin permissions, `list_select_related` and avoiding N+1 in admin, facets (Django 5.0), who should use the admin (internal staff, not customers), securing the admin URL and access.
- **5.2 Management commands:** `BaseCommand`, `add_arguments`, `handle`, output styles, exit codes with `CommandError`, idempotent and re-runnable commands, running commands from cron and schedulers, data import and maintenance commands, testing commands with `call_command`.
- **5.3 Signals:** Built-in signals (`pre_save`, `post_save`, `m2m_changed`, `request_finished`), custom signals, `@receiver`, why signals hide control flow, `bulk_create` not firing signals, `transaction.on_commit` for side effects, when an explicit function call is better.
- **5.4 Caching:** The cache framework, backends (local memory, Redis, Memcached, database, file), `cache.get`, `set`, `get_or_set`, per-view and template fragment caching, cache keys and versioning, invalidation strategies, cache stampede, what must never be cached (per-user data without a key), `cache_page` and the `Vary` header.
- **5.5 Email and the messages framework:** `send_mail`, `EmailMessage`, `EmailMultiAlternatives`, backends (console, file, SMTP, locmem), HTML emails with templates, sending email outside the request cycle, the modern email API (Django 6.0), `django-anymail` overview, the `messages` framework and message levels.
- **5.6 Internationalization, localization, and time zones:** `USE_TZ` and aware datetimes, storing UTC and converting at the edge, `timezone.now()`, translation with `gettext` and `gettext_lazy`, `makemessages` and `compilemessages`, `LocaleMiddleware`, formats, pluralization, right-to-left languages overview.
- **5.7 Serialization, fixtures, and contrib apps:** `dumpdata`, `loaddata`, fixtures and why they are poor for tests at scale, the serialization framework, `django.contrib.sitemaps`, `syndication`, `humanize`, `redirects`, `sites`, `django.contrib.postgres`, writing a custom system check.
- **5.8 Logging configuration:** The `LOGGING` dictionary, loggers, handlers, formatters and filters, the built-in `django`, `django.request`, and `django.db.backends` loggers, JSON logging, `structlog` overview, request ID propagation, `mail_admins` and its limits, never logging secrets or personal data, why `print` is not used in production.

---

## Phase 6: Security

- **6.1 Django's security model:** CSRF protection and the double-submit token mechanism, XSS and automatic escaping, `mark_safe` and `|safe` and their dangers, SQL injection and why the ORM protects you, clickjacking and `X-Frame-Options`, host header validation, what Django does not protect against.
- **6.2 Secure settings and headers:** `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SECURE_PROXY_SSL_HEADER` and its risk when misconfigured, `SECURE_CONTENT_TYPE_NOSNIFF`, Content Security Policy (Django 6.0, `django-csp` before), CORS with `django-cors-headers`, `manage.py check --deploy`, `SECRET_KEY_FALLBACKS` for key rotation.
- **6.3 Common vulnerabilities in Django code:** The OWASP Top 10 mapped to Django, insecure direct object references, mass assignment through `ModelForm` and serializers, open redirects and `url_has_allowed_host_and_scheme`, unsafe file upload and path traversal, raw SQL and `extra()`, server-side request forgery, unsafe deserialization, verbose error pages in production, user enumeration.
- **6.4 Secrets and dependency hygiene:** Environment variables and secret managers, rotating secrets, keeping secrets out of Git history, `pip-audit`, Dependabot and Renovate, the Django security release policy and applying patch releases promptly, pinning and lock files, reviewing third-party packages.

---

## Phase 7: Building APIs

- **7.1 JSON APIs with plain Django:** HTTP API design and REST principles, parsing JSON bodies, `JsonResponse`, status codes and error formats, writing an API view by hand, validating input with forms, authentication on API views, why this approach does not scale and what a framework adds.
- **7.2 Django REST Framework fundamentals:** Installation, serializers and `ModelSerializer`, `APIView`, generic views, `ViewSet` and routers, `Request` and `Response`, the browsable API, `@api_view`, how DRF extends Django views, the DRF request lifecycle.
- **7.3 Django REST Framework in depth:** Authentication classes, permission classes and custom permissions, throttling, pagination styles, filtering with `django-filter`, nested and writable serializers, serializer validation, `SerializerMethodField` and N+1 traps, exception handling, content negotiation, async support limits.
- **7.4 API documentation and schemas:** OpenAPI with `drf-spectacular`, schema annotations, generating client SDKs, API versioning strategies (URL, header), deprecation policy, contract testing overview, keeping documentation true to the code.
- **7.5 Django Ninja and API alternatives:** Pydantic-based request and response schemas, async-first design, type-hint-driven routing, comparing DRF, Django Ninja, and plain Django views, choosing based on team and project, migration costs.
- **7.6 API design patterns for production:** Idempotency keys, ETags and conditional requests, cursor vs offset pagination, bulk endpoints, partial updates with `PATCH`, webhooks and signature verification, problem-details error responses (RFC 9457), request validation limits, rate limiting, multi-tenant API scoping.

---

## Phase 8: Asynchronous Django and Real-Time Features

- **8.1 WSGI, ASGI, and how Django runs:** The WSGI and ASGI specifications, `wsgi.py` and `asgi.py`, Gunicorn, Uvicorn, Daphne, and Granian, how Django runs sync views under ASGI, thread-sensitive execution, when async actually helps (I/O-bound waits) and when it does not.
- **8.2 Async views and the async ORM:** `async def` views, async-capable QuerySet methods (`aget`, `acreate`, `afirst`, `acount`), `async for` over a `QuerySet`, `sync_to_async` and `async_to_sync`, `SynchronousOnlyOperation`, what remains synchronous, database connections in async code, async middleware, mixing sync and async safely.
- **8.3 WebSockets with Channels:** ASGI and the `channels` package, consumers, routing, channel layers with Redis, groups and broadcasting, authentication over WebSockets, scaling and deployment considerations, when to use WebSockets vs polling vs server-sent events.
- **8.4 Streaming responses and server-sent events:** `StreamingHttpResponse`, `FileResponse`, streaming large downloads and exports, async streaming responses, server-sent events for progress and notifications, buffering and proxy pitfalls, timeouts.
- **8.5 Outbound HTTP and concurrency inside views:** Calling other services with `httpx` (sync and async), timeouts, retries and backoff, circuit breaker idea, `asyncio.gather` and `TaskGroup` inside async views, protecting the request cycle from slow dependencies, deciding when to move the call to a background task.

---

## Phase 9: Background Work and Task Processing

- **9.1 Why background tasks:** Request time limits, work that does not belong in a request (email, reports, imports, webhooks), at-least-once delivery and idempotent tasks, `transaction.on_commit` for enqueueing safely, the transactional outbox pattern, task failure, retries, and dead-letter handling.
- **9.2 Django's built-in tasks framework (Django 6.0):** The `django.tasks` API, the `@task` decorator, `enqueue`, task results and statuses, the built-in development backends, the role of third-party backends for production, limits of the built-in interface, migrating to and from other task systems.
- **9.3 Celery:** Architecture (producer, broker, worker, result backend), Redis and RabbitMQ as brokers, configuring Celery with Django, defining and calling tasks, retries with backoff, `acks_late`, time limits, task routing and queues, periodic tasks with Celery beat, monitoring with Flower, common production failures.
- **9.4 Alternatives and task design patterns:** Huey, RQ, Dramatiq, and Procrastinate (PostgreSQL-backed), management commands with cron, passing IDs instead of objects, idempotency keys, task chains and groups, saga pattern overview, rate-limited external calls, testing tasks.

---

## Phase 10: Testing, Debugging, and Performance

- **10.1 Django's test framework:** `TestCase`, `TransactionTestCase`, `SimpleTestCase`, the test client and `RequestFactory`, assertions for responses, templates, redirects, and queries, the test database lifecycle, `override_settings`, `mail.outbox`, running tests and the test runner options.
- **10.2 `pytest-django` and test data:** `pytest-django` configuration, the `db` and `client` fixtures, `factory_boy` and `model_bakery`, `Faker`, parametrizing tests, `conftest.py` for shared fixtures, `pytest-xdist` for parallel runs, keeping test databases fast with `--reuse-db`.
- **10.3 Testing views, APIs, and integrations:** Testing views, forms, models, and managers, testing permissions and authentication, DRF `APIClient`, testing admin, mocking external services with `responses` or `respx`, controlling time with `time-machine`, testing background tasks and signals, testing migrations.
- **10.4 Coverage and continuous integration:** `coverage.py` and branch coverage, meaningful coverage vs vanity metrics, CI with a PostgreSQL service container, running `makemigrations --check`, `check --deploy`, `ruff`, and `mypy` together, test matrix across Python and Django versions.
- **10.5 Debugging:** `django-debug-toolbar`, `django-silk`, `shell_plus`, `pdb` and `breakpoint()` in a running server, logging SQL with `django.db.backends`, reproducing production bugs locally, debugging async code, debugging migration problems.
- **10.6 Profiling and load testing:** Counting queries, `EXPLAIN ANALYZE` on slow ORM queries, PostgreSQL `pg_stat_statements` and the slow query log, `py-spy` and `cProfile` for Python hot spots, load testing with Locust, interpreting latency percentiles, finding the real bottleneck before optimizing.
- **10.7 Optimization checklist:** Eliminating N+1, indexing, selecting fewer columns, `bulk_*` operations, pagination of large lists, caching layers, offloading to background tasks, template fragment caching, response compression, CDN for static assets, avoiding premature optimization.

---

## Phase 11: Production Ecosystem

- **11.1 PostgreSQL in depth for Django:** `psycopg` 3, `JSONB` queries, full-text search with `SearchVector` and `SearchRank`, trigram similarity, extensions, `AddIndexConcurrently`, advisory locks, table partitioning overview, PgBouncer and connection limits, backups and `pg_dump`.
- **11.2 Redis with Django:** Cache backend, session backend, Channels layer, Celery broker, `django-redis`, atomic counters for rate limiting, distributed locks and their limits, key naming and expiry, persistence settings and what to use Redis for and not.
- **11.3 Search:** PostgreSQL full-text search first, when to move to Elasticsearch, OpenSearch, or Meilisearch, keeping an external index in sync with the database, reindexing, eventual consistency, search-as-a-service overview.
- **11.4 File storage and CDNs:** `STORAGES` and `django-storages`, S3-compatible object storage, pre-signed URLs for direct uploads and private downloads, separating static and media hosting, CDN caching and invalidation, image processing and thumbnails, virus scanning overview.
- **11.5 Email and third-party integrations:** Transactional email providers with `django-anymail`, webhooks in and out, verifying signatures, payment provider integration patterns (Stripe as the model), retries and idempotency for outbound calls, wrapping third-party SDKs behind your own interface, failure isolation.
- **11.6 Multi-tenancy and data patterns:** Shared schema with a tenant column, schema-per-tenant with `django-tenants`, database-per-tenant, tenant scoping in managers and middleware, preventing cross-tenant leaks, audit logging and `django-simple-history`, soft deletes and data retention, personal data and deletion requests.
- **11.7 Observability:** Structured logs, Sentry for error tracking, OpenTelemetry tracing for Django and Celery, Prometheus metrics, liveness and readiness endpoints, correlation IDs across services, alerting on symptoms and not causes, dashboards that answer real questions.

---

## Phase 12: Deployment and Operations

- **12.1 Production settings and the deployment checklist:** Settings structure for production, `DEBUG = False` consequences, `ALLOWED_HOSTS`, secrets from the environment, `check --deploy` output, static and media in production, running migrations at deploy time, the twelve-factor app applied to Django.
- **12.2 Application servers and reverse proxies:** Gunicorn and Uvicorn workers, sizing workers and threads, Granian overview, Nginx or Caddy in front, TLS termination, proxy headers and `SECURE_PROXY_SSL_HEADER`, timeouts and buffering, serving static files through the proxy, graceful reloads.
- **12.3 Docker for Django:** A multi-stage `Dockerfile` with `uv`, running as a non-root user, `collectstatic` at build time, entrypoint scripts and migrations, health checks, image size and layer caching, separating web, worker, and beat containers, a production-like Compose file.
- **12.4 Deployment targets:** Platform-as-a-service options (Render, Fly.io, Railway, Heroku), a virtual machine with `systemd`, container services (AWS ECS, Google Cloud Run), Kubernetes overview, managed PostgreSQL and Redis, cost and operational trade-offs, choosing for a team size.
- **12.5 CI/CD and safe releases:** A GitHub Actions pipeline (test, lint, build image, push, deploy), backward-compatible migrations and the expand and contract pattern, adding columns, indexes, and constraints without long locks, zero-downtime deploys, rollbacks, blue-green and canary overview, feature flags.
- **12.6 Operations:** Database backups and tested restores, scaling web and worker processes independently, connection limits under load, monitoring and on-call basics, incident response and postmortems, handling a security release, cost control.

---

## Phase 13: Professional Practices

- **13.1 Architecture and where logic belongs:** Thin views, fat models vs service layers, services and selectors, keeping the ORM out of templates, domain logic that does not import Django, `transaction.atomic` placement, avoiding circular imports between apps, a decision guide for placing code.
- **13.2 Design principles and patterns in Django:** SOLID applied to Django, dependency injection without a framework, the ORM as an implementation of Repository and Unit of Work, when an extra repository layer is waste, strategy and observer in Django terms, the outbox pattern, event-driven designs, CQRS overview, the modular monolith.
- **13.3 Reusable apps and packaging:** Writing a pluggable app, `AppConfig` and app settings with defaults, packaging with `pyproject.toml`, shipping templates and migrations, testing across a Django and Python version matrix with `nox` or `tox`, publishing to PyPI.
- **13.4 Clean code with Django conventions:** Naming, query placement, settings access through `django.conf.settings`, string references for relations, avoiding business logic in `save` overrides and signals, keeping templates dumb, consistent error handling, review checklist for Django pull requests.
- **13.5 Documentation and collaboration:** Docstrings, README structure for a Django project, architecture decision records, OpenAPI as documentation, onboarding guides, pull requests and branching workflows, commit message conventions, changelogs, contributing to Django itself overview.
- **13.6 Upgrading and long-term maintenance:** The release and deprecation timeline, moving from one LTS to the next, running with `-W error::DeprecationWarning`, `django-upgrade` and `pyupgrade`, third-party package compatibility, dependency update automation, keeping a project upgradable.

---

## Phase 14: Capstone Projects (Django Plus a Fixed Set of Packages)

Allowed packages: Django, Django REST Framework, `drf-spectacular`, `django-filter`, `psycopg`, `django-environ`, `pytest-django`, `factory_boy`, `Celery`, `redis`, `gunicorn`, `whitenoise`, `ruff`, and `mypy` with `django-stubs`. Other web frameworks and low-code admin replacements are not allowed.

1. **Server-rendered publishing platform:** Custom user model, registration and login, role-based permissions, models with relationships, forms and formsets, templates with inheritance, admin customization, image uploads, full-text search on PostgreSQL, query counts asserted in tests, `check --deploy` passing.
2. **Booking and inventory REST API:** DRF with serializers and viewsets, token or session authentication, object-level permissions, filtering and cursor pagination, OpenAPI schema, transaction-safe booking with `select_for_update`, idempotency keys, rate limiting, a `pytest` suite with factories, and a CI pipeline.
3. **Order processing backend:** Orders, payments through a mock provider with signed webhooks, background tasks with Celery or the Django tasks framework, the outbox pattern, expand-and-contract migrations, structured logging and error tracking, a multi-stage Docker image, Docker Compose for the full stack, and a deployment to a real host with a written runbook.