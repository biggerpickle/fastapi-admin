# Learning Plan — FastAPI Admin API

**Goal:** build a small but production-shaped Admin API (users, roles, CRUD resources) while learning:

1. **JWT authentication** (access + refresh tokens, password hashing, role-based access)
2. **SQLAlchemy ORM 2.0** (models, relationships, sessions, migrations, query patterns)
3. **Professional response design** (consistent envelope, typed schemas, error handling, pagination)
4. **Real dev workflow** — Postgres in Docker, a Makefile, structured logging, a containerized app

**Approach:** one project, built in layers. Each phase ends with a checkpoint you can verify. Don't skip phases — each one assumes the previous.

**Estimated pace:** ~2–4 hours per phase. 15 phases total.

---

## Target project

An admin backend for a simple content system:

- `users` — login, roles (`admin`, `staff`), profile
- `categories` — simple lookup table
- `articles` — belongs to a user and a category (tests relationships)
- `audit_logs` — write-only trail (tests "fire and forget" writes)

Everything an admin panel needs: auth, list + filter + paginate, create/update/delete, consistent errors.

---

## Target folder structure

Build toward this. Don't create it all on day one — add folders as phases need them.

```
fastapi-admin/
├── app/
│   ├── main.py              # FastAPI() instance, router includes, exception handlers
│   ├── core/
│   │   ├── config.py        # Settings via pydantic-settings
│   │   ├── logging.py       # dictConfig, request-id filter, formatters
│   │   ├── security.py      # password hashing, JWT encode/decode
│   │   └── exceptions.py    # custom AppError classes
│   ├── middleware/
│   │   └── request_context.py   # request id + access log
│   ├── db/
│   │   ├── base.py          # DeclarativeBase, naming conventions
│   │   ├── session.py       # engine, sessionmaker, get_db dependency
│   │   └── models/          # SQLAlchemy models
│   │       ├── user.py
│   │       ├── category.py
│   │       └── article.py
│   ├── schemas/             # Pydantic models (request/response)
│   │   ├── common.py        # envelope, pagination, error shapes
│   │   ├── user.py
│   │   └── article.py
│   ├── api/
│   │   ├── deps.py          # DbDep, CurrentUserDep, AdminDep
│   │   └── v1/
│   │       ├── auth.py
│   │       ├── users.py
│   │       └── articles.py
│   ├── repositories/        # DB access, no HTTP knowledge
│   └── services/            # business rules, no HTTP knowledge
├── alembic/
├── tests/
├── compose.yaml             # postgres (+ adminer, + app later)
├── Dockerfile               # multi-stage build of the app
├── .dockerignore
├── .env / .env.example
├── Makefile
├── pyproject.toml
└── plan.md
```

**The layering rule to internalize:** routers know HTTP, services know rules, repositories know SQL. A repository never raises `HTTPException`; a router never writes a `select()`.

---

## Phase 0 — Setup

**Goal:** a running server and the tooling you'll use all the way through.

```bash
uv init
uv add "fastapi[standard]" sqlalchemy alembic pydantic-settings \
       "psycopg[binary]" "pyjwt" "pwdlib[argon2]"
uv add --dev pytest pytest-asyncio httpx ruff ty
```

`psycopg[binary]` is psycopg 3 — the current Postgres driver. The `[binary]` extra ships precompiled wheels, so you don't need Postgres headers on your Mac.

Add the entrypoint to `pyproject.toml` so `fastapi dev` needs no path:

```toml
[tool.fastapi]
entrypoint = "app.main:app"
```

Create `app/main.py` with a single `/health` endpoint, plus `app/core/config.py`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: str
    secret_key: str
    access_token_minutes: int = 15
    refresh_token_days: int = 7
    log_level: str = "INFO"
    debug: bool = False


