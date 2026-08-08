---
id: TASK-027
type: task
title: Agent session memory and persistence
status: todo
priority: medium
story: STORY-016
epic: EPIC-004
repo: thinker-agent
estimate: 4
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, agent, memory, persistence]
---

# TASK-027: Agent session memory and persistence

## Description

Implement the `Session` model and memory management for agent sessions. Sessions store the full message history in Redis (for fast access during a conversation) and persist to PostgreSQL for long-term retrieval.

## Acceptance Criteria

- [ ] `Session` dataclass: `id` (UUID), `tenant_id`, `provider`, `model_id`, `system_prompt`, `messages: list[Message]`, `created_at`, `updated_at`
- [ ] `SessionStore.save(session)`: serialize messages to JSON and write to Redis with TTL=24h; also persist to Postgres
- [ ] `SessionStore.load(session_id)`: load from Redis first; fall back to Postgres if not in cache
- [ ] Context window management: if message history exceeds `max_tokens` (80% of model's context window), truncate oldest messages while preserving the system prompt and last 5 turns
- [ ] `GET /sessions/` and `GET /sessions/{id}/` REST endpoints
- [ ] Session created by `POST /sessions/`, returned with `id`

## Technical Notes

Use Redis `SETEX` for the cache with a 24-hour TTL. Store messages as a JSON list in a single Redis key `session:{id}:messages`. Postgres model stores a compressed snapshot for long-term storage.
