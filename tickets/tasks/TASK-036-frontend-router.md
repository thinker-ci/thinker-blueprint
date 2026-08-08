---
id: TASK-036
type: task
title: Frontend client-side router
status: todo
priority: high
story: STORY-019
epic: EPIC-006
repo: thinker-frontend
estimate: 3
assignee: unassigned
sprint: sprint-003
created: 2026-08-08
updated: 2026-08-08
labels: [frontend, router, javascript]
---

# TASK-036: Frontend client-side router

## Description

Implement a lightweight client-side router using the History API. The router maps URL paths to component render functions and handles browser back/forward navigation.

## Acceptance Criteria

- [ ] `router.js` exports `navigate(path)`, `onRouteChange(handler)`, and `routes` map
- [ ] Routes defined: `/` (Dashboard), `/projects` (ProjectList), `/board` (KanbanBoard), `/roadmap` (Roadmap), `/agent` (AgentChat), `/settings` (Settings), `/login` (Login)
- [ ] `navigate(path)` calls `history.pushState` and re-renders the matched component
- [ ] `popstate` event handler re-renders on browser back/forward
- [ ] Clicking `<a data-link href="/...">` intercepts the click and uses `navigate()` instead of a full reload
- [ ] 404 route renders an error component
- [ ] Auth guard: if no JWT in `sessionStorage`, redirect to `/login` before rendering any protected route

## Technical Notes

```js
const routes = [
  { path: '/', component: Dashboard },
  { path: '/login', component: Login, public: true },
]
```

Match routes by exact path first, then by prefix, then fall through to 404. The router must work without a build step (pure ES modules, `import` via `<script type="module">`).
