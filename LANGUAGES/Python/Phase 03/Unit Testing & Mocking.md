```
Unit test = test ONE thing in isolation.

The problem:
  Your UserService depends on:
    → Database (PostgreSQL)
    → Email service (SendGrid)
    → Cache (Redis)
    → Payment service (Stripe)

  To test UserService.register():
    Without mocking: spin up real DB, real Redis, real email server
    → Slow, flaky, expensive, needs internet, tests interfere with each other

  With mocking: replace all dependencies with fake objects
    → Fast (milliseconds), reliable, no external services needed

Mock = a fake object that:
  ✅ Pretends to be the real thing
  ✅ Returns what you tell it to return
  ✅ Records what was called on it
  ✅ Lets you verify interactions

"Don't test the database in a unit test.
 Test that your code CALLS the database correctly."
```

---

## Setup

Bash

```
cd ~/projects/testing_mastery
# (already set up from 6.1)
pip install pytest-mock faker
touch tests/unit/test_mocking.py
touch src/services.py
touch src/email_service.py
```

---

## 1. 🎭 unittest.mock Fundamentals

Python

```
# tests/unit/test_mocking.py
from unittest.mock import (
    MagicMock,
    AsyncMock,
    patch,
    call,
    sentinel,
    PropertyMock,
    Mock,
    create_autospec,
)
import pytest


# ─────────────────────────────────────────
# MOCK — the basic fake object
# ─────────────────────────────────────────
def test_basic_mock():
    """A Mock accepts any attribute or method call."""
    mock = MagicMock()

    # Access any attribute — auto-created
    print(mock.anything)           # MagicMock()
    print(mock.some_method())      # MagicMock()

    # Configure return values
    mock.get_user.return_value = {"id": 1, "name": "Alice"}
    mock.count.return_value = 42
    mock.is_valid.return_value = True

    result = mock.get_user(1)
    assert result == {"id": 1, "name": "Alice"}
    assert mock.count() == 42
    assert mock.is_valid() is True


def test_mock_call_tracking():
    """Mocks record every call made to them."""
    mock = MagicMock()

    # Call the mock
    mock.send_email("alice@test.com", "Welcome!")
    mock.send_email("bob@test.com", "Hello!")
    mock.log("INFO", "Emails sent")

    # Verify it was called
    assert mock.send_email.called                      # was called at all?
    assert mock.send_email.call_count == 2             # how many times?

    # Verify specific call
    mock.send_email.assert_called_with("bob@test.com", "Hello!")

    # Verify first call
    mock.send_email.assert_any_call("alice@test.com", "Welcome!")

    # Verify called once (fails if called 0 or 2+ times)
    mock.log.assert_called_once_with("INFO", "Emails sent")

    # Verify NOT called
    mock.delete_user.assert_not_called()

    # Inspect all calls
    print(mock.send_email.call_args_list)
    # [call('alice@test.com', 'Welcome!'), call('bob@test.com', 'Hello!')]


def test_mock_side_effects():
    """Side effects control what happens when mock is called."""
    mock = MagicMock()

    # Return different values on successive calls
    mock.get_next.side_effect = [1, 2, 3, 4, 5]
    assert mock.get_next() == 1
    assert mock.get_next() == 2
    assert mock.get_next() == 3

    # Raise an exception
    mock.risky_operation.side_effect = ValueError("Connection failed")
    with pytest.raises(ValueError, match="Connection failed"):
        mock.risky_operation()

    # Use a function as side effect
    call_log = []
    def capture_and_return(arg):
        call_log.append(arg)
        return f"processed_{arg}"

    mock.process.side_effect = capture_and_return
    result = mock.process("item")
    assert result == "processed_item"
    assert "item" in call_log


def test_mock_attributes():
    """Configure nested attributes and properties."""
    mock = MagicMock()

    # Nested attribute access
    mock.user.profile.avatar_url = "https://cdn.example.com/avatar.jpg"
    assert mock.user.profile.avatar_url == "https://cdn.example.com/avatar.jpg"

    # Chain method calls
    mock.db.query.return_value.filter.return_value.first.return_value = {
        "id": 1, "name": "Alice"
    }
    result = mock.db.query("User").filter(id=1).first()
    assert result == {"id": 1, "name": "Alice"}

    # spec: mock only allows attributes the real object has
    class User:
        id: int
        name: str
        def save(self): pass

    strict_mock = MagicMock(spec=User)
    strict_mock.id = 1
    strict_mock.name = "Alice"

    with pytest.raises(AttributeError):
        strict_mock.nonexistent_attr   # spec prevents fake attributes!
```

---

## 2. 🔧 patch — Replacing Real Objects in Tests

Python

```
# src/services.py — code we'll test
import os
import time
from typing import Optional
from datetime import datetime


def get_current_time() -> str:
    return datetime.now().isoformat()


def get_environment() -> str:
    return os.getenv("APP_ENV", "development")


class DatabaseClient:
    def connect(self, url: str) -> bool:
        # Would actually connect to a database
        raise RuntimeError("Use mock in tests!")

    def query(self, sql: str) -> list:
        raise RuntimeError("Use mock in tests!")

    def insert(self, table: str, data: dict) -> dict:
        raise RuntimeError("Use mock in tests!")


class EmailClient:
    def send(self, to: str, subject: str, body: str) -> bool:
        # Would actually send an email
        raise RuntimeError("Use mock in tests!")


class PaymentGateway:
    def charge(self, amount: float, card_token: str) -> dict:
        # Would actually charge a credit card
        raise RuntimeError("Use mock in tests!")


class NotificationService:
    def __init__(self, email_client: EmailClient, db: DatabaseClient):
        self.email = email_client
        self.db = db

    def send_welcome(self, user_email: str, user_name: str) -> bool:
        """Send welcome email and log it to the database."""
        # Send email
        sent = self.email.send(
            to=user_email,
            subject="Welcome!",
            body=f"Hello, {user_name}! Welcome to our platform."
        )

        if sent:
            # Log to database
            self.db.insert("email_logs", {
                "to": user_email,
                "type": "welcome",
                "sent_at": get_current_time(),
            })

        return sent

    def notify_payment(
        self,
        user_email: str,
        amount: float,
        currency: str = "USD"
    ) -> bool:
        return self.email.send(
            to=user_email,
            subject="Payment Received",
            body=f"We received ${amount:.2f} {currency}."
        )


class OrderService:
    def __init__(
        self,
        db: DatabaseClient,
        payment: PaymentGateway,
        notifier: NotificationService
    ):
        self.db = db
        self.payment = payment
        self.notifier = notifier

    def place_order(
        self,
        user_id: int,
        user_email: str,
        items: list,
        card_token: str
    ) -> dict:
        """Place an order: charge payment + save to DB + notify."""
        # Calculate total
        total = sum(item["price"] * item["quantity"] for item in items)

        # Charge payment
        charge = self.payment.charge(total, card_token)
        if charge.get("status") != "success":
            raise RuntimeError("Payment failed")

        # Save order to database
        order = self.db.insert("orders", {
            "user_id": user_id,
            "items": items,
            "total": total,
            "payment_id": charge["payment_id"],
        })

        # Send confirmation
        self.notifier.notify_payment(user_email, total)

        return order
```

