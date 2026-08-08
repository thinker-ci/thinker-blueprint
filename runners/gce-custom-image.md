# Building a Custom GCE Runner Image

This guide walks through building the custom VM image that the GCE runner uses for
on-demand job provisioning.  The image is built with [HashiCorp Packer](https://www.packer.io/)
and stored in Google Cloud as an image family, so `gce_image: "family/thinker-ci-runner"`
always resolves to the latest good image automatically.

The finished image contains:
- Debian 12 (Bookworm) base
- Docker Engine (pinned version, daemon pre-configured)
- The Thinker CI job agent (`/etc/thinker-ci/agent.sh`)
- Security hardening (SSH locked down, auditd, fail2ban)
- Performance tuning (kernel parameters, cgroup v2, SSD I/O scheduler)
- Pre-pulled base Docker images to reduce cold-start time

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Repository layout](#2-repository-layout)
3. [Packer template](#3-packer-template)
4. [Provisioning script](#4-provisioning-script)
5. [The thinker-ci agent](#5-the-thinker-ci-agent)
6. [systemd unit](#6-systemd-unit)
7. [Security hardening](#7-security-hardening)
8. [Performance tuning](#8-performance-tuning)
9. [Pre-pulling Docker images](#9-pre-pulling-docker-images)
10. [Sealing the image](#10-sealing-the-image)
11. [Building and publishing](#11-building-and-publishing)
12. [Testing the image](#12-testing-the-image)
13. [Image family management](#13-image-family-management)
14. [Customising for your environment](#14-customising-for-your-environment)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Prerequisites

### Tools

```bash
# Packer — version 1.11 or later
brew install packer            # macOS
# or
wget -qO- https://releases.hashicorp.com/packer/1.11.2/packer_1.11.2_linux_amd64.zip \
  | bsdtar -xf- && sudo mv packer /usr/local/bin/

# Google Cloud SDK
curl https://sdk.cloud.google.com | bash
gcloud init
gcloud auth application-default login
```

### GCP IAM

The identity running Packer needs:

| Role | Purpose |
|---|---|
| `roles/compute.instanceAdmin.v1` | Create/delete the builder VM |
| `roles/compute.imageUser` | Create images from the builder disk |
| `roles/iam.serviceAccountUser` | Impersonate the builder SA |
| `roles/storage.objectAdmin` on the Packer GCS bucket | Store temporary metadata |

### Environment variables

```bash
export GCP_PROJECT="my-gcp-project"
export GCP_ZONE="us-central1-a"
export GCP_NETWORK="ci-network"
export GCP_SUBNETWORK="ci-subnet-us-central1"
export THINKER_CI_VERSION="1.0.0"   # baked into the image for introspection
```

---

## 2. Repository layout

Keep image-building assets in a dedicated directory, ideally a separate repo or a
`packer/` subdirectory in an infrastructure monorepo:

```
packer/
├── thinker-ci-runner.pkr.hcl       # Packer template (HCL2)
├── variables.pkrvars.hcl            # Variable overrides (not committed)
├── scripts/
│   ├── 01-system-update.sh          # OS updates and base packages
│   ├── 02-docker.sh                 # Docker Engine installation
│   ├── 03-docker-daemon.sh          # Docker daemon configuration
│   ├── 04-agent.sh                  # thinker-ci agent installation
│   ├── 05-security.sh               # Hardening
│   ├── 06-performance.sh            # Kernel/system tuning
│   ├── 07-prepull.sh                # Pre-pull common Docker images
│   └── 99-seal.sh                   # Image cleanup before snapshot
└── files/
    ├── agent.sh                     # Job agent script
    ├── thinker-ci-agent.service     # systemd unit
    ├── daemon.json                  # Docker daemon config
    ├── sysctl-thinker-ci.conf       # Kernel parameters
    └── limits-thinker-ci.conf       # ulimit overrides
```

---

## 3. Packer template

**`packer/thinker-ci-runner.pkr.hcl`**

```hcl
packer {
  required_plugins {
    googlecompute = {
      source  = "github.com/hashicorp/googlecompute"
      version = ">= 1.1.6"
    }
  }
}

# ── Variables ────────────────────────────────────────────────────────────────

variable "project_id"          { type = string }
variable "zone"                 { type = string  default = "us-central1-a" }
variable "network"              { type = string  default = "default" }
variable "subnetwork"           { type = string  default = "" }
variable "machine_type"         { type = string  default = "n2-standard-4" }
variable "disk_size_gb"         { type = number  default = 50 }
variable "thinker_ci_version"   { type = string  default = "dev" }
variable "image_family"         { type = string  default = "thinker-ci-runner" }
variable "builder_sa"           { type = string  default = "" }

locals {
  timestamp  = formatdate("YYYYMMDD-hhmmss", timestamp())
  image_name = "${var.image_family}-${replace(var.thinker_ci_version, ".", "-")}-${local.timestamp}"
}

# ── Source ───────────────────────────────────────────────────────────────────

source "googlecompute" "runner" {
  project_id   = var.project_id
  zone         = var.zone
  machine_type = var.machine_type

  # Use Debian 12 as the base — stable LTS, great Docker support.
  source_image_family  = "debian-12"
  source_image_project = "debian-cloud"

  disk_size    = var.disk_size_gb
  disk_type    = "pd-ssd"             # SSD for faster Docker layer I/O

  network    = var.network
  subnetwork = var.subnetwork != "" ? var.subnetwork : null

  # No external IP — Packer communicates via IAP tunnel.
  use_iap                  = true
  use_internal_ip          = true
  omit_external_ip         = true

  service_account_email    = var.builder_sa != "" ? var.builder_sa : null
  impersonate_service_account = var.builder_sa != "" ? var.builder_sa : null

  # Shielded VM — enables vTPM and Integrity Monitoring.
  enable_secure_boot          = true
  enable_vtpm                 = true
  enable_integrity_monitoring = true

  # The builder VM is tagged so your firewall can allow IAP access.
  tags = ["packer-builder"]

  image_name        = local.image_name
  image_family      = var.image_family
  image_description = "Thinker CI runner image v${var.thinker_ci_version} built ${local.timestamp}"

  image_labels = {
    thinker-ci-version = replace(var.thinker_ci_version, ".", "-")
    built-by           = "packer"
    image-family       = var.image_family
  }

  # Metadata on the builder instance only — not inherited by job VMs.
  metadata = {
    block-project-ssh-keys = "true"
  }

  ssh_username         = "packer"
  temporary_key_pair_type = "ed25519"
}

# ── Build ────────────────────────────────────────────────────────────────────

build {
  name    = "thinker-ci-runner"
  sources = ["source.googlecompute.runner"]

  # Upload static files first so provisioning scripts can reference them.
  provisioner "file" {
    sources     = ["files/"]
    destination = "/tmp/thinker-ci-files/"
  }

  # Run provisioning scripts in order.
  provisioner "shell" {
    environment_vars = [
      "THINKER_CI_VERSION=${var.thinker_ci_version}",
      "DEBIAN_FRONTEND=noninteractive",
    ]
    scripts = [
      "scripts/01-system-update.sh",
      "scripts/02-docker.sh",
      "scripts/03-docker-daemon.sh",
      "scripts/04-agent.sh",
      "scripts/05-security.sh",
      "scripts/06-performance.sh",
      "scripts/07-prepull.sh",
      "scripts/99-seal.sh",
    ]
    execute_command = "sudo bash '{{ .Path }}'"
  }

  # Verify the image from inside the builder before snapshotting.
  provisioner "shell" {
    inline = [
      "docker --version",
      "systemctl is-enabled thinker-ci-agent",
      "test -x /etc/thinker-ci/agent.sh && echo 'Agent OK'",
      "sysctl vm.max_map_count | grep -q 262144 && echo 'Kernel params OK'",
    ]
  }
}
```

**`packer/variables.pkrvars.hcl`** (add to `.gitignore`):

```hcl
project_id         = "my-gcp-project"
zone               = "us-central1-a"
network            = "ci-network"
subnetwork         = "ci-subnet-us-central1"
thinker_ci_version = "1.2.0"
builder_sa         = "packer-builder@my-gcp-project.iam.gserviceaccount.com"
```

---

## 4. Provisioning script

### `scripts/01-system-update.sh` — OS updates and base packages

```bash
#!/usr/bin/env bash
set -euo pipefail

apt-get update -y
apt-get upgrade -y

# Base utilities
apt-get install -y --no-install-recommends \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  jq \
  git \
  unzip \
  make \
  build-essential \
  net-tools \
  iproute2 \
  dnsutils \
  htop \
  iotop \
  sysstat \
  auditd \
  fail2ban \
  ufw \
  python3 \
  python3-pip \
  python3-venv

# Google Cloud ops agent — structured logging + metrics to Cloud Monitoring
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
bash add-google-cloud-ops-agent-repo.sh --also-install
systemctl enable google-cloud-ops-agent

# gcloud CLI — needed by the agent's self-delete step
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] \
  https://packages.cloud.google.com/apt cloud-sdk main" \
  > /etc/apt/sources.list.d/google-cloud-sdk.list

apt-get update -y
apt-get install -y google-cloud-cli

apt-get autoremove -y
apt-get clean
rm -rf /var/lib/apt/lists/*
```

### `scripts/02-docker.sh` — Docker Engine

```bash
#!/usr/bin/env bash
set -euo pipefail

# Pin to a specific Docker Engine version for reproducibility.
DOCKER_VERSION="27.3.1"

# Add Docker's official GPG key.
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" \
  > /etc/apt/sources.list.d/docker.list

apt-get update -y
apt-get install -y --no-install-recommends \
  "docker-ce=${DOCKER_VERSION}*" \
  "docker-ce-cli=${DOCKER_VERSION}*" \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# Keep Docker from being auto-upgraded — version is pinned for reproducibility.
apt-mark hold docker-ce docker-ce-cli

# Create the thinker-ci runner user that will own job containers.
useradd --system --shell /sbin/nologin --create-home --home-dir /var/lib/thinker-ci thinker-ci
usermod -aG docker thinker-ci

systemctl enable docker
systemctl start docker

echo "Docker ${DOCKER_VERSION} installed and started"
docker --version
```

### `scripts/03-docker-daemon.sh` — Docker daemon configuration

```bash
#!/usr/bin/env bash
set -euo pipefail

cp /tmp/thinker-ci-files/daemon.json /etc/docker/daemon.json
chmod 0644 /etc/docker/daemon.json

systemctl restart docker
systemctl is-active docker

echo "Docker daemon configured"
```

**`files/daemon.json`** — Docker daemon settings:

```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "max-concurrent-downloads": 6,
  "max-concurrent-uploads": 4,
  "experimental": false,
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "icc": false,
  "default-address-pools": [
    { "base": "172.80.0.0/12", "size": 24 }
  ]
}
```

Key decisions in `daemon.json`:

| Setting | Value | Rationale |
|---|---|---|
| `storage-driver` | `overlay2` | Best performance on SSD; default for Debian |
| `log-driver` | `json-file` | Logs readable by `docker logs`; capped to prevent disk fill |
| `live-restore` | `true` | Running containers survive dockerd restarts |
| `icc` | `false` | Disable inter-container communication by default (security) |
| `no-new-privileges` | `true` | Prevent container privilege escalation |
| `userland-proxy` | `false` | Use iptables hairpin NAT instead of a proxy process |

### `scripts/04-agent.sh` — thinker-ci agent installation

```bash
#!/usr/bin/env bash
set -euo pipefail

AGENT_DIR="/etc/thinker-ci"
mkdir -p "${AGENT_DIR}"

# Copy the agent script and systemd unit from the uploaded files.
cp /tmp/thinker-ci-files/agent.sh "${AGENT_DIR}/agent.sh"
chmod 0750 "${AGENT_DIR}/agent.sh"
chown thinker-ci:thinker-ci "${AGENT_DIR}/agent.sh"

cp /tmp/thinker-ci-files/thinker-ci-agent.service /etc/systemd/system/thinker-ci-agent.service
chmod 0644 /etc/systemd/system/thinker-ci-agent.service

systemctl daemon-reload
systemctl enable thinker-ci-agent

# Write the baked version for introspection.
echo "${THINKER_CI_VERSION}" > "${AGENT_DIR}/version"
chmod 0444 "${AGENT_DIR}/version"

echo "thinker-ci agent installed (version ${THINKER_CI_VERSION})"
```

---

## 5. The thinker-ci agent

The agent is a shell script that runs inside the VM on first boot.  It reads its
instructions from the GCE instance metadata server, executes the job, and reports
results back to the Console.

**`files/agent.sh`**:

```bash
#!/usr/bin/env bash
# thinker-ci job agent — runs once per VM lifetime
set -euo pipefail

METADATA_URL="http://metadata.google.internal/computeMetadata/v1/instance/attributes"
METADATA_HEADERS=(-H "Metadata-Flavor: Google")

# ── Helpers ──────────────────────────────────────────────────────────────────

log() { echo "[thinker-ci] $(date -u +%FT%TZ) $*"; }

meta() {
  # Retry up to 5 times — metadata server may not be ready immediately at boot.
  local key="$1" attempt
  for attempt in 1 2 3 4 5; do
    local val
    val=$(curl -sf "${METADATA_HEADERS[@]}" "${METADATA_URL}/${key}") && echo "${val}" && return
    log "Metadata '${key}' not ready (attempt ${attempt}/5), retrying in 3s..."
    sleep 3
  done
  log "ERROR: Could not fetch metadata key '${key}' after 5 attempts"
  exit 1
}

post_logs() {
  local payload="$1"
  curl -sf \
    -X POST \
    -H "Authorization: Token ${RUNNER_TOKEN}" \
    -H "Content-Type: application/json" \
    "${CONSOLE_URL}/api/v1/jobs/${JOB_ID}/ingest-logs/" \
    -d "${payload}" \
    > /dev/null
}

report_status() {
  local status="$1" exit_code="$2"
  curl -sf \
    -X POST \
    -H "Authorization: Token ${RUNNER_TOKEN}" \
    -H "Content-Type: application/json" \
    "${CONSOLE_URL}/api/v1/jobs/${JOB_ID}/report-status/" \
    -d "{\"status\": \"${status}\", \"exit_code\": ${exit_code}}"
}

send_log_line() {
  local line="$1"
  local payload
  payload=$(jq -n --arg l "${line}" '{"lines": [$l]}')
  post_logs "${payload}"
}

# ── Bootstrap ─────────────────────────────────────────────────────────────────

log "Agent starting on $(hostname)"
CONSOLE_URL=$(meta "thinker-ci-console-url")
JOB_ID=$(meta "thinker-ci-job-id")
RUNNER_TOKEN=$(meta "thinker-ci-runner-token")
JOB_IMAGE=$(meta "thinker-ci-job-image")

log "Console: ${CONSOLE_URL} | Job: ${JOB_ID}"

# ── Fetch job definition ──────────────────────────────────────────────────────

JOB_DEF=$(curl -sf \
  -H "Authorization: Token ${RUNNER_TOKEN}" \
  "${CONSOLE_URL}/api/v1/jobs/${JOB_ID}/")

# Parse steps from the job definition.
# Steps are stored in the pipeline config under jobs[].steps.
STEPS=$(echo "${JOB_DEF}" | jq -r '.run.pipeline.config // empty')

# ── Workspace ─────────────────────────────────────────────────────────────────

WORKSPACE="/workspace/job-${JOB_ID}"
mkdir -p "${WORKSPACE}"
chown thinker-ci:thinker-ci "${WORKSPACE}"

# ── Clone the repository ──────────────────────────────────────────────────────

COMMIT_SHA=$(echo "${JOB_DEF}" | jq -r '.run.commit_sha // empty')
REPO_URL=$(echo "${JOB_DEF}" | jq -r '.run.pipeline.project.repo_url')

send_log_line "[thinker-ci] Cloning ${REPO_URL}"
git clone --depth=1 "${REPO_URL}" "${WORKSPACE}/src" 2>&1 | while IFS= read -r line; do
  send_log_line "${line}"
done

if [[ -n "${COMMIT_SHA}" ]]; then
  git -C "${WORKSPACE}/src" fetch --depth=1 origin "${COMMIT_SHA}"
  git -C "${WORKSPACE}/src" checkout FETCH_HEAD
fi

# ── Pull the Docker image ─────────────────────────────────────────────────────

if [[ -n "${JOB_IMAGE}" ]]; then
  send_log_line "[thinker-ci] Pulling image: ${JOB_IMAGE}"
  docker pull "${JOB_IMAGE}" 2>&1 | while IFS= read -r line; do
    send_log_line "${line}"
  done
fi

# ── Execute each step ─────────────────────────────────────────────────────────

EXIT_CODE=0
SEQ=0

run_step() {
  local step="$1"
  send_log_line "[thinker-ci] \$ ${step}"
  set +e
  docker run --rm \
    --workdir /workspace \
    --volume "${WORKSPACE}/src:/workspace" \
    --env CI=true \
    --env THINKER_CI=true \
    --env THINKER_CI_JOB_ID="${JOB_ID}" \
    --env THINKER_CI_COMMIT_SHA="${COMMIT_SHA}" \
    --network bridge \
    --memory 4g \
    --cpus 2 \
    "${JOB_IMAGE:-debian:12-slim}" \
    bash -c "${step}" 2>&1 | while IFS= read -r line; do
      send_log_line "${line}"
    done
  local step_exit=$?
  set -e
  return ${step_exit}
}

# Read steps from the pipeline config; fall back to a default no-op.
STEPS_JSON=$(echo "${JOB_DEF}" | \
  jq -c '[.run.pipeline.config.stages[]?.jobs[]? | select(.name == "'"${JOB_ID}"'") | .steps[]?] // []')

if [[ "${STEPS_JSON}" == "[]" ]]; then
  send_log_line "[thinker-ci] WARNING: No steps found for job ${JOB_ID}"
else
  while IFS= read -r step; do
    if ! run_step "${step}"; then
      EXIT_CODE=$?
      send_log_line "[thinker-ci] Step failed with exit code ${EXIT_CODE}"
      break
    fi
  done < <(echo "${STEPS_JSON}" | jq -r '.[]')
fi

# ── Report result ─────────────────────────────────────────────────────────────

STATUS="success"
[[ ${EXIT_CODE} -ne 0 ]] && STATUS="failed"

send_log_line "[thinker-ci] Job ${STATUS} (exit ${EXIT_CODE})"
report_status "${STATUS}" "${EXIT_CODE}"

log "Job ${JOB_ID} complete — status=${STATUS} exit_code=${EXIT_CODE}"

# ── Self-deletion ─────────────────────────────────────────────────────────────

SELF_DESTRUCT=$(meta "thinker-ci-self-destruct" 2>/dev/null || echo "false")
if [[ "${SELF_DESTRUCT}" == "true" ]]; then
  log "Self-destruct enabled — deleting this instance"
  INSTANCE_NAME=$(curl -sf "${METADATA_HEADERS[@]}" \
    "http://metadata.google.internal/computeMetadata/v1/instance/name")
  ZONE=$(curl -sf "${METADATA_HEADERS[@]}" \
    "http://metadata.google.internal/computeMetadata/v1/instance/zone" \
    | awk -F/ '{print $NF}')
  PROJECT=$(curl -sf "${METADATA_HEADERS[@]}" \
    "http://metadata.google.internal/computeMetadata/v1/project/project-id")

  # The VM SA needs compute.instances.delete on itself.
  # This is granted via the thinker-ci-runner-self-delete IAM binding.
  gcloud compute instances delete "${INSTANCE_NAME}" \
    --zone="${ZONE}" --project="${PROJECT}" --quiet
fi
```

---

## 6. systemd unit

The agent runs exactly once per boot via a systemd `oneshot` unit.

**`files/thinker-ci-agent.service`**:

```ini
[Unit]
Description=Thinker CI Job Agent
After=network-online.target docker.service google-cloud-ops-agent.service
Wants=network-online.target docker.service
# Do not restart — each VM runs exactly one job.
StartLimitIntervalSec=0

[Service]
Type=oneshot
RemainAfterExit=no
User=thinker-ci
Group=thinker-ci
SupplementaryGroups=docker
ExecStart=/etc/thinker-ci/agent.sh

# Capture all output to the journal (visible in Cloud Logging via ops-agent).
StandardOutput=journal
StandardError=journal
SyslogIdentifier=thinker-ci-agent

# Give the job the full allowed timeout before systemd kills it.
TimeoutStartSec=infinity

# Resource limits for the agent process itself (not Docker containers).
LimitNOFILE=65536
LimitNPROC=4096
OOMScoreAdjust=-500

[Install]
WantedBy=multi-user.target
```

---

## 7. Security hardening

**`scripts/05-security.sh`**:

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── SSH ───────────────────────────────────────────────────────────────────────
# Lock down SSH — the builder uses IAP, job VMs should never accept SSH.
cat > /etc/ssh/sshd_config.d/99-thinker-ci.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
X11Forwarding no
AllowTcpForwarding no
PrintLastLog no
MaxAuthTries 3
LoginGraceTime 30
EOF

# Remove any authorized keys that Packer left behind.
# The real cleanup happens in 99-seal.sh; this is defence-in-depth.
rm -rf /home/packer/.ssh /root/.ssh 2>/dev/null || true

# ── firewall (ufw) ────────────────────────────────────────────────────────────
# Only allow established connections in; VMs have no external IP anyway.
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw --force enable

# ── auditd ────────────────────────────────────────────────────────────────────
systemctl enable auditd
cat > /etc/audit/rules.d/thinker-ci.rules <<'EOF'
# Watch Docker socket and config for tampering.
-w /var/run/docker.sock -p rwxa -k docker-socket
-w /etc/docker/ -p wa -k docker-config
# Watch the thinker-ci agent.
-w /etc/thinker-ci/ -p rwxa -k thinker-ci-agent
# Log all privileged executions.
-a always,exit -F arch=b64 -S execve -F euid=0 -k root-exec
EOF

# ── fail2ban ──────────────────────────────────────────────────────────────────
cat > /etc/fail2ban/jail.d/thinker-ci.conf <<'EOF'
[sshd]
enabled = true
maxretry = 3
bantime  = 3600
findtime = 600
EOF
systemctl enable fail2ban

# ── Disable unnecessary services ──────────────────────────────────────────────
for svc in bluetooth avahi-daemon cups ModemManager; do
  systemctl disable "${svc}" 2>/dev/null || true
done

# ── Kernel security parameters ────────────────────────────────────────────────
cat > /etc/sysctl.d/99-thinker-ci-security.conf <<'EOF'
# Restrict ptrace to direct children only (prevents Docker breakouts via ptrace).
kernel.yama.ptrace_scope = 1
# Disable core dumps for setuid binaries.
fs.suid_dumpable = 0
# Protect against SYN flood.
net.ipv4.tcp_syncookies = 1
# Ignore ICMP redirects.
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
# Do not send ICMP redirects.
net.ipv4.conf.all.send_redirects = 0
# Ignore source-routed packets.
net.ipv4.conf.all.accept_source_route = 0
EOF
sysctl --system

echo "Security hardening complete"
```

---

## 8. Performance tuning

**`scripts/06-performance.sh`**:

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Kernel parameters for CI workloads ────────────────────────────────────────
cp /tmp/thinker-ci-files/sysctl-thinker-ci.conf /etc/sysctl.d/60-thinker-ci-perf.conf
sysctl --system

# ── File descriptor and process limits ────────────────────────────────────────
cp /tmp/thinker-ci-files/limits-thinker-ci.conf /etc/security/limits.d/thinker-ci.conf

# ── I/O scheduler — use mq-deadline for NVMe/SSD ─────────────────────────────
# Persist across reboots via udev rule.
cat > /etc/udev/rules.d/60-thinker-ci-io.rules <<'EOF'
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="vd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
EOF

# ── cgroup v2 ─────────────────────────────────────────────────────────────────
# Debian 12 ships with cgroup v2 by default; confirm and set delegation.
if [[ "$(stat -fc %T /sys/fs/cgroup/)" != "cgroup2fs" ]]; then
  # Enable cgroup v2 on the next boot.
  sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"/' \
    /etc/default/grub
  update-grub
fi

# Delegate cgroup subtree to Docker.
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/delegate.conf <<'EOF'
[Service]
Delegate=yes
EOF
systemctl daemon-reload

# ── Transparent hugepages ──────────────────────────────────────────────────────
# Disable THP — it causes latency spikes in JVM and database workloads.
cat > /etc/systemd/system/disable-thp.service <<'EOF'
[Unit]
Description=Disable Transparent Hugepages
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl enable disable-thp

# ── journald — cap log size ────────────────────────────────────────────────────
mkdir -p /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/thinker-ci.conf <<'EOF'
[Journal]
SystemMaxUse=2G
RuntimeMaxUse=500M
Compress=yes
EOF

echo "Performance tuning complete"
```

**`files/sysctl-thinker-ci.conf`**:

```ini
# Docker and container networking
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1

# Large-scale builds open many files and processes.
fs.file-max        = 1048576
fs.inotify.max_user_watches    = 524288
fs.inotify.max_user_instances  = 8192

# Elasticsearch / JVM workloads require this.
vm.max_map_count = 262144

# Reduce swap tendency for a build environment.
vm.swappiness = 10

# Network throughput for parallel asset downloads.
net.core.somaxconn               = 65535
net.ipv4.tcp_max_syn_backlog     = 65535
net.core.netdev_max_backlog      = 65535
net.ipv4.tcp_fin_timeout         = 15
net.core.rmem_max                = 134217728
net.core.wmem_max                = 134217728
net.ipv4.tcp_rmem                = 4096 87380 134217728
net.ipv4.tcp_wmem                = 4096 65536 134217728
```

**`files/limits-thinker-ci.conf`**:

```
# Applied to all users, including thinker-ci and the docker daemon.
*    soft nofile  65536
*    hard nofile  131072
*    soft nproc   32768
*    hard nproc   65536
root soft nofile  65536
root hard nofile  131072
```

---

## 9. Pre-pulling Docker images

Pre-pulling common base images dramatically reduces job startup time for the first
job after a VM boots (subsequent containers reuse layers from the daemon cache).

**`scripts/07-prepull.sh`**:

```bash
#!/usr/bin/env bash
set -euo pipefail

# These images are baked into the Docker layer cache on the boot disk.
# Keep this list in sync with the images your pipelines actually use.
# Pulling images you don't use wastes disk and build time.
IMAGES=(
  "debian:12-slim"
  "ubuntu:22.04"
  "python:3.12-slim"
  "python:3.11-slim"
  "node:20-slim"
  "node:18-slim"
  "golang:1.22-bookworm"
  "openjdk:21-slim"
  "alpine:3.20"
  "docker:27-dind"
)

for image in "${IMAGES[@]}"; do
  echo "Pulling ${image} ..."
  docker pull "${image}"
done

# Verify all images were pulled successfully.
for image in "${IMAGES[@]}"; do
  docker image inspect "${image}" > /dev/null || {
    echo "ERROR: ${image} not found after pull"
    exit 1
  }
done

# Report total cache size for the image build summary.
echo ""
echo "Docker image cache size:"
docker system df

echo "Pre-pull complete — $(docker images --format 'table {{.Repository}}:{{.Tag}}\t{{.Size}}' | wc -l) images cached"
```

**Disk sizing guidance:**

| Base images pulled | Recommended `gce_disk_size_gb` |
|---|---|
| 0 (no pre-pull) | 30 GiB |
| 5–10 slim images | 50 GiB |
| 10–20 full images | 100 GiB |
| + build caches (node_modules, Go module cache) | 150 GiB |

---

## 10. Sealing the image

Before Packer snapshots the disk, `99-seal.sh` removes all traces of the build
session to ensure VMs booted from this image start clean.

**`scripts/99-seal.sh`**:

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Remove build-time SSH credentials ─────────────────────────────────────────
rm -rf /home/packer /root/.ssh

# ── Clear temporary Packer files ──────────────────────────────────────────────
rm -rf /tmp/thinker-ci-files /tmp/packer* /tmp/*.sh

# ── Remove package manager caches ─────────────────────────────────────────────
apt-get clean
rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*

# ── Clear logs accumulated during image build ─────────────────────────────────
journalctl --rotate --vacuum-time=1s 2>/dev/null || true
find /var/log -type f -name "*.log" -delete
find /var/log -type f -name "syslog*" -delete
find /var/log -type f -name "auth.log*" -delete
find /var/log -type f -name "kern.log*" -delete
find /var/log -type f -name "dpkg.log*" -delete
find /var/log -type f -name "apt*" -delete
truncate -s 0 /var/log/btmp /var/log/wtmp /var/log/lastlog 2>/dev/null || true

# ── Reset machine-id ──────────────────────────────────────────────────────────
# A cloned machine-id causes DBUS, systemd-journald, and DNS issues.
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -sf /etc/machine-id /var/lib/dbus/machine-id

# ── Clear cloud-init state so VMs personalise on first boot ──────────────────
cloud-init clean --logs --seed 2>/dev/null || true

# ── Remove Docker container runtime state (keep images, remove containers) ────
# This ensures VMs boot with a clean container namespace.
docker system prune -f --volumes 2>/dev/null || true

# ── Remove any SSH host keys — they are regenerated on first boot ─────────────
rm -f /etc/ssh/ssh_host_*

# ── Zero out free disk space for better compression ───────────────────────────
# Uncomment if you want smaller image transfer sizes (adds ~5 min to build).
# dd if=/dev/zero of=/tmp/bigfile bs=1M 2>/dev/null || true
# rm -f /tmp/bigfile
# sync

echo "Image sealed successfully"
```

---

## 11. Building and publishing

```bash
cd packer/

# Initialize plugins (first time only).
packer init thinker-ci-runner.pkr.hcl

# Validate the template.
packer validate -var-file=variables.pkrvars.hcl thinker-ci-runner.pkr.hcl

# Build the image.  This takes 10–20 minutes.
packer build -var-file=variables.pkrvars.hcl thinker-ci-runner.pkr.hcl
```

Packer creates the image in your GCP project and adds it to the `thinker-ci-runner`
image family.  The image name will be something like
`thinker-ci-runner-1-2-0-20260807-143022`.

Confirm the image was published:

```bash
gcloud compute images list \
  --project="${GCP_PROJECT}" \
  --filter="family:thinker-ci-runner" \
  --format="table(name,family,status,creationTimestamp)"
```

---

## 12. Testing the image

### Manual smoke test

```bash
# Launch a VM from the new image (one-off, not a real CI job).
gcloud compute instances create thinker-ci-smoke-test \
  --project="${GCP_PROJECT}" \
  --zone="${GCP_ZONE}" \
  --machine-type=n2-standard-4 \
  --image-family=thinker-ci-runner \
  --image-project="${GCP_PROJECT}" \
  --disk-size=50 \
  --disk-type=pd-ssd \
  --no-address \
  --metadata=thinker-ci-self-destruct=false \
  --scopes=cloud-platform

# Connect via IAP (no external IP).
gcloud compute ssh thinker-ci-smoke-test \
  --project="${GCP_PROJECT}" \
  --zone="${GCP_ZONE}" \
  --tunnel-through-iap

# Inside the VM — verify key components:
docker --version
docker run --rm debian:12-slim echo "Docker works"
systemctl status thinker-ci-agent
cat /etc/thinker-ci/version
sysctl vm.max_map_count
sysctl fs.file-max
sysctl net.ipv4.ip_forward

# Exit and clean up.
exit
gcloud compute instances delete thinker-ci-smoke-test \
  --project="${GCP_PROJECT}" --zone="${GCP_ZONE}" --quiet
```

### Automated verification with a real job

1. Update your GCE runner record to point at the new image:

```bash
curl -X PATCH https://console.thinker-ci.io/api/v1/runners/1/ \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"gce_image": "family/thinker-ci-runner"}'
```

2. Trigger a test pipeline manually and verify:
   - VM is created (visible in GCP console)
   - Logs stream into the Console
   - VM self-deletes after the job completes

---

## 13. Image family management

### Naming convention

```
thinker-ci-runner-{semver-with-dashes}-{YYYYMMDD}-{HHMMSS}
e.g.  thinker-ci-runner-1-2-0-20260807-143022
```

### Deprecation workflow

After publishing a new image, deprecate the previous generation to prevent
accidental use while keeping it available for rollback:

```bash
# Find the image being replaced.
OLD_IMAGE=$(gcloud compute images list \
  --project="${GCP_PROJECT}" \
  --filter="family:thinker-ci-runner status:READY" \
  --sort-by="~creationTimestamp" \
  --format="value(name)" \
  | sed -n '2p')   # Second row = previous image

# Deprecate it (still accessible by name, not selected by family).
gcloud compute images deprecate "${OLD_IMAGE}" \
  --project="${GCP_PROJECT}" \
  --state=DEPRECATED \
  --replacement="$(gcloud compute images describe-from-family thinker-ci-runner \
      --project="${GCP_PROJECT}" --format='value(name)')"

echo "Deprecated ${OLD_IMAGE}"
```

### Deletion policy

Keep at least two versions:
- **READY** — the current production image (selected by family)
- **DEPRECATED** — the previous version, kept for rollback

Delete images older than 90 days:

```bash
gcloud compute images list \
  --project="${GCP_PROJECT}" \
  --filter="family:thinker-ci-runner status:DEPRECATED" \
  --format="value(name,creationTimestamp)" \
  | while read name ts; do
      age=$(( ( $(date +%s) - $(date -d "${ts}" +%s) ) / 86400 ))
      if (( age > 90 )); then
        echo "Deleting ${name} (${age} days old)"
        gcloud compute images delete "${name}" --project="${GCP_PROJECT}" --quiet
      fi
    done
```

### Automating image builds

Add image builds to your CI pipeline (ideally using Thinker CI itself, once it's
stable):

```yaml
# Pipeline config for building the runner image
stages:
  - name: build-image
    jobs:
      - name: packer-build
        image: hashicorp/packer:latest
        tags: ["linux"]
        steps:
          - "packer init packer/thinker-ci-runner.pkr.hcl"
          - "packer build -var thinker_ci_version=${CI_COMMIT_TAG} packer/thinker-ci-runner.pkr.hcl"
```

Trigger on semver tags.  Never push a new image directly to the `READY` family
without the smoke test passing.

---

## 14. Customising for your environment

### Different base OS

| OS | Source image family | Source project | Notes |
|---|---|---|---|
| Debian 12 (recommended) | `debian-12` | `debian-cloud` | LTS, minimal, Docker-friendly |
| Ubuntu 22.04 LTS | `ubuntu-2204-lts` | `ubuntu-os-cloud` | Larger package ecosystem |
| Ubuntu 24.04 LTS | `ubuntu-2404-lts` | `ubuntu-os-cloud` | Latest LTS |
| Container-Optimized OS | `cos-stable` | `cos-cloud` | Docker only, no apt, no systemd services |

Container-Optimized OS is **not recommended** for Thinker CI because the agent
script requires `apt`, `git`, `gcloud`, and systemd service management.

### GPU workloads

Add a GPU to the builder and job VMs for machine-learning pipelines:

```hcl
# In the Packer source block:
accelerator {
  count = 1
  type  = "nvidia-tesla-t4"
}
on_host_maintenance = "TERMINATE"   # Required for GPU instances

# Add to 02-docker.sh — NVIDIA Container Toolkit:
distribution=$(. /etc/os-release; echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor \
  -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  > /etc/apt/sources.list.d/nvidia-container-toolkit.list
apt-get update -y
apt-get install -y nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker
```

Use `gce_machine_type: "n1-standard-8"` with `gce_network_tags: ["gpu"]` in the
runner config.

### ARM64 workloads

Build a separate image family for ARM:

```hcl
source "googlecompute" "runner-arm64" {
  machine_type        = "t2a-standard-4"    # Tau T2A — ARM-based
  source_image_family = "debian-12-arm64"
  source_image_project = "debian-cloud"
  image_family        = "thinker-ci-runner-arm64"
  # ... other settings identical to the x86 template
}
```

Use `tags: ["arm64"]` on the runner and `tags: ["arm64"]` in the pipeline job to
route ARM jobs to the ARM runner pool.

### Custom Docker registry mirror

Pre-configure a registry mirror to avoid Docker Hub rate limits:

```json
// Add to daemon.json:
{
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

`mirror.gcr.io` is Google's Docker Hub mirror, available free within GCP.

### Build secrets and artifact caching

Mount a GCS bucket for shared caches (node_modules, pip, Maven, Gradle):

```bash
# In agent.sh — before running steps:
CACHE_BUCKET="gs://my-ci-cache/${JOB_PIPELINE_SLUG}"
mkdir -p /workspace/cache

# Pull cache if it exists.
gsutil -m cp -r "${CACHE_BUCKET}/" /workspace/cache/ 2>/dev/null || true

# After running steps — push updated cache.
gsutil -m rsync -r /workspace/cache/ "${CACHE_BUCKET}/"
```

---

## 15. Troubleshooting

### VM boots but agent never starts

Check the serial console output — available without SSH:

```bash
gcloud compute instances get-serial-port-output INSTANCE_NAME \
  --project="${GCP_PROJECT}" --zone="${GCP_ZONE}"
```

Common causes:
- `thinker-ci-agent.service` failed — check `journalctl -u thinker-ci-agent`
- Network not ready before agent started — verify `After=network-online.target` in the unit
- Metadata server unreachable — ensure the VM is in a network with metadata access

### Agent cannot reach the Console

The VM has no external IP and reaches the internet through Cloud NAT.  Verify:

```bash
# From inside the VM via serial console or SSH (if temporarily enabled):
curl -v https://console.thinker-ci.io/api/v1/me/ \
  -H "Authorization: Token test"
```

If this fails, check:
1. Cloud NAT is configured for the subnetwork
2. The firewall allows egress on port 443
3. The Console URL in `GCE_RUNNER_CONSOLE_URL` is reachable from GCP

### Self-deletion fails

The agent's self-delete step calls `gcloud compute instances delete` using the VM's
service account.  This requires:

```
roles/compute.instanceAdmin    (scoped to the instance itself, not the whole project)
```

Or use a custom IAM binding:

```bash
gcloud projects add-iam-policy-binding "${GCP_PROJECT}" \
  --member="serviceAccount:thinker-ci-runner@${GCP_PROJECT}.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1" \
  --condition='resource.name.startsWith("projects/my-project/zones/us-central1-a/instances/thinker-ci-job")'
```

The condition restricts deletion to instances whose names start with the job prefix.

### Packer build fails — IAP tunnel not connecting

```bash
# Ensure the IAP API is enabled.
gcloud services enable iap.googleapis.com --project="${GCP_PROJECT}"

# Ensure the builder SA has the tunnelResourceAccessor role.
gcloud projects add-iam-policy-binding "${GCP_PROJECT}" \
  --member="serviceAccount:packer-builder@${GCP_PROJECT}.iam.gserviceaccount.com" \
  --role="roles/iap.tunnelResourceAccessor"

# Firewall: allow IAP's IP range to reach the builder on port 22.
gcloud compute firewall-rules create allow-iap-ssh \
  --project="${GCP_PROJECT}" \
  --network=ci-network \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=packer-builder
```

### Job VMs accumulating (not self-deleting)

Run the orphan reaper manually:

```bash
# In the Console environment:
python manage.py shell -c "
from apps.pipelines.tasks import reap_orphaned_gce_instances
reap_orphaned_gce_instances()
"
```

Or trigger the Celery task directly:

```bash
celery -A console call apps.pipelines.tasks.reap_orphaned_gce_instances
```
