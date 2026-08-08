---
id: TASK-021
type: task
title: BaseLLMAdapter abstract base class and data classes
status: todo
priority: high
story: STORY-011
epic: EPIC-004
repo: thinker-agent
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, llm, adapter, abc]
---

# TASK-021: BaseLLMAdapter abstract base class and data classes

## Description

Define the abstract interface that all LLM provider adapters must implement. Also define the shared data classes (`Message`, `Tool`, `CompletionResult`, `StreamChunk`) used across all adapters.

## Acceptance Criteria

- [ ] `BaseLLMAdapter` ABC in `adapters/base.py` with abstract methods: `complete(messages, tools, **kwargs) -> CompletionResult`, `stream(messages, tools, **kwargs) -> AsyncIterator[StreamChunk]`, `embed(text) -> list[float]`
- [ ] `CompletionResult` dataclass: `content`, `tool_calls`, `usage: UsageStats`, `model`, `stop_reason`
- [ ] `StreamChunk` dataclass: `type` (text/tool_call/end), `content`, `tool_call`
- [ ] `UsageStats` dataclass: `input_tokens`, `output_tokens`, `cache_read`, `cache_write`
- [ ] Type annotations on all methods
- [ ] Docstrings on all classes and methods
- [ ] Conformance test helper `assert_adapter_conforms(adapter)` for use in provider-specific tests

## Technical Notes

Use `@dataclass(frozen=True)` for the data classes to make them immutable. Use `abc.ABC` and `@abstractmethod`. All async methods.
