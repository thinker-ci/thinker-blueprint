---
id: TASK-007
type: task
title: JWT token issuance and refresh endpoints
status: todo
priority: high
story: STORY-004
epic: EPIC-002
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, jwt, auth]
---

# TASK-007: JWT token issuance and refresh endpoints

## Description

Implement email/password login and JWT token refresh using `djangorestframework-simplejwt`. The JWT payload must include the `tenant_id` claim so middleware can resolve the tenant without a database round-trip.

## Acceptance Criteria

- [ ] `POST /api/v1/auth/tokens/` — email + password → `{access, refresh}` token pair
- [ ] `POST /api/v1/auth/tokens/refresh/` — refresh token → new access token
- [ ] Access token payload includes: `user_id`, `tenant_id`, `role`, `exp`
- [ ] `TenantMiddleware` reads `tenant_id` from the JWT and sets `_current_tenant` in the ContextVar
- [ ] 401 returned when JWT is missing, expired, or tampered with
- [ ] Token expiry: access = 15 min, refresh = 7 days (configurable via settings)

## Technical Notes

Configure `SIMPLE_JWT` in settings:
```python
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
}
```

Create a custom `TokenObtainPairSerializer` that injects `tenant_id` and `role` into the payload.