Python

```
# tests/unit/test_patch.py
from unittest.mock import MagicMock, patch, call
import pytest
import src.services as services_module
from src.services import (
    NotificationService, OrderService,
    EmailClient, DatabaseClient, PaymentGateway
)


# ─────────────────────────────────────────
# PATCH AS DECORATOR
# ─────────────────────────────────────────
class TestPatchDecorator:

    @patch("src.services.get_current_time")
    def test_send_welcome_logs_timestamp(self, mock_time):
        """
        patch replaces get_current_time during this test.
        After test: automatically restored.
        """
        # Configure mock
        mock_time.return_value = "2024-01-15T14:32:01"

        # Create mocks for dependencies
        mock_email = MagicMock(spec=EmailClient)
        mock_email.send.return_value = True

        mock_db = MagicMock(spec=DatabaseClient)
        mock_db.insert.return_value = {"id": 1}

        # Create service with mocked dependencies
        service = NotificationService(mock_email, mock_db)
        result = service.send_welcome("alice@test.com", "Alice")

        # Verify behavior
        assert result is True

        # Verify email was sent correctly
        mock_email.send.assert_called_once_with(
            to="alice@test.com",
            subject="Welcome!",
            body="Hello, Alice! Welcome to our platform."
        )

        # Verify DB was called with the mocked timestamp
        mock_db.insert.assert_called_once_with(
            "email_logs",
            {
                "to": "alice@test.com",
                "type": "welcome",
                "sent_at": "2024-01-15T14:32:01",  # our mocked time
            }
        )

    @patch("os.getenv")
    def test_get_environment(self, mock_getenv):
        """Patch os.getenv to control environment."""
        mock_getenv.return_value = "production"
        result = services_module.get_environment()
        assert result == "production"
        mock_getenv.assert_called_with("APP_ENV", "development")


# ─────────────────────────────────────────
# PATCH AS CONTEXT MANAGER
# ─────────────────────────────────────────
class TestPatchContext:

    def test_patch_context_manager(self):
        """patch() as context manager — more readable for one-offs."""
        with patch("src.services.get_current_time") as mock_time:
            mock_time.return_value = "2024-01-15T00:00:00"

            mock_email = MagicMock(spec=EmailClient)
            mock_email.send.return_value = True
            mock_db = MagicMock(spec=DatabaseClient)

            service = NotificationService(mock_email, mock_db)
            service.send_welcome("alice@test.com", "Alice")

            # Verify the mock was used
            assert mock_time.called

        # Outside the context manager: get_current_time is real again

    def test_multiple_patches(self):
        """Patch multiple things at once."""
        with patch("src.services.get_current_time") as mock_time, \
             patch("src.services.os.getenv") as mock_env:

            mock_time.return_value = "2024-01-15"
            mock_env.return_value = "testing"

            # Both are patched in this block
            assert services_module.get_environment() == "testing"


# ─────────────────────────────────────────
# PATCH OBJECT ATTRIBUTES
# ─────────────────────────────────────────
class TestPatchObject:

    def test_patch_object(self):
        """patch.object patches a specific method on an object."""
        email_client = EmailClient()

        with patch.object(email_client, "send") as mock_send:
            mock_send.return_value = True

            mock_db = MagicMock(spec=DatabaseClient)
            service = NotificationService(email_client, mock_db)
            result = service.send_welcome("alice@test.com", "Alice")

        assert result is True
        mock_send.assert_called_once()

    def test_patch_dict(self):
        """patch.dict patches dictionaries."""
        config = {"debug": False, "env": "production"}

        with patch.dict(config, {"debug": True, "new_key": "added"}):
            assert config["debug"] is True       # patched
            assert config["env"] == "production" # original preserved
            assert config["new_key"] == "added"  # new key added

        # After context: config is restored
        assert config["debug"] is False
        assert "new_key" not in config

    def test_patch_environment_variable(self, monkeypatch):
        """Use monkeypatch for env vars (pytest's built-in)."""
        monkeypatch.setenv("DATABASE_URL", "test://localhost/testdb")

        result = services_module.get_environment()
        # The env var is set for this test only
        import os
        assert os.environ.get("DATABASE_URL") == "test://localhost/testdb"
```

---

## 3. 🔌 pytest-mock — Cleaner Mocking

Python

```
# tests/unit/test_pytest_mock.py
"""
pytest-mock provides the 'mocker' fixture:
  - Cleaner syntax than unittest.mock
  - Auto-cleanup after each test (no manual restore)
  - Better error messages
"""
import pytest
from src.services import NotificationService, OrderService


class TestWithMocker:

    def test_send_welcome_success(self, mocker):
        """mocker fixture — simplest way to mock in pytest."""

        # Mock dependencies
        mock_email = mocker.MagicMock()
        mock_email.send.return_value = True

        mock_db = mocker.MagicMock()
        mock_db.insert.return_value = {"id": 1}

        # Mock module-level function
        mocker.patch(
            "src.services.get_current_time",
            return_value="2024-01-15T14:32:01"
        )

        service = NotificationService(mock_email, mock_db)
        result = service.send_welcome("alice@test.com", "Alice")

        assert result is True
        mock_email.send.assert_called_once()

    def test_send_welcome_email_fails(self, mocker):
        """Test failure scenario."""
        mock_email = mocker.MagicMock()
        mock_email.send.return_value = False   # email fails!

        mock_db = mocker.MagicMock()
        mocker.patch("src.services.get_current_time", return_value="2024-01-15")

        service = NotificationService(mock_email, mock_db)
        result = service.send_welcome("alice@test.com", "Alice")

        assert result is False
        # DB should NOT be called when email fails
        mock_db.insert.assert_not_called()

    def test_order_placement_success(self, mocker):
        """Test the complete order flow."""
        # Mock all dependencies
        mock_db = mocker.MagicMock()
        mock_payment = mocker.MagicMock()
        mock_notifier = mocker.MagicMock()

        # Configure return values
        mock_payment.charge.return_value = {
            "status": "success",
            "payment_id": "pay_123",
        }
        mock_db.insert.return_value = {
            "id": 1,
            "user_id": 42,
            "total": 59.98,
        }
        mock_notifier.notify_payment.return_value = True

        # Create service
        service = OrderService(mock_db, mock_payment, mock_notifier)

        # Execute
        order = service.place_order(
            user_id=42,
            user_email="alice@test.com",
            items=[
                {"name": "Book", "price": 29.99, "quantity": 2},
            ],
            card_token="tok_visa"
        )

        # Verify the order was created
        assert order["id"] == 1
        assert order["total"] == 59.98

        # Verify payment was charged correctly
        mock_payment.charge.assert_called_once_with(59.98, "tok_visa")

        # Verify order was saved to DB
        mock_db.insert.assert_called_once_with(
            "orders",
            {
                "user_id": 42,
                "items": [{"name": "Book", "price": 29.99, "quantity": 2}],
                "total": 59.98,
                "payment_id": "pay_123",
            }
        )

        # Verify customer was notified
        mock_notifier.notify_payment.assert_called_once_with(
            "alice@test.com", 59.98
        )

    def test_order_payment_fails(self, mocker):
        """Test that order is rolled back when payment fails."""
        mock_db = mocker.MagicMock()
        mock_payment = mocker.MagicMock()
        mock_notifier = mocker.MagicMock()

        # Payment fails
        mock_payment.charge.return_value = {
            "status": "declined",
            "error": "Insufficient funds"
        }

        service = OrderService(mock_db, mock_payment, mock_notifier)

        with pytest.raises(RuntimeError, match="Payment failed"):
            service.place_order(
                user_id=42,
                user_email="alice@test.com",
                items=[{"name": "Book", "price": 29.99, "quantity": 1}],
                card_token="tok_declined"
            )

        # Verify DB was NOT touched when payment failed
        mock_db.insert.assert_not_called()
        mock_notifier.notify_payment.assert_not_called()

    def test_spy_real_object(self, mocker):
        """
        spy: wrap real object, track calls, but execute real code.
        Use when you want real behavior BUT also want to verify calls.
        """
        from src.calculator import add

        # Spy on a real function
        spy = mocker.spy(
            __import__("src.calculator", fromlist=["add"]),
            "add"
        )

        result = add(2, 3)   # real add() is called!

        assert result == 5           # real result
        assert spy.call_count == 1   # but we tracked the call
        spy.assert_called_once_with(2, 3)
```

