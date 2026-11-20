# Langfuse 06: Self-Hosting

Self-hosting is the main reason to choose Langfuse: traces contain customer
messages, and some customers require that data stays on your infrastructure.

## Local with Docker Compose (Verified)

```bash
git clone https://github.com/langfuse/langfuse.git
cd langfuse
docker compose up
```

Open http://localhost:3000 and create the first user with **Sign up**. There
is no default login.

Then point your app at it:

```bash
LANGFUSE_BASE_URL="http://localhost:3000"
LANGFUSE_PUBLIC_KEY="pk-lf-..."      # from the project you create in the UI
LANGFUSE_SECRET_KEY="sk-lf-..."
```

## What runs (Verified where noted)

| Service | Role |
| --- | --- |
| `langfuse-web` | UI and API, port 3000 (Verified) |
| `worker` | Background processing of incoming events (Verified) |
| ClickHouse | Analytical store for traces, observations, scores (Verified) |
| MinIO / S3 | Blob storage for events and media uploads, port 9090 (Verified) |
| PostgreSQL | Users, projects, prompts, settings (From memory) |
| Redis | Queue and cache (From memory) |
| `langfuse-ai-gateway` | Optional, beta, port 3031 (Verified) |

Read `docker-compose.yml` for the exact list in your version.

## Before exposing it to a network (Verified)

Every sensitive line in `docker-compose.yml` is marked `# CHANGEME`. Replace
all of them with long random values. This includes database passwords, the
encryption key and the auth secret.

## Running it on a server (Verified)

- Minimum for a VM: 4 cores, 16 GiB memory, about 100 GiB disk. Traces
  accumulate, so watch disk usage.
- Docker Compose has no high availability and no horizontal scaling. For that
  the docs recommend Kubernetes (there is a Helm chart).

## Production hardening checklist

1. Put it behind HTTPS with a reverse proxy (Nginx or Caddy).
2. Change every `CHANGEME` secret; store them in a secrets manager.
3. Do not expose database, ClickHouse, Redis or MinIO ports publicly.
4. Back up PostgreSQL and ClickHouse; test a restore.
5. Use managed services where you can (managed Postgres, S3 instead of MinIO).
6. Set data retention so disk does not fill.
7. Restrict sign-up, or configure single sign-on.
8. Monitor the containers and disk space.
9. Pin an image version and upgrade deliberately; read upgrade notes.

## Your app must survive Langfuse being down

- The SDK sends in the background, so a slow or unreachable Langfuse does not
  block requests.
- Prompts are cached client-side, so the app keeps working with the last
  fetched prompt.
- Still wrap your own explicit calls (`create_score`, `get_prompt` on first
  start) in error handling, and provide a fallback prompt.

## Cloud or self-hosted?

| Question | If yes |
| --- | --- |
| Must customer data stay on your servers or in your country? | Self-host |
| Do you have someone to run ClickHouse and backups? | Self-host is realistic |
| Small team, no ops time? | Cloud |
| Very high trace volume and cost-sensitive? | Self-host can be cheaper |

## Exercise

1. Run Langfuse locally with Docker Compose.
2. Point your agent at it and confirm traces arrive.
3. Stop the Langfuse containers and confirm your agent still answers.
4. Write a one-page deployment plan for a 4-core VM: domain, HTTPS, secrets,
   backups, retention.
