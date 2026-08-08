# System Architecture Overview

## High-level diagram

```
                       ┌─────────────────────────────────────────┐
                       │              Thinker CI Platform          │
                       │                                           │
   Git Provider        │  ┌──────────┐      ┌─────────────────┐  │
   (GitHub/GitLab  ────┼─►│  Django  │      │   PostgreSQL     │  │
    /Bitbucket)        │  │  Console │◄────►│   (primary DB)   │  │
        │              │  │  + DRF   │      └─────────────────┘  │
        │ webhooks     │  └────┬─────┘                           │
        │              │       │ enqueues                         │
   Browser/CLI         │  ┌────▼─────┐      ┌─────────────────┐  │
   (users) ────────────┼─►│  Redis   │      │  Celery Worker  │  │
                       │  │  broker  │─────►│  (job dispatch) │  │
                       │  └──────────┘      └────────┬────────┘  │
                       │                             │            │
                       └─────────────────────────────┼────────────┘
                                                      │
                              ┌───────────────────────┼──────────────────┐
                              │            Runner Layer                    │
                              │                                            │
                              │  ┌──────────────┐  ┌──────────────────┐  │
                              │  │ Docker Runner│  │ Kubernetes Runner │  │
                              │  │ (containers) │  │    (k8s pods)    │  │
                              │  └──────────────┘  └──────────────────┘  │
                              └────────────────────────────────────────────┘
```

## Components

### Console (Django + DRF)

The central API server. Responsible for:
- Authenticating users and managing sessions (token-based)
- Serving the REST API consumed by the web UI and CLI
- Receiving and validating webhook payloads from Git providers
- Storing pipeline definitions, run history, and job logs in PostgreSQL
- Enqueueing pipeline dispatch tasks onto the Redis broker

### Redis

Acts as both the Celery task broker and result backend. All async work flows through Redis queues. In production this should be a Redis Sentinel or Cluster deployment for high availability.

### Celery Workers

Long-running processes that consume tasks from Redis. Primary responsibilities:
- `dispatch_pipeline_run` — reads pipeline config, creates Job records, schedules individual jobs
- `execute_job` — selects an available runner and submits the job container
- `reap_stale_runners` — periodic task to mark runners offline when heartbeats stop

### Runners

Separate processes (or pods) that register with the Console API and execute individual Jobs. Two runtime targets are supported:

**Docker runner** — launches each job inside an isolated Docker container on the host. Suited for single-machine or small-team deployments.

**Kubernetes runner** — spawns a transient Kubernetes Pod per job in the configured namespace. Suited for scalable, cloud-native deployments.

Runners poll for work and send a heartbeat every 30 seconds. A runner not seen for 5 minutes is marked offline by the `reap_stale_runners` task.

### PostgreSQL

The sole source of truth for all persistent state:
- Users and authentication tokens
- Projects and their Git provider configuration
- Pipeline definitions (stored as JSON config blobs)
- PipelineRun and Job records with status transitions
- JobLog rows (one per output line, ordered by sequence)

## Deployment topologies

| Topology | When to use |
|---|---|
| `docker-compose.yml` | Production single-node or staging |
| `docker-compose.dev.yml` | Local development with live reload |
| `k8s/` manifests | Production multi-node Kubernetes cluster |

See [operations/deployment.md](../operations/deployment.md) for step-by-step instructions.
