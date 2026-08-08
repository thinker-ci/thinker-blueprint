---
name: ticket
description: Create and manage thinker-ci tickets in the file-system ticketing system
---

# Ticket Management

## Creating a ticket

1. Determine the ticket type: `epic` | `story` | `task` | `bug` | `spike`
2. Find the next available ID by listing files in the relevant directory
3. Create the file at `tickets/{type}s/{ID}-{slug}.md`
4. Fill ALL frontmatter fields
5. Write a clear description, acceptance criteria, and technical notes

## ID format

| Type | Prefix | Example | Directory |
|------|--------|---------|-----------|
| Epic | EPIC | EPIC-011 | `tickets/epics/` |
| Story | STORY | STORY-031 | `tickets/stories/` |
| Task | TASK | TASK-041 | `tickets/tasks/` |
| Bug | BUG | BUG-001 | `tickets/bugs/` |
| Spike | SPIKE | SPIKE-001 | `tickets/spikes/` |

To find the next ID: `ls tickets/tasks/ | sort | tail -1` — extract the number and increment by 1.

## Required frontmatter fields

### Story

```yaml
---
id: STORY-NNN
type: story
title: Short descriptive title
status: backlog
priority: high
epic: EPIC-NNN
repo: thinker-console
estimate: 5
sprint: sprint-001
assignee: unassigned
milestone: M1-foundation
created: 2026-08-08
updated: 2026-08-08
labels: [backend, django]
---
```

### Task

```yaml
---
id: TASK-NNN
type: task
title: Short descriptive title
status: todo
priority: high
story: STORY-NNN
epic: EPIC-NNN
repo: thinker-console
estimate: 3
sprint: sprint-001
assignee: unassigned
created: 2026-08-08
updated: 2026-08-08
labels: [backend, django]
---
```

## Ticket body sections

```markdown
# TASK-NNN: Title

## Description
What is this and why does it need to exist?

## Acceptance Criteria
- [ ] Concrete, testable criterion 1
- [ ] Concrete, testable criterion 2

## Technical Notes
Implementation details, code snippets, constraints.
```

## Linking tickets

When creating a **task**, also:
- Add the task ID to the parent story's `## Tasks` table

When creating a **story**, also:
- Add the story ID to the parent epic's `## Stories` table
- Add the story to the relevant sprint file if it belongs in a sprint

## Status transitions

Valid status values and their meaning:

| Status | Meaning |
|--------|---------|
| `backlog` | Defined but not scheduled |
| `todo` | Scheduled in a sprint, ready to start |
| `in-progress` | Being worked on now |
| `review` | Code written, awaiting review |
| `done` | Merged and deployed |
| `blocked` | Cannot proceed; see `blocked_by` |
| `cancelled` | Will not be done |

## Moving between sprints

To move a ticket to a different sprint:
1. Update the ticket's `sprint:` field
2. Update the old sprint file to remove the story from its story list
3. Update the new sprint file to add the story to its story list
4. Update `updated:` to today's date
