```
Why do senior engineers write tests?

WITHOUT tests:
  "It worked on my machine"
  "I'm afraid to change this code"
  "We need to manually test everything after each change"
  "The bug was introduced 3 weeks ago — which commit?"
  3am production fire because you missed an edge case

WITH tests:
  Change code confidently
  Catch bugs before they reach production
  Document behavior (tests are living documentation)
  Refactor fearlessly
  Sleep through the night

The testing pyramid:

          /\
         /  \
        / E2E \          ← few, slow, test full system
       /────────\
      / Integration\     ← some, medium, test components together
     /──────────────\
    /   Unit Tests    \  ← many, fast, test one thing at a time
   /──────────────────\

This phase: pytest from zero to professional.
```

---

## Setup

Bash

```
cd ~/projects
mkdir testing_mastery
cd testing_mastery
python3 -m venv venv
source venv/bin/activate

pip install pytest pytest-cov pytest-asyncio \
    pytest-mock faker factory-boy httpx \
    fastapi uvicorn sqlalchemy

mkdir -p src tests/{unit,integration,fixtures}
touch src/__init__.py
touch tests/__init__.py
touch tests/unit/__init__.py
touch tests/integration/__init__.py
touch pytest.ini
code .
```

ini

```
# pytest.ini — pytest configuration
[pytest]
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --tb=short
    --strict-markers
    -p no:warnings
markers =
    unit: Unit tests (fast, no external dependencies)
    integration: Integration tests (may use DB, HTTP)
    slow: Slow tests (skip in quick test runs)
    smoke: Smoke tests (quick sanity check)
```

---

## 1. 🚀 pytest Basics

### Your First Test

Python

```
# src/calculator.py — code we'll test
def add(a: float, b: float) -> float:
    return a + b

def subtract(a: float, b: float) -> float:
    return a - b

def multiply(a: float, b: float) -> float:
    return a * b

def divide(a: float, b: float) -> float:
    if b == 0:
        raise ZeroDivisionError("Cannot divide by zero")
    return a / b

def is_even(n: int) -> bool:
    return n % 2 == 0

def factorial(n: int) -> int:
    if n < 0:
        raise ValueError("Factorial of negative number")
    if n == 0:
        return 1
    return n * factorial(n - 1)
```

Python

```
# tests/unit/test_calculator.py

# pytest discovers tests by:
# 1. Files named test_*.py or *_test.py
# 2. Functions named test_*
# 3. Classes named Test*

from src.calculator import add, subtract, multiply, divide, is_even, factorial
import pytest


# ─────────────────────────────────────────
# BASIC TESTS
# ─────────────────────────────────────────
def test_add():
    """Test basic addition."""
    result = add(2, 3)
    assert result == 5


def test_add_floats():
    """Test addition with floats."""
    result = add(0.1, 0.2)
    # Floating point! Don't use == directly
    assert abs(result - 0.3) < 1e-10
    # OR use pytest.approx:
    assert result == pytest.approx(0.3)


def test_add_negative():
    result = add(-5, 3)
    assert result == -2


def test_subtract():
    assert subtract(10, 4) == 6
    assert subtract(0, 5) == -5


def test_multiply():
    assert multiply(3, 4) == 12
    assert multiply(-2, 5) == -10
    assert multiply(0, 1000) == 0


# ─────────────────────────────────────────
# TESTING EXCEPTIONS
# ─────────────────────────────────────────
def test_divide_by_zero():
    """Test that dividing by zero raises ZeroDivisionError."""
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)


def test_divide_by_zero_message():
    """Test exception message."""
    with pytest.raises(ZeroDivisionError, match="Cannot divide by zero"):
        divide(10, 0)


def test_divide_by_zero_stores_exception():
    """Inspect the exception object."""
    with pytest.raises(ZeroDivisionError) as exc_info:
        divide(10, 0)

    exception = exc_info.value
    assert str(exception) == "Cannot divide by zero"
    assert isinstance(exception, ZeroDivisionError)


def test_factorial_negative():
    with pytest.raises(ValueError, match="Factorial of negative"):
        factorial(-1)


def test_factorial_zero():
    assert factorial(0) == 1


def test_factorial_positive():
    assert factorial(5) == 120
    assert factorial(1) == 1


# ─────────────────────────────────────────
# BOOLEAN ASSERTIONS
# ─────────────────────────────────────────
def test_is_even():
    assert is_even(4) is True
    assert is_even(3) is False
    assert is_even(0) is True


# ─────────────────────────────────────────
# COLLECTION ASSERTIONS
# ─────────────────────────────────────────
def test_list_operations():
    items = [1, 2, 3, 4, 5]

    assert len(items) == 5
    assert 3 in items
    assert 99 not in items
    assert items[0] == 1
    assert items[-1] == 5

    # Check list contents (order matters)
    assert items == [1, 2, 3, 4, 5]

    # Check list contents (order doesn't matter)
    assert sorted(items) == [1, 2, 3, 4, 5]


def test_dict_operations():
    user = {"id": 1, "name": "Alice", "role": "admin"}

    assert user["name"] == "Alice"
    assert "email" not in user
    assert user.get("age") is None
    assert len(user) == 3

    # Check subset of dict
    assert {"id": 1, "name": "Alice"}.items() <= user.items()


def test_string_assertions():
    message = "Hello, World!"

    assert message.startswith("Hello")
    assert message.endswith("!")
    assert "World" in message
    assert len(message) == 13
    assert message.lower() == "hello, world!"
```

Bash

```
# Run your tests
pytest                          # run all tests
pytest tests/unit/              # specific directory
pytest tests/unit/test_calculator.py  # specific file
pytest tests/unit/test_calculator.py::test_add  # specific test
pytest -v                       # verbose output
pytest -v --tb=long             # verbose with full traceback
pytest -k "add"                 # run tests matching "add"
pytest -k "add or multiply"     # run tests matching pattern
pytest -x                       # stop at first failure
pytest --lf                     # run last failed tests
pytest -q                       # quiet output (less noise)
```

