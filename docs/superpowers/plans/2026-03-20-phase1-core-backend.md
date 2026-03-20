# Phase 1: Core Backend — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a working FastAPI backend where users can register, log workouts, browse exercises, and see a simple volume-based score — testable entirely via Swagger UI.

**Architecture:** Single FastAPI application with PostgreSQL. Separated into models (SQLAlchemy ORM), schemas (Pydantic validation), services (business logic), and routes (thin HTTP handlers). JWT access tokens for auth, bcrypt for password hashing.

**Tech Stack:** Python 3.11+, FastAPI, SQLAlchemy 2.0 (async), Alembic, PostgreSQL, python-jose (JWT), passlib[bcrypt], pytest, httpx (async test client)

**Spec:** `docs/superpowers/specs/2026-03-20-competitive-ai-gym-platform-design.md` — Phase 1 section

---

## Phase 1 File Map

Every file created in this phase and its responsibility:

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app, route registration, startup/shutdown
│   ├── config.py                  # Settings from env vars (DATABASE_URL, JWT_SECRET, etc.)
│   ├── database.py                # Async SQLAlchemy engine, session factory, Base
│   │
│   ├── models/
│   │   ├── __init__.py            # Re-exports all models for Alembic discovery
│   │   ├── user.py                # User table: email, hashed_password, username, category, body stats
│   │   ├── exercise.py            # Exercise table: name, category, muscle_groups, difficulty, type
│   │   ├── workout.py             # Workout table: user_id, date, type, duration, score
│   │   ├── exercise_log.py        # ExerciseLog table: workout_id, exercise_id, sets/reps/weight/distance/pace/duration/rpe
│   │   └── score.py               # Score table: user_id, category, period, volume_score, total_score
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py                # UserCreate, UserLogin, UserResponse, UserUpdate, TokenResponse
│   │   ├── exercise.py            # ExerciseCreate, ExerciseResponse
│   │   ├── workout.py             # WorkoutCreate, ExerciseLogCreate, WorkoutResponse, WorkoutUpdate
│   │   └── score.py               # ScoreResponse
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth.py                # hash_password, verify_password, create_token, register, login
│   │   ├── workout.py             # create_workout, get_workout, list_workouts, update_workout, delete_workout
│   │   └── scoring.py             # calculate_volume_score, recalculate_user_scores
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── auth.py                # POST /register, POST /login
│   │   ├── profile.py             # GET /profile, PUT /profile, GET /users/{id}
│   │   ├── exercises.py           # GET /exercises, POST /exercises (admin seed)
│   │   ├── workouts.py            # POST /workouts, GET /workouts/history, GET/PUT/DELETE /workouts/{id}
│   │   └── scores.py              # GET /scores/me
│   │
│   └── middleware/
│       ├── __init__.py
│       ├── auth.py                # get_current_user dependency (JWT decode)
│       └── error_handler.py       # Global exception handler middleware
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                # Test database, async client fixture, test user factory
│   ├── unit/
│   │   ├── __init__.py
│   │   └── test_scoring.py        # Scoring algorithm unit tests
│   └── integration/
│       ├── __init__.py
│       ├── test_auth.py           # Registration, login, token validation
│       ├── test_exercises.py      # Exercise CRUD
│       ├── test_workouts.py       # Workout CRUD + score recalculation
│       └── test_profile.py        # Profile view/edit
│
├── alembic/
│   ├── env.py                     # Alembic config pointing to app models
│   └── versions/                  # Migration files
│
├── alembic.ini                    # Alembic settings
├── requirements.txt               # All dependencies
├── .env.example                   # Template for env vars
└── .gitignore
```

---

## Task 1: Project Scaffolding

**Files:**
- Create: `backend/requirements.txt`
- Create: `backend/.env.example`
- Create: `backend/.gitignore`

- [ ] **Step 1: Create requirements.txt**

```
# backend/requirements.txt
fastapi==0.115.0
uvicorn[standard]==0.30.6
sqlalchemy[asyncio]==2.0.35
asyncpg==0.29.0
alembic==1.13.2
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
pydantic[email-validator]==2.9.2
pydantic-settings==2.5.2
python-dotenv==1.0.1

# Testing
pytest==8.3.3
pytest-asyncio==0.24.0
httpx==0.27.2
```

- [ ] **Step 2: Create .env.example**

```
# backend/.env.example
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/gym_platform
TEST_DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/gym_platform_test
JWT_SECRET=change-this-to-a-random-secret-key
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=30
```

- [ ] **Step 3: Create .gitignore**

```
# backend/.gitignore
__pycache__/
*.pyc
.env
*.db
.pytest_cache/
venv/
.venv/
```

- [ ] **Step 4: Create directory structure**

Run:
```bash
cd backend
mkdir -p app/models app/schemas app/services app/routes app/middleware
mkdir -p tests/unit tests/integration
touch app/__init__.py app/models/__init__.py app/schemas/__init__.py
touch app/services/__init__.py app/routes/__init__.py app/middleware/__init__.py
touch tests/__init__.py tests/unit/__init__.py tests/integration/__init__.py
```

- [ ] **Step 5: Install dependencies**

Run:
```bash
cd backend
python -m venv venv
source venv/Scripts/activate  # Windows Git Bash
pip install -r requirements.txt
```
Expected: All packages install successfully.

- [ ] **Step 6: Commit**

```bash
git add backend/
git commit -m "chore: scaffold Phase 1 project structure with dependencies"
```

---

## Task 2: Config & Database Setup

**Files:**
- Create: `backend/app/config.py`
- Create: `backend/app/database.py`

- [ ] **Step 1: Create config.py**

```python
# backend/app/config.py
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://postgres:password@localhost:5432/gym_platform"
    test_database_url: str = "postgresql+asyncpg://postgres:password@localhost:5432/gym_platform_test"
    jwt_secret: str = "change-this-to-a-random-secret-key"
    jwt_algorithm: str = "HS256"
    jwt_expiration_minutes: int = 30

    model_config = {"env_file": ".env"}


settings = Settings()
```

- [ ] **Step 2: Create database.py**

```python
# backend/app/database.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from app.config import settings

engine = create_async_engine(settings.database_url, echo=False)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_db():
    async with async_session() as session:
        yield session
```

- [ ] **Step 3: Commit**

```bash
git add backend/app/config.py backend/app/database.py
git commit -m "feat: add config and async database setup"
```

---

## Task 3: User Model & Migration

**Files:**
- Create: `backend/app/models/user.py`
- Modify: `backend/app/models/__init__.py`
- Create: `backend/alembic.ini`
- Create: `backend/alembic/env.py`

- [ ] **Step 1: Create user model**

```python
# backend/app/models/user.py
import enum
import uuid
from datetime import datetime, timezone

from sqlalchemy import String, Float, Text, Enum as SAEnum, ARRAY, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class AthleteCategory(str, enum.Enum):
    crossfit = "crossfit"
    bodybuilding = "bodybuilding"
    running = "running"
    endurance = "endurance"
    hybrid = "hybrid"


class DietaryPreference(str, enum.Enum):
    none = "none"
    vegan = "vegan"
    vegetarian = "vegetarian"
    keto = "keto"
    paleo = "paleo"
    halal = "halal"
    other = "other"


