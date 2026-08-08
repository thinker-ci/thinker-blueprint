---
id: TASK-017
type: task
title: Review / PR workflow model
status: todo
priority: medium
story: STORY-010
epic: EPIC-003
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [backend, pm, review, django]
---

# TASK-017: Review / PR workflow model

## Description

Implement the `Review` model that links a story to a code review (pull request) with status tracking. When a pipeline run is triggered, it can optionally reference a review.

## Acceptance Criteria

- [ ] `Review` model: `id`, `tenant` (FK), `story` (FK Story, nullable), `title`, `pull_request_url` (URLField), `status` (choices: open, approved, changes-requested, merged, closed), `reviewer` (FK User, nullable), `created_by` (FK User), `created_at`, `updated_at`
- [ ] `POST /api/v1/pm/reviews/` creates a review linked to a story
- [ ] `PATCH /api/v1/pm/reviews/{id}/` updates status
- [ ] When review status changes to `merged`, linked story auto-transitions to `done` (via signal)
- [ ] Migration runs cleanly

## Technical Notes

Use a `post_save` signal on `Review` to check for the `merged` transition and update the related story. Guard against infinite loops by checking if the status field actually changed using `update_fields`.