---

## 2. 📊 Assertions — The Complete Guide

Python

```
# tests/unit/test_assertions.py
import pytest
from typing import List, Dict


# ─────────────────────────────────────────
# NUMERIC ASSERTIONS
# ─────────────────────────────────────────
def test_numeric():
    assert 2 + 2 == 4
    assert 10 > 5
    assert 5 < 10
    assert 5 >= 5
    assert 5 <= 6
    assert 5 != 6

    # Floating point comparison
    assert 0.1 + 0.2 == pytest.approx(0.3)
    assert 0.1 + 0.2 == pytest.approx(0.3, abs=1e-9)      # absolute tolerance
    assert 0.1 + 0.2 == pytest.approx(0.3, rel=1e-6)      # relative tolerance
    assert 1.000001 == pytest.approx(1.0, rel=1e-5)

    # Numbers in collections
    values = [1.1, 2.2, 3.3]
    assert values == pytest.approx([1.1, 2.2, 3.3])


# ─────────────────────────────────────────
# EXCEPTION ASSERTIONS
# ─────────────────────────────────────────
def test_exception_types():
    # Exact exception type
    with pytest.raises(ValueError):
        int("not_a_number")

    # Base class also works
    with pytest.raises(Exception):
        int("not_a_number")

    # Match message with regex
    with pytest.raises(ValueError, match=r"invalid literal.*'not_a_number'"):
        int("not_a_number")

    # Inspect exception
    with pytest.raises(TypeError) as exc_info:
        1 + "two"

    assert "unsupported operand" in str(exc_info.value)
    assert exc_info.type is TypeError


def test_no_exception():
    """Sometimes you want to verify no exception is raised."""
    # Just run the code — if it raises, test fails
    result = int("42")
    assert result == 42


# ─────────────────────────────────────────
# NONE ASSERTIONS
# ─────────────────────────────────────────
def test_none():
    value = None
    assert value is None
    assert value is not True
    assert not value


def test_not_none():
    value = "hello"
    assert value is not None
    assert value


# ─────────────────────────────────────────
# COLLECTION ASSERTIONS
# ─────────────────────────────────────────
def test_collections():
    # Lists
    items = [3, 1, 4, 1, 5, 9, 2, 6]
    assert len(items) == 8
    assert 4 in items
    assert 7 not in items
    assert items[0] == 3
    assert items[-1] == 6

    # Sorted comparison (order doesn't matter)
    assert sorted(items) == [1, 1, 2, 3, 4, 5, 6, 9]

    # All items match condition
    numbers = [2, 4, 6, 8]
    assert all(n % 2 == 0 for n in numbers)

    # At least one matches
    mixed = [1, 2, 3, 4, 5]
    assert any(n > 4 for n in mixed)

    # Sets
    s1 = {1, 2, 3}
    s2 = {2, 3, 4}
    assert s1 & s2 == {2, 3}           # intersection
    assert s1 | s2 == {1, 2, 3, 4}    # union
    assert {1, 2}.issubset(s1)

    # Dicts
    user = {"name": "Alice", "role": "admin", "age": 25}
    assert "name" in user
    assert user.get("missing") is None
    assert user.get("missing", "default") == "default"
    assert list(user.keys()) == ["name", "role", "age"]


# ─────────────────────────────────────────
# PYTEST-SPECIFIC HELPERS
# ─────────────────────────────────────────
def test_pytest_helpers():
    # pytest.approx for floats
    assert 0.1 + 0.2 == pytest.approx(0.3)

    # pytest.approx works with dicts
    result = {"x": 0.1 + 0.2, "y": 1.0 + 2.0}
    expected = {"x": 0.3, "y": 3.0}
    assert result == pytest.approx(expected)

    # Negative assertions
    assert not False
    assert not None
    assert not []
    assert not {}
    assert not ""
    assert not 0
```

---

## 3. 🎣 Fixtures — The Heart of pytest

### What Are Fixtures?

text

```
Fixtures = reusable setup code that tests depend on.

Without fixtures:
  def test_create_user():
      db = setup_database()       # repeated in every test
      user = create_test_user()   # repeated in every test
      result = create_user(user)
      assert result.id is not None
      teardown_database(db)       # repeated in every test

  def test_get_user():
      db = setup_database()       # repeated again
      user = create_test_user()   # repeated again
      ...

With fixtures:
  @pytest.fixture
  def db():
      database = setup_database()
      yield database              # setup done, test runs
      teardown_database(database) # cleanup after test

  @pytest.fixture
  def test_user(db):
      return create_test_user(db)

  def test_create_user(db, test_user):
      # db and test_user automatically provided!
      result = create_user(test_user)
      assert result.id is not None
```

Python