class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    username: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    category: Mapped[str] = mapped_column(
        SAEnum(AthleteCategory, name="athlete_category"), nullable=False
    )
    bio: Mapped[str | None] = mapped_column(Text, nullable=True)
    avatar_url: Mapped[str | None] = mapped_column(String(500), nullable=True)
    height_cm: Mapped[float | None] = mapped_column(Float, nullable=True)
    weight_kg: Mapped[float | None] = mapped_column(Float, nullable=True)
    body_fat_pct: Mapped[float | None] = mapped_column(Float, nullable=True)
    dietary_preference: Mapped[str | None] = mapped_column(
        SAEnum(DietaryPreference, name="dietary_preference"), nullable=True, default="none"
    )
    injuries: Mapped[list[str] | None] = mapped_column(
        ARRAY(Text), nullable=True, default=list
    )
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )
```

- [ ] **Step 2: Export model in __init__.py**

```python
# backend/app/models/__init__.py
from app.models.user import User

__all__ = ["User"]
```

- [ ] **Step 3: Initialize Alembic**

Run:
```bash
cd backend
alembic init alembic
```
Expected: Creates `alembic/` directory and `alembic.ini`.

- [ ] **Step 4: Configure alembic.ini**

Edit `alembic.ini` — set `sqlalchemy.url` to empty (we'll override in env.py):
```ini
sqlalchemy.url =
```

- [ ] **Step 5: Configure alembic/env.py**

Replace `alembic/env.py` with:

```python
# backend/alembic/env.py
import asyncio
from logging.config import fileConfig

from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

from alembic import context

from app.config import settings
from app.database import Base
from app.models import *  # noqa: F401,F403 — ensures all models are registered

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

- [ ] **Step 6: Create the database and run first migration**

Run:
```bash
# Create the databases first (if not exist)
createdb gym_platform
createdb gym_platform_test

# Generate migration
cd backend
alembic revision --autogenerate -m "create users table"

# Apply migration
alembic upgrade head
```
Expected: `users` table created in PostgreSQL with all columns.

- [ ] **Step 7: Commit**

```bash
git add backend/app/models/ backend/alembic/ backend/alembic.ini
git commit -m "feat: add User model with Alembic migration"
```

---

## Task 4: Auth Schemas

**Files:**
- Create: `backend/app/schemas/user.py`

- [ ] **Step 1: Create user schemas**

```python
# backend/app/schemas/user.py
import uuid
from datetime import datetime

from pydantic import BaseModel, EmailStr, Field


class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    username: str = Field(min_length=3, max_length=100)
    category: str = Field(pattern="^(crossfit|bodybuilding|running|endurance|hybrid)$")


class UserLogin(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


class UserResponse(BaseModel):
    id: uuid.UUID
    email: str
    username: str
    category: str
    bio: str | None = None
    avatar_url: str | None = None
    height_cm: float | None = None
    weight_kg: float | None = None
    body_fat_pct: float | None = None
    dietary_preference: str | None = None
    injuries: list[str] | None = None
    created_at: datetime

    model_config = {"from_attributes": True}


class UserPublicResponse(BaseModel):
    """Public profile — omits sensitive fields (email, body_fat, diet, injuries)."""
    id: uuid.UUID
    username: str
    category: str
    bio: str | None = None
    avatar_url: str | None = None
    height_cm: float | None = None
    weight_kg: float | None = None
    created_at: datetime

    model_config = {"from_attributes": True}


class UserUpdate(BaseModel):
    username: str | None = Field(None, min_length=3, max_length=100)
    category: str | None = Field(None, pattern="^(crossfit|bodybuilding|running|endurance|hybrid)$")
    bio: str | None = None
    avatar_url: str | None = None
    height_cm: float | None = None
    weight_kg: float | None = None
    body_fat_pct: float | None = None
    dietary_preference: str | None = None
    injuries: list[str] | None = None
```

- [ ] **Step 2: Commit**

```bash
git add backend/app/schemas/user.py
git commit -m "feat: add Pydantic schemas for user auth and profile"
```

---

## Task 5: Auth Service

**Files:**
- Create: `backend/app/services/auth.py`
- Create: `backend/tests/integration/test_auth.py`
- Create: `backend/tests/conftest.py`

- [ ] **Step 1: Create test fixtures (conftest.py)**

```python
# backend/tests/conftest.py
import asyncio
import uuid
from typing import AsyncGenerator

import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.config import settings
from app.database import Base, get_db
from app.main import app

# Use a real PostgreSQL test database — SQLite doesn't support ARRAY columns
# or PostgreSQL-specific operators (e.g., .any()) that our models use.
# Create this database before running tests: createdb gym_platform_test
TEST_DATABASE_URL = settings.test_database_url

engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSession = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest_asyncio.fixture(autouse=True)
async def setup_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


async def override_get_db():
    async with TestSession() as session:
        yield session


app.dependency_overrides[get_db] = override_get_db


@pytest_asyncio.fixture
async def client() -> AsyncGenerator[AsyncClient, None]:
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c


@pytest_asyncio.fixture
async def auth_headers(client: AsyncClient) -> dict:
    """Register a test user and return auth headers."""
    user_data = {
        "email": f"test_{uuid.uuid4().hex[:8]}@example.com",
        "password": "testpassword123",
        "username": f"testuser_{uuid.uuid4().hex[:8]}",
        "category": "bodybuilding",
    }
    await client.post("/register", json=user_data)
    response = await client.post(
        "/login", json={"email": user_data["email"], "password": user_data["password"]}
    )
    token = response.json()["data"]["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

- [ ] **Step 2: Write failing auth tests**

```python
# backend/tests/integration/test_auth.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_register_success(client: AsyncClient):
    response = await client.post("/register", json={
        "email": "new@example.com",
        "password": "securepass123",
        "username": "newuser",
        "category": "crossfit",
    })
    assert response.status_code == 201
    data = response.json()["data"]
    assert data["email"] == "new@example.com"
    assert data["username"] == "newuser"
    assert "password" not in data
    assert "hashed_password" not in data


@pytest.mark.asyncio
async def test_register_duplicate_email(client: AsyncClient):
    user = {
        "email": "dupe@example.com",
        "password": "securepass123",
        "username": "user1",
        "category": "crossfit",
    }
    await client.post("/register", json=user)
    user["username"] = "user2"
    response = await client.post("/register", json=user)
    assert response.status_code == 409


@pytest.mark.asyncio
async def test_register_short_password(client: AsyncClient):
    response = await client.post("/register", json={
        "email": "short@example.com",
        "password": "short",
        "username": "shortpw",
        "category": "crossfit",
    })
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_login_success(client: AsyncClient):
    await client.post("/register", json={
        "email": "login@example.com",
        "password": "securepass123",
        "username": "loginuser",
        "category": "bodybuilding",
    })
    response = await client.post("/login", json={
        "email": "login@example.com",
        "password": "securepass123",
    })
    assert response.status_code == 200
    data = response.json()["data"]
    assert "access_token" in data
    assert data["token_type"] == "bearer"


@pytest.mark.asyncio
async def test_login_wrong_password(client: AsyncClient):
    await client.post("/register", json={
        "email": "wrongpw@example.com",
        "password": "securepass123",
        "username": "wrongpwuser",
        "category": "running",
    })
    response = await client.post("/login", json={
        "email": "wrongpw@example.com",
        "password": "wrongpassword",
    })
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_login_nonexistent_user(client: AsyncClient):
    response = await client.post("/login", json={
        "email": "nobody@example.com",
        "password": "whatever123",
    })
    assert response.status_code == 401
```

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
cd backend
python -m pytest tests/integration/test_auth.py -v
```
Expected: FAIL — `app.main` doesn't exist yet.

- [ ] **Step 4: Create auth service**

