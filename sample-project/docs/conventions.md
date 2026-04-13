# Coding Conventions — test-basic

## Python

- Python 3.10+
- Virtualenv at project root: `.venv/` (created with `uv venv`)
- Install packages: `uv pip install -r backend/requirements.txt`
- Run tests from project root: `pytest backend/tests/ -v`
- Use `httpx.AsyncClient` with `ASGITransport(app=app)` for in-process tests
- `conftest.py` must override the database URL to SQLite in-memory for tests

## FastAPI

- Routers in `backend/app/routers/` — one file per resource
- Models in `backend/app/models/` — one file per model
- Return JSON only from API routes
- CORS: allow `http://localhost:5173` (Vite dev server)
- `get_db` dependency in `app/database.py` or `app/main.py`

## SQLAlchemy

- Synchronous engine: `create_engine("sqlite:///./backend/data.db", connect_args={"check_same_thread": False})`
- `Base = declarative_base()` in `database.py`
- `Base.metadata.create_all(bind=engine)` called at app startup

## Item Model

```python
class Item(Base):
    __tablename__ = "items"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False)
    done = Column(Boolean, default=False)
```

## API Conventions

- `GET /api/items` → list all items as `[{id, name, done}]`
- `POST /api/items` body `{name: str}` → 201 + created item
- `DELETE /api/items/{id}` → 204 no content
- `GET /api/health` → `{"status": "ok"}`

## Frontend

- Vue 3 Composition API (`<script setup>`)
- Vite as build tool; proxy `/api` to `http://localhost:8000` in `vite.config.js`
- Axios instance in `src/api/client.js` with `baseURL: '/api'`
- Components in `src/components/`
- Run build check: `npm run build --prefix frontend`
- No TypeScript — plain JS

## Git

- Commit after each feature: `git add -A && git commit -m "feat: F0X description"`
- Never commit `.env` or `*.db` files

## All commands run from project root (not from backend/ or frontend/)

- Tests: `pytest backend/tests/test_foo.py -v`
- Frontend build: `npm run build --prefix frontend`
- Install deps: `uv pip install -r backend/requirements.txt`