```
# tests/unit/test_fixtures.py
import pytest
from typing import Generator, List
import time


# ─────────────────────────────────────────
# BASIC FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
def sample_user() -> dict:
    """A basic user dictionary for tests."""
    return {
        "id": 1,
        "name": "Alice Smith",
        "email": "alice@test.com",
        "role": "user",
        "is_active": True,
    }


@pytest.fixture
def admin_user() -> dict:
    """An admin user for authorization tests."""
    return {
        "id": 2,
        "name": "Admin User",
        "email": "admin@test.com",
        "role": "admin",
        "is_active": True,
    }


@pytest.fixture
def sample_post(sample_user) -> dict:
    """A post that belongs to sample_user."""
    # Fixtures can depend on other fixtures!
    return {
        "id": 1,
        "title": "My Test Post",
        "content": "This is test content.",
        "author_id": sample_user["id"],
        "status": "published",
    }


def test_user_has_correct_role(sample_user):
    """Fixture is passed as parameter — pytest injects it."""
    assert sample_user["role"] == "user"


def test_admin_has_correct_role(admin_user):
    assert admin_user["role"] == "admin"


def test_post_belongs_to_user(sample_user, sample_post):
    """Multiple fixtures can be used in one test."""
    assert sample_post["author_id"] == sample_user["id"]


# ─────────────────────────────────────────
# FIXTURES WITH SETUP AND TEARDOWN
# ─────────────────────────────────────────
@pytest.fixture
def temp_file(tmp_path) -> Generator:
    """
    Create a temp file, yield it, delete after test.
    tmp_path is a pytest built-in fixture providing a temp dir.
    """
    file_path = tmp_path / "test_data.txt"
    file_path.write_text("initial content")

    yield file_path   # ← test runs here

    # Teardown: cleanup after test
    if file_path.exists():
        file_path.unlink()
    print(f"\nCleaned up: {file_path}")


def test_file_has_content(temp_file):
    content = temp_file.read_text()
    assert content == "initial content"


def test_can_write_to_file(temp_file):
    temp_file.write_text("new content")
    assert temp_file.read_text() == "new content"


# ─────────────────────────────────────────
# FIXTURE SCOPE — how long fixtures live
# ─────────────────────────────────────────
"""
Scope options:
  function  (default): new fixture for each test function
  class:               shared within a test class
  module:              shared within a test module (file)
  package:             shared within a test package
  session:             shared for the entire test session

Use broader scope to save setup time for expensive resources.
"""

setup_count = 0


@pytest.fixture(scope="function")
def function_scoped():
    """Created and destroyed for EACH test function."""
    global setup_count
    setup_count += 1
    print(f"\nFunction fixture setup #{setup_count}")
    yield f"function_{setup_count}"
    print(f"\nFunction fixture teardown #{setup_count}")


@pytest.fixture(scope="module")
def module_scoped():
    """Created ONCE for the entire module, destroyed after."""
    print("\nModule fixture setup — expensive operation!")
    # e.g., start a test server, load large data, connect to DB
    data = {"expensive": "data", "loaded_at": time.time()}
    yield data
    print("\nModule fixture teardown")


@pytest.fixture(scope="session")
def session_scoped():
    """Created ONCE for the entire test session."""
    print("\nSession fixture setup — runs ONCE for all tests!")
    yield {"session": "data"}
    print("\nSession fixture teardown")


def test_scope_one(function_scoped, module_scoped, session_scoped):
    assert function_scoped.startswith("function_")
    assert "expensive" in module_scoped


def test_scope_two(function_scoped, module_scoped, session_scoped):
    # function_scoped is new (different instance)
    # module_scoped is SAME object as in test_scope_one
    # session_scoped is SAME object as in test_scope_one
    assert "expensive" in module_scoped


# ─────────────────────────────────────────
# AUTOUSE FIXTURES — run automatically
# ─────────────────────────────────────────
@pytest.fixture(autouse=True)
def reset_state():
    """Runs before EVERY test in this module — no need to include it."""
    # Setup
    print("\nSetting up test environment...")

    yield

    # Teardown
    print("\nCleaning up after test...")


def test_with_automatic_reset():
    """This test gets reset_state automatically."""
    pass


# ─────────────────────────────────────────
# BUILT-IN FIXTURES
# ─────────────────────────────────────────
def test_tmp_path(tmp_path):
    """pytest provides tmp_path — temporary directory."""
    test_file = tmp_path / "test.txt"
    test_file.write_text("hello")
    assert test_file.read_text() == "hello"
    # Directory cleaned up automatically


def test_capsys(capsys):
    """pytest provides capsys — capture stdout/stderr."""
    print("Hello from test!")

    captured = capsys.readouterr()
    assert "Hello from test!" in captured.out


def test_monkeypatch(monkeypatch):
    """pytest provides monkeypatch — temporarily modify objects."""
    import os

    # Set an environment variable for this test
    monkeypatch.setenv("MY_TEST_VAR", "test_value")
    assert os.environ.get("MY_TEST_VAR") == "test_value"

    # After test: automatically restored (MY_TEST_VAR removed)
```

---

## 4. 📋 conftest.py — Shared Fixtures

Python

```
# tests/conftest.py
"""
conftest.py is automatically loaded by pytest.
Fixtures defined here are available to ALL tests.
No import needed — pytest finds them automatically.

Rule: Put shared fixtures in conftest.py
      Put test-specific fixtures in the test file itself
"""
import pytest
from typing import Generator, Dict, Any
from pathlib import Path


# ─────────────────────────────────────────
# SHARED DATA FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
def user_data() -> Dict[str, Any]:
    """Base user data used across many tests."""
    return {
        "name": "Alice Smith",
        "email": "alice@test.com",
        "password": "SecurePass1!",
        "role": "user"
    }


@pytest.fixture
def admin_data() -> Dict[str, Any]:
    return {
        "name": "Admin User",
        "email": "admin@test.com",
        "password": "AdminPass1!",
        "role": "admin"
    }


@pytest.fixture
def multiple_users() -> list:
    """List of test users."""
    return [
        {"id": i, "name": f"User {i}", "email": f"user{i}@test.com"}
        for i in range(1, 6)
    ]


# ─────────────────────────────────────────
# CONFIGURATION FIXTURES
# ─────────────────────────────────────────
@pytest.fixture(scope="session")
def test_config() -> Dict:
    """App configuration for testing."""
    return {
        "environment": "testing",
        "debug": True,
        "database_url": "sqlite:///test.db",
        "secret_key": "test-secret-key",
        "jwt_secret": "test-jwt-secret",
    }


# ─────────────────────────────────────────
# FILE FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
def sample_csv_file(tmp_path) -> Path:
    """Create a sample CSV file for tests."""
    csv_path = tmp_path / "users.csv"
    csv_path.write_text(
        "id,name,email,role\n"
        "1,Alice,alice@test.com,user\n"
        "2,Bob,bob@test.com,admin\n"
        "3,Charlie,charlie@test.com,user\n"
    )
    return csv_path


@pytest.fixture
def sample_json_file(tmp_path) -> Path:
    """Create a sample JSON file for tests."""
    import json
    json_path = tmp_path / "config.json"
    json_path.write_text(json.dumps({
        "version": "1.0",
        "features": ["auth", "posts", "comments"],
        "settings": {"max_posts": 100}
    }, indent=2))
    return json_path
```