---

## 4. ⚡ Async Mocking

Python

```
# src/async_services.py
import asyncio
from typing import Optional


class AsyncUserRepository:
    async def get_by_id(self, user_id: int) -> Optional[dict]:
        raise RuntimeError("Use mock in tests!")

    async def get_by_email(self, email: str) -> Optional[dict]:
        raise RuntimeError("Use mock in tests!")

    async def create(self, data: dict) -> dict:
        raise RuntimeError("Use mock in tests!")

    async def update(self, user_id: int, data: dict) -> dict:
        raise RuntimeError("Use mock in tests!")


class AsyncEmailService:
    async def send(self, to: str, subject: str, body: str) -> bool:
        raise RuntimeError("Use mock in tests!")


class AsyncCacheService:
    async def get(self, key: str) -> Optional[dict]:
        raise RuntimeError("Use mock in tests!")

    async def set(self, key: str, value: dict, ttl: int = 300) -> bool:
        raise RuntimeError("Use mock in tests!")

    async def delete(self, key: str) -> bool:
        raise RuntimeError("Use mock in tests!")


class AsyncUserService:
    def __init__(
        self,
        repo: AsyncUserRepository,
        cache: AsyncCacheService,
        email: AsyncEmailService,
    ):
        self.repo = repo
        self.cache = cache
        self.email = email

    async def get_user(self, user_id: int) -> Optional[dict]:
        """Get user with caching."""
        cache_key = f"user:{user_id}"

        # Try cache first
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        # Cache miss: hit database
        user = await self.repo.get_by_id(user_id)
        if not user:
            return None

        # Store in cache
        await self.cache.set(cache_key, user, ttl=300)
        return user

    async def register(self, name: str, email: str, password: str) -> dict:
        """Register a new user."""
        # Check if email exists
        existing = await self.repo.get_by_email(email)
        if existing:
            raise ValueError(f"Email {email} already registered")

        # Create user
        user = await self.repo.create({
            "name": name,
            "email": email,
            "password_hash": f"hashed_{password}",
        })

        # Send welcome email (don't wait for it)
        asyncio.create_task(
            self.email.send(
                to=email,
                subject="Welcome!",
                body=f"Hello, {name}!"
            )
        )

        return user

    async def update_profile(self, user_id: int, data: dict) -> dict:
        """Update user and invalidate cache."""
        user = await self.repo.update(user_id, data)

        # Invalidate cache
        await self.cache.delete(f"user:{user_id}")

        return user
```

Python