```python
# backend/app/services/auth.py
import logging
from datetime import datetime, timedelta, timezone
from uuid import UUID

from jose import jwt
from passlib.context import CryptContext
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate

logger = logging.getLogger(__name__)
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def create_access_token(user_id: UUID) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expiration_minutes)
    payload = {"sub": str(user_id), "exp": expire}
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)


async def register(db: AsyncSession, data: UserCreate) -> User:
    user = User(
        email=data.email,
        hashed_password=hash_password(data.password),
        username=data.username,
        category=data.category,
    )
    db.add(user)
    try:
        await db.commit()
        await db.refresh(user)
    except IntegrityError:
        await db.rollback()
        raise ValueError("Email or username already exists")
    logger.info("User registered: user_id=%s email=%s", user.id, user.email)
    return user


async def login(db: AsyncSession, email: str, password: str) -> str:
    result = await db.execute(select(User).where(User.email == email))
    user = result.scalar_one_or_none()
    if not user or not verify_password(password, user.hashed_password):
        raise ValueError("Invalid email or password")
    token = create_access_token(user.id)
    logger.info("User logged in: user_id=%s", user.id)
    return token


async def get_user_by_id(db: AsyncSession, user_id: UUID) -> User | None:
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()


async def update_user(db: AsyncSession, user: User, data: UserUpdate) -> User:
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(user, field, value)
    await db.commit()
    await db.refresh(user)
    logger.info("User updated: user_id=%s fields=%s", user.id, list(update_data.keys()))
    return user
```

- [ ] **Step 5: Create auth middleware (JWT dependency)**

```python
# backend/app/middleware/auth.py
from uuid import UUID

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jose import JWTError, jwt
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import get_db
from app.models.user import User
from app.services.auth import get_user_by_id

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(
            token, settings.jwt_secret, algorithms=[settings.jwt_algorithm]
        )
        user_id = UUID(payload["sub"])
    except (JWTError, ValueError, KeyError):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
        )
    user = await get_user_by_id(db, user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )
    return user
```

- [ ] **Step 6: Create error handler middleware**

```python
# backend/app/middleware/error_handler.py
import logging
import traceback

from fastapi import Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

logger = logging.getLogger(__name__)


class ErrorHandlerMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        try:
            return await call_next(request)
        except Exception as exc:
            logger.error(
                "Unhandled exception: %s\n%s",
                str(exc),
                traceback.format_exc(),
            )
            return JSONResponse(
                status_code=500,
                content={"error": "Internal server error", "detail": "An unexpected error occurred"},
            )
```

- [ ] **Step 7: Create auth routes**

```python
# backend/app/routes/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.schemas.user import UserCreate, UserLogin, UserResponse, TokenResponse
from app.services import auth as auth_service

router = APIRouter()


@router.post("/register", status_code=status.HTTP_201_CREATED)
async def register(data: UserCreate, db: AsyncSession = Depends(get_db)):
    try:
        user = await auth_service.register(db, data)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail=str(e))
    return {"data": UserResponse.model_validate(user), "message": "User registered successfully"}


@router.post("/login")
async def login(data: UserLogin, db: AsyncSession = Depends(get_db)):
    try:
        token = await auth_service.login(db, data.email, data.password)
    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid email or password",
        )
    return {"data": TokenResponse(access_token=token)}
```

- [ ] **Step 8: Create profile routes**

```python
# backend/app/routes/profile.py
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.middleware.auth import get_current_user
from app.models.user import User
from app.schemas.user import UserResponse, UserPublicResponse, UserUpdate
from app.services import auth as auth_service

router = APIRouter()


@router.get("/profile")
async def get_profile(user: User = Depends(get_current_user)):
    return {"data": UserResponse.model_validate(user)}


@router.put("/profile")
async def update_profile(
    data: UserUpdate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    updated = await auth_service.update_user(db, user, data)
    return {"data": UserResponse.model_validate(updated), "message": "Profile updated"}


@router.get("/users/{user_id}")
async def get_user(user_id: UUID, db: AsyncSession = Depends(get_db)):
    user = await auth_service.get_user_by_id(db, user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found")
    # Public profile — omits email, body_fat_pct, dietary_preference, injuries
    return {"data": UserPublicResponse.model_validate(user)}
```

- [ ] **Step 9: Create main.py (app entry point)**

```python
# backend/app/main.py
import logging

from fastapi import FastAPI

from app.middleware.error_handler import ErrorHandlerMiddleware
from app.routes import auth, profile

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)

app = FastAPI(title="Competitive AI Gym Platform", version="0.1.0")

app.add_middleware(ErrorHandlerMiddleware)

app.include_router(auth.router, tags=["Auth"])
app.include_router(profile.router, tags=["Profile"])


@app.get("/health")
async def health():
    return {"status": "ok"}
```

- [ ] **Step 10: Run tests to verify they pass**

Run:
```bash
cd backend
python -m pytest tests/integration/test_auth.py -v
```
Expected: All 6 tests PASS.

- [ ] **Step 11: Commit**

```bash
git add backend/app/ backend/tests/
git commit -m "feat: add user auth (register, login, JWT) with profile endpoints"
```

---

## Task 6: Exercise Model & Routes

**Files:**
- Create: `backend/app/models/exercise.py`
- Create: `backend/app/schemas/exercise.py`
- Create: `backend/app/routes/exercises.py`
- Create: `backend/tests/integration/test_exercises.py`

- [ ] **Step 1: Write failing exercise tests**

```python
# backend/tests/integration/test_exercises.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_create_exercise(client: AsyncClient, auth_headers: dict):
    response = await client.post("/exercises", json={
        "name": "Barbell Back Squat",
        "category": ["bodybuilding", "crossfit"],
        "muscle_groups": ["quads", "glutes", "core"],
        "exercise_type": "strength",
        "difficulty": 4,
        "instructions": "Bar on upper back, squat to parallel, drive up through heels.",
    }, headers=auth_headers)
    assert response.status_code == 201
    data = response.json()["data"]
    assert data["name"] == "Barbell Back Squat"
    assert data["difficulty"] == 4


@pytest.mark.asyncio
async def test_list_exercises(client: AsyncClient, auth_headers: dict):
    await client.post("/exercises", json={
        "name": "Bench Press",
        "category": ["bodybuilding"],
        "muscle_groups": ["chest", "triceps"],
        "exercise_type": "strength",
        "difficulty": 3,
        "instructions": "Lower bar to chest, press up.",
    }, headers=auth_headers)
    response = await client.get("/exercises", headers=auth_headers)
    assert response.status_code == 200
    assert len(response.json()["data"]) >= 1


@pytest.mark.asyncio
async def test_list_exercises_filter_by_category(client: AsyncClient, auth_headers: dict):
    await client.post("/exercises", json={
        "name": "5K Run",
        "category": ["running", "endurance"],
        "muscle_groups": ["quads", "calves"],
        "exercise_type": "cardio",
        "difficulty": 2,
        "instructions": "Run 5 kilometers at a steady pace.",
    }, headers=auth_headers)
    response = await client.get("/exercises?category=running", headers=auth_headers)
    assert response.status_code == 200
    exercises = response.json()["data"]
    for ex in exercises:
        assert "running" in ex["category"]


@pytest.mark.asyncio
async def test_list_exercises_pagination(client: AsyncClient, auth_headers: dict):
    response = await client.get("/exercises?limit=1&offset=0", headers=auth_headers)
    assert response.status_code == 200
    body = response.json()
    assert "total" in body
    assert body["limit"] == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
cd backend
python -m pytest tests/integration/test_exercises.py -v
```
Expected: FAIL — exercise model/routes don't exist yet.

- [ ] **Step 3: Create exercise model**

