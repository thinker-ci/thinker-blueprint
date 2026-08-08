---
id: TASK-018
type: task
title: Sprint board API view
status: todo
priority: high
story: STORY-007
epic: EPIC-003
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, api]
---

# TASK-018: Sprint board API view

## Description

Implement `GET /api/v1/pm/sprints/current/board/` which returns the kanban board for the active sprint, with stories grouped by status column.

## Acceptance Criteria

- [ ] Returns `{sprint: {...}, columns: [{status, label, stories: [{id, title, story_points, assignee, task_count}]}]}`
- [ ] Columns: backlog, todo, in-progress, review, done, blocked
- [ ] Each story entry includes task counts (total / done)
- [ ] Returns 404 when no active sprint exists
- [ ] Response time < 200ms with 50 stories (use `select_related` and `prefetch_related`)
- [ ] Documented with `@extend_schema`

## Technical Notes

Use a single query with `Sprint.current()`, then `prefetch_related("stories__tasks")`. Avoid N+1 queries. Group stories by status in Python rather than separate DB queries per column.
