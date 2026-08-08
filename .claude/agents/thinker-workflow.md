---
name: thinker-workflow
description: Development workflow for thinker-ci — pick a ticket, implement it, commit, PR
---

# Thinker CI Development Workflow

When this skill is loaded, follow these steps to work on the thinker-ci platform:

## Step 1: Find the next ticket

Read `tickets/sprints/sprint-001.md` to find the active sprint.
Then list stories in the sprint and find the highest-priority `todo` or `in-progress` one.
Read the full story file from `tickets/stories/`.

If you already know which ticket to work on (e.g. user said "work on TASK-042"), read that ticket directly.

## Step 2: Understand the context

1. Read the story's parent epic from `tickets/epics/`
2. Read all tasks under this story from `tickets/tasks/` (filter by `story:` field in frontmatter)
3. Identify which repo the work lives in (ticket's `repo:` field)
4. Read the relevant existing code in that repo

## Step 3: Update ticket to in-progress

Edit the ticket's YAML frontmatter: change `status: todo` to `status: in-progress` and update the `updated:` date to today.

## Step 4: Plan and implement

Before writing code:
- Read existing models/views/tests related to the feature
- Check if there are related migrations that need updating
- Identify test cases from the story's Acceptance Criteria

Then implement:
- Write the code following the Technical Notes in the ticket
- Write tests covering all Acceptance Criteria
- Run tests to verify everything passes

## Step 5: Commit

```bash
git add -A
git commit -m "feat(STORY-NNN): brief description of what was done"
```

Commit message format: `type(TICKET-ID): description`
Types: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`

## Step 6: Mark done

Edit the ticket file: `status: done`, update `updated:` to today's date.
If all tasks under a story are done, mark the story `done` too.
If all stories in an epic are done, mark the epic `done` too.

## Definition of Done

- [ ] Code written and working
- [ ] Tests written and passing (`pytest` exits 0)
- [ ] Linting clean (`ruff check .` exits 0)
- [ ] Ticket status updated to `done`
- [ ] Commit created with ticket ID in message
- [ ] No regressions in existing tests

## Priority order

When multiple tickets are `todo`, pick in this order:

1. `critical` priority before `high` before `medium` before `low`
2. Tickets in the active sprint before backlog tickets
3. Earlier story IDs before later ones (lower number = older dependency)
4. If a ticket is marked `blocked`, skip it and note what's blocking it

## Escalating blockers

If blocked, set ticket `status: blocked` and add these fields to the frontmatter:

```yaml
blocked_by: TASK-042
blocked_reason: "waiting for Tenant model to be merged before middleware can be implemented"
```

Then move to the next highest-priority unblocked ticket.