Python

```
# tests/unit/conftest.py — unit test specific fixtures
import pytest
from unittest.mock import AsyncMock, MagicMock


@pytest.fixture
def mock_db():
    """Mock database session for unit tests."""
    mock = MagicMock()
    mock.query.return_value.filter.return_value.first.return_value = None
    mock.add = MagicMock()
    mock.commit = MagicMock()
    mock.rollback = MagicMock()
    return mock


@pytest.fixture
def mock_cache():
    """Mock Redis cache for unit tests."""
    cache = MagicMock()
    cache.get = MagicMock(return_value=None)
    cache.set = MagicMock(return_value=True)
    cache.delete = MagicMock(return_value=True)
    return cache


@pytest.fixture
def mock_email_service():
    """Mock email service."""
    service = AsyncMock()
    service.send = AsyncMock(return_value=True)
    return service
```

---

## 5. 📐 Parametrize — Test Multiple Cases

Python

```
# tests/unit/test_parametrize.py
import pytest
from src.calculator import add, divide, is_even, factorial


# ─────────────────────────────────────────
# BASIC PARAMETRIZE
# ─────────────────────────────────────────
@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (-5, -3, -8),
    (100, 200, 300),
    (0.5, 0.5, 1.0),
])
def test_add_many(a, b, expected):
    """Test add with many different inputs."""
    assert add(a, b) == pytest.approx(expected)

# pytest generates 6 test cases:
# test_add_many[1-2-3]
# test_add_many[0-0-0]
# test_add_many[-1-1-0]
# etc.


# ─────────────────────────────────────────
# PARAMETRIZE WITH IDS
# ─────────────────────────────────────────
@pytest.mark.parametrize("a, b, expected", [
    (10, 2, 5.0),
    (9, 3, 3.0),
    (1, 4, 0.25),
    (-10, 2, -5.0),
], ids=[
    "basic_division",
    "exact_division",
    "fraction_result",
    "negative_dividend",
])
def test_divide_cases(a, b, expected):
    """Test divide with named test cases."""
    assert divide(a, b) == pytest.approx(expected)


# ─────────────────────────────────────────
# PARAMETRIZE WITH MARKS
# ─────────────────────────────────────────
@pytest.mark.parametrize("number, expected", [
    (2, True),
    (3, False),
    (0, True),
    (-2, True),
    (-3, False),
    (1_000_000, True),
    pytest.param(1, False, id="one_is_odd"),
    pytest.param(999, False, marks=pytest.mark.slow),  # mark specific case
])
def test_is_even_parametrize(number, expected):
    assert is_even(number) == expected


# ─────────────────────────────────────────
# MULTIPLE PARAMETRIZE DECORATORS (cartesian product)
# ─────────────────────────────────────────
@pytest.mark.parametrize("a", [1, 2, 3])
@pytest.mark.parametrize("b", [10, 20])
def test_add_cartesian(a, b):
    """Tests every combination: 1+10, 1+20, 2+10, 2+20, 3+10, 3+20"""
    result = add(a, b)
    assert result == a + b


# ─────────────────────────────────────────
# PARAMETRIZE EXCEPTIONS
# ─────────────────────────────────────────
@pytest.mark.parametrize("n, expected_error", [
    (-1, ValueError),
    (-100, ValueError),
])
def test_factorial_errors(n, expected_error):
    with pytest.raises(expected_error):
        factorial(n)


@pytest.mark.parametrize("n, expected", [
    (0, 1),
    (1, 1),
    (2, 2),
    (3, 6),
    (4, 24),
    (5, 120),
    (10, 3628800),
])
def test_factorial_values(n, expected):
    assert factorial(n) == expected


# ─────────────────────────────────────────
# REAL BACKEND EXAMPLES
# ─────────────────────────────────────────
from src.calculator import divide


def validate_email(email: str) -> bool:
    """Simple email validator."""
    import re
    pattern = r'^[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))


@pytest.mark.parametrize("email, is_valid", [
    ("alice@test.com", True),
    ("user.name+tag@example.co.uk", True),
    ("user@subdomain.example.com", True),
    ("not-an-email", False),
    ("missing@tld", False),
    ("@nodomain.com", False),
    ("spaces in@email.com", False),
    ("", False),
    ("a@b.c", True),
])
def test_email_validation(email, is_valid):
    assert validate_email(email) == is_valid


def get_http_status_message(code: int) -> str:
    messages = {
        200: "OK",
        201: "Created",
        400: "Bad Request",
        401: "Unauthorized",
        403: "Forbidden",
        404: "Not Found",
        422: "Unprocessable Entity",
        500: "Internal Server Error",
    }
    return messages.get(code, "Unknown")


@pytest.mark.parametrize("code, expected", [
    (200, "OK"),
    (201, "Created"),
    (400, "Bad Request"),
    (401, "Unauthorized"),
    (404, "Not Found"),
    (500, "Internal Server Error"),
    (999, "Unknown"),
])
def test_status_messages(code, expected):
    assert get_http_status_message(code) == expected
```

---

## 6. 🏷️ Markers — Organize and Filter Tests

Python

