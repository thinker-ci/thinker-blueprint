# Architecture Decision Records

## ADR-001: Django as the web framework

**Status:** Accepted

**Context:** We needed a web framework with a mature ORM, admin interface, and ecosystem for building a REST API with user authentication, background task support, and webhook handling.

**Decision:** Use Django with Django REST Framework.

**Rationale:**
- Django's ORM and migration system are well-suited for the relational data model (projects → pipelines → runs → jobs → logs)
- DRF provides token authentication, serializers, and viewsets out of the box
- Django Admin gives us a free management UI for operators
- The Python ecosystem has first-class libraries for both Docker (`docker-py`) and Kubernetes (`kubernetes`) — our two runtime targets

**Alternatives considered:**
- FastAPI — better async support but thinner ecosystem; we would have had to build auth, admin, and ORM tooling from scratch
- Rails — excellent for this kind of CRUD/API product but the team's expertise is in Python

---

## ADR-002: Celery + Redis for async task processing

**Status:** Accepted

**Context:** Pipeline dispatch and job execution cannot happen in the HTTP request-response cycle. We need durable, retryable async tasks.

**Decision:** Use Celery with Redis as the broker and result backend.

**Rationale:**
- Celery integrates natively with Django and auto-discovers tasks from installed apps
- Redis is operationally simple and sufficient for our queue volume
- Celery Beat handles periodic tasks (runner heartbeat reaping, future scheduled pipelines) without a separate cron daemon

**Alternatives considered:**
- Django-Q — lighter weight but smaller community and fewer retry/routing primitives
- RabbitMQ — more durable but adds operational complexity we don't need yet; Redis can be upgraded to Sentinel/Cluster for HA

---

## ADR-003: PostgreSQL as the sole datastore

**Status:** Accepted

**Context:** Thinker CI has a clearly relational domain model. We considered whether any data should live in a document store or time-series DB.

**Decision:** Use PostgreSQL for all persistent state, including job logs (as rows in `job_logs`).

**Rationale:**
- Storing logs as rows lets us use `LIMIT`/`OFFSET` pagination and sequence-ordered queries without a separate log aggregation service
- JSON columns (`pipeline.config`) give us schema flexibility for pipeline definitions without a separate document store
- Fewer moving parts to operate, back up, and reason about

**Trade-off:** `job_logs` will grow quickly. Mitigation: partition by job_id, archive logs older than 90 days to object storage (future work).

---

## ADR-004: Docker + Kubernetes as dual runner targets

**Status:** Accepted

**Context:** Users range from solo developers on a single machine to platform teams running large Kubernetes clusters.

**Decision:** Support both Docker (single-host) and Kubernetes (cluster) runner types via a pluggable `runner_type` field.

**Rationale:**
- Docker runners allow zero-Kubernetes setup for small teams
- Kubernetes runners enable horizontal scale and workload isolation for larger teams
- The runner model abstracts the execution target: the `execute_job` Celery task selects the right SDK path based on `runner.runner_type`

---

## ADR-005: Token-based authentication (no JWT)

**Status:** Accepted

**Context:** We need an auth scheme that works for both human users (browser) and machine clients (CLI, runners).

**Decision:** Use DRF's built-in `TokenAuthentication` (opaque tokens stored in the database).

**Rationale:**
- Simpler to revoke — deleting the `Token` row immediately invalidates all sessions
- No clock-skew issues
- Runners use long-lived tokens; users can rotate their tokens via `POST /api/v1/auth/token/rotate/`

**Trade-off:** Tokens require a DB lookup on every request. At our expected scale this is negligible. If we need to scale to tens of thousands of concurrent API clients, we can layer JWT on top without breaking the API contract.
