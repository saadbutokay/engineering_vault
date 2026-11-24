```
Unit tests:         Test ONE function/class in isolation.
                    Mock everything external.
                    Fast. Hundreds of them.

Integration tests:  Test multiple components WORKING TOGETHER.
                    Use real database, real cache.
                    Mock only truly external services (Stripe, etc.)
                    Slower. Dozens of them.

E2E tests:          Test the full system from user's perspective.
                    Browser, real network, real everything.
                    Slowest. A few of them.

Why integration tests matter:
  Unit tests pass ✅ but app is broken ❌

  UserService.register() works in isolation.
  UserRepository.create() works in isolation.
  But together? Maybe they disagree on field names.
  Integration tests catch this.

What we'll test:
  ✅ Database operations (real SQLite/PostgreSQL)
  ✅ FastAPI routes with real services and real DB
  ✅ Authentication flow end-to-end
  ✅ Data persistence across requests
  ✅ Transaction rollback on failure
```

---

## Setup

Bash

```
cd ~/projects/testing_mastery
pip install pytest-asyncio httpx sqlalchemy \
    aiosqlite asyncpg alembic python-jose \
    passlib bcrypt

# For Docker-based integration tests (optional but professional):
pip install testcontainers

# Project structure
mkdir -p tests/integration
touch tests/integration/__init__.py
touch tests/integration/conftest.py
touch tests/integration/test_auth.py
touch tests/integration/test_users.py
touch tests/integration/test_posts.py
```

---

## 1. 🏗️ Integration Test Architecture

Python

```
"""
Integration test setup strategy:

Option A: SQLite in-memory (fastest, recommended for CI)
  - Creates fresh DB per test session
  - No PostgreSQL needed
  - Some PostgreSQL features missing (e.g., ARRAY type)
  - Best for most cases

Option B: PostgreSQL with transactions (recommended for production parity)
  - Real PostgreSQL
  - Each test wrapped in transaction
  - Rollback after each test (fast cleanup)
  - Best for production-like testing

Option C: PostgreSQL with TestContainers (full isolation)
  - Docker starts real PostgreSQL just for tests
  - Full isolation, fresh DB every run
  - Slower (Docker startup)
  - Best for CI/CD pipelines

We'll cover A and B. C is shown conceptually.
"""
```

---

## 2. 🗄️ Test Database Setup

Python

```
# tests/integration/conftest.py
"""
Integration test fixtures.
All tests in this directory get these fixtures automatically.
"""
import asyncio
import pytest
import pytest_asyncio
from typing import AsyncGenerator, Generator
from sqlalchemy import create_engine, event, text
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.ext.asyncio import (
    create_async_engine, AsyncSession, async_sessionmaker
)
from httpx import AsyncClient, ASGITransport
from fastapi import FastAPI

# Import your app components
# (adjust these imports to your project structure)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import Column, Integer, String, Boolean, DateTime, func


# ─────────────────────────────────────────
# BASE MODEL (for tests)
# ─────────────────────────────────────────
class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), nullable=False, unique=True)
    password_hash = Column(String(255), nullable=False)
    role = Column(String(20), nullable=False, default="user")
    is_active = Column(Boolean, nullable=False, default=True)
    is_verified = Column(Boolean, nullable=False, default=False)
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now())


class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer, primary_key=True)
    title = Column(String(500), nullable=False)
    content = Column(String, nullable=False)
    slug = Column(String(500), unique=True)
    status = Column(String(20), nullable=False, default="draft")
    author_id = Column(Integer, nullable=False)
    view_count = Column(Integer, nullable=False, default=0)
    created_at = Column(DateTime, server_default=func.now())


# ─────────────────────────────────────────
# OPTION A: SQLite In-Memory
# ─────────────────────────────────────────
SQLITE_URL = "sqlite:///./test.db"
SQLITE_ASYNC_URL = "sqlite+aiosqlite:///./test.db"


@pytest.fixture(scope="session")
def sync_engine():
    """
    Synchronous SQLite engine for the test session.
    scope="session" = created once for all tests.
    """
    engine = create_engine(
        SQLITE_URL,
        connect_args={"check_same_thread": False},
        echo=False,  # set True to see SQL in tests
    )

    # Create all tables
    Base.metadata.create_all(engine)

    yield engine

    # Cleanup: drop tables after all tests
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture(scope="session")
def sync_session_factory(sync_engine):
    """Session factory for the test session."""
    return sessionmaker(
        bind=sync_engine,
        autocommit=False,
        autoflush=False,
    )


@pytest.fixture
def db_session(sync_engine, sync_session_factory) -> Generator[Session, None, None]:
    """
    Provide a database session that ROLLS BACK after each test.

    This is the KEY technique for fast, isolated integration tests:
    1. Begin a transaction
    2. Run the test (which does DB operations)
    3. Roll back the transaction (restores original state)

    Each test gets a clean database, and no cleanup is needed!
    """
    # Start transaction
    connection = sync_engine.connect()
    transaction = connection.begin()

    # Bind session to this connection (so same transaction)
    session = sync_session_factory(bind=connection)

    # Nested transaction for savepoints
    nested = connection.begin_nested()

    # SQLite doesn't support SAVEPOINT well, so handle that
    @event.listens_for(session, "after_transaction_end")
    def restart_savepoint(session, transaction):
        if transaction.nested and not transaction._parent.nested:
            session.expire_all()
            nested._parent.begin_nested()

    yield session

    # Teardown: ALWAYS rollback
    session.close()
    transaction.rollback()
    connection.close()


# ─────────────────────────────────────────
# OPTION B: Async SQLite
# ─────────────────────────────────────────
@pytest_asyncio.fixture(scope="session")
async def async_engine():
    """Async SQLite engine."""
    engine = create_async_engine(
        SQLITE_ASYNC_URL,
        echo=False,
    )

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()


@pytest_asyncio.fixture
async def async_session(async_engine) -> AsyncGenerator[AsyncSession, None]:
    """
    Async database session with rollback.
    """
    async with async_engine.connect() as connection:
        await connection.begin()

        session_factory = async_sessionmaker(
            bind=connection,
            expire_on_commit=False,
        )

        async with session_factory() as session:
            await session.begin_nested()
            yield session
            await session.rollback()

        await connection.rollback()


# ─────────────────────────────────────────
# FASTAPI TEST CLIENT
# ─────────────────────────────────────────
def create_test_app(session_provider) -> FastAPI:
    """Create a FastAPI app configured for testing."""
    from fastapi import FastAPI, Depends
    from fastapi.middleware.cors import CORSMiddleware

    app = FastAPI(title="Test App")

    @app.get("/health")
    def health():
        return {"status": "healthy"}

    return app


@pytest_asyncio.fixture
async def test_app(async_session) -> FastAPI:
    """
    FastAPI app with overridden database dependency.
    Uses the test database session.
    """
    from fastapi import FastAPI
    app = create_test_app(async_session)
    return app


@pytest_asyncio.fixture
async def async_client(test_app) -> AsyncGenerator[AsyncClient, None]:
    """
    Async HTTP client for testing FastAPI.
    Uses ASGITransport (no real network).
    """
    async with AsyncClient(
        transport=ASGITransport(app=test_app),
        base_url="http://test",
    ) as client:
        yield client
```

