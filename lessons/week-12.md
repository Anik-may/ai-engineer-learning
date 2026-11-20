# Week 12: Deploy on AWS and Build the LLM Gateway (Project 3)

General knowledge; confirm AWS and LiteLLM details in current docs.

## Goal

Project 3 is live on AWS: one API in front of several model providers with
routing, fallback, caching and cost tracking.

## Target architecture

```
Internet
   |
[ Application Load Balancer ]  HTTPS, public subnets
   |
[ ECS Fargate service: gateway ]  private subnets, 2 tasks
   |            |              |
[ Redis ]   [ RDS Postgres ]  [ Secrets Manager ]
 cache        usage log         provider keys
   |
Model providers (two or more)
```

## ECS vocabulary

| Term | Meaning |
| --- | --- |
| Cluster | A group of services |
| Task definition | Blueprint: image, CPU, memory, env, secrets, roles |
| Task | A running copy of the task definition |
| Service | Keeps N tasks running and connects them to the load balancer |
| Fargate | AWS runs the machines; you only define tasks |

## Gateway features

| Feature | Detail |
| --- | --- |
| Unified API | One request format; the gateway translates per provider |
| Virtual keys | Each app or tenant gets its own key; provider keys stay in the gateway |
| Routing | Model alias such as `fast` or `smart` mapped to real models |
| Fallback | On error or timeout, try the next provider |
| Retries | Exponential backoff with jitter on 429 and 5xx |
| Caching | Exact-match in Redis, tenant in the key |
| Rate limits | Requests and tokens per minute, per key |
| Budgets | Stop or alert when a tenant exceeds spend |
| Usage log | Tenant, model, tokens, cost, latency, status per request |
| Streaming | Pass tokens through without buffering |

Build it on LiteLLM (it provides most of this) or write a thin FastAPI
service yourself to learn the internals. Doing the second for one provider
pair teaches more and is easier to explain in interviews.

## Streaming through a load balancer

- Use server-sent events and disable proxy buffering.
- Raise the load balancer idle timeout above your longest response.
- Measure time to first token separately from total time.

## Secrets

Reference Secrets Manager entries in the task definition so they are
injected as environment variables at start. Give the task's execution role
permission to read only those secrets.

## Infrastructure as code

Define the stack in Terraform or AWS CDK instead of clicking in the console.
Even a small Terraform file for the service, load balancer and security
groups is worth showing in the repository.

## Do

1. Build the gateway with two providers and model aliases `fast` and `smart`.
2. Add virtual keys, per-key rate limits and the usage log table.
3. Add fallback and retries; test by revoking one provider key.
4. Add the Redis cache.
5. Deploy to ECS Fargate behind a load balancer with HTTPS.
6. Point project 1 at the gateway instead of calling providers directly.
7. Write Terraform or CDK for the service.
8. Record: p95 latency added by the gateway, cache hit rate, cost per
   1,000 requests by alias.

## Checkpoint

- [ ] Public HTTPS URL answers a chat request with streaming.
- [ ] Killing one provider does not break requests.
- [ ] Usage table shows cost per tenant.
- [ ] Deployed from the pipeline, not by hand.

## Interview questions

1. Why put a gateway in front of LLM providers?
2. How do you handle a provider's 429 responses?
3. How do you track cost per tenant?
4. What breaks streaming behind a load balancer, and how do you fix it?