```
# tests/unit/test_markers.py
import pytest
import time


# ─────────────────────────────────────────
# BUILT-IN MARKERS
# ─────────────────────────────────────────

@pytest.mark.skip(reason="Feature not implemented yet")
def test_future_feature():
    """This test is skipped entirely."""
    assert False   # never runs


@pytest.mark.skipif(
    condition=True,   # real: platform.system() == "Windows"
    reason="Test only works on Linux"
)
def test_linux_only():
    pass


@pytest.mark.xfail(
    reason="Known bug — see GitHub issue #42",
    strict=False    # False: xfail is OK, True: must fail
)
def test_known_bug():
    """Expected to fail — won't count as test failure."""
    assert 1 == 2   # fails, but test suite still passes


@pytest.mark.xfail(strict=True, reason="Must fail (not yet implemented)")
def test_not_implemented():
    raise NotImplementedError("Feature pending")


# ─────────────────────────────────────────
# CUSTOM MARKERS (defined in pytest.ini)
# ─────────────────────────────────────────
@pytest.mark.unit
def test_pure_logic():
    """Pure logic, no external dependencies."""
    assert 2 + 2 == 4


@pytest.mark.slow
def test_slow_operation():
    """Marked as slow — can skip with: pytest -m 'not slow'"""
    time.sleep(0.5)
    assert True


@pytest.mark.smoke
def test_app_starts():
    """Quick sanity check — run in: pytest -m smoke"""
    assert True


@pytest.mark.integration
def test_integration_thing():
    """Requires external services — pytest -m integration"""
    pass


# ─────────────────────────────────────────
# COMBINING MARKERS
# ─────────────────────────────────────────
@pytest.mark.slow
@pytest.mark.integration
def test_slow_integration():
    """Multiple markers — pytest -m 'slow and integration'"""
    pass


# ─────────────────────────────────────────
# RUNNING WITH MARKERS
# ─────────────────────────────────────────
"""
Run specific markers:
  pytest -m unit           # only unit tests
  pytest -m "not slow"     # skip slow tests
  pytest -m "smoke"        # only smoke tests
  pytest -m "unit and not slow"  # combine
  pytest -m "unit or integration"

In CI/CD:
  Quick check:   pytest -m "not slow and not integration"
  Full suite:    pytest
  Smoke test:    pytest -m smoke
"""
```

---

## 7. 🧩 Testing Real Code — Business Logic

Python

```
# src/user_service.py — the code we'll test properly
from typing import Optional, Dict, List
from dataclasses import dataclass, field
from datetime import datetime
import hashlib
import re


class UserValidationError(Exception):
    def __init__(self, message: str, field: str = None):
        super().__init__(message)
        self.message = message
        self.field = field


class DuplicateEmailError(Exception):
    pass


@dataclass
class User:
    id: int
    name: str
    email: str
    password_hash: str
    role: str = "user"
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.now)


class UserRepository:
    """Simple in-memory user repository for demonstration."""

    def __init__(self):
        self._users: Dict[int, User] = {}
        self._next_id = 1

    def save(self, user: User) -> User:
        if user.id is None:
            user.id = self._next_id
            self._next_id += 1
        self._users[user.id] = user
        return user

    def find_by_id(self, user_id: int) -> Optional[User]:
        return self._users.get(user_id)

    def find_by_email(self, email: str) -> Optional[User]:
        return next(
            (u for u in self._users.values() if u.email == email.lower()),
            None
        )

    def find_all(self) -> List[User]:
        return list(self._users.values())

    def delete(self, user_id: int) -> bool:
        if user_id in self._users:
            del self._users[user_id]
            return True
        return False


class UserService:
    """Business logic for user management."""

    BANNED_DOMAINS = {"tempmail.com", "throwaway.com"}
    MIN_PASSWORD_LENGTH = 8

    def __init__(self, repository: UserRepository):
        self.repo = repository

    def _hash_password(self, password: str) -> str:
        return hashlib.sha256(password.encode()).hexdigest()

    def _validate_email(self, email: str) -> str:
        email = email.lower().strip()
        if not re.match(r'^[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}$', email):
            raise UserValidationError("Invalid email format", field="email")
        domain = email.split("@")[1]
        if domain in self.BANNED_DOMAINS:
            raise UserValidationError(
                f"Email domain '{domain}' is not allowed",
                field="email"
            )
        return email

    def _validate_password(self, password: str) -> None:
        if len(password) < self.MIN_PASSWORD_LENGTH:
            raise UserValidationError(
                f"Password must be at least {self.MIN_PASSWORD_LENGTH} characters",
                field="password"
            )
        if not any(c.isupper() for c in password):
            raise UserValidationError(
                "Password must contain at least one uppercase letter",
                field="password"
            )
        if not any(c.isdigit() for c in password):
            raise UserValidationError(
                "Password must contain at least one number",
                field="password"
            )

    def _validate_name(self, name: str) -> str:
        name = name.strip()
        if len(name) < 2:
            raise UserValidationError(
                "Name must be at least 2 characters",
                field="name"
            )
        return name.title()

    def register(
        self,
        name: str,
        email: str,
        password: str,
        role: str = "user"
    ) -> User:
        """Register a new user with full validation."""
        # Validate inputs
        name = self._validate_name(name)
        email = self._validate_email(email)
        self._validate_password(password)

        # Check uniqueness
        if self.repo.find_by_email(email):
            raise DuplicateEmailError(f"Email '{email}' already registered")

        # Create user
        user = User(
            id=None,
            name=name,
            email=email,
            password_hash=self._hash_password(password),
            role=role,
        )
        return self.repo.save(user)

    def authenticate(self, email: str, password: str) -> Optional[User]:
        """Verify credentials — return user if valid, None if not."""
        user = self.repo.find_by_email(email.lower())
        if not user:
            return None
        if user.password_hash != self._hash_password(password):
            return None
        if not user.is_active:
            return None
        return user

    def update_email(self, user_id: int, new_email: str) -> User:
        """Update user email with validation."""
        user = self.repo.find_by_id(user_id)
        if not user:
            raise ValueError(f"User {user_id} not found")

        new_email = self._validate_email(new_email)

        existing = self.repo.find_by_email(new_email)
        if existing and existing.id != user_id:
            raise DuplicateEmailError(f"Email '{new_email}' already taken")

        user.email = new_email
        return self.repo.save(user)

    def deactivate(self, user_id: int) -> User:
        """Deactivate a user account."""
        user = self.repo.find_by_id(user_id)
        if not user:
            raise ValueError(f"User {user_id} not found")
        user.is_active = False
        return self.repo.save(user)

    def promote_to_admin(self, user_id: int, requester_id: int) -> User:
        """Promote user to admin — only admins can do this."""
        requester = self.repo.find_by_id(requester_id)
        if not requester or requester.role != "admin":
            raise PermissionError("Only admins can promote users")

        user = self.repo.find_by_id(user_id)
        if not user:
            raise ValueError(f"User {user_id} not found")
        if user.role == "admin":
            raise ValueError(f"User {user_id} is already an admin")

        user.role = "admin"
        return self.repo.save(user)
```

