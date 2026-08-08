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
| [tickets/](tickets/) | File-system ticketing system — epics, stories, tasks, sprints, roadmap, milestones |

## What is Thinker CI?

Thinker CI is a self-hosted automation platform combining CI/CD, project management, AI agents, and DevOps tooling in a single product. It is designed to be simpler than GitHub Actions, more scriptable than Jenkins, and multi-tenant from the ground up.

**CI/CD**
- Git-native pipelines triggered by push, pull-request, tag, or schedule events
- Three runner types: Docker containers, Kubernetes pods, or ephemeral GCE VMs (one VM per job, self-deletes after run)
- Real-time log streaming; pipeline config validated at save and dispatch with full error reporting
- HMAC-verified webhooks from GitHub, GitLab, and Bitbucket; every delivery stored for audit

**Project Management**
- Scrum-style hierarchy: Epics → Stories → Tasks → Sub-tasks
- Sprints, milestones, roadmaps, reviews, comments, and activity logs
- Kanban board, backlog, and sprint velocity tracking — all in the same product as your pipelines

**AI Agents**
- Multi-LLM agent service supporting Claude, Gemini, OpenAI, and Amazon Bedrock
- Agent loop (think → act → observe) with a tool registry covering pipelines, PM, tickets, code, and deployments
- MCP server so Claude Code and other MCP clients can interact with thinker-ci directly

**Infrastructure**
- Vanilla JS + Tailwind frontend — fast, no framework dependencies
- Self-hosted Git (Gitea) with SSO proxy and per-tenant org isolation
- Terraform for GCP and AWS; production-ready Kubernetes manifests for all services
- OpenAPI schema at `/api/schema/`, Swagger UI at `/api/docs/`
- Multi-tenant: shared database, row-level isolation, tenant resolved from header or subdomain

## Repositories

| Repo | Purpose |
|---|---|
| [thinker-console](https://github.com/thinker-ci/thinker-console) | Django + DRF API server, Celery workers, multi-tenancy, PM system, AI provider management, pipeline config validation, webhook handler, GCE/Docker/Kubernetes runner support |
| [thinker-agent](https://github.com/thinker-ci/thinker-agent) | FastAPI multi-LLM agent service — Claude, Gemini, OpenAI, Amazon Bedrock; agent loop (think→act→observe), tool registry, streaming sessions |
| [thinker-mcp](https://github.com/thinker-ci/thinker-mcp) | MCP server exposing thinker-ci as 22 tools (tickets, pipelines, PM, agents) and resources (OpenAPI schema, blueprint docs); use with Claude Code or any MCP client |
| [thinker-web](https://github.com/thinker-ci/thinker-web) | Vanilla JS + Tailwind CSS SPA — dashboard, kanban board, backlog, sprints, roadmap, epics, pipelines, runners, agent chat, Swagger UI, settings |
| [thinker-git](https://github.com/thinker-ci/thinker-git) | Self-hosted Git service — Gitea + FastAPI SSO proxy; one Gitea org per tenant, webhook forwarding to thinker-console |
| [thinker-infra](https://github.com/thinker-ci/thinker-infra) | Terraform modules for GCP (GKE, Cloud SQL, Memorystore, Artifact Registry) and AWS (EKS, RDS, ElastiCache, ECR); Kubernetes manifests for all services |
| [thinker-blueprint](https://github.com/thinker-ci/thinker-blueprint) | This repo — architecture, ADRs, glossary, API reference, pipeline schema, webhook docs, file-system ticketing system, Claude Code skills |

## Quick start

See [operations/deployment.md](operations/deployment.md) for setup instructions.
