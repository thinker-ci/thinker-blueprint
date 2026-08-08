---
id: TASK-014
type: task
title: Sprint model and migration
status: todo
priority: high
story: STORY-007
epic: EPIC-003
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, django]
---

# TASK-014: Sprint model and migration

## Description

Implement the `Sprint` model. Sprints have a fixed time-box (start/end dates), planned vs completed velocity, and a status lifecycle.

## Acceptance Criteria

- [ ] `Sprint` model: `id`, `tenant` (FK), `name`, `goal` (TextField), `status` (choices: planning, active, completed), `start_date` (DateField), `end_date` (DateField), `velocity_planned` (PositiveSmallIntegerField, default 0), `velocity_completed` (PositiveSmallIntegerField, default 0), `created_at`, `updated_at`
- [ ] Only one sprint may be in `active` status per tenant (enforced in `clean()`)
- [ ] Migration runs cleanly
- [ ] `Sprint.current()` classmethod returns the active sprint or `None`
- [ ] Admin registered

## Technical Notes

```python
@classmethod
def current(cls):
    return cls.objects.filter(status="active").first()
```

Enforce single active sprint via `UniqueConstraint` with a condition:
```python
UniqueConstraint(fields=["tenant"], condition=Q(status="active"), name="unique_active_sprint_per_tenant")
```