---

## 3. 🔨 Real Integration Tests — Repository Layer

Python

```
# tests/integration/test_repository.py
"""
Test the repository layer against a REAL database.
No mocks for the database — that's the point!
"""
import pytest
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from datetime import datetime, timezone

from tests.integration.conftest import User, Post


# ─────────────────────────────────────────
# SIMPLE REPOSITORY for testing
# ─────────────────────────────────────────
class UserRepository:
    def __init__(self, session: Session):
        self.session = session

    def create(self, name: str, email: str, password_hash: str,
               role: str = "user") -> User:
        user = User(
            name=name,
            email=email,
            password_hash=password_hash,
            role=role,
        )
        self.session.add(user)
        self.session.flush()  # assigns ID without committing
        return user

    def get_by_id(self, user_id: int) -> User | None:
        return self.session.get(User, user_id)

    def get_by_email(self, email: str) -> User | None:
        from sqlalchemy import select, func
        stmt = select(User).where(
            func.lower(User.email) == email.lower()
        )
        return self.session.execute(stmt).scalar_one_or_none()

    def get_all(self, skip: int = 0, limit: int = 20) -> list:
        from sqlalchemy import select
        stmt = select(User).offset(skip).limit(limit)
        return list(self.session.execute(stmt).scalars().all())

    def update(self, user: User, **updates) -> User:
        for key, value in updates.items():
            setattr(user, key, value)
        self.session.flush()
        return user

    def delete(self, user: User) -> None:
        self.session.delete(user)
        self.session.flush()

    def count(self) -> int:
        from sqlalchemy import select, func
        return self.session.execute(
            select(func.count(User.id))
        ).scalar_one()


# ─────────────────────────────────────────
# FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
def user_repo(db_session: Session) -> UserRepository:
    """User repository using the test database session."""
    return UserRepository(db_session)


@pytest.fixture
def created_user(user_repo: UserRepository) -> User:
    """A user that exists in the test database."""
    return user_repo.create(
        name="Alice Smith",
        email="alice@test.com",
        password_hash="hashed_password_abc"
    )


@pytest.fixture
def created_admin(user_repo: UserRepository) -> User:
    return user_repo.create(
        name="Admin User",
        email="admin@test.com",
        password_hash="hashed_admin_pass",
        role="admin"
    )


# ─────────────────────────────────────────
# TESTS
# ─────────────────────────────────────────
class TestUserRepository:
    """Integration tests for UserRepository."""

    def test_create_user(self, user_repo: UserRepository):
        """Creating a user persists it to the database."""
        user = user_repo.create(
            name="Bob Jones",
            email="bob@test.com",
            password_hash="some_hash"
        )

        # ID is assigned by the database
        assert user.id is not None
        assert isinstance(user.id, int)
        assert user.id > 0

        # Data persisted correctly
        assert user.name == "Bob Jones"
        assert user.email == "bob@test.com"
        assert user.role == "user"    # default
        assert user.is_active is True  # default
        assert user.is_verified is False  # default

    def test_create_sets_created_at(self, user_repo: UserRepository, db_session: Session):
        """Database auto-sets created_at timestamp."""
        user = user_repo.create(
            name="Charlie",
            email="charlie@test.com",
            password_hash="hash"
        )

        db_session.commit()  # needed for server_default to appear
        db_session.refresh(user)

        assert user.created_at is not None

    def test_get_by_id_exists(self, user_repo, created_user):
        """Finding existing user by ID returns the user."""
        found = user_repo.get_by_id(created_user.id)

        assert found is not None
        assert found.id == created_user.id
        assert found.email == created_user.email

    def test_get_by_id_not_exists(self, user_repo):
        """Finding non-existent user returns None."""
        found = user_repo.get_by_id(9999)
        assert found is None

    def test_get_by_email(self, user_repo, created_user):
        """Finding user by email returns the correct user."""
        found = user_repo.get_by_email("alice@test.com")
        assert found is not None
        assert found.id == created_user.id

    def test_get_by_email_case_insensitive(self, user_repo, created_user):
        """Email lookup is case-insensitive."""
        found = user_repo.get_by_email("ALICE@TEST.COM")
        assert found is not None
        assert found.id == created_user.id

    def test_get_by_email_not_found(self, user_repo):
        """Finding non-existent email returns None."""
        assert user_repo.get_by_email("nobody@test.com") is None

    def test_get_all(self, user_repo, created_user, created_admin):
        """Getting all users returns all created users."""
        users = user_repo.get_all()

        assert len(users) >= 2  # at least our two test users
        emails = [u.email for u in users]
        assert "alice@test.com" in emails
        assert "admin@test.com" in emails

    def test_get_all_pagination(self, user_repo):
        """Pagination limits results correctly."""
        # Create 5 users
        for i in range(5):
            user_repo.create(
                name=f"Paging User {i}",
                email=f"paging{i}@test.com",
                password_hash="hash"
            )

        page1 = user_repo.get_all(skip=0, limit=3)
        page2 = user_repo.get_all(skip=3, limit=3)

        assert len(page1) == 3
        assert len(page2) >= 2

        # No overlap
        page1_ids = {u.id for u in page1}
        page2_ids = {u.id for u in page2}
        assert page1_ids.isdisjoint(page2_ids)

    def test_update_user(self, user_repo, created_user):
        """Updating a user changes the fields."""
        updated = user_repo.update(
            created_user,
            name="Alice Johnson",
            is_verified=True
        )

        assert updated.name == "Alice Johnson"
        assert updated.is_verified is True
        assert updated.email == "alice@test.com"  # unchanged

        # Verify by fetching again
        fetched = user_repo.get_by_id(created_user.id)
        assert fetched.name == "Alice Johnson"

    def test_delete_user(self, user_repo, created_user):
        """Deleting a user removes them from the database."""
        user_id = created_user.id
        user_repo.delete(created_user)

        # Should not be found anymore
        assert user_repo.get_by_id(user_id) is None

    def test_unique_email_constraint(self, user_repo, created_user):
        """Cannot create two users with the same email."""
        with pytest.raises(IntegrityError):
            user_repo.create(
                name="Duplicate",
                email="alice@test.com",  # already exists!
                password_hash="hash2"
            )

    def test_count(self, user_repo, created_user, created_admin):
        """Count returns correct number of users."""
        count = user_repo.count()
        assert count >= 2

    def test_data_isolation_between_tests(self, user_repo):
        """
        Each test starts with a clean state.
        The created_user from previous test is NOT here
        because each test is wrapped in a rollback.
        """
        users = user_repo.get_all()
        # This test has no fixtures creating users
        # so the database should have 0 users
        assert len(users) == 0
```

