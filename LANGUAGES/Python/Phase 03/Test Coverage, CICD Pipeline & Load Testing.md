```
Phase 6.1: pytest fundamentals
Phase 6.2: mocking and unit tests
Phase 6.3: integration tests
Phase 6.4: the professional finishing touches

Three topics this phase:

1. TEST COVERAGE
   "Which lines of your code are NOT tested?"
   Find the gaps before production finds them.

2. CI/CD PIPELINE
   "Run tests automatically on every code change"
   Never merge broken code.
   Never deploy untested code.

3. LOAD TESTING
   "How does your API behave under pressure?"
   100 users? 1000? 10,000?
   Find the breaking point before your users do.
```

---

## Setup

Bash

```
cd ~/projects/testing_mastery
pip install pytest-cov coverage locust

# For CI/CD examples we'll use GitHub Actions (no install needed)
# The YAML files go in your repository

mkdir -p .github/workflows
touch .github/workflows/ci.yml
touch locustfile.py
```

---

## 1. 📊 Test Coverage — Measuring What's Tested

### What is Coverage?

text

```
Coverage = percentage of your code executed by tests.

Line coverage:     Is this line executed by any test?
Branch coverage:   Is every if/else branch taken?
Function coverage: Is this function called by any test?

Example:

def process(value):
    if value > 0:          ← branch 1
        return "positive"  ← line
    elif value < 0:        ← branch 2
        return "negative"  ← line
    else:                  ← branch 3
        return "zero"      ← line  ← NOT covered!

If your tests only call process(1) and process(-1):
  Line coverage:   75% (3 of 4 lines)
  Branch coverage: 67% (2 of 3 branches)

The zero branch has a potential bug nobody will catch.
```

Python

```
# src/billing.py — code with various coverage scenarios
from typing import Optional
from datetime import datetime, timezone
from decimal import Decimal


class BillingError(Exception):
    pass


class Invoice:
    """Invoice with multiple code paths to test."""

    VALID_CURRENCIES = {"USD", "EUR", "GBP", "JPY"}
    TAX_RATES = {
        "US": Decimal("0.08"),
        "GB": Decimal("0.20"),
        "DE": Decimal("0.19"),
    }

    def __init__(
        self,
        amount: Decimal,
        currency: str = "USD",
        country: str = "US",
    ):
        if amount <= 0:
            raise BillingError("Amount must be positive")
        if currency not in self.VALID_CURRENCIES:
            raise BillingError(f"Invalid currency: {currency}")

        self.amount = amount
        self.currency = currency
        self.country = country
        self.discount = Decimal("0")
        self.items = []
        self.created_at = datetime.now(timezone.utc)
        self.paid_at: Optional[datetime] = None

    def add_item(self, name: str, price: Decimal, quantity: int = 1) -> None:
        if quantity <= 0:
            raise BillingError("Quantity must be positive")
        self.items.append({
            "name": name,
            "price": price,
            "quantity": quantity,
            "total": price * quantity,
        })

    def apply_discount(self, percent: Decimal) -> None:
        if percent < 0 or percent > 100:
            raise BillingError("Discount must be 0-100%")
        self.discount = percent

    def calculate_tax(self) -> Decimal:
        """Calculate tax based on country."""
        rate = self.TAX_RATES.get(self.country, Decimal("0"))
        return self.amount * rate

    def subtotal(self) -> Decimal:
        """Subtotal before tax."""
        if self.items:
            return sum(item["total"] for item in self.items)
        return self.amount

    def total(self) -> Decimal:
        """Total after discount and tax."""
        sub = self.subtotal()
        discounted = sub * (1 - self.discount / 100)
        tax = discounted * self.TAX_RATES.get(self.country, Decimal("0"))
        return discounted + tax

    def mark_paid(self) -> None:
        """Mark invoice as paid."""
        if self.paid_at is not None:
            raise BillingError("Invoice already paid")
        self.paid_at = datetime.now(timezone.utc)

    @property
    def is_paid(self) -> bool:
        return self.paid_at is not None

    @property
    def status(self) -> str:
        if self.is_paid:
            return "paid"
        if self.total() == 0:
            return "zero_amount"    # ← hard to test, often missed!
        return "pending"

    def to_dict(self) -> dict:
        return {
            "amount": float(self.amount),
            "currency": self.currency,
            "country": self.country,
            "discount": float(self.discount),
            "subtotal": float(self.subtotal()),
            "tax": float(self.calculate_tax()),
            "total": float(self.total()),
            "items": self.items,
            "status": self.status,
            "is_paid": self.is_paid,
            "created_at": self.created_at.isoformat(),
        }
```

### Running Coverage

Bash

```
# Basic coverage run
pytest --cov=src tests/

# Show which lines are missing
pytest --cov=src --cov-report=term-missing tests/

# HTML report (best for visual inspection)
pytest --cov=src --cov-report=html tests/
open htmlcov/index.html    # Mac
# xdg-open htmlcov/index.html  # Linux

# XML report (for CI tools like SonarQube)
pytest --cov=src --cov-report=xml tests/

# Multiple formats at once
pytest --cov=src \
    --cov-report=term-missing \
    --cov-report=html \
    --cov-report=xml \
    tests/

# Fail if coverage below threshold
pytest --cov=src --cov-fail-under=80 tests/

# Run with branch coverage (stricter)
pytest --cov=src --cov-branch tests/
```

