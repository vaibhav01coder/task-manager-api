# Requirements — Task Management REST API

## User Story

As a user, I want a REST API to manage my tasks so I can create, read, update and delete tasks from any client application, enabling reliable and scalable task management to replace the current error-prone spreadsheet process.

---

## Clarifying Q&A

| # | Question | Answer |
|---|----------|--------|
| 1 | What authentication or authorization mechanism, if any, should protect the API endpoints (e.g., API key, JWT, or none for local use only)? | No authentication for now; open API |
| 2 | Should the GET /tasks list endpoint support pagination, and if so, what are the default page size and maximum record limits? | Include pagination with `page` and `limit` query params |
| 3 | What are the exact validation rules for each task field — for example, maximum title length, accepted date format for due_date, and whether description is required? | Return 404 with JSON error message *(note: full field-level validation rules remain to be confirmed — defaults defined in FR section below)* |
| 4 | Should deleted tasks be permanently removed from the database or soft-deleted (e.g., marked with a deleted flag) to allow potential recovery? | *(Not directly answered — defaulting to hard delete based on acceptance criteria; soft-delete recommended for future consideration)* |
| 5 | What specific unit test scenarios are required — for example, should tests cover only happy paths or also validation errors, edge cases, and database failures? | *(Not directly answered — tests will cover happy paths and validation/error cases as detailed in FR section below)* |
| 6 | What is the target database for each environment? | SQLite for local development; PostgreSQL for production |
| 7 | Is Docker support required for deployment? | Docker support is optional; primary deployment is `python app.py` |

---

## Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | The API **must** expose a `POST /tasks` endpoint that creates a new task accepting the fields: `title` (required), `description` (optional), `priority` (required; enum: `low`, `medium`, `high`), and `due_date` (required; ISO 8601 format `YYYY-MM-DD`) |
| FR-02 | The API **must** expose a `GET /tasks` endpoint that returns a paginated list of all tasks, supporting optional query filters: `status` and `priority` |
| FR-03 | The `GET /tasks` endpoint **must** support `page` (default: 1) and `limit` (default: 20) query parameters for pagination; responses must include metadata: `total`, `page`, `limit`, and `pages` |
| FR-04 | The API **must** expose a `GET /tasks/{id}` endpoint that returns a single task by its unique identifier |
| FR-05 | The API **must** expose a `PUT /tasks/{id}` endpoint that updates one or more fields (`title`, `description`, `priority`, `due_date`, `status`) of an existing task |
| FR-06 | The API **must** expose a `DELETE /tasks/{id}` endpoint that permanently removes a task from the database |
| FR-07 | The API **must** expose a `PATCH /tasks/{id}/complete` endpoint that sets the task `status` field to `complete` and records a `completed_at` timestamp |
| FR-08 | All API endpoints **must** return responses in JSON format with appropriate HTTP status codes (200, 201, 204, 400, 404, 500) |
| FR-09 | The API **must** return a `404` HTTP status with a descriptive JSON error body (e.g., `{"error": "Task not found", "id": 42}`) for any request referencing a non-existent task ID |
| FR-10 | The API **must** validate all input fields and return a `400` HTTP status with a JSON error body listing specific field-level validation errors on failure |
| FR-11 | Input validation **must** enforce the following rules: `title` max 255 characters and non-empty; `priority` must be one of `low`, `medium`, `high`; `due_date` must be a valid date in `YYYY-MM-DD` format and not in the past on creation; `description` has no maximum length constraint |
| FR-12 | Each task record **must** persist the following fields: `id` (auto-generated integer), `title`, `description`, `priority`, `status` (default: `pending`; enum: `pending`, `in_progress`, `complete`), `due_date`, `created_at`, `updated_at`, `completed_at` (nullable) |
| FR-13 | The application **must** use SQLite as the database for local development, managed via SQLAlchemy ORM |
| FR-14 | The application **must** support PostgreSQL as the production database, with the target database configurable via an environment variable (e.g., `DATABASE_URL`) requiring no code changes |
| FR-15 | The application **must** start locally with the single command `python app.py` and auto-create required database tables on first run if they do not exist |
| FR-16 | The application **must** include a `README.md` with: prerequisites, installation steps, environment variable descriptions, how to run locally, how to run tests, and example API requests/responses |
| FR-17 | The application **must** include unit tests covering: successful creation of a task, retrieval of all tasks and a single task, update and delete operations, the complete endpoint, 404 responses for unknown IDs, and 400 responses for invalid input |
| FR-18 | An optional `Dockerfile` and `docker-compose.yml` **may** be provided to support containerised local deployment; this is not required for acceptance |

---

## Non-Functional Requirements

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-01 | Performance | The API must respond to any single-resource endpoint (`GET /tasks/{id}`, `POST`, `PUT`, `PATCH`, `DELETE`) within 300 ms under normal local load |
| NFR-02 | Performance | The `GET /tasks` list endpoint must respond within 500 ms for datasets up to 10,000 task records on local SQLite |
| NFR-03 | Scalability | Database configuration must be abstracted via `DATABASE_URL` so the app can switch from SQLite to PostgreSQL without code changes, supporting future scaling |
| NFR-04 | Reliability | The application must return a structured JSON error response (including an `error` field) for all unhandled exceptions with HTTP 500 to prevent raw stack trace exposure |
| NFR-05 | Maintainability | Code must be organised into logical modules (e.g., `models`, `routes`, `validators`) rather than a single monolithic file |
| NFR-06 | Maintainability | All dependencies must be declared in a `requirements.txt` file with pinned version numbers |
| NFR-07 | Testability | Unit tests must be executable with a single command (e.g., `pytest`) and must use an in-memory SQLite database to avoid side effects on development data |
| NFR-08 | Portability | The application must run on Python 3.9 or higher on macOS, Linux, and Windows without OS-specific dependencies |
| NFR-09 | Observability | The application must log each incoming request (method, path, response status code) to stdout using Python's standard `logging` module |
| NFR-10 | Usability | The `README.md` must include at least one working `curl` or equivalent example for every endpoint |

---

## Out of Scope

- User authentication, authorisation, or any form of access control (API keys, OAuth, JWT)
- Multi-user support or task ownership/assignment to user accounts
- Soft-delete or task archiving and recovery functionality
- Real-time notifications or webhooks on task state changes
- File or attachment support on tasks
- Sub-tasks, task dependencies, or hierarchical task structures
- Frontend or client application of any kind
- Automated CI/CD pipeline configuration
- Production infrastructure provisioning or cloud deployment
- Rate limiting or API throttling
- Email or push notification reminders for due dates
- Bulk create, update, or delete operations in a single request
- Full-text search across task fields