---

## 4. 🌐 Integration Tests — Full FastAPI App

Python

```
# src/full_app.py — a complete FastAPI app to test
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from pydantic import BaseModel, EmailStr
from typing import Optional, List
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from passlib.context import CryptContext
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import os


# App configuration
SECRET_KEY = os.getenv("SECRET_KEY", "test-secret-key-32-chars-minimum!!")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

# Database (will be overridden in tests)
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./app.db")
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)


# ─── Database dependency ──────────────────────────────────────
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# ─── Schemas ─────────────────────────────────────────────────
class UserCreate(BaseModel):
    name: str
    email: EmailStr
    password: str


class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    role: str
    is_active: bool
    is_verified: bool

    class Config:
        from_attributes = True


class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    user: UserResponse


class LoginRequest(BaseModel):
    email: str
    password: str


class PostCreate(BaseModel):
    title: str
    content: str
    status: str = "draft"


class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    slug: str
    status: str
    author_id: int
    view_count: int

    class Config:
        from_attributes = True


# ─── Security utilities ──────────────────────────────────────
def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def create_token(data: dict, expire_minutes: int = ACCESS_TOKEN_EXPIRE_MINUTES) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(minutes=expire_minutes)
    to_encode["exp"] = expire
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    # Import here to avoid circular imports
    from tests.integration.conftest import User
    user = db.get(User, int(user_id))
    if user is None:
        raise credentials_exception
    return user


# ─── App ─────────────────────────────────────────────────────
app = FastAPI(title="Integration Test App")


# ─── Auth routes ─────────────────────────────────────────────
@app.post("/auth/register", response_model=TokenResponse, status_code=201)
def register(data: UserCreate, db: Session = Depends(get_db)):
    from tests.integration.conftest import User
    from sqlalchemy import select, func

    # Check duplicate email
    existing = db.execute(
        select(User).where(func.lower(User.email) == data.email.lower())
    ).scalar_one_or_none()

    if existing:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Email already registered"
        )

    # Validate password
    if len(data.password) < 8:
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            detail="Password must be at least 8 characters"
        )

    # Create user
    user = User(
        name=data.name.strip().title(),
        email=data.email.lower(),
        password_hash=hash_password(data.password),
    )
    db.add(user)
    db.flush()
    db.refresh(user)

    # Generate tokens
    access_token = create_token({"sub": str(user.id), "type": "access"})
    refresh_token = create_token(
        {"sub": str(user.id), "type": "refresh"},
        expire_minutes=60 * 24 * 7
    )

    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        user=UserResponse.model_validate(user)
    )


@app.post("/auth/login", response_model=TokenResponse)
def login(data: LoginRequest, db: Session = Depends(get_db)):
    from tests.integration.conftest import User
    from sqlalchemy import select, func

    user = db.execute(
        select(User).where(func.lower(User.email) == data.email.lower())
    ).scalar_one_or_none()

    if not user or not verify_password(data.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password"
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Account is deactivated"
        )

    access_token = create_token({"sub": str(user.id), "type": "access"})
    refresh_token = create_token(
        {"sub": str(user.id), "type": "refresh"},
        expire_minutes=60 * 24 * 7
    )

    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        user=UserResponse.model_validate(user)
    )


@app.get("/auth/me", response_model=UserResponse)
def get_me(current_user=Depends(get_current_user)):
    return current_user


# ─── User routes ─────────────────────────────────────────────
@app.get("/users", response_model=List[UserResponse])
def list_users(
    page: int = 1,
    page_size: int = 20,
    db: Session = Depends(get_db),
    current_user=Depends(get_current_user)
):
    if current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Admin only")

    from tests.integration.conftest import User
    from sqlalchemy import select

    skip = (page - 1) * page_size
    users = db.execute(
        select(User).offset(skip).limit(page_size)
    ).scalars().all()

    return users


@app.patch("/users/{user_id}", response_model=UserResponse)
def update_user(
    user_id: int,
    updates: dict,
    db: Session = Depends(get_db),
    current_user=Depends(get_current_user)
):
    from tests.integration.conftest import User

    if current_user.id != user_id and current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Forbidden")

    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    for key, value in updates.items():
        if hasattr(user, key) and key not in ("id", "password_hash"):
            setattr(user, key, value)

    db.flush()
    db.refresh(user)
    return user


# ─── Post routes ─────────────────────────────────────────────
@app.get("/posts", response_model=List[PostResponse])
def list_posts(
    page: int = 1,
    page_size: int = 20,
    status: Optional[str] = None,
    db: Session = Depends(get_db)
):
    from tests.integration.conftest import Post
    from sqlalchemy import select

    skip = (page - 1) * page_size
    query = select(Post)
    if status:
        query = query.where(Post.status == status)

    posts = db.execute(query.offset(skip).limit(page_size)).scalars().all()
    return posts


@app.post("/posts", response_model=PostResponse, status_code=201)
def create_post(
    data: PostCreate,
    db: Session = Depends(get_db),
    current_user=Depends(get_current_user)
):
    import re
    slug = re.sub(r'[^a-z0-9\s-]', '', data.title.lower())
    slug = re.sub(r'[\s-]+', '-', slug).strip('-')

    from tests.integration.conftest import Post
    post = Post(
        title=data.title,
        content=data.content,
        slug=slug,
        status=data.status,
        author_id=current_user.id,
    )
    db.add(post)
    db.flush()
    db.refresh(post)
    return post


@app.get("/posts/{post_id}", response_model=PostResponse)
def get_post(post_id: int, db: Session = Depends(get_db)):
    from tests.integration.conftest import Post
    from sqlalchemy import update

    post = db.get(Post, post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # Increment view count
    db.execute(
        update(Post)
        .where(Post.id == post_id)
        .values(view_count=Post.view_count + 1)
    )
    db.refresh(post)
    return post
```

