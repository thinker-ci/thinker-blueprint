---
id: TASK-002
type: task
title: TenantModel abstract base class and TenantManager
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
labels: [backend, django, manager]
---

# TASK-002: TenantModel abstract base class and TenantManager

## Description

Implement the `TenantModel` abstract base class and the `TenkerManager` that automatically scopes all queries to the current tenant stored in a `ContextVar`.

## Acceptance Criteria

- [ ] `_current_tenant: ContextVar[Tenant | None]` defined in `apps/tenants/context.py`
- [ ] `set_current_tenant(tenant)` and `get_current_tenant()` helpers
- [ ] `TenantManager.get_queryset()` raises `RuntimeError` when no tenant in context
- [ ] `TenantModel` abstract base: adds `tenant = ForeignKey(Tenant, on_delete=CASCADE, db_index=True)` and sets `objects = TenantManager()`
- [ ] Unit test: manager raises when context is empty
- [ ] Unit test: manager filters to correct tenant when context is set

## Technical Notes

```python
# apps/tenants/context.py
from contextvars import ContextVar
_current_tenant: ContextVar = ContextVar("current_tenant", default=None)

# apps/tenants/managers.py
class TenantManager(models.Manager):
    def get_queryset(self):
        tenant = _current_tenant.get()
        if tenant is None:
            raise RuntimeError("No tenant in context — call set_current_tenant() first")
        return super().get_queryset().filter(tenant=tenant)
```