### Coverage Configuration

ini

```
# .coveragerc — coverage configuration
[run]
source = src
branch = True                # enable branch coverage
omit =
    src/__init__.py
    src/**/migrations/*
    src/**/alembic/*
    tests/*
    *conftest*
    src/main.py              # entry point, not business logic

[report]
fail_under = 80              # fail if below 80%
show_missing = True          # show uncovered line numbers
exclude_lines =
    # Standard excludes
    pragma: no cover
    def __repr__
    def __str__
    raise NotImplementedError
    if TYPE_CHECKING:
    @overload
    pass
    \.\.\.

    # Abstract methods
    @abstractmethod

    # Debug code
    if settings.DEBUG:
    if __name__ == .__main__.:

[html]
directory = htmlcov
title = My Project Coverage

[xml]
output = coverage.xml
```

Python

```
# pytest.ini — integrate coverage settings
# [pytest]
# addopts =
#     --cov=src
#     --cov-report=term-missing
#     --cov-fail-under=80
#     --cov-branch
```

### Writing Tests for Coverage

Python

```
# tests/unit/test_billing_coverage.py
"""
Writing tests specifically to improve coverage.
Process: run coverage → see gaps → write tests for gaps.
"""
import pytest
from decimal import Decimal
from src.billing import Invoice, BillingError


class TestInvoiceBasics:

    def test_create_invoice(self):
        inv = Invoice(Decimal("100.00"))
        assert inv.amount == Decimal("100.00")
        assert inv.currency == "USD"
        assert inv.country == "US"
        assert inv.discount == Decimal("0")

    def test_create_with_all_params(self):
        inv = Invoice(Decimal("200"), currency="EUR", country="DE")
        assert inv.currency == "EUR"
        assert inv.country == "DE"

    def test_invalid_amount(self):
        with pytest.raises(BillingError, match="Amount must be positive"):
            Invoice(Decimal("0"))

    def test_negative_amount(self):
        with pytest.raises(BillingError, match="Amount must be positive"):
            Invoice(Decimal("-50"))

    def test_invalid_currency(self):
        with pytest.raises(BillingError, match="Invalid currency"):
            Invoice(Decimal("100"), currency="XYZ")

    @pytest.mark.parametrize("currency", ["USD", "EUR", "GBP", "JPY"])
    def test_all_valid_currencies(self, currency):
        inv = Invoice(Decimal("100"), currency=currency)
        assert inv.currency == currency


class TestInvoiceItems:

    @pytest.fixture
    def invoice(self):
        return Invoice(Decimal("100"))

    def test_add_item(self, invoice):
        invoice.add_item("Widget", Decimal("10.00"), 2)
        assert len(invoice.items) == 1
        assert invoice.items[0]["total"] == Decimal("20.00")

    def test_add_multiple_items(self, invoice):
        invoice.add_item("Widget A", Decimal("10.00"), 2)
        invoice.add_item("Widget B", Decimal("5.00"), 3)
        assert len(invoice.items) == 2

    def test_add_item_invalid_quantity(self, invoice):
        with pytest.raises(BillingError, match="Quantity must be positive"):
            invoice.add_item("Widget", Decimal("10"), 0)

    def test_add_item_negative_quantity(self, invoice):
        with pytest.raises(BillingError):
            invoice.add_item("Widget", Decimal("10"), -1)

    def test_subtotal_with_items(self, invoice):
        invoice.add_item("A", Decimal("30"), 1)
        invoice.add_item("B", Decimal("20"), 2)
        assert invoice.subtotal() == Decimal("70")

    def test_subtotal_without_items_uses_amount(self, invoice):
        """Without items, subtotal = amount."""
        assert invoice.subtotal() == Decimal("100")


class TestDiscountAndTax:

    @pytest.fixture
    def invoice(self):
        return Invoice(Decimal("100"))

    @pytest.mark.parametrize("percent, expected_total", [
        (Decimal("0"),   Decimal("108.00")),   # no discount, 8% US tax
        (Decimal("10"),  Decimal("97.20")),    # 10% off, then tax
        (Decimal("50"),  Decimal("54.00")),    # 50% off, then tax
        (Decimal("100"), Decimal("0.00")),     # fully discounted
    ])
    def test_discount_with_tax(self, invoice, percent, expected_total):
        invoice.apply_discount(percent)
        assert invoice.total() == pytest.approx(expected_total, abs=Decimal("0.01"))

    def test_invalid_discount_negative(self, invoice):
        with pytest.raises(BillingError, match="0-100"):
            invoice.apply_discount(Decimal("-1"))

    def test_invalid_discount_over_100(self, invoice):
        with pytest.raises(BillingError, match="0-100"):
            invoice.apply_discount(Decimal("101"))

    @pytest.mark.parametrize("country, expected_rate", [
        ("US", Decimal("0.08")),
        ("GB", Decimal("0.20")),
        ("DE", Decimal("0.19")),
        ("XX", Decimal("0")),    # unknown country = no tax
    ])
    def test_tax_by_country(self, country, expected_rate):
        inv = Invoice(Decimal("100"), country=country)
        expected_tax = Decimal("100") * expected_rate
        assert inv.calculate_tax() == pytest.approx(expected_tax)


class TestPaymentStatus:

    def test_initial_status_pending(self):
        inv = Invoice(Decimal("100"))
        assert inv.status == "pending"
        assert not inv.is_paid

    def test_mark_paid(self):
        inv = Invoice(Decimal("100"))
        inv.mark_paid()
        assert inv.is_paid
        assert inv.status == "paid"
        assert inv.paid_at is not None

    def test_mark_paid_twice_fails(self):
        inv = Invoice(Decimal("100"))
        inv.mark_paid()
        with pytest.raises(BillingError, match="already paid"):
            inv.mark_paid()

    def test_zero_amount_status(self):
        """
        This is the 'zero_amount' branch that's often missed!
        Coverage helps us find this.
        """
        inv = Invoice(Decimal("100"))
        inv.apply_discount(Decimal("100"))  # 100% discount → total = 0
        assert inv.status == "zero_amount"

    def test_to_dict_structure(self):
        inv = Invoice(Decimal("100"))
        d = inv.to_dict()

        required_keys = [
            "amount", "currency", "country", "discount",
            "subtotal", "tax", "total", "items",
            "status", "is_paid", "created_at"
        ]
        for key in required_keys:
            assert key in d, f"Missing key: {key}"


class TestCoverageEdgeCases:
    """Tests specifically for hard-to-reach code paths."""

    def test_jpn_currency(self):
        """JPY is a valid currency — cover this branch."""
        inv = Invoice(Decimal("1000"), currency="JPY")
        assert inv.currency == "JPY"

    def test_gbp_with_gb_tax(self):
        """Test UK invoices with GBP currency and GB tax."""
        inv = Invoice(Decimal("100"), currency="GBP", country="GB")
        # 20% UK tax
        assert inv.calculate_tax() == Decimal("20.00")

    def test_items_total_vs_amount(self):
        """Items override the base amount for subtotal."""
        inv = Invoice(Decimal("999"))   # base amount
        inv.add_item("Widget", Decimal("10"), 3)  # 3 × £10 = £30
        # Subtotal uses items, not base amount
        assert inv.subtotal() == Decimal("30")

    def test_to_dict_with_paid_invoice(self):
        inv = Invoice(Decimal("100"))
        inv.mark_paid()
        d = inv.to_dict()
        assert d["is_paid"] is True
        assert d["status"] == "paid"
```

