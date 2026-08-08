---
id: TASK-023
type: task
title: Gemini (Google) LLM adapter
status: todo
priority: medium
story: STORY-012
epic: EPIC-004
repo: thinker-agent
estimate: 4
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, llm, gemini, google]
---

# TASK-023: Gemini (Google) LLM adapter

## Description

Implement `GeminiAdapter` which wraps `google-generativeai` and implements `BaseLLMAdapter`. Support function calling (Gemini's equivalent of tool use) and streaming.

## Acceptance Criteria

- [ ] `GeminiAdapter` in `adapters/gemini.py` implements all `BaseLLMAdapter` methods
- [ ] Supports `gemini-1.5-pro`, `gemini-1.5-flash`, `gemini-2.0-flash-exp` model selection
- [ ] Maps `BaseTool` definitions to Gemini `FunctionDeclaration` format
- [ ] Streaming via `generate_content_async(stream=True)`
- [ ] `GOOGLE_API_KEY` read from environment
- [ ] Message format conversion: maps `BaseLLMAdapter` message dicts to Gemini `Content` objects
- [ ] Conformance test suite passes

## Technical Notes

Use `google-generativeai>=0.8.0`. Note that Gemini uses `parts` inside `Content` objects, while thinker-agent uses a flat message dict. The adapter is responsible for all format conversion.