```python
# backend/app/models/exercise.py
import uuid

from sqlalchemy import String, Integer, Text, ARRAY, Enum as SAEnum
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base

import enum


class ExerciseType(str, enum.Enum):
    strength = "strength"
    cardio = "cardio"
    olympic = "olympic"
    bodyweight = "bodyweight"
    flexibility = "flexibility"


class Exercise(Base):
    __tablename__ = "exercises"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    name: Mapped[str] = mapped_column(String(200), unique=True, nullable=False)
    category: Mapped[list[str]] = mapped_column(ARRAY(String), nullable=False)
    muscle_groups: Mapped[list[str]] = mapped_column(ARRAY(String), nullable=False)
    exercise_type: Mapped[str] = mapped_column(
        SAEnum(ExerciseType, name="exercise_type"), nullable=False
    )
    difficulty: Mapped[int] = mapped_column(Integer, nullable=False)
    instructions: Mapped[str | None] = mapped_column(Text, nullable=True)
```

- [ ] **Step 4: Create exercise schema**

```python
# backend/app/schemas/exercise.py
import uuid
from typing import Literal

from pydantic import BaseModel, Field, field_validator

VALID_CATEGORIES = {"crossfit", "bodybuilding", "running", "endurance", "hybrid"}


class ExerciseCreate(BaseModel):
    name: str = Field(max_length=200)
    category: list[str]
    muscle_groups: list[str]
    exercise_type: str = Field(pattern="^(strength|cardio|olympic|bodyweight|flexibility)$")
    difficulty: int = Field(ge=1, le=5)
    instructions: str | None = None

    @field_validator("category")
    @classmethod
    def validate_categories(cls, v: list[str]) -> list[str]:
        if not v:
            raise ValueError("At least one category is required")
        invalid = set(v) - VALID_CATEGORIES
        if invalid:
            raise ValueError(f"Invalid categories: {invalid}. Must be one of: {VALID_CATEGORIES}")
        return v


class ExerciseResponse(BaseModel):
    id: uuid.UUID
    name: str
    category: list[str]
    muscle_groups: list[str]
    exercise_type: str
    difficulty: int
    instructions: str | None = None

    model_config = {"from_attributes": True}
```

- [ ] **Step 5: Create exercise routes**

```python
# backend/app/routes/exercises.py
from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.middleware.auth import get_current_user
from app.models.exercise import Exercise
from app.models.user import User
from app.schemas.exercise import ExerciseCreate, ExerciseResponse

router = APIRouter(prefix="/exercises", tags=["Exercises"])


@router.post("", status_code=status.HTTP_201_CREATED)
async def create_exercise(
    data: ExerciseCreate,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
):
    exercise = Exercise(**data.model_dump())
    db.add(exercise)
    try:
        await db.commit()
        await db.refresh(exercise)
    except Exception:
        await db.rollback()
        raise HTTPException(status_code=409, detail="Exercise already exists")
    return {"data": ExerciseResponse.model_validate(exercise), "message": "Exercise created"}


@router.get("")
async def list_exercises(
    category: str | None = None,
    limit: int = Query(default=20, le=100),
    offset: int = Query(default=0, ge=0),
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
):
    query = select(Exercise)
    count_query = select(func.count()).select_from(Exercise)

    if category:
        query = query.where(Exercise.category.any(category))
        count_query = count_query.where(Exercise.category.any(category))

    total_result = await db.execute(count_query)
    total = total_result.scalar()

    query = query.limit(limit).offset(offset)
    result = await db.execute(query)
    exercises = result.scalars().all()

    return {
        "data": [ExerciseResponse.model_validate(e) for e in exercises],
        "total": total,
        "limit": limit,
        "offset": offset,
    }
```

- [ ] **Step 6: Register exercises router in main.py**

Add to `backend/app/main.py`:
```python
from app.routes import auth, profile, exercises

# ... after existing include_router calls:
app.include_router(exercises.router)
```

- [ ] **Step 7: Update models __init__.py**

```python
# backend/app/models/__init__.py
from app.models.user import User
from app.models.exercise import Exercise

__all__ = ["User", "Exercise"]
```

- [ ] **Step 8: Generate and run migration**

Run:
```bash
cd backend
alembic revision --autogenerate -m "create exercises table"
alembic upgrade head
```

- [ ] **Step 9: Run tests to verify they pass**

Run:
```bash
cd backend
python -m pytest tests/integration/test_exercises.py -v
```
Expected: All 4 tests PASS.

- [ ] **Step 10: Commit**

```bash
git add backend/
git commit -m "feat: add exercise model, schema, and CRUD routes with pagination"
```

---

## Task 7: Workout & ExerciseLog Models

**Files:**
- Create: `backend/app/models/workout.py`
- Create: `backend/app/models/exercise_log.py`

- [ ] **Step 1: Create workout model**

```python
# backend/app/models/workout.py
import enum
import uuid
from datetime import date, datetime

from sqlalchemy import Date, Float, Integer, String, Text, ForeignKey, Enum as SAEnum, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


class WorkoutType(str, enum.Enum):
    strength = "strength"
    cardio = "cardio"
    mixed = "mixed"
    competition = "competition"


class Workout(Base):
    __tablename__ = "workouts"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    date: Mapped[date] = mapped_column(Date, nullable=False)
    workout_type: Mapped[str] = mapped_column(
        SAEnum(WorkoutType, name="workout_type"), nullable=False
    )
    duration_minutes: Mapped[int | None] = mapped_column(Integer, nullable=True)
    notes: Mapped[str | None] = mapped_column(Text, nullable=True)
    score: Mapped[float | None] = mapped_column(Float, nullable=True, default=0.0)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )

    exercise_logs = relationship("ExerciseLog", back_populates="workout", cascade="all, delete-orphan")
```

- [ ] **Step 2: Create exercise log model**

```python
# backend/app/models/exercise_log.py
import uuid

from sqlalchemy import Integer, Float, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


class ExerciseLog(Base):
    __tablename__ = "exercise_logs"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    workout_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("workouts.id", ondelete="CASCADE"), nullable=False
    )
    exercise_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("exercises.id"), nullable=False
    )
    sets: Mapped[int | None] = mapped_column(Integer, nullable=True)
    reps: Mapped[int | None] = mapped_column(Integer, nullable=True)
    weight_kg: Mapped[float | None] = mapped_column(Float, nullable=True)
    distance_km: Mapped[float | None] = mapped_column(Float, nullable=True)
    pace: Mapped[float | None] = mapped_column(Float, nullable=True)
    duration_seconds: Mapped[int | None] = mapped_column(Integer, nullable=True)
    rpe: Mapped[int | None] = mapped_column(Integer, nullable=True)

    workout = relationship("Workout", back_populates="exercise_logs")
    exercise = relationship("Exercise")
```

- [ ] **Step 3: Update models __init__.py**

```python
# backend/app/models/__init__.py
from app.models.user import User
from app.models.exercise import Exercise
from app.models.workout import Workout
from app.models.exercise_log import ExerciseLog

__all__ = ["User", "Exercise", "Workout", "ExerciseLog"]
```

- [ ] **Step 4: Generate and run migration**

Run:
```bash
cd backend
alembic revision --autogenerate -m "create workouts and exercise_logs tables"
alembic upgrade head
```

- [ ] **Step 5: Commit**

```bash
git add backend/app/models/
git commit -m "feat: add Workout and ExerciseLog models with relationships"
```

---

## Task 8: Workout Schemas & Service

**Files:**
- Create: `backend/app/schemas/workout.py`
- Create: `backend/app/services/workout.py`

- [ ] **Step 1: Create workout schemas**

