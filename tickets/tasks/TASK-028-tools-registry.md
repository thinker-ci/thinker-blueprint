---
id: TASK-028
type: task
title: Agent tools registry
status: todo
priority: high
story: STORY-015
epic: EPIC-004
repo: thinker-agent
estimate: 3
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, agent, tools, registry]
---

# TASK-028: Agent tools registry

## Description

Implement a tool registry that maps tool names to callable handlers. Tools can be registered at startup and looked up by the agent loop. Each tool has a JSON Schema definition used to describe it to the LLM.

## Acceptance Criteria

- [ ] `ToolRegistry` class with `register(name, handler, schema)` and `get(name) -> ToolHandler`
- [ ] `BaseTool` dataclass: `name`, `description`, `input_schema` (JSON Schema dict)
- [ ] `ToolHandler` callable type: `async (arguments: dict) -> str`
- [ ] `ToolRegistry.list_tools() -> list[BaseTool]` used by adapters to pass tool definitions to LLMs
- [ ] Tools registered at module import time via a `@registry.tool(schema=...)` decorator
- [ ] Error handling: unknown tool name raises `ToolNotFoundError`; handler exceptions caught and returned as error strings

## Technical Notes

```python
@registry.tool(schema={...})
async def my_tool(arguments: dict) -> str:
    ...
```

A global `registry = ToolRegistry()` in `tools/__init__.py` is imported by the agent loop and all tool modules.
