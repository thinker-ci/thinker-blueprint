---
id: TASK-037
type: task
title: Login page and JWT auth flow
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
labels: [frontend, auth, jwt, login]
---

# TASK-037: Login page and JWT auth flow

## Description

Implement the login page component with email/password form, JWT storage, auth guard, and logout functionality.

## Acceptance Criteria

- [ ] Login page renders email + password form with a submit button
- [ ] Form `POST`s to `THINKER_CONSOLE_URL/api/v1/auth/tokens/` via `fetch()`
- [ ] On success: stores `access` and `refresh` tokens in `sessionStorage`; redirects to `/`
- [ ] On failure: shows inline error message (invalid credentials / server error)
- [ ] Logout button calls `sessionStorage.clear()` and navigates to `/login`
- [ ] Auth guard in router checks `sessionStorage.getItem("access_token")` before rendering protected pages
- [ ] Token refresh: if a 401 is received on any API call, attempt to refresh using the `refresh` token; on failure, redirect to `/login`
- [ ] "Login with Google" button (links to `/api/v1/auth/oauth/google/` — backend redirect)

## Technical Notes

Store tokens as `sessionStorage.setItem("access_token", data.access)`. Use a global `apiFetch(url, options)` wrapper that adds the `Authorization: Bearer {token}` header and handles token refresh automatically.
