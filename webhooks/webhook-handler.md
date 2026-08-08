# Webhook Handler

Thinker CI receives push, pull request, and tag events from GitHub, GitLab, and Bitbucket via a single webhook endpoint. This document covers how the handler works, how to configure each provider, and how to debug delivery problems.

---

## Endpoint

```
POST /hooks/<project_slug>/
```

The endpoint is separate from the REST API (`/api/v1/`) to make clear that it is machine-to-machine only. It uses no user session and is explicitly CSRF-exempt. Authentication is via HMAC signature or token, not a user token.

---

## Processing pipeline

Every inbound request goes through ten steps in order:

```
1.  Resolve project slug → 404 if no project found

2.  Read raw body bytes (required for HMAC verification; must happen
    before Django parses the body as JSON)

3.  Create WebhookDelivery audit record with status=received

4.  Detect provider from project.provider field
    (github | gitlab | bitbucket)

5.  Verify signature / token
    - GitHub:    X-Hub-Signature-256: sha256=<HMAC-SHA256 hex>
    - GitLab:    X-Gitlab-Token: <plain text>
    - Bitbucket: X-Hub-Signature: sha256=<HMAC-SHA256 hex>
    → 403 + audit record updated to rejected on failure

6.  Idempotency check
    - Extract delivery_id from X-GitHub-Delivery or X-Request-UUID
    - If a processed/skipped delivery with the same ID exists,
      return 200 with the cached run_ids immediately

7.  Parse event
    - Map provider-specific headers + payload → normalised WebhookEvent
      { event_type, branch, commit_sha, commit_message, sender, ... }
    - Unsupported events (e.g. PR closed) raise ParseError → skipped

8.  Match pipelines
    - Filter active Pipelines for the project
    - Check event_type in Pipeline.trigger_events
    - Check fnmatch(branch, pattern) for each pattern in
      Pipeline.trigger_branches (empty list = all branches pass)

9.  Create runs
    - For each matched pipeline: create PipelineRun, enqueue
      dispatch_pipeline_run.delay(run_id)
    - Collect run IDs

10. Update WebhookDelivery
    - status = processed (if ≥1 pipeline matched)
    - status = skipped   (if 0 pipelines matched)
    - Store run_ids list
```

---

## Signature verification

### GitHub

GitHub signs every delivery with the webhook secret you configure in the repository settings. The signature is sent as:

```
X-Hub-Signature-256: sha256=<hex digest>
```

Verification:

```python
expected = "sha256=" + hmac.new(
    secret.encode(), raw_body, hashlib.sha256
).hexdigest()
result = hmac.compare_digest(expected, request_header)
```

`hmac.compare_digest` is used throughout to prevent timing attacks.

### GitLab

GitLab sends the secret as a plain-text header (no HMAC):

```
X-Gitlab-Token: <your secret>
```

Verification uses `hmac.compare_digest` on the two strings to maintain constant-time semantics.

### Bitbucket

Bitbucket mirrors GitHub's HMAC-SHA256 approach but uses a different header name:

```
X-Hub-Signature: sha256=<hex digest>
```

Verification is identical to GitHub.

---

## Supported events per provider

### GitHub

| `X-GitHub-Event` | `event_type` | Notes |
|---|---|---|
| `push` (branch ref) | `push` | `refs/heads/<branch>` |
| `push` (tag ref) | `tag` | `refs/tags/<tag>` |
| `pull_request` (opened / synchronize / reopened) | `pull_request` | Other actions → skipped |
| `create` | `push` or `tag` | Depends on `ref_type` |
| `ping` | `ping` | Always skipped; returns `{"detail":"pong"}` |

### GitLab

| `X-Gitlab-Event` | `event_type` | Notes |
|---|---|---|
| `Push Hook` | `push` | |
| `Tag Push Hook` | `tag` | |
| `Merge Request Hook` (open / update / reopen) | `pull_request` | Other actions → skipped |

### Bitbucket

| `X-Event-Key` | `event_type` | Notes |
|---|---|---|
| `repo:push` (branch change) | `push` | |
| `repo:push` (tag change) | `tag` | |
| `pullrequest:created` | `pull_request` | action=opened |
| `pullrequest:updated` | `pull_request` | action=synchronize |

---

## Idempotency

Git providers retry failed deliveries (GitHub retries on 5xx; Bitbucket retries twice). Without idempotency handling, a transient server error would cause duplicate pipeline runs.

Thinker CI handles this with a `UniqueConstraint` on `(project, delivery_id)` with a partial condition `delivery_id != ""`. When the same delivery ID arrives again:

1. The handler finds the existing `WebhookDelivery` with status `processed` or `skipped`
2. Returns 200 immediately with the original `run_ids` (or empty list if skipped)
3. Does not create new `PipelineRun` records

