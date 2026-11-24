```
Phase 5.1: You understand asyncio fundamentals.
Phase 5.2: You apply async to databases — the most common
           I/O operation in any backend.

The problem with sync DB in async FastAPI:

async def get_users():              # async endpoint
    users = db.query(User).all()    # SYNC — blocks event loop!
    return users
    # While this query runs (50ms), NO other requests can run.
    # 1000 concurrent users = disaster.

The solution: async database drivers

asyncpg:          PostgreSQL async driver (fastest Python PG driver)
SQLAlchemy 2.0:   async ORM (uses asyncpg under the hood)
Alembic:          still sync for migrations (that's fine)
databases:        lightweight async query library

Real production numbers:
  Sync:  ~200 requests/second
  Async: ~2000+ requests/second
         with same hardware
```

---

## Setup

Bash

```
cd ~/projects
mkdir async_db
cd async_db
python3 -m venv venv
source venv/bin/activate

pip install fastapi "uvicorn[standard]" \
    sqlalchemy[asyncio] asyncpg \
    alembic python-dotenv \
    pydantic pydantic-settings \
    greenlet   # required by SQLAlchemy async

# Create database
createdb -U postgres async_blog_db

# Project structure
mkdir -p app/{core,models,repositories,services,api/v1}
touch app/__init__.py
touch app/core/{config.py,database.py,deps.py}
touch app/models/{base.py,user.py,post.py}
touch app/repositories/{base.py,user_repo.py,post_repo.py}
touch app/services/{user_service.py,post_service.py}
touch app/api/v1/{router.py,users.py,posts.py}
touch app/main.py
touch .env
code .
```

Bash

```
# .env
DATABASE_URL=postgresql+asyncpg://myapp_user:myapp_password@localhost:5432/async_blog_db
DATABASE_SYNC_URL=postgresql://myapp_user:myapp_password@localhost:5432/async_blog_db
```

---

## 1. 🔌 asyncpg — Raw Async PostgreSQL

Python

```
# Understanding asyncpg directly before SQLAlchemy
# asyncpg is the fastest Python PostgreSQL driver

import asyncio
import asyncpg
from typing import Optional, List, Dict, Any


# ─────────────────────────────────────────
# DIRECT CONNECTION (simple scripts)
# ─────────────────────────────────────────
async def direct_connection_demo():
    """Use a single connection for simple scripts."""

    conn = await asyncpg.connect(
        "postgresql://myapp_user:myapp_password@localhost:5432/async_blog_db"
    )

    try:
        # Create table
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS demo_users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
            )
        """)

        # INSERT with RETURNING
        row = await conn.fetchrow(
            "INSERT INTO demo_users (name, email) VALUES ($1, $2) RETURNING *",
            "Alice Smith",
            "alice@test.com"
        )
        print(f"Created: id={row['id']}, name={row['name']}")

        # SELECT one row
        user = await conn.fetchrow(
            "SELECT * FROM demo_users WHERE email = $1",
            "alice@test.com"
        )
        print(f"Found: {dict(user)}")

        # SELECT multiple rows
        users = await conn.fetch("SELECT * FROM demo_users LIMIT 10")
        print(f"All users: {[dict(u) for u in users]}")

        # Scalar value
        count = await conn.fetchval("SELECT COUNT(*) FROM demo_users")
        print(f"Total: {count}")

        # UPDATE
        await conn.execute(
            "UPDATE demo_users SET name = $1 WHERE id = $2",
            "Alice Johnson",
            user["id"]
        )

        # DELETE
        await conn.execute(
            "DELETE FROM demo_users WHERE email = $1",
            "alice@test.com"
        )

    finally:
        await conn.close()    # always close!


# ─────────────────────────────────────────
# CONNECTION POOL (production standard)
# ─────────────────────────────────────────
async def pool_demo():
    """
    Connection pool: pre-create connections, reuse them.
    Critical for production — creating a new connection
    for every request would be too slow.
    """

    # Create pool at startup
    pool = await asyncpg.create_pool(
        "postgresql://myapp_user:myapp_password@localhost:5432/async_blog_db",
        min_size=5,          # minimum connections kept alive
        max_size=20,         # maximum connections allowed
        max_inactive_connection_lifetime=300,   # close idle after 5min
        command_timeout=60,  # query timeout
    )

    try:
        # Use pool — automatically borrows/returns connections
        async with pool.acquire() as conn:
            result = await conn.fetchval("SELECT version()")
            print(f"PostgreSQL: {result[:50]}...")

        # Concurrent queries — each gets its own connection from pool
        async def fetch_user(user_id: int) -> Optional[Dict]:
            async with pool.acquire() as conn:
                row = await conn.fetchrow(
                    "SELECT * FROM demo_users WHERE id = $1",
                    user_id
                )
                return dict(row) if row else None

        # All run concurrently using different pool connections
        results = await asyncio.gather(
            fetch_user(1),
            fetch_user(2),
            fetch_user(3),
        )
        print(f"Concurrent results: {results}")

    finally:
        await pool.close()   # close all connections at shutdown


# ─────────────────────────────────────────
# TRANSACTIONS with asyncpg
# ─────────────────────────────────────────
async def transaction_demo():
    pool = await asyncpg.create_pool(
        "postgresql://myapp_user:myapp_password@localhost:5432/async_blog_db"
    )

    async with pool.acquire() as conn:
        # Transaction context manager
        async with conn.transaction():
            try:
                await conn.execute(
                    "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
                    100.0, 1
                )
                await conn.execute(
                    "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
                    100.0, 2
                )
                # Both commit together if no exception
            except Exception:
                # Auto-rolled back by context manager
                raise

        # Savepoints for partial rollback
        async with conn.transaction():
            await conn.execute("INSERT INTO demo_users (name, email) VALUES ($1, $2)",
                             "Alice", "alice2@test.com")

            savepoint = conn.transaction()
            async with savepoint:
                try:
                    await conn.execute("INSERT INTO demo_users (name, email) VALUES ($1, $2)",
                                     "Bob", "alice2@test.com")  # duplicate email!
                except asyncpg.UniqueViolationError:
                    print("Duplicate caught — savepoint rolled back, outer transaction continues")

    await pool.close()


# Run demos
asyncio.run(direct_connection_demo())
```

