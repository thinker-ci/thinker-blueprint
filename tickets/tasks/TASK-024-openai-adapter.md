---
id: TASK-024
type: task
title: OpenAI LLM adapter
status: todo
priority: medium
story: STORY-013
epic: EPIC-004
repo: thinker-agent
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, llm, openai]
---

# TASK-024: OpenAI LLM adapter

## Description

Implement `OpenAIAdapter` which wraps `openai>=1.0` async client and implements `BaseLLMAdapter`. Support parallel tool calls and streaming.

## Acceptance Criteria

- [ ] `OpenAIAdapter` in `adapters/openai_adapter.py` implements all `BaseLLMAdapter` methods
- [ ] Supports `gpt-4o`, `gpt-4o-mini`, `o1` model selection
- [ ] Tool use: maps `BaseTool` to OpenAI `function` format; deserialises `tool_calls` in responses
- [ ] Streaming via `AsyncStream` with `delta` accumulation
- [ ] `OPENAI_API_KEY` read from environment
- [ ] Handles parallel tool calls (multiple `tool_calls` in a single response)
- [ ] Conformance test suite passes

## Technical Notes

The OpenAI SDK uses `AsyncOpenAI`. Note that O1 models do not support streaming or system messages — add a guard in `stream()` and `complete()` that falls back to non-streaming for O1 and moves the system message to the first user message.
