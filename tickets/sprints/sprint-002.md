---
id: sprint-002
title: PM System and AI Agents Sprint
status: planning
start_date: 2026-08-22
end_date: 2026-09-05
goal: Build the full PM data layer and ship the first working AI agent with Claude and Gemini adapters
velocity_planned: 55
velocity_completed: 0
---

# Sprint 002: PM System and AI Agents

## Goal

By the end of this sprint: all PM models (epics, stories, tasks, sprints, milestones) are in the database with APIs and admin. The sprint board and backlog endpoints are live. thinker-agent can run an agentic loop against Claude and Gemini, with memory persisted to Redis.

## Stories in this sprint

| Story | Title | Points | Status |
|-------|-------|--------|--------|
| STORY-005 | Epic and story CRUD | 8 | backlog |
| STORY-006 | Task and subtask management | 5 | backlog |
| STORY-007 | Sprint planning and board | 8 | backlog |
| STORY-008 | Roadmap API | 5 | backlog |
| STORY-009 | Milestone tracking | 5 | backlog |
| STORY-010 | Review/PR workflow | 5 | backlog |
| STORY-011 | Claude LLM adapter | 8 | backlog |
| STORY-012 | Gemini adapter | 8 | backlog |
| STORY-015 | Tool-use agent loop | 8 | backlog |
| STORY-016 | Agent memory | 5 | backlog |

## Tasks in this sprint

- TASK-011: Epic model and migration (STORY-005)
- TASK-012: Story model and migration (STORY-005)
- TASK-013: Task and subtask model (STORY-006)
- TASK-014: Sprint model and migration (STORY-007)
- TASK-015: Milestone model and migration (STORY-009)
- TASK-016: Roadmap model and API (STORY-008)
- TASK-017: Review model (STORY-010)
- TASK-018: Sprint board API view (STORY-007)
- TASK-019: Backlog list API view (STORY-005)
- TASK-020: Activity log (STORY-010)
- TASK-021: BaseLLMAdapter ABC and data classes (STORY-011)
- TASK-022: Claude (Anthropic) adapter (STORY-011)
- TASK-023: Gemini (Google) adapter (STORY-012)
- TASK-024: OpenAI adapter (STORY-013)
- TASK-025: Bedrock adapter (STORY-014)
- TASK-026: Agent agentic loop (STORY-015)
- TASK-027: Agent session memory (STORY-016)
- TASK-028: Tools registry (STORY-015)
- TASK-029: thinker-console API tools for the agent (STORY-018)
- TASK-030: PM tools for the agent (STORY-018)

## Daily Notes

_(notes to be added during sprint execution)_

## Retrospective

_(to be filled at end of sprint on 2026-09-05)_

### What went well

### What to improve

### Action items
