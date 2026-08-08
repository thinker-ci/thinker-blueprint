---
id: TASK-033
type: task
title: MCP pipeline tools
status: todo
priority: high
story: STORY-018
epic: EPIC-005
repo: thinker-mcp
estimate: 2
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [mcp, pipelines, tools]
---

# TASK-033: MCP pipeline tools

## Description

Implement and register the six pipeline tools in `tools/pipelines.py`: `pipelines_list`, `pipeline_trigger`, `run_get`, `run_logs`, `run_cancel`, `runners_list`.

## Acceptance Criteria

- [ ] All six tools implemented using `console_get` / `console_post` helpers
- [ ] `pipeline_trigger` correctly passes `branch` and optional `commit_sha` in the request body
- [ ] `run_cancel` posts to `/api/v1/runs/{id}/cancel/` with an empty body
- [ ] Tools registered in `server.py` with correct JSON Schemas (required vs optional parameters)
- [ ] Integration test using `httpx.MockTransport` to mock the console API
- [ ] Error responses from the API are passed through as text (not re-raised)

## Technical Notes

Use the existing `console_get` and `console_post` helpers from `client.py`. Do not introduce new HTTP client code. The tools are thin wrappers that serialize parameters to URL path / query string / body and return the raw JSON response text.
