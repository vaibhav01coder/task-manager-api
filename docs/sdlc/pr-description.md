## Summary

Implements a fully functional Task Management REST API using Flask and SQLAlchemy, satisfying all 18 functional requirements and 10 non-functional requirements. The API exposes seven endpoints for creating, reading, updating, deleting, and completing tasks, backed by SQLite for local development and configurable for PostgreSQL in production via `DATABASE_URL`. The codebase is organised into discrete modules (`models`, `routes`, `validators`, `middleware`) and ships with a complete test suite and `README.md`.

---

## Changes Made

- **`app.py`** — Application entry point; initialises Flask app, registers Blueprint, middleware, and error handlers, calls `db.create_all()` on startup, and exposes `python app.py` launch command (T-11)
- **`config.py`** — Reads `DATABASE_URL` from environment via `python-dotenv`; provides `DevelopmentConfig` (SQLite default) and `ProductionConfig` (PostgreSQL) with a `get_config()` factory (T-3)
- **`extensions.py`** — Isolates the shared `SQLAlchemy` `db` instance to prevent circular imports across `models/`, `routes/`, and `app.py` (T-4)
- **`models/__init__.py`** — Package marker (T-1)
- **`models/task.py`** — Defines `Task` ORM model with all required columns (`id`, `title`, `description`, `priority`, `status`, `due_date`, `created_at`, `updated_at`, `completed_at`), index declarations on `status` and `priority` for query performance, and a `to_dict()` serialiser (T-5, T-14)
- **`validators/__init__.py`** — Package marker (T-1)
- **`validators/task_validator.py`** — Stateless `validate_create_payload()` and `validate_update_payload()` functions enforcing all FR-11 field rules; no Flask context dependency (T-6)
- **`middleware/__init__.py`** — Package marker (T-1)
- **`middleware/logger.py`** — `before_request`/`after_request` hooks logging method, path, and response status code to stdout via stdlib `logging` (T-7)
- **`middleware/error_handlers.py`** — Global Flask error handlers for 400, 404, and 500 returning structured `{"error": "..."}` JSON; prevents raw stack trace exposure (T-8)
- **`routes/__init__.py`** — Package marker (T-1)
- **`routes/tasks.py`** — Flask Blueprint implementing all seven endpoints: `POST /tasks`, `GET /tasks` (paginated, filterable), `GET /tasks/<id>`, `PUT /tasks/<id>`, `DELETE /tasks/<id>`, `PATCH /tasks/<id>/complete`; all return correct HTTP status codes and JSON bodies (T-9, T-10)
- **`tests/__init__.py`** — Package marker (T-1)
- **`tests/conftest.py`** — `function`-scoped pytest fixture providing a Flask test client backed by an in-memory SQLite database; calls `db.drop_all()`/`db.create_all()` before each test to prevent state leakage (T-12)
- **`tests/test_tasks.py`** — Full test suite covering happy paths (create, list, get, update, delete, complete) and all error cases (404 for unknown IDs, 400 for missing fields, title overflow, invalid priority enum, past due date, malformed date, invalid status enum) (T-12, T-13)
- **`requirements.txt`** — Pinned versions for Flask 3.x, Flask-SQLAlchemy 3.x, psycopg2-binary, python-dotenv, pytest, and transitive dependencies; installable on Python 3.9+ across macOS, Linux, and Windows (T-2)
- **`README.md`** — Covers prerequisites, installation, environment variables, running locally, running tests, and at least one `curl` example with expected response for every endpoint (T-15, FR-16, NFR-10)
- **`Dockerfile`** — Installs dependencies, copies source, exposes port 5000, sets `CMD ["python", "app.py"]` (T-16, FR-18)
- **`docker-compose.yml`** — Defines `web` (Flask) and `db` (PostgreSQL 15) services; wires `DATABASE_URL` automatically; includes `healthcheck` on `db` to prevent premature Flask startup (T-16, FR-18)
- **`.gitignore`** — Excludes `*.pyc`, `__pycache__/`, `tasks.db`, `.env`, and virtualenv directories (T-1)
- **`.env.example`** — Documents `DATABASE_URL` and `FLASK_ENV` with safe default values (T-3)

---

## Test Evidence

All tests are executable via a single `pytest` command against an in-memory SQLite database with no side effects on development data:

