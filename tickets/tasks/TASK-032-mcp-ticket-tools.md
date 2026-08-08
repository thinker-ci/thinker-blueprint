---
id: TASK-032
type: task
title: MCP ticket tools (file-system read/write)
status: todo
priority: high
story: STORY-018
epic: EPIC-005
repo: thinker-mcp
estimate: 3
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [mcp, tickets, tools]
---

# TASK-032: MCP ticket tools (file-system read/write)

## Description

Verify and extend the existing `tools/tickets.py` to ensure all four tools (`tickets_list`, `ticket_get`, `ticket_update_status`, `ticket_create`) are production-ready, including robust frontmatter parsing and error handling.

## Acceptance Criteria

- [ ] `tickets_list` returns correctly filtered results for all filter combinations
- [ ] `ticket_get` returns the full markdown including frontmatter; 404-style JSON on missing ticket
- [ ] `ticket_update_status` atomically rewrites only the `status:` line; does not corrupt other frontmatter
- [ ] `ticket_create` generates unique IDs, creates the correct directory, and writes valid frontmatter
- [ ] All tools handle missing `THINKER_BLUEPRINT_PATH` gracefully with a clear error message
- [ ] Unit tests using a temp directory with fixture ticket files
- [ ] Tools registered in `server.py` with correct JSON Schemas

## Technical Notes

The frontmatter parser in `tickets.py` is a simple line-by-line parser — consider switching to `python-frontmatter` (already in dependencies) for robustness. Re-use the existing `_parse_frontmatter` helper if it passes all test cases.