```
# tests/unit/test_async_mocking.py
import pytest
import asyncio
from unittest.mock import AsyncMock, MagicMock, patch, call

from src.async_services import (
    AsyncUserService, AsyncUserRepository,
    AsyncEmailService, AsyncCacheService
)


# ─────────────────────────────────────────
# ASYNCMOCK — for mocking async functions
# ─────────────────────────────────────────
class TestAsyncMocking:

    @pytest.mark.asyncio
    async def test_get_user_cache_hit(self):
        """Test cache hit — DB should NOT be called."""
        # AsyncMock for async methods
        mock_repo = AsyncMock(spec=AsyncUserRepository)
        mock_cache = AsyncMock(spec=AsyncCacheService)
        mock_email = AsyncMock(spec=AsyncEmailService)

        # Configure cache to return a user (cache hit)
        cached_user = {"id": 1, "name": "Alice", "email": "alice@test.com"}
        mock_cache.get.return_value = cached_user

        service = AsyncUserService(mock_repo, mock_cache, mock_email)
        result = await service.get_user(1)

        # Should return cached user
        assert result == cached_user

        # Cache should be checked
        mock_cache.get.assert_called_once_with("user:1")

        # DB should NOT be called (cache hit)
        mock_repo.get_by_id.assert_not_called()

    @pytest.mark.asyncio
    async def test_get_user_cache_miss(self):
        """Test cache miss — DB should be called and result cached."""
        mock_repo = AsyncMock(spec=AsyncUserRepository)
        mock_cache = AsyncMock(spec=AsyncCacheService)
        mock_email = AsyncMock(spec=AsyncEmailService)

        # Cache miss
        mock_cache.get.return_value = None

        # DB returns user
        db_user = {"id": 1, "name": "Alice", "email": "alice@test.com"}
        mock_repo.get_by_id.return_value = db_user

        # Cache set succeeds
        mock_cache.set.return_value = True

        service = AsyncUserService(mock_repo, mock_cache, mock_email)
        result = await service.get_user(1)

        # Should return DB user
        assert result == db_user

        # Verify execution order
        mock_cache.get.assert_called_once_with("user:1")
        mock_repo.get_by_id.assert_called_once_with(1)
        mock_cache.set.assert_called_once_with("user:1", db_user, ttl=300)

    @pytest.mark.asyncio
    async def test_get_user_not_found(self):
        """Test user not found in DB."""
        mock_repo = AsyncMock(spec=AsyncUserRepository)
        mock_cache = AsyncMock(spec=AsyncCacheService)
        mock_email = AsyncMock(spec=AsyncEmailService)

        mock_cache.get.return_value = None    # cache miss
        mock_repo.get_by_id.return_value = None  # not in DB

        service = AsyncUserService(mock_repo, mock_cache, mock_email)
        result = await service.get_user(999)

        assert result is None
        # Should NOT cache None
        mock_cache.set.assert_not_called()

    @pytest.mark.asyncio
    async def test_register_success(self):
        """Test user registration."""
        mock_repo = AsyncMock(spec=AsyncUserRepository)
        mock_cache = AsyncMock(spec=AsyncCacheService)
        mock_email = AsyncMock(spec=AsyncEmailService)

        # Email doesn't exist
        mock_repo.get_by_email.return_value = None
        # Create returns new user
        new_user = {"id": 1, "name": "Alice", "email": "alice@test.com"}
        mock_repo.create.return_value = new_user

        service = AsyncUserService(mock_repo, mock_cache, mock_email)

        # Run with event loop to allow background tasks
        result = await service.register("Alice", "alice@test.com", "SecurePass1")

        assert result == new_user

        # Verify email check
        mock_repo.get_by_email.assert_called_once_with("alice@test.com")

        # Verify user creation
        mock_repo.create.assert_called_once_with({
            "name": "Alice",
            "email": "alice@test.com",
            "password_hash": "hashed_SecurePass1",
        })

        # Allow background task (email sending) to run
        await asyncio.sleep(0)

        # Email should be sent in background
        mock_email.send.assert_called_once_with(
            to="alice@test.com",
            subject="Welcome!",
            body="Hello, Alice!"
        )

    @pytest.mark.asyncio
    async def test_register_duplicate_email(self):
        """Test registration with duplicate email."""
        mock_repo = AsyncMock(spec=AsyncUserRepository)
        mock_cache = AsyncMock(spec=AsyncCacheService)
        mock_email = AsyncMock(spec=AsyncEmailService)

        # Email already exists
        mock_repo.get_by_email.return_value = {
            "id": 1,
            "email": "alice@test.com"
        }

        service = AsyncUserService(mock_repo, mock_cache, mock_email)

        with pytest.raises(ValueError, match="already registered"):
            await service.register("Alice 2", "alice@test.com", "Pass123")

        # Should NOT create user
        mock_repo.create.assert_not_called()
        # Should NOT send email
        mock_email.send.assert_not_called()

    @pytest.mark.asyncio
    async def test_update_profile_invalidates_cache(self):
        """Test that updating profile invalidates cache."""
        mock_repo = AsyncMock(spec=AsyncUserRepository)
        mock_cache = AsyncMock(spec=AsyncCacheService)
        mock_email = AsyncMock(spec=AsyncEmailService)

        updated_user = {"id": 1, "name": "Alice Updated"}
        mock_repo.update.return_value = updated_user

        service = AsyncUserService(mock_repo, mock_cache, mock_email)
        result = await service.update_profile(1, {"name": "Alice Updated"})

        assert result == updated_user
        mock_repo.update.assert_called_once_with(1, {"name": "Alice Updated"})
        # Cache MUST be invalidated after update
        mock_cache.delete.assert_called_once_with("user:1")


# ─────────────────────────────────────────
# ASYNC MOCK WITH MOCKER FIXTURE
# ─────────────────────────────────────────
class TestAsyncWithMocker:

    @pytest.mark.asyncio
    async def test_async_with_mocker(self, mocker):
        """pytest-mock + AsyncMock."""
        mock_repo = mocker.AsyncMock(spec=AsyncUserRepository)
        mock_cache = mocker.AsyncMock(spec=AsyncCacheService)
        mock_email = mocker.AsyncMock(spec=AsyncEmailService)

        mock_cache.get.return_value = None
        mock_repo.get_by_id.return_value = {"id": 42, "name": "Alice"}
        mock_cache.set.return_value = True

        service = AsyncUserService(mock_repo, mock_cache, mock_email)
        user = await service.get_user(42)

        assert user["name"] == "Alice"
        mocker.assert_called = mock_cache.get.assert_called_once_with("user:42")

    @pytest.mark.asyncio
    async def test_mock_async_side_effect(self, mocker):
        """AsyncMock with side effects."""
        mock_repo = mocker.AsyncMock(spec=AsyncUserRepository)
        mock_cache = mocker.AsyncMock(spec=AsyncCacheService)
        mock_email = mocker.AsyncMock(spec=AsyncEmailService)

        # Return different values on successive calls
        mock_cache.get.side_effect = [None, {"id": 1, "name": "Alice"}]

        service = AsyncUserService(mock_repo, mock_cache, mock_email)

        # First call: cache miss
        mock_repo.get_by_id.return_value = {"id": 1, "name": "Alice"}
        user1 = await service.get_user(1)

        # Second call: cache hit (from side_effect list)
        user2 = await service.get_user(1)

        assert user1 == user2
        assert mock_cache.get.call_count == 2
```

---

## 5. 🏭 Factories & Fake Data

Python