---

## 5. 🧪 Integration Tests — Auth Flow

Python

```
# tests/integration/test_auth.py
import pytest
from httpx import AsyncClient
from sqlalchemy.orm import Session
from tests.integration.conftest import User
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


# ─────────────────────────────────────────
# FIXTURES
# ─────────────────────────────────────────
@pytest.fixture
def client(db_session: Session):
    """
    HTTP client that uses the test database.
    Override get_db dependency to use test session.
    """
    from httpx import Client
    from src.full_app import app, get_db

    # Override the database dependency
    app.dependency_overrides[get_db] = lambda: db_session

    with Client(app=app, base_url="http://test") as c:
        yield c

    app.dependency_overrides.clear()


@pytest.fixture
def registered_user(client) -> dict:
    """Register a user and return the response data."""
    response = client.post("/auth/register", json={
        "name": "Alice Smith",
        "email": "alice@test.com",
        "password": "SecurePass1!"
    })
    assert response.status_code == 201
    return response.json()


@pytest.fixture
def auth_headers(registered_user) -> dict:
    """Auth headers for the registered user."""
    return {"Authorization": f"Bearer {registered_user['access_token']}"}


@pytest.fixture
def admin_user(db_session: Session) -> User:
    """Create an admin user directly in DB."""
    admin = User(
        name="Admin User",
        email="admin@test.com",
        password_hash=pwd_context.hash("AdminPass1!"),
        role="admin",
        is_active=True,
    )
    db_session.add(admin)
    db_session.flush()
    db_session.refresh(admin)
    return admin


@pytest.fixture
def admin_headers(client, admin_user) -> dict:
    """Auth headers for the admin user."""
    response = client.post("/auth/login", json={
        "email": "admin@test.com",
        "password": "AdminPass1!"
    })
    assert response.status_code == 200
    return {"Authorization": f"Bearer {response.json()['access_token']}"}


# ─────────────────────────────────────────
# REGISTRATION TESTS
# ─────────────────────────────────────────
class TestRegistration:
    """End-to-end registration flow with real database."""

    def test_register_success(self, client):
        """Successful registration returns tokens and user data."""
        response = client.post("/auth/register", json={
            "name": "Bob Jones",
            "email": "bob@test.com",
            "password": "SecurePass1!"
        })

        assert response.status_code == 201
        data = response.json()

        # Check response structure
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["token_type"] == "bearer"

        # Check user data
        user = data["user"]
        assert user["name"] == "Bob Jones"
        assert user["email"] == "bob@test.com"
        assert user["role"] == "user"
        assert user["is_active"] is True
        assert "password" not in user       # never expose password
        assert "password_hash" not in user  # never expose hash

    def test_register_normalizes_data(self, client):
        """Registration normalizes name and email."""
        response = client.post("/auth/register", json={
            "name": "  alice smith  ",
            "email": "ALICE@TEST.COM",
            "password": "SecurePass1!"
        })

        assert response.status_code == 201
        user = response.json()["user"]
        assert user["name"] == "Alice Smith"  # title-cased + stripped
        assert user["email"] == "alice@test.com"  # lowercased

    def test_register_duplicate_email(self, client, registered_user):
        """Cannot register with existing email."""
        response = client.post("/auth/register", json={
            "name": "Alice 2",
            "email": "alice@test.com",  # already registered
            "password": "SecurePass1!"
        })

        assert response.status_code == 409
        assert "already registered" in response.json()["detail"]

    def test_register_duplicate_email_case_insensitive(self, client, registered_user):
        """Email uniqueness check is case-insensitive."""
        response = client.post("/auth/register", json={
            "name": "Alice 3",
            "email": "ALICE@TEST.COM",  # same as alice@test.com!
            "password": "SecurePass1!"
        })

        assert response.status_code == 409

    def test_register_weak_password(self, client):
        """Weak password rejected."""
        response = client.post("/auth/register", json={
            "name": "Alice",
            "email": "alice@test.com",
            "password": "weak"  # too short
        })

        assert response.status_code == 422

    def test_register_missing_fields(self, client):
        """Missing required fields rejected."""
        response = client.post("/auth/register", json={
            "name": "Alice"
            # missing email and password
        })

        assert response.status_code == 422

    def test_register_invalid_email_format(self, client):
        """Invalid email format rejected."""
        response = client.post("/auth/register", json={
            "name": "Alice",
            "email": "not-an-email",
            "password": "SecurePass1!"
        })

        assert response.status_code == 422

    def test_register_persists_to_database(self, client, db_session: Session):
        """Registration actually saves user to the database."""
        client.post("/auth/register", json={
            "name": "Bob",
            "email": "bob@test.com",
            "password": "SecurePass1!"
        })

        # Directly query the database to verify persistence
        from sqlalchemy import select
        user = db_session.execute(
            select(User).where(User.email == "bob@test.com")
        ).scalar_one_or_none()

        assert user is not None
        assert user.name == "Bob"
        assert user.password_hash != "SecurePass1!"  # must be hashed!
        assert len(user.password_hash) > 50  # bcrypt hash is long


# ─────────────────────────────────────────
# LOGIN TESTS
# ─────────────────────────────────────────
class TestLogin:

    def test_login_success(self, client, registered_user):
        """Correct credentials return valid tokens."""
        response = client.post("/auth/login", json={
            "email": "alice@test.com",
            "password": "SecurePass1!"
        })

        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["user"]["email"] == "alice@test.com"

    def test_login_wrong_password(self, client, registered_user):
        """Wrong password returns 401."""
        response = client.post("/auth/login", json={
            "email": "alice@test.com",
            "password": "WrongPassword1!"
        })

        assert response.status_code == 401
        assert "Incorrect" in response.json()["detail"]

    def test_login_nonexistent_email(self, client):
        """Non-existent email returns 401 (same message for security)."""
        response = client.post("/auth/login", json={
            "email": "nobody@test.com",
            "password": "SecurePass1!"
        })

        assert response.status_code == 401
        # Same error message as wrong password (don't reveal if email exists)
        assert "Incorrect" in response.json()["detail"]

    def test_login_case_insensitive_email(self, client, registered_user):
        """Email login is case-insensitive."""
        response = client.post("/auth/login", json={
            "email": "ALICE@TEST.COM",  # uppercase
            "password": "SecurePass1!"
        })

        assert response.status_code == 200

    def test_login_deactivated_user(self, client, registered_user, db_session: Session):
        """Deactivated user cannot login."""
        # Deactivate the user directly in DB
        user = db_session.execute(
            __import__("sqlalchemy", fromlist=["select"]).select(User)
            .where(User.email == "alice@test.com")
        ).scalar_one()
        user.is_active = False
        db_session.flush()

        response = client.post("/auth/login", json={
            "email": "alice@test.com",
            "password": "SecurePass1!"
        })

        assert response.status_code == 403
        assert "deactivated" in response.json()["detail"].lower()


# ─────────────────────────────────────────
# PROTECTED ENDPOINT TESTS
# ─────────────────────────────────────────
class TestProtectedEndpoints:

    def test_get_me_success(self, client, auth_headers, registered_user):
        """Authenticated user can get their profile."""
        response = client.get("/auth/me", headers=auth_headers)

        assert response.status_code == 200
        data = response.json()
        assert data["email"] == "alice@test.com"
        assert data["name"] == "Alice Smith"

    def test_get_me_no_token(self, client):
        """Request without token returns 401."""
        response = client.get("/auth/me")
        assert response.status_code == 401

    def test_get_me_invalid_token(self, client):
        """Invalid token returns 401."""
        response = client.get(
            "/auth/me",
            headers={"Authorization": "Bearer invalid.token.here"}
        )
        assert response.status_code == 401

    def test_get_me_expired_token(self, client, registered_user):
        """Expired token returns 401."""
        from datetime import datetime, timedelta, timezone
        from jose import jwt

        # Create an already-expired token
        expired_token = jwt.encode(
            {
                "sub": str(registered_user["user"]["id"]),
                "exp": datetime.now(timezone.utc) - timedelta(hours=1)
            },
            "test-secret-key-32-chars-minimum!!",
            algorithm="HS256"
        )

        response = client.get(
            "/auth/me",
            headers={"Authorization": f"Bearer {expired_token}"}
        )
        assert response.status_code == 401

    def test_token_works_for_multiple_requests(self, client, auth_headers):
        """Valid token works for multiple sequential requests."""
        for _ in range(3):
            response = client.get("/auth/me", headers=auth_headers)
            assert response.status_code == 200
```