### Coverage Report Interpretation

Python

```
"""
HOW TO READ THE COVERAGE REPORT:

Terminal output:
Name                 Stmts   Miss  Cover   Missing
--------------------------------------------------
src/billing.py          78      8    90%   45, 67-70, 89, 102

Stmts  = total lines of code
Miss   = lines not executed by tests
Cover  = percentage covered
Missing= line numbers not covered

Line 45:    one specific line never executed
Lines 67-70: a block of 4 lines never executed
Line 89:    another single line
Line 102:   another single line

ACTION: Look at these lines in your code.
        Write tests that exercise those code paths.

COMMON GAPS:
  - Error handling code (exception handlers)
  - Edge cases (empty list, None values, 0)
  - "else" branches in conditionals
  - Timeout/retry fallbacks
  - Admin-only functionality

COVERAGE TARGETS (industry standard):
  < 60%:  Risky. Many untested paths.
  60-80%: Acceptable for internal tools.
  80-90%: Good for most projects.
  90-95%: Excellent. Production ready.
  > 95%:  Exceptional. Some overhead.

DON'T: Write tests just to increase the number.
DO:    Write tests for real behavior and business logic.
       Coverage is a TOOL, not a GOAL.
"""
```

---

## 2. 🔄 CI/CD Pipeline — GitHub Actions

### What is CI/CD?

text

```
CI = Continuous Integration
     Every time you push code → tests run automatically
     If tests fail → you know immediately (not in production)

CD = Continuous Deployment
     If tests pass → automatically deploy to staging/production

The pipeline:
  Push code → GitHub
      ↓
  GitHub Actions starts a workflow
      ↓
  Install dependencies
      ↓
  Run linting (ruff, black, mypy)
      ↓
  Run unit tests
      ↓
  Run integration tests
      ↓
  Check coverage threshold
      ↓
  Build Docker image (if all pass)
      ↓
  Deploy to staging (if all pass)
      ↓
  Deploy to production (manual approval)
```

### Basic CI Pipeline

YAML