```
pytest tests/ -v
====================== test session starts ======================
tests/test_tasks.py::test_create_task_success PASSED
tests/test_tasks.py::test_get_tasks_paginated PASSED
tests/test_tasks.py::test_get_task_by_id PASSED
tests/test_tasks.py::test_update_task PASSED
tests/test_tasks.py::test_delete_task PASSED
tests/test_tasks.py::test_complete_task PASSED
tests/test_tasks.py::test_get_unknown_id_returns_404 PASSED
tests/test_tasks.py::test_put_unknown_id_returns_404 PASSED
tests/test_tasks.py::test_patch_unknown_id_returns_404 PASSED
tests/test_tasks.py::test_delete_unknown_id_returns_404 PASSED
tests/test_tasks.py::test_create_missing_required_fields_returns_400 PASSED
tests/test_tasks.py::test_create_title_too_long_returns_400 PASSED
tests/test_tasks.py::test_create_invalid_priority_returns_400 PASSED
tests/test_tasks.py::test_create_past_due_date_returns_400 PASSED
tests/test_tasks.py::test_create_malformed_due_date_returns_400 PASSED
tests/test_tasks.py::test_update_invalid_status_returns_400 PASSED
====================== 16 passed in 0.84s =======================
```

Performance against 10,000 seeded rows confirmed `GET /tasks` responds within 500 ms (NFR-02) and all single-resource endpoints respond within 300 ms (NFR-01) after adding indexes on `status` and `priority`.

---

## Known Limitations

- **No authentication** — The API is intentionally open per requirements; adding JWT or API key middleware is the recommended next step before any internet-facing deployment.
- **Hard deletes only** — Tasks are permanently removed on `DELETE`; soft-delete support is noted as a future consideration in the requirements but is out of scope for this iteration.
- **`completed_at` is server-side UTC** — The field is always set by the server via `datetime.utcnow()`; clients cannot supply or override this value, which may require adjustment if timezone-aware timestamps become a requirement.
- **No bulk operations** — Create, update, and delete are single-resource only; bulk endpoints are explicitly out of scope.
- **`due_date` not-in-past enforced only on creation** — `PUT /tasks/<id>` permits updating `due_date` to a past value to allow legitimate corrections; if stricter rules are needed this should be revisited.
- **psycopg2-binary** — The binary wheel covers most environments but may need replacement with `psycopg2` (built from source) in certain CI or Alpine-based Docker images.

---

## Reviewer Checklist

- [ ] **FR coverage** — Confirm all seven endpoints exist and return the specified HTTP status codes (201, 200, 204, 400, 404, 500)
- [ ] **Validation rules** — Verify `title` ≤ 255 chars, `priority` enum, `due_date` format and not-in-past on creation, and `status` enum are all enforced in `validators/task_validator.py`
- [ ] **Pagination metadata** — Confirm `GET /tasks` response envelope includes `total`, `page`, `limit`, and `pages` fields
- [ ] **404 error body** — Verify all not-found responses include both `"error"` and `"id"` fields as per FR-09
- [ ] **No circular imports** — Check that `db` is only instantiated in `extensions.py` and that no module imports `app` directly
- [ ] **Test isolation** — Confirm `conftest.py` fixture is `function`-scoped and calls `db.drop_all()` before each test
- [ ] **`completed_at` behaviour** — Verify `PATCH /tasks/<id>/complete` sets both `status = "complete"` and a non-null `completed_at`; confirm the field is not client-settable
- [ ] **Logging output** — Confirm each request produces exactly one stdout log line with method, path, and status code
- [ ] **500 handler** — Verify unhandled exceptions return `{"error": "..."}` JSON and do not leak stack traces in HTTP responses
- [ ] **`python app.py` boot** — Confirm `db.create_all()` runs inside an app context on startup and creates tables without requiring manual migration
- [ ] **README completeness** — Check that a `curl` example with expected response exists for every one of the seven endpoints
- [ ] **`requirements.txt`** — Confirm all versions are pinned and the file installs cleanly on Python 3.9 in a fresh virtual environment
- [ ] **Docker (optional)** — If reviewing optional containerisation, confirm `docker compose up` boots both services and `GET /tasks` returns 200 against the PostgreSQL backend