---

## 6. 🧪 Integration Tests — Full CRUD Flow

Python

```
# tests/integration/test_posts.py
import pytest
from httpx import Client
from sqlalchemy.orm import Session
from tests.integration.conftest import User, Post
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


@pytest.fixture
def client(db_session: Session):
    from src.full_app import app, get_db
    app.dependency_overrides[get_db] = lambda: db_session
    with Client(app=app, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
def user_with_token(client, db_session: Session) -> tuple:
    """Register user and get auth headers."""
    response = client.post("/auth/register", json={
        "name": "Alice Smith",
        "email": "alice@test.com",
        "password": "SecurePass1!"
    })
    data = response.json()
    headers = {"Authorization": f"Bearer {data['access_token']}"}
    return data["user"], headers


@pytest.fixture
def other_user_with_token(client) -> tuple:
    """Second user for authorization tests."""
    response = client.post("/auth/register", json={
        "name": "Bob Jones",
        "email": "bob@test.com",
        "password": "SecurePass1!"
    })
    data = response.json()
    headers = {"Authorization": f"Bearer {data['access_token']}"}
    return data["user"], headers


class TestPostCRUD:
    """End-to-end post CRUD with real database."""

    def test_create_post(self, client, user_with_token):
        """Create a post and verify it's persisted."""
        user, headers = user_with_token

        response = client.post("/posts", json={
            "title": "My First Post",
            "content": "This is the content of my first post.",
            "status": "published"
        }, headers=headers)

        assert response.status_code == 201
        post = response.json()

        assert post["id"] is not None
        assert post["title"] == "My First Post"
        assert post["content"] == "This is the content of my first post."
        assert post["status"] == "published"
        assert post["author_id"] == user["id"]
        assert post["slug"] == "my-first-post"
        assert post["view_count"] == 0

    def test_create_post_generates_slug(self, client, user_with_token):
        """Slug is auto-generated from title."""
        _, headers = user_with_token

        response = client.post("/posts", json={
            "title": "Hello World! How Are You?",
            "content": "Content here"
        }, headers=headers)

        assert response.status_code == 201
        slug = response.json()["slug"]
        # Special chars removed, spaces → hyphens
        assert "hello" in slug
        assert "world" in slug
        assert "!" not in slug
        assert " " not in slug

    def test_create_post_requires_auth(self, client):
        """Cannot create post without authentication."""
        response = client.post("/posts", json={
            "title": "Unauthorized Post",
            "content": "This should fail"
        })

        assert response.status_code == 401

    def test_get_post_increments_views(self, client, user_with_token):
        """Getting a post increments its view count."""
        _, headers = user_with_token

        # Create post
        create_response = client.post("/posts", json={
            "title": "View Count Test",
            "content": "Content for testing view counts."
        }, headers=headers)
        post_id = create_response.json()["id"]

        # Get post 3 times
        for _ in range(3):
            client.get(f"/posts/{post_id}")

        # Check view count
        response = client.get(f"/posts/{post_id}")
        assert response.json()["view_count"] == 4  # 3 + this one

    def test_get_post_not_found(self, client):
        """Getting non-existent post returns 404."""
        response = client.get("/posts/99999")
        assert response.status_code == 404

    def test_list_posts_no_auth(self, client, user_with_token):
        """Public endpoint — anyone can list published posts."""
        _, headers = user_with_token

        # Create some posts
        for i in range(3):
            client.post("/posts", json={
                "title": f"Post {i}",
                "content": f"Content {i}",
                "status": "published"
            }, headers=headers)

        # List without auth
        response = client.get("/posts")
        assert response.status_code == 200
        posts = response.json()
        assert len(posts) >= 3

    def test_list_posts_filter_by_status(self, client, user_with_token):
        """Can filter posts by status."""
        _, headers = user_with_token

        # Create published and draft posts
        client.post("/posts", json={
            "title": "Published Post",
            "content": "Content",
            "status": "published"
        }, headers=headers)
        client.post("/posts", json={
            "title": "Draft Post",
            "content": "Content",
            "status": "draft"
        }, headers=headers)

        # Filter by published
        response = client.get("/posts?status=published")
        posts = response.json()
        assert all(p["status"] == "published" for p in posts)

        # Filter by draft
        response = client.get("/posts?status=draft")
        posts = response.json()
        assert all(p["status"] == "draft" for p in posts)

    def test_list_posts_pagination(self, client, user_with_token):
        """Pagination works correctly."""
        _, headers = user_with_token

        # Create 10 posts
        for i in range(10):
            client.post("/posts", json={
                "title": f"Paginate Post {i}",
                "content": f"Content {i}"
            }, headers=headers)

        page1 = client.get("/posts?page=1&page_size=5").json()
        page2 = client.get("/posts?page=2&page_size=5").json()

        assert len(page1) == 5
        assert len(page2) == 5

        # No overlap
        p1_ids = {p["id"] for p in page1}
        p2_ids = {p["id"] for p in page2}
        assert p1_ids.isdisjoint(p2_ids)


class TestDataPersistence:
    """Verify data actually persists across multiple requests."""

    def test_register_then_login_then_access(self, client):
        """Complete user journey: register → login → access profile."""

        # 1. Register
        register_resp = client.post("/auth/register", json={
            "name": "Journey User",
            "email": "journey@test.com",
            "password": "SecurePass1!"
        })
        assert register_resp.status_code == 201
        user_id = register_resp.json()["user"]["id"]

        # 2. Login (different request — simulates separate session)
        login_resp = client.post("/auth/login", json={
            "email": "journey@test.com",
            "password": "SecurePass1!"
        })
        assert login_resp.status_code == 200
        token = login_resp.json()["access_token"]

        # 3. Access profile
        profile_resp = client.get(
            "/auth/me",
            headers={"Authorization": f"Bearer {token}"}
        )
        assert profile_resp.status_code == 200
        profile = profile_resp.json()
        assert profile["id"] == user_id
        assert profile["email"] == "journey@test.com"

    def test_create_post_visible_in_list(self, client, user_with_token):
        """Created posts appear in listing."""
        _, headers = user_with_token

        # Create a post
        create_resp = client.post("/posts", json={
            "title": "Unique Test Post XYZ",
            "content": "Content for persistence test.",
            "status": "published"
        }, headers=headers)
        assert create_resp.status_code == 201
        post_id = create_resp.json()["id"]

        # List posts — the new post should appear
        list_resp = client.get("/posts?status=published")
        post_ids = [p["id"] for p in list_resp.json()]
        assert post_id in post_ids

    def test_database_state_isolated_per_test(self, client):
        """
        Each test gets a fresh database state.
        The posts from previous tests are NOT here.
        """
        response = client.get("/posts")
        posts = response.json()
        # This test creates no posts, so list should be empty
        assert len(posts) == 0


class TestTransactions:
    """Verify transactional behavior."""

    def test_registration_is_atomic(self, client, db_session: Session):
        """
        Registration either fully succeeds or fails.
        Tests atomicity of the registration flow.
        """
        from sqlalchemy import select

        # Force an error by providing duplicate email
        # (after creating first user)
        client.post("/auth/register", json={
            "name": "First User",
            "email": "atomic@test.com",
            "password": "SecurePass1!"
        })

        # This should fail
        failed_resp = client.post("/auth/register", json={
            "name": "Second User",
            "email": "atomic@test.com",  # duplicate!
            "password": "SecurePass1!"
        })

        assert failed_resp.status_code == 409

        # Only ONE user with this email should exist
        count = len(
            db_session.execute(
                select(User).where(User.email == "atomic@test.com")
            ).scalars().all()
        )
        assert count == 1  # not 2!
```