```
# .github/workflows/ci.yml
name: CI Pipeline

# When to run this workflow
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

# Cancel in-progress runs for the same branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ─────────────────────────────────────
  # JOB 1: Code Quality Checks
  # ─────────────────────────────────────
  quality:
    name: Code Quality
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"              # cache pip dependencies

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install ruff black mypy

      - name: Lint with Ruff
        run: ruff check src/ tests/

      - name: Format check with Black
        run: black --check src/ tests/

      - name: Type check with mypy
        run: mypy src/ --ignore-missing-imports
        continue-on-error: true     # don't fail pipeline for type errors yet

  # ─────────────────────────────────────
  # JOB 2: Unit Tests
  # ─────────────────────────────────────
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: quality          # run only if quality checks pass

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-mock faker factory-boy

      - name: Run unit tests with coverage
        run: |
          pytest tests/unit/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -v \
            --tb=short

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: unit-coverage
          path: coverage.xml
          retention-days: 7

      - name: Coverage comment on PR
        if: github.event_name == 'pull_request'
        uses: MishaKav/pytest-coverage-comment@main
        with:
          pytest-xml-coverage-path: ./coverage.xml

  # ─────────────────────────────────────
  # JOB 3: Integration Tests
  # ─────────────────────────────────────
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: quality

    # Start PostgreSQL as a service container
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        # Health check to wait for PostgreSQL to be ready
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379/0
      SECRET_KEY: test-secret-key-for-ci-only-32-chars!!
      JWT_SECRET_KEY: test-jwt-secret-for-ci-32-chars!!
      ENVIRONMENT: testing

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run database migrations
        run: alembic upgrade head

      - name: Run integration tests
        run: |
          pytest tests/integration/ \
            --cov=src \
            --cov-report=xml \
            -v \
            --tb=short \
            -m "integration"

      - name: Upload integration coverage
        uses: actions/upload-artifact@v4
        with:
          name: integration-coverage
          path: coverage.xml

  # ─────────────────────────────────────
  # JOB 4: Build Docker Image
  # ─────────────────────────────────────
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    if: github.ref == 'refs/heads/main'    # only on main branch

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha     # GitHub Actions cache
          cache-to: type=gha,mode=max

  # ─────────────────────────────────────
  # JOB 5: Deploy to Staging
  # ─────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging       # requires approval in GitHub settings

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          echo "Deploying to staging..."
          # Real: SSH to server, pull new image, restart containers
          # Or: Update ECS task definition, trigger deployment

      - name: Run smoke tests on staging
        run: |
          # Quick sanity check that staging is up
          curl -f https://staging.myapp.com/health || exit 1

      - name: Notify deployment
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: "Staging deployment ${{ job.status }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Advanced CI Patterns

YAML

```
# .github/workflows/advanced_ci.yml
name: Advanced CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # ─────────────────────────────────────
  # MATRIX TESTING — test on multiple Python versions
  # ─────────────────────────────────────
  test-matrix:
    name: Test Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
        # Optional: test on multiple OS
        # os: [ubuntu-latest, windows-latest, macos-latest]
      fail-fast: false   # don't cancel other matrix jobs on failure

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"

      - name: Install
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Test
        run: pytest tests/unit/ -v

  # ─────────────────────────────────────
  # DEPENDENCY CACHING — speed up workflows
  # ─────────────────────────────────────
  cached-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      # Cache pip dependencies based on requirements hash
      - name: Cache dependencies
        uses: actions/cache@v4
        id: cache
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install (uses cache if available)
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/unit/ -q

  # ─────────────────────────────────────
  # SECURITY SCANNING
  # ─────────────────────────────────────
  security:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install security tools
        run: |
          pip install bandit safety pip-audit

      - name: Bandit security scan (Python code)
        run: bandit -r src/ -f json -o bandit-report.json || true

      - name: Safety check (known vulnerabilities in deps)
        run: safety check --full-report

      - name: pip-audit (audit installed packages)
        run: pip-audit

      - name: Upload security report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: security-report
          path: bandit-report.json

  # ─────────────────────────────────────
  # COVERAGE REPORT — track over time
  # ─────────────────────────────────────
  coverage:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install
        run: pip install -r requirements.txt pytest pytest-cov

      - name: Run tests with coverage
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=lcov

      # Send to Codecov (free coverage tracking service)
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage.xml
          flags: unittests
          fail_ci_if_error: false

      # Send to Coveralls (alternative to Codecov)
      # - name: Upload to Coveralls
      #   uses: coverallsapp/github-action@v2
```

### PR Status Checks

YAML

```
# .github/workflows/pr_checks.yml
name: PR Checks

on:
  pull_request:

jobs:
  # Run fast checks first, expensive ones after
  lint:
    name: "1. Lint & Format"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"
      - run: pip install ruff black
      - run: ruff check src/ tests/
      - run: black --check src/ tests/

  unit:
    name: "2. Unit Tests"
    runs-on: ubuntu-latest
    needs: lint                # only if lint passes
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest tests/unit/ --cov=src --cov-fail-under=80 -q

  integration:
    name: "3. Integration Tests"
    runs-on: ubuntu-latest
    needs: unit
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 5s --health-retries 5

    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/test

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: alembic upgrade head
      - run: pytest tests/integration/ -q

  # PR check summary
  pr-status:
    name: "All Checks Passed"
    runs-on: ubuntu-latest
    needs: [lint, unit, integration]
    if: always()
    steps:
      - name: Check all jobs passed
        run: |
          if [[ "${{ needs.lint.result }}" == "success" ]] && \
             [[ "${{ needs.unit.result }}" == "success" ]] && \
             [[ "${{ needs.integration.result }}" == "success" ]]; then
            echo "All checks passed! ✅"
            exit 0
          else
            echo "Some checks failed! ❌"
            exit 1
