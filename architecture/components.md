# Component Reference

## thinker-console

**Repo:** https://github.com/thinker-ci/thinker-console
**Technology:** Python 3.12, Django 5.x, Django REST Framework, Celery, jsonschema

**Responsibilities:**
- User authentication and authorization (token-based, custom `AUTH_USER_MODEL`)
- Multi-tenancy — shared DB, row-level isolation via `Tenant` FK; tenant resolved from `X-Tenant-Slug` header or subdomain
- Project and pipeline CRUD with JSON Schema + semantic config validation
- Webhook ingestion from GitHub, GitLab, Bitbucket, and thinker-git
- Pipeline run orchestration via Celery tasks
- Job log storage and retrieval
- Runner registration, heartbeat tracking, GCE VM provisioning
- PM system — epics, stories, tasks, sprints, milestones, roadmaps, reviews, activity logs
- AI provider management — Fernet-encrypted API keys for Claude, Gemini, OpenAI, Bedrock
- OpenAPI schema at `/api/schema/`, Swagger UI at `/api/docs/`

**Key Django apps:**

| App | Models | Purpose |
|---|---|---|
| `apps.users` | `User` | Custom auth user |
| `apps.tenants` | `Tenant`, `TenantMembership` | Multi-tenant isolation |
| `apps.projects` | `Project` | Git repository connected to Thinker CI |
| `apps.pipelines` | `Pipeline`, `PipelineRun`, `Job`, `JobLog` | Pipeline definitions, run history, log output |
| `apps.runners` | `Runner` | Registered build agents (Docker, Kubernetes, GCE) |
| `apps.webhooks` | `WebhookDelivery` | Inbound webhook audit log |
| `apps.pm` | `Epic`, `Story`, `Task`, `Sprint`, `Milestone`, `Roadmap`, `Review`, `Comment`, `ActivityLog` | Scrum PM system |
| `apps.ai` | `AIProvider` | Encrypted LLM API key storage |

**Process types:**

| Process | Command | Instances |
|---|---|---|
| Web | `gunicorn console.wsgi` | 2–10 (HPA) |
| Worker | `celery -A console worker` | 3–20 (HPA) |
| Beat | `celery -A console beat` | 1 (singleton) |

---

## thinker-agent

**Repo:** https://github.com/thinker-ci/thinker-agent
**Technology:** Python 3.12, FastAPI, SQLAlchemy async, asyncpg

**Responsibilities:**
- Host persistent agent sessions backed by PostgreSQL
- Run the think → act → observe loop (max 20 iterations per turn)
- Adapt to four LLM providers through a common `LLMProvider` interface
- Maintain conversation history with context-window management
- Execute registered tools and return observations back to the loop
- Stream responses over SSE

**LLM providers:**

| Provider | Module | SDK |
|---|---|---|
| Anthropic Claude | `llm/claude.py` | `anthropic` |
| Google Gemini | `llm/gemini.py` | `google-generativeai` |
| OpenAI / Codex | `llm/openai_provider.py` | `openai` |
| Amazon Bedrock | `llm/bedrock.py` | `boto3` bedrock-runtime |

**Tool groups:**

| Module | Tools |
|---|---|
| `tools/console_tools.py` | list_pipelines, trigger_pipeline, get_run_status, get_job_logs, cancel_run, list_runners |
| `tools/pm_tools.py` | list_epics, create_story, update_task_status, get_sprint_board, add_comment |
| `tools/ticket_tools.py` | Read and write file-system tickets in thinker-blueprint |
| `tools/code_tools.py` | Sandboxed shell execution |
| `tools/deployment_tools.py` | Deployment pipeline triggers |
| `tools/search_tools.py` | Web search |

**Process types:**

| Process | Command | Instances |
|---|---|---|
| API | `uvicorn thinker_agent.main:app` | 2–10 (HPA) |

---

## thinker-mcp

**Repo:** https://github.com/thinker-ci/thinker-mcp
**Technology:** Python 3.12, MCP SDK, httpx, pydantic-settings

**Responsibilities:**
- Expose thinker-ci as a Model Context Protocol server over stdio
- Proxy tool calls to thinker-console and thinker-agent via HTTP
- Expose blueprint docs and the live OpenAPI spec as MCP resources

**Tools (22 total):**

| Group | Tools |
|---|---|
| Tickets | `tickets_list`, `ticket_get`, `ticket_update_status`, `ticket_create` |
| Pipelines | `pipelines_list`, `pipeline_trigger`, `run_get`, `run_logs`, `run_cancel`, `runners_list` |
| PM | `epics_list`, `stories_list`, `tasks_list`, `story_update`, `task_create`, `sprint_board`, `sprints_list`, `milestones_list` |
| Agents | `agent_session_create`, `agent_message_send`, `agent_sessions_list` |