---

## 2. 🗺️ SQLAlchemy Async — The Production Way

### Async Engine & Session

Python

```
# app/core/database.py
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    AsyncEngine,
    create_async_engine,
    async_sessionmaker,
)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import event
from typing import AsyncGenerator
import logging
from app.core.config import settings

logger = logging.getLogger(__name__)


# ─────────────────────────────────────────
# CREATE ASYNC ENGINE
# ─────────────────────────────────────────
engine: AsyncEngine = create_async_engine(
    settings.DATABASE_URL,           # must use postgresql+asyncpg://

    # Pool configuration
    pool_size=settings.DB_POOL_SIZE,            # permanent connections
    max_overflow=settings.DB_MAX_OVERFLOW,      # extra connections
    pool_pre_ping=True,                         # test before use
    pool_recycle=3600,                          # recycle after 1 hour
    pool_timeout=30,                            # wait up to 30s for connection

    # Debugging (shows SQL in development)
    echo=settings.DB_ECHO,

    # asyncpg-specific options
    connect_args={
        "server_settings": {
            "application_name": settings.APP_NAME,
        }
    },
)


# ─────────────────────────────────────────
# ASYNC SESSION FACTORY
# ─────────────────────────────────────────
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,   # IMPORTANT: don't expire after commit
    # Without this, accessing attrs after commit hits DB again!
    autocommit=False,
    autoflush=False,
)


# ─────────────────────────────────────────
# BASE MODEL
# ─────────────────────────────────────────
class Base(DeclarativeBase):
    """Base for all SQLAlchemy models."""
    pass


# ─────────────────────────────────────────
# DEPENDENCY: Async session per request
# ─────────────────────────────────────────
async def get_async_db() -> AsyncGenerator[AsyncSession, None]:
    """
    FastAPI dependency that provides an async database session.

    Usage:
        async def my_route(db: AsyncSession = Depends(get_async_db)):
            ...
    """
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()


# ─────────────────────────────────────────
# DATABASE INITIALIZATION
# ─────────────────────────────────────────
async def init_db() -> None:
    """Create all tables. Called at application startup."""
    async with engine.begin() as conn:
        # Create all tables defined in models
        await conn.run_sync(Base.metadata.create_all)
    logger.info("Database tables initialized")


async def drop_db() -> None:
    """Drop all tables. Useful for testing."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    logger.info("Database tables dropped")


async def close_db() -> None:
    """Close the engine and all connections. Called at shutdown."""
    await engine.dispose()
    logger.info("Database connections closed")
```

### Async Models

Python

```
# app/models/base.py
from sqlalchemy import Column, DateTime, Integer, func
from sqlalchemy.orm import declared_attr
from app.core.database import Base


class TimestampMixin:
    """Add created_at and updated_at to any model."""

    created_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    updated_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )


class SoftDeleteMixin:
    """Add soft delete capability."""

    deleted_at = Column(DateTime(timezone=True), nullable=True)

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None
```

Python

```
# app/models/user.py
from sqlalchemy import (
    Column, Integer, String, Boolean, Text,
    DateTime, CheckConstraint, Index
)
from sqlalchemy.orm import relationship
from app.core.database import Base
from app.models.base import TimestampMixin, SoftDeleteMixin


class User(Base, TimestampMixin, SoftDeleteMixin):
    """User model."""
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), nullable=False, unique=True, index=True)
    password_hash = Column(String(255), nullable=False)
    bio = Column(Text, nullable=True)
    role = Column(String(20), nullable=False, server_default="user")
    is_active = Column(Boolean, nullable=False, server_default="true")
    is_verified = Column(Boolean, nullable=False, server_default="false")

    __table_args__ = (
        CheckConstraint(
            "role IN ('user', 'admin', 'moderator')",
            name="check_user_role"
        ),
        Index("idx_users_email_active", "email",
              postgresql_where="deleted_at IS NULL"),
        Index("idx_users_role_active", "role", "is_active"),
    )

    # Relationships
    posts = relationship(
        "Post",
        back_populates="author",
        cascade="all, delete-orphan",
        lazy="select",    # sync lazy load — but we won't use this
    )

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

Python

```
# app/models/post.py
from sqlalchemy import (
    Column, Integer, String, Text, Boolean,
    DateTime, ForeignKey, CheckConstraint, Index
)
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import ARRAY
from sqlalchemy import String as SAString
from app.core.database import Base
from app.models.base import TimestampMixin, SoftDeleteMixin


class Post(Base, TimestampMixin, SoftDeleteMixin):
    """Post model."""
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(500), nullable=False)
    content = Column(Text, nullable=False)
    slug = Column(String(500), unique=True, index=True)
    status = Column(String(20), nullable=False, server_default="draft")
    tags = Column(ARRAY(SAString), nullable=False, server_default="{}")
    view_count = Column(Integer, nullable=False, server_default="0")
    like_count = Column(Integer, nullable=False, server_default="0")
    published_at = Column(DateTime(timezone=True), nullable=True)

    author_id = Column(
        Integer,
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )

    __table_args__ = (
        CheckConstraint(
            "status IN ('draft', 'published', 'archived')",
            name="check_post_status"
        ),
        Index("idx_posts_author_status", "author_id", "status"),
        Index(
            "idx_posts_published",
            "published_at",
            postgresql_where="status = 'published'"
        ),
    )

    # Relationship
    author = relationship("User", back_populates="posts", lazy="select")

    def __repr__(self) -> str:
        return f"<Post id={self.id} title={self.title!r}>"
```

---

## 3. 📁 Async Repositories

Python

```
# app/repositories/base.py
from typing import TypeVar, Generic, Type, Optional, List, Any, Sequence
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, update, delete
from sqlalchemy.orm import DeclarativeBase

