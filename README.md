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

## What is Thinker CI?

Thinker CI is a self-hosted CI/CD platform built on Django and Celery, designed to orchestrate build and deployment pipelines on Docker and Kubernetes infrastructure. It provides:

- **Git-native pipelines** — YAML-configured pipelines triggered by push, pull-request, tag, or schedule events
- **Pluggable runners** — build agents that run jobs inside Docker containers or Kubernetes pods
- **Real-time log streaming** — job output streamed to the console as it executes
- **REST API** — all functionality exposed via a versioned JSON API
- **Multi-project support** — projects linked to GitHub, GitLab, or Bitbucket repositories

## Repositories

| Repo | Purpose |
|---|---|
| [thinker-console](https://github.com/thinker-ci/thinker-console) | Django web application and API server |
| [thinker-blueprint](https://github.com/thinker-ci/thinker-blueprint) | This repo — architecture and documentation |

## Quick start

See [operations/deployment.md](operations/deployment.md) for setup instructions.
