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
A registered build agent that executes Jobs. Runners communicate with the Console via REST API, sending heartbeats every 30 seconds and accepting job assignments. Runners can be Docker-type or Kubernetes-type.

**Project**
A Git repository connected to Thinker CI. Projects belong to an owner (user), have zero or more members, and contain one or more Pipelines. Projects store the webhook secret used to verify incoming events from the Git provider.

**Webhook**
An HTTP POST request sent by a Git provider (GitHub, GitLab, Bitbucket) to the Console when a repository event occurs (push, pull request, tag). Webhooks are verified using HMAC-SHA256 and used to trigger Pipeline Runs automatically.

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