settings = Settings()
```

Commit a `.env.example` with fake values; put the real `.env` in `.gitignore` from the very first commit.

**Learn:**
- Why `fastapi dev` instead of `uvicorn` directly (reload, nicer output, same app)
- `pydantic-settings` for config — never read `os.environ` scattered through the code
- Config is validated at startup: a missing `SECRET_KEY` should crash the app immediately, not at the first login

**Checkpoint:** `GET /health` returns `{"status": "ok"}` and `/docs` renders.

---

## Phase 1 — Postgres with Docker Compose

**Goal:** a real Postgres on your machine without installing Postgres on your machine.

```yaml
# compose.yaml
services:
  db:
    image: postgres:17-alpine
    container_name: fastapi-admin-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: fastapi_admin
    ports:
      - "5433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d fastapi_admin"]
      interval: 5s
      timeout: 5s
      retries: 10

  adminer:
    image: adminer:5
    restart: unless-stopped
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Then in `.env`:

```
DATABASE_URL=postgresql+psycopg://admin:admin@localhost:5433/fastapi_admin
```

```bash
docker compose up -d
docker compose ps
docker compose logs -f db
docker compose exec db psql -U admin -d fastapi_admin -c "\dt"
```

**Learn:**
- **The two port numbers.** `"5433:5432"` means *host 5433 → container 5432*. From your Mac you connect to `localhost:5433`. From another container on the same compose network you connect to `db:5432` — the service name is the hostname. Getting this backwards is the #1 Docker-Postgres confusion, and it bites again in Phase 13
- **Named volumes = your data.** `pgdata` is where the database actually lives. `docker compose down` stops containers and keeps it; `docker compose down -v` **deletes it**. Learn that difference now, deliberately: run `down -v`, bring it back up, watch the schema be gone
- **Healthchecks.** `pg_isready` is how other services know the DB is actually accepting connections — "container started" is not the same as "database ready"
- `postgres:17-alpine` — pin the major version. `latest` will silently upgrade and break your data directory one day
- Adminer at `http://localhost:8080` gives you a GUI (System: PostgreSQL, Server: `db`, user/pass `admin`) so you can see rows while you learn the ORM
- `.env` holds the dev password; it is never committed, and production values never live in compose

**Checkpoint:** `docker compose up -d`, then `psql` inside the container lists an empty database. Stop with `down`, start again, and the database is still there.

---

## Phase 2 — Makefile

**Goal:** stop retyping long commands, and give the project one obvious entry point.

```make
.DEFAULT_GOAL := help
.PHONY: help install up down reset logs psql dev migrate revision downgrade test lint fmt check

help:  ## Show available commands
	@grep -E '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-12s\033[0m %s\n", $$1, $$2}'

install:  ## Install dependencies
	uv sync

up:  ## Start Postgres (and adminer)
	docker compose up -d

down:  ## Stop containers, keep data
	docker compose down

reset:  ## Stop containers and DELETE the database volume
	docker compose down -v

logs:  ## Tail database logs
	docker compose logs -f db

psql:  ## Open a psql shell inside the db container
	docker compose exec db psql -U admin -d fastapi_admin

dev: up  ## Start the API with reload
	uv run fastapi dev

migrate:  ## Apply all migrations
	uv run alembic upgrade head

revision:  ## Create a migration: make revision m="add users"
	uv run alembic revision --autogenerate -m "$(m)"

downgrade:  ## Roll back one migration
	uv run alembic downgrade -1

test:  ## Run the test suite
	uv run pytest -q

lint:  ## Lint and type-check
	uv run ruff check .
	uv run ty check

fmt:  ## Format code
	uv run ruff format .
	uv run ruff check --fix .

check: lint test  ## Everything CI would run
```

