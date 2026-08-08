---
id: TASK-005
type: task
title: Tenant isolation integration tests
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
labels: [backend, testing, integration]
---

# TASK-005: Tenant isolation integration tests

## Description

Write integration tests that verify data from one tenant cannot be accessed from another tenant's context, both at the queryset layer and through the API.

## Acceptance Criteria

- [ ] Test fixture creates two tenants (A and B) and one user in each
- [ ] Queryset test: objects created under tenant A are invisible when context is set to tenant B
- [ ] API test: user from tenant A gets 404 on a resource belonging to tenant B
- [ ] Test for `RuntimeError` when no tenant is set in context
- [ ] Tests run with `pytest -m tenant_isolation` marker
- [ ] All tests pass in CI

## Technical Notes

Use `pytest-django` with `@pytest.fixture` for tenant setup. Use `contextlib.contextmanager` to swap `_current_tenant` within a test. Reset the ContextVar in a `finally` block to avoid test pollution.