```python
# backend/app/schemas/workout.py
import uuid
from datetime import date, datetime

from pydantic import BaseModel, Field

from app.schemas.exercise import ExerciseResponse


class ExerciseLogCreate(BaseModel):
    exercise_id: uuid.UUID
    sets: int | None = Field(None, ge=1)
    reps: int | None = Field(None, ge=1)
    weight_kg: float | None = Field(None, ge=0)
    distance_km: float | None = Field(None, ge=0)
    pace: float | None = Field(None, ge=0)
    duration_seconds: int | None = Field(None, ge=0)
    rpe: int | None = Field(None, ge=1, le=10)


class WorkoutCreate(BaseModel):
    date: date
    workout_type: str = Field(pattern="^(strength|cardio|mixed|competition)$")
    duration_minutes: int | None = Field(None, ge=1)
    notes: str | None = None
    exercises: list[ExerciseLogCreate] = Field(min_length=1)


class ExerciseLogResponse(BaseModel):
    id: uuid.UUID
    exercise_id: uuid.UUID
    exercise: ExerciseResponse | None = None
    sets: int | None = None
    reps: int | None = None
    weight_kg: float | None = None
    distance_km: float | None = None
    pace: float | None = None
    duration_seconds: int | None = None
    rpe: int | None = None

    model_config = {"from_attributes": True}


class WorkoutResponse(BaseModel):
    id: uuid.UUID
    user_id: uuid.UUID
    date: date
    workout_type: str
    duration_minutes: int | None = None
    notes: str | None = None
    score: float | None = None
    exercise_logs: list[ExerciseLogResponse] = []
    created_at: datetime

    model_config = {"from_attributes": True}


class WorkoutUpdate(BaseModel):
    date: date | None = None
    workout_type: str | None = Field(None, pattern="^(strength|cardio|mixed|competition)$")
    duration_minutes: int | None = Field(None, ge=1)
    notes: str | None = None
    exercises: list[ExerciseLogCreate] | None = None
```

- [ ] **Step 2: Create workout service**

```python
# backend/app/services/workout.py
import logging
from uuid import UUID

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.models.exercise_log import ExerciseLog
from app.models.workout import Workout
from app.schemas.workout import WorkoutCreate, WorkoutUpdate

logger = logging.getLogger(__name__)


async def create_workout(db: AsyncSession, user_id: UUID, data: WorkoutCreate) -> Workout:
    workout = Workout(
        user_id=user_id,
        date=data.date,
        workout_type=data.workout_type,
        duration_minutes=data.duration_minutes,
        notes=data.notes,
    )
    db.add(workout)
    await db.flush()

    for ex in data.exercises:
        log = ExerciseLog(workout_id=workout.id, **ex.model_dump())
        db.add(log)

    await db.commit()

    # Reload with exercise_logs and their exercises
    result = await db.execute(
        select(Workout)
        .where(Workout.id == workout.id)
        .options(selectinload(Workout.exercise_logs).selectinload(ExerciseLog.exercise))
    )
    workout = result.scalar_one()
    logger.info("Workout created: workout_id=%s user_id=%s", workout.id, user_id)
    return workout


async def get_workout(db: AsyncSession, workout_id: UUID, user_id: UUID) -> Workout | None:
    result = await db.execute(
        select(Workout)
        .where(Workout.id == workout_id, Workout.user_id == user_id)
        .options(selectinload(Workout.exercise_logs).selectinload(ExerciseLog.exercise))
    )
    return result.scalar_one_or_none()


async def list_workouts(
    db: AsyncSession, user_id: UUID, limit: int = 20, offset: int = 0
) -> tuple[list[Workout], int]:
    count_result = await db.execute(
        select(func.count()).select_from(Workout).where(Workout.user_id == user_id)
    )
    total = count_result.scalar()

    result = await db.execute(
        select(Workout)
        .where(Workout.user_id == user_id)
        .options(selectinload(Workout.exercise_logs).selectinload(ExerciseLog.exercise))
        .order_by(Workout.date.desc())
        .limit(limit)
        .offset(offset)
    )
    workouts = result.scalars().all()
    return workouts, total


async def update_workout(
    db: AsyncSession, workout: Workout, data: WorkoutUpdate
) -> Workout:
    update_data = data.model_dump(exclude_unset=True, exclude={"exercises"})
    for field, value in update_data.items():
        setattr(workout, field, value)

    if data.exercises is not None:
        # Delete old logs, replace with new
        await db.execute(
            ExerciseLog.__table__.delete().where(ExerciseLog.workout_id == workout.id)
        )
        for ex in data.exercises:
            log = ExerciseLog(workout_id=workout.id, **ex.model_dump())
            db.add(log)

    await db.commit()

    # Reload
    result = await db.execute(
        select(Workout)
        .where(Workout.id == workout.id)
        .options(selectinload(Workout.exercise_logs).selectinload(ExerciseLog.exercise))
    )
    workout = result.scalar_one()
    logger.info("Workout updated: workout_id=%s", workout.id)
    return workout


async def delete_workout(db: AsyncSession, workout: Workout) -> None:
    # Capture before delete — ORM object is detached after commit
    user_id = workout.user_id
    workout_id = workout.id
    logger.info("Workout deleted: workout_id=%s user_id=%s", workout_id, user_id)
    await db.delete(workout)
    await db.commit()
    return user_id  # Caller uses this for score recalculation
```

- [ ] **Step 3: Commit**

```bash
git add backend/app/schemas/workout.py backend/app/services/workout.py
git commit -m "feat: add workout schemas and service (CRUD with exercise logs)"
```

---

## Task 9: Workout Routes & Tests

**Files:**
- Create: `backend/app/routes/workouts.py`
- Create: `backend/tests/integration/test_workouts.py`

- [ ] **Step 1: Write failing workout tests**

```python
# backend/tests/integration/test_workouts.py
import uuid
from datetime import date

import pytest
from httpx import AsyncClient


async def _create_exercise(client: AsyncClient, headers: dict, name: str = "Test Squat") -> str:
    """Helper: create an exercise and return its ID."""
    resp = await client.post("/exercises", json={
        "name": name,
        "category": ["bodybuilding"],
        "muscle_groups": ["quads"],
        "exercise_type": "strength",
        "difficulty": 4,
        "instructions": "Test exercise.",
    }, headers=headers)
    return resp.json()["data"]["id"]


@pytest.mark.asyncio
async def test_create_workout(client: AsyncClient, auth_headers: dict):
    ex_id = await _create_exercise(client, auth_headers, f"Squat_{uuid.uuid4().hex[:6]}")
    response = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "duration_minutes": 60,
        "notes": "Leg day",
        "exercises": [{
            "exercise_id": ex_id,
            "sets": 4,
            "reps": 8,
            "weight_kg": 100.0,
            "rpe": 8,
        }],
    }, headers=auth_headers)
    assert response.status_code == 201
    data = response.json()["data"]
    assert data["workout_type"] == "strength"
    assert len(data["exercise_logs"]) == 1
    assert data["exercise_logs"][0]["sets"] == 4


@pytest.mark.asyncio
async def test_create_workout_no_exercises_fails(client: AsyncClient, auth_headers: dict):
    response = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [],
    }, headers=auth_headers)
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_list_workouts(client: AsyncClient, auth_headers: dict):
    ex_id = await _create_exercise(client, auth_headers, f"Bench_{uuid.uuid4().hex[:6]}")
    await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 3, "reps": 10, "weight_kg": 60.0}],
    }, headers=auth_headers)
    response = await client.get("/workouts/history", headers=auth_headers)
    assert response.status_code == 200
    assert len(response.json()["data"]) >= 1
    assert "total" in response.json()


@pytest.mark.asyncio
async def test_get_workout(client: AsyncClient, auth_headers: dict):
    ex_id = await _create_exercise(client, auth_headers, f"DL_{uuid.uuid4().hex[:6]}")
    create_resp = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 5, "reps": 5, "weight_kg": 140.0}],
    }, headers=auth_headers)
    workout_id = create_resp.json()["data"]["id"]
    response = await client.get(f"/workouts/{workout_id}", headers=auth_headers)
    assert response.status_code == 200
    assert response.json()["data"]["id"] == workout_id


@pytest.mark.asyncio
async def test_update_workout(client: AsyncClient, auth_headers: dict):
    ex_id = await _create_exercise(client, auth_headers, f"OHP_{uuid.uuid4().hex[:6]}")
    create_resp = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 3, "reps": 8, "weight_kg": 50.0}],
    }, headers=auth_headers)
    workout_id = create_resp.json()["data"]["id"]
    response = await client.put(f"/workouts/{workout_id}", json={
        "notes": "Updated notes",
    }, headers=auth_headers)
    assert response.status_code == 200
    assert response.json()["data"]["notes"] == "Updated notes"


@pytest.mark.asyncio
async def test_delete_workout(client: AsyncClient, auth_headers: dict):
    ex_id = await _create_exercise(client, auth_headers, f"Row_{uuid.uuid4().hex[:6]}")
    create_resp = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 4, "reps": 10, "weight_kg": 80.0}],
    }, headers=auth_headers)
    workout_id = create_resp.json()["data"]["id"]
    response = await client.delete(f"/workouts/{workout_id}", headers=auth_headers)
    assert response.status_code == 200

    get_resp = await client.get(f"/workouts/{workout_id}", headers=auth_headers)
    assert get_resp.status_code == 404
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
cd backend
python -m pytest tests/integration/test_workouts.py -v
```
Expected: FAIL — workout routes don't exist.

