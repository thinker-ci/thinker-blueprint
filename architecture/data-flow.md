# Data Flow

## Webhook-triggered pipeline run

```
1.  Developer pushes a commit to GitHub / GitLab / Bitbucket / thinker-git

2.  Git provider sends:  POST /hooks/{project_slug}/

3.  WebhookView (10-step pipeline):
      a. Look up Project by slug — 404 if not found
      b. Read raw body bytes before any parsing
      c. Store a WebhookDelivery audit record (status=received)
      d. Detect provider from project.provider
      e. Verify HMAC-SHA256 signature (GitHub, Bitbucket, thinker-git)
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
      - Validates pipeline.config (JSON Schema + semantic checks)
      - On error: sets run.status=failed, writes run.error_message, returns
      - On success: sets run.status=running, creates Job records, enqueues
        execute_job.delay(job_id) for each job

5.  execute_job tasks run (respecting stage ordering):
      - Selects an available Runner (status=online) matching job tags
      - Marks the Runner busy and the Job running
      - Branches on runner_type:
          docker     → docker-py SDK, streams stdout/stderr to JobLog
          kubernetes → kubernetes SDK, pod per job, streams pod logs
          gce        → provision_vm(), poll status, terminate_vm()

6.  When the job completes:
      - Job status = success or failed
      - Runner returned to available
      - PipelineRun status resolves when all jobs are terminal

7.  thinker-web polls GET /api/v1/runs/{id}/ and
    GET /api/v1/jobs/{id}/logs/ to show live progress
```

## Manual pipeline trigger

```
User clicks "Run" in thinker-web  →  POST /api/v1/pipelines/{id}/trigger/
                                   ←  201 PipelineRun created
                                   →  (same dispatch flow as step 4 above)
```

## Agent chat session (thinker-web → thinker-agent → tools)

```
1.  User opens /agents in thinker-web, selects a provider, types a message

2.  thinker-web  →  POST agent.thinker.ci/sessions/{id}/messages
                     { "content": "trigger the main pipeline for my-project" }

3.  thinker-agent: AgentLoop.run()
      a. Calls LLMProvider.complete() with conversation history + tool schemas
      b. LLM responds with a tool call: pipeline_trigger(pipeline_id=42, branch="main")
      c. AgentLoop executes the tool: POST api.thinker.ci/api/v1/pipelines/42/trigger/
      d. Observation returned to LLM: { "run_id": 99, "status": "pending" }
      e. LLM responds with natural language: "Pipeline triggered — run #99 is pending."
      f. Loop ends (no more tool calls)

4.  thinker-agent streams tokens back to thinker-web via SSE
      event: token  data: "Pipeline triggered"
      event: token  data: " — run #99 is pending."
      event: done

5.  thinker-web appends tokens to the chat bubble in real time
    Tool calls shown as collapsible cards: "🔧 pipeline_trigger" + input/output JSON
```

## MCP tool call (Claude Code → thinker-mcp → thinker-console)

```
1.  Claude Code has thinker-mcp configured in .claude.json

2.  User: "list the pipelines for my-project"

3.  Claude Code  →  MCP tool call: pipelines_list(project_slug="my-project")

4.  thinker-mcp  →  GET api.thinker.ci/api/v1/projects/my-project/pipelines/
                     Authorization: Token <THINKER_CONSOLE_TOKEN>

5.  thinker-console  →  200 [{ "id": 1, "name": "CI", ... }, ...]

6.  thinker-mcp  →  TextContent(JSON)  →  Claude Code

7.  Claude Code uses the result to answer the user
```

## Git push via thinker-git

```
1.  Developer: git push git.thinker.ci/acme/my-repo main

2.  Gitea receives push, fires system webhook to nginx
    POST nginx /hooks/gitea-system/

3.  nginx  →  proxy:8002 /hooks/gitea-system/

4.  proxy/webhooks.py:
      a. Verify HMAC-SHA256 signature (Gitea shared secret)
      b. Extract tenant slug from organisation name (org = tenant slug)
      c. POST api.thinker.ci/hooks/{tenant_slug}-{repo_slug}/
         X-Hub-Signature-256: <recomputed HMAC using project.webhook_secret>

5.  thinker-console processes the webhook as a normal push event
    (same 10-step pipeline as above)
```

## SSO flow (thinker-web → thinker-git proxy)

```
1.  thinker-web needs a Gitea token to show repo browsing UI

2.  thinker-web  →  POST git.thinker.ci/api/git/auth/sso
                     X-Thinker-Token: <console token>

3.  proxy/auth.py:
      a. GET api.thinker.ci/api/v1/me/  — validate token, get username/tenant
      b. GET/POST gitea /api/v1/orgs/{tenant_slug}  — ensure tenant org exists
      c. GET/POST gitea /api/v1/admin/users/{username}  — ensure user exists
      d. POST gitea /api/v1/users/{username}/tokens  — create or return token
      e. Add user to tenant org if not already a member

4.  proxy  →  200 { "gitea_token": "...", "gitea_url": "https://git.thinker.ci" }

5.  thinker-web stores gitea_token in localStorage, uses it for Gitea API calls
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

Story / Task (PM):

  backlog ──► todo ──► in_progress ──► in_review ──► done
                                           │
                                           └──► blocked
```
