# Architecture — test-basic

## Overview

Single-repo project. All code lives under `projects/test-basic/`.

## Directory Structure

```
pipetest/
├── .venv/                  # Python virtualenv (uv venv)
├── backend/
│   ├── app/
│   │   ├── main.py         # FastAPI entry point + CORS
│   │   ├── database.py     # SQLAlchemy setup
│   │   ├── models/
│   │   │   └── item.py     # Item(id, name, done)
│   │   └── routers/
│   │       └── items.py    # GET/POST/DELETE /api/items
│   ├── tests/
│   │   ├── conftest.py     # async client fixture, in-memory DB
│   │   ├── test_health.py
│   │   └── test_items.py
│   └── requirements.txt
├── frontend/               # Vue 3 + Vite
│   ├── src/
│   │   ├── main.js
│   │   ├── App.vue
│   │   ├── api/client.js   # axios baseURL '/api'
│   │   └── components/
│   │       ├── ItemList.vue
│   │       └── AddItem.vue
│   ├── vite.config.js      # proxy /api → http://localhost:8000
│   └── package.json
└── development-plan.md
```

## Stack

| Layer    | Technology                        |
|----------|-----------------------------------|
| Backend  | FastAPI + Uvicorn                 |
| ORM      | SQLAlchemy (sync)                 |
| Database | SQLite (`backend/data.db`)        |
| Tests    | pytest + httpx (ASGITransport)    |
| Frontend | Vue 3 + Vite + Axios              |

## Key Decisions

- Vite proxies `/api/*` to `http://localhost:8000` — no CORS issues in dev
- Backend allows CORS from `http://localhost:5173`
- Frontend is **not** served by FastAPI — separate dev servers
- `.venv/` at project root; harness injects it into PATH automatically
- Test database is in-memory SQLite — no file created during tests

## conftest.py Pattern

```python
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.database import Base, get_db

@pytest.fixture
async def client():
    engine = create_engine("sqlite:///:memory:", connect_args={"check_same_thread": False})
    TestingSessionLocal = sessionmaker(bind=engine)
    Base.metadata.create_all(bind=engine)

    def override_get_db():
        db = TestingSessionLocal()
        try:
            yield db
        finally:
            db.close()

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac

    Base.metadata.drop_all(bind=engine)
    app.dependency_overrides.clear()
```