```
# tests/factories.py
"""
Test data factories using factory_boy.
Much better than hardcoded test data:
  ✅ DRY — define structure once
  ✅ Unique — each factory call creates slightly different data
  ✅ Readable — `UserFactory()` vs copying dicts everywhere
  ✅ Flexible — override specific fields easily
"""
import factory
from factory import faker
from factory.faker import Faker
from datetime import datetime, timezone
from typing import Dict, Any


# ─────────────────────────────────────────
# DICT FACTORIES (for simple dict-based tests)
# ─────────────────────────────────────────
class UserDictFactory(factory.DictFactory):
    """Creates user dictionaries."""
    id = factory.Sequence(lambda n: n + 1)    # 1, 2, 3, ...
    name = Faker("name")                       # random real-looking name
    email = factory.LazyAttribute(
        lambda obj: f"{obj.name.lower().replace(' ', '.')}@test.com"
    )
    password = "SecurePass1!"
    role = "user"
    is_active = True
    is_verified = False
    created_at = factory.LazyFunction(
        lambda: datetime.now(timezone.utc).isoformat()
    )


class AdminUserDictFactory(UserDictFactory):
    """Admin user — inherits from UserFactory."""
    role = "admin"
    is_verified = True


class PostDictFactory(factory.DictFactory):
    id = factory.Sequence(lambda n: n + 1)
    title = Faker("sentence", nb_words=6)
    content = Faker("paragraphs", nb=3, as_list=False)
    slug = factory.LazyAttribute(
        lambda obj: obj.title.lower().replace(" ", "-").rstrip(".")
    )
    status = "published"
    author_id = factory.Sequence(lambda n: n + 1)
    view_count = factory.Faker("random_int", min=0, max=10000)
    like_count = factory.Faker("random_int", min=0, max=1000)
    tags = factory.LazyFunction(lambda: ["python", "backend"])
    created_at = factory.LazyFunction(
        lambda: datetime.now(timezone.utc).isoformat()
    )


class OrderDictFactory(factory.DictFactory):
    id = factory.Sequence(lambda n: n + 1)
    user_id = factory.Sequence(lambda n: n + 1)
    status = "pending"
    total = factory.Faker("pydecimal", left_digits=3, right_digits=2, positive=True)
    items = factory.LazyFunction(lambda: [
        {"name": "Product 1", "price": 29.99, "quantity": 1}
    ])
    created_at = factory.LazyFunction(
        lambda: datetime.now(timezone.utc).isoformat()
    )


# ─────────────────────────────────────────
# USING FACTORIES IN TESTS
# ─────────────────────────────────────────
def test_factory_basics():
    """Understand factory_boy basics."""

    # Create one user
    user = UserDictFactory()
    print(user)
    # {'id': 1, 'name': 'Jennifer Smith', 'email': 'jennifer.smith@test.com', ...}

    # Create another — different data, incremented ID
    user2 = UserDictFactory()
    assert user["id"] != user2["id"]
    assert user["email"] != user2["email"]

    # Override specific fields
    admin = UserDictFactory(
        name="Alice Admin",
        email="alice@test.com",
        role="admin",
        is_active=True
    )
    assert admin["name"] == "Alice Admin"
    assert admin["role"] == "admin"

    # Use admin factory
    admin2 = AdminUserDictFactory()
    assert admin2["role"] == "admin"
    assert admin2["is_verified"] is True

    # Create batch
    users = UserDictFactory.create_batch(5)
    assert len(users) == 5
    ids = [u["id"] for u in users]
    assert len(set(ids)) == 5  # all unique


# ─────────────────────────────────────────
# FAKER — generate realistic fake data
# ─────────────────────────────────────────
from faker import Faker as FakerLib


class FakeDataHelper:
    """
    Centralized fake data generation.
    Use in tests where you need realistic-looking data.
    """

    def __init__(self):
        self.fake = FakerLib(seed=42)  # seed for reproducibility

    def user(self, **overrides) -> Dict[str, Any]:
        data = {
            "name": self.fake.name(),
            "email": self.fake.email(),
            "phone": self.fake.phone_number(),
            "address": self.fake.address(),
            "company": self.fake.company(),
        }
        data.update(overrides)
        return data

    def post(self, **overrides) -> Dict[str, Any]:
        data = {
            "title": self.fake.sentence(nb_words=5),
            "content": self.fake.text(max_nb_chars=2000),
            "tags": self.fake.words(nb=3),
        }
        data.update(overrides)
        return data

    def credit_card(self) -> Dict[str, Any]:
        return {
            "number": self.fake.credit_card_number(),
            "expiry": self.fake.credit_card_expire(),
            "cvv": self.fake.credit_card_security_code(),
            "holder": self.fake.name(),
        }

    def ip_address(self) -> str:
        return self.fake.ipv4()

    def url(self) -> str:
        return self.fake.url()


# ─────────────────────────────────────────
# FIXTURES USING FACTORIES
# ─────────────────────────────────────────
```

Python

```
# tests/unit/test_with_factories.py
import pytest
from tests.factories import (
    UserDictFactory, AdminUserDictFactory,
    PostDictFactory, OrderDictFactory, FakeDataHelper
)
from unittest.mock import MagicMock, AsyncMock


@pytest.fixture
def fake():
    """Shared fake data helper."""
    return FakeDataHelper()


@pytest.fixture
def user():
    """Single test user."""
    return UserDictFactory()


@pytest.fixture
def admin():
    """Single admin user."""
    return AdminUserDictFactory()


@pytest.fixture
def users():
    """List of test users."""
    return UserDictFactory.create_batch(10)


@pytest.fixture
def posts(user):
    """Posts belonging to user."""
    return PostDictFactory.create_batch(5, author_id=user["id"])


# ─────────────────────────────────────────
# TESTS USING FACTORIES
# ─────────────────────────────────────────
class TestWithFactories:

    def test_user_creation(self, user):
        """Factory creates valid user."""
        assert user["id"] is not None
        assert "@" in user["email"]
        assert user["role"] == "user"
        assert user["is_active"] is True

    def test_admin_has_correct_role(self, admin):
        assert admin["role"] == "admin"
        assert admin["is_verified"] is True

    def test_posts_belong_to_user(self, user, posts):
        """All posts belong to the fixture user."""
        for post in posts:
            assert post["author_id"] == user["id"]

    def test_unique_users_in_batch(self, users):
        """Factory generates unique users."""
        emails = [u["email"] for u in users]
        assert len(set(emails)) == len(emails)  # all unique

    def test_override_factory_fields(self):
        """Override specific fields."""
        user = UserDictFactory(
            email="specific@test.com",
            role="moderator",
            is_verified=True
        )
        assert user["email"] == "specific@test.com"
        assert user["role"] == "moderator"
        # Other fields still generated
        assert user["name"] is not None

    def test_fake_data_realistic(self, fake):
        """Faker generates realistic-looking data."""
        user = fake.user()
        assert "@" in user["email"] or "." in user["email"]
        assert len(user["name"]) > 0
        assert len(user["phone"]) > 0

    def test_service_with_factory_data(self, mocker, user):
        """Use factory data in service tests."""
        mock_repo = mocker.MagicMock()
        mock_repo.find_by_email.return_value = None  # email not taken

        # Factory gives us realistic user data
        mock_repo.save.return_value = {
            **user,
            "id": 1,
            "password_hash": "hashed_SecurePass1"
        }

        # Test service uses this realistic data
        from src.user_service import UserService, UserRepository
        repo = mock_repo
        service = UserService(repo)

        result = service.register(
            name=user["name"],
            email=user["email"],
            password="SecurePass1"
        )

        assert result["email"] == user["email"].lower()
```

---

## 6. 🎯 Testing FastAPI Routes

Python

```
# src/api_routes.py
from fastapi import FastAPI, HTTPException, Depends, status
from pydantic import BaseModel
from typing import Optional


class CreateUserRequest(BaseModel):
    name: str
    email: str
    password: str


class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    role: str
    is_active: bool


class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    author_id: int


# Dependency (we'll mock this in tests)
def get_user_service():
    """Would return real service in production."""
    raise NotImplementedError("Inject in tests")


def get_current_user():
    """Would validate JWT in production."""
    raise NotImplementedError("Inject in tests")


app = FastAPI()


@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(
    data: CreateUserRequest,
    service=Depends(get_user_service)
):
    """Create a new user."""
    try:
        user = service.register(data.name, data.email, data.password)
        return user
    except ValueError as e:
        raise HTTPException(status_code=409, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=422, detail=str(e))


@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int,
    service=Depends(get_user_service),
    current_user=Depends(get_current_user)
):
    """Get user by ID."""
    user = service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@app.get("/users")
def list_users(
    page: int = 1,
    page_size: int = 20,
    role: Optional[str] = None,
    service=Depends(get_user_service),
    current_user=Depends(get_current_user)
):
    """List users (admin only)."""
    if current_user.get("role") != "admin":
        raise HTTPException(status_code=403, detail="Admin only")

    return service.get_users(page=page, page_size=page_size, role=role)


@app.delete("/users/{user_id}", status_code=204)
def delete_user(
    user_id: int,
    service=Depends(get_user_service),
    current_user=Depends(get_current_user)
):
    """Delete a user."""
    if current_user.get("role") != "admin" and current_user.get("id") != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")

    success = service.delete_user(user_id)
    if not success:
        raise HTTPException(status_code=404, detail="User not found")
```

