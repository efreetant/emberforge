# Global Gotcha Library
#
# This file is a READ-ONLY cross-project reference. The skill and its agents
# read from it but NEVER write to it. Project-specific lessons are recorded
# in {PROJECT_DIR}/docs/project-lessons.md via the `append_lesson` tool.
#
# Format: - [tag1, tag2, ...] description of gotcha and fix
# Tags should match keywords in feature descriptions, expected_files, and tech stack.
# Severity prefix: leading ! = always inject for matching features (e.g., - ![fastapi, routing])

## FastAPI

- ![fastapi, routing] Route ordering: `/path/literal` MUST be registered before `/{param}` in the same router — FastAPI matches in declaration order. Fix: put `/profile/me`, `/search`, etc. before `/{id}`.
- [fastapi, sqlalchemy, depends] `Depends(get_db)` must be overridden in TestClient via `app.dependency_overrides[get_db] = override_get_db`; otherwise tests hit production DB.
- [fastapi, cors] CORS middleware must be added before any router is included; add it right after `app = FastAPI()`.
- [fastapi, enum] SQLAlchemy `Enum("val1", "val2", name="type_name")` on SQLite does not enforce the constraint — validation must happen in Pydantic schema or router logic.
- [fastapi, startup] Never run `uvicorn` or `npm run dev` in `execute_command` — they block forever. Use `python -c "import app.main; print('ok')"` for import smoke tests instead.
- [fastapi, status] Creation endpoints should use `status_code=201`; FastAPI defaults to 200 which breaks some clients.

## SQLAlchemy / Alembic

- ![alembic, migration] `alembic.ini` `sqlalchemy.url` must be `%(DATABASE_URL)s`; `alembic/env.py` must read it via `config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])`.
- [alembic, sqlite] SQLite does not support `ALTER COLUMN` — Alembic autogenerate may produce invalid migrations for column type changes. Always review generated migration files before applying.
- [alembic, autogenerate] `alembic/env.py` must import ALL model modules (or `app.models`) before `target_metadata = Base.metadata` — otherwise autogenerate produces empty migrations.
- [alembic, init] After `alembic init alembic`, the generated `env.py` uses a hardcoded URL — replace with env var reading immediately.
- [sqlalchemy, sqlite, test] In-memory SQLite tests need `connect_args={"check_same_thread": False}` and `StaticPool` from sqlalchemy.pool to share the same connection across threads.
- [sqlalchemy, json] Storing JSON arrays in `Column(String)`: always `json.dumps()` on write and `json.loads()` on read. Do NOT use `Column(JSON)` on SQLite — it has no native JSON type.
- [sqlalchemy, relationship] Lazy loading raises `DetachedInstanceError` outside a session. Use `joinedload()` or set `lazy="joined"` for relationships accessed after session close.
- [sqlalchemy, unique] `UniqueConstraint` on multiple columns requires `__table_args__ = (UniqueConstraint("col1", "col2"),)` — not a column-level `unique=True`.

## Vuetify Brand Theme Override

- ![vuetify, brand, theme] When `docs/brand-guideline.json` is present and contains a `vuetify_theme` section, Vuetify MUST be initialised with that theme in `main.js`. Vuetify's default primary is Material blue `#1976D2` — if the brand overrides it to `#000000`, failing to apply the theme makes every button, chip, and focused element the wrong color.
- ![vuetify, brand, elevation] Brand-zero-elevation projects must set `elevation: 0` in Vuetify's `defaults` option (in `createVuetify`), NOT on individual components. `defaults: { VCard: { elevation: 0 }, VDialog: { elevation: 0 }, VNavigationDrawer: { elevation: 0 }, VAppBar: { elevation: 0 } }` — inline `:elevation="0"` on every usage is fragile and will be missed.
- ![vuetify, brand, v-app] `<v-app>` must carry `theme="cohereLight"` (or whatever `defaultTheme` is in the brand). Without this attribute, the theme is registered but never activated — the app renders with Vuetify defaults.
- [vuetify, brand, fonts] Brand typography fonts (e.g., Space Grotesk, Inter, JetBrains Mono) are web fonts — add them via `<link>` in `index.html`, NOT as npm packages. The Google Fonts CDN URL should be in `index.html`'s `<head>`.
- [vuetify, brand, btn-defaults] When the brand defaults `VBtn` to `variant: 'text'`, CTA buttons must explicitly override: `v-btn variant="flat" color="#000000"`. Forgetting the explicit variant on CTAs leaves them transparent/ghost when they should be solid dark.
- [vuetify, brand, shadow-leak] Even with `elevation: 0` in defaults, some Vuetify components (VSheet, VList) still add `box-shadow` via CSS variables. If a shadow appears unexpectedly, override with `style="box-shadow: none"` or a scoped CSS rule `box-shadow: none !important`.

