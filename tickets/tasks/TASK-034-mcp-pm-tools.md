---
id: TASK-034
type: task
title: MCP project-management tools
status: todo
priority: medium
story: STORY-018
epic: EPIC-005
repo: thinker-mcp
estimate: 2
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [mcp, pm, tools]
---

# TASK-034: MCP project-management tools

## Description

Implement and register the eight PM tools in `tools/pm.py`: `epics_list`, `stories_list`, `tasks_list`, `story_update`, `task_create`, `sprint_board`, `sprints_list`, `milestones_list`.

## Acceptance Criteria

- [ ] All eight tools implemented using `console_get`, `console_post`, `console_patch` helpers
- [ ] `story_update` only sends non-empty fields in the PATCH body
- [ ] `epics_list` and `stories_list` pass filter params as query parameters (not body)
- [ ] Tools registered in `server.py` with correct JSON Schemas
- [ ] Integration test using `httpx.MockTransport`
- [ ] All parameters marked optional in the JSON Schema where they genuinely are

## Technical Notes

The `console_get` helper in `client.py` accepts an optional `params` dict for query parameters. Use `params or None` pattern to avoid sending empty dicts. The `console_patch` helper sends a JSON body.
