# GCE Runner

The GCE runner provisions an ephemeral Google Compute Engine VM for every CI job.
Each VM boots from a custom image, executes exactly one job, streams its output back
to the Console, and then deletes itself.  No VM is ever reused between jobs, giving
every build a guaranteed clean environment.

---

## Why ephemeral VMs instead of persistent agents?

| Property | Persistent runner (Docker/k8s) | Ephemeral GCE VM |
|---|---|---|
| Environment cleanliness | Relies on container isolation | Fresh OS on every job — no state leakage |
| Machine type per job | Fixed | Any machine type, including GPU, high-memory, or ARM |
| Docker-in-Docker | Privileged containers required | Native Docker daemon — no privilege escalation |
| Startup latency | < 1 s (container start) | 20–60 s (VM boot) |
| Cost model | Always-on capacity | Pay only when a job runs |
| Isolation | Container namespace | Full VM boundary |
| Secret exposure risk | Host-level secrets visible in `/proc` | Zero cross-job exposure |

For workloads that need real Docker builds, GPU access, or strong job-to-job isolation,
the GCE runner is the right choice.  For fast unit tests where < 1 s cold-start matters,
use the Docker or Kubernetes runner.

---

## Architecture

```
Console (Django)
    │
    │  1. Job dispatched via Celery
    ▼
execute_gce_job task
    │
    │  2. Calls google-cloud-compute to INSERT instance
    │     — machine type, image, zone, labels from Runner model
    │     — job_id / token / console_url injected via instance metadata
    ▼
GCE VM (custom image)
    │
    │  3. systemd starts /etc/thinker-ci/agent.sh on first boot
    │     a. Reads metadata: job_id, console_url, token
    │     b. Fetches job definition  GET /api/v1/jobs/{id}/
    │     c. Pulls Docker image
    │     d. Runs each step inside Docker
    │     e. Streams log lines  POST /api/v1/jobs/{id}/ingest-logs/
    │     f. Reports result     POST /api/v1/jobs/{id}/report-status/
    │     g. Calls self-delete  gcloud compute instances delete $(hostname)
    ▼
execute_gce_job task (polling)
    │
    │  4. Detects terminal job.status → calls terminate_vm() as safety net
    │     (VM may already be gone from self-deletion)
```

---

## Runner model — GCE fields

When creating a GCE runner via `POST /api/v1/runners/`, set `runner_type` to `"gce"`
and supply the following fields:

| Field | Required | Default | Notes |
|---|---|---|---|
| `gce_project_id` | Yes | — | GCP project that owns the VMs |
| `gce_zone` | Yes | — | e.g. `us-central1-a` |
| `gce_machine_type` | No | `n2-standard-4` | Any valid GCE machine type |
| `gce_image` | Yes | — | Image name or `family/IMAGE_FAMILY` |
| `gce_image_project` | No | same as `gce_project_id` | GCP project owning the image |
| `gce_service_account` | Yes | — | SA email attached to VMs |
| `gce_network` | No | `default` | VPC network name |
| `gce_subnetwork` | No | — | Subnetwork (required for custom-mode VPCs) |
| `gce_disk_size_gb` | No | `50` | Boot disk size |
| `gce_use_spot` | No | `false` | Spot VMs — cheaper but may be reclaimed |
| `gce_resource_labels` | No | `{}` | GCP resource labels on job VMs |
| `gce_network_tags` | No | `["thinker-ci-runner"]` | Firewall-targeting network tags |
| `gce_scopes` | No | `["https://www.googleapis.com/auth/cloud-platform"]` | OAuth2 scopes |

**Example — create a GCE runner:**

```bash
curl -X POST https://console.thinker-ci.io/api/v1/runners/ \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "gce-us-central1",
    "runner_type": "gce",
    "tags": ["linux", "docker", "gce"],
    "max_concurrent_jobs": 20,
    "gce_project_id": "my-gcp-project",
    "gce_zone": "us-central1-a",
    "gce_machine_type": "n2-standard-8",
    "gce_image": "family/thinker-ci-runner",
    "gce_image_project": "my-gcp-project",
    "gce_service_account": "thinker-ci-runner@my-gcp-project.iam.gserviceaccount.com",
    "gce_network": "ci-network",
    "gce_subnetwork": "ci-subnet-us-central1",
    "gce_disk_size_gb": 100,
    "gce_use_spot": true,
    "gce_resource_labels": {"team": "platform", "env": "ci"},
    "gce_network_tags": ["thinker-ci-runner", "no-external-ip"]
  }'
```

---

## IAM requirements

The identity that runs the Console (Workload Identity, service account key, or ADC)
needs the following roles in `gce_project_id`:

| Role | Why |
|---|---|
| `roles/compute.instanceAdmin.v1` | Create, list, and delete instances |
| `roles/iam.serviceAccountUser` | Act as `gce_service_account` when creating VMs |

The VM service account (`gce_service_account`) needs:

| Role | Why |
|---|---|
| `roles/logging.logWriter` | Write Cloud Logging entries (optional, for structured logs) |
| `roles/storage.objectViewer` | Pull artifacts from a GCS bucket (if used) |

Scope it to the minimum — it does **not** need `compute.*` roles because it deletes
itself using the metadata server's embedded token scope, not the SA directly.

---

## Network topology

VMs are created with **no external IP** (`access_configs: []`).  Outbound traffic
(Docker image pulls, git clone, package downloads) flows through Cloud NAT.

Required firewall rule — allow the VM agent to call back to the Console:
- Direction: EGRESS
- Target tags: `thinker-ci-runner`
- Destination: Console's IP or private IP range
- Port: 443/TCP

If the Console runs inside the same VPC (recommended), no egress firewall rule is
needed beyond the default allow-internal.

---

## Spot VM behaviour

When `gce_use_spot = true`, VMs use GCE Spot provisioning model.  If GCP reclaims the
VM mid-job, the job will time out and be marked `failed` by the Console poll loop.
The `reap_orphaned_gce_instances` task will also clean up any leftover instances.

Mitigations:
- Set `RUNNER_JOB_TIMEOUT_SECONDS` to a value short enough to keep retry cost low.
- Enable Celery task retries on `execute_gce_job` (currently `max_retries=3`).
- Reserve Spot runners for non-critical jobs; use standard VMs for deploy jobs.

---

## Cancellation

`POST /api/v1/runs/{id}/cancel/` sets the run and all its jobs to `cancelled`.
The `execute_gce_job` poll loop detects the status change and calls `terminate_vm()`
immediately, rather than waiting for the timeout.

---

## Observability

Every GCE VM writes serial port output to Cloud Logging under the resource
`gce_instance`.  Filter by label `thinker-ci-job-id` to find the logs for a
specific job even if the Console's `job_logs` table is empty (e.g., the VM crashed
before the agent could send any lines).

```
resource.type="gce_instance"
labels."thinker-ci-job-id"="12345"
```
