# Data Flow

## Webhook-triggered pipeline run

```
1.  Developer pushes a commit to GitHub

2.  GitHub sends a POST /api/v1/webhooks/{project_slug}/ payload
      - Console validates the HMAC-SHA256 signature using project.webhook_secret
      - Console matches the event type and branch against active Pipeline records

3.  Console creates a PipelineRun (status=pending, trigger_event=push)

4.  Console enqueues dispatch_pipeline_run.delay(run_id) onto the Celery broker

5.  A Celery worker dequeues the task:
      - Reads pipeline.config to enumerate stages and jobs
      - Creates a Job record per job definition (status=pending)
      - Enqueues execute_job.delay(run_id, job_name) for each job

6.  execute_job tasks run (respecting stage ordering):
      - Selects an available Runner (status=online) that matches the job's tag requirements
      - Marks the Runner busy and the Job running
      - Submits the job container to Docker or Kubernetes
      - Streams stdout/stderr, writing JobLog rows as they arrive

7.  When the container exits:
      - Job status is set to success or failed based on exit code
      - Runner is returned to online
      - If all jobs in the run are terminal, PipelineRun status is resolved

8.  The browser console polls GET /api/v1/runs/{id}/ and
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
