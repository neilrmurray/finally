# FinAlly — Comprehensive Project Review

**Date:** 2026-05-18
**Reviewer:** Claude Sonnet 4.6
**Scope:** Full codebase audit against PLAN.md; readiness for next development phase

---

## 1. What Has Been Built

### Completed: Market Data Subsystem

The only code that exists is the market data subsystem in `backend/app/market/` (8 modules). Everything else called for by PLAN.md — the FastAPI application entry point, database layer, portfolio/trade/watchlist/chat API routes, frontend, Docker infrastructure, and start/stop scripts — has not been started.

| Component | Status |
|---|---|
| Market data models, cache, interface | Complete |
| GBM simulator | Complete |
| Massive (Polygon.io) client | Complete |
| Factory (env-based selection) | Complete |
| SSE stream endpoint (factory) | Complete |
| 73 unit/integration tests | Complete |
| Rich terminal demo | Complete |
| FastAPI app entry point (`main.py` / `app.py`) | Missing |
| Database schema + lazy init | Missing |
| Portfolio, trade, watchlist, chat API routes | Missing |
| LLM (OpenAI gpt-4o) integration | Missing |
| Frontend (Next.js) | Missing |
| Dockerfile (multi-stage) | Missing |
| docker-compose.yml | Missing |
| `db/` directory with `.gitkeep` | Missing |
| `scripts/` start/stop scripts | Missing |
| `test/` Playwright E2E tests | Missing |
| `.env.example` | Missing |

---

## 2. Code Quality Assessment

### Market Data — Strong

The market data code is well-structured and matches the PLAN.md specification closely. Specific observations:

**`backend/app/market/models.py`**
- `PriceUpdate` is a frozen, slotted dataclass — good immutability and memory efficiency.
- `change_percent` guards against `previous_price == 0` — correct.
- `to_dict()` computes `change`, `change_percent`, and `direction` as properties and serializes them — this is the right shape for SSE and the frontend.

**`backend/app/market/cache.py`**
- Thread-safe with a single `Lock`. Correct for the single-writer / multiple-reader model.
- The `version` property is read without acquiring the lock. This is a benign data race (integer reads/writes are atomic on CPython) but is worth noting. It does not affect correctness for its intended use (SSE change detection) since a missed increment just delays one SSE event by 500ms.
- `get_all()` returns a shallow copy of `_prices` — correct; prevents mutation of the internal dict by callers.
- `update()` uses `timestamp or time.time()` — this is a subtle bug: if `timestamp` is passed as `0.0` (a valid Unix epoch timestamp), it will be treated as falsy and replaced with `time.time()`. Should be `timestamp if timestamp is not None else time.time()`.

**`backend/app/market/simulator.py`**
- GBM implementation is mathematically correct: `S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)`.
- Cholesky decomposition for correlated moves is properly implemented.
- `_rebuild_cholesky` is called on every `add_ticker` / `remove_ticker` — acceptable at n < 50.
- The `_run_loop` catches all exceptions and logs them — prevents a bad tick from killing the background task.
- `SimulatorDataSource.add_ticker()` silently does nothing if `self._sim` is `None` (i.e., `start()` was never called). A log warning or guard would make this more debuggable.

**`backend/app/market/stream.py`**
- The `router` is created at module level (line 17) and then the route is registered inside `create_stream_router()` on every call. If `create_stream_router()` is called more than once (e.g., in tests), the `/prices` route will be registered multiple times on the same router object, which FastAPI will silently accept but may cause unexpected behavior. The router should be created inside the factory function.
- SSE event format uses `data: {all_tickers_as_dict}\n\n` — this sends the full snapshot of all tickers on every version bump. PLAN.md specifies this behavior, so it is correct, but note it means a single-ticker price change causes a full re-send of all tickers.
- `request.client` could theoretically be `None` in some proxy configurations; `client_ip` safely falls back to `"unknown"` — handled.
- Disconnect detection via `await request.is_disconnected()` is correct for Starlette/FastAPI.

**`backend/app/market/massive_client.py`**
- `asyncio.to_thread` for the synchronous `RESTClient` call is the correct pattern to avoid blocking the event loop.
- Error handling in `_poll_once` catches all exceptions and logs without re-raising — correct for a polling loop.
- `add_ticker` normalizes to uppercase and strips whitespace. `SimulatorDataSource.add_ticker` does not — inconsistency. The simulator will accept `"aapl"` as a new ticker distinct from `"AAPL"`.

**`backend/app/market/seed_prices.py`**
- Prices are reasonable approximations; they are the simulator's starting point, not real-time values, which is appropriate.
- TSLA is in `CORRELATION_GROUPS["tech"]` but `_pairwise_correlation` explicitly short-circuits it to `TSLA_CORR = 0.3` for any pair involving TSLA. This is correct per the spec but means TSLA is simultaneously in the tech set and treated independently — a minor conceptual oddity, not a bug.

