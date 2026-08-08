---
id: TASK-039
type: task
title: Kanban board component
status: todo
priority: high
story: STORY-020
epic: EPIC-006
repo: thinker-frontend
estimate: 5
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [frontend, kanban, drag-drop, ui]
---

# TASK-039: Kanban board component

## Description

Implement a drag-and-drop kanban board for the active sprint. Stories are displayed in columns by status. Dragging a card to a new column calls the API to update the story's status.

## Acceptance Criteria

- [ ] Six columns: Backlog, To Do, In Progress, Review, Done, Blocked
- [ ] Each card shows: story title, story points badge, assignee avatar, task progress (e.g. "3/5 tasks done")
- [ ] Drag-and-drop using the HTML5 Drag and Drop API (no external library)
- [ ] On drop: call `PATCH /api/v1/pm/stories/{id}/` with `{status: new_column_status}`; update UI optimistically
- [ ] On API failure: revert card to previous column and show a toast error
- [ ] Empty column state: "No stories" placeholder text
- [ ] Data fetched from `GET /api/v1/pm/sprints/current/board/`
- [ ] Board re-fetches when switching to the `/board` route

## Technical Notes

Use `dragstart`, `dragover`, `drop`, and `dragend` events. Store `draggingCardId` in component state. Apply `dragover` highlight CSS to the target column. Implement optimistic update by moving the card in the local data structure before the API call resolves.
