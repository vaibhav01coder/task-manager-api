# Implementation Plan — Task Management REST API

## Tasks

1. **T-1: Initialise Project Structure & Version Control** — Create the top-level directory layout: `models/`, `routes/`, `validators/`, `middleware/`, `tests/`; add `.gitignore`, empty `__init__.py` files, and an initial `README.md` stub. Establish the canonical folder skeleton that all subsequent tasks populate. Depends on: none. Est: 1h

2. **T-2: Pin Dependencies & Create `requirements.txt`** — Research and pin exact versions for: Flask 3.x, Flask-SQLAlchemy 3.x, psycopg2-binary, python-dotenv, pytest, and any transitive dependencies needed. Produce a `requirements.txt` that installs cleanly on Python 3.9+ across macOS, Linux, and Windows. Depends on: T-1. Est: 1h

3. **T-3: Implement `config.py`** — Read `DATABASE_URL` from the environment (via python-dotenv); define `DevelopmentConfig` defaulting to `sqlite:///tasks.db` and `ProductionConfig` for PostgreSQL; expose a `get_config()` factory so `app.py` selects the correct config class without code changes when `DATABASE_URL` is set. Depends on: T-1. Est: 1h

4. **T-4: Implement `extensions.py`** — Instantiate the shared `SQLAlchemy` `db` object in isolation to prevent circular imports between `app.py`, `models/`, and `routes/`. This single module is imported by all layers that need database access. Depends on: T-1. Est: 0.5h

5. **T-5: Implement `models/task.py`** — Define the `Task` SQLAlchemy ORM model with columns: `id` (auto-increment integer PK), `title` (String 255, non-nullable), `description` (Text, nullable), `priority` (String enum: `low`/`medium`/`high`, non-nullable), `status` (String enum: `pending`/`in_progress`/`complete`, default `pending`), `due_date` (Date, non-nullable), `created_at` (DateTime, server default now), `updated_at` (DateTime, onupdate now), `completed_at` (DateTime, nullable). Implement a `to_dict()` method serialising all fields to JSON-safe Python types (ISO strings for dates). Depends on: T-4. Est: 2h

6. **T-6: Implement `validators/task_validator.py`** — Write stateless validation functions `validate_create_payload(data)` and `validate_update_payload(data)` that return a dict of field-level error messages. Enforce: `title` non-empty and ≤ 255 chars; `priority` one of `low`/`medium`/`high`; `due_date` valid `YYYY-MM-DD` and not in the past on creation; `status` one of the three allowed enum values on update; `description` unconstrained. Functions must be pure Python with no Flask application context dependency. Depends on: T-1. Est: 2h

7. **T-7: Implement `middleware/logger.py`** — Register Flask `@app.before_request` and `@app.after_request` hooks using Python's stdlib `logging` module to write one stdout log line per request containing: HTTP method, path, and response status code. Depends on: T-1. Est: 1h

8. **T-8: Implement `middleware/error_handlers.py`** — Register global Flask error handlers for HTTP 400, 404, and 500. Each handler must return a `jsonify` response with a top-level `error` field and an appropriate descriptive message, ensuring raw stack traces are never exposed to clients. Depends on: T-4. Est: 1h

9. **T-9: Implement `routes/tasks.py` — Core CRUD Endpoints** — Build a Flask Blueprint implementing: `POST /tasks` (201), `GET /tasks/{id}` (200/404), `PUT /tasks/{id}` (200/404/400), `DELETE /tasks/{id}` (204/404). Each handler calls the appropriate validator, raises 404 with the structured error body `{"error": "Task not found", "id": <id>}` when no record is found, and calls `db.session` ORM operations. Wire up pagination skeleton for `GET /tasks` (implemented fully in T-10). Depends on: T-5, T-6, T-8. Est: 3h

10. **T-10: Implement `routes/tasks.py` — List & Complete Endpoints** — Add `GET /tasks` with `page` (default 1) and `limit` (default 20) query parameters, optional `status` and `priority` filter parameters, SQLAlchemy `.filter()` chaining, and a response envelope containing `{"tasks": [...], "total": n, "page": n, "limit": n, "pages": n}`. Add `PATCH /tasks/{id}/complete` that sets `status = "complete"` and `completed_at = datetime.utcnow()` server-side (200/404). Depends on: T-9. Est: 2h

11. **T-11: Implement `app.py` — Application Entry Point** — Create the Flask app factory or direct app object; call `config.get_config()` to configure the database; initialise `db` via `db.init_app(app)`; register the tasks Blueprint, logger middleware, and error handlers; call `db.create_all()` inside an app context on startup to auto-provision schema; add `if __name__ == "__main__": app.run()` so `python app.py` launches the server. Depends on: T-3, T-7, T-8, T-10. Est: 1.5h

12. **T-12: Write Unit Tests — Happy Paths** — In `tests/conftest.py`, configure a pytest fixture that creates a Flask test client backed by an in-memory SQLite database (`sqlite:///:memory:`) and calls `db.create_all()` before each test. Write happy-path tests for: `POST /tasks` returns 201 with correct fields; `GET /tasks` returns paginated envelope; `GET /tasks/{id}` returns the task; `PUT /tasks/{id}` returns updated fields; `DELETE /tasks/{id}` returns 204; `PATCH /tasks/{id}/complete` returns 200 with `status = complete` and a non-null `completed_at`. Depends on: T-11. Est: 3h

