# Data Flow

## Webhook-triggered pipeline run

```
1.  Developer pushes a commit to GitHub / GitLab / Bitbucket

2.  Git provider sends:  POST /hooks/{project_slug}/

3.  WebhookView (10-step pipeline):
      a. Look up Project by slug — 404 if not found
      b. Read raw body bytes before any parsing
      c. Store a WebhookDelivery audit record (status=received)
      d. Detect provider from project.provider
      e. Verify HMAC-SHA256 signature (GitHub, Bitbucket)
         or plain token (GitLab) — 403 + audit record on failure
      f. Check idempotency: if delivery_id already processed, return
         200 with cached run_ids
      g. Parse provider-specific headers + payload into a normalised
         WebhookEvent { event_type, branch, commit_sha, ... }
      h. Match WebhookEvent against active Pipelines using trigger_events
         list and fnmatch glob on trigger_branches (empty = all branches)
      i. For each matched Pipeline, create a PipelineRun and enqueue
         dispatch_pipeline_run.delay(run_id)
      j. Update WebhookDelivery to processed (or skipped if no matches)

4.  dispatch_pipeline_run Celery task:
      - Sets PipelineRun status = running
      - Reads pipeline.config to enumerate stages and jobs
      - Creates a Job record per job definition (status=pending)
      - Enqueues execute_job.delay(run_id, job_name) for each job

5.  execute_job tasks run (respecting stage ordering):
      - Selects an available Runner (status=online) matching job tags
      - Marks the Runner busy and the Job running
      - Branches on runner_type:
          docker     → docker-py SDK, streams stdout/stderr directly
          kubernetes → kubernetes SDK, pod per job
          gce        → provision_vm(), poll status, terminate_vm()

6.  When the job completes:
      - Job status = success or failed (exit code for docker/k8s;
        agent POST /api/v1/jobs/{id}/report-status/ for GCE)
      - Runner is returned to available
      - PipelineRun status resolves when all jobs are terminal

7.  Browser polls GET /api/v1/runs/{id}/ and
    GET /api/v1/jobs/{id}/logs/ to show live progress
```

## Manual pipeline trigger

```
User clicks "Run" in the console  →  POST /api/v1/pipelines/{id}/trigger/
                                   ←  201 PipelineRun created
                                   →  (same dispatch flow as step 4 above)
```

## Runner heartbeat

```
Runner process  →  POST /api/v1/runners/heartbeat/  every 30 s
                   { "token": "...", "version": "1.0.0" }
Console         →  Sets runner.status=online, runner.last_heartbeat_at=now()

Celery beat     →  reap_stale_runners() every 60 s
                   Sets status=offline for runners last seen > 5 min ago
```

## Status state machine

```
PipelineRun / Job:

  pending ──► running ──► success
                 │
                 ├──► failed
                 │
                 └──► cancelled (user-initiated via POST /runs/{id}/cancel/)
```
