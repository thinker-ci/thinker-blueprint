---
id: TASK-019
type: task
title: Backlog list API view with filtering and ordering
status: todo
priority: medium
story: STORY-005
epic: EPIC-003
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, api]
---

# TASK-019: Backlog list API view with filtering and ordering

## Description

Implement `GET /api/v1/pm/backlog/` which returns all unscheduled stories (those with `sprint=None`) ordered by priority and creation date, with filtering by epic and status.

## Acceptance Criteria

- [ ] `GET /api/v1/pm/backlog/` returns stories where `sprint` is null
- [ ] Query params: `epic`, `status`, `priority`, `assignee`
- [ ] Default ordering: `priority desc, created_at asc`
- [ ] Paginated: 25 items per page with `next`/`previous` links
- [ ] Each story includes: `id`, `title`, `status`, `priority`, `story_points`, `epic_id`, `epic_title`, `assignee`
- [ ] Documented with `@extend_schema`

## Technical Notes

Use `django-filter` (`FilterSet`) for the query params. Use `OrderingFilter` or `?ordering=priority` for sort control. Use `select_related("epic", "assignee")` to avoid N+1.
