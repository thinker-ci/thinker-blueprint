# Component Reference

## Console (thinker-console)

**Technology:** Python 3.12, Django 5.x, Django REST Framework

**Responsibilities:**
- User authentication and authorization (token-based)
- Project and pipeline CRUD
- Webhook ingestion from Git providers
- Pipeline run orchestration (via Celery tasks)
- Job log storage and retrieval
- Runner registration and health tracking

**Key Django apps:**

| App | Models | Purpose |
|---|---|---|
| `apps.users` | `User` | Custom auth user with avatar and bio |
| `apps.projects` | `Project` | Git repository connected to Thinker CI |
| `apps.pipelines` | `Pipeline`, `PipelineRun`, `Job`, `JobLog` | Pipeline definitions, execution history, log output |
| `apps.runners` | `Runner` | Registered build agents |
| `apps.api` | — | URL routing, DRF router |

**Process types:**

| Process | Command | Instances |
|---|---|---|
| Web | `gunicorn console.wsgi` | 2–10 (HPA) |
| Worker | `celery -A console worker` | 2–20 (HPA) |
| Beat | `celery -A console beat` | 1 (singleton) |

---

## Runner Agent

Not yet a separate repository — currently the runner logic lives within the Celery worker using the `docker` and `kubernetes` Python SDKs.

**Docker execution path:**
1. Pull the job's image (or use a cached layer)
2. Create a container with the job's environment variables and workspace volume
3. Attach to stdout/stderr and stream lines to `JobLog`
4. On exit, capture exit code and update `Job.status`

**Kubernetes execution path:**
1. Create a `v1.Pod` spec from the job definition
2. Watch pod phase transitions via the k8s watch API
3. Stream pod logs to `JobLog`
4. On pod completion/failure, clean up and record result

---

## PostgreSQL

**Version:** 16

**Schema highlights:**

| Table | Approx. row growth |
|---|---|
| `users` | Slow — admin-managed |
| `projects` | Slow |
| `pipelines` | Slow |
| `pipeline_runs` | Medium — one per commit/trigger |
| `jobs` | Medium — one per job per run |
| `job_logs` | High — hundreds to thousands per job |
| `runners` | Slow |

`job_logs` is the largest table by far. Consider partitioning by `job_id` or archiving logs older than 90 days.

---

## Redis

**Version:** 7

Used for:
- Celery task queue (broker)
- Celery task results (result backend)

Not used for application-level caching at this time (Django's cache framework is not wired up).

---

## GitHub / GitLab / Bitbucket (external)

Thinker CI listens for webhooks. Each project stores a `webhook_secret` generated at creation time. All incoming webhook payloads are verified using HMAC-SHA256 before processing.

Supported event types:
- `push` — branch push
- `pull_request` — PR opened, synchronised, or re-opened
- `create` — tag creation