ModelType = TypeVar("ModelType")


class AsyncBaseRepository(Generic[ModelType]):
    """
    Generic async repository with CRUD operations.
    All operations are async — non-blocking.
    """

    def __init__(self, model: Type[ModelType], session: AsyncSession):
        self.model = model
        self.session = session

    # ─────────────────────────────────────────
    # CREATE
    # ─────────────────────────────────────────
    async def create(self, **kwargs) -> ModelType:
        """Create and flush (not commit) a new instance."""
        instance = self.model(**kwargs)
        self.session.add(instance)
        await self.session.flush()      # assigns ID without committing
        await self.session.refresh(instance)   # load server defaults
        return instance

    async def bulk_create(self, records: List[dict]) -> List[ModelType]:
        """Efficiently create multiple records."""
        instances = [self.model(**data) for data in records]
        self.session.add_all(instances)
        await self.session.flush()
        return instances

    # ─────────────────────────────────────────
    # READ
    # ─────────────────────────────────────────
    async def get_by_id(self, id: int) -> Optional[ModelType]:
        """Get record by primary key. O(1) with PK index."""
        return await self.session.get(self.model, id)

    async def get_one(self, **filters) -> Optional[ModelType]:
        """Get single record matching filters."""
        stmt = select(self.model)
        for key, value in filters.items():
            stmt = stmt.where(getattr(self.model, key) == value)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_all(
        self,
        skip: int = 0,
        limit: int = 100,
        order_by=None,
        **filters
    ) -> List[ModelType]:
        """Get multiple records with filtering and pagination."""
        stmt = select(self.model)
        for key, value in filters.items():
            if value is not None:
                stmt = stmt.where(getattr(self.model, key) == value)
        if order_by is not None:
            stmt = stmt.order_by(order_by)
        stmt = stmt.offset(skip).limit(limit)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def count(self, **filters) -> int:
        """Count records matching filters."""
        stmt = select(func.count()).select_from(self.model)
        for key, value in filters.items():
            if value is not None:
                stmt = stmt.where(getattr(self.model, key) == value)
        result = await self.session.execute(stmt)
        return result.scalar_one()

    async def exists(self, **filters) -> bool:
        """Check if any record matches filters."""
        stmt = select(func.count()).select_from(self.model)
        for key, value in filters.items():
            stmt = stmt.where(getattr(self.model, key) == value)
        result = await self.session.execute(stmt)
        return result.scalar_one() > 0

    # ─────────────────────────────────────────
    # UPDATE
    # ─────────────────────────────────────────
    async def update(self, instance: ModelType, **updates) -> ModelType:
        """Update fields on an instance."""
        for key, value in updates.items():
            if hasattr(instance, key) and value is not None:
                setattr(instance, key, value)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def bulk_update(self, filter_by: dict, updates: dict) -> int:
        """
        Update multiple records at once.
        More efficient than loading each and updating individually.
        """
        stmt = (
            update(self.model)
            .where(*[
                getattr(self.model, k) == v
                for k, v in filter_by.items()
            ])
            .values(**updates)
            .execution_options(synchronize_session="fetch")
        )
        result = await self.session.execute(stmt)
        return result.rowcount

    # ─────────────────────────────────────────
    # DELETE
    # ─────────────────────────────────────────
    async def delete(self, instance: ModelType) -> None:
        """Delete a single instance."""
        await self.session.delete(instance)
        await self.session.flush()

    async def delete_by_id(self, id: int) -> bool:
        """Delete by primary key. Returns True if deleted."""
        instance = await self.get_by_id(id)
        if not instance:
            return False
        await self.delete(instance)
        return True

    async def bulk_delete(self, **filters) -> int:
        """Delete multiple records matching filters."""
        stmt = delete(self.model)
        for key, value in filters.items():
            stmt = stmt.where(getattr(self.model, key) == value)
        result = await self.session.execute(stmt)
        return result.rowcount
```

Python

```
# app/repositories/user_repo.py
from typing import Optional, List, Tuple
from sqlalchemy import select, func, or_, and_
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime, timezone

from app.models.user import User
from app.repositories.base import AsyncBaseRepository


