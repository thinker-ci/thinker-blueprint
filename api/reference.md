# API Reference

Base URL: `https://<host>/api/v1/`

All endpoints require `Authorization: Token <token>` except where noted.

---

## Authentication

### Obtain a token
```
POST /auth/token/
Body: { "username": "...", "password": "..." }
Response: { "token": "abc123..." }
```

### Rotate your token
```
POST /auth/token/rotate/
Response: { "token": "new-token..." }
```

---

## Current user

```
GET    /me/           — retrieve your profile
PATCH  /me/           — update bio, avatar_url, etc.
```

---

## Projects

```
GET    /projects/             — list your projects (owned + member)
POST   /projects/             — create a project
GET    /projects/<slug>/      — retrieve a project
PATCH  /projects/<slug>/      — update a project
DELETE /projects/<slug>/      — deactivate a project
```

---

## Pipelines

```
GET    /projects/<slug>/pipelines/    — list pipelines for a project
POST   /projects/<slug>/pipelines/    — create a pipeline
GET    /pipelines/<id>/               — retrieve a pipeline
PATCH  /pipelines/<id>/               — update a pipeline
DELETE /pipelines/<id>/               — delete a pipeline
POST   /pipelines/<id>/trigger/       — manually trigger a run
```

**Trigger body (optional):**
```json
{
  "branch": "main",
  "commit_sha": "abc1234",
  "commit_message": "fix: correct retry logic"
}
```

---

## Pipeline Runs

```
GET    /pipelines/<id>/runs/   — list runs for a pipeline
GET    /runs/<id>/             — retrieve a run (includes jobs)
POST   /runs/<id>/cancel/      — cancel a pending or running run
```

---

## Job Logs

```
GET    /jobs/<id>/logs/        — retrieve all log lines for a job
```

Response:
```json
[
  { "sequence": 1, "line": "[thinker-ci] Starting job: test", "timestamp": "..." },
  { "sequence": 2, "line": "pytest --tb=short", "timestamp": "..." }
]
```

---

## Job log ingest (GCE VM agents)

Called by the Thinker CI job agent running inside a GCE VM to stream output lines
back to the Console.  Requires a valid user token (the Console injects a system token
via instance metadata).

```
POST   /jobs/<id>/ingest-logs/     — append log lines for a job
POST   /jobs/<id>/report-status/   — report terminal status for a job
```

**Ingest-logs body:**
```json
{ "lines": ["Running pytest...", "PASSED", "1 passed in 0.42s"] }
```

**Report-status body:**
```json
{ "status": "success", "exit_code": 0 }
```
`status` must be `"success"` or `"failed"`.

---

## Runners (admin only)

```
GET    /runners/              — list runners
POST   /runners/              — register a new runner (returns token)
GET    /runners/<id>/         — retrieve a runner
PATCH  /runners/<id>/         — update runner configuration
DELETE /runners/<id>/         — deregister a runner
POST   /runners/heartbeat/    — runner liveness check (no auth required; uses runner token in body)
```

**Heartbeat body:**
```json
{ "token": "<runner-token>", "version": "1.0.0" }
```

**GCE runner fields** (included in runner create/update body when `runner_type` is `"gce"`):

```json
{
  "runner_type": "gce",
  "gce_project_id": "my-gcp-project",
  "gce_zone": "us-central1-a",
  "gce_machine_type": "n2-standard-8",
  "gce_image": "family/thinker-ci-runner",
  "gce_image_project": "my-gcp-project",
  "gce_service_account": "thinker-ci-runner@my-gcp-project.iam.gserviceaccount.com",
  "gce_network": "ci-network",
  "gce_subnetwork": "ci-subnet-us-central1",
  "gce_disk_size_gb": 100,
  "gce_use_spot": true,
  "gce_resource_labels": { "team": "platform" },
  "gce_network_tags": ["thinker-ci-runner", "no-external-ip"],
  "gce_scopes": ["https://www.googleapis.com/auth/cloud-platform"]
}
```

---

## Webhook endpoint

Base URL: `https://<host>/hooks/` (not under `/api/v1/`)

```
POST   /hooks/<project_slug>/
```

Inbound-only endpoint for Git providers. Not authenticated with a user token — identity is established via HMAC signature or token header. No CSRF protection (machine-to-machine).

Every inbound request is stored as a `WebhookDelivery` audit record before processing, including rejected requests.

### Provider-specific request shape

#### GitHub

| Header | Value |
|---|---|
| `X-GitHub-Event` | Event name, e.g. `push`, `pull_request`, `ping` |
| `X-GitHub-Delivery` | UUID identifying this delivery (used for idempotency) |
| `X-Hub-Signature-256` | `sha256=<HMAC-SHA256 hex of body using webhook_secret>` |

Supported events: `push` (branch and tag), `pull_request` (opened, synchronize, reopened), `create`, `ping`.

#### GitLab

| Header | Value |
|---|---|
| `X-Gitlab-Event` | Event name, e.g. `Push Hook`, `Tag Push Hook`, `Merge Request Hook` |
| `X-Gitlab-Token` | The plain-text `webhook_secret` from the project |

Supported events: `Push Hook`, `Tag Push Hook`, `Merge Request Hook` (open, update, reopen).

#### Bitbucket

| Header | Value |
|---|---|
| `X-Event-Key` | Event name, e.g. `repo:push`, `pullrequest:created` |
| `X-Request-UUID` | UUID identifying this delivery (used for idempotency) |
| `X-Hub-Signature` | `sha256=<HMAC-SHA256 hex of body using webhook_secret>` |

Supported events: `repo:push` (branch and tag), `pullrequest:created`, `pullrequest:updated`.

### Responses

| Status | Meaning |
|---|---|
| 200 | Accepted and queued, or ping acknowledged, or idempotent re-delivery |
| 200 (skipped) | Signature valid but event matched no pipelines |
| 403 | Signature verification failed |
| 404 | Project slug not found |
| 500 | Unexpected server error |

**Success body:**
```json
{ "detail": "Queued 2 pipeline run(s).", "run_ids": [17, 18] }
```

**Ping body:**
```json
{ "detail": "pong" }
```

**Idempotent re-delivery:**
```json
{ "detail": "Already processed.", "run_ids": [17] }
```

### Setting the webhook secret

Generate a random secret (e.g. `openssl rand -hex 32`) and store it in `Project.webhook_secret`. Then configure your Git provider to sign deliveries with that secret:

- **GitHub**: Repository → Settings → Webhooks → Add webhook → Secret
- **GitLab**: Repository → Settings → Webhooks → Secret token
- **Bitbucket**: Repository → Settings → Webhooks → (secret is passed as the signature key)

---

## Pipeline config schema

A pipeline config is a JSON document stored in `Pipeline.config`:

```json
{
  "image": "python:3.12-slim",
  "stages": [
    {
      "name": "build",
      "jobs": [
        {
          "name": "install",
          "image": "python:3.12-slim",
          "tags": ["linux"],
          "steps": [
            "pip install -r requirements/base.txt"
          ]
        }
      ]
    },
    {
      "name": "test",
      "jobs": [
        {
          "name": "pytest",
          "steps": ["pytest --tb=short -q"]
        }
      ]
    }
  ]
}
```
