---
id: TASK-004
type: task
title: Tenant Django admin interface
status: todo
priority: medium
story: STORY-003
epic: EPIC-002
repo: thinker-console
estimate: 2
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, admin, django]
---

# TASK-004: Tenant Django admin interface

## Description

Register `Tenant` and `TenantMembership` in the Django admin with appropriate list displays, filters, and inline views.

## Acceptance Criteria

- [ ] `TenantAdmin`: `list_display = [id, slug, name, created_at]`, `search_fields = [slug, name]`
- [ ] `TenantMembershipInline` (TabularInline) shown inside `TenantAdmin`
- [ ] `TenantMembershipAdmin`: `list_display = [tenant, user, role]`, `list_filter = [role]`
- [ ] Slug field marked read-only in change view to prevent accidental mutation
- [ ] Admin accessible at `/admin/tenants/`

## Technical Notes

```python
@admin.register(Tenant)
class TenantAdmin(admin.ModelAdmin):
    list_display = ["id", "slug", "name", "created_at"]
    search_fields = ["slug", "name"]
    readonly_fields = ["id", "slug", "created_at"]
    inlines = [TenantMembershipInline]
```
