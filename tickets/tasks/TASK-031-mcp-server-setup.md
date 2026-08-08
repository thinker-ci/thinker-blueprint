---
id: TASK-031
type: task
title: thinker-mcp server setup and stdio transport
status: todo
priority: high
story: STORY-017
epic: EPIC-005
repo: thinker-mcp
estimate: 3
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [mcp, python, server]
---

# TASK-031: thinker-mcp server setup and stdio transport

## Description

Wire up the MCP `Server("thinker-ci")` with all tool and resource handlers, and expose it via the `stdio_server()` transport so MCP clients can connect by spawning the process.

## Acceptance Criteria

- [ ] `server.py` uses `mcp.server.Server` and `mcp.server.stdio.stdio_server`
- [ ] `@server.list_tools()` handler returns all registered tools
- [ ] `@server.call_tool()` handler dispatches to the correct tool function
- [ ] `@server.list_resources()` and `@server.read_resource()` handlers registered
- [ ] `entrypoint()` function calls `asyncio.run(main())`
- [ ] `pyproject.toml` script entry: `thinker-mcp = "thinker_mcp.server:entrypoint"`
- [ ] Server tested with `mcp-inspector` and Claude Code

## Technical Notes

The server communicates over stdin/stdout using the MCP JSON-RPC protocol. All tool and resource modules must be imported at the top of `server.py` so their handlers are available. The `_dispatch` function is a simple `if/elif` chain — keep it readable.