```

### pre-commit Hooks

YAML

```
# .pre-commit-config.yaml
# Install: pip install pre-commit
# Setup:   pre-commit install
# Run:     pre-commit run --all-files

repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-merge-conflict
      - id: debug-statements          # catch debug print/breakpoint
      - id: check-added-large-files
        args: [--maxkb=500]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
        args: [--ignore-missing-imports]

  - repo: local
    hooks:
      - id: pytest-unit
        name: pytest (unit tests only)
        entry: pytest tests/unit/ -q --tb=short
        language: system
        pass_filenames: false
        always_run: true
        stages: [pre-push]    # only on push, not on commit
```

---

## 3. 🔫 Load Testing with Locust

### What is Load Testing?

text

```
Load testing answers:
  "How does my API perform under X concurrent users?"
  "What's my API's maximum throughput?"
  "Where does my API break?"
  "How does response time change as load increases?"

Types of load tests:
  Load test:        Normal expected traffic → verify performance
  Stress test:      Push beyond normal → find breaking point
  Spike test:       Sudden traffic surge → recovery behavior
  Soak test:        Low but sustained → memory leaks, degradation

Key metrics:
  RPS:              Requests per second (throughput)
  Response time:    P50, P95, P99 (median, 95th, 99th percentile)
  Error rate:       % of requests that fail
  CPU/Memory:       Server resource usage

Industry targets:
  API response:     P95 < 200ms, P99 < 500ms
  Error rate:       < 0.1% in normal conditions
  Throughput:       Depends on product (100-10,000+ RPS)
```

### Locust Basics

Python

```
# locustfile.py
"""
Locust load testing.

Run:
  locust -f locustfile.py --host=http://localhost:8000

Then open: http://localhost:8089
  - Set number of users and spawn rate
  - Click Start
  - Watch real-time stats

Headless mode (for CI):
  locust -f locustfile.py \
    --host=http://localhost:8000 \
    --users=100 \
    --spawn-rate=10 \
    --run-time=1m \
    --headless \
    --csv=results
"""
from locust import HttpUser, task, between, constant, events
from locust import LoadTestShape
import random
import json
import logging

logger = logging.getLogger(__name__)


# ─────────────────────────────────────────
# BASIC USER — simulates one real user
# ─────────────────────────────────────────
class BasicAPIUser(HttpUser):
    """
    Simulates a typical API user.
    Each Locust user runs these tasks in random weighted order.
    """
    # Wait between 1-5 seconds between requests
    # (simulates real user behavior, not hammering)
    wait_time = between(1, 5)

    # Shared state across all users (class-level)
    token = None

    def on_start(self):
        """
        Runs when user starts.
        Login and store the token.
        """
        self.register_and_login()

    def register_and_login(self):
        """Register a unique user and get auth token."""
        import uuid
        unique_id = str(uuid.uuid4())[:8]

        # Register
        response = self.client.post(
            "/auth/register",
            json={
                "name": f"Load Test User {unique_id}",
                "email": f"loadtest_{unique_id}@test.com",
                "password": "LoadTestPass1!"
            }
        )

        if response.status_code == 201:
            self.token = response.json().get("access_token")
        else:
            logger.error(f"Registration failed: {response.status_code}")

    def auth_headers(self) -> dict:
        """Return authorization headers."""
        if self.token:
            return {"Authorization": f"Bearer {self.token}"}
        return {}

    # ─── Tasks ───────────────────────────────────────────────
    # @task(N) — weight: higher N = more likely to be called

    @task(5)    # most common: browse posts
    def browse_posts(self):
        """GET /posts — most frequent action."""
        page = random.randint(1, 5)
        self.client.get(
            f"/posts?page={page}&page_size=20",
            name="/posts (list)"   # group similar URLs in stats
        )

    @task(3)    # less common: view specific post
    def view_post(self):
        """GET /posts/{id} — view a specific post."""
        post_id = random.randint(1, 100)
        with self.client.get(
            f"/posts/{post_id}",
            name="/posts/{id} (detail)",
            catch_response=True   # handle expected errors
        ) as response:
            if response.status_code == 404:
                response.success()  # 404 is OK — post might not exist

    @task(2)    # occasional: create content
    def create_post(self):
        """POST /posts — create a new post."""
        if not self.token:
            return

        response = self.client.post(
            "/posts",
            json={
                "title": f"Load Test Post {random.randint(1, 9999)}",
                "content": "This is load test content. " * 10,
                "status": "published"
            },
            headers=self.auth_headers()
        )

        if response.status_code == 201:
            # Track created post IDs for subsequent tests
            pass

    @task(1)    # rare: check profile
    def check_profile(self):
        """GET /auth/me — check own profile."""
        if not self.token:
            return
        self.client.get("/auth/me", headers=self.auth_headers())

    @task(1)
    def health_check(self):
        """GET /health — quick health check."""
        self.client.get("/health")


