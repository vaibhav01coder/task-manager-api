# Design Review — Task Management REST API

## Risks & Gaps Identified

### R-1 · No Authentication or Authorisation Layer

**Risk:** All six CRUD endpoints are fully public, meaning any network-reachable client can create, modify, or permanently delete any task without identity verification, making the API unsafe for any multi-user or internet-exposed deployment.
**Decision:** Document the gap explicitly in the README as a known limitation; add a placeholder `middleware/auth.py` stub with a `@require_api_key` decorator that reads an `API_KEY` env var, wired as a Blueprint `before_request` hook, so auth can be activated without restructuring routes.

---

### R-2 · SQLite Concurrency Unsafe Under Concurrent Writes

**Risk:** SQLite's file-level locking means that even modest concurrent write traffic in a shared development or staging environment will produce `OperationalError: database is locked`, causing silent data loss or 500 responses that are difficult to reproduce.
**Decision:** Add `connect_args={"check_same_thread": False}` and `SQLALCHEMY_ENGINE_OPTIONS = {"pool_pre_ping": True}` to `DevelopmentConfig` only, and document clearly that SQLite is strictly single-process; add a startup warning log when `DATABASE_URL` is absent or resolves to SQLite, advising against concurrent use.

---

### R-3 · `db.create_all()` Called at Startup Is Unsafe for Production Schema Management

**Risk:** Auto-provisioning the schema via `db.create_all()` in `app.py` will silently no-op on column additions or renames in an existing production database, meaning schema drift is never detected and migrations are never applied, risking data corruption or runtime errors after upgrades.
**Decision:** Integrate Alembic (via Flask-Migrate) as the migration driver; remove `db.create_all()` from `app.py` and replace it with a `flask db upgrade` step in the README's deployment runbook and in the `docker-compose.yml` entrypoint, while retaining `db.create_all()` only inside the pytest fixture for in-memory test databases.

---

### R-4 · Unbound Pagination `limit` Allows Denial-of-Service via Large Result Sets

**Risk:** The `GET /tasks` pagination handler does not appear to cap the client-supplied `limit` parameter, so a caller requesting `limit=1000000` will load the entire table into memory in a single SQLAlchemy query, exhausting application memory and degrading or crashing the server.
**Decision:** Enforce a hard server-side cap of `limit = min(requested_limit, 100)` with a default of `20` inside the route handler; document the maximum in the README curl examples and return the effective `limit` value in the pagination metadata so clients observe the clamping transparently.

---

### R-5 · `completed_at` Timestamp Relies on Uncontrolled Server Clock With No Timezone Anchor

**Risk:** Using `datetime.utcnow()` or `datetime.now()` without an explicit UTC timezone produces naive datetime objects; if the server's timezone changes or SQLAlchemy serialises the value differently across SQLite and PostgreSQL, `completed_at` values will be inconsistent and comparison queries will return wrong results.
**Decision:** Standardise on `datetime.now(timezone.utc)` throughout the model and handler; store the field as `DateTime(timezone=True)` in the SQLAlchemy column definition; ensure `to_dict()` serialises it as an ISO-8601 string with the `+00:00` suffix so clients always receive an unambiguous timestamp.

---

### R-6 · No Rate Limiting Exposes the API to Abuse and Resource Exhaustion

**Risk:** Without any request throttling, a single client can issue unlimited POST or GET requests per second, exhausting database connection pool slots, CPU, and memory on the host, effectively denying service to all other consumers even without malicious intent.
**Decision:** Add `Flask-Limiter` with an in-process memory store as a zero-infrastructure default (configurable to Redis via a `RATELIMIT_STORAGE_URL` env var for production); apply a conservative global default of `200 per day; 50 per hour` and document override instructions in the README, ensuring the limiter is disabled in the pytest fixture to keep tests deterministic.

---

## Agreed Design Decisions