- [ ] **Step 3: Create workout routes**

```python
# backend/app/routes/workouts.py
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.middleware.auth import get_current_user
from app.models.user import User
from app.schemas.workout import WorkoutCreate, WorkoutResponse, WorkoutUpdate
from app.services import workout as workout_service

router = APIRouter(prefix="/workouts", tags=["Workouts"])


@router.post("", status_code=status.HTTP_201_CREATED)
async def create_workout(
    data: WorkoutCreate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    workout = await workout_service.create_workout(db, user.id, data)
    return {"data": WorkoutResponse.model_validate(workout), "message": "Workout logged"}


@router.get("/history")
async def list_workouts(
    limit: int = Query(default=20, le=100),
    offset: int = Query(default=0, ge=0),
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    workouts, total = await workout_service.list_workouts(db, user.id, limit, offset)
    return {
        "data": [WorkoutResponse.model_validate(w) for w in workouts],
        "total": total,
        "limit": limit,
        "offset": offset,
    }


@router.get("/{workout_id}")
async def get_workout(
    workout_id: UUID,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    workout = await workout_service.get_workout(db, workout_id, user.id)
    if not workout:
        raise HTTPException(status_code=404, detail="Workout not found")
    return {"data": WorkoutResponse.model_validate(workout)}


@router.put("/{workout_id}")
async def update_workout(
    workout_id: UUID,
    data: WorkoutUpdate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    workout = await workout_service.get_workout(db, workout_id, user.id)
    if not workout:
        raise HTTPException(status_code=404, detail="Workout not found")
    updated = await workout_service.update_workout(db, workout, data)
    return {"data": WorkoutResponse.model_validate(updated), "message": "Workout updated"}


@router.delete("/{workout_id}")
async def delete_workout(
    workout_id: UUID,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    workout = await workout_service.get_workout(db, workout_id, user.id)
    if not workout:
        raise HTTPException(status_code=404, detail="Workout not found")
    await workout_service.delete_workout(db, workout)
    return {"message": "Workout deleted"}
```

- [ ] **Step 4: Register workouts router in main.py**

Add to `backend/app/main.py`:
```python
from app.routes import auth, profile, exercises, workouts

# ... add:
app.include_router(workouts.router)
```

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
cd backend
python -m pytest tests/integration/test_workouts.py -v
```
Expected: All 6 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add backend/
git commit -m "feat: add workout CRUD with exercise logs, pagination, and tests"
```

---

## Task 10: Simple Volume Scoring

**Files:**
- Create: `backend/app/models/score.py`
- Create: `backend/app/schemas/score.py`
- Create: `backend/app/services/scoring.py`
- Create: `backend/app/routes/scores.py`
- Create: `backend/tests/unit/test_scoring.py`

**Note:** `exercise_alternatives` junction table from the design spec is deferred to Phase 2/3 (needed for AI coach injury-based suggestions, not for Phase 1 scoring).

- [ ] **Step 1: Create score model first (scoring service imports it)**

```python
# backend/app/models/score.py
import uuid
from datetime import datetime

from sqlalchemy import Float, Integer, String, ForeignKey, UniqueConstraint, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class Score(Base):
    __tablename__ = "scores"
    __table_args__ = (
        UniqueConstraint("user_id", "category", "period", name="uq_user_category_period"),
    )

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    category: Mapped[str] = mapped_column(String(50), nullable=False)
    period: Mapped[str] = mapped_column(String(20), nullable=False, default="all_time")
    total_score: Mapped[float] = mapped_column(Float, default=0.0)
    volume_score: Mapped[float] = mapped_column(Float, default=0.0)
    consistency_score: Mapped[float] = mapped_column(Float, default=0.0)
    intensity_score: Mapped[float] = mapped_column(Float, default=0.0)
    streak_days: Mapped[int] = mapped_column(Integer, default=0)
    calculated_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

- [ ] **Step 2: Write failing scoring unit tests**

```python
# backend/tests/unit/test_scoring.py
import pytest

from app.services.scoring import calculate_volume_score


def test_strength_volume():
    """4 sets × 8 reps × 100kg × difficulty 4 = 12800"""
    exercise_logs = [
        {"sets": 4, "reps": 8, "weight_kg": 100.0, "difficulty": 4,
         "distance_km": None, "pace": None, "duration_seconds": None},
    ]
    score = calculate_volume_score(exercise_logs)
    assert score == 12800.0


def test_cardio_volume():
    """5km at 5.0 min/km pace → pace_multiplier = 10/5 = 2.0 → 5 × 2.0 = 10.0"""
    exercise_logs = [
        {"sets": None, "reps": None, "weight_kg": None, "difficulty": 2,
         "distance_km": 5.0, "pace": 5.0, "duration_seconds": None},
    ]
    score = calculate_volume_score(exercise_logs)
    assert score == 10.0


def test_timed_volume():
    """120 seconds × difficulty 3 = 360"""
    exercise_logs = [
        {"sets": None, "reps": None, "weight_kg": None, "difficulty": 3,
         "distance_km": None, "pace": None, "duration_seconds": 120},
    ]
    score = calculate_volume_score(exercise_logs)
    assert score == 360.0


def test_mixed_volume():
    """Strength + cardio in same workout."""
    exercise_logs = [
        {"sets": 3, "reps": 10, "weight_kg": 80.0, "difficulty": 3,
         "distance_km": None, "pace": None, "duration_seconds": None},
        {"sets": None, "reps": None, "weight_kg": None, "difficulty": 2,
         "distance_km": 3.0, "pace": 6.0, "duration_seconds": None},
    ]
    score = calculate_volume_score(exercise_logs)
    # Strength: 3×10×80×3 = 7200
    # Cardio: 3.0 × (10/6) = 5.0
    assert score == pytest.approx(7205.0, rel=0.01)