Python

```
# tests/unit/test_api_routes.py
import pytest
from fastapi.testclient import TestClient
from unittest.mock import MagicMock
from src.api_routes import app, get_user_service, get_current_user
from tests.factories import UserDictFactory, AdminUserDictFactory


# ─────────────────────────────────────────
# SETUP: Override dependencies
# ─────────────────────────────────────────
@pytest.fixture
def mock_service():
    """Mock user service."""
    return MagicMock()


@pytest.fixture
def regular_user():
    """Simulate logged-in regular user."""
    return UserDictFactory(id=1, role="user")


@pytest.fixture
def admin():
    """Simulate logged-in admin user."""
    return AdminUserDictFactory(id=99)


@pytest.fixture
def client_as_user(mock_service, regular_user):
    """Test client authenticated as regular user."""
    app.dependency_overrides[get_user_service] = lambda: mock_service
    app.dependency_overrides[get_current_user] = lambda: regular_user

    with TestClient(app) as client:
        yield client, mock_service

    app.dependency_overrides.clear()


@pytest.fixture
def client_as_admin(mock_service, admin):
    """Test client authenticated as admin."""
    app.dependency_overrides[get_user_service] = lambda: mock_service
    app.dependency_overrides[get_current_user] = lambda: admin

    with TestClient(app) as client:
        yield client, mock_service

    app.dependency_overrides.clear()


@pytest.fixture
def anon_client(mock_service):
    """Test client with no authentication."""
    app.dependency_overrides[get_user_service] = lambda: mock_service
    # No get_current_user override — dependency will fail if accessed

    with TestClient(app, raise_server_exceptions=False) as client:
        yield client, mock_service

    app.dependency_overrides.clear()


# ─────────────────────────────────────────
# TESTS
# ─────────────────────────────────────────
class TestCreateUser:

    def test_create_user_success(self, anon_client):
        """POST /users creates a user."""
        client, mock_service = anon_client

        new_user = UserDictFactory(
            id=1,
            name="Alice Smith",
            email="alice@test.com"
        )
        mock_service.register.return_value = new_user

        response = client.post("/users", json={
            "name": "Alice Smith",
            "email": "alice@test.com",
            "password": "SecurePass1!"
        })

        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "Alice Smith"
        assert data["email"] == "alice@test.com"

        # Verify service was called correctly
        mock_service.register.assert_called_once_with(
            "Alice Smith", "alice@test.com", "SecurePass1!"
        )

    def test_create_user_duplicate_email(self, anon_client):
        """POST /users with existing email returns 409."""
        client, mock_service = anon_client

        mock_service.register.side_effect = ValueError(
            "Email already registered"
        )

        response = client.post("/users", json={
            "name": "Alice",
            "email": "existing@test.com",
            "password": "SecurePass1!"
        })

        assert response.status_code == 409
        assert "already registered" in response.json()["detail"]

    def test_create_user_missing_fields(self, anon_client):
        """POST /users with missing fields returns 422."""
        client, mock_service = anon_client

        response = client.post("/users", json={
            "name": "Alice"
            # missing email and password
        })

        assert response.status_code == 422
        # Service should NOT be called
        mock_service.register.assert_not_called()


class TestGetUser:

    def test_get_user_success(self, client_as_user):
        """GET /users/{id} returns user."""
        client, mock_service = client_as_user

        user = UserDictFactory(id=1)
        mock_service.get_user.return_value = user

        response = client.get("/users/1")

        assert response.status_code == 200
        assert response.json()["id"] == 1

    def test_get_user_not_found(self, client_as_user):
        """GET /users/{id} returns 404 for missing user."""
        client, mock_service = client_as_user

        mock_service.get_user.return_value = None

        response = client.get("/users/9999")

        assert response.status_code == 404
        assert "not found" in response.json()["detail"].lower()

    def test_get_user_requires_auth(self):
        """GET /users/{id} without auth returns 500 (dependency error)."""
        # No override for get_current_user
        with TestClient(app, raise_server_exceptions=False) as client:
            app.dependency_overrides[get_user_service] = lambda: MagicMock()
            response = client.get("/users/1")
            # Dependency will fail
            assert response.status_code in [401, 500]
            app.dependency_overrides.clear()


class TestListUsers:

    def test_list_users_admin_success(self, client_as_admin):
        """GET /users returns list for admin."""
        client, mock_service = client_as_admin

        users = UserDictFactory.create_batch(3)
        mock_service.get_users.return_value = {
            "items": users,
            "total": 3,
            "page": 1
        }

        response = client.get("/users")

        assert response.status_code == 200
        mock_service.get_users.assert_called_once_with(
            page=1, page_size=20, role=None
        )

    def test_list_users_regular_user_forbidden(self, client_as_user):
        """GET /users returns 403 for regular user."""
        client, mock_service = client_as_user

        response = client.get("/users")

        assert response.status_code == 403
        mock_service.get_users.assert_not_called()

    def test_list_users_with_filters(self, client_as_admin):
        """GET /users with query params."""
        client, mock_service = client_as_admin

        mock_service.get_users.return_value = {"items": [], "total": 0}

        response = client.get("/users?page=2&page_size=10&role=admin")

        assert response.status_code == 200
        mock_service.get_users.assert_called_once_with(
            page=2, page_size=10, role="admin"
        )


class TestDeleteUser:

    def test_admin_can_delete_any_user(self, client_as_admin):
        """Admin can delete any user."""
        client, mock_service = client_as_admin
        mock_service.delete_user.return_value = True

        response = client.delete("/users/1")

        assert response.status_code == 204
        mock_service.delete_user.assert_called_once_with(1)

    def test_user_can_delete_self(self, client_as_user, regular_user):
        """User can delete their own account."""
        client, mock_service = client_as_user
        mock_service.delete_user.return_value = True

        # User ID matches their own ID
        user_id = regular_user["id"]
        response = client.delete(f"/users/{user_id}")

        assert response.status_code == 204

    def test_user_cannot_delete_other(self, client_as_user):
        """Regular user cannot delete another user's account."""
        client, mock_service = client_as_user

        # Try to delete user 999 (not their own)
        response = client.delete("/users/999")

        assert response.status_code == 403
        mock_service.delete_user.assert_not_called()

    def test_delete_nonexistent_user(self, client_as_admin):
        """Deleting non-existent user returns 404."""
        client, mock_service = client_as_admin
        mock_service.delete_user.return_value = False

        response = client.delete("/users/9999")

        assert response.status_code == 404
```

---

## 7. 🎪 Advanced Mocking Patterns

Python