Python

```
# tests/unit/test_user_service.py
import pytest
from src.user_service import (
    UserService, UserRepository, UserValidationError,
    DuplicateEmailError, User
)


# ─────────────────────────────────────────
# FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
def repo():
    """Fresh repository for each test."""
    return UserRepository()


@pytest.fixture
def service(repo):
    """UserService with a fresh repository."""
    return UserService(repo)


@pytest.fixture
def registered_user(service):
    """A pre-registered user."""
    return service.register(
        name="Alice Smith",
        email="alice@test.com",
        password="SecurePass1"
    )


@pytest.fixture
def admin_user(service):
    """A pre-registered admin user."""
    return service.register(
        name="Admin User",
        email="admin@test.com",
        password="AdminPass1",
        role="admin"
    )


# ─────────────────────────────────────────
# REGISTRATION TESTS
# ─────────────────────────────────────────
class TestRegistration:
    """Group related tests in a class."""

    def test_register_success(self, service):
        """Happy path: successful registration."""
        user = service.register(
            name="Alice Smith",
            email="alice@test.com",
            password="SecurePass1"
        )

        assert user.id is not None
        assert user.name == "Alice Smith"
        assert user.email == "alice@test.com"
        assert user.role == "user"
        assert user.is_active is True
        assert user.password_hash != "SecurePass1"  # must be hashed!

    def test_register_normalizes_name(self, service):
        """Name is title-cased and stripped."""
        user = service.register(
            name="  alice smith  ",
            email="alice@test.com",
            password="SecurePass1"
        )
        assert user.name == "Alice Smith"

    def test_register_normalizes_email(self, service):
        """Email is lowercased."""
        user = service.register(
            name="Alice",
            email="ALICE@TEST.COM",
            password="SecurePass1"
        )
        assert user.email == "alice@test.com"

    def test_register_assigns_id(self, service):
        """Each user gets a unique sequential ID."""
        user1 = service.register("Alice", "alice@test.com", "SecurePass1")
        user2 = service.register("Bob", "bob@test.com", "SecurePass1")

        assert user1.id != user2.id
        assert user1.id is not None
        assert user2.id is not None

    def test_register_duplicate_email(self, service, registered_user):
        """Cannot register with an existing email."""
        with pytest.raises(DuplicateEmailError):
            service.register(
                name="Alice 2",
                email="alice@test.com",  # already registered
                password="SecurePass1"
            )

    def test_register_custom_role(self, service):
        """Can register with custom role."""
        user = service.register(
            name="Admin",
            email="admin@test.com",
            password="AdminPass1",
            role="admin"
        )
        assert user.role == "admin"

    # ─── Email validation ──────────────────────────────────────
    @pytest.mark.parametrize("invalid_email", [
        "not-an-email",
        "missing@tld",
        "@nodomain.com",
        "",
        "spaces in@email.com",
        "double@@at.com",
    ])
    def test_register_invalid_email(self, service, invalid_email):
        """Invalid emails raise validation error."""
        with pytest.raises(UserValidationError) as exc_info:
            service.register("Alice", invalid_email, "SecurePass1")
        assert exc_info.value.field == "email"

    @pytest.mark.parametrize("banned_email", [
        "user@tempmail.com",
        "user@throwaway.com",
    ])
    def test_register_banned_domain(self, service, banned_email):
        """Banned email domains are rejected."""
        with pytest.raises(UserValidationError, match="not allowed"):
            service.register("Alice", banned_email, "SecurePass1")

    # ─── Password validation ───────────────────────────────────
    @pytest.mark.parametrize("weak_password, expected_error", [
        ("short", "at least 8"),
        ("nouppercase1", "uppercase"),
        ("NONUMBER", "number"),
        ("", "at least 8"),
    ])
    def test_register_weak_password(
        self,
        service,
        weak_password,
        expected_error
    ):
        with pytest.raises(UserValidationError) as exc_info:
            service.register("Alice", "alice@test.com", weak_password)
        assert exc_info.value.field == "password"
        assert expected_error in exc_info.value.message

    # ─── Name validation ──────────────────────────────────────
    @pytest.mark.parametrize("short_name", ["A", "", " "])
    def test_register_short_name(self, service, short_name):
        with pytest.raises(UserValidationError) as exc_info:
            service.register(short_name, "alice@test.com", "SecurePass1")
        assert exc_info.value.field == "name"


# ─────────────────────────────────────────
# AUTHENTICATION TESTS
# ─────────────────────────────────────────
class TestAuthentication:

    def test_authenticate_success(self, service, registered_user):
        """Correct credentials return the user."""
        user = service.authenticate("alice@test.com", "SecurePass1")
        assert user is not None
        assert user.id == registered_user.id
        assert user.email == "alice@test.com"

    def test_authenticate_wrong_password(self, service, registered_user):
        """Wrong password returns None."""
        user = service.authenticate("alice@test.com", "WrongPassword1")
        assert user is None

    def test_authenticate_wrong_email(self, service, registered_user):
        """Non-existent email returns None."""
        user = service.authenticate("wrong@test.com", "SecurePass1")
        assert user is None

    def test_authenticate_inactive_user(self, service, registered_user, repo):
        """Inactive users cannot authenticate."""
        service.deactivate(registered_user.id)
        user = service.authenticate("alice@test.com", "SecurePass1")
        assert user is None

    def test_authenticate_case_insensitive_email(self, service, registered_user):
        """Email lookup is case-insensitive."""
        user = service.authenticate("ALICE@TEST.COM", "SecurePass1")
        assert user is not None


# ─────────────────────────────────────────
# AUTHORIZATION TESTS
# ─────────────────────────────────────────
class TestAuthorization:

    def test_promote_to_admin_success(
        self, service, registered_user, admin_user
    ):
        """Admin can promote a user."""
        promoted = service.promote_to_admin(
            user_id=registered_user.id,
            requester_id=admin_user.id
        )
        assert promoted.role == "admin"

    def test_promote_to_admin_requires_admin(
        self, service, registered_user
    ):
        """Non-admin cannot promote users."""
        # Create another regular user
        other = service.register("Bob", "bob@test.com", "BobPass1")

        with pytest.raises(PermissionError, match="Only admins"):
            service.promote_to_admin(
                user_id=other.id,
                requester_id=registered_user.id   # not an admin!
            )

    def test_promote_already_admin(self, service, admin_user):
        """Cannot promote an already-admin user."""
        with pytest.raises(ValueError, match="already an admin"):
            service.promote_to_admin(
                user_id=admin_user.id,
                requester_id=admin_user.id
            )

    def test_promote_nonexistent_user(self, service, admin_user):
        """Promoting non-existent user raises error."""
        with pytest.raises(ValueError, match="not found"):
            service.promote_to_admin(
                user_id=9999,
                requester_id=admin_user.id
            )


# ─────────────────────────────────────────
# EDGE CASES AND BOUNDARY TESTS
# ─────────────────────────────────────────
class TestEdgeCases:

    def test_register_minimum_valid_password(self, service):
        """Test password at minimum valid length."""
        user = service.register(
            name="Alice",
            email="alice@test.com",
            password="Secure1!"   # exactly 8 chars
        )
        assert user is not None

    def test_register_long_name(self, service):
        """Very long names should work."""
        long_name = "A" * 100
        user = service.register(
            name=long_name,
            email="alice@test.com",
            password="SecurePass1"
        )
        # Title case: first char upper, rest lower
        assert user.name == long_name.title()

    def test_deactivate_nonexistent_user(self, service):
        """Deactivating non-existent user raises error."""
        with pytest.raises(ValueError, match="not found"):
            service.deactivate(9999)

    def test_update_email_to_same_email(self, service, registered_user):
        """User can update email to their own current email."""
        updated = service.update_email(
            registered_user.id,
            "alice@test.com"   # same email they already have
        )
        assert updated.email == "alice@test.com"

    def test_password_hashing_is_deterministic(self, service):
        """Same password always produces same hash."""
        service.register("Alice", "alice@test.com", "SecurePass1")
        user = service.authenticate("alice@test.com", "SecurePass1")
        assert user is not None   # hash comparison works consistently

    def test_multiple_registrations(self, service):
        """System handles multiple users correctly."""
        for i in range(10):
            service.register(
                name=f"User {i}",
                email=f"user{i}@test.com",
                password="SecurePass1"
            )

        all_users = service.repo.find_all()
        assert len(all_users) == 10
        emails = [u.email for u in all_users]
        assert len(set(emails)) == 10  # all unique
```