---

## 7. 📊 Testing with TestContainers (Concept)

Python

```
"""
testcontainers: Start real PostgreSQL in Docker for tests.

Pro: True production parity
Con: Requires Docker, slower startup

Install: pip install testcontainers
"""

# tests/integration/conftest_postgres.py (optional)
# Use this instead of conftest.py when you want real PostgreSQL

import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker


# Conceptual example — requires Docker
def get_postgres_container():
    """
    Start a real PostgreSQL container for testing.
    Requires: Docker running + testcontainers installed
    """
    try:
        from testcontainers.postgres import PostgresContainer

        container = PostgresContainer("postgres:16-alpine")
        container.start()
        return container
    except ImportError:
        pytest.skip("testcontainers not installed")


@pytest.fixture(scope="session")
def postgres_url():
    """
    Get a real PostgreSQL URL for integration tests.
    Uses TestContainers to spin up a fresh PostgreSQL.
    """
    try:
        from testcontainers.postgres import PostgresContainer

        with PostgresContainer("postgres:16-alpine") as postgres:
            url = postgres.get_connection_url()
            print(f"\nUsing PostgreSQL: {url}")
            yield url

    except ImportError:
        # Fall back to SQLite if testcontainers not available
        yield "sqlite:///./test_fallback.db"


@pytest.fixture(scope="session")
def pg_engine(postgres_url):
    """PostgreSQL engine with real PostgreSQL."""
    from tests.integration.conftest import Base

    engine = create_engine(postgres_url)
    Base.metadata.create_all(engine)

    yield engine

    Base.metadata.drop_all(engine)
    engine.dispose()
```