# ─────────────────────────────────────────
# REALISTIC USER JOURNEYS
# ─────────────────────────────────────────
class BlogReaderUser(HttpUser):
    """Simulates a user who only reads content."""
    wait_time = between(2, 8)    # readers are slower

    @task(10)
    def browse_home(self):
        self.client.get("/posts?page=1&status=published", name="/posts (home)")

    @task(5)
    def read_post(self):
        post_id = random.randint(1, 50)
        with self.client.get(
            f"/posts/{post_id}",
            name="/posts/{id}",
            catch_response=True
        ) as r:
            if r.status_code == 404:
                r.success()

    @task(2)
    def search_posts(self):
        terms = ["python", "fastapi", "django", "api", "database"]
        self.client.get(
            f"/posts?q={random.choice(terms)}",
            name="/posts (search)"
        )


class BlogWriterUser(HttpUser):
    """Simulates an authenticated content creator."""
    wait_time = between(5, 15)   # writers think before acting

    def on_start(self):
        self._login()

    def _login(self):
        import uuid
        uid = str(uuid.uuid4())[:8]
        r = self.client.post("/auth/register", json={
            "name": f"Writer {uid}",
            "email": f"writer_{uid}@test.com",
            "password": "WriterPass1!"
        })
        self.token = r.json().get("access_token") if r.status_code == 201 else None

    def _auth(self):
        return {"Authorization": f"Bearer {self.token}"} if self.token else {}

    @task(3)
    def create_draft(self):
        self.client.post("/posts", json={
            "title": f"Draft Post {random.randint(1, 9999)}",
            "content": "Draft content " * 20,
            "status": "draft"
        }, headers=self._auth())

    @task(1)
    def publish_content(self):
        self.client.post("/posts", json={
            "title": f"Published {random.randint(1, 9999)}",
            "content": "Published content " * 20,
            "status": "published"
        }, headers=self._auth())


# ─────────────────────────────────────────
# LOAD TEST SHAPES — control traffic patterns
# ─────────────────────────────────────────
class StepLoadShape(LoadTestShape):
    """
    Step load: increase users in steps.
    Shows how performance degrades at each step.

    Users:  10 → 25 → 50 → 75 → 100 → 75 → 50 → 25
    Time:    0   1m   2m   3m    4m   5m   6m   7m
    """

    step_time = 60      # seconds per step
    step_load = 25      # users per step
    spawn_rate = 10     # users added per second
    max_users = 100     # peak load

    def tick(self):
        run_time = self.get_run_time()

        # Increase phase
        if run_time < self.step_time * 4:
            current_step = run_time // self.step_time
            user_count = self.step_load * (current_step + 1)
            return min(user_count, self.max_users), self.spawn_rate

        # Decrease phase (gradual scale down)
        elif run_time < self.step_time * 7:
            decrease_step = (run_time - self.step_time * 4) // self.step_time
            user_count = self.max_users - (self.step_load * decrease_step)
            return max(user_count, self.step_load), self.spawn_rate

        # Done
        return None


class SpikeTestShape(LoadTestShape):
    """
    Spike test: sudden huge traffic burst.
    Tests system recovery after overload.

    Users: 10 ─── 200 ─── 10
    Time:   0     30s    2m
    """

    def tick(self):
        run_time = self.get_run_time()

        if run_time < 30:
            return 10, 2           # baseline: 10 users

        elif run_time < 60:
            return 200, 50         # spike! 200 users very fast

        elif run_time < 120:
            return 10, 5           # back to normal, gradually

        return None                # stop test


class SoakTestShape(LoadTestShape):
    """
    Soak test: constant load for a long time.
    Detects memory leaks and gradual degradation.
    """

    def tick(self):
        run_time = self.get_run_time()
        soak_duration = 60 * 30   # 30 minutes

        if run_time < 30:
            return 50, 5           # ramp up
        elif run_time < soak_duration:
            return 50, 5           # sustained load
        elif run_time < soak_duration + 30:
            return 0, 5            # ramp down
        return None


# ─────────────────────────────────────────
# CUSTOM EVENTS — track business metrics
# ─────────────────────────────────────────
from locust import events as locust_events

@locust_events.init.add_listener
def on_locust_init(environment, **kwargs):
    """Called when Locust starts."""
    print("Load test starting...")
    print(f"Target: {environment.host}")


@locust_events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    """Called when load test ends."""
    stats = environment.stats
    print("\n=== Load Test Results ===")
    print(f"Total requests: {stats.total.num_requests:,}")
    print(f"Failures: {stats.total.num_failures:,}")
    print(f"Error rate: {stats.total.fail_ratio * 100:.1f}%")
    print(f"Avg response: {stats.total.avg_response_time:.0f}ms")
    print(f"P95 response: {stats.total.get_response_time_percentile(0.95):.0f}ms")
    print(f"RPS: {stats.total.current_rps:.1f}")
```

### Load Test Configuration

Python

```
# locust.conf — locust configuration
[DEFAULT]
# host = http://localhost:8000
headless = false
users = 50
spawn-rate = 5
run-time = 5m

# HTML report output
html = load-test-report.html

# CSV output for analysis
csv = load-test-results

# Log level
loglevel = WARNING

# Stop test if error rate exceeds 10%
# stop-timeout = 30
```

Bash

```
# ─────────────────────────────────────────
# RUNNING LOAD TESTS
# ─────────────────────────────────────────

# Start your API first:
uvicorn src.main:app --host 0.0.0.0 --port 8000