---

## 8. 📊 Test Coverage

Bash

```
# Install coverage (already in requirements)
pip install pytest-cov

# Run with coverage
pytest --cov=src tests/

# HTML report (more useful)
pytest --cov=src --cov-report=html tests/
# Opens: htmlcov/index.html in browser

# Coverage with terminal output
pytest --cov=src --cov-report=term-missing tests/

# Set minimum coverage threshold (CI/CD use)
pytest --cov=src --cov-fail-under=80 tests/

# Show which lines are NOT covered
pytest --cov=src --cov-report=term-missing:skip-covered tests/
```

ini

```
# .coveragerc — coverage configuration
[run]
source = src
omit =
    src/__init__.py
    src/**/migrations/*
    src/**/tests/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise NotImplementedError
    if TYPE_CHECKING:
    pass

[html]
directory = htmlcov
```

Python

```
# What coverage shows you:

# src/user_service.py
def dangerous_function(x):
    if x > 0:
        return x * 2          # ✅ covered (tests call this)
    else:
        return 0              # ❌ not covered (no test for x <= 0)

# Coverage report:
# user_service.py  85%  Missing: 45, 89-92, 103
# The missing lines = code your tests don't reach
# Those lines could have bugs you'll never catch
```

---

## 9. 🎯 Async Tests

Python

