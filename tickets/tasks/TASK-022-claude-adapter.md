---
id: TASK-022
type: task
title: Claude (Anthropic) LLM adapter
status: todo
priority: high
story: STORY-011
epic: EPIC-004
repo: thinker-agent
estimate: 4
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, llm, claude, anthropic]
---

# TASK-022: Claude (Anthropic) LLM adapter

## Description

Implement `ClaudeAdapter` which wraps the Anthropic Python SDK and implements `BaseLLMAdapter`. Support streaming, tool use, and prompt caching for long system prompts.

## Acceptance Criteria

- [ ] `ClaudeAdapter` in `adapters/claude.py` implements all `BaseLLMAdapter` abstract methods
- [ ] Supports model selection via `model` constructor param (default: `claude-opus-4-5`)
- [ ] Tool use: serialises `BaseTool` definitions to Anthropic's `tools` format; deserialises tool call blocks
- [ ] Streaming: yields `StreamChunk` objects via `async for` using the Anthropic `stream()` context manager
- [ ] Prompt caching: adds `{"cache_control": {"type": "ephemeral"}}` to system prompts >1024 tokens
- [ ] Token usage populated in `CompletionResult.usage`
- [ ] `ANTHROPIC_API_KEY` read from environment; missing key raises `ConfigurationError`
- [ ] Conformance test suite passes

## Technical Notes

```python
from anthropic import AsyncAnthropic
class ClaudeAdapter(BaseLLMAdapter):
    def __init__(self, model: str = "claude-opus-4-5"):
        self.client = AsyncAnthropic()
        self.model = model
```
