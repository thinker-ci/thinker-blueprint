---
id: TASK-011
type: task
title: Epic model and migration
status: todo
priority: high
story: STORY-005
epic: EPIC-003
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, django]
---

# TASK-011: Epic model and migration

## Description

Implement the `Epic` model in `apps/pm/models/epics.py`. Epics are the top-level unit of work in the PM system, grouping related stories toward a milestone.

## Acceptance Criteria

- [ ] `Epic` model: `id`, `tenant` (FK), `title`, `description` (TextField), `status` (choices: backlog, active, done, cancelled), `priority` (choices: low, medium, high, critical), `milestone` (FK nullable), `created_by` (FK User), `created_at`, `updated_at`
- [ ] `EpicManager` extends `TenantManager` to scope queries by tenant
- [ ] Migration `0001_epic.py` runs cleanly
- [ ] `__str__` returns `f"EPIC-{id}: {title}"`
- [ ] Admin registered with list display and filters

## Technical Notes

Inherit from `TenantModel`. Use `TextChoices` for status and priority to get `.label` support in serializers. Index on `(tenant, status)` for efficient list queries.
