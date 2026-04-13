# Test-basic — Project Knowledge Base Index

## Core Documents

| Document | Contents |
|---|---|
| [development-plan.md](../development-plan.md) | Project goal, tech stack, architecture, feature sequence |
| [docs/architecture.md](architecture.md) | Directory structure, stack table, conftest pattern |
| [docs/conventions.md](conventions.md) | Backend + frontend conventions, API contract, command rules |

## Feature Plans (created by agent during planning sessions)

Located in `docs/feature-plans/`:
- `F01.md` — Backend Scaffold
- `F02.md` — Frontend Scaffold
- `F03.md` — Item CRUD
- `F04.md` — Item List UI

## Quick Reference

**API endpoints:**
- `GET /api/health` → `{"status": "ok"}`
- `GET /api/items` → list of items
- `POST /api/items` body `{name}` → 201 + item
- `DELETE /api/items/{id}` → 204

**Test commands (run from project root):**
- `pytest backend/tests/test_health.py -v`
- `pytest backend/tests/test_items.py -v`
- `pytest backend/tests/ -v`
- `npm run build --prefix frontend`

**Stack:** FastAPI + SQLAlchemy + SQLite + pytest + httpx | Vue 3 + Vite + Axios