**Learn (the Make gotchas that waste an hour):**
- **Recipe lines must start with a real TAB**, not spaces. Configure your editor for `Makefile` before you write a line
- **Each recipe line runs in its own shell.** `cd app` on one line does not affect the next. Chain with `&&` or use `\` continuations
- `.PHONY` tells Make the target is a command, not a file. Without it, a file named `test` in your repo makes `make test` silently do nothing
- `$$` escapes a `$` so the shell sees it (that's why the `help` awk block has `$$1`)
- Prerequisites: `dev: up` means `make dev` starts Docker first. Use this to make the right thing automatic
- Pass arguments with variables: `make revision m="add users"`

**Checkpoint:** `make help` prints the command list, and `make dev` brings up Postgres and the API in one command from a cold machine.

---

## Phase 3 — SQLAlchemy ORM fundamentals (no FastAPI yet)

**Goal:** understand the ORM in isolation. Write a throwaway script, not endpoints.

Modern SQLAlchemy 2.0 style — typed, no legacy `Column()`:

```python
from datetime import datetime
from sqlalchemy import ForeignKey, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    articles: Mapped[list["Article"]] = relationship(back_populates="author")
```

**Learn, in this order:**
1. `create_engine` + `Session` — what a session actually is (unit of work + identity map)
2. `session.add()` / `flush()` / `commit()` — the difference, and when the INSERT really happens
3. `select()` + `session.scalars()` — the 2.0 query API (`session.query()` is legacy, don't learn it)
4. Filtering: `where()`, `and_`, `or_`, `ilike`, `in_`
5. Relationships: `relationship()` + `back_populates`, one-to-many and many-to-one
6. **Lazy loading and the N+1 problem** — then fix it with `selectinload()` / `joinedload()`
7. `cascade="all, delete-orphan"` — what happens to articles when a user is deleted

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload

stmt = (
    select(Article)
    .options(selectinload(Article.author))
    .where(Article.is_published.is_(True))
    .order_by(Article.created_at.desc())
    .limit(20)
)
articles = session.scalars(stmt).all()
```

**Exercise:** turn on `create_engine(..., echo=True)` and read the SQL for a list query with and without `selectinload`. Count the queries. This is the single most valuable hour in the whole plan. Keep `make psql` open in a second terminal and check the rows the ORM claims to have written.

**Checkpoint:** a script that creates users + articles against the Dockerized Postgres, queries them with a join, and prints them — with no N+1.

---

## Phase 4 — Wiring the ORM into FastAPI

**Goal:** one session per request, closed properly, injected as a dependency.

```python
# app/db/session.py
from collections.abc import Generator
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine(settings.database_url, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# app/api/deps.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session

DbDep = Annotated[Session, Depends(get_db)]
```

**Learn:**
- Dependencies with `yield` — cleanup runs after the response is sent (default `scope="request"`)
- `Annotated` type aliases so `DbDep` is written once and reused everywhere
- Why `expire_on_commit=False` matters when you return ORM objects from an endpoint
- Sync `def` endpoints with sync SQLAlchemy: FastAPI runs them in a threadpool, so they don't block the loop. **Do not** call blocking SQLAlchemy inside an `async def` endpoint
- `pool_pre_ping=True` — matters specifically here: restarting the Postgres container kills every pooled connection, and without pre-ping your next request fails with a stale-connection error. Try it: `docker compose restart db`, then hit an endpoint
- `Depends(get_db, max_concurrency=N)` to keep request concurrency inside your connection pool size

**Checkpoint:** `GET /users` reads real rows from Postgres through the dependency, and still works after `docker compose restart db`.

---

## Phase 5 — Alembic migrations

**Goal:** stop using `Base.metadata.create_all()` like a toy.

```bash
uv run alembic init alembic
make revision m="create users"
make migrate
```

**Learn:**
- Point `target_metadata = Base.metadata` in `env.py`, and import every model so autogenerate sees them
- Read `alembic.ini` vs `env.py`: pull the URL from your `Settings` in `env.py` instead of hardcoding it in the ini, so the Docker and local URLs both work
- Read every generated migration before applying it — autogenerate misses renames and type changes
- `upgrade` / `downgrade` / `alembic history` / `alembic current`
- Naming conventions on `DeclarativeBase.metadata` so constraint names are stable and downgradeable
- Postgres-specific types you'll meet here: `UUID`, `JSONB`, `TIMESTAMP WITH TIME ZONE`, native `ENUM` (and why enums are painful to migrate — prefer a `String` + a check constraint, or a lookup table)