**`backend/pyproject.toml`**
- `openai` and `pydantic` are absent from `[project.dependencies]` but will be required for LLM integration (per the `cerebras` skill spec: `uv add openai pydantic`). They must be added before LLM work begins.
- `python-dotenv` is absent. PLAN.md §5 says the backend reads `.env` from the project root. Without `python-dotenv` (or equivalent), environment variables will only be available if the process is started with `--env-file` (Docker) or variables are set in the shell. For local development outside Docker, `.env` will not be loaded automatically. This needs to be resolved.
- `aiosqlite` or `sqlite3` async wrapper is absent. SQLite work will need a driver decision.
- The `[tool.ruff.lint]` `ignore = ["E501"]` combined with `line-length = 100` means line-length is set but not enforced by lint (only by the formatter). This is intentional but worth knowing.

**`backend/tests/`**
- 73 tests covering models (100%), cache (100%), simulator (~98%), factory (100%), and Massive client (mocked). Coverage is strong for the completed subsystem.
- `tests/conftest.py` provides an `event_loop_policy` fixture but does not yield or return a new policy object — it instantiates `DefaultEventLoopPolicy` and returns it. For `pytest-asyncio`, this fixture signature is recognized; this is fine for current usage.
- `test_simulator.py` line 48 tests `sim._tickers` (a private attribute). Accessing private attributes in tests is acceptable for a unit test but creates a coupling to implementation details.

### Root-Level Issues

**`.env` contains a real API key and is committed to git (shown in git status as tracked but modified).** The `.gitignore` lists `.env` — meaning it was previously not tracked. The current state where `.env` appears in `git status` as modified suggests it may have been force-added or the gitignore was applied after the initial commit. **This is a security issue: the OpenAI API key `sk-proj-...` visible in `.env` should be rotated immediately.**

**`README.md` (root) contradicts PLAN.md in two places:**
- States AI uses `LiteLLM → OpenRouter (Cerebras inference)`, but PLAN.md §9 and §13 decided on `OpenAI SDK → gpt-4o → direct OpenAI`.
- References `OPENROUTER_API_KEY` as the required environment variable, but PLAN.md uses `OPENAI_API_KEY`.

**`backend/README.md`** has `uv sync --dev` in two places. The correct flag is `uv sync --extra dev` (as documented in `backend/CLAUDE.md`). Minor documentation inconsistency.

**`.claude/skills/cerebras/SKILL.md`** is labeled "cerebras-inference" but the code examples use the vanilla OpenAI SDK pointed at OpenAI directly (not Cerebras/OpenRouter). The skill name is misleading given the actual integration pattern. The SKILL.md title should reflect what it does: OpenAI SDK with gpt-4o.

---

## 3. Architecture Adherence

| PLAN.md Requirement | Adherence |
|---|---|
| Strategy pattern (ABC + two implementations) | Correct |
| PriceCache as single point of truth | Correct |
| SSE over WebSockets | Correct |
| `create_market_data_source` factory | Correct |
| `MASSIVE_API_KEY` env var drives selection | Correct |
| SSE endpoint at `GET /api/stream/prices` | Implemented as a factory router; will need mounting on a FastAPI app (not yet built) |
| Seed prices for default 10 tickers | Correct |
| ~500ms update interval | Correct (default `update_interval=0.5`) |
| GBM with correlated moves | Correct (Cholesky) |
| Random shock events | Correct (0.1% per tick) |
| Simulator runs as in-process background task | Correct (`asyncio.create_task`) |
| Massive REST polling (not WebSocket) | Correct |
| `uv` project management | Correct |
| SQLite at `db/finally.db` | Not started |
| Lazy DB init on first request | Not started |
| All specified API endpoints | Not started |
| `user_id = "default"` in all tables | Not started |
| LLM structured output | Not started |
| `LLM_MOCK=true` mock mode | Not started |
| Frontend: Next.js static export | Not started |
| Tailwind + dark theme | Not started |
| Lightweight Charts (TradingView) | Not started |
| Multi-stage Dockerfile | Not started |
| `scripts/` start/stop | Not started |
| `test/` Playwright E2E | Not started |

---

## 4. Issues and Gaps

### Critical

1. **API key exposed in `.env`** (`C:\Users\neilr\OneDrive\Documents\Cursor Projects\finally\.env`, line 1). The key should be rotated and `.env` should never appear in git history.

2. **No FastAPI app entry point exists.** `backend/app/` contains only `__init__.py` and `market/`. There is no `main.py`, `app.py`, or equivalent file to instantiate the FastAPI application, register the stream router, configure CORS/static file serving, or define startup/shutdown lifecycle hooks. The market data subsystem cannot currently be served.

