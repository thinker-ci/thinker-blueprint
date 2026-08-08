# System Architecture Overview

## High-level diagram

```
                        ┌──────────────────────────────────────────────────────────────┐
                        │                     Thinker CI Platform                       │
                        │                                                                │
  Git Provider          │  ┌─────────────┐     ┌──────────────┐     ┌───────────────┐ │
  (GitHub / GitLab  ────┼─►│   thinker-  │     │  PostgreSQL   │     │     Redis     │ │
   / Bitbucket /        │  │   console   │◄───►│  (primary DB) │     │  (broker +    │ │
   thinker-git)         │  │ Django+DRF  │     └──────────────┘     │   cache)      │ │
         │              │  └──────┬──────┘                           └───────┬───────┘ │
         │ webhooks     │         │ enqueues                                  │         │
         │              │         ▼                                           ▼         │
  Browser / CLI         │  ┌─────────────────────────────────────────────────────────┐ │
  (users) ──────────────┼─►│                  Celery Workers                         │ │
                        │  │  dispatch_pipeline_run · execute_job · reap_stale_runners│ │
  thinker-web  ─────────┼─►│                                                         │ │
  (Vanilla JS SPA)      │  └───────────────────────────┬─────────────────────────────┘ │
                        │                               │                                │
                        │  ┌────────────────┐           │                                │
                        │  │  thinker-agent │           │                                │
                        │  │  FastAPI + LLM │◄──────────┤ (tools via HTTP)              │
                        │  │  (Claude /     │           │                                │
                        │  │   Gemini /     │           │                                │
                        │  │   OpenAI /     │           │                                │
                        │  │   Bedrock)     │           │                                │
                        │  └────────────────┘           │                                │
                        │                               │                                │
                        │  ┌────────────────┐           │                                │
                        │  │  thinker-mcp   │           │                                │
                        │  │  MCP server    │◄──────────┤ (tools + resources)           │
                        │  └────────────────┘           │                                │
                        │                               │                                │
                        └───────────────────────────────┼────────────────────────────────┘
                                                         │
                                 ┌───────────────────────┼────────────────────┐
                                 │               Runner Layer                   │
                                 │                                              │
                                 │  ┌──────────────┐  ┌──────────────┐        │
                                 │  │    Docker    │  │  Kubernetes  │        │
                                 │  │   Runner     │  │    Runner    │        │
                                 │  │ (containers) │  │  (k8s pods)  │        │
                                 │  └──────────────┘  └──────────────┘        │
                                 │                                              │
                                 │  ┌───────────────────────────────────────┐  │
                                 │  │             GCE Runner                 │  │
                                 │  │  provision VM → run job → self-delete  │  │
                                 │  └───────────────────────────────────────┘  │
                                 └──────────────────────────────────────────────┘
```

## Service map

| Service | Repo | Port | Technology |
|---|---|---|---|
| API server | thinker-console | 8000 | Django 5, DRF, Celery |
| Agent service | thinker-agent | 8001 | FastAPI, SQLAlchemy async |
| MCP server | thinker-mcp | stdio | Python MCP SDK |
| Frontend | thinker-web | 80 | Vanilla JS, Tailwind CSS |
| Git hosting | thinker-git | 3000 (Gitea) / 8002 (proxy) | Gitea, FastAPI |
| Documentation | thinker-blueprint | — | Markdown |
| Infrastructure | thinker-infra | — | Terraform, Kubernetes |

## Components

### thinker-console (Django + DRF)

The central API server and coordinator. Responsible for:
- User authentication and multi-tenant access control
- REST API consumed by thinker-web, thinker-agent, thinker-mcp, and CLI clients
- Receiving and verifying webhook payloads from Git providers and thinker-git
- Storing all persistent state in PostgreSQL
- Enqueueing and orchestrating pipeline runs via Celery
- PM system: epics, stories, tasks, sprints, milestones, roadmaps, reviews
- AI provider configuration (encrypted API keys for Claude, Gemini, OpenAI, Bedrock)

### thinker-agent (FastAPI)

Multi-LLM agent service. Responsible for:
- Hosting agent sessions backed by a persistent SQLAlchemy database
- Running the think → act → observe agent loop (up to 20 iterations)
- Adapting to four LLM providers via a common `LLMProvider` interface
- Exposing a tool registry: pipeline tools, PM tools, ticket tools, code execution, deployments
- Streaming responses via SSE to thinker-web and MCP clients

### thinker-mcp (Python MCP SDK)

MCP server that exposes thinker-ci as tools and resources to Claude Code and other MCP clients. Proxies calls to thinker-console and thinker-agent over HTTP. Exposes:
- 22 tools across tickets, pipelines, PM, and agents
- Resources: live OpenAPI schema, blueprint docs, individual tickets

### thinker-web (Vanilla JS + Tailwind)

Single-page frontend application. No framework. Hash-based routing, reactive store, streaming chat support. Pages: dashboard, kanban board, backlog, sprints, roadmap, epics, pipelines, pipeline runs, runners, agent chat, Swagger UI, settings.

### thinker-git (Gitea + FastAPI proxy)

Self-hosted Git service. One Gitea org per tenant for isolation. The FastAPI proxy:
- Validates thinker-console tokens and creates/returns per-user Gitea tokens (SSO)
- Receives Gitea system webhooks and forwards them to thinker-console with HMAC verification
- Transparently proxies all other Gitea API calls with admin token injection

### Redis

Celery task broker and result backend. In production: Redis Sentinel or Cluster for HA.

### PostgreSQL

Sole source of truth for all persistent state. Shared database with row-level tenant isolation. In production: Cloud SQL (GCP) or RDS (AWS) with PITR enabled.

## Deployment topologies

| Topology | When to use |
|---|---|
| `docker-compose.yml` in each repo | Local development |
| `thinker-infra/kubernetes/` manifests | Production Kubernetes cluster |
| `thinker-infra/terraform/gcp/` | Provision GKE, Cloud SQL, Memorystore, Artifact Registry |
| `thinker-infra/terraform/aws/` | Provision EKS, RDS, ElastiCache, ECR |

See [operations/deployment.md](../operations/deployment.md) for step-by-step instructions.