**Checkpoint:** `make reset && make up && make migrate` rebuilds the entire schema from nothing. This is the real test that your migrations are honest.

---

## Phase 6 — Password hashing and the user model

**Goal:** correct credential storage before touching tokens.

```python
# app/core/security.py
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()


def hash_password(plain: str) -> str:
    return password_hash.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return password_hash.verify(plain, hashed)
```

**Learn:**
- Why Argon2/bcrypt and never SHA-256 for passwords (slow by design, salted)
- Hashing is CPU-blocking — keep it in `def` endpoints, not `async def`
- Login must return the *same* error for "unknown email" and "wrong password" (no user enumeration)
- The response schema never includes `hashed_password` — enforce it with the schema, not discipline

**Checkpoint:** `POST /auth/register` stores an Argon2 hash; the response has no password field at all. Confirm the hash with `make psql`.

---

## Phase 7 — JWT authentication

**Goal:** the core auth loop, end to end.

```python
# app/core/security.py
from datetime import datetime, timedelta, timezone
import jwt

def create_access_token(subject: str, expires_in: timedelta) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": subject,
        "iat": now,
        "exp": now + expires_in,
        "type": "access",
    }
    return jwt.encode(payload, settings.secret_key, algorithm="HS256")
```

```python
# app/api/deps.py
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")


def get_current_user(db: DbDep, token: Annotated[str, Depends(oauth2_scheme)]) -> User:
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise TokenExpiredError()
    except jwt.InvalidTokenError:
        raise InvalidTokenError()
    ...


CurrentUserDep = Annotated[User, Depends(get_current_user)]
```

**Learn, in this order:**
1. JWT anatomy — header / payload / signature; decode one at jwt.io and see that the payload is **readable by anyone**. Never put secrets in it
2. Registered claims: `sub`, `exp`, `iat`, plus a custom `type` to separate access from refresh
3. `OAuth2PasswordBearer` — what it gives you (the Authorize button in `/docs`, header extraction)
4. Short-lived access token (15 min) + long-lived refresh token (7 days), with `POST /auth/refresh`
5. Always verify `exp` *and* that the token type matches the endpoint — a refresh token must not work as an access token
6. HS256 vs RS256 — symmetric is fine for one service; asymmetric when other services verify your tokens

**Learn (the honest limits):** a JWT can't be un-issued. To support logout/ban you need either a short expiry you tolerate, a `jti` denylist in Redis/DB, or a `token_version` column on the user that's checked on decode. Pick one and implement it — this is what separates a tutorial from a real system.

**Checkpoint:** login → call a protected endpoint → wait for expiry → get 401 → refresh → works again.

---

## Phase 8 — Authorization (roles)

**Goal:** an admin panel needs more than "logged in".

```python
def require_roles(*roles: str):
    def dependency(user: CurrentUserDep) -> User:
        if user.role not in roles:
            raise ForbiddenError("Insufficient permissions")
        return user
    return dependency


AdminDep = Annotated[User, Depends(require_roles("admin"))]
```

**Learn:**
- A dependency factory — a function that returns a dependency, so you can parametrize it
- Router-level dependencies for whole sections:
  ```python
  router = APIRouter(prefix="/admin", tags=["admin"], dependencies=[Depends(require_roles("admin"))])
  ```
- 401 vs 403: *who are you* vs *you may not do this*
- Object-level checks (a staff user may edit only their own article) belong in the service layer, not the dependency

**Checkpoint:** a staff token gets 403 on `DELETE /users/{id}`; an admin token gets 204.

---

## Phase 9 — Professional response design

**Goal:** every endpoint answers in a shape a frontend can rely on.

### 9a. Schemas per direction

Never reuse one model for input and output:

```python
class UserBase(BaseModel):
    email: EmailStr
    full_name: str


class UserCreate(UserBase):
    password: str = Field(min_length=8)


class UserUpdate(BaseModel):
    full_name: str | None = None
    is_active: bool | None = None


class UserPublic(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    role: str
    created_at: datetime
```

