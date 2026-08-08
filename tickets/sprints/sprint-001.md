---
id: sprint-001
title: Foundation Sprint
status: active
start_date: 2026-08-08
end_date: 2026-08-22
goal: Get the core platform, multi-tenancy, and CI/CD pipeline working end-to-end
velocity_planned: 42
velocity_completed: 0
---

# Sprint 001: Foundation

## Goal

Stand up a working thinker-console Django application with multi-tenancy, authentication (JWT + OAuth), and the first CI pipeline running against the repo. By the end of this sprint an engineer can log in, see their tenant's data, and trigger a pipeline run.

## Stories in this sprint

| Story | Title | Points | Status |
|-------|-------|--------|--------|
| STORY-001 | Django project scaffold and settings management | 8 | todo |
| STORY-002 | Celery worker and task queue setup | 5 | todo |
| STORY-003 | Tenant model and row-level security | 8 | todo |
| STORY-004 | Tenant-aware middleware and request routing | 8 | todo |
| STORY-027 | CI pipeline for thinker-console | 8 | todo |
| STORY-028 | Environment management and secrets | 5 | todo |

## Tasks in this sprint

- TASK-001: Tenant Django model and migrations (STORY-003)
- TASK-002: TenantModel abstract base and TenantManager (STORY-003)
- TASK-003: Tenant API endpoints (STORY-003)
- TASK-004: Tenant Django admin interface (STORY-003)
- TASK-005: Tenant isolation integration tests (STORY-003)
- TASK-006: OAuth SSO integration for tenant login (STORY-004)
- TASK-007: JWT token issuance and refresh endpoints (STORY-004)
- TASK-008: OpenAPI schema generation (STORY-001)
- TASK-009: Encrypted field storage for tenant secrets (STORY-003)
- TASK-010: Development data seeds and fixtures (STORY-001)

## Daily Notes

### 2026-08-08 — Sprint start

Sprint kicked off. STORY-001 (Django scaffold) is the day-one dependency — nothing else can progress until the project exists and docker compose runs.

## Retrospective

_(to be filled at end of sprint on 2026-08-22)_

### What went well

### What to improve

### Action items
