---
id: TASK-013
type: task
title: Task and subtask model
status: todo
priority: high
story: STORY-006
epic: EPIC-003
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, django]
---

# TASK-013: Task and subtask model

## Description

Implement the `Task` model with optional self-referential subtask support (parent FK). Tasks belong to stories and represent concrete implementation units with hour estimates.

## Acceptance Criteria

- [ ] `Task` model: `id`, `tenant` (FK), `story` (FK Story), `parent` (FK self, nullable — for subtasks), `title`, `description`, `task_type` (choices: task, bug, spike), `status`, `estimate` (hours, PositiveSmallIntegerField), `assignee` (FK User, nullable), `created_at`, `updated_at`
- [ ] `Task.objects.filter(parent=None)` returns top-level tasks only
- [ ] Migration runs cleanly
- [ ] `TaskSerializer` with nested `subtasks` list (read-only)
- [ ] Admin registered

## Technical Notes

Use `parent = models.ForeignKey("self", null=True, blank=True, on_delete=models.SET_NULL, related_name="subtasks")`. Keep nesting to one level deep — no recursive subtask queries needed.