class UserRepository(AsyncBaseRepository[User]):
    """Async repository for User model."""

    def __init__(self, session: AsyncSession):
        super().__init__(User, session)

    # ─────────────────────────────────────────
    # USER-SPECIFIC QUERIES
    # ─────────────────────────────────────────
    async def get_active_by_id(self, user_id: int) -> Optional[User]:
        """Get active (non-deleted) user by ID."""
        stmt = select(User).where(
            and_(
                User.id == user_id,
                User.deleted_at.is_(None)
            )
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> Optional[User]:
        """Find user by email (case-insensitive)."""
        stmt = select(User).where(
            and_(
                func.lower(User.email) == email.lower().strip(),
                User.deleted_at.is_(None)
            )
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def email_taken(
        self,
        email: str,
        exclude_id: Optional[int] = None
    ) -> bool:
        """Check if email is already registered."""
        stmt = select(func.count()).select_from(User).where(
            func.lower(User.email) == email.lower()
        )
        if exclude_id:
            stmt = stmt.where(User.id != exclude_id)
        result = await self.session.execute(stmt)
        return result.scalar_one() > 0

    async def search(
        self,
        query: str = "",
        role: Optional[str] = None,
        is_active: Optional[bool] = None,
        skip: int = 0,
        limit: int = 20,
    ) -> Tuple[List[User], int]:
        """
        Search users with filters.
        Returns (users, total_count) in single round trip.
        """
        # Base filter: not deleted
        base_filter = User.deleted_at.is_(None)

        # Additional filters
        filters = [base_filter]
        if query:
            term = f"%{query}%"
            filters.append(
                or_(User.name.ilike(term), User.email.ilike(term))
            )
        if role:
            filters.append(User.role == role)
        if is_active is not None:
            filters.append(User.is_active == is_active)

        # Count query
        count_stmt = (
            select(func.count())
            .select_from(User)
            .where(and_(*filters))
        )

        # Data query
        data_stmt = (
            select(User)
            .where(and_(*filters))
            .order_by(User.created_at.desc())
            .offset(skip)
            .limit(limit)
        )

        # Execute both in parallel! Saves one round trip.
        count_result, data_result = await asyncio.gather(
            self.session.execute(count_stmt),
            self.session.execute(data_stmt)
        )

        total = count_result.scalar_one()
        users = list(data_result.scalars().all())

        return users, total

    async def soft_delete(self, user: User) -> User:
        """Soft delete: mark as deleted without removing."""
        user.deleted_at = datetime.now(timezone.utc)
        user.is_active = False
        await self.session.flush()
        return user

    async def get_stats_by_role(self) -> List[dict]:
        """Get user counts grouped by role."""
        stmt = (
            select(
                User.role,
                func.count(User.id).label("total"),
                func.count(User.id).filter(
                    User.is_active == True
                ).label("active")
            )
            .where(User.deleted_at.is_(None))
            .group_by(User.role)
        )
        result = await self.session.execute(stmt)
        return [
            {"role": r.role, "total": r.total, "active": r.active}
            for r in result.all()
        ]

    async def get_by_ids(self, user_ids: List[int]) -> List[User]:
        """Fetch multiple users by IDs in one query."""
        stmt = select(User).where(
            and_(
                User.id.in_(user_ids),
                User.deleted_at.is_(None)
            )
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

Python

```
# app/repositories/post_repo.py
import asyncio
from typing import Optional, List, Tuple
from sqlalchemy import select, func, or_, and_, update
from sqlalchemy.orm import selectinload, joinedload
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime, timezone, timedelta

from app.models.post import Post
from app.models.user import User
from app.repositories.base import AsyncBaseRepository


class PostRepository(AsyncBaseRepository[Post]):
    """Async repository for Post model."""

    def __init__(self, session: AsyncSession):
        super().__init__(Post, session)

    async def get_with_author(self, post_id: int) -> Optional[Post]:
        """
        Get post with author loaded in same query.
        Uses joinedload to avoid N+1.
        """
        stmt = (
            select(Post)
            .options(joinedload(Post.author))   # JOIN in single query
            .where(
                and_(
                    Post.id == post_id,
                    Post.deleted_at.is_(None)
                )
            )
        )
        result = await self.session.execute(stmt)
        return result.unique().scalar_one_or_none()

    async def get_by_slug(self, slug: str) -> Optional[Post]:
        """Find post by URL slug."""
        stmt = (
            select(Post)
            .options(joinedload(Post.author))
            .where(
                and_(
                    Post.slug == slug,
                    Post.deleted_at.is_(None)
                )
            )
        )
        result = await self.session.execute(stmt)
        return result.unique().scalar_one_or_none()

    async def search(
        self,
        query: str = "",
        status: Optional[str] = None,
        author_id: Optional[int] = None,
        tag: Optional[str] = None,
        skip: int = 0,
        limit: int = 20,
    ) -> Tuple[List[Post], int]:
        """Search posts with filters — parallel count + data query."""
        filters = [Post.deleted_at.is_(None)]

        if status:
            filters.append(Post.status == status)
        if author_id:
            filters.append(Post.author_id == author_id)
        if tag:
            filters.append(Post.tags.contains([tag]))
        if query:
            term = f"%{query}%"
            filters.append(
                or_(
                    Post.title.ilike(term),
                    Post.content.ilike(term)
                )
            )

        combined_filter = and_(*filters)

        # Execute count and data concurrently
        count_stmt = (
            select(func.count())
            .select_from(Post)
            .where(combined_filter)
        )
        data_stmt = (
            select(Post)
            .options(joinedload(Post.author))
            .where(combined_filter)
            .order_by(Post.created_at.desc())
            .offset(skip)
            .limit(limit)
        )

        count_result, data_result = await asyncio.gather(
            self.session.execute(count_stmt),
            self.session.execute(data_stmt)
        )

        return (
            list(data_result.unique().scalars().all()),
            count_result.scalar_one()
        )

    async def get_trending(self, limit: int = 10) -> List[Post]:
        """Get most viewed posts in last 7 days."""
        week_ago = datetime.now(timezone.utc) - timedelta(days=7)
        stmt = (
            select(Post)
            .options(joinedload(Post.author))
            .where(
                and_(
                    Post.status == "published",
                    Post.published_at >= week_ago,
                    Post.deleted_at.is_(None)
                )
            )
            .order_by(Post.view_count.desc())
            .limit(limit)
        )
        result = await self.session.execute(stmt)
        return list(result.unique().scalars().all())

    async def increment_views(self, post_id: int) -> None:
        """
        Atomically increment view count.
        Uses UPDATE statement — no load required.
        """
        stmt = (
            update(Post)
            .where(Post.id == post_id)
            .values(view_count=Post.view_count + 1)
        )
        await self.session.execute(stmt)

    async def slug_exists(
        self,
        slug: str,
        exclude_id: Optional[int] = None
    ) -> bool:
        """Check if slug is taken."""
        stmt = select(func.count()).select_from(Post).where(
            Post.slug == slug
        )
        if exclude_id:
            stmt = stmt.where(Post.id != exclude_id)
        result = await self.session.execute(stmt)
        return result.scalar_one() > 0

    async def soft_delete(self, post: Post) -> Post:
        post.deleted_at = datetime.now(timezone.utc)
        await self.session.flush()
        return post
```

---

## 4. 🔧 Async Services

Python

```
# app/services/user_service.py
import asyncio
from typing import Optional, Dict, Any, List
from sqlalchemy.ext.asyncio import AsyncSession

from app.repositories.user_repo import UserRepository
from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate
from app.core.security import hash_password, verify_password
from app.core.exceptions import (
    NotFoundException, ConflictException,
    ForbiddenException, ValidationException
)


class UserService:
    """Async service for user business logic."""

    def __init__(self, session: AsyncSession):
        self.session = session
        self.repo = UserRepository(session)

    async def get_user_or_404(self, user_id: int) -> User:
        """Get user or raise NotFoundException."""
        user = await self.repo.get_active_by_id(user_id)
        if not user:
            raise NotFoundException("User", user_id)
        return user

    async def register(self, data: UserCreate) -> User:
        """Register a new user."""
        # Check email uniqueness
        if await self.repo.email_taken(data.email):
            raise ConflictException("User", "email", data.email)

        return await self.repo.create(
            name=data.name.strip().title(),
            email=data.email.lower().strip(),
            password_hash=hash_password(data.password),
            role="user"
        )

    async def authenticate(self, email: str, password: str) -> User:
        """Verify credentials and return user."""
        user = await self.repo.get_by_email(email)

        if not user or not verify_password(password, user.password_hash):
            raise ValidationException("Invalid email or password")

        if not user.is_active:
            raise ForbiddenException("access this account — it is deactivated")

        return user

    async def get_users(
        self,
        page: int = 1,
        page_size: int = 20,
        **filters
    ) -> Dict[str, Any]:
        """Get paginated users."""
        skip = (page - 1) * page_size
        users, total = await self.repo.search(
            skip=skip,
            limit=page_size,
            **filters
        )
        return {
            "items": users,
            "total": total,
            "page": page,
            "page_size": page_size,
            "total_pages": (total + page_size - 1) // page_size,
            "has_next": page * page_size < total,
            "has_prev": page > 1,
        }

    async def update_user(
        self,
        user_id: int,
        data: UserUpdate,
        requesting_user: User
    ) -> User:
        """Update user with authorization check."""
        user = await self.get_user_or_404(user_id)

        if user.id != requesting_user.id and not requesting_user.is_admin:
            raise ForbiddenException("update another user's profile")

        updates = data.model_dump(exclude_unset=True)
        return await self.repo.update(user, **updates)

    async def delete_user(self, user_id: int, requesting_user: User) -> None:
        """Soft delete with authorization."""
        user = await self.get_user_or_404(user_id)

        if user.id != requesting_user.id and not requesting_user.is_admin:
            raise ForbiddenException("delete another user's account")

        await self.repo.soft_delete(user)

    async def get_dashboard_data(self, user_id: int) -> Dict[str, Any]:
        """
        Fetch all dashboard data concurrently.
        This is the power of async — multiple queries in parallel.
        """
        from app.repositories.post_repo import PostRepository
        post_repo = PostRepository(self.session)

        # Run multiple queries simultaneously
        user, posts_result, stats = await asyncio.gather(
            self.repo.get_active_by_id(user_id),
            post_repo.search(
                author_id=user_id,
                status="published",
                limit=5
            ),
            self.repo.get_stats_by_role()
        )

        if not user:
            raise NotFoundException("User", user_id)

        posts, post_count = posts_result

        return {
            "user": user,
            "recent_posts": posts,
            "total_posts": post_count,
            "platform_stats": stats
        }
```

---

## 5. 🛣️ Async FastAPI Routes

Python

```
# app/api/v1/users.py
from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Optional, Annotated

from app.core.database import get_async_db
from app.core.deps import get_current_user, get_current_admin
from app.services.user_service import UserService
from app.models.user import User
from app.schemas.user import (
    UserCreate, UserUpdate, UserResponse, UserListResponse
)
from app.schemas.common import PaginatedResponse, PaginationMeta

router = APIRouter(prefix="/users", tags=["Users"])

# Type aliases
AsyncDB = Annotated[AsyncSession, Depends(get_async_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
AdminUser = Annotated[User, Depends(get_current_admin)]


def get_user_service(db: AsyncDB) -> UserService:
    return UserService(db)

UserSvc = Annotated[UserService, Depends(get_user_service)]


@router.get("", response_model=PaginatedResponse[UserListResponse])
async def list_users(
    service: UserSvc,
    _: AdminUser,    # admin only
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    search: Optional[str] = Query(None, max_length=100),
    role: Optional[str] = Query(None),
    is_active: Optional[bool] = Query(None),
):
    """List all users with pagination. Admin only."""
    result = await service.get_users(
        page=page,
        page_size=page_size,
        query=search or "",
        role=role,
        is_active=is_active
    )

    return PaginatedResponse(
        data=result["items"],
        pagination=PaginationMeta(
            total=result["total"],
            page=result["page"],
            page_size=result["page_size"],
            total_pages=result["total_pages"],
            has_next=result["has_next"],
            has_prev=result["has_prev"],
        )
    )


@router.get("/me", response_model=UserResponse)
async def get_me(current_user: CurrentUser):
    """Get current user's profile."""
    return current_user


@router.get("/me/dashboard")
async def get_my_dashboard(
    current_user: CurrentUser,
    service: UserSvc
):
    """
    Get dashboard data — multiple async queries run concurrently.
    Shows the power of async: much faster than sync version.
    """
    return await service.get_dashboard_data(current_user.id)


@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserSvc,
    current_user: CurrentUser
):
    """Get user by ID."""
    user = await service.get_user_or_404(user_id)
    # Non-admins can only see their own profile
    if user.id != current_user.id and not current_user.is_admin:
        from fastapi import HTTPException
        raise HTTPException(status_code=403, detail="Forbidden")
    return user


@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    data: UserUpdate,
    service: UserSvc,
    current_user: CurrentUser
):
    """Update user profile."""
    return await service.update_user(user_id, data, current_user)


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    service: UserSvc,
    current_user: CurrentUser
):
    """Delete user account."""
    await service.delete_user(user_id, current_user)
```

---

## 6. 🔄 Async Dependencies & Authentication

Python

```
# app/core/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Optional, Annotated

from app.core.database import get_async_db
from app.core.security import decode_token
from app.models.user import User
from app.repositories.user_repo import UserRepository

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="/api/v1/auth/login"
)
oauth2_optional = OAuth2PasswordBearer(
    tokenUrl="/api/v1/auth/login",
    auto_error=False
)


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_async_db)
) -> User:
    """Async dependency: get authenticated user."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = decode_token(token)
        user_id = payload.get("sub")
        if not user_id:
            raise credentials_exception
    except ValueError:
        raise credentials_exception

    # Async database lookup
    repo = UserRepository(db)
    user = await repo.get_active_by_id(int(user_id))

    if not user:
        raise credentials_exception

    return user


async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """Ensure user is active."""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Account is deactivated"
        )
    return current_user


async def get_current_admin(
    current_user: User = Depends(get_current_active_user)
) -> User:
    """Ensure user is admin."""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    return current_user


async def get_optional_user(
    token: Optional[str] = Depends(oauth2_optional),
    db: AsyncSession = Depends(get_async_db)
) -> Optional[User]:
    """Get user if authenticated, None otherwise."""
    if not token:
        return None
    try:
        payload = decode_token(token)
        user_id = payload.get("sub")
        if user_id:
            repo = UserRepository(db)
            return await repo.get_active_by_id(int(user_id))
    except Exception:
        pass
    return None
```

---

## 7. 🔒 Alembic Async Migrations

Python

```
# alembic/env.py — configured for async engine
"""
Important: Alembic runs migrations SYNCHRONOUSLY.
Even though your app uses async, migrations use sync connections.
This is correct and expected — migrations aren't in hot path.
"""
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from app.core.database import Base
from app.models import user, post  # import all models!
from app.core.config import settings

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Use SYNC URL for migrations
SYNC_URL = settings.DATABASE_URL.replace(
    "postgresql+asyncpg://",
    "postgresql://"
)
config.set_main_option("sqlalchemy.url", SYNC_URL)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    """Run migrations in offline mode (generate SQL)."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        compare_server_default=True,
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Run migrations using async engine."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    """Run migrations in online mode."""
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Bash

```
# Migration commands (same as before)
alembic revision --autogenerate -m "create users and posts tables"
alembic upgrade head
alembic current
alembic history
```

---

## 8. 🔬 Advanced Async Patterns

### Avoiding Common Async DB Mistakes

Python

```
import asyncio
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.models.user import User
from app.models.post import Post


# ─────────────────────────────────────────
# MISTAKE 1: Lazy loading in async (N+1 problem)
# ─────────────────────────────────────────
async def bad_get_posts_with_authors(db: AsyncSession) -> list:
    """
    ❌ WRONG: Triggers lazy loading for each post.
    Each author access = new DB query = N+1 queries.
    """
    result = await db.execute(select(Post).limit(10))
    posts = result.scalars().all()

    data = []
    for post in posts:
        # ❌ This triggers a NEW DB query for EACH post!
        # In async, this would cause MissingGreenlet error
        # because lazy loading is sync!
        author_name = post.author.name   # CRASH or wrong behavior!
        data.append({"title": post.title, "author": author_name})
    return data


async def good_get_posts_with_authors(db: AsyncSession) -> list:
    """
    ✅ CORRECT: Load everything in one query with joinedload.
    """
    from sqlalchemy.orm import joinedload

    result = await db.execute(
        select(Post)
        .options(joinedload(Post.author))   # JOIN = one query total
        .where(Post.deleted_at.is_(None))
        .limit(10)
    )
    posts = result.unique().scalars().all()

    return [
        {"title": p.title, "author": p.author.name}
        for p in posts
    ]


# ─────────────────────────────────────────
# MISTAKE 2: Multiple round trips vs batching
# ─────────────────────────────────────────
async def bad_fetch_multiple(
    db: AsyncSession,
    user_ids: list[int]
) -> list:
    """❌ WRONG: N separate queries instead of one."""
    users = []
    for user_id in user_ids:
        result = await db.execute(
            select(User).where(User.id == user_id)
        )
        user = result.scalar_one_or_none()
        if user:
            users.append(user)
    return users


async def good_fetch_multiple(
    db: AsyncSession,
    user_ids: list[int]
) -> list:
    """✅ CORRECT: Single query with IN clause."""
    result = await db.execute(
        select(User).where(User.id.in_(user_ids))
    )
    return list(result.scalars().all())


# ─────────────────────────────────────────
# MISTAKE 3: Not using expire_on_commit=False
# ─────────────────────────────────────────
async def bad_return_after_commit(db: AsyncSession) -> dict:
    """
    ❌ WRONG: Accessing user after commit without expire_on_commit=False.
    SQLAlchemy expires all attributes after commit by default.
    Accessing them would trigger lazy load (crash in async!).
    """
    user = User(name="Alice", email="alice@test.com", password_hash="hash")
    db.add(user)
    await db.commit()
    # user.name here would trigger a lazy load → crash!
    return {"id": user.id, "name": user.name}  # ❌ might fail!


async def good_return_after_commit(db: AsyncSession) -> dict:
    """
    ✅ CORRECT: Use expire_on_commit=False in session factory.
    Then attributes remain accessible after commit.
    We set this in AsyncSessionLocal creation.
    """
    user = User(name="Alice", email="alice@test.com", password_hash="hash")
    db.add(user)
    await db.flush()     # assigns ID
    await db.refresh(user)  # loads all attributes
    # Now user.name is safe to access even after commit
    return {"id": user.id, "name": user.name}  # ✅ works!


# ─────────────────────────────────────────
# CORRECT: Concurrent queries in service
# ─────────────────────────────────────────
async def get_post_with_all_data(
    db: AsyncSession,
    post_id: int
) -> dict:
    """
    Fetch post + related data concurrently.
    All queries run simultaneously.
    """
    from sqlalchemy.orm import joinedload
    from app.repositories.post_repo import PostRepository
    from app.repositories.user_repo import UserRepository

    post_repo = PostRepository(db)
    user_repo = UserRepository(db)

    # Start all queries at once
    post_task = post_repo.get_with_author(post_id)
    trending_task = post_repo.get_trending(limit=5)
    stats_task = user_repo.get_stats_by_role()

    # Wait for all to complete
    post, trending, stats = await asyncio.gather(
        post_task,
        trending_task,
        stats_task
    )

    if not post:
        return None

    # Increment views (fire and don't wait)
    asyncio.create_task(post_repo.increment_views(post_id))

    return {
        "post": post,
        "trending": trending,
        "platform_stats": stats
    }
```

### Async Context Manager for Transactions

Python

```
from contextlib import asynccontextmanager
from sqlalchemy.ext.asyncio import AsyncSession


@asynccontextmanager
async def transaction(session: AsyncSession):
    """
    Async context manager for explicit transaction control.

    Usage:
        async with transaction(db) as tx:
            user = await user_repo.create(...)
            post = await post_repo.create(author_id=user.id, ...)
    """
    async with session.begin():
        try:
            yield session
        except Exception:
            await session.rollback()
            raise


@asynccontextmanager
async def savepoint(session: AsyncSession, name: str = "sp"):
    """Nested transaction using savepoint."""
    async with session.begin_nested():
        yield session


# Usage
async def transfer_with_savepoints(db: AsyncSession):
    async with transaction(db):
        # Main transaction
        user1 = await UserRepository(db).get_active_by_id(1)
        user2 = await UserRepository(db).get_active_by_id(2)

        try:
            # Nested operation with savepoint
            async with savepoint(db):
                user1.is_verified = True
                user2.is_verified = True
                await db.flush()
                # If this fails, only this part rolls back
        except Exception:
            print("Savepoint rolled back, outer continues")

        # This part still commits
        user1.is_active = True
        await db.flush()
```

---

## 9. ⚡ Performance Comparison

Python

```
# benchmark.py — comparing sync vs async DB access
import asyncio
import time
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

SYNC_URL = "postgresql://myapp_user:myapp_password@localhost:5432/async_blog_db"
ASYNC_URL = "postgresql+asyncpg://myapp_user:myapp_password@localhost:5432/async_blog_db"


# ─────────────────────────────────────────
# SYNC BENCHMARK
# ─────────────────────────────────────────
def sync_benchmark(n_requests: int = 50):
    """Simulate N concurrent requests with sync DB."""
    engine = create_engine(SYNC_URL, pool_size=5, max_overflow=10)
    SessionLocal = sessionmaker(bind=engine)

    def handle_request(request_id: int) -> dict:
        with SessionLocal() as db:
            # Simulate getting a user (50ms query)
            time.sleep(0.05)
            return {"request_id": request_id, "user": "found"}

    start = time.perf_counter()
    # Sequential — each request waits for previous
    results = [handle_request(i) for i in range(n_requests)]
    elapsed = time.perf_counter() - start

    print(f"Sync ({n_requests} requests): {elapsed:.2f}s "
          f"({n_requests/elapsed:.0f} req/s)")


# ─────────────────────────────────────────
# ASYNC BENCHMARK
# ─────────────────────────────────────────
async def async_benchmark(n_requests: int = 50):
    """Simulate N concurrent requests with async DB."""
    engine = create_async_engine(ASYNC_URL, pool_size=5, max_overflow=10)
    AsyncSessionLocal = async_sessionmaker(engine)

    async def handle_request(request_id: int) -> dict:
        async with AsyncSessionLocal() as db:
            # Simulate getting a user (50ms query)
            await asyncio.sleep(0.05)
            return {"request_id": request_id, "user": "found"}

    start = time.perf_counter()
    # Concurrent — all requests run simultaneously
    results = await asyncio.gather(*[
        handle_request(i) for i in range(n_requests)
    ])
    elapsed = time.perf_counter() - start

    print(f"Async ({n_requests} requests): {elapsed:.2f}s "
          f"({n_requests/elapsed:.0f} req/s)")

    await engine.dispose()


# Run comparison
print("=== DB Performance Comparison ===\n")
sync_benchmark(50)
asyncio.run(async_benchmark(50))

# Expected output:
# Sync  (50 requests): 2.50s (20 req/s)
# Async (50 requests): 0.05s (1000 req/s) ← 50x faster!
```

---

## 10. 🧪 Testing Async DB Code

Python

```
# tests/conftest.py
import asyncio
import pytest
import pytest_asyncio
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    AsyncSession, create_async_engine, async_sessionmaker
)
from httpx import AsyncClient, ASGITransport
from app.main import create_app
from app.core.database import Base, get_async_db

# Use SQLite for testing (no PostgreSQL needed)
TEST_DB_URL = "sqlite+aiosqlite:///./test.db"

test_engine = create_async_engine(
    TEST_DB_URL,
    connect_args={"check_same_thread": False},
)
TestSessionLocal = async_sessionmaker(
    bind=test_engine,
    expire_on_commit=False,
)


@pytest_asyncio.fixture(scope="session")
async def setup_database():
    """Create tables once for the entire test session."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await test_engine.dispose()


@pytest_asyncio.fixture
async def db(setup_database) -> AsyncGenerator[AsyncSession, None]:
    """Provide a clean database session for each test."""
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()   # undo changes after each test


@pytest_asyncio.fixture
async def client(db: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """Async HTTP test client."""
    app = create_app()

    # Override DB dependency
    async def override_get_db():
        yield db

    app.dependency_overrides[get_async_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as ac:
        yield ac

    app.dependency_overrides.clear()


# tests/test_users.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession
from app.repositories.user_repo import UserRepository
from app.core.security import hash_password


@pytest.fixture
async def test_user(db: AsyncSession):
    """Create a test user."""
    repo = UserRepository(db)
    user = await repo.create(
        name="Test User",
        email="test@test.com",
        password_hash=hash_password("SecurePass1!"),
    )
    await db.flush()
    return user


@pytest.fixture
async def auth_headers(client: AsyncClient, test_user):
    """Get auth headers for test user."""
    response = await client.post("/api/v1/auth/login", json={
        "email": "test@test.com",
        "password": "SecurePass1!"
    })
    assert response.status_code == 200
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


class TestUserAPI:
    @pytest.mark.asyncio
    async def test_get_me(self, client: AsyncClient, auth_headers: dict):
        response = await client.get(
            "/api/v1/users/me",
            headers=auth_headers
        )
        assert response.status_code == 200
        assert response.json()["email"] == "test@test.com"

    @pytest.mark.asyncio
    async def test_list_users_requires_admin(
        self,
        client: AsyncClient,
        auth_headers: dict
    ):
        response = await client.get("/api/v1/users", headers=auth_headers)
        assert response.status_code == 403

    @pytest.mark.asyncio
    async def test_update_own_profile(
        self,
        client: AsyncClient,
        auth_headers: dict
    ):
        response = await client.patch(
            "/api/v1/users/1",
            json={"bio": "I love Python!"},
            headers=auth_headers
        )
        assert response.status_code == 200
        assert response.json()["bio"] == "I love Python!"

    @pytest.mark.asyncio
    async def test_concurrent_requests(self, client: AsyncClient, auth_headers: dict):
        """Test that async handles concurrent requests correctly."""
        import asyncio

        async def get_profile():
            return await client.get("/api/v1/users/me", headers=auth_headers)

        # All requests should work correctly when run concurrently
        responses = await asyncio.gather(*[get_profile() for _ in range(10)])
        assert all(r.status_code == 200 for r in responses)


# pytest.ini or pyproject.toml
# [tool.pytest.ini_options]
# asyncio_mode = "auto"
# testpaths = ["tests"]
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│               ASYNC DATABASE OPERATIONS                        │
│                                                                │
│  LAYERS:                                                       │
│  asyncpg     → raw driver (fastest, most control)            │
│  SQLAlchemy  → ORM with async support                        │
│  Repository  → encapsulates all async DB operations           │
│  Service     → business logic using repositories             │
│  FastAPI     → async routes using services                    │
│                                                                │
│  KEY RULES:                                                    │
│  ✅ expire_on_commit=False in AsyncSessionLocal              │
│  ✅ joinedload() for relationships (not lazy!)               │
│  ✅ Use IN queries instead of loops                          │
│  ✅ asyncio.gather() for parallel queries                    │
│  ✅ refresh() after flush() to get server defaults           │
│  ❌ Never access relationships without loading first          │
│  ❌ Never use time.sleep() in async routes                   │
│  ❌ Never block event loop with sync operations              │
│                                                                │
│  ASYNC SESSION LIFECYCLE:                                      │
│  Request comes in                                              │
│  → get_async_db() creates session                             │
│  → Route handler uses session                                 │
│  → Success: session.commit()                                  │
│  → Failure: session.rollback()                               │
│  → Always: session.close()                                   │
│                                                                │
│  CONCURRENT QUERIES:                                           │
│  results = await asyncio.gather(                              │
│      repo.get_user(user_id),     ← runs simultaneously       │
│      repo.get_posts(user_id),    ← runs simultaneously       │
│      repo.get_stats(),           ← runs simultaneously       │
│  )                                                             │
│                                                                │
│  MIGRATIONS:                                                   │
│  Alembic still uses SYNC connection (that's fine)            │
│  Use postgresql:// (not postgresql+asyncpg://) for Alembic   │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  Why can't you use psycopg2 with async FastAPI?
2.  What is asyncpg and when would you use it directly?
3.  What is expire_on_commit=False and why is it critical?
4.  What is the N+1 problem in async SQLAlchemy?
    How do you solve it?
5.  What is the difference between joinedload and selectinload?
6.  Why does flush() exist separately from commit()?
7.  How do you run multiple DB queries concurrently?
8.  What is the correct URL format for async SQLAlchemy?
9.  Why do Alembic migrations use sync connections?
10. What does session.refresh(instance) do?
11. What is the per-key locking pattern and why use it?
12. How do you test async FastAPI routes?
    What is ASGITransport?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Async Post Service
# Build a complete PostService with async methods:
# - create_post(data, author) → Post
# - get_post_or_404(post_id) → Post (increment views)
# - update_post(post_id, data, user) → Post (check ownership)
# - delete_post(post_id, user) → None (soft delete)
# - get_trending(limit=10) → List[Post]
# - search_posts(query, tag, page, size) → Paginated
# Use asyncio.gather where possible

# Exercise 2: Concurrent Dashboard
# Build an endpoint: GET /api/v1/dashboard
# That returns ALL of these in ONE request:
# - current user profile
# - my last 5 posts
# - platform stats (total users, total posts)
# - trending posts (top 3)
# - my unread notifications count
# Everything must run CONCURRENTLY
# Benchmark: should complete in ~time of slowest query

# Exercise 3: Bulk Operations
# Build async bulk operations:
# POST /api/v1/users/bulk-create → create up to 100 users
# POST /api/v1/posts/bulk-publish → publish multiple posts
# DELETE /api/v1/posts/bulk-delete → soft delete multiple
# Each should:
# - Validate all inputs first
# - Execute in a single transaction
# - Return success/failure per item

# Exercise 4: Async Background Tasks
# Add to your app:
# - When user registers: background task sends welcome email (simulated)
# - When post published: background task updates search index (simulated)
# - When user deletes account: background task cleanup (simulated)
# Use FastAPI's BackgroundTasks
# Add logging to verify tasks run after response is sent

# Exercise 5: Performance Test
# Create a benchmark endpoint:
# GET /api/v1/benchmark?mode=sync|async&queries=10
# sync mode: run N queries sequentially
# async mode: run N queries concurrently with asyncio.gather
# Return timing for each
# Document the difference in your README
```

---

## ✅ Phase 5.2 Complete!

**You now know:**

text

```
✅ asyncpg — raw async PostgreSQL driver
✅ SQLAlchemy async engine and session setup
✅ expire_on_commit=False and why it matters
✅ Async models and relationships
✅ Async repositories with parallel queries
✅ Avoiding N+1 with joinedload/selectinload
✅ Concurrent DB queries with asyncio.gather()
✅ Async service layer
✅ Async FastAPI routes and dependencies
✅ Alembic with async engine
✅ Testing async code with pytest-asyncio
✅ Common async DB mistakes and how to avoid them
```