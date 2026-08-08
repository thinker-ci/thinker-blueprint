---
id: TASK-030
type: task
title: PM tools for the agent (epics, stories, sprint board)
status: todo
priority: medium
story: STORY-018
epic: EPIC-004
repo: thinker-agent
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, agent, tools, pm]
---

# TASK-030: PM tools for the agent (epics, stories, sprint board)

## Description

Register tools that allow the agent to read and update PM data: listing epics/stories, checking the sprint board, and updating story status.

## Acceptance Criteria

- [ ] `list_epics` tool: calls `GET /api/v1/pm/epics/`
- [ ] `list_stories` tool: calls `GET /api/v1/pm/stories/` with optional `epic` and `sprint` filters
- [ ] `get_sprint_board` tool: calls `GET /api/v1/pm/sprints/current/board/`
- [ ] `update_story_status` tool: calls `PATCH /api/v1/pm/stories/{id}/` to change status
- [ ] `create_task` tool: calls `POST /api/v1/pm/tasks/`
- [ ] Each tool registered in the global tool registry with JSON Schema
- [ ] Unit tests with mocked responses

## Technical Notes

Reuse `ConsoleClient` from TASK-029. These tools share the same base URL and auth token. Keep tool implementations thin — just serialize arguments and call the API.