```
# tests/unit/test_advanced_mocking.py
import pytest
from unittest.mock import (
    MagicMock, AsyncMock, patch, call,
    PropertyMock, sentinel, create_autospec
)


# ─────────────────────────────────────────
# PROPERTY MOCK
# ─────────────────────────────────────────
class User:
    def __init__(self, role: str):
        self._role = role

    @property
    def is_admin(self) -> bool:
        return self._role == "admin"

    @property
    def role(self) -> str:
        return self._role


def test_property_mock():
    """Mock a property value."""
    user = User("user")

    with patch.object(User, "is_admin", new_callable=PropertyMock) as mock_is_admin:
        mock_is_admin.return_value = True

        # Now user.is_admin returns True even though role is "user"
        assert user.is_admin is True
        mock_is_admin.assert_called_once()


# ─────────────────────────────────────────
# CREATE_AUTOSPEC — strict mocking
# ─────────────────────────────────────────
class EmailService:
    def send(self, to: str, subject: str, body: str) -> bool:
        raise NotImplementedError

    def send_bulk(self, emails: list) -> dict:
        raise NotImplementedError


def test_autospec():
    """
    create_autospec: mock that ENFORCES the real interface.
    Prevents calling mock with wrong arguments.
    """
    # Regular mock — accepts anything (dangerous!)
    regular_mock = MagicMock()
    regular_mock.send("alice", "hi", "body", "extra_arg", wrong=True)
    # No error! But this would fail with real EmailService.

    # Autospec — enforces real interface
    strict_mock = create_autospec(EmailService)

    # Correct call — works
    strict_mock.send("alice@test.com", "Subject", "Body")

    # Wrong number of args — fails!
    with pytest.raises(TypeError):
        strict_mock.send("only_one_arg")

    # Wrong kwarg — fails!
    with pytest.raises(TypeError):
        strict_mock.send("alice@test.com", "sub", "body", nonexistent_param=True)

    # Calling non-existent method — fails!
    with pytest.raises(AttributeError):
        strict_mock.nonexistent_method()


# ─────────────────────────────────────────
# SENTINEL — unique values for testing
# ─────────────────────────────────────────
def test_sentinel():
    """
    sentinel creates unique named objects.
    Use when you need a distinct value but don't care what it is.
    """
    mock = MagicMock()
    mock.get_user.return_value = sentinel.USER_OBJECT

    result = mock.get_user(1)

    # Test that it's the exact sentinel, not just any truthy value
    assert result is sentinel.USER_OBJECT
    # Readable in test output — shows "sentinel.USER_OBJECT" not "<MagicMock>"


# ─────────────────────────────────────────
# CALL OBJECT — verify complex call args
# ─────────────────────────────────────────
def test_call_args_verification():
    """Verify complex argument combinations."""
    mock = MagicMock()

    mock.process(1, 2, operation="add", verbose=True)
    mock.process(3, 4, operation="multiply", verbose=False)

    # Verify specific calls
    assert mock.process.call_args_list == [
        call(1, 2, operation="add", verbose=True),
        call(3, 4, operation="multiply", verbose=False),
    ]

    # Access call args directly
    first_call = mock.process.call_args_list[0]
    assert first_call.args == (1, 2)
    assert first_call.kwargs == {"operation": "add", "verbose": True}


# ─────────────────────────────────────────
# MOCKING CONTEXT MANAGERS
# ─────────────────────────────────────────
def read_from_file(filename: str) -> str:
    with open(filename, 'r') as f:
        return f.read()


def test_mock_context_manager(mocker):
    """Mock file open with context manager."""
    mock_file = mocker.mock_open(read_data="file content here")

    with mocker.patch("builtins.open", mock_file):
        result = read_from_file("any_file.txt")

    assert result == "file content here"
    mock_file.assert_called_once_with("any_file.txt", "r")


# ─────────────────────────────────────────
# MOCKING CLASSES (replace entire class)
# ─────────────────────────────────────────
def create_and_send_email(to: str, subject: str) -> bool:
    """Function that creates EmailService internally."""
    service = EmailService()   # we want to mock this
    return service.send(to, subject, "Default body")


def test_mock_class_instantiation(mocker):
    """
    Patch the class itself.
    Any call to EmailService() returns our mock.
    """
    mock_instance = mocker.MagicMock()
    mock_instance.send.return_value = True

    # Patch the class — EmailService() now returns mock_instance
    with mocker.patch("src.services.EmailService", return_value=mock_instance):
        result = create_and_send_email("alice@test.com", "Hello!")

    assert result is True
    mock_instance.send.assert_called_once_with(
        "alice@test.com", "Hello!", "Default body"
    )


# ─────────────────────────────────────────
# FREEZING TIME
# ─────────────────────────────────────────
from datetime import datetime, timezone


def get_user_age_message(birth_year: int) -> str:
    current_year = datetime.now(timezone.utc).year
    age = current_year - birth_year
    return f"User is {age} years old"


def test_freeze_time(mocker):
    """Control what 'now' means in tests."""
    # Mock datetime.now to return a fixed time
    fixed_time = datetime(2024, 1, 15, tzinfo=timezone.utc)
    mock_datetime = mocker.patch("src.services.datetime")
    mock_datetime.now.return_value = fixed_time

    # Now we know exactly what "now" is
    # Tests are predictable regardless of when they run
    result = get_user_age_message(1990)
    # 2024 - 1990 = 34
    assert "34" in result


# ─────────────────────────────────────────
# VERIFY ORDER OF CALLS
# ─────────────────────────────────────────
def test_call_order(mocker):
    """Verify mocks were called in specific order."""
    parent_mock = MagicMock()

    # Access child mocks through parent to track order
    parent_mock.validate("data")
    parent_mock.save("data")
    parent_mock.notify("data")

    # Verify order
    assert parent_mock.mock_calls == [
        call.validate("data"),
        call.save("data"),
        call.notify("data"),
    ]

    # Or use a manager mock to track calls across different mocks
    manager = MagicMock()
    db = manager.db
    cache = manager.cache
    notifier = manager.notifier

    db.save("record")
    cache.set("key", "value")
    notifier.send("event")

    assert manager.mock_calls == [
        call.db.save("record"),
        call.cache.set("key", "value"),
        call.notifier.send("event"),
    ]
```

---

## 8. 📏 What to Mock vs What Not to Mock

Python

