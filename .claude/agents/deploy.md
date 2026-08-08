---
name: deploy
description: Deployment workflow — trigger pipelines, verify environments, roll back
---

# Deployment Workflow

## Identifying the right pipeline

For a given repo + environment:

1. Call `GET /api/v1/projects/` to find the project (match by slug, e.g. `thinker-console`)
2. Call `GET /api/v1/projects/{slug}/pipelines/` to list pipelines
3. Find the pipeline whose `trigger_events` includes the target event (e.g. `push`) and whose `branch_pattern` matches the branch (e.g. `main`, `release/*`)

The pipeline's `environment` field tells you where it deploys: `dev`, `staging`, or `production`.

## Triggering a deployment

Via the thinker-mcp tool (if using Claude Code with thinker-mcp connected):

```
tool: pipeline_trigger
  pipeline_id: 42
  branch: main
  commit_sha: abc1234  # optional; omit to use HEAD of the branch
```

Via the API directly:

```bash
curl -X POST https://console.thinker.ci/api/v1/pipelines/42/trigger/ \
  -H "Authorization: Token $THINKER_CONSOLE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"branch": "main", "commit_sha": "abc1234"}'
```

## Monitoring a run

Poll `GET /api/v1/runs/{id}/` every 10 seconds until `status` is `success` or `failed`.

```
status values: pending | queued | running | success | failed | cancelled
```

Fetch logs for a failing job: `GET /api/v1/jobs/{job_id}/logs/`

## Environment order

**Always deploy in this order: `dev` → `staging` → `production`**

Never skip staging for a production deploy. If dev is broken, fix it before promoting to staging. If staging is broken, do not promote to production.

Approval required for production: deployment to production is gated on:
- All jobs in the staging run have status `success`
- No open `critical` bugs in the active sprint
- Explicit confirmation from the user before calling `pipeline_trigger` for production

## Rollback

If a production deployment fails or a critical regression is detected:

1. Identify the last successful production run:
   `GET /api/v1/projects/{slug}/pipelines/{id}/runs/?status=success&limit=5`
2. Find the `commit_sha` of the last good run from the run metadata
3. Trigger a new run with that commit SHA:
   `pipeline_trigger(pipeline_id=42, branch="main", commit_sha="<last-good-sha>")`
4. Monitor the rollback run until status is `success`
5. Update the relevant deployment ticket to `status: blocked`:
   ```yaml
   status: blocked
   blocked_reason: "Production rollback to <sha> executed on 2026-09-05 due to <incident>"
   ```
6. Open a `BUG-NNN` ticket describing the regression and link it to the failed deployment

## Pre-deployment checklist

Before triggering any staging or production deployment:

- [ ] `git log --oneline main -10` — confirm expected commits are in the branch
- [ ] `pytest` passes in CI (check the latest dev run)
- [ ] No `critical` or `high` open bugs in `tickets/bugs/` for this repo
- [ ] Migrations: if new migrations exist, confirm they are backward-compatible (no column drops, no NOT NULL without a default)
- [ ] `CHANGELOG.md` updated (for production only)
