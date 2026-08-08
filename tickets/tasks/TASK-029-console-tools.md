---
id: TASK-029
type: task
title: thinker-console API tools for the agent
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
labels: [ai, agent, tools, console]
---

# TASK-029: thinker-console API tools for the agent

## Description

Register a set of tools that allow the agent to interact with thinker-console via its REST API: listing projects, triggering pipelines, and reading pipeline run status.

## Acceptance Criteria

- [ ] `list_projects` tool: calls `GET /api/v1/projects/` on thinker-console
- [ ] `trigger_pipeline` tool: calls `POST /api/v1/pipelines/{id}/trigger/`
- [ ] `get_run_status` tool: calls `GET /api/v1/runs/{id}/`
- [ ] `get_run_logs` tool: calls `GET /api/v1/jobs/{id}/logs/`
- [ ] Each tool registered in the global tool registry with JSON Schema
- [ ] Console API URL and token read from `THINKER_CONSOLE_URL` / `THINKER_CONSOLE_TOKEN` env vars
- [ ] Unit tests with mocked HTTP responses

## Technical Notes

Use `httpx.AsyncClient` for all HTTP calls. Create a `ConsoleClient` class in `clients/console.py` shared by all console tools. Re-raise HTTP errors as `ToolExecutionError` with the response body included.
