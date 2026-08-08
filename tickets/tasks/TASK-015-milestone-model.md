---
id: TASK-015
type: task
title: Milestone model and migration
status: todo
priority: medium
story: STORY-009
epic: EPIC-003
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, django]
---

# TASK-015: Milestone model and migration

## Description

Implement the `Milestone` model. Milestones represent major deliverable checkpoints with target dates, tracking the aggregate progress of the epics that belong to them.

## Acceptance Criteria

- [ ] `Milestone` model: `id`, `tenant` (FK), `name`, `slug` (unique per tenant), `description`, `target_date` (DateField), `status` (choices: planned, in-progress, achieved, cancelled), `created_at`, `updated_at`
- [ ] `Milestone.progress` property: (done_story_points / total_story_points) × 100, returns 0 when no stories
- [ ] Migration runs cleanly
- [ ] `MilestoneSerializer` includes `progress` as a computed field
- [ ] Admin registered with progress shown in list display

## Technical Notes

The `progress` computation requires a DB aggregation query over related epics and their stories. Implement as a property on the model that calls `.aggregate(Sum("story_points"))` on the related story queryset.