**Resources:**

| URI | Content |
|---|---|
| `thinker://openapi/schema` | Live OpenAPI JSON from thinker-console |
| `thinker://docs/architecture` | Concatenated architecture/*.md |
| `thinker://tickets/{id}` | Individual ticket file from thinker-blueprint |

**Claude Code config (`.claude.json`):**
```json
{
  "mcpServers": {
    "thinker-ci": {
      "command": "uvx",
      "args": ["thinker-mcp"],
      "env": {
        "THINKER_CONSOLE_URL": "https://api.thinker.ci",
        "THINKER_CONSOLE_TOKEN": "...",
        "THINKER_AGENT_URL": "https://agent.thinker.ci",
        "THINKER_BLUEPRINT_PATH": "/path/to/thinker-blueprint"
      }
    }
  }
}
```

---

## thinker-web

**Repo:** https://github.com/thinker-ci/thinker-web
**Technology:** Vanilla JS (ES modules), Tailwind CSS, no framework

**Responsibilities:**
- Provide the primary browser UI for all thinker-ci features
- Hash-based client-side routing with lazy-loaded page modules
- Reactive store for cross-component state (user, tenant, notifications)
- Streaming agent chat via SSE from thinker-agent
- Embedded Swagger UI for the thinker-console OpenAPI schema

**Pages:**

| Route | Page |
|---|---|
| `/dashboard` | Activity summary, run stats |
| `/projects`, `/projects/:slug` | Project list, pipeline + webhook detail |
| `/pipelines`, `/runs/:id` | Run list, live log tail |
| `/runners` | Runner pool list and registration |
| `/pm/board` | Kanban board (story cards by status) |
| `/pm/backlog` | Story backlog grouped by epic |
| `/pm/sprints` | Sprint list + velocity progress |
| `/pm/roadmap` | SVG timeline: milestones + epic bars |
| `/pm/epics` | Epic list with story progress |
| `/pm/stories/:id` | Story detail — tasks checklist, comments |
| `/agents` | Agent session list + streaming chat |
| `/api-docs` | Swagger UI from live OpenAPI schema |
| `/settings` | Workspace, LLM keys, members, security |

---

## thinker-git

**Repo:** https://github.com/thinker-ci/thinker-git
**Technology:** Gitea, FastAPI proxy, nginx, PostgreSQL

**Responsibilities:**
- Provide self-hosted Git hosting as part of the thinker-ci platform
- Isolate tenants via one Gitea organisation per tenant
- Bridge thinker-console token auth into per-user Gitea tokens (SSO)
- Forward Gitea system webhooks to thinker-console for pipeline triggering

**Services:**

| Service | Port | Role |
|---|---|---|
| gitea | 3000 | Git server, web UI, SSH |
| proxy | 8002 | FastAPI — SSO, webhook forwarding, transparent API proxy |
| nginx | 80 | Routes `/api/git/*` → proxy, `/hooks/*` → proxy, `/*` → gitea |
| postgres | 5432 | Gitea database |

**SSO flow:**
```
thinker-web → POST /auth/sso  (X-Thinker-Token: <console token>)
proxy       → GET  thinker-console /api/v1/me/  (validate token)
proxy       → GET/POST gitea /api/v1/admin/users/{username}  (get or create)
proxy       → POST gitea /api/v1/users/{username}/tokens
proxy       ← 200 { "gitea_token": "..." }
thinker-web ← stores gitea_token for Gitea API calls
```

---

## PostgreSQL

**Version:** 15
**Production:** Cloud SQL (GCP) or RDS (AWS), PITR enabled, deletion protection in prod

**Schema highlights:**

| Table | Approx. row growth |
|---|---|
| `tenants`, `tenant_memberships` | Slow |
| `users` | Slow |
| `projects` | Slow |
| `pipelines` | Slow |
| `pipeline_runs` | Medium — one per commit/trigger |
| `jobs` | Medium — one per job per run |
| `job_logs` | High — hundreds to thousands per job |
| `webhook_deliveries` | Medium — one per inbound webhook |
| `pm_*` (epics, stories, tasks…) | Medium |

`job_logs` is the largest table by far. Partition by `job_id` or archive to object storage after 90 days.

---

## Redis

**Version:** 7
**Production:** Memorystore (GCP) or ElastiCache (AWS), STANDARD_HA tier

Used for:
- Celery task broker
- Celery result backend

---

## External Git Providers

GitHub, GitLab, and Bitbucket send HMAC-SHA256 verified webhooks to `POST /hooks/{project_slug}/`. Every delivery is stored as a `WebhookDelivery` audit record regardless of outcome.

Supported event types: `push`, `pull_request`, `tag`, `ping`.

See [webhooks/webhook-handler.md](../webhooks/webhook-handler.md) for provider setup guides.