```
"""
THE GOLDEN RULES OF MOCKING:

MOCK:
  ✅ External services (Stripe, SendGrid, AWS, external APIs)
  ✅ Database calls (in unit tests)
  ✅ Cache (Redis) calls (in unit tests)
  ✅ File system operations (in unit tests)
  ✅ Time-dependent code (datetime.now(), time.time())
  ✅ Random values (random.random(), uuid.uuid4())
  ✅ Network calls (HTTP requests)
  ✅ System calls (os.getenv, subprocess)

DON'T MOCK:
  ❌ The code you're testing (that's what you're testing!)
  ❌ Pure functions with no side effects
  ❌ Data classes and simple objects
  ❌ Simple Python built-ins (list, dict, str, etc.)
  ❌ Private implementation details

INTEGRATION TESTS (Phase 6.3):
  Use REAL database, REAL cache
  Mock only truly external services (Stripe, email, etc.)

THE DEEPER QUESTION:
  "If I mock everything, am I testing anything real?"

  Unit tests: mock dependencies, test logic in isolation
  Integration tests: real DB, real Redis, test components together
  E2E tests: full system, minimal mocking

  You need ALL THREE for confidence.
"""

# Example of WHAT NOT to mock:
from src.calculator import add


def test_dont_mock_what_youre_testing():
    """Don't mock the function you're testing — that's circular."""
    # ❌ WRONG:
    # with patch("src.calculator.add") as mock_add:
    #     mock_add.return_value = 5
    #     assert add(2, 3) == 5  # you're testing your mock!

    # ✅ RIGHT:
    assert add(2, 3) == 5  # test the real function


def test_dont_mock_simple_data_structures():
    """No need to mock dicts, lists, strings."""
    # ❌ WRONG: mock_list = MagicMock()
    # ✅ RIGHT:
    items = [1, 2, 3]
    assert len(items) == 3
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│                UNIT TESTING & MOCKING                         │
│                                                                │
│  MOCK TYPES:                                                   │
│  MagicMock      → sync mock (any attribute/method)           │
│  AsyncMock      → async mock (await-able methods)            │
│  create_autospec→ strict mock (enforces real interface)       │
│  PropertyMock   → mock @property                              │
│                                                                │
│  PATCHING:                                                     │
│  @patch("module.thing")         → decorator                   │
│  with patch("module.thing"):    → context manager             │
│  patch.object(obj, "method")    → specific object            │
│  patch.dict(dict, new_values)   → dict override              │
│  mocker.patch("module.thing")   → pytest-mock (cleanest)    │
│                                                                │
│  CONFIGURE MOCKS:                                              │
│  mock.method.return_value = x   → fixed return value         │
│  mock.method.side_effect = [x,y]→ different per call         │
│  mock.method.side_effect = exc  → raise exception            │
│  mock.method.side_effect = func → run function               │
│                                                                │
│  VERIFY CALLS:                                                 │
│  mock.method.assert_called_once()                             │
│  mock.method.assert_called_once_with(args)                    │
│  mock.method.assert_called_with(args)                         │
│  mock.method.assert_any_call(args)                            │
│  mock.method.assert_not_called()                              │
│  mock.method.call_count == N                                  │
│  mock.method.call_args_list == [call(x), call(y)]            │
│                                                                │
│  FACTORIES:                                                    │
│  UserDictFactory()              → one user dict               │
│  UserDictFactory(role="admin")  → override field             │
│  UserDictFactory.create_batch(5)→ list of 5 users            │
│  Faker()                        → realistic fake data         │
│                                                                │
│  ASYNC:                                                        │
│  AsyncMock()                    → mock async methods          │
│  @pytest.mark.asyncio           → run async test             │
│  await mock.async_method()      → works naturally            │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  What is the difference between MagicMock and AsyncMock?
2.  What does patch() do? Where do you apply the patch?
3.  What is the difference between:
    mock.method.assert_called_with(x) and
    mock.method.assert_called_once_with(x)?
4.  When would you use side_effect vs return_value?
5.  What is create_autospec and why is it safer than MagicMock?
6.  What is the mocker fixture from pytest-mock?
    How does it differ from unittest.mock.patch?
7.  How do you mock a property (@property)?
8.  How do you verify two mocks were called in a specific order?
9.  What is a factory (factory_boy) and why use it?
10. How do you test a FastAPI route in isolation?
    What is dependency_overrides?
11. What should you mock in unit tests?
    What should you NOT mock?
12. How do you mock async context managers?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Mock External Services
# Build and test a WeatherAlertService:
# - get_weather(city) → calls external weather API (mock this)
# - check_alerts(city) → get weather, check thresholds
#     if temp > 35°C: return "heat_warning"
#     if temp < -10°C: return "cold_warning"
#     if wind > 100km/h: return "storm_warning"
#     else: return "all_clear"
# - send_alert(city, alert_type) → sends email (mock this)
# Write tests for:
# - Each weather condition (parametrize)
# - Email sent when alert triggered
# - Email NOT sent when all_clear
# - API failure handled gracefully
# - Cache: don't call API twice for same city in 5 minutes

# Exercise 2: Test a Payment Flow
# Build PaymentService:
# - create_payment_intent(user_id, amount) → calls Stripe (mock)
# - confirm_payment(intent_id, card_token) → calls Stripe (mock)
# - handle_webhook(payload, signature) → verify sig + update DB
# Write tests:
# - Successful payment creates order in DB
# - Failed payment doesn't create order
# - Stripe exception handled gracefully
# - Webhook with invalid signature rejected
# - Webhook with valid signature updates order status

# Exercise 3: Async Cache Tests
# Build AsyncCacheService:
# - get(key) → Optional[Any] (async)
# - set(key, value, ttl=300) → bool (async)
# - get_or_set(key, factory_fn, ttl) → Any (async)
#     factory_fn is an async callable
# Write async tests:
# - get returns None on miss
# - set then get returns value
# - get_or_set calls factory on miss
# - get_or_set uses cache on hit (factory NOT called)
# - Multiple concurrent get_or_set for same key:
#     factory called ONCE (use asyncio.gather to simulate)
# Use AsyncMock for the factory

# Exercise 4: Factory Exercise
# Create factories for a blog system:
# - UserFactory: id, name, email, role, is_active
# - PostFactory: id, title, slug, content, status, author_id, tags
# - CommentFactory: id, content, post_id, author_id, parent_id
# Then write tests using ONLY factories for test data:
# - Published posts have tags (at least one)
# - Comment belongs to the post from its fixture
# - Admin users are always verified
# - Create a "nested comments" scenario with factory

# Exercise 5: API Test Suite
# Build a complete test suite for these endpoints:
# POST /auth/login → returns JWT or 401
# GET  /posts      → paginated, filter by tag
# POST /posts      → create (auth required)
# GET  /posts/{id} → get post, increment views
# DELETE /posts/{id} → delete (admin or author only)
# For each endpoint write:
# - Happy path test
# - Auth failure test (where applicable)
# - Input validation test
# - Edge case test
# Use factories for all test data
# Mock the service layer completely
```

---

## ✅ Phase 6.2 Complete!

**You now know:**

text

```
✅ MagicMock and AsyncMock — creating fake objects
✅ return_value and side_effect — configuring mocks
✅ Verifying calls — assert_called_once_with, call_args_list
✅ patch() — replacing real objects during tests
✅ patch as decorator, context manager, patch.object
✅ mocker fixture — pytest-mock's cleaner API
✅ Async mocking with AsyncMock
✅ factory_boy — generating test data
✅ Faker — realistic fake data
✅ Testing FastAPI routes with dependency_overrides
✅ PropertyMock, create_autospec, sentinel
✅ What to mock vs what not to mock
✅ Spy — track calls but execute real code
```