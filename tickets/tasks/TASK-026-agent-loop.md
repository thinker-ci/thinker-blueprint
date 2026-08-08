---
id: TASK-026
type: task
title: Agent agentic loop with tool-use orchestration
status: todo
priority: high
story: STORY-015
epic: EPIC-004
repo: thinker-agent
estimate: 5
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, agent, loop, tools]
---

# TASK-026: Agent agentic loop with tool-use orchestration

## Description

Implement the core agent execution loop in `agent/loop.py`. The loop calls the LLM, checks for tool calls, executes them, appends results to the message history, and repeats until the LLM produces a final answer or a turn limit is reached.

## Acceptance Criteria

- [ ] `AgentLoop.run(session, user_message) -> CompletionResult` executes the loop
- [ ] Handles multi-turn tool-call chains (up to `max_turns=10`, configurable)
- [ ] Each tool call: look up handler in the tool registry, execute, append `tool_result` message
- [ ] On `stop_reason == "end_turn"` or no tool calls: exit loop and return the final response
- [ ] On `max_turns` exceeded: return a final message explaining the limit was hit
- [ ] Streaming variant: `AgentLoop.stream(session, user_message) -> AsyncIterator[StreamChunk]`
- [ ] Loop state (messages list) persisted to the session after each turn

## Technical Notes

The loop receives a `Session` object that holds the accumulated message history. Append user and assistant messages after each iteration. Tool call execution is synchronous within the async loop (no parallelism needed in v1).