| ID | Decision |
|---|---|
| DD-01 | Flask 3.x is the web framework; `python app.py` is the single launch command |
| DD-02 | `DATABASE_URL` env var drives SQLAlchemy; no code changes required to switch from SQLite to PostgreSQL |
| DD-03 | Modular directory structure: `models/`, `routes/`, `validators/`, `middleware/` |
| DD-04 | `extensions.py` owns the shared `db` instance to prevent circular imports |
| DD-05 | Hard delete on `DELETE /tasks/{id}`; schema is forward-compatible with a future `deleted_at` soft-delete column |
| DD-06 | Validators are stateless functions returning error dicts; no exceptions raised, no Flask context required |
| DD-07 | In-memory SQLite (`sqlite:///:memory:`) is used exclusively for the pytest suite |
| DD-08 | `completed_at` is set server-side in the `PATCH /tasks/{id}/complete` handler, standardised to UTC with timezone info (updated per R-5) |
| DD-09 | `GET /tasks` always returns pagination metadata (`total`, `page`, `limit`, `pages`); `limit` is capped at 100 server-side (updated per R-4) |
| DD-10 | Global 500 handler returns structured JSON; raw stack traces never reach the client |
| DD-11 | Flask-Migrate (Alembic) manages production schema changes; `db.create_all()` is restricted to test fixtures only (added per R-3) |
| DD-12 | Flask-Limiter provides rate limiting with a memory store default and optional Redis backend via `RATELIMIT_STORAGE_URL` (added per R-6) |
| DD-13 | A `@require_api_key` decorator stub is provided in `middleware/auth.py`; auth is opt-in via env var and documented as a known gap (added per R-1) |

---

## Architecture Updates Applied

**R-1 → Auth stub added.** `middleware/auth.py` added to the component table with responsibility "Optional API key guard; reads `API_KEY` env var; applied per-Blueprint via `before_request`". The component diagram gains an `AuthMiddleware` node between `Router` and `Validator` with a dashed "optional guard" edge. README gains a security notice section.

**R-2 → SQLite concurrency warning added.** `DevelopmentConfig` in `config.py` gains `connect_args` and `pool_pre_ping` settings. `app.py` startup sequence gains a conditional `logging.warning` when the resolved database dialect is `sqlite`. README prerequisites section notes the single-process constraint.

**R-3 → Flask-Migrate integrated.** `Flask-Migrate` added to `requirements.txt` and the tech stack table (row: *Schema Migrations | Flask-Migrate (Alembic) | Safe incremental schema changes; `flask db upgrade` replaces `create_all` in production*). `app.py` initialises `Migrate(app, db)`. `docker-compose.yml` entrypoint updated to `flask db upgrade && python app.py`. `db.create_all()` retained only inside `conftest.py` pytest fixture.

**R-4 → Pagination cap enforced.** Route handler in `routes/tasks.py` updated with `limit = min(request.args.get('limit', 20, type=int), 100)`. Data flow diagram node `U` updated to read "200 OK + tasks array + pagination metadata — limit capped at 100". README curl examples updated to reflect the maximum.

**R-5 → UTC-aware timestamps standardised.** `models/task.py` `completed_at` column changed to `DateTime(timezone=True)`. All `datetime.now()` / `datetime.utcnow()` calls replaced with `datetime.now(timezone.utc)`. `to_dict()` serialises datetime fields with `.isoformat()` which produces the `+00:00` suffix. DD-08 updated accordingly.

**R-6 → Rate limiting added.** `Flask-Limiter` added to `requirements.txt` and tech stack table (row: *Rate Limiting | Flask-Limiter | Protects against abuse; memory store default; Redis-backed in production via `RATELIMIT_STORAGE_URL`*). `extensions.py` gains a `limiter` instance. `conftest.py` sets `RATELIMIT_ENABLED = False` for tests. Component diagram gains a `RateLimiter` node between `Client` and `Router`.