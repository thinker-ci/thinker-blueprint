---
id: TASK-006
type: task
title: OAuth SSO integration for tenant login
status: todo
priority: high
story: STORY-004
epic: EPIC-002
repo: thinker-console
estimate: 5
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, oauth, sso, auth]
---

# TASK-006: OAuth SSO integration for tenant login

## Description

Integrate Google OAuth 2.0 (and optionally GitHub) as a login provider. On successful OAuth callback, resolve or create the tenant-scoped user and issue a JWT.

## Acceptance Criteria

- [ ] `GET /api/v1/auth/oauth/google/` redirects to Google consent screen
- [ ] `GET /api/v1/auth/oauth/google/callback/` handles the authorization code, fetches user info, creates or links the user, and returns a JWT pair
- [ ] OAuth state parameter used to prevent CSRF
- [ ] On first login, user is associated with a tenant derived from their email domain (or an invite link)
- [ ] `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` read from environment
- [ ] Integration test using mocked Google responses

## Technical Notes

Use `authlib` or `python-social-auth` for the OAuth flow. Store the OAuth provider and provider user ID on the user model to handle re-logins without creating duplicate accounts.
