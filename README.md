# Digital Twin API

A FastAPI backend for a domain-agnostic message-testing platform: operators
define a **domain** (market/audience context), spin up a **study** with
candidate **messages** and an **avatar** panel (AI personas — digital twins
of the target audience), run the study through an SSR pipeline, and get
back a ranked recommendation plus a qualitative report.


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


## Contributing

**Branch naming.** GitHub doesn't natively enforce branch-name prefixes, so
this is convention rather than a hard gate — branch off `main` using one of:

- `feature/short-description` — new features
- `fix/short-description` — bug fixes
- `chore/short-description` — maintenance, dependencies, refactoring

**Pull requests** auto-populate from
[`.github/pull_request_template.md`](.github/pull_request_template.md).

**CI** ([`.github/workflows/ci.yml`](.github/workflows/ci.yml)) runs `ruff
check` on every push and pull request to `main`. Run the same check locally
before pushing:

```bash
pip install -r requirements-dev.txt
ruff check .
```

**Dependency updates** ([`.github/dependabot.yml`](.github/dependabot.yml))
opens weekly PRs to bump pip dependencies and the GitHub Actions used in
`ci.yml`. [`CODEOWNERS`](.github/CODEOWNERS) assigns `@shtewari19` as the
default reviewer for everything — only takes effect on PRs if branch
protection has "Require review from Code Owners" enabled.