13. **T-13: Write Unit Tests — Error & Validation Cases** — Extend the test suite with: 404 response for unknown ID on GET, PUT, PATCH, and DELETE; 400 response for missing required fields on POST; 400 for `title` exceeding 255 characters; 400 for invalid `priority` enum; 400 for `due_date` in the past; 400 for malformed date format; 400 for invalid `status` enum on PUT. Verify all error responses include the `error` field. Depends on: T-12. Est: 2h

14. **T-14: Integration Smoke Test & Performance Check** — Run the full application locally via `python app.py` with SQLite; execute manual `curl` requests against all seven endpoints to confirm end-to-end correctness; seed 10,000 rows into the SQLite database and time `GET /tasks` to verify the ≤ 500 ms NFR-02 requirement; time single-resource endpoints for the ≤ 300 ms NFR-01 requirement. Document any index additions needed (e.g., on `status`, `priority`) and apply them to `models/task.py`. Depends on: T-13. Est: 2h

15. **T-15: Write `README.md`** — Produce the complete README covering: prerequisites (Python 3.9+, pip); installation steps (`git clone`, `pip install -r requirements.txt`, `.env` setup); environment variable descriptions (`DATABASE_URL`, `FLASK_ENV`); how to run locally (`python app.py`); how to run the test suite (`pytest`); and at least one working `curl` example with expected response for every endpoint (`POST`, `GET` list with filters, `GET` by ID, `PUT`, `DELETE`, `PATCH /complete`). Depends on: T-14. Est: 2h

16. **T-16: Optional Docker Support** — Write a `Dockerfile` that installs dependencies, copies source, exposes port 5000, and sets `CMD ["python", "app.py"]`. Write a `docker-compose.yml` with a `web` service (the Flask app) and a `db` service (PostgreSQL 15), wiring `DATABASE_URL` automatically. Confirm `docker compose up` boots the full stack. Depends on: T-15. Est: 2h

---

## Milestones

| Milestone | Tasks | Deliverable |
|---|---|---|
| **M-1: Project Scaffold** | T-1, T-2, T-3, T-4 | Versioned repo with full directory structure, pinned `requirements.txt`, and working config/extensions modules |
| **M-2: Data & Validation Layer** | T-5, T-6 | `Task` ORM model with `to_dict()` and stateless validator functions, both independently testable |
| **M-3: API Fully Operational** | T-7, T-8, T-9, T-10, T-11 | All seven endpoints live; `python app.py` starts cleanly; request logging and structured error handling active |
| **M-4: Test Suite Complete** | T-12, T-13 | `pytest` passes all happy-path and error-case tests against in-memory SQLite; zero side effects on dev data |
| **M-5: Production-Ready Release** | T-14, T-15 | Performance requirements verified; complete `README.md` with curl examples for every endpoint |
| **M-6: Optional Containerisation** | T-16 | `Dockerfile` and `docker-compose.yml` tested; PostgreSQL end-to-end confirmed via `docker compose up` |

---

## Risk Mitigations

| Risk | Mitigation | Owner |
|---|---|---|
| Circular import errors between `app.py`, `models/`, and `routes/` when sharing the `db` instance | Isolate `db = SQLAlchemy()` in `extensions.py` (DD-04); import only from `extensions` in models and routes; never import `app` directly in sub-modules | Tech Lead |
| SQLite `due_date` not-in-past validation passing in tests due to fixture dates becoming stale | Use `datetime.date.today()` dynamically in both validator and tests; generate `due_date` as `today + timedelta(days=1)` in all test fixtures | Tech Lead |
| `GET /tasks` exceeding 500 ms NFR-02 on 10,000 rows without indexes | Add SQLAlchemy `Index` declarations on `status` and `priority` columns in `models/task.py`; verify timing in T-14 and add a composite index if needed | Tech Lead |
| psycopg2-binary wheel unavailable on a developer's OS/Python version combination | Pin `psycopg2-binary` in `requirements.txt` but document that it is only required when `DATABASE_URL` points to PostgreSQL; SQLite path requires no additional driver | Tech Lead |
| `completed_at` timestamp inconsistency if server clock differs from client expectations | Set `completed_at = datetime.utcnow()` server-side in the `PATCH /complete` handler (DD-08); document UTC assumption in README; never accept `completed_at` as a client-supplied field | Tech Lead |
| Global 500 handler masking genuine application bugs during development | Set `FLASK_ENV=development` to enable Flask debug mode locally, which surfaces tracebacks in the terminal while the 500 handler still protects HTTP responses; confirm handler is active only when debug mode is off in tests | Tech Lead |
| Test database state leaking between test cases causing false positives or failures | Use a pytest `function`-scoped fixture that drops and recreates all tables via `db.drop_all()` / `db.create_all()` before each individual test function | Tech Lead |
| Docker Compose PostgreSQL service not ready before Flask container starts | Add `depends_on` with a `healthcheck` on the `db` service in `docker-compose.yml` and a retry loop or `pg_isready` check in the entrypoint script | Tech Lead |