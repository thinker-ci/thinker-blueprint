# Glossary

## Core domain terms

**Pipeline**
A named, versioned definition of the build/test/deploy workflow for a Project. Stored as a JSON config blob containing one or more Stages. Pipelines are triggered by Git events or manually.

**Pipeline Run**
A single execution of a Pipeline, initiated by a trigger event (push, pull request, tag, schedule, or manual). A Run tracks the overall status and links to all Jobs produced by that execution.

**Stage**
A logical group of Jobs within a Pipeline. Jobs within the same stage may run in parallel; stages run sequentially by default. Example stages: `build`, `test`, `deploy`.

**Job**
The smallest unit of work in a Pipeline. A Job maps to a single container execution on a Runner. Each Job belongs to exactly one Stage and one PipelineRun.

**Step**
An individual command within a Job (e.g., `npm install`, `pytest`, `docker build`). Steps are defined in the pipeline config and executed sequentially inside the Job container.

**JobLog**
A single line of stdout/stderr output captured from a running Job container. Stored as rows in the database ordered by `sequence` number. Streamed to the console in real time.

**Runner**
A registered build agent that executes Jobs. Runners communicate with the Console via REST API. Three runner types are supported: Docker (persistent agent on a single host), Kubernetes (pods on a cluster), and GCE (ephemeral VMs provisioned per job).

**GCE Runner**
A runner of type `gce` that provisions a fresh Google Compute Engine VM for every job. The VM boots from a custom image containing the Thinker CI job agent, executes exactly one job, and then self-deletes. The Runner model record describes the pool configuration (machine type, image, zone, network) rather than a specific running machine.

**Job Agent**
The shell script (`/etc/thinker-ci/agent.sh`) embedded in the GCE custom image. It reads its job assignment from the GCE instance metadata server, clones the repo, runs each step inside Docker, streams log output back to the Console via `POST /api/v1/jobs/{id}/ingest-logs/`, and reports the terminal status via `POST /api/v1/jobs/{id}/report-status/`.

**Image Family**
A Google Cloud concept used to group versioned VM images.  Thinker CI uses the family name `thinker-ci-runner`.  Setting `gce_image: "family/thinker-ci-runner"` always resolves to the latest published image in that family, enabling seamless image updates without changing runner configuration.

**Spot VM**
A GCE Spot (formerly Preemptible) VM that is significantly cheaper (60–91%) than a standard VM but may be reclaimed by Google at any time.  Enabled per runner pool via `gce_use_spot: true`.  Suitable for non-critical or retry-tolerant CI jobs.

**Self-destruct**
The behaviour of a GCE job VM deleting itself via `gcloud compute instances delete` after the job completes.  Controlled by the `thinker-ci-self-destruct` instance metadata key.  Backed up by the `reap_orphaned_gce_instances` Celery task.

**Project**
A Git repository connected to Thinker CI. Projects belong to an owner (user), have zero or more members, and contain one or more Pipelines. Projects store the webhook secret used to verify incoming events from the Git provider.

**Webhook**
An HTTP POST request sent by a Git provider (GitHub, GitLab, Bitbucket) to the Console when a repository event occurs (push, pull request, tag). Webhooks arrive at `POST /hooks/<project_slug>/` and are verified using HMAC-SHA256 (GitHub, Bitbucket) or a plain token header (GitLab). Used to trigger Pipeline Runs automatically.

**WebhookDelivery**
An audit log record created for every inbound webhook request, including rejected ones. Stores the raw headers, raw payload, normalised event fields, processing status (`received`, `processed`, `skipped`, `rejected`, `error`), and the IDs of any PipelineRuns created. Viewable in the Django admin.

**Delivery ID**
A provider-supplied UUID included in each webhook request (`X-GitHub-Delivery` for GitHub, `X-Request-UUID` for Bitbucket). Used to make webhook processing idempotent — if the same delivery ID arrives twice (e.g. due to a provider retry), the second request returns the cached result without re-running pipelines.

**Webhook secret**
A random string stored on the Project (`webhook_secret`) and shared with the Git provider. GitHub and Bitbucket use it as the HMAC-SHA256 key; GitLab compares it as a plain token. Generated per-project — `openssl rand -hex 32` is a suitable command.

**Provider event**
The raw event name from the Git provider's webhook header, before normalisation. Examples: `push` (GitHub), `Push Hook` (GitLab), `repo:push` (Bitbucket). Stored on `WebhookDelivery.provider_event` for debugging.

**Normalised event type**
The provider-agnostic event classification used for pipeline matching. Values: `push`, `pull_request`, `tag`, `ping`. Stored on `WebhookDelivery.event_type` and matched against `Pipeline.trigger_events`.

**Trigger event**
The action that initiated a Pipeline Run. Values: `push`, `pull_request`, `tag`, `schedule`, `manual`.

---

## Infrastructure terms

**Console**
The Django web application and API server. The central coordinator of the Thinker CI platform.

**Worker**
A Celery worker process that consumes tasks from the Redis broker. Workers dispatch pipeline runs, execute jobs, and run periodic maintenance tasks.

**Beat**
The Celery beat scheduler process. Runs periodic tasks on a cron-like schedule (e.g., reaping stale runners). Must run as a singleton — do not scale this process horizontally.

**Broker**
Redis, used as the message transport between the Console (task producer) and Celery Workers (task consumers).

**Heartbeat**
A `POST /api/v1/runners/heartbeat/` request sent by a Runner every 30 seconds to signal that it is online and healthy. A runner not seen for 5 minutes is marked `offline` by the `reap_stale_runners` periodic task.

**Token**
An opaque 40-character string used to authenticate API requests. Users hold one token at a time and can rotate it. Runners each have a long-lived registration token.

---

## Status values

| Term | Entity | Meaning |
|---|---|---|
| `pending` | Run, Job | Created but not yet started |
| `running` | Run, Job | Currently executing |
| `success` | Run, Job | Completed with exit code 0 |
| `failed` | Run, Job | Completed with non-zero exit code |
| `cancelled` | Run, Job | Stopped by user before completion |
| `online` | Runner | Sending heartbeats; available for work |
| `offline` | Runner | Not seen recently; will not receive jobs |
| `busy` | Runner | At or above `max_concurrent_jobs` capacity |
