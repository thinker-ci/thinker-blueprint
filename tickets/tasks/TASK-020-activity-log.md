---
id: TASK-020
type: task
title: Activity log for PM object changes
status: todo
priority: low
story: STORY-010
epic: EPIC-003
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, audit, django]
---

# TASK-020: Activity log for PM object changes

## Description

Implement an `ActivityLog` model that records every meaningful change to PM objects (stories, tasks, sprints) — status changes, assignee changes, story-point updates. Provides the audit trail visible in the UI.

## Acceptance Criteria

- [ ] `ActivityLog` model: `id`, `tenant` (FK), `actor` (FK User), `verb` (CharField, e.g. "status_changed"), `object_type` (ContentType FK), `object_id`, `data` (JSONField — before/after values), `created_at`
- [ ] `post_save` signals on `Story` and `Task` write activity entries when key fields change
- [ ] `GET /api/v1/pm/activity/?object_type=story&object_id=42` returns recent log entries
- [ ] Log entries paginated (50 per page)
- [ ] Old entries pruned after 90 days via Celery periodic task

## Technical Notes

Use `django.contrib.contenttypes.fields.GenericForeignKey` for `object_type` + `object_id` to keep the table generic. Store the delta in `data = {"field": "status", "from": "todo", "to": "in-progress"}`.