## Vue 3 / Vuetify 3

- ![vuetify, datepicker] `v-date-picker` in Vuetify 3 has completely different props from v2. Use `v-model` for the selected date. Props like `no-title`, `show-current`, `landscape` from v2 do not exist in v3.
- ![vuetify, install] Vuetify 3 requires explicit style import: `import 'vuetify/styles'` AND `import '@mdi/font/css/materialdesignicons.css'` in `main.js` — missing either causes blank icons or unstyled components.
- [vuetify, components] Vuetify 3 does NOT auto-import components by default. Either use `components: { ...components }` in `createVuetify()` or install the Vuetify Vite plugin for auto-importing.
- [vue3, pinia] `useStore()` composables must be called inside `setup()` or a component lifecycle hook — not at module level, or Pinia throws "no active Pinia" error.
- [vue3, router, guard] `useRouter()` and `useRoute()` cannot be called outside `<script setup>` or `setup()`. For navigation guards use the callback form `router.beforeEach((to, from, next) => {...})`.
- [vue3, script-setup] `defineProps`, `defineEmits`, `defineExpose` are macros — do NOT import them; they are available automatically inside `<script setup>`.

## Vite / npm

- ![npm, vite, scaffold] `npm create vite@latest` may show an interactive prompt. Add `--yes` or use the explicit form: `npm create vite@latest frontend -- --template vue --yes` to avoid blocking.
- [npm, windows] On Windows, npm script `&&` chaining can fail in some shells. Use separate commands or `cross-env` package if chaining is required.
- [vite, proxy] Vite proxy config: `server: { proxy: { '/api': 'http://localhost:8000' } }` must be in `vite.config.js` `server` block — not at root level.
- [npm, install] After scaffolding with `npm create vite`, run `npm install` before `npm run build` — scaffold does not auto-install.
- [vite, wsl] On WSL/Windows with file system watching issues, add `server: { watch: { usePolling: true } }` to `vite.config.js`.

## Python / JWT / Auth

- [jose, import] Package is `python-jose[cryptography]` but import is `from jose import jwt` — NOT `from python_jose import jwt`.
- [passlib, bcrypt] `passlib[bcrypt]` requires the `bcrypt` package separately in some environments. Add both `passlib` and `bcrypt` to `requirements.txt`.
- [jwt, expiry] JWT `exp` claim must be a UTC timestamp. Use `datetime.utcnow() + timedelta(days=7)` — NOT `datetime.now()` which is timezone-local.
- [pydantic, v2] Pydantic v2 (used in FastAPI 0.100+) uses `model_config = ConfigDict(from_attributes=True)` instead of `class Config: orm_mode = True`.
- [pydantic, email] `EmailStr` requires `pydantic[email]` (installs `email-validator`). Add `pydantic[email]` to `requirements.txt`.

## pytest / Testing

- [pytest, conftest] `conftest.py` fixtures must use `yield` not `return` for teardown logic. Missing teardown leaves in-memory DB state between tests.
- [pytest, testclient] `TestClient` from `fastapi.testclient` wraps the app ASGI — use `with TestClient(app) as client:` (context manager) to trigger startup/shutdown events.
- [pytest, import] Running `pytest` from a subdirectory (e.g., `cd backend && pytest`) requires either a `conftest.py` at the `backend/` root or `pythonpath = .` in `pytest.ini` / `pyproject.toml`.
- [pytest, windows] Use `python -m pytest` not bare `pytest` on Windows — avoids PATH issues with virtual environments.

## Windows / Cross-platform

- [windows, python] Always use `python` not `python3` on Windows. `python3` is not aliased by default on Windows Python installs.
- [windows, path] Use forward slashes `/` in all paths inside Python code. Windows accepts them; backslashes cause cross-platform breakage.
- [windows, venv] Activate venv with `venv\Scripts\activate` on Windows (not `source venv/bin/activate`). But inside emberforge, prefer `python -m pip` and `python -m pytest` to avoid activation dependency.
- [windows, npm] `npm` commands work in Git Bash and PowerShell but may need `npm.cmd` in some subprocess contexts on Windows.