---

## 8. ⚡ Async Integration Tests

Python

```
# tests/integration/test_async_integration.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import AsyncSession

from tests.integration.conftest import User, Post


@pytest.fixture(scope="session")
def event_loop_policy():
    """Use default event loop policy."""
    import asyncio
    return asyncio.DefaultEventLoopPolicy()


class TestAsyncIntegration:
    """
    Async integration tests with real async database.
    """

    @pytest.mark.asyncio
    async def test_async_repository_create(self, async_session: AsyncSession):
        """Test async repository with real database."""
        from sqlalchemy import select

        user = User(
            name="Async Test User",
            email="async@test.com",
            password_hash="hash_here",
        )
        async_session.add(user)
        await async_session.flush()
        await async_session.refresh(user)

        assert user.id is not None
        assert user.name == "Async Test User"

        # Verify with a query
        result = await async_session.execute(
            select(User).where(User.email == "async@test.com")
        )
        found = result.scalar_one_or_none()
        assert found is not None
        assert found.id == user.id

    @pytest.mark.asyncio
    async def test_async_multiple_operations(self, async_session: AsyncSession):
        """Test multiple async operations in sequence."""
        from sqlalchemy import select, func

        # Create multiple users
        users = [
            User(
                name=f"User {i}",
                email=f"user{i}@test.com",
                password_hash="hash"
            )
            for i in range(5)
        ]
        async_session.add_all(users)
        await async_session.flush()

        # Count them
        count = await async_session.execute(
            select(func.count()).select_from(User)
        )
        assert count.scalar_one() == 5

        # Query specific one
        result = await async_session.execute(
            select(User).where(User.email == "user2@test.com")
        )
        user2 = result.scalar_one_or_none()
        assert user2 is not None
        assert user2.name == "User 2"

    @pytest.mark.asyncio
    async def test_concurrent_database_operations(self, async_session: AsyncSession):
        """
        Test concurrent async database operations.
        Shows real async value — multiple ops without blocking.
        """
        import asyncio

        async def create_user(email: str) -> User:
            user = User(
                name=f"Concurrent {email}",
                email=email,
                password_hash="hash"
            )
            async_session.add(user)
            await async_session.flush()
            return user

        # Create 5 users "concurrently"
        # (they share the same session, so actually sequential
        # but demonstrates the async pattern)
        users = await asyncio.gather(*[
            create_user(f"concurrent{i}@test.com")
            for i in range(5)
        ])

        assert len(users) == 5
        assert all(u.id is not None for u in users)
        ids = [u.id for u in users]
        assert len(set(ids)) == 5  # all unique
```

---

## 9. 🔧 Test Utilities & Helpers

Python

