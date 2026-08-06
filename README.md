# Core API

A FastAPI backend for a domain-agnostic message-testing platform: operators
define a **domain** (market/audience context), spin up a **study** with
candidate **messages** and an **avatar** panel (AI personas), run the study
through an SSR pipeline, and get back a ranked recommendation plus a
qualitative report.

This repo currently holds a **walking skeleton** — one real, DB-backed
endpoint (`GET /api/v1/domains`) proving the full stack round-trips, not a
complete implementation.

## Stack

- **API:** FastAPI + Pydantic v2 (async)
- **Database:** PostgreSQL 16 + pgvector, via SQLAlchemy 2.0 (asyncpg)
- **Cache / rate limiting:** Redis
- **Local dev services:** Docker Compose

## Requirements

- Python 3.11+
- Docker Desktop (for Postgres + Redis)

## Setup

```bash
# 1. Configure environment
cp .env.example .env

# 2. Start Postgres (pgvector) + Redis
docker compose up -d

# 3. Create a virtualenv and install dependencies
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 4. Create the schema and seed dev data
python scripts/apply_schema.py
python scripts/seed_dev_data.py

# 5. Run the API
uvicorn app.main:app --reload
```

## Verify it's working

```bash
curl http://localhost:8000/health
curl http://localhost:8000/api/v1/domains
```

The second call should return the 4 domains seeded by
`scripts/seed_dev_data.py`. Interactive API docs are at
`http://localhost:8000/docs`.

## Project layout

```
app/
  main.py            FastAPI entrypoint, /health
  core/
    config.py         Settings (env-driven), loaded from .env
  db/
    base.py            Shared SQLAlchemy declarative base
    session.py          Async engine + per-request session dependency
    models/              One ORM model per table (users, domains so far)
  api/
    deps.py             Shared dependencies: DB session, current-user stub
    pagination.py        Cursor-pagination helpers (encode/decode)
    v1/
      router.py           Aggregates all /api/v1 routers
      domains.py           GET /domains
  schemas/              Pydantic request/response models for every API schema
scripts/
  apply_schema.py       Applies setup.sql directly (see Schema management)
  seed_dev_data.py       Seeds the fixed dev user + 4 sample domains
  setup.sql              The database schema (source of truth)
docker-compose.yml      Postgres (pgvector) + Redis for local dev
```

## Notes on current design decisions

**Auth is stubbed.** `app/api/deps.py`'s `get_current_user` always returns
the single dev user seeded by `scripts/seed_dev_data.py`
(`APP_DEV_USER_ID` in `.env`), not a real authenticated identity. Real
Entra ID JWT validation replaces this later — every route already depends
on `get_current_user`, so only that function's body will need to change.

**Schema management has no migration tool yet.** `scripts/apply_schema.py`
applies `setup.sql` directly via `asyncpg`; there's no migration history and
no `downgrade`. Re-running it against an already-schema'd database will fail
on `CREATE TABLE`/`CREATE SCHEMA` — for now, drop and recreate the database
to reset. Alembic (or similar) is worth reintroducing once there's an actual
second schema version to manage.

**`/api/v1` prefix.** Matches the API spec's documented base path, and
leaves room for a `v2` later without breaking existing clients.

**Only `User` and `Domain` are modeled as SQLAlchemy ORM classes.** The rest
of `setup.sql`'s tables (studies, messages, avatars, runs, ...) get a model
each as their verticals are built out.
