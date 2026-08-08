---
id: TASK-009
type: task
title: Encrypted field storage for tenant secrets
status: todo
priority: medium
story: STORY-003
epic: EPIC-002
repo: thinker-console
estimate: 3
assignee: unassigned
sprint: sprint-001
created: 2026-08-08
updated: 2026-08-08
labels: [backend, security, encryption]
---

# TASK-009: Encrypted field storage for tenant secrets

## Description

Implement an `EncryptedTextField` custom Django field that transparently encrypts values at write time and decrypts at read time. Used to store OAuth client secrets and API keys scoped to a tenant.

## Acceptance Criteria

- [ ] `EncryptedTextField` subclasses `models.TextField`; overrides `from_db_value` and `get_prep_value`
- [ ] Encryption uses Fernet (AES-128-CBC + HMAC-SHA256) from the `cryptography` package
- [ ] Encryption key read from `FIELD_ENCRYPTION_KEY` env var (base64-encoded 32 bytes)
- [ ] Rotating keys: support a list of keys, decrypt with the first that works, re-encrypt with the primary
- [ ] Unit tests confirm plaintext is never stored in the database
- [ ] Migration guard: field cannot be applied without `FIELD_ENCRYPTION_KEY` set

## Technical Notes

```python
from cryptography.fernet import Fernet, MultiFernet

class EncryptedTextField(models.TextField):
    def from_db_value(self, value, expression, connection):
        return get_fernet().decrypt(value.encode()).decode() if value else value

    def get_prep_value(self, value):
        return get_fernet().encrypt(value.encode()).decode() if value else value
```
