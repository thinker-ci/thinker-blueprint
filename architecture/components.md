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

Runner logic lives within the Celery worker using the `docker`, `kubernetes`, and
`google-cloud-compute` Python SDKs.  Three execution backends are supported:

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

**GCE execution path (ephemeral VMs):**
1. `execute_gce_job` task calls `apps.runners.gce.provision_vm()`
2. A fresh GCE VM is created using the runner pool configuration (machine type, image, zone, network)
3. Job context (job ID, Console URL, auth token) is injected via GCE instance metadata
4. The VM boots the custom Thinker CI image; systemd starts the job agent on first boot
5. The agent clones the repo, runs each step inside Docker, and streams log lines to `POST /api/v1/jobs/{id}/ingest-logs/`
6. On completion the agent calls `POST /api/v1/jobs/{id}/report-status/` and self-deletes
7. The Celery poll loop detects terminal status and calls `terminate_vm()` as a safety net

See [runners/gce-runner.md](../runners/gce-runner.md) for configuration and IAM requirements.
See [runners/gce-custom-image.md](../runners/gce-custom-image.md) for building the VM image.

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
