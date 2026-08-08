# Thinker CI Blueprint

This repository is the authoritative reference for the Thinker CI/CD platform. It covers architecture, design decisions, terminology, API contracts, and operational guidance.

## Contents

| Section | What you'll find |
|---|---|
| [architecture/](architecture/) | System overview, component diagrams, data flow |
| [design/](design/) | Architecture Decision Records, technology choices, trade-offs |
| [terminology/](terminology/) | Canonical glossary of all domain terms |
| [api/](api/) | API design principles, endpoint reference, webhook contracts |
| [operations/](operations/) | Deployment, scaling, observability, runbooks |
| [runners/](runners/) | Runner type deep-dives — GCE runner config and custom image guide |
| [webhooks/](webhooks/) | Webhook handler — provider setup, event mapping, idempotency, security |
| [pipelines/](pipelines/) | Pipeline config schema — field reference, validation rules, error format, examples |

## What is Thinker CI?

Thinker CI is a self-hosted CI/CD platform built on Django and Celery, designed to orchestrate build and deployment pipelines on Docker, Kubernetes, and Google Compute Engine infrastructure. It provides:

- **Git-native pipelines** — YAML-configured pipelines triggered by push, pull-request, tag, or schedule events
- **Pluggable runners** — three execution backends: Docker containers, Kubernetes pods, or ephemeral GCE VMs
- **On-demand GCE VMs** — each job gets a fresh VM provisioned from a custom image; no cross-job state leakage
- **Real-time log streaming** — job output streamed to the console as it executes
- **REST API** — all functionality exposed via a versioned JSON API
- **Multi-project support** — projects linked to GitHub, GitLab, or Bitbucket repositories
- **Webhook integration** — inbound HMAC-verified webhooks trigger pipelines automatically; every delivery is stored for audit and debugging
- **Config validation** — pipeline configs are validated against a JSON Schema at save time and again at dispatch; all errors reported at once with precise field paths

## Repositories

| Repo | Purpose |
|---|---|
| [thinker-console](https://github.com/thinker-ci/thinker-console) | Django web application and API server |
| [thinker-blueprint](https://github.com/thinker-ci/thinker-blueprint) | This repo — architecture and documentation |

## Quick start

See [operations/deployment.md](operations/deployment.md) for setup instructions.
