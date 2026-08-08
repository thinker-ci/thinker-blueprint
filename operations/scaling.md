# Scaling Guide

## Bottlenecks and remediation

### Web tier (Django/Gunicorn)

Scale horizontally. The web tier is stateless — all state lives in PostgreSQL. The Kubernetes HPA scales `thinker-console-web` between 2 and 10 replicas at 70% CPU utilization.

Session state is token-based with no sticky sessions required.

### Worker tier (Celery)

Scale horizontally. Workers are also stateless — they pull tasks from Redis and write results to PostgreSQL. The Kubernetes HPA scales `thinker-console-worker` between 2 and 20 replicas at 60% CPU.

Increase `--concurrency` per worker for I/O-bound workloads (log streaming). Keep it lower for CPU-bound workloads.

### Beat (scheduler)

**Do not scale horizontally.** Run exactly one Beat instance. Multiple Beat processes will fire duplicate periodic tasks. In Kubernetes, ensure the Beat deployment has `replicas: 1` and no HPA.

### Redis

For small-to-medium deployments, a single Redis instance is sufficient. For high availability:
- Use Redis Sentinel (managed: ElastiCache with Multi-AZ, Cloud Memorystore)
- Or Redis Cluster for very high throughput

### PostgreSQL

The `job_logs` table grows fastest. Recommendations:
1. Partition `job_logs` by `job_id` range (future work)
2. Add a background task to archive and delete logs older than 90 days
3. Use a read replica for log queries (the log viewer is read-heavy)

## Resource estimates

| Component | Baseline | Sustained (100 concurrent jobs) |
|---|---|---|
| Web pods | 2 × 256 Mi RAM | 4 × 256 Mi RAM |
| Worker pods | 2 × 512 Mi RAM | 10 × 512 Mi RAM |
| Redis | 512 Mi | 1 Gi |
| PostgreSQL | 2 vCPU / 4 Gi | 4 vCPU / 16 Gi |
