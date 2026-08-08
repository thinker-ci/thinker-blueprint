---
id: TASK-008
type: task
title: OpenAPI schema generation and documentation endpoint
status: todo
priority: medium
story: STORY-001
epic: EPIC-001
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, openapi, docs]
---

# TASK-008: OpenAPI schema generation and documentation endpoint

## Description

Configure `drf-spectacular` to auto-generate an OpenAPI 3.x schema from all DRF viewsets. Expose the schema and a Swagger UI at predictable URLs.

## Acceptance Criteria

- [ ] `GET /api/schema/?format=json` returns a valid OpenAPI 3.x schema (used by thinker-mcp)
- [ ] `GET /api/schema/?format=yaml` returns YAML format
- [ ] `GET /api/docs/` renders Swagger UI
- [ ] `GET /api/redoc/` renders ReDoc
- [ ] Schema includes security scheme: `bearerAuth` (JWT)
- [ ] All viewsets have `@extend_schema` decorators with descriptions and response examples
- [ ] Schema validates with `spectral lint` in CI

## Technical Notes

```python
# settings/base.py
SPECTACULAR_SETTINGS = {
    "TITLE": "thinker-console API",
    "VERSION": "0.1.0",
    "SERVE_INCLUDE_SCHEMA": False,
    "SECURITY": [{"bearerAuth": []}],
}
```

Add `drf-spectacular` to dependencies and wire up `SpectacularAPIView`, `SpectacularSwaggerView`, `SpectacularRedocView`.
