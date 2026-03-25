# DEPSENTRY_DATABASE_AND_API_FOUNDATION.md
_Prepared by: CD | Date: March 2026_

---

## Overview

Stand up the database layer and API response conventions for DepSentry. After this task, the project has SQLAlchemy models for User and Project, Alembic migrations, a consistent response envelope on every endpoint, centralized error handling, and a health check. This is the foundation — auth, CRUD, and scanning all depend on these patterns being in place first.

| Field | Detail |
|---|---|
| **Scope** | Database models, migrations, response envelope, error handling middleware, health endpoint. No auth, no CRUD, no Docker. |
| **Risk** | Low — greenfield setup, no existing behavior to break |
| **Stack** | Python 3.12+, FastAPI, SQLAlchemy 2.0 (async), Alembic, SQLite (local dev) |
| **Files affected** | `app/models/`, `app/core/`, `app/api/`, `alembic/`, `alembic.ini`, `.env.example` |
| **Depends on** | Project bootstrap complete (FastAPI skeleton, `pyproject.toml` or `requirements.txt`, basic folder structure) |
| **Blocks** | AUTH_SYSTEM task doc (needs User model + DB session), PROJECT_CRUD task doc (needs Project model + response envelope) |

---

## Context

This is the first implementation task for DepSentry after project bootstrap. The spec (PROJECT_2_DEPSENTRY_SPEC.md) defines Phase 1 as "Working API skeleton with auth, database, and one working endpoint." That's too wide for a single task — this doc covers the database and API conventions layer only.

**Constraints:**
- Use SQLite for local dev to avoid requiring PostgreSQL for initial development. PostgreSQL will come with Docker in a later task.
- Use SQLAlchemy 2.0-style async patterns — not the legacy 1.x `Query` API.
- Response envelope must match the format defined below exactly. Every future endpoint will use it.

**Architecture decisions to capture in the measurement framework after this task:**
- Why SQLAlchemy 2.0 async over sync
- Why SQLite for local dev vs. PostgreSQL from the start
- Why Alembic over raw SQL migrations
- Response envelope format rationale

---

## Goal

After this task completes, the following must be true:
1. SQLAlchemy async engine and session factory are configured and working.
2. `User` and `Project` models exist with an Alembic migration that creates their tables.
3. Every API response — success and error — uses a consistent JSON envelope.
4. Unhandled exceptions return a safe, structured error (no stack traces, no internal paths).
5. A `/v1/health` endpoint returns a 200 with the envelope format.
6. `.env.example` documents all required environment variables.

---

## Tasks

### Task 1 — Project structure conventions

Ensure the following directory structure exists. Create any missing directories/files. Do not reorganize or rename anything the bootstrap already created unless it directly conflicts.

```
depsentry/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app instance, middleware registration
│   ├── config.py                # Settings via pydantic-settings
│   ├── core/
│   │   ├── __init__.py
│   │   ├── database.py          # Engine, session factory, Base
│   │   └── response.py          # Envelope helpers
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── project.py
│   └── api/
│       ├── __init__.py
│       └── v1/
│           ├── __init__.py
│           └── health.py        # Health endpoint router
├── alembic/
│   └── versions/
├── alembic.ini
├── .env.example
├── .gitignore
└── requirements.txt (or pyproject.toml — use whichever bootstrap created)
```

**Footgun:** Do not install packages beyond what's listed in Task 2. Do not add test infrastructure yet — that's a separate task doc.

---

### Task 2 — Dependencies

Add these to `requirements.txt` (or `pyproject.toml` — match whatever bootstrap created):

```
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
sqlalchemy[asyncio]>=2.0.0
aiosqlite>=0.20.0
alembic>=1.13.0
pydantic>=2.0.0
pydantic-settings>=2.0.0
python-dotenv>=1.0.0
```

**Footgun:** Do not add `asyncpg`, `psycopg2`, `pytest`, `httpx`, or any auth libraries (PyJWT, passlib, bcrypt). Those come in later task docs. Only install what's listed above.

---

### Task 3 — Configuration (`app/config.py`)

Use `pydantic-settings` to load config from environment variables with `.env` file fallback.

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # App
    APP_NAME: str = "DepSentry"
    DEBUG: bool = False

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./depsentry.db"

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()
```

Create `.env.example`:

```bash
# App
APP_NAME=DepSentry
DEBUG=true