# Interactive mode (browser UI):
locust -f locustfile.py --host=http://localhost:8000

# Headless mode (for CI/automated):
locust -f locustfile.py \
    --host=http://localhost:8000 \
    --users=100 \
    --spawn-rate=10 \
    --run-time=2m \
    --headless \
    --csv=results \
    --html=report.html

# Specific user class:
locust -f locustfile.py \
    --host=http://localhost:8000 \
    --users=50 \
    --spawn-rate=5 \
    --run-time=1m \
    --headless \
    BlogReaderUser    # only use this user class

# Custom shape:
locust -f locustfile.py \
    --host=http://localhost:8000 \
    --headless \
    StepLoadShape

# Distributed (multiple machines):
# Master node:
locust -f locustfile.py --master
# Worker nodes (on other machines):
locust -f locustfile.py --worker --master-host=192.168.1.100
```

### Analyzing Load Test Results

Python

```
# analyze_results.py — analyze CSV output from Locust
import csv
import json
from pathlib import Path


def analyze_load_test(csv_prefix: str = "results") -> dict:
    """
    Analyze Locust CSV output and produce a summary.
    """
    stats_file = Path(f"{csv_prefix}_stats.csv")
    failures_file = Path(f"{csv_prefix}_failures.csv")

    if not stats_file.exists():
        print(f"No stats file found: {stats_file}")
        return {}

    # Read stats
    with open(stats_file) as f:
        reader = csv.DictReader(f)
        stats = list(reader)

    # Find totals row
    totals = next((s for s in stats if s["Name"] == "Aggregated"), None)

    if not totals:
        return {}

    summary = {
        "total_requests": int(totals["Request Count"]),
        "total_failures": int(totals["Failure Count"]),
        "error_rate_pct": float(totals["Failure Count"]) /
                          float(totals["Request Count"]) * 100
                          if int(totals["Request Count"]) > 0 else 0,
        "avg_response_ms": float(totals["Average Response Time"]),
        "min_response_ms": float(totals["Min Response Time"]),
        "max_response_ms": float(totals["Max Response Time"]),
        "p50_ms": float(totals["50%"]),
        "p95_ms": float(totals["95%"]),
        "p99_ms": float(totals["99%"]),
        "avg_rps": float(totals["Requests/s"]),
    }

    # Read failures
    if failures_file.exists():
        with open(failures_file) as f:
            reader = csv.DictReader(f)
            failures = list(reader)
        summary["failure_types"] = [
            {
                "method": f["Method"],
                "name": f["Name"],
                "error": f["Error"],
                "occurrences": int(f["Occurrences"])
            }
            for f in failures
        ]

    # Per-endpoint stats
    summary["endpoints"] = [
        {
            "name": s["Name"],
            "requests": int(s["Request Count"]),
            "avg_ms": float(s["Average Response Time"]),
            "p95_ms": float(s["95%"]),
            "failures": int(s["Failure Count"]),
        }
        for s in stats
        if s["Name"] != "Aggregated"
    ]

    return summary


def check_thresholds(summary: dict) -> list:
    """Check if results meet performance thresholds."""
    violations = []

    if summary.get("error_rate_pct", 0) > 1.0:
        violations.append(
            f"Error rate {summary['error_rate_pct']:.1f}% exceeds 1% threshold"
        )

    if summary.get("p95_ms", 0) > 500:
        violations.append(
            f"P95 response time {summary['p95_ms']:.0f}ms exceeds 500ms threshold"
        )

    if summary.get("p99_ms", 0) > 2000:
        violations.append(
            f"P99 response time {summary['p99_ms']:.0f}ms exceeds 2000ms threshold"
        )

    return violations


if __name__ == "__main__":
    summary = analyze_load_test("results")
    print(json.dumps(summary, indent=2))

    violations = check_thresholds(summary)
    if violations:
        print("\n❌ Performance threshold violations:")
        for v in violations:
            print(f"  - {v}")
        exit(1)
    else:
        print("\n✅ All performance thresholds met!")
```

---

## 4. 🔄 Complete CI/CD with Load Testing

YAML

```
# .github/workflows/full_pipeline.yml
name: Full Pipeline

on:
  push:
    branches: [main]

jobs:
  test:
    name: Tests & Coverage
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest tests/ --cov=src --cov-fail-under=80 -q

  build:
    name: Build & Push
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - name: Deploy
        run: |
          # Update deployment with new image
          echo "Deploying ${{ github.sha }} to staging"

  load-test-staging:
    name: Load Test Staging
    needs: deploy-staging
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - run: pip install locust

      - name: Wait for staging to be ready
        run: |
          for i in {1..30}; do
            if curl -sf https://staging.myapp.com/health; then
              echo "Staging is ready!"
              break
            fi
            echo "Waiting... ($i/30)"
            sleep 10
          done

      - name: Run load test
        run: |
          locust -f locustfile.py \
            --host=https://staging.myapp.com \
            --users=50 \
            --spawn-rate=5 \
            --run-time=2m \
            --headless \
            --csv=staging-results \
            --html=staging-load-test.html

      - name: Check performance thresholds
        run: python analyze_results.py staging-results

      - name: Upload load test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: load-test-results
          path: |
            staging-results*.csv
            staging-load-test.html

  deploy-production:
    name: Deploy to Production
    needs: load-test-staging
    runs-on: ubuntu-latest
    environment: production    # requires manual approval!

    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # Update production deployment
