# Deployment Guide

## Prerequisites

- Docker 24+ and Docker Compose v2
- Or: a Kubernetes 1.28+ cluster with `kubectl` configured
- A PostgreSQL 16 instance (provided by compose, or external)
- A Redis 7 instance (provided by compose, or external)

---

## Local development

```bash
git clone https://github.com/thinker-ci/thinker-console.git
cd thinker-console
cp .env.example .env
# Edit .env — set SECRET_KEY at minimum

# Start all services (postgres + redis + django dev server + celery worker)
docker compose -f docker/docker-compose.dev.yml up

# In a separate terminal, run migrations
docker compose -f docker/docker-compose.dev.yml exec web python manage.py migrate

# Create a superuser
docker compose -f docker/docker-compose.dev.yml exec web python manage.py createsuperuser
```

API is now available at http://localhost:8000/api/v1/

---

## Production (Docker Compose)

```bash
cp .env.example .env
# Set SECRET_KEY, DATABASE_URL, REDIS_URL, ALLOWED_HOSTS, POSTGRES_PASSWORD

docker compose -f docker/docker-compose.yml up -d
docker compose -f docker/docker-compose.yml exec web python manage.py migrate
docker compose -f docker/docker-compose.yml exec web python manage.py createsuperuser
```

---

## Production (Kubernetes)

```bash
# 1. Create the namespace
kubectl apply -f k8s/namespace.yaml

# 2. Create the secrets (substitute real values)
kubectl -n thinker-ci create secret generic thinker-ci-secrets \
  --from-literal=SECRET_KEY='<your-secret-key>' \
  --from-literal=DATABASE_URL='postgres://thinker:pw@postgres:5432/thinker_ci' \
  --from-literal=REDIS_URL='redis://redis:6379/0'

# 3. Apply remaining manifests
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# 4. Run migrations as a one-off job
kubectl -n thinker-ci run migrate --image=ghcr.io/thinker-ci/thinker-console:latest \
  --restart=Never --env-from=configmap/thinker-ci-config --env-from=secret/thinker-ci-secrets \
  -- python manage.py migrate
```

---

## Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `SECRET_KEY` | Yes | — | Django secret key |
| `DEBUG` | No | `false` | Enable debug mode |
| `ALLOWED_HOSTS` | Yes | — | Comma-separated hostnames |
| `DATABASE_URL` | Yes | SQLite | PostgreSQL connection string |
| `REDIS_URL` | Yes | localhost | Redis connection string |
| `DOCKER_HOST` | No | `unix:///var/run/docker.sock` | Docker daemon socket |
| `K8S_NAMESPACE` | No | `thinker-ci` | Kubernetes namespace for job pods |
| `MAX_CONCURRENT_JOBS` | No | `10` | Global job concurrency cap |
| `JOB_TIMEOUT_SECONDS` | No | `3600` | Hard timeout per job container |