# Database — SQLite for local dev. PostgreSQL for Docker (added later).
DATABASE_URL=sqlite+aiosqlite:///./depsentry.db
# For PostgreSQL (future): postgresql+asyncpg://user:password@localhost:5432/depsentry
```

Ensure `.gitignore` includes:
```
.env
.env.local
*.db
__pycache__/
```

---

### Task 4 — Database engine and session (`app/core/database.py`)

Set up SQLAlchemy 2.0 async engine and session factory. Provide a `get_db` dependency for FastAPI.

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase

from app.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
)

async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_db():
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

**Footgun:** `expire_on_commit=False` is intentional — without it, accessing attributes after commit raises `MissingGreenlet` errors in async SQLAlchemy. Do not remove it.

---

### Task 5 — Models (`app/models/user.py`, `app/models/project.py`)

#### 5A — User model

```python
import uuid
from datetime import datetime, timezone
from sqlalchemy import String, DateTime, Boolean
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.core.database import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[str] = mapped_column(
        String(36), primary_key=True, default=lambda: str(uuid.uuid4())
    )
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    # Relationships
    projects: Mapped[list["Project"]] = relationship(back_populates="owner")
```

#### 5B — Project model

```python
import uuid
from datetime import datetime, timezone
from sqlalchemy import String, DateTime, ForeignKey, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.core.database import Base


class Project(Base):
    __tablename__ = "projects"

    id: Mapped[str] = mapped_column(
        String(36), primary_key=True, default=lambda: str(uuid.uuid4())
    )
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    ecosystem: Mapped[str] = mapped_column(String(50), nullable=False)  # "npm" | "pip"
    owner_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("users.id"), nullable=False, index=True
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    # Relationships
    owner: Mapped["User"] = relationship(back_populates="projects")
```

#### 5C — Model exports (`app/models/__init__.py`)

```python
from app.models.user import User
from app.models.project import Project

__all__ = ["User", "Project"]
```

**Design note:** UUIDs stored as strings (not native UUID type) for SQLite compatibility. When PostgreSQL is added later, these can stay as strings or migrate to native `UUID` — that's a future decision.

**Footgun:** The `ecosystem` field uses a plain string, not an enum. This is intentional — adding enum values in SQLite requires table recreation. Validation of allowed values ("npm", "pip") will happen at the Pydantic layer in the CRUD task doc, not at the DB level.

---

### Task 6 — Alembic setup and initial migration

#### 6A — Initialize Alembic

Run from project root:
```bash
alembic init alembic
```

#### 6B — Configure `alembic.ini`

Set `sqlalchemy.url` to empty (we'll override from code):
```ini
sqlalchemy.url =
```

#### 6C — Configure `alembic/env.py`

Replace the generated `env.py` with async-compatible config:

```python
import asyncio
from logging.config import fileConfig

from sqlalchemy import pool
from sqlalchemy.ext.asyncio import create_async_engine
from alembic import context