```
# tests/unit/test_async.py
import pytest
import asyncio
from typing import Optional


# Code to test
async def fetch_user_async(user_id: int) -> Optional[dict]:
    """Async function — simulate async DB query."""
    await asyncio.sleep(0)   # yields control (simulates I/O)
    if user_id > 0:
        return {"id": user_id, "name": f"User {user_id}"}
    return None


async def process_users_async(user_ids: list) -> list:
    """Process multiple users concurrently."""
    tasks = [fetch_user_async(uid) for uid in user_ids]
    results = await asyncio.gather(*tasks)
    return [r for r in results if r is not None]


# ─────────────────────────────────────────
# ASYNC TESTS with pytest-asyncio
# ─────────────────────────────────────────

@pytest.mark.asyncio
async def test_fetch_user_success():
    """Test async function."""
    user = await fetch_user_async(1)
    assert user is not None
    assert user["id"] == 1
    assert user["name"] == "User 1"


@pytest.mark.asyncio
async def test_fetch_user_not_found():
    user = await fetch_user_async(-1)
    assert user is None


@pytest.mark.asyncio
async def test_process_users_concurrent():
    """Test concurrent processing."""
    users = await process_users_async([1, 2, 3, 4, 5])
    assert len(users) == 5
    ids = [u["id"] for u in users]
    assert sorted(ids) == [1, 2, 3, 4, 5]


@pytest.mark.asyncio
async def test_invalid_user_ids_filtered():
    """Invalid user IDs are filtered out."""
    users = await process_users_async([1, -1, 2, -2, 3])
    assert len(users) == 3


# ─────────────────────────────────────────
# ASYNC FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
async def async_client():
    """Async fixture — for async resources."""
    # Setup
    client = {"connected": True, "requests": 0}
    yield client
    # Teardown
    client["connected"] = False


@pytest.mark.asyncio
async def test_with_async_fixture(async_client):
    assert async_client["connected"] is True


# ─────────────────────────────────────────
# CONFIGURE asyncio_mode
# ─────────────────────────────────────────
# In pytest.ini:
# [pytest]
# asyncio_mode = auto
#
# With asyncio_mode = auto:
# - No need for @pytest.mark.asyncio decorator
# - All async test functions run with event loop
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│                    PYTEST FUNDAMENTALS                         │
│                                                                │
│  DISCOVERY RULES:                                              │
│  Files: test_*.py or *_test.py                                │
│  Functions: test_*                                             │
│  Classes: Test* (no __init__ needed)                          │
│                                                                │
│  ASSERTIONS:                                                   │
│  assert x == y              ← equality                        │
│  assert x is None           ← identity                        │
│  assert x in collection     ← membership                      │
│  assert x == pytest.approx  ← floats                         │
│  pytest.raises(Error)       ← exceptions                      │
│                                                                │
│  FIXTURES:                                                     │
│  @pytest.fixture            ← basic fixture                   │
│  yield                      ← setup / teardown point          │
│  scope="function|module|session" ← lifetime                   │
│  autouse=True               ← run automatically              │
│  conftest.py                ← shared fixtures                 │
│                                                                │
│  PARAMETRIZE:                                                  │
│  @pytest.mark.parametrize("a,b", [(1,2),(3,4)])               │
│  pytest.param(value, marks=pytest.mark.skip)                  │
│                                                                │
│  MARKERS:                                                      │
│  @pytest.mark.skip          ← always skip                    │
│  @pytest.mark.skipif        ← conditional skip               │
│  @pytest.mark.xfail         ← expected failure               │
│  @pytest.mark.slow/unit/integration ← custom                 │
│                                                                │
│  CLI:                                                          │
│  pytest -v                  ← verbose                         │
│  pytest -k "pattern"        ← filter by name                 │
│  pytest -m "marker"         ← filter by marker               │
│  pytest -x                  ← stop on first failure          │
│  pytest --cov=src           ← with coverage                  │
│  pytest --lf                ← run last failed                 │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  How does pytest discover test files and functions?
2.  What is the difference between assert x == y
    and assert x is y? When to use each?
3.  What is pytest.approx and why do you need it?
4.  What is a fixture and why is it better than setup/teardown?
5.  What does yield do in a fixture?
6.  What is the difference between function, module,
    and session scope?
7.  What is conftest.py and how does pytest find it?
8.  What does @pytest.mark.parametrize do?
    Write an example.
9.  When would you use @pytest.mark.xfail?
10. What does pytest --cov=src show you?
11. How do you test async functions with pytest?
12. What is the autouse=True fixture option?
13. How do you run only tests matching a pattern?
14. What does pytest.raises() do?
    How do you check the exception message?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Test a Password Validator
# Write a PasswordValidator class:
# - min_length (default 8)
# - require_uppercase: bool (default True)
# - require_number: bool (default True)
# - require_special: bool (default False)
# - banned_passwords: list (e.g., ["password", "123456"])
# Method: validate(password) → list of error messages (empty = valid)
# Write parametrized tests covering:
# - Valid passwords (empty error list)
# - Each validation rule failure
# - Banned passwords
# - Edge cases (empty string, single char, all spaces)

# Exercise 2: Test an Order Calculator
# Build OrderCalculator:
# - add_item(name, price, quantity) → None
# - remove_item(name) → None
# - apply_discount(percent) → None (0-100)
# - apply_coupon(code) → None (SAVE10=10%, HALF=50%)
# - subtotal() → float
# - tax(rate=0.1) → float
# - total() → float
# Write tests covering:
# - Happy path for each method
# - Invalid inputs (negative price, > 100% discount)
# - Combined operations (add items + discount + tax)
# - Edge cases (empty cart, remove nonexistent item)
# Use fixtures for a pre-populated cart

# Exercise 3: Test a Rate Limiter
# Build RateLimiter:
# - max_requests: int
# - window_seconds: int
# - is_allowed(user_id) → bool
# - remaining(user_id) → int
# - reset(user_id) → None
# Write tests for:
# - First request always allowed
# - Requests within limit are allowed
# - Exceeding limit blocks requests
# - Window expiry resets the counter (mock time)
# - Multiple users don't affect each other
# Use monkeypatch to control time.time()

# Exercise 4: Fixture Exercise
# Create a conftest.py with these fixtures:
# - db_connection (scope=session): mock database connection
# - db_transaction (scope=function): begins/rolls back transaction
# - seed_users (depends on db_transaction): inserts 5 test users
# - seed_posts (depends on seed_users): inserts 10 test posts
# Write tests that:
# - Use multiple fixtures
# - Verify teardown happens (db_transaction rolls back)
# - Work in isolation (each test has fresh data)

# Exercise 5: Async Test Suite
# Write async tests for a UserCache class:
# class UserCache:
#     async def get(self, user_id) → Optional[dict]
#     async def set(self, user_id, user, ttl=60) → None
#     async def delete(self, user_id) → bool
#     async def get_or_fetch(self, user_id, fetcher) → dict
#       (checks cache, calls fetcher if miss)
# Test:
# - Cache miss returns None
# - Set then get returns same data
# - Expired entries return None (mock time)
# - get_or_fetch calls fetcher only on cache miss
# - Multiple concurrent get_or_fetch for same key
#   calls fetcher only ONCE (stampede prevention)
```

---

## ✅ Phase 6.1 Complete!

**You now know:**

text

```
✅ pytest test discovery rules
✅ All assertion styles including pytest.approx
✅ Fixtures — basic, with teardown, scoped
✅ conftest.py — shared fixtures across test files
✅ @pytest.mark.parametrize — test multiple cases
✅ Built-in and custom markers
✅ skip, xfail, skipif
✅ pytest CLI commands
✅ Test coverage with pytest-cov
✅ Async tests with pytest-asyncio
✅ Testing real business logic with classes
✅ How to organize tests (AAA pattern, classes)
```