def test_empty_logs():
    score = calculate_volume_score([])
    assert score == 0.0
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
cd backend
python -m pytest tests/unit/test_scoring.py -v
```
Expected: FAIL — `calculate_volume_score` doesn't exist.

- [ ] **Step 3: Create scoring service**

```python
# backend/app/services/scoring.py
import logging
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.models.exercise_log import ExerciseLog
from app.models.score import Score
from app.models.workout import Workout

logger = logging.getLogger(__name__)


def calculate_volume_score(exercise_logs: list[dict]) -> float:
    """Calculate total volume score from exercise log dicts.

    Each dict must have: sets, reps, weight_kg, difficulty, distance_km, pace, duration_seconds.

    Strength volume: sets × reps × weight_kg × difficulty
    Cardio volume: distance_km × (10 / pace)
    Timed volume: duration_seconds × difficulty

    Phase 1 note: Returns raw volume without normalization. This means scores
    are NOT comparable across categories (a bodybuilder scores ~10,000s while a
    runner scores ~10s). Category-specific normalization is added in Phase 2
    when the full scoring formula (consistency + volume + intensity) is implemented.
    """
    total = 0.0
    for log in exercise_logs:
        sets = log.get("sets")
        reps = log.get("reps")
        weight = log.get("weight_kg")
        difficulty = log.get("difficulty", 1)
        distance = log.get("distance_km")
        pace = log.get("pace")
        duration = log.get("duration_seconds")

        if sets and reps and weight:
            total += sets * reps * weight * difficulty
        elif distance and pace and pace > 0:
            total += distance * (10.0 / pace)
        elif duration:
            total += duration * difficulty

    return total


async def recalculate_user_scores(db: AsyncSession, user_id: UUID, category: str) -> None:
    """Recalculate the volume-based score for a user.

    Phase 1: volume_score only, stored as total_score.
    Phase 2 will add consistency and intensity components.
    """
    result = await db.execute(
        select(Workout)
        .where(Workout.user_id == user_id)
        .options(selectinload(Workout.exercise_logs).selectinload(ExerciseLog.exercise))
    )
    workouts = result.scalars().all()

    # Build exercise log dicts with difficulty from exercise model
    all_logs = []
    for workout in workouts:
        for log in workout.exercise_logs:
            all_logs.append({
                "sets": log.sets,
                "reps": log.reps,
                "weight_kg": log.weight_kg,
                "difficulty": log.exercise.difficulty if log.exercise else 1,
                "distance_km": log.distance_km,
                "pace": log.pace,
                "duration_seconds": log.duration_seconds,
            })

    volume = calculate_volume_score(all_logs)

    # Upsert score — Phase 1 uses only volume, stored as total_score
    existing = await db.execute(
        select(Score).where(
            Score.user_id == user_id,
            Score.category == category,
            Score.period == "all_time",
        )
    )
    score = existing.scalar_one_or_none()

    if score:
        old_score = score.total_score
        score.volume_score = volume
        score.total_score = volume
    else:
        old_score = 0.0
        score = Score(
            user_id=user_id,
            category=category,
            period="all_time",
            volume_score=volume,
            total_score=volume,
        )
        db.add(score)

    await db.commit()
    logger.info(
        "Score recalculated: user_id=%s old=%.1f new=%.1f",
        user_id, old_score, volume,
    )

    # Also update the per-workout scores
    for workout in workouts:
        logs = []
        for log in workout.exercise_logs:
            logs.append({
                "sets": log.sets,
                "reps": log.reps,
                "weight_kg": log.weight_kg,
                "difficulty": log.exercise.difficulty if log.exercise else 1,
                "distance_km": log.distance_km,
                "pace": log.pace,
                "duration_seconds": log.duration_seconds,
            })
        workout.score = calculate_volume_score(logs)

    await db.commit()
```

- [ ] **Step 5: Create score schema**

```python
# backend/app/schemas/score.py
import uuid
from datetime import datetime

from pydantic import BaseModel


class ScoreResponse(BaseModel):
    id: uuid.UUID
    user_id: uuid.UUID
    category: str
    period: str
    total_score: float
    volume_score: float
    consistency_score: float
    intensity_score: float
    streak_days: int
    calculated_at: datetime

    model_config = {"from_attributes": True}
```

- [ ] **Step 6: Create score route**

```python
# backend/app/routes/scores.py
from fastapi import APIRouter, Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.middleware.auth import get_current_user
from app.models.score import Score
from app.models.user import User
from app.schemas.score import ScoreResponse

router = APIRouter(prefix="/scores", tags=["Scores"])


@router.get("/me")
async def get_my_scores(
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Score).where(Score.user_id == user.id)
    )
    scores = result.scalars().all()
    return {"data": [ScoreResponse.model_validate(s) for s in scores]}
```

- [ ] **Step 7: Integrate scoring into workout service**

Add to the end of `create_workout`, `update_workout`, and `delete_workout` in `backend/app/services/workout.py`:

```python
from app.services.scoring import recalculate_user_scores
```

In `create_workout`, after `logger.info(...)`:
```python
    # Recalculate scores after logging workout
    user_result = await db.execute(select(User).where(User.id == user_id))
    user = user_result.scalar_one()
    await recalculate_user_scores(db, user_id, user.category)
    return workout
```

In `update_workout`, after `logger.info(...)`:
```python
    user_result = await db.execute(select(User).where(User.id == workout.user_id))
    user = user_result.scalar_one()
    await recalculate_user_scores(db, workout.user_id, user.category)
    return workout
```

In `delete_workout`, after `await db.commit()` — use the returned `user_id` (captured before delete):
```python
    user_id = await workout_service.delete_workout(db, workout)
    user_result = await db.execute(select(User).where(User.id == user_id))
    user = user_result.scalar_one()
    await recalculate_user_scores(db, user_id, user.category)
