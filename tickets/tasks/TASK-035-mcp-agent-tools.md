---
id: TASK-035
type: task
title: MCP agent session tools
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
labels: [mcp, agent, tools]
---

# TASK-035: MCP agent session tools

## Description

Implement and register the three agent tools in `tools/agents.py`: `agent_session_create`, `agent_message_send`, `agent_sessions_list`.

## Acceptance Criteria

- [ ] `agent_session_create` posts to `THINKER_AGENT_URL/sessions` with provider, model_id, system_prompt
- [ ] `agent_message_send` posts to `THINKER_AGENT_URL/sessions/{id}/messages`; returns the full assistant response text
- [ ] `agent_sessions_list` gets `THINKER_AGENT_URL/sessions`
- [ ] Tools registered in `server.py` with correct JSON Schemas
- [ ] `agent_client()` from `client.py` used for all requests (Bearer auth, 60s timeout)
- [ ] Integration test using `httpx.MockTransport`

## Technical Notes

The agent API uses Bearer token authentication (`Authorization: Bearer {token}`) unlike the console API which uses Django Token auth. The `agent_client()` helper already handles this. Use `agent_post` and `agent_get` from `client.py`.
