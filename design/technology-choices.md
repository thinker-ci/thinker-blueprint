# Technology Choices

## Language: Python 3.12

Python is the dominant language for DevOps tooling and infrastructure automation. Both the Docker SDK (`docker-py`) and the official Kubernetes client (`kubernetes`) are Python-native, making Python the natural choice for a platform that orchestrates containers.

## Framework: Django 5.x + Django REST Framework 3.x

See [ADR-001](decisions.md#adr-001-django-as-the-web-framework).

## Task queue: Celery 5.x + Redis 7

See [ADR-002](decisions.md#adr-002-celery--redis-for-async-task-processing).

## Database: PostgreSQL 16

See [ADR-003](decisions.md#adr-003-postgresql-as-the-sole-datastore).

## Container runtime: Docker 24+ / Kubernetes 1.28+

See [ADR-004](decisions.md#adr-004-docker--kubernetes-as-dual-runner-targets).

## Container image base: python:3.12-slim

Minimal Debian-based image. We install only `build-essential` and `libpq-dev` for compiling `psycopg2`. Total image size is approximately 180 MB.

## Web server: Gunicorn

Synchronous WSGI server with 4 worker processes per pod. For future WebSocket support (live log streaming), we will migrate to Uvicorn/ASGI — the `ASGI_APPLICATION` setting is already wired.

## Settings management: django-environ

Reads settings from environment variables and `.env` files. Avoids hardcoding secrets in code. See `.env.example` for the full variable list.

## CI/CD for this project: Thinker CI itself

Once the platform reaches MVP stability, all builds for `thinker-console` will be run through Thinker CI, dogfooding the product.