3. **`stream.py` module-level router bug** (`backend/app/market/stream.py`, line 17): `router = APIRouter(...)` is created at module import time, not inside `create_stream_router()`. The route handler is registered inside the factory function. If `create_stream_router()` is ever called more than once (e.g., tests creating multiple app instances), the `/prices` route will be registered multiple times on the same router object. Fix: move `router = APIRouter(...)` inside `create_stream_router()`.

### Significant

4. **`timestamp or time.time()` falsy-zero bug** (`backend/app/market/cache.py`, line 30). Should be `timestamp if timestamp is not None else time.time()`. This only affects the Massive client (which passes actual Unix timestamps), but passing `0.0` would silently use `time.time()` instead.

5. **Missing dependencies for upcoming work:** `openai`, `pydantic`, `python-dotenv`, and an async SQLite driver (`aiosqlite` is standard for FastAPI + SQLite) are not in `pyproject.toml`. These must be added before backend API work begins.

6. **Ticker normalization inconsistency:** `MassiveDataSource.add_ticker()` uppercases and strips the ticker; `SimulatorDataSource.add_ticker()` does not. The watchlist API endpoint (when built) will need to normalize tickers before calling `add_ticker()`, or the simulator should be updated to match.

7. **Missing project scaffolding:** The following are absent and are prerequisites for any integration or deployment work:
   - `db/` directory with `.gitkeep`
   - `Dockerfile`
   - `docker-compose.yml`
   - `scripts/start_mac.sh`, `scripts/stop_mac.sh`, `scripts/start_windows.ps1`, `scripts/stop_windows.ps1`
   - `test/` directory
   - `.env.example`

### Minor

8. **`README.md` (root) describes wrong LLM provider** — says OpenRouter/Cerebras/LiteLLM; should be OpenAI SDK direct. References `OPENROUTER_API_KEY`; should be `OPENAI_API_KEY`.

9. **`backend/README.md` wrong flag** — `uv sync --dev` should be `uv sync --extra dev`.

10. **`version` property read without lock** (`backend/app/market/cache.py`, line 65–67). Not a real bug in CPython (integer reads are atomic), but technically a data race. A future multi-threaded non-CPython runtime would expose this.

11. **No application-level logging configuration.** The market data modules use `logging.getLogger(__name__)` correctly, but without a root logger configured in the app entry point, log output will be silent by default.

12. **`conftest.py` fixture is vestigial.** The `event_loop_policy` fixture in `backend/tests/conftest.py` instantiates and returns a `DefaultEventLoopPolicy` but does not set it. `pytest-asyncio >= 0.21` with `asyncio_mode = "auto"` (set in `pyproject.toml`) handles event loops automatically; this fixture does nothing useful.

---

## 5. Readiness for Next Phase

The market data subsystem is solid and ready to be integrated. The next development phase should proceed in this order:

**Step 1 — Backend application foundation (prerequisite for everything)**
- Create `backend/app/main.py` (FastAPI app, lifespan handler for market data start/stop, static file serving)
- Add missing dependencies: `openai`, `pydantic`, `python-dotenv`, `aiosqlite`
- Create `backend/app/database.py` with schema SQL and lazy init logic
- Mount `create_stream_router(cache)` onto the app

**Step 2 — Database + Portfolio/Watchlist/Trade API routes**
- Implement `backend/app/db/` schema and seed logic
- Implement `GET /api/portfolio`, `POST /api/portfolio/trade`, `GET /api/portfolio/history`
- Implement `GET /api/watchlist`, `POST /api/watchlist`, `DELETE /api/watchlist/{ticker}`
- Implement `GET /api/health`
- Wire watchlist changes to `source.add_ticker()` / `source.remove_ticker()`

**Step 3 — LLM chat integration**
- Implement `POST /api/chat` and `GET /api/chat/history`
- Use OpenAI SDK with `client.beta.chat.completions.parse()` for structured outputs
- Implement `LLM_MOCK=true` mock path

**Step 4 — Frontend**
- Next.js with TypeScript, `output: 'export'`, Tailwind CSS
- EventSource connection to `/api/stream/prices`
- Lightweight Charts (TradingView) for sparklines and main chart
- All UI panels per PLAN.md §10

**Step 5 — Docker + scripts**
- Multi-stage Dockerfile
- `scripts/` start/stop for macOS and Windows
- `db/` directory with `.gitkeep`, `.env.example`

**Step 6 — Testing**
- Backend pytest for portfolio math, trade edge cases, LLM mock
- Frontend React Testing Library unit tests
- `test/docker-compose.test.yml` + Playwright E2E

The stream router module-level router issue (item 3 above) should be fixed before the FastAPI app is created.
