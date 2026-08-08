---
name: sprint
description: Sprint planning, execution tracking, and retrospectives for thinker-ci
---

# Sprint Management

## Reading the current sprint

Read `tickets/sprints/sprint-001.md` (replace with the current sprint number). It contains:
- `status`: planning | active | completed
- `velocity_planned`: sum of story points scheduled for this sprint
- `velocity_completed`: sum of story points actually done
- `goal`: one-paragraph description of what success looks like
- Story list with current statuses

## Sprint planning

1. Identify the next sprint number (look at the latest file in `tickets/sprints/`)
2. Read all stories with `status: backlog` from `tickets/stories/`
3. Sort by priority: `critical` > `high` > `medium` > `low`
4. Estimate total story points to hit the sprint goal (use previous sprint's `velocity_completed` as the cap)
5. For each selected story:
   - Update story frontmatter: `sprint: sprint-NNN`, `status: todo`
   - Add story ID to the sprint file's story list
6. Update `velocity_planned` in the sprint frontmatter with the sum of selected story points
7. Set sprint `status: active` when the sprint starts

## Checking sprint health

For each story in the sprint:
```
- status: todo / in-progress / review / done / blocked
- tasks: count total, done, in-progress, blocked
- story_points: sum contribution to velocity
```

Flag as unhealthy if:
- More than 2 stories are `blocked`
- Less than 30% of story points are `done` at the sprint midpoint
- Any story has been `in-progress` for more than 5 days without a commit

## Velocity calculation

```
velocity_completed = sum(story_points for story in sprint if story.status == "done")
```

Update `velocity_completed` in the sprint file after each story is marked done.

## Closing a sprint

1. Mark all `done` stories — verify `status: done` in each story file
2. For stories still not done:
   - Move to next sprint: update `sprint: sprint-NNN+1` in the story file
   - Remove from current sprint's story list
   - Add to next sprint's story list
3. Calculate final velocity: sum of completed story points
4. Update sprint file:
   ```yaml
   status: completed
   velocity_completed: 34
   ```
5. Write retrospective notes in the `## Retrospective` section:
   - What went well (≥3 items)
   - What to improve (≥2 items)
   - Action items (specific, assigned, with a due date)

## Sprint file format

```markdown
---
id: sprint-002
title: PM System and AI Agents Sprint
status: active
start_date: 2026-08-22
end_date: 2026-09-05
goal: ...
velocity_planned: 55
velocity_completed: 0
---

# Sprint 002: PM System and AI Agents

## Goal
...

## Stories in this sprint
| Story | Title | Points | Status |
...

## Daily Notes
### 2026-08-22
...

## Retrospective
_(filled at sprint close)_
```
