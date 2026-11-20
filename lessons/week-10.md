# Week 10: Containers Done Properly

General knowledge; confirm tool versions in their docs.

## Goal

Turn your project into production-quality images and a one-command local
stack.

## A good Python Dockerfile

```dockerfile
# ---- build stage ----
FROM python:3.12-slim AS build
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# ---- runtime stage ----
FROM python:3.12-slim
WORKDIR /app
RUN useradd --create-home appuser
COPY --from=build /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8000/health')"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Why each choice:

| Choice | Reason |
| --- | --- |
| Multi-stage build | Smaller image, no build tools in production |
| Slim base image | Smaller and fewer vulnerabilities |
| Copy lock file before code | Dependency layer is cached between builds |
| Non-root user | Limits damage if the container is compromised |
| `PYTHONUNBUFFERED=1` | Logs appear immediately |
| Health check | Orchestrator can restart a dead container |
| Pinned versions | Reproducible builds |

Add a `.dockerignore`: `.git`, `.venv`, `__pycache__`, `.env`, `tests`,
`data`.

## Never bake secrets into an image

Secrets come from environment variables injected at runtime (from Secrets
Manager in AWS). Check with `docker history` that no key is in a layer.

## Docker Compose for local development

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }
      qdrant: { condition: service_started }

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_PASSWORD: dev
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s

  redis:
    image: redis:7

  qdrant:
    image: qdrant/qdrant
    volumes: ["qdrant:/qdrant/storage"]

volumes:
  pgdata:
  qdrant:
```

## The twelve-factor ideas that matter here

1. Config in environment variables.
2. Stateless processes; state in backing services.
3. Logs to stdout as a stream.
4. Same image in dev, staging and production; only config differs.
5. Fast startup and graceful shutdown (handle SIGTERM, finish in-flight
   requests, flush traces).

## AI-specific points

- Add `/health` (process alive) and `/ready` (database, vector store and
  model provider reachable).
- Streaming responses need a server and proxy that do not buffer.
- Do not download models at container start in production; bake them into
  the image or mount them.
- Set timeouts on every outbound model call.

## Do

1. Write the multi-stage Dockerfile for project 1. Record the image size
   before and after.
2. Run as non-root and confirm with `docker exec ... whoami`.
3. Write the Compose file; `docker compose up` must bring up everything.
4. Add `/health` and `/ready`.
5. Scan the image for vulnerabilities (`docker scout` or `trivy`).
6. Push the image to ECR.

## Checkpoint

- [ ] `docker compose up` starts the whole stack from a clean clone.
- [ ] Image under about 300 MB and runs as non-root.
- [ ] No secrets in the image.
- [ ] Image is in ECR.

## Interview questions

1. Why a multi-stage build?
2. What is the difference between a liveness and a readiness check?
3. How do you pass secrets to a container?
4. What happens to in-flight requests when a container is stopped?