GitLab does not include a delivery UUID, so GitLab deliveries do not benefit from idempotency protection. Duplicate runs from GitLab retries are benign (they run the same commit twice).

---

## Branch matching with glob patterns

`Pipeline.trigger_branches` is a JSON list of patterns. An empty list means "all branches".

Each branch name from the webhook payload is tested against every pattern using Python's `fnmatch.fnmatch`. This enables glob patterns:

| Pattern | Matches |
|---|---|
| `main` | Only `main` |
| `release/*` | `release/1.0`, `release/hotfix` |
| `hotfix-*` | `hotfix-123`, `hotfix-auth` |
| `feature/**` | Not supported — `fnmatch` uses `*` not `**` |

To match any branch, leave `trigger_branches` empty (preferred) or include `*`.

---

## Audit trail

Every delivery — including rejected ones — is stored in `webhook_deliveries`. The Django admin exposes a read-only view at `/admin/webhooks/webhookdelivery/` with:

- Raw headers and payload JSON
- Normalised event fields (branch, commit SHA, sender)
- Processing status and error message
- IDs of pipeline runs created

This allows post-hoc debugging of any delivery without needing provider-side logs.

---

## Configuring GitHub

1. Open the repository in GitHub.
2. Go to **Settings → Webhooks → Add webhook**.
3. Set:
   - **Payload URL**: `https://<your-host>/hooks/<project-slug>/`
   - **Content type**: `application/json`
   - **Secret**: the value of `project.webhook_secret`
   - **Which events**: select individual events → Push, Pull requests, Create (for tags via `create` event)
4. Save. GitHub will send a `ping` event immediately to verify connectivity.

---

## Configuring GitLab

1. Open the project in GitLab.
2. Go to **Settings → Webhooks**.
3. Set:
   - **URL**: `https://<your-host>/hooks/<project-slug>/`
   - **Secret token**: the value of `project.webhook_secret`
   - **Trigger**: Push events, Tag push events, Merge request events
4. Save. Use **Test** to send a sample push event.

---

## Configuring Bitbucket

1. Open the repository in Bitbucket.
2. Go to **Repository settings → Webhooks → Add webhook**.
3. Set:
   - **URL**: `https://<your-host>/hooks/<project-slug>/`
   - **Secret**: the value of `project.webhook_secret`
   - **Triggers**: Repository push, Pull Request Created, Pull Request Updated
4. Save.

---

## Troubleshooting

### 403 Forbidden

Signature mismatch. Common causes:

- Wrong webhook secret in the Git provider settings.
- Trailing newline in the secret (copy-paste issue).
- A proxy is re-encoding the body before it reaches Django (e.g., a load balancer stripping chunked encoding). Ensure the raw bytes are passed through unchanged.

Check `WebhookDelivery.raw_headers` in the admin to confirm which signature header arrived.

### 404 Not Found

The project slug in the URL does not match any project. Verify the slug on the Project detail page and update the webhook URL in the Git provider.

### Delivery shows status=skipped, no run_ids

The signature verified and the event parsed, but no pipeline matched. Check:

1. `Pipeline.trigger_events` — does it include `push` / `pull_request` / `tag`?
2. `Pipeline.trigger_branches` — does the incoming branch match at least one pattern?
3. `Pipeline.is_active` — is the pipeline enabled?

`WebhookDelivery.branch` shows what branch name the handler extracted from the payload.

### Duplicate runs (GitLab only)

GitLab does not include a delivery UUID so duplicate-delivery protection is not available. If GitLab retries are causing double runs, check your network/firewall for intermittent 5xx responses that trigger retries. The `WebhookDelivery` audit log will show multiple `processed` records for the same commit SHA.

### Testing locally

Use `ngrok` or Cloudflare Tunnel to expose your local Django dev server to the internet, then point the Git provider webhook at the tunnel URL.

```bash
ngrok http 8000
# note the forwarding URL, e.g. https://abc123.ngrok.io
# set webhook URL to: https://abc123.ngrok.io/hooks/<slug>/
```

---

## Security notes

- **Never log the raw `X-Hub-Signature-256` header** in production — it is the verifier, not the key, but logging it would allow an attacker who reads logs to forge future deliveries if they also learn the body.
- **Use `hmac.compare_digest` everywhere** — direct string comparison (`==`) is vulnerable to timing attacks.
- **Rotate webhook secrets** via `Project.webhook_secret`; update the corresponding setting in the Git provider immediately after.
- **Require HTTPS** in production. Sending signatures over HTTP exposes the secret to network observers.
- **Scope the webhook** to only the events you need. Receiving all events wastes bandwidth and broadens the attack surface.
