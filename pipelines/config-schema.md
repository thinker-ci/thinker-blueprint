# Pipeline Config Schema

`Pipeline.config` is a JSON document that defines the full build/test/deploy workflow
for a project. This document covers every field, the validation rules applied to it,
how errors are reported at each layer, and a set of annotated examples.

---

## Schema overview

```
{
  "image":  <string>          — optional default Docker image for all jobs
  "stages": [                 — required; at least one stage
    {
      "name": <string>        — required; [a-zA-Z0-9_-] only
      "jobs": [               — required; at least one job
        {
          "name":        <string>   — required; [a-zA-Z0-9_-] only
          "image":       <string>   — optional (overrides top-level image)
          "tags":        [<string>] — optional; runner tag selectors
          "steps":       [<string>] — required; at least one command
          "environment": {<string>: <string>}  — optional env vars
          "timeout":     <integer>  — optional; 1–86400 seconds
          "allow_failure": <bool>   — optional; default false
        }
      ]
    }
  ]
}
```

Thinker CI validates configs using JSON Schema (Draft 7) plus a set of semantic
checks that the schema alone cannot express. See the [Validation rules](#validation-rules)
section for the full rule set.

---

## Field reference

### Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | string | No | Default Docker image inherited by every job that omits its own `image`. If absent, every job must supply its own `image`. |
| `stages` | array | **Yes** | Ordered list of stages. Stages run sequentially. Must contain at least one entry. |

### Stage fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | **Yes** | Stage identifier. Must match `^[a-zA-Z0-9_-]+$`. Must be unique within the config. |
| `jobs` | array | **Yes** | Jobs in this stage. Jobs within the same stage run concurrently (when enough runners are available). Must contain at least one entry. |

### Job fields

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `name` | string | **Yes** | `^[a-zA-Z0-9_-]+$`, unique within stage | Job identifier. Used as the Celery task key. |
| `image` | string | Conditional | Non-empty | Docker image for this job. Required if no top-level `image` is set. Overrides the top-level `image` when both are present. |
| `tags` | array of strings | No | Items non-empty, unique | Runner tag selectors. A job is only dispatched to a runner whose tags are a superset of these values. Empty list = any runner. |
| `steps` | array of strings | **Yes** | At least one item, items non-empty | Shell commands run in order inside the job container. A non-zero exit code from any step fails the job unless `allow_failure` is `true`. |
| `environment` | object | No | Values must be strings | Environment variables injected into the container. Keys and values are arbitrary strings. |
| `timeout` | integer | No | 1–86400 | Job-level timeout in seconds. If the job exceeds this, the runner kills the container and marks the job failed. Default: runner-level timeout. |
| `allow_failure` | boolean | No | — | If `true`, a non-zero exit code is recorded but does not propagate a failure status to the PipelineRun. Default: `false`. |

---

## Validation rules

Validation runs in two layers. Both layers are checked on every save; the second only
runs if the first passes (semantic checks assume a structurally correct config).

### Layer 1: JSON Schema (structural)

| Rule | Error path example | Error message |
|---|---|---|
| `config` must be a JSON object | `config` | `config must be a JSON object` |
| `stages` is required | `config` | `'stages' is a required property` |
| `stages` must have at least one entry | `stages` | `[] is too short` |
| Stage `name` is required | `stages[0]` | `'name' is a required property` |
| Stage `name` must match `^[a-zA-Z0-9_-]+$` | `stages[0].name` | `'my stage' does not match …` |
| Stage `jobs` is required | `stages[0]` | `'jobs' is a required property` |
| Stage `jobs` must have at least one entry | `stages[0].jobs` | `[] is too short` |
| Job `name` is required | `stages[0].jobs[0]` | `'name' is a required property` |
| Job `name` must match `^[a-zA-Z0-9_-]+$` | `stages[0].jobs[0].name` | `'my job' does not match …` |
| Job `steps` is required | `stages[0].jobs[0]` | `'steps' is a required property` |
| Job `steps` must have at least one entry | `stages[0].jobs[0].steps` | `[] is too short` |
| `timeout` must be 1–86400 | `stages[0].jobs[0].timeout` | `86401 is greater than the maximum …` |
| `environment` values must be strings | `stages[0].jobs[0].environment` | `8080 is not of type 'string'` |
| No extra top-level keys | `config` | `Additional properties are not allowed ('x' was unexpected)` |
| No extra job keys | `stages[0].jobs[0]` | `Additional properties are not allowed ('x' was unexpected)` |

All errors from this layer are collected and reported together — validation does not
stop at the first problem.

### Layer 2: Semantic checks

These run after the schema passes. They catch problems the schema cannot express:

| Rule | Error message |
|---|---|
| Stage names must be unique within the config | `stages[1].name: duplicate stage name 'build'` |
| Job names must be unique within a stage | `stages[0].jobs[1].name: duplicate job name 'test' in stage 'build'` |
| Each job must have an `image` or the top-level `image` must be set | `stages[0].jobs[0]: 'image' is required on the job or at the top level` |

---

## Where validation runs

Validation is enforced at three points. A config that was saved with bad data (e.g.
created before the validator existed or via direct DB manipulation) is caught at
dispatch time.

### 1. API create/update — `PipelineSerializer`

`POST /api/v1/projects/<slug>/pipelines/` and
`PATCH /api/v1/pipelines/<id>/` run the validator in the `config` field's serializer
validator. On failure the API returns `400` with `config` containing the error list:

```json
HTTP 400 Bad Request
{
  "config": [
    "stages[0].jobs[0].steps: [] is too short",
    "stages[0].jobs[0]: 'image' is required on the job or at the top level"
  ]
}
```

The response is always an array even when there is only one error, so clients can
iterate it uniformly.

### 2. Admin / full_clean() — `Pipeline.clean()`

The Django admin and any code that calls `pipeline.full_clean()` triggers validation via
the model's `clean()` method. The Django admin displays each error string as a separate
bullet under the `config` field.

### 3. Task dispatch — `dispatch_pipeline_run`

Before creating any jobs, `dispatch_pipeline_run` re-validates the config. If it fails:

- `PipelineRun.status` → `failed`
- `PipelineRun.finished_at` → set to now
- `PipelineRun.error_message` → bulleted list of all validation errors
- No `Job` records are created

The run appears in the API as:

```json
{
  "id": 42,
  "status": "failed",
  "error_message": "Config validation failed:\n  • stages[0].jobs[0].steps: [] is too short\n  • stages[0].jobs[0]: 'image' is required on the job or at the top level",
  "jobs": []
}
```

---

## Error path format

Error paths use a consistent notation across all validation layers:

| Path | Meaning |
|---|---|
| `config` | Problem at the root of the config object |
| `stages` | Problem with the stages array itself |
| `stages[0]` | Problem with the first stage object |
| `stages[0].name` | Problem with the first stage's `name` field |
| `stages[0].jobs[1]` | Problem with the second job in the first stage |
| `stages[0].jobs[1].steps` | Problem with that job's `steps` field |

---

## Full example

A complete multi-stage pipeline config:

```json
{
  "image": "python:3.12-slim",
  "stages": [
    {
      "name": "build",
      "jobs": [
        {
          "name": "install-deps",
          "steps": [
            "pip install -r requirements/base.txt",
            "pip install -r requirements/development.txt"
          ]
        },
        {
          "name": "lint",
          "steps": ["ruff check .", "mypy apps/"]
        }
      ]
    },
    {
      "name": "test",
      "jobs": [
        {
          "name": "pytest",
          "steps": ["pytest --tb=short -q"],
          "environment": {
            "DJANGO_SETTINGS_MODULE": "console.settings",
            "SECRET_KEY": "test-secret"
          },
          "timeout": 300
        }
      ]
    },
    {
      "name": "deploy",
      "jobs": [
        {
          "name": "push-image",
          "image": "docker:24-dind",
          "tags": ["docker"],
          "steps": [
            "docker build -t myapp:$COMMIT_SHA .",
            "docker push myapp:$COMMIT_SHA"
          ],
          "environment": {
            "DOCKER_BUILDKIT": "1"
          },
          "timeout": 600,
          "allow_failure": false
        }
      ]
    }
  ]
}
```

### Minimal valid config (single job)

```json
{
  "stages": [
    {
      "name": "test",
      "jobs": [
        {
          "name": "run-tests",
          "image": "node:20-slim",
          "steps": ["npm ci", "npm test"]
        }
      ]
    }
  ]
}
```

---

## Common mistakes

### Missing `stages` key

```json
{ "image": "python:3.12" }
```
Error: `config: 'stages' is a required property`

### Empty `stages` array

```json
{ "stages": [] }
```
Error: `stages: [] is too short`

### Empty `steps` array

```json
{
  "image": "python:3.12",
  "stages": [{ "name": "test", "jobs": [{ "name": "pytest", "steps": [] }] }]
}
```
Error: `stages[0].jobs[0].steps: [] is too short`

### No image anywhere

```json
{
  "stages": [{ "name": "test", "jobs": [{ "name": "pytest", "steps": ["pytest"] }] }]
}
```
Error: `stages[0].jobs[0]: 'image' is required on the job or at the top level`

### Duplicate stage names

```json
{
  "image": "python:3.12",
  "stages": [
    { "name": "test", "jobs": [{ "name": "a", "steps": ["x"] }] },
    { "name": "test", "jobs": [{ "name": "b", "steps": ["y"] }] }
  ]
}
```
Error: `stages[1].name: duplicate stage name 'test'`

### Invalid characters in name

```json
{
  "image": "python:3.12",
  "stages": [{ "name": "my stage", "jobs": [{ "name": "j", "steps": ["x"] }] }]
}
```
Error: `stages[0].name: 'my stage' does not match '^[a-zA-Z0-9_-]+$'`

### Non-string environment value

```json
{
  "image": "python:3.12",
  "stages": [{
    "name": "test",
    "jobs": [{
      "name": "j",
      "steps": ["x"],
      "environment": { "PORT": 8080 }
    }]
  }]
}
```
Error: `stages[0].jobs[0].environment.PORT: 8080 is not of type 'string'`
(Use `"PORT": "8080"` — all environment values must be strings.)

---

## Extending the schema

The schema is defined in `apps/pipelines/config_schema.py` in the `thinker-console`
repository. The `_PIPELINE_SCHEMA` dict is the authoritative JSON Schema; `_semantic_errors`
holds the cross-field rules. To add a new job field:

1. Add it to `_PIPELINE_SCHEMA["properties"]["stages"]["items"]["properties"]["jobs"]["items"]["properties"]`
2. If the field is required, add it to the `"required"` list on the job object
3. Add a test in `apps/pipelines/tests.py` — both a valid case and an invalid case
4. Update this document
