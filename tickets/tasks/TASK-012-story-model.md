---
id: TASK-012
type: task
title: Story model and migration
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

# TASK-012: Story model and migration

## Description

Implement the `Story` model in `apps/pm/models/stories.py`. Stories represent individual units of work within an epic, tracked in sprints with story-point estimates.

## Acceptance Criteria

- [ ] `Story` model: `id`, `tenant` (FK), `epic` (FK Epic, nullable), `title`, `description`, `status` (choices: backlog, todo, in-progress, review, done, blocked), `priority`, `story_points` (PositiveSmallIntegerField, nullable), `sprint` (FK Sprint, nullable), `assignee` (FK User, nullable), `created_at`, `updated_at`
- [ ] Migration runs cleanly
- [ ] Ordering: `["-created_at"]` by default
- [ ] Admin registered with `list_filter = ["status", "priority", "sprint"]`
- [ ] `StorySerializer` exposes all fields with proper read-only set

## Technical Notes

Use `TenantModel` as the base class. The `sprint` FK should use `related_name="stories"` for reverse lookups from the sprint board view.
