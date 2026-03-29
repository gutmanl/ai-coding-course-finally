# FinAlly Codebase Review

**Date:** 2026-03-28
**Reviewer:** Claude Opus 4.6 (automated review agent)
**Scope:** Full codebase review -- backend, frontend, tests, Docker, scripts, configuration, and alignment with PLAN.md

---

## Executive Summary

The FinAlly project is in an **early stage of implementation**. Only the market data subsystem (`backend/app/market/`) has been built and tested. The vast majority of the platform described in PLAN.md -- the FastAPI application, database layer, portfolio management, trade execution, LLM chat integration, the entire frontend, Docker configuration, E2E tests, and start/stop scripts -- does not yet exist. The market data code that does exist is well-structured and well-tested, but several critical and major gaps remain before this project can function as described.

**Estimated completion: approximately 15-20% of the full project scope.**

---

## Findings by Severity

### CRITICAL

**C1. No FastAPI application entry point exists.**
There is no `main.py`, `app.py`, or any file that instantiates a `FastAPI()` application. The SSE streaming router created by `create_stream_router()` is never mounted anywhere. The backend cannot be started with uvicorn. Without this, the entire system is non-functional -- there is no server to serve API endpoints, SSE streams, or static files.

**C2. API key present in working directory.**
The file `.env` contains a live OpenRouter API key (`sk-or-v1-...`). While `.env` is listed in `.gitignore`, the key is readable by any process or person with filesystem access. If this file was ever committed before being gitignored (which can happen easily), the key is permanently in git history. The key should be rotated immediately. For a course capstone project that may be cloned by students, this is a significant secret exposure risk.

**C3. No database layer exists.**
PLAN.md specifies six database tables (`users_profile`, `watchlist`, `positions`, `trades`, `portfolio_snapshots`, `chat_messages`) with lazy initialization. None of these exist. The `backend/schema/` directory mentioned in the plan does not exist. There is no SQLite connection logic, no schema SQL, no seed data logic, and no migration code.

**C4. No portfolio, trade, watchlist, or chat API endpoints exist.**
The plan specifies 8 REST endpoints (plus the SSE stream). Only the SSE stream router has been implemented. The following are entirely missing:
- `GET /api/portfolio`
- `POST /api/portfolio/trade`
- `GET /api/portfolio/history`
- `GET /api/watchlist`
- `POST /api/watchlist`
- `DELETE /api/watchlist/{ticker}`
- `POST /api/chat`
- `GET /api/health`

**C5. The entire frontend does not exist.**
The `frontend/` directory is completely absent. There is no Next.js project, no TypeScript code, no UI components, no Tailwind configuration, no static export setup. This means no watchlist panel, no charts, no sparklines, no portfolio heatmap, no trade bar, no AI chat panel, and no connection status indicator.

### MAJOR

**M1. No Dockerfile or Docker configuration.**
The plan specifies a multi-stage Dockerfile (Node stage for frontend build, Python stage for backend) and a `docker-compose.yml`. Neither file exists. The application cannot be containerized or deployed.

**M2. No start/stop scripts.**
The plan specifies `scripts/start_mac.sh`, `scripts/stop_mac.sh`, `scripts/start_windows.ps1`, and `scripts/stop_windows.ps1`. The `scripts/` directory does not exist.

**M3. No E2E test infrastructure.**
The plan specifies a `test/` directory with Playwright E2E tests and a `docker-compose.test.yml`. This directory does not exist.

**M4. No LLM integration.**
The plan specifies LiteLLM integration via OpenRouter with structured outputs, mock mode, system prompting, and auto-execution of trades. None of this code exists. The `litellm` package is not listed as a dependency in `pyproject.toml`.

**M5. The `db/` directory and `.gitkeep` are missing.**
The plan specifies `db/.gitkeep` to ensure the directory exists in the repo for Docker volume mounting. Neither the directory nor the file exists.

**M6. PriceCache uses `threading.Lock` in an asyncio context.**
The `PriceCache` class in `cache.py` uses `threading.Lock` for thread safety. The lock is acquired in `update()`, `get()`, `get_all()`, and other methods called from async coroutines (the simulator loop, the SSE generator). While technically correct, it blocks the asyncio event loop for the duration of the lock hold. For the current workload this is negligible, but it is a latent issue if the cache grows or lock contention increases. Consider `asyncio.Lock` or a lock-free design.

**M7. SSE stream sends all prices on every change, not per-ticker deltas.**
The plan says "emits an SSE event only when prices have changed" with version-based change detection. The implementation checks the global cache version, but when any ticker changes, it serializes and sends *all* ticker prices. With 10 tickers updating at 500ms intervals, virtually every SSE event contains all 10 prices, even though typically only a subset changed. This wastes bandwidth and does not match the plan's implied per-ticker granularity.

**M8. Previous review findings remain unaddressed.**
The Codex review (`planning/REVIEW-codex-v1.md`) raised five valid concerns:
1. Price coverage breaks if a held ticker is removed from the watchlist -- no resolution in plan or code
2. Persistence model inconsistency (repo `db/` vs Docker named volume) -- no resolution
3. Trade execution lacks explicit atomicity/transaction guarantees -- no resolution
4. SSE has no keepalive for cloud proxy environments -- no resolution
5. Mock chat payload tests a duplicate watchlist add (NVDA already seeded) -- no resolution

These should be addressed before the relevant features are implemented.

### MINOR

