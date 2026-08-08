---
id: TASK-040
type: task
title: Pipeline run detail view with live log streaming
status: todo
priority: medium
story: STORY-021
epic: EPIC-006
repo: thinker-frontend
estimate: 4
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [frontend, pipeline, logs, streaming, ui]
---

# TASK-040: Pipeline run detail view with live log streaming

## Description

Implement the pipeline run detail page (`/runs/{id}`) that shows the jobs in a run, their statuses, and live-streamed log output for the selected job.

## Acceptance Criteria

- [ ] Page fetches run metadata from `GET /api/v1/runs/{id}/` on mount; auto-refreshes every 5s while run is active
- [ ] Job list sidebar: each job shown with name, status badge, and duration
- [ ] Clicking a job fetches logs from `GET /api/v1/jobs/{id}/logs/` and displays them in a scrollable terminal-style pane
- [ ] Logs auto-scroll to bottom as new content arrives
- [ ] "Cancel Run" button calls `POST /api/v1/runs/{id}/cancel/`; disabled when run is not active
- [ ] Status badges use colour: green=success, red=failed, yellow=running, grey=pending
- [ ] Page title updates to include run status: "Run #42 — success"

## Technical Notes

Use `setInterval` for polling run status. For log display, use a `<pre>` with `white-space: pre-wrap; font-family: monospace` styled with Tailwind `font-mono`. Append new log lines to the `<pre>` instead of re-rendering the whole element to preserve scroll position.