```
# tests/utils.py — shared test utilities
from typing import Any, Dict, Optional
from sqlalchemy.orm import Session
from httpx import Client
from tests.integration.conftest import User, Post
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


class TestHelper:
    """
    Helper methods for integration tests.
    Reduces boilerplate in test files.
    """

    def __init__(self, client: Client, db: Session):
        self.client = client
        self.db = db

    def register_user(
        self,
        name: str = "Test User",
        email: str = "test@test.com",
        password: str = "SecurePass1!",
        role: str = "user"
    ) -> Dict[str, Any]:
        """Register a user via API and return response data."""
        response = self.client.post("/auth/register", json={
            "name": name,
            "email": email,
            "password": password,
        })
        assert response.status_code == 201, f"Registration failed: {response.text}"
        data = response.json()
        if role != "user":
            # Update role directly in DB
            from sqlalchemy import select
            user = self.db.execute(
                select(User).where(User.email == email.lower())
            ).scalar_one()
            user.role = role
            self.db.flush()
        return data

    def auth_headers(self, email: str, password: str) -> Dict[str, str]:
        """Login and return auth headers."""
        response = self.client.post("/auth/login", json={
            "email": email,
            "password": password
        })
        assert response.status_code == 200, f"Login failed: {response.text}"
        token = response.json()["access_token"]
        return {"Authorization": f"Bearer {token}"}

    def create_user_in_db(
        self,
        name: str = "DB User",
        email: str = "dbuser@test.com",
        role: str = "user",
    ) -> User:
        """Create user directly in database (faster than API)."""
        user = User(
            name=name,
            email=email.lower(),
            password_hash=pwd_context.hash("SecurePass1!"),
            role=role,
            is_active=True,
        )
        self.db.add(user)
        self.db.flush()
        self.db.refresh(user)
        return user

    def create_post_in_db(
        self,
        title: str = "Test Post",
        author_id: int = 1,
        status: str = "published",
    ) -> Post:
        """Create post directly in database."""
        import re
        slug = re.sub(r'[^a-z0-9-]', '-', title.lower())
        slug = re.sub(r'-+', '-', slug).strip('-')

        post = Post(
            title=title,
            content=f"Content of {title}",
            slug=slug,
            status=status,
            author_id=author_id,
        )
        self.db.add(post)
        self.db.flush()
        self.db.refresh(post)
        return post

    def assert_response_has_fields(
        self,
        response_data: dict,
        expected: dict
    ) -> None:
        """Assert response contains expected field values."""
        for key, value in expected.items():
            assert key in response_data, f"Missing field: {key}"
            assert response_data[key] == value, (
                f"Field {key}: expected {value!r}, got {response_data[key]!r}"
            )


@pytest.fixture
def helper(client, db_session):
    """Test helper fixture."""
    return TestHelper(client, db_session)


# Usage in tests:
# def test_something(helper):
#     user = helper.register_user(email="test@test.com")
#     headers = helper.auth_headers("test@test.com", "SecurePass1!")
#     post = helper.create_post_in_db(author_id=user["user"]["id"])
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│                    INTEGRATION TESTING                         │
│                                                                │
│  KEY CONCEPT: Test REAL components working together           │
│  Database: real (SQLite for speed, PostgreSQL for parity)     │
│  External APIs: still mocked                                  │
│                                                                │
│  ISOLATION TECHNIQUE:                                          │
│  ┌──────────────────────────────────────┐                     │
│  │  BEGIN TRANSACTION                   │                     │
│  │    ↓                                 │                     │
│  │  yield session (test runs here)      │                     │
│  │    ↓                                 │                     │
│  │  ROLLBACK                            │                     │
│  └──────────────────────────────────────┘                     │
│  Each test: clean DB state, fast, no cleanup needed          │
│                                                                │
│  FIXTURE HIERARCHY:                                            │
│  session scope: engine (once per session)                     │
│  session scope: tables created once                           │
│  function scope: transaction per test + rollback             │
│  function scope: HTTP client with overridden DB dep          │
│                                                                │
│  DEPENDENCY OVERRIDE:                                          │
│  app.dependency_overrides[get_db] = lambda: test_session     │
│  ← inject test DB into FastAPI                                │
│  app.dependency_overrides.clear()  ← cleanup after           │
│                                                                │
│  WHAT TO TEST:                                                 │
│  ✅ Full auth flow (register → login → access)               │
│  ✅ Data persistence across requests                          │
│  ✅ Database constraints (unique email, FK, etc.)             │
│  ✅ Pagination, filtering, sorting                            │
│  ✅ Authorization (who can do what)                           │
│  ✅ Transaction atomicity                                      │
│  ✅ Error responses with correct status codes                 │
│                                                                │
│  OPTIONS:                                                      │
│  SQLite      → fast, no setup, not 100% PostgreSQL parity    │
│  PostgreSQL  → real parity, needs running instance            │
│  TestContainers → Docker PostgreSQL, full isolation           │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  What is the difference between unit and integration tests?
    When do you need integration tests?
2.  What is the transaction rollback pattern and why is it faster
    than creating/dropping tables per test?
3.  How do you override FastAPI dependencies in tests?
4.  What is app.dependency_overrides used for?
5.  What does scope="session" mean on a fixture?
    How does it differ from scope="function"?
6.  Why do we use SQLite for integration tests sometimes?
    What are its limitations?
7.  What is TestContainers and when would you use it?
8.  How do you test database constraints in integration tests?
9.  What does db_session.flush() do vs db_session.commit()?
10. How do you test that a transaction rolls back on failure?
11. What is the difference between AsyncClient and TestClient?
12. Why do integration tests need to test data persistence
    specifically?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Comment System Integration Tests
# Add a comments system to the blog app:
# POST /posts/{post_id}/comments → create comment (auth required)
# GET  /posts/{post_id}/comments → list comments (public)
# PATCH /comments/{id}           → edit comment (author only)
# DELETE /comments/{id}          → delete (author or admin)
#
# Write integration tests for:
# - Create comment, appears in list
# - Nested comments (reply to comment)
# - Edit own comment
# - Cannot edit other's comment
# - Delete removes from list
# - Comment count tracked on post

# Exercise 2: User Management Integration Tests
# Test the full user management lifecycle:
# - Register → profile is correct
# - Update profile → changes persisted
# - Change password → can login with new, not old
# - Deactivate → cannot login
# - Admin promotes user → role updated
# - Delete account → gone from DB
# Test isolation: each test starts fresh

# Exercise 3: Search and Filter Tests
# Add search to your posts:
# GET /posts?q=python        → search in title and content
# GET /posts?author=alice    → filter by author name
# GET /posts?tag=python      → filter by tag
# GET /posts?sort=views      → sort by view count
#
# Write integration tests verifying:
# - Search finds matching posts
# - Search is case-insensitive
# - Multiple filters work together
# - Sorting is correct
# - Empty results handled gracefully

# Exercise 4: Transaction Tests
# Write tests that verify transactional behavior:
# Scenario: Place an order
#   1. Validate cart items exist
#   2. Reserve inventory
#   3. Create order
#   4. Process payment
# If step 4 fails → steps 2 and 3 should be rolled back
# Test: inventory is restored when payment fails
# Test: order doesn't exist when payment fails

# Exercise 5: Concurrent Access Tests
# Test race conditions and concurrent access:
# - Two users register with same email simultaneously
#   → only one should succeed
# - Multiple requests increment view count
#   → count should be exact
# - User updates profile and reads simultaneously
#   → always consistent data
# Use threading or asyncio to simulate concurrency
```

---

## ✅ Phase 6.3 Complete!

**You now know:**

text

```
✅ Integration test architecture
✅ SQLite in-memory for fast integration tests
✅ Transaction rollback pattern for test isolation
✅ Async database integration testing
✅ FastAPI dependency override for testing
✅ Test HTTP client with ASGITransport
✅ Testing full auth flows (register → login → access)
✅ Testing data persistence across requests
✅ Testing database constraints (unique, FK)
✅ Testing authorization (who can do what)
✅ TestContainers concept (real PostgreSQL in Docker)
✅ Test helpers to reduce boilerplate
✅ Fixture hierarchy (session → module → function scope)
```