**Learn:** `from_attributes=True` to build a schema straight from an ORM object; prefer declaring the **return type** on the path operation (`-> UserPublic`) so FastAPI validates, filters and documents it; use `response_model=` only when the returned type differs from the public schema.

### 9b. A consistent envelope

```python
class Meta(BaseModel):
    page: int
    per_page: int
    total: int
    total_pages: int


class Response[T](BaseModel):
    success: bool = True
    message: str = "OK"
    data: T


class PaginatedResponse[T](BaseModel):
    success: bool = True
    message: str = "OK"
    data: list[T]
    meta: Meta
```

```python
@router.get("/")
def list_users(db: DbDep, params: PageParamsDep) -> PaginatedResponse[UserPublic]:
    ...
```

**Decide deliberately:** an envelope is a trade-off. It gives the frontend one predictable shape; it costs you one extra nesting level and slightly noisier OpenAPI. Either choose it everywhere or choose bare bodies + HTTP status everywhere — inconsistency is the only truly wrong answer. Write your decision down in the README.

### 9c. Errors, centrally

```python
# app/core/exceptions.py
class AppError(Exception):
    status_code = 400
    code = "bad_request"

    def __init__(self, message: str, details: dict | None = None): ...


class NotFoundError(AppError):
    status_code = 404
    code = "not_found"
```

```python
# app/main.py
@app.exception_handler(AppError)
def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "success": False,
            "message": exc.message,
            "error": {"code": exc.code, "details": exc.details},
        },
    )
```

**Learn:**
- Services raise `AppError` subclasses; only handlers translate them to HTTP. The service layer stays reusable from a CLI or a worker
- Override `RequestValidationError` so 422 bodies match your envelope too — this is the one most tutorials skip
- A catch-all `Exception` handler that logs the traceback and returns a generic 500 — never the stack trace. In Phase 10 you'll add the request id to that response so a user can quote it to you
- Correct statuses: 201 + `Location` on create, 204 with no body on delete, 409 on unique-constraint conflict
- Catch `sqlalchemy.exc.IntegrityError` and translate it to your 409 — don't let a raw Postgres error reach the client

### 9d. Pagination, filtering, sorting

```python
@dataclass
class PageParams:
    page: int = 1
    per_page: int = 20
    q: str | None = None
    sort: str = "-created_at"


def get_page_params(
    page: Annotated[int, Query(ge=1)] = 1,
    per_page: Annotated[int, Query(ge=1, le=100)] = 20,
    q: Annotated[str | None, Query(max_length=100)] = None,
    sort: Annotated[str, Query(pattern=r"^-?\w+$")] = "-created_at",
) -> PageParams:
    return PageParams(page=page, per_page=per_page, q=q, sort=sort)


PageParamsDep = Annotated[PageParams, Depends(get_page_params)]
```

**Learn:**
- Cap `per_page` — an uncapped limit is a denial-of-service hole
- Whitelist sortable columns by name; never interpolate a user string into `order_by`
- `func.count()` over the filtered query for `total` — and know that offset pagination gets slow on big tables (read about keyset/cursor pagination, implement it if you have time)

**Checkpoint:** `GET /articles?page=2&per_page=10&q=fast&sort=-created_at` returns the envelope with correct `meta`, and a bad `per_page=1000` returns a 422 in your error shape.

---

## Phase 10 — Logging

**Goal:** when something breaks in production, you can find out what happened. `print()` cannot do that.

### 10a. Configure logging once, at startup