from app.config import settings
from app.core.database import Base
from app.models import User, Project  # noqa: F401 — must import to register models

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = settings.DATABASE_URL
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    connectable = create_async_engine(
        settings.DATABASE_URL,
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

#### 6D — Generate and run initial migration

```bash
alembic revision --autogenerate -m "create users and projects tables"
alembic upgrade head
```

**Footgun:** The `# noqa: F401` import comment on the model imports is required. Without importing the models in `env.py`, Alembic's autogenerate won't detect them. Do not remove those imports even though they appear unused.

---

### Task 7 — Response envelope (`app/core/response.py`)

All API responses must use this envelope. No endpoint should return a naked dict or model.

```python
from datetime import datetime, timezone
from typing import Any
import uuid

from fastapi.responses import JSONResponse


def _meta() -> dict:
    return {
        "requestId": f"req_{uuid.uuid4().hex[:12]}",
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }


def success_response(data: Any, status_code: int = 200) -> JSONResponse:
    return JSONResponse(
        status_code=status_code,
        content={
            "ok": True,
            "data": data,
            "meta": _meta(),
        },
    )


def error_response(
    code: str,
    message: str,
    status_code: int = 400,
    details: list[dict] | None = None,
) -> JSONResponse:
    content: dict[str, Any] = {
        "ok": False,
        "error": {
            "code": code,
            "message": message,
        },
        "meta": _meta(),
    }
    if details:
        content["error"]["details"] = details
    return JSONResponse(status_code=status_code, content=content)
```

---

### Task 8 — Error handling middleware (`app/main.py`)

Register exception handlers on the FastAPI app so unhandled errors and validation errors return the envelope format. **Never expose stack traces, file paths, or internal error details.**

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError

from app.core.response import error_response

app = FastAPI(title="DepSentry", version="0.1.0")


@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    details = []
    for error in exc.errors():
        field = " → ".join(str(loc) for loc in error["loc"])
        details.append({"field": field, "issue": error["msg"]})
    return error_response(
        code="VALIDATION_ERROR",
        message="Request validation failed",
        status_code=422,
        details=details,
    )


@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    # Log the real error server-side (logging setup comes in a later task)
    # For now, print to stdout so it's visible during development
    print(f"[ERROR] Unhandled exception: {type(exc).__name__}: {exc}")
    return error_response(
        code="INTERNAL_ERROR",
        message="An unexpected error occurred",
        status_code=500,
    )
```

After the exception handlers, include the router registration:

```python
from app.api.v1.health import router as health_router

app.include_router(health_router, prefix="/v1")
```

**Footgun:** The generic `Exception` handler must come last. FastAPI matches exception handlers by specificity. The `RequestValidationError` handler must be registered before the generic one, or it will never trigger.

**Footgun:** Do not add CORS middleware in this task. CORS config comes in the security hardening task doc. Adding it now with permissive defaults creates a security pattern that's easy to forget to tighten later.

---

### Task 9 — Health endpoint (`app/api/v1/health.py`)

```python
from fastapi import APIRouter

from app.core.response import success_response

router = APIRouter(tags=["health"])


@router.get("/health")
async def health_check():
    return success_response(
        data={"status": "healthy", "service": "depsentry"},
    )
```

This endpoint does not check DB connectivity — that's a `/ready` endpoint concern for a later task. `/health` confirms the API process is running and responding.

---

## Out of Scope

- **No auth.** No JWT, no password hashing, no login endpoints, no auth middleware. That's the next task doc.
- **No CRUD endpoints.** No project create/read/update/delete. That depends on auth being in place.
- **No Docker.** No Dockerfile, no docker-compose.yml. That's a later task.
- **No tests.** No pytest, no test files, no test infrastructure. Separate task doc.
- **No CORS middleware.** Comes with security hardening.
- **No additional models** (Scan, Dependency, Finding). Those come in Phase 2.
- **Do not add `asyncpg` or PostgreSQL drivers.** SQLite only for now.
- **Do not create seed data or management scripts.**
- **Do not refactor the bootstrap's `main.py` beyond what's specified in Task 8.** If bootstrap created a root endpoint or hello-world route, leave it — it'll be cleaned up in a later pass.

---

## Acceptance Criteria

- [ ] `pip install -r requirements.txt` (or equivalent) succeeds with no errors
- [ ] `.env.example` exists and documents `DATABASE_URL`, `APP_NAME`, `DEBUG`
- [ ] `.env` and `*.db` are in `.gitignore`
- [ ] `alembic upgrade head` creates a `depsentry.db` file with `users` and `projects` tables
- [ ] `alembic downgrade base` drops both tables cleanly (migration is reversible)
- [ ] `uvicorn app.main:app --reload` starts without errors
- [ ] `GET /v1/health` returns `200` with body: `{"ok": true, "data": {"status": "healthy", "service": "depsentry"}, "meta": {"requestId": "req_...", "timestamp": "..."}}`
- [ ] Hitting a nonexistent route returns `{"ok": false, "error": {"code": "NOT_FOUND", ...}}` — **not** FastAPI's default HTML 404 (note: this may require adding a catch-all or customizing the 404 handler in Task 8; if FastAPI's default 404 is already JSON, verify it matches the envelope format)
- [ ] Sending an invalid request body to any future endpoint (test with health by adding a temp Pydantic body) returns the `VALIDATION_ERROR` envelope with field-level details
- [ ] No stack traces, file paths, or internal error details appear in any error response body
- [ ] `app/models/__init__.py` exports both `User` and `Project`
- [ ] All model files use SQLAlchemy 2.0 `Mapped` / `mapped_column` syntax — no legacy `Column()` calls

---

_Pass this document directly to CC. All tasks are self-contained and executable sequentially._
_Archive to: docs/archive/DEPSENTRY_DATABASE_AND_API_FOUNDATION.md after completion._