```

**Note**: The `delete_workout` function now returns `user_id` because the ORM object is detached after commit. The route handler in `workouts.py` should call scoring directly after getting the user_id back. Update `delete_workout` in the routes to:
```python
@router.delete("/{workout_id}")
async def delete_workout(
    workout_id: UUID,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    workout = await workout_service.get_workout(db, workout_id, user.id)
    if not workout:
        raise HTTPException(status_code=404, detail="Workout not found")
    await workout_service.delete_workout(db, workout)
    await recalculate_user_scores(db, user.id, user.category)
    return {"message": "Workout deleted"}
```

This is cleaner — the route already has the authenticated `user` object, so it passes `user.id` and `user.category` directly to the scoring service instead of re-querying after delete.

Add the import at top of `workout.py`:
```python
from sqlalchemy import func, select
from app.models.user import User
from app.services.scoring import recalculate_user_scores
```

- [ ] **Step 8: Update models __init__.py and main.py**

```python
# backend/app/models/__init__.py
from app.models.user import User
from app.models.exercise import Exercise
from app.models.workout import Workout
from app.models.exercise_log import ExerciseLog
from app.models.score import Score

__all__ = ["User", "Exercise", "Workout", "ExerciseLog", "Score"]
```

Add to `backend/app/main.py`:
```python
from app.routes import auth, profile, exercises, workouts, scores

app.include_router(scores.router)
```

- [ ] **Step 9: Generate and run migration**

Run:
```bash
cd backend
alembic revision --autogenerate -m "create scores table"
alembic upgrade head
```

- [ ] **Step 10: Run all tests**

Run:
```bash
cd backend
python -m pytest tests/ -v
```
Expected: All unit and integration tests PASS — scoring unit tests pass, workout tests now trigger score recalculation.

- [ ] **Step 11: Commit**

```bash
git add backend/
git commit -m "feat: add volume-based scoring with automatic recalculation on workout changes"
```

---

## Task 11: Profile Tests

**Files:**
- Create: `backend/tests/integration/test_profile.py`

- [ ] **Step 1: Write profile tests**

```python
# backend/tests/integration/test_profile.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_get_profile(client: AsyncClient, auth_headers: dict):
    response = await client.get("/profile", headers=auth_headers)
    assert response.status_code == 200
    data = response.json()["data"]
    assert "email" in data
    assert "username" in data
    assert "category" in data


@pytest.mark.asyncio
async def test_update_profile(client: AsyncClient, auth_headers: dict):
    response = await client.put("/profile", json={
        "bio": "Lifting heavy things",
        "height_cm": 180.0,
        "weight_kg": 85.0,
    }, headers=auth_headers)
    assert response.status_code == 200
    data = response.json()["data"]
    assert data["bio"] == "Lifting heavy things"
    assert data["height_cm"] == 180.0


@pytest.mark.asyncio
async def test_get_profile_unauthenticated(client: AsyncClient):
    response = await client.get("/profile")
    assert response.status_code == 403
```

- [ ] **Step 2: Run tests**

Run:
```bash
cd backend
python -m pytest tests/integration/test_profile.py -v
```
Expected: All 3 tests PASS (routes already exist from Task 5).

- [ ] **Step 3: Commit**

```bash
git add backend/tests/integration/test_profile.py
git commit -m "test: add profile endpoint integration tests"
```

---

## Task 12: Scoring Integration Tests

**Files:**
- Create: `backend/tests/integration/test_scoring_integration.py`

- [ ] **Step 1: Write scoring integration tests**

These verify the critical path: workout changes trigger score recalculation.

```python
# backend/tests/integration/test_scoring_integration.py
import uuid
from datetime import date

import pytest
from httpx import AsyncClient


async def _create_exercise(client: AsyncClient, headers: dict, name: str, difficulty: int = 4) -> str:
    resp = await client.post("/exercises", json={
        "name": name,
        "category": ["bodybuilding"],
        "muscle_groups": ["quads"],
        "exercise_type": "strength",
        "difficulty": difficulty,
        "instructions": "Test.",
    }, headers=headers)
    return resp.json()["data"]["id"]


@pytest.mark.asyncio
async def test_score_created_on_workout_log(client: AsyncClient, auth_headers: dict):
    """Logging a workout should create a score entry."""
    ex_id = await _create_exercise(client, auth_headers, f"ScoreTest_{uuid.uuid4().hex[:6]}")
    await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 4, "reps": 8, "weight_kg": 100.0}],
    }, headers=auth_headers)

    response = await client.get("/scores/me", headers=auth_headers)
    assert response.status_code == 200
    scores = response.json()["data"]
    assert len(scores) >= 1
    # 4 sets × 8 reps × 100kg × difficulty 4 = 12800
    assert scores[0]["volume_score"] == 12800.0
    assert scores[0]["total_score"] == 12800.0


@pytest.mark.asyncio
async def test_score_updates_on_workout_edit(client: AsyncClient, auth_headers: dict):
    """Editing a workout should recalculate the score."""
    ex_id = await _create_exercise(client, auth_headers, f"EditScore_{uuid.uuid4().hex[:6]}", difficulty=3)
    create_resp = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 3, "reps": 10, "weight_kg": 50.0}],
    }, headers=auth_headers)
    workout_id = create_resp.json()["data"]["id"]

    # Update: change to 5×5×80kg
    await client.put(f"/workouts/{workout_id}", json={
        "exercises": [{"exercise_id": ex_id, "sets": 5, "reps": 5, "weight_kg": 80.0}],
    }, headers=auth_headers)

    response = await client.get("/scores/me", headers=auth_headers)
    scores = response.json()["data"]
    # 5 × 5 × 80 × 3 = 6000
    assert scores[0]["volume_score"] == 6000.0


@pytest.mark.asyncio
async def test_score_updates_on_workout_delete(client: AsyncClient, auth_headers: dict):
    """Deleting a workout should recalculate the score (to 0 if only workout)."""
    ex_id = await _create_exercise(client, auth_headers, f"DelScore_{uuid.uuid4().hex[:6]}")
    create_resp = await client.post("/workouts", json={
        "date": str(date.today()),
        "workout_type": "strength",
        "exercises": [{"exercise_id": ex_id, "sets": 3, "reps": 10, "weight_kg": 60.0}],
    }, headers=auth_headers)
    workout_id = create_resp.json()["data"]["id"]

    # Verify score exists
    score_resp = await client.get("/scores/me", headers=auth_headers)
    assert score_resp.json()["data"][0]["total_score"] > 0

    # Delete the workout
    await client.delete(f"/workouts/{workout_id}", headers=auth_headers)

    # Score should be 0 (no workouts left)
    score_resp = await client.get("/scores/me", headers=auth_headers)
    assert score_resp.json()["data"][0]["total_score"] == 0.0
```

- [ ] **Step 2: Run tests**

Run:
```bash
cd backend
python -m pytest tests/integration/test_scoring_integration.py -v
```
Expected: All 3 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add backend/tests/integration/test_scoring_integration.py
git commit -m "test: add scoring integration tests for create, update, and delete triggers"
```

---

## Task 13: pytest Configuration

**Files:**
- Create: `backend/pyproject.toml`

- [ ] **Step 1: Create pyproject.toml**

```toml
# backend/pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

This silences pytest-asyncio warnings about default mode and sets the test discovery path.

- [ ] **Step 2: Commit**

```bash
git add backend/pyproject.toml
git commit -m "chore: add pytest configuration"
```

---

## Task 14: Final Verification & Cleanup

- [ ] **Step 1: Run the full test suite**

Run:
```bash
cd backend
python -m pytest tests/ -v --tb=short
```
Expected: All tests PASS.

- [ ] **Step 2: Start the server and verify Swagger UI**

Run:
```bash
cd backend
uvicorn app.main:app --reload
```
Open `http://localhost:8000/docs` — verify all endpoints appear:
- POST /register, POST /login
- GET/PUT /profile, GET /users/{id}
- GET/POST /exercises
- POST /workouts, GET /workouts/history, GET/PUT/DELETE /workouts/{id}
- GET /scores/me
- GET /health

- [ ] **Step 3: Manual smoke test via Swagger**

1. Register a user
2. Login → copy token
3. Create an exercise (Barbell Squat, difficulty 4)
4. Log a workout with that exercise (4×8×100kg)
5. Check GET /scores/me → should show volume_score = 12800.0
6. Edit the workout (change to 5×5×120kg)
7. Check GET /scores/me → score should update (5×5×120×4 = 12000.0)

- [ ] **Step 4: Commit any final fixes**

```bash
git add -A
git commit -m "chore: Phase 1 complete — core backend with auth, exercises, workouts, and scoring"
```

- [ ] **Step 5: Push to GitHub**

```bash
git push origin main
```

---

## Summary

Phase 1 delivers 12 API endpoints:

| Endpoint | Purpose |
|----------|---------|
| `POST /register` | Create account |
| `POST /login` | Get JWT token |
| `GET /profile` | View own profile |
| `PUT /profile` | Edit profile |
| `GET /users/{id}` | View another user |
| `POST /exercises` | Add exercise to library |
| `GET /exercises` | Browse exercises (filterable, paginated) |
| `POST /workouts` | Log a workout (triggers score recalc) |
| `GET /workouts/history` | List own workouts (paginated) |
| `GET /workouts/{id}` | View single workout |
| `PUT /workouts/{id}` | Edit workout (triggers score recalc) |
| `DELETE /workouts/{id}` | Delete workout (triggers score recalc) |
| `GET /scores/me` | View own scores |
| `GET /health` | Health check |

All tested, all following the constitution's separation of concerns, and ready for Phase 2 (full scoring + leaderboards).
