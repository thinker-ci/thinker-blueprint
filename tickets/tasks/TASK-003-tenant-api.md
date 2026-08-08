---
id: TASK-003
type: task
title: Tenant API endpoints (CRUD)
status: todo
priority: high
story: STORY-003
epic: EPIC-002
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, api, drf]
---

# TASK-003: Tenant API endpoints (CRUD)

## Description

Implement Django REST Framework serializers, viewsets, and URL routing for the `Tenant` resource.

## Acceptance Criteria

- [ ] `TenantSerializer` with all fields; `slug` read-only after creation
- [ ] `TenantViewSet` (ModelViewSet): list, retrieve, create, update, partial_update — no destroy
- [ ] `POST /api/v1/tenants/` restricted to superusers (`IsAdminUser`)
- [ ] `GET /api/v1/tenants/{slug}/` accessible to tenant members
- [ ] `TenantMembershipSerializer` and `TenantMembershipViewSet`
- [ ] OpenAPI schema annotations (`@extend_schema`) on all actions
- [ ] Tests: create tenant (admin only), retrieve, list, member access control

## Technical Notes

Use `drf-spectacular` for OpenAPI. Router prefix: `api/v1/tenants/`. Use `lookup_field = "slug"` on the viewset.
