---
id: TASK-016
type: task
title: Roadmap model and API
status: todo
priority: medium
story: STORY-008
epic: EPIC-003
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, django]
---

# TASK-016: Roadmap model and API

## Description

Implement a `Roadmap` model that represents a planning view grouping milestones and epics by quarter. The roadmap API provides the data for the frontend roadmap timeline.

## Acceptance Criteria

- [ ] `Roadmap` model: `id`, `tenant` (FK), `name`, `year`, `quarters` (JSONField — list of `{quarter, milestones, epics}`)
- [ ] `GET /api/v1/pm/roadmap/` returns the current tenant's roadmap with nested milestone and epic summaries
- [ ] Roadmap data is computed dynamically from Milestone and Epic models — no manual data entry
- [ ] Response includes `{milestone_id, name, target_date, progress, epics: [...]}`
- [ ] API endpoint documented with `@extend_schema`

## Technical Notes

The roadmap endpoint is a read-only API (`RetrieveAPIView`). Construct the response by querying milestones ordered by `target_date` and prefetching related epics and their story counts.
