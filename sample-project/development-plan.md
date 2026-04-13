# Pipetest — Harness Pipeline Validation Project

A minimal project used to validate the complete agent-harness development loop end-to-end:
init → implement → verify → mark complete → next feature.

## Goal

Prove that a fresh project can be scaffolded, implemented, and verified by the agent
without human intervention, using the full harness pipeline.

## Tech Stack

- **Backend**: FastAPI + SQLite (via SQLAlchemy) + pytest + httpx
- **Frontend**: Vue 3 + Vite + Axios (separate dev server, proxies /api to backend)
- **Database**: SQLite file at `backend/data.db`
- **Python env**: `.venv/` at project root, managed with `uv`
- **Node**: npm, managed globally

## Architecture

```
pipetest/
├── .venv/                        # Python virtualenv (uv venv)
├── backend/
│   ├── app/
│   │   ├── main.py               # FastAPI app + CORS
│   │   ├── database.py           # SQLAlchemy engine + Base
│   │   ├── models/
│   │   │   └── item.py           # Item model
│   │   └── routers/
│   │       └── items.py          # CRUD endpoints
│   ├── tests/
│   │   ├── conftest.py           # async test client fixture
│   │   ├── test_health.py
│   │   └── test_items.py
│   └── requirements.txt
├── frontend/                     # Vite + Vue 3 project
│   ├── src/
│   │   ├── main.js
│   │   ├── App.vue
│   │   ├── api/client.js         # axios instance
│   │   └── components/
│   │       ├── ItemList.vue
│   │       └── AddItem.vue
│   ├── vite.config.js            # /api proxy → localhost:8000
│   └── package.json
└── development-plan.md
```

## Features (ordered)

### Phase 1
- **F01** — Backend Scaffold: FastAPI + health endpoint + SQLite setup + pytest
- **F02** — Frontend Scaffold: Vue 3 + Vite + axios + proxy config

### Phase 2
- **F03** — Item CRUD: backend Item model + GET/POST/DELETE endpoints + tests

### Phase 3
- **F04** — Item List UI: Vue components (ItemList + AddItem) wired to backend API

## Key Decisions

- Frontend and backend are separate processes in development (Vite dev server + uvicorn)
- Vite proxies `/api/*` to `http://localhost:8000` — no CORS issues in dev
- Backend CORS allows `http://localhost:5173` for development
- SQLite keeps the project dependency-free — no DB server needed
- `.venv/` at project root; all test commands use bare `pytest` (PATH injected by harness)
- Frontend build (`npm run build --prefix frontend`) is the frontend "test"
- Tests use in-memory SQLite and httpx ASGITransport — no running server needed