**m1. The `.gitignore` is a generic Python template, not project-specific.**
It does not include entries for `db/finally.db`, `frontend/node_modules/`, `frontend/.next/`, `frontend/out/`, or other project-specific artifacts that will be needed.

**m2. The `conftest.py` fixture `event_loop_policy` is unused.**
The fixture in `backend/tests/conftest.py` defines `event_loop_policy` but no test requests it. With `pytest-asyncio` in `auto` mode and `asyncio_default_fixture_loop_scope = "function"`, it is redundant.

**m3. Ticker normalization is inconsistent across data sources.**
`MassiveDataSource.add_ticker()` normalizes input to uppercase and strips whitespace. Neither `SimulatorDataSource` nor `GBMSimulator` performs any normalization. If a downstream caller passes `"aapl"` to the simulator, it creates a separate entry from `"AAPL"`. This will cause bugs when the watchlist and portfolio systems are integrated.

**m4. The SSE stream router uses a module-level singleton.**
In `stream.py`, `router = APIRouter(...)` is created at module scope. The `create_stream_router()` factory registers a handler on this shared object. If called twice (e.g., in tests), the same route would be registered twice on the same router instance. This breaks the factory pattern's isolation.

**m5. `_fetch_snapshots` return type is unparameterized.**
`MassiveDataSource._fetch_snapshots()` returns `list` without a type parameter. Should be `list[Any]` or a specific SDK type.

**m6. The `massive` package version range is overly broad.**
`pyproject.toml` specifies `massive>=1.0.0` with no upper bound. The code directly accesses SDK internals (`SnapshotMarketType`, `RESTClient`, `snap.last_trade.price`, `snap.last_trade.timestamp`). A breaking major version bump would cause runtime failures. The lockfile mitigates this in practice, but the specifier should be tighter (e.g., `massive>=1.0.0,<2.0.0`).

**m7. No type-checking configuration.**
The project uses type annotations throughout but has no `mypy` or `pyright` configuration. Adding type checking would catch issues early, especially as the codebase grows with database and API layers.

---

## What Exists and Works Well

The market data subsystem is the one completed component, and it is well-executed:

1. **Clean abstraction.** The `MarketDataSource` ABC with `SimulatorDataSource` and `MassiveDataSource` implementations follows the strategy pattern correctly. Downstream code is source-agnostic via the factory.

2. **Correct GBM math.** The simulator uses proper geometric Brownian motion with Cholesky-decomposed correlation matrices. The formula `S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)` is implemented correctly. Correlated random draws and shock events add realism.

3. **Thorough test suite.** 73 tests across 6 modules with good coverage. Tests verify edge cases (empty tickers, nonexistent removal, duplicate adds), math properties (prices always positive after 10k steps), and integration behavior (cache population, lifecycle). The Massive client tests use proper mocking.

4. **Thoughtful SSE design.** Version-based change detection avoids busy-sending. The `retry: 1000\n\n` directive enables automatic browser reconnection. Client disconnect detection (`request.is_disconnected()`) prevents resource leaks.

5. **Good demo script.** `market_data_demo.py` provides a polished Rich terminal dashboard that visually demonstrates the simulator with sparklines, color-coded arrows, and an event log.

6. **Well-organized package.** Clean `__init__.py` with `__all__`, docstrings on all public classes and methods, consistent code style.

---

## Alignment with PLAN.md

| Plan Section | Status | Notes |
|---|---|---|
| Market data simulator | Complete | GBM with correlations, shock events, seed prices |
| Market data Massive client | Complete | REST polling, timestamp conversion, error handling |
| Price cache | Complete | Thread-safe, versioned, full API |
| SSE streaming router | Complete | Router exists but is not mounted to any FastAPI app |
| FastAPI application | **Missing** | No app, no routing, no static file serving |
| Database / schema | **Missing** | No SQLite, no tables, no seed logic |
| Portfolio endpoints | **Missing** | No trade execution, P&L calculation, history |
| Watchlist endpoints | **Missing** | No CRUD operations |
| Chat / LLM integration | **Missing** | No LiteLLM, no structured outputs, no mock mode |
| Frontend (all) | **Missing** | No Next.js project whatsoever |
| Dockerfile | **Missing** | No containerization |
| docker-compose.yml | **Missing** | No compose file |
| Start/stop scripts | **Missing** | No scripts directory |
| E2E tests | **Missing** | No Playwright tests |
| db/ directory | **Missing** | No volume mount target |

---

## Recommendations (Priority Order)

1. **Rotate the OpenRouter API key immediately** -- it is exposed in the local `.env` file.
2. **Create the FastAPI application entry point** -- this unblocks all API development and SSE mounting.
3. **Implement the database layer** -- schema SQL, lazy init, seed data. This unblocks portfolio, watchlist, and chat.
4. **Build the REST API endpoints** -- portfolio, watchlist, trade execution, health check.
5. **Add LLM integration** -- add `litellm` dependency, implement chat endpoint, structured output parsing, mock mode.
6. **Create the frontend** -- initialize Next.js project with TypeScript and Tailwind, configure static export.
7. **Add the Dockerfile** -- multi-stage build as specified in the plan.
8. **Create `db/.gitkeep`** and add project-specific `.gitignore` entries.
9. **Address the Codex review findings** -- particularly the watchlist/positions price subscription gap and trade atomicity requirements.
10. **Normalize ticker input consistently** across both `SimulatorDataSource` and `MassiveDataSource`.
