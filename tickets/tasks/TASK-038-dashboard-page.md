---
id: TASK-038
type: task
title: Dashboard home page
status: todo
priority: medium
story: STORY-019
epic: EPIC-006
repo: thinker-frontend
estimate: 3
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [frontend, dashboard, ui]
---

# TASK-038: Dashboard home page

## Description

Implement the dashboard home page (`/`) showing a high-level summary: active sprint progress, recent pipeline runs, and a shortcut to the kanban board.

## Acceptance Criteria

- [ ] Sprint progress card: shows sprint name, goal, and a progress bar (completed story points / planned)
- [ ] Recent runs widget: shows last 5 pipeline runs with status badge (success/failed/running), branch, and elapsed time; links to the run detail view
- [ ] Quick-action buttons: "View Board", "New Story", "Trigger Pipeline"
- [ ] All data fetched from thinker-console API on mount: `GET /api/v1/pm/sprints/current/board/` and `GET /api/v1/runs/?limit=5`
- [ ] Loading skeleton shown while data is fetching
- [ ] Error state shown if API calls fail

## Technical Notes

Implement as a plain JS component in `components/Dashboard.js`. Mount by replacing `document.getElementById("app").innerHTML`. Use `async/await` with `apiFetch()` for API calls. Apply Tailwind utility classes for layout.
