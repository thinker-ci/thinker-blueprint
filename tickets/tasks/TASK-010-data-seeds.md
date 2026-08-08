---
id: TASK-010
type: task
title: Development data seeds and fixtures
status: todo
priority: low
story: STORY-001
epic: EPIC-001
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, fixtures, devx]
---

# TASK-010: Development data seeds and fixtures

## Description

Create a `manage.py seed` management command that populates the database with realistic development data (tenants, users, projects, pipelines) so engineers can start with a working local environment immediately.

## Acceptance Criteria

- [ ] `python manage.py seed` creates: 2 tenants, 4 users (one owner + members per tenant), 2 projects with pipelines
- [ ] Command is idempotent (safe to run multiple times)
- [ ] Output shows what was created or skipped
- [ ] `pytest` fixture `db_seed` exposes the seed data for integration tests
- [ ] `docker compose up` docs updated to mention `seed` command

## Technical Notes

Use `get_or_create` for idempotency. Place the command at `apps/core/management/commands/seed.py`. Seed data should use deterministic UUIDs and slugs (e.g., `tenant-alpha`, `tenant-beta`) so tests can reference them by name.
