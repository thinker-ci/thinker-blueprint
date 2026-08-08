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

## ADR-006: GCE ephemeral VMs as a third runner type

**Status:** Accepted

**Context:** Docker runners require a permanently-running host; Kubernetes runners require
a cluster.  Some workloads need full VM isolation, GPU access, large build caches on SSD,
or a completely clean environment on every run — none of which are easily satisfied
by containers.

**Decision:** Add a `gce` runner type that provisions a fresh Google Compute Engine VM per
job using `google-cloud-compute`, then deletes it when the job is done.

**Rationale:**
- Each job gets a guaranteed-clean OS; no cross-job state leakage is possible
- Machine type is configurable per runner pool — including GPU, high-memory, and ARM instances
- Docker-in-Docker is native (no privileged containers or DinD side-cars needed)
- Spot VMs reduce cost by 60–91% for workloads that tolerate preemption
- The `Runner` model already has a `runner_type` discriminator, so adding `gce` requires
  no schema changes beyond new GCE-specific fields

**Alternatives considered:**
- Always use Docker-in-Docker on Kubernetes — possible but requires privileged pods and
  couples job isolation to cluster-level security policy
- Reuse a pool of long-lived GCE VMs — avoids boot latency but sacrifices cleanliness
  and makes secret exposure risks harder to reason about

**Trade-offs:**
- VM boot latency (20–60 s) is much higher than container start (< 1 s)
- Requires `google-cloud-compute` credentials on the Console process
- The job-agent-in-VM callback architecture (log ingest, status report) adds two new API
  endpoints that must be hardened against abuse

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

---

## ADR-007: JSON Schema + semantic validation for pipeline configs

**Status:** Accepted

**Context:** `Pipeline.config` is a freeform `JSONField`. Without validation, a typo
(empty `steps`, missing `image`, duplicate stage name) is only discovered when a job
tries to execute, producing a cryptic Celery error deep in the task log rather than
a clear message at the point of authorship.

**Decision:** Validate `Pipeline.config` at every entry point using a two-layer
approach in `apps/pipelines/config_schema.py`:

1. **JSON Schema (Draft 7)** — enforces structural correctness: required keys, type
   constraints, pattern restrictions on names, numeric bounds on `timeout`, and an
   `additionalProperties: false` guard against undocumented keys.
2. **Semantic checks** — enforce cross-field rules the schema cannot express: unique
   stage names, unique job names within a stage, and the requirement that every job
   has an `image` (either per-job or inherited from the top-level field).

**Enforcement points:**
- `PipelineSerializer.validate_config` — `400` with a full error list before the
  record is saved
- `Pipeline.clean()` — caught by Django admin and any `full_clean()` caller
- `dispatch_pipeline_run` Celery task — fail-fast guard before any jobs are created;
  writes a readable error to `PipelineRun.error_message`

**Rationale:**
- Reporting all errors in one response (not just the first) lets authors fix a
  broken config in one round-trip instead of iterating through problems one at a time
- Enforcing validation in the task as well means configs saved before the validator
  existed are still caught cleanly at dispatch time
- `jsonschema` Draft 7 is the de facto Python standard; no bespoke parser needed
- `additionalProperties: false` prevents silent schema drift — adding an undocumented
  key is an immediate error, not a silent no-op

**Alternatives considered:**
- Validate only at the API layer — configs created via Django shell or migrations
  would bypass validation; runtime errors would still be cryptic
- Store configs as YAML in a separate file — adds filesystem coupling and complicates
  the API; the JSON-in-DB approach is already established

**Trade-offs:**
- `jsonschema` adds a dependency (`jsonschema==4.25.1` in `requirements/base.txt`)
- Schema changes require updating `_PIPELINE_SCHEMA`, semantic checks, tests, and
  this document — but that friction is intentional: undocumented schema changes
  should not be easy to slip through
