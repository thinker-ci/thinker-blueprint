---
id: TASK-001
type: task
title: Create Tenant Django model and migrations
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
labels: [backend, django, database]
---

# TASK-001: Create Tenant Django model and migrations

## Description

Implement the `Tenant` model in `apps/tenants/models.py`. This is the root object that every other data model will reference via `ForeignKey`.

## Acceptance Criteria

- [ ] `apps/tenants/` Django app created and added to `INSTALLED_APPS`
- [ ] `Tenant` model with: `id` (UUIDField, PK), `slug` (SlugField, unique, immutable), `name` (CharField), `created_at`, `updated_at`
- [ ] `TenantMembership` model with: `tenant` (FK), `user` (FK to AUTH_USER_MODEL), `role` (choices: owner, admin, member)
- [ ] Migration `0001_initial.py` runs cleanly with `migrate`
- [ ] `Tenant` and `TenantMembership` registered in `apps/tenants/admin.py`
- [ ] `__str__` methods return readable strings

## Technical Notes

```python
# apps/tenants/models.py
import uuid
from django.db import models

class Tenant(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    slug = models.SlugField(unique=True, max_length=64)
    name = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"{self.name} ({self.slug})"
```

Slug must be validated as URL-safe and immutable (override `save()` to reject slug changes after creation).