```python
# app/core/logging.py
import logging
from logging.config import dictConfig
from contextvars import ContextVar

request_id_ctx: ContextVar[str] = ContextVar("request_id", default="-")


class RequestIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_ctx.get()
        return True


def setup_logging(level: str = "INFO", json_logs: bool = False) -> None:
    dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "filters": {"request_id": {"()": RequestIdFilter}},
        "formatters": {
            "console": {
                "format": "%(asctime)s %(levelname)-8s [%(request_id)s] %(name)s: %(message)s",
            },
            "json": {"()": "app.core.logging.JsonFormatter"},
        },
        "handlers": {
            "default": {
                "class": "logging.StreamHandler",
                "stream": "ext://sys.stdout",
                "formatter": "json" if json_logs else "console",
                "filters": ["request_id"],
            },
        },
        "loggers": {
            "uvicorn.error": {"level": level},
            "uvicorn.access": {"handlers": ["default"], "level": "WARNING", "propagate": False},
            "sqlalchemy.engine": {"level": "WARNING"},
        },
        "root": {"handlers": ["default"], "level": level},
    })
```

Call `setup_logging(...)` in your lifespan/startup, before anything else logs. Then everywhere else:

```python
import logging

logger = logging.getLogger(__name__)
```

### 10b. Request ID middleware

```python
# app/middleware/request_context.py
import logging, time, uuid
from fastapi import Request

logger = logging.getLogger("app.access")


@app.middleware("http")
async def request_context(request: Request, call_next):
    rid = request.headers.get("X-Request-ID") or uuid.uuid4().hex
    token = request_id_ctx.set(rid)
    start = time.perf_counter()
    try:
        response = await call_next(request)
    except Exception:
        logger.exception("%s %s failed", request.method, request.url.path)
        raise
    finally:
        request_id_ctx.reset(token)

    elapsed_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Request-ID"] = rid
    logger.info(
        "%s %s %s %.1fms",
        request.method, request.url.path, response.status_code, elapsed_ms,
    )
    return response
```

Then put `rid` in the body of every error response, so a user reporting a bug gives you the exact log line.

**Learn:**
- **`logging.getLogger(__name__)` per module**, never a global logger object — the name is what lets you raise or silence one subsystem
- **`contextvars`** are the mechanism that makes a request id work under concurrency. A module-level global would leak between requests; a `ContextVar` is per-task. Always `reset(token)` in a `finally`
- `logger.exception(...)` inside an `except` block — it attaches the traceback automatically. `logger.error(str(e))` throws away the stack
- **Lazy formatting:** `logger.info("user %s", user_id)`, not an f-string. The formatting is skipped entirely when the level is off
- Levels that mean something: DEBUG = tracing you'd disable in prod; INFO = business events (user registered, article published); WARNING = handled but suspicious (repeated failed login); ERROR = the request failed and someone should look
- **Never log secrets.** No passwords, no tokens, no `Authorization` header, no full request bodies on auth routes. Write a redaction helper and use it
- `uvicorn.access` logs duplicate your middleware — pick one. The config above silences it and keeps yours (which has the request id)
- `sqlalchemy.engine` at `INFO` prints every SQL statement. Useful while learning Phase 3, far too loud afterwards
- **12-factor:** log to stdout, never to a file. In Phase 13, Docker collects stdout; `docker compose logs -f api` is your log viewer. Writing to a file inside a container is how logs get lost
- JSON in production so a log platform can index the fields; human-readable in dev. `structlog` is the popular library if you want this without hand-rolling — learn the stdlib version first so you understand what it's wrapping

**Checkpoint:** one request produces one access log line with a request id; the same id appears in the 500 response body and in the traceback log; `curl -H "X-Request-ID: abc123"` is echoed back in the response header.

---

## Phase 11 — Structure: repositories and services

**Goal:** get logic out of the routers.

```python
# repositories/user.py — knows SQL, not HTTP
def get_by_email(db: Session, email: str) -> User | None:
    return db.scalar(select(User).where(User.email == email))


# services/user.py — knows rules, not HTTP
def register_user(db: Session, data: UserCreate) -> User:
    if user_repo.get_by_email(db, data.email):
        raise ConflictError("Email already registered")
    user = User(email=data.email, hashed_password=hash_password(data.password))
    db.add(user)
    db.commit()
    db.refresh(user)
    logger.info("user registered id=%s", user.id)
    return user


# api/v1/users.py — knows HTTP only
@router.post("/", status_code=201)
def create_user(db: DbDep, data: UserCreate, _: AdminDep) -> Response[UserPublic]:
    user = user_service.register_user(db, data)
    return Response(message="User created", data=UserPublic.model_validate(user))
```