```

---

## 📊 Complete Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│              TESTING PHASE COMPLETE OVERVIEW                   │
│                                                                │
│  COVERAGE:                                                     │
│  pytest --cov=src --cov-report=html                           │
│  .coveragerc: branch=True, fail_under=80                      │
│  Find gaps → write tests → improve coverage                   │
│  Target: 80-90% for most projects                             │
│                                                                │
│  CI/CD PIPELINE:                                               │
│  Push → Lint → Unit Tests → Integration Tests → Build →       │
│       → Deploy Staging → Load Test → Deploy Production        │
│                                                                │
│  GITHUB ACTIONS KEY CONCEPTS:                                  │
│  on: push/pull_request → trigger                              │
│  jobs: → separate stages                                       │
│  needs: → job ordering                                        │
│  services: → PostgreSQL, Redis in CI                          │
│  matrix: → test multiple Python versions                      │
│  cache: → speed up dependency install                         │
│  environment: → staging/production approval gates             │
│  secrets: → API keys, tokens                                  │
│                                                                │
│  LOAD TESTING WITH LOCUST:                                     │
│  HttpUser + @task(weight) → simulate user behavior            │
│  wait_time = between(1, 5) → realistic spacing                │
│  catch_response=True → handle expected errors                 │
│  LoadTestShape → custom traffic patterns                      │
│  --headless --csv → CI/automated mode                         │
│                                                                │
│  KEY METRICS TO TRACK:                                         │
│  Error rate: < 0.1% normal, < 1% under load                  │
│  P95 response: < 200ms for most APIs                         │
│  P99 response: < 500ms for most APIs                         │
│  RPS: measure your baseline and growth capacity               │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  What is the difference between line coverage and
    branch coverage? Which is more thorough?
2.  What does --cov-fail-under=80 do?
3.  What is .coveragerc used for? Name 3 settings.
4.  What is the purpose of exclude_lines in .coveragerc?
5.  What is a GitHub Actions workflow? What triggers it?
6.  What is needs: in a GitHub Actions job?
7.  How do you start a PostgreSQL service in GitHub Actions?
8.  What is a GitHub Actions environment (staging/production)?
9.  What is concurrency: in GitHub Actions?
10. What is a LoadTestShape in Locust?
11. What is the difference between @task(1) and @task(5)?
12. What does on_start() do in a Locust User class?
13. What is wait_time = between(1, 5)?
14. What does --headless mode do in Locust?
15. What are the key load test metrics to check?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Coverage Improvement
# Run coverage on your Phase 4 FastAPI project.
# Find the top 5 uncovered files/functions.
# Write tests to bring coverage from wherever it is to 85%+.
# Document what you found and how you fixed each gap.

# Exercise 2: CI Pipeline for Your Project
# Create a complete GitHub Actions workflow for your blog API:
# a) Lint job: ruff + black check
# b) Unit test job: with SQLite, coverage report
# c) Integration test job: with PostgreSQL service
# d) Security scan: bandit + safety
# e) Build Docker image job
# f) Matrix test: Python 3.11 and 3.12
# Make sure each job fails correctly when code is broken.

# Exercise 3: Pre-commit Hooks
# Set up pre-commit hooks for your project:
# - ruff (lint + format)
# - black (format check)
# - trailing whitespace
# - end of file fixer
# - check yaml/json/toml
# - no debug statements
# - run unit tests on push (not commit)
# Test: make a commit with a bug → verify hook catches it

# Exercise 4: Locust Load Test
# Write a complete Locust load test for your blog API:
# - ReadOnlyUser: browse posts, search, view post
# - AuthenticatedUser: register, create post, check profile
# - AdminUser: list users, moderate content
# Include:
# - Realistic wait times
# - Proper error handling (404 is ok for posts)
# - Custom LoadTestShape simulating a "lunch rush"
#   (traffic spikes from 10 to 100 users at noon)
# Run headless for 5 minutes and check thresholds

# Exercise 5: Performance Regression Test
# Add a load test to your CI pipeline:
# After deploy to staging → run 1 minute load test
# If P95 > 300ms OR error rate > 1% → block production deploy
# Save results as GitHub Actions artifacts
# Compare current vs previous run results
# Send notification if performance degrades
```

---

## 🎉 Phase 6 Complete!

text

```
✅ 6.1 — pytest Fundamentals
         Test discovery, assertions, fixtures,
         conftest.py, parametrize, markers

✅ 6.2 — Unit Testing & Mocking
         MagicMock, AsyncMock, patch, mocker,
         factories, FastAPI route testing

✅ 6.3 — Integration Testing
         Real database, transaction rollback,
         dependency override, full auth flows

✅ 6.4 — Coverage, CI/CD & Load Testing
         pytest-cov, GitHub Actions pipeline,
         Locust load testing, performance thresholds
```

**You now write tests like a professional engineer.**  
**Your code is tested at unit, integration, and load level.**  
**Your pipeline catches bugs before production does.**