**Learn:**
- Who owns `commit()` — pick the service layer and be consistent (the router should never commit)
- One HTTP operation per function — don't branch on method inside one handler
- Put `prefix`, `tags` and shared `dependencies` on the `APIRouter`, not on `include_router()`
- Version from day one: `/api/v1`

**Checkpoint:** no router function contains a `select()` or an `if` on business rules.

---

## Phase 12 — Testing

**Goal:** prove auth and responses actually work.

```python
@pytest.fixture
def client(db_session):
    app.dependency_overrides[get_db] = lambda: db_session
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

**Learn:**
- `app.dependency_overrides` — the reason dependency injection was worth it
- A transactional fixture: begin a transaction per test, roll back after, so tests don't leak state
- Test against **Postgres, not SQLite** — they disagree on enums, JSON, case sensitivity and constraint behavior, and a green SQLite suite can hide a production failure. Add a second database to compose (`db_test`, different host port) or create `fastapi_admin_test` on the same server
- Minimum cases to cover: login success, login wrong password, protected endpoint without token (401), with expired token (401), with staff token on an admin route (403), pagination `meta` math, 422 body shape

Add to the Makefile:

```make
test: up  ## Run the test suite against the Dockerized DB
	uv run pytest -q
```

**Checkpoint:** `make test` is green from a cold machine, and you can break auth on purpose and watch a test catch it.

---

## Phase 13 — Containerize the app itself

**Goal:** the whole stack starts with one command, on any machine.

```dockerfile
# Dockerfile
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim AS builder
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

COPY . .
RUN uv sync --frozen --no-dev

FROM python:3.13-slim-bookworm
WORKDIR /app

RUN useradd --create-home --uid 1000 app
COPY --from=builder --chown=app:app /app /app
ENV PATH="/app/.venv/bin:$PATH"
USER app

EXPOSE 8000
CMD ["fastapi", "run", "app/main.py", "--host", "0.0.0.0", "--port", "8000"]
```

Add to `compose.yaml`:

```yaml
  api:
    build: .
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql+psycopg://admin:admin@db:5432/fastapi_admin
      SECRET_KEY: dev-only-not-a-real-secret
      LOG_LEVEL: INFO
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
```

And a `.dockerignore` (at minimum): `.venv`, `__pycache__`, `.git`, `.env`, `tests`, `*.md`.

**Learn:**
- **`db`, not `localhost`.** Inside the compose network the hostname is the service name and the port is the *container* port (5432), not the published one (5433). `localhost` inside a container means that container itself — this is the single most common "connection refused" in Docker
- **Layer caching:** copy `pyproject.toml` + `uv.lock` and install *before* copying your source. Otherwise every code edit reinstalls every dependency. Build twice and watch the difference
- **Multi-stage:** the final image has no build toolchain, no uv, no dev dependencies — smaller and less to attack
- **Run as a non-root user.** Default Docker is root; that's a real finding in any security review
- `fastapi run` (not `dev`) in the image — no reload, production defaults
- `--host 0.0.0.0` is mandatory in a container. Binding to `127.0.0.1` means nothing outside the container can reach it
- `depends_on: condition: service_healthy` uses the healthcheck from Phase 1 — this is what stops the API racing Postgres on startup
- Migrations don't belong in `CMD`. Run `docker compose run --rm api alembic upgrade head` as a separate step, or an entrypoint script — so that scaling to 2 containers doesn't run migrations twice
- `EXPOSE` documents; `ports:` actually publishes. They are not the same thing
- For a dev-mode container, mount the source (`volumes: - .:/app`) and run `fastapi dev --host 0.0.0.0` — understand why that mount is wrong for production

Makefile additions:

```make
build:  ## Build the app image
	docker compose build

sh:  ## Shell into the api container
	docker compose exec api bash

docker-migrate:  ## Run migrations inside the container
	docker compose run --rm api alembic upgrade head
```

**Checkpoint:** `make reset && docker compose up -d --build && make docker-migrate` gives a working API at `http://localhost:8000/docs` on a machine with nothing installed but Docker.

---

## Phase 14 — Production polish

- CORS middleware configured for the real admin frontend origin (not `*` with credentials)
- `max_body_size` on the app or on upload routes
- Rate limiting on `/auth/login` (even a simple in-memory limiter teaches the shape)
- `SECRET_KEY` from env, different per environment, rotation plan written down. Dev secrets in compose are fine; production secrets never are
- Health vs readiness: `/health` says the process is alive, `/ready` actually runs `SELECT 1` against the DB. Orchestrators need both
- Resource limits and `restart: unless-stopped` on compose services; a `postgres` backup command in the Makefile (`pg_dump` via `docker compose exec`)
- Decide sync vs async stack, then be consistent: `async def` endpoints require async SQLAlchemy (`create_async_engine`, `AsyncSession`, `psycopg` async or `asyncpg`). Mixing sync SQLAlchemy into `async def` is the most common performance bug in FastAPI projects
- Connection pool sizing: `pool_size` + `max_overflow` must stay under Postgres `max_connections` divided by the number of app containers

---

## Reference order (read these, in this order)

1. FastAPI — Tutorial: Dependencies, Security (OAuth2 + JWT), Response Model, Handling Errors
2. SQLAlchemy 2.0 — ORM Quick Start, then "Writing SELECT statements", then "Relationship Loading Techniques"
3. Alembic — Tutorial + Auto Generating Migrations
4. Python docs — Logging HOWTO and the Logging Cookbook (short, and most people never read them)
5. Docker — Compose file reference + "Best practices for writing Dockerfiles"
6. RFC 7519 (JWT) — skim the registered claims section only
7. OWASP Authentication Cheat Sheet — for the parts tutorials leave out

---

## Notes on choices in this plan

- **Postgres in Docker from day one.** You don't have Postgres installed and you don't need to — a container gives you the real database, pinned to a version, deletable with one command. Learning against SQLite and switching later means re-learning the parts where they differ, exactly when you can least afford it
- **`compose.yaml`, no `version:` key.** That's the current Compose spec. `docker-compose.yml` and a `version: "3.8"` line are legacy; the latter now prints a warning
- **SQLAlchemy over SQLModel:** you asked for SQLAlchemy ORM specifically. SQLModel (same author as FastAPI) wraps SQLAlchemy and merges the ORM model with the Pydantic schema — less boilerplate, less control. Learn SQLAlchemy first; SQLModel will then be obvious, and you'll know what it's hiding
- **psycopg 3 over psycopg2:** psycopg2 is maintenance-only. The URL prefix differs (`postgresql+psycopg://` vs `postgresql+psycopg2://`) — copying an old tutorial's URL with the new driver installed is a confusing first error
- **PyJWT over python-jose:** python-jose is widely copied in old tutorials but poorly maintained. PyJWT is the current default
- **pwdlib over passlib:** passlib is unmaintained and breaks against newer bcrypt releases; pwdlib is the recommended replacement
- **stdlib `logging` before `structlog`:** structlog is better, but it's a wrapper. Understand handlers, filters and formatters first or you'll be debugging a library you don't understand

---

## Progress

- [ ] Phase 0 — Setup
- [ ] Phase 1 — Postgres with Docker Compose
- [ ] Phase 2 — Makefile
- [ ] Phase 3 — SQLAlchemy ORM fundamentals
- [ ] Phase 4 — ORM + FastAPI dependency
- [ ] Phase 5 — Alembic migrations
- [ ] Phase 6 — Password hashing
- [ ] Phase 7 — JWT authentication
- [ ] Phase 8 — Role-based authorization
- [ ] Phase 9 — Professional responses
- [ ] Phase 10 — Logging
- [ ] Phase 11 — Repositories and services
- [ ] Phase 12 — Testing
- [ ] Phase 13 — Containerize the app
- [ ] Phase 14 — Production polish
