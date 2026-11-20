# Week 14: Monitoring, Alerting and Phase 3 Wrap-Up

General knowledge.

## Goal

Know within minutes when your service is slow, failing or expensive, and
finish the phase 3 project.

## Three kinds of telemetry

| Kind | Answers | Tools |
| --- | --- | --- |
| Logs | What happened in this request? | CloudWatch Logs, Loki |
| Metrics | How is the system doing overall? | CloudWatch Metrics, Prometheus, Grafana |
| Traces | Where did the time go across services? | OpenTelemetry, LangSmith, Langfuse |

## The four golden signals, adapted for LLM services

| Signal | Classic | LLM addition |
| --- | --- | --- |
| Latency | Request duration p50, p95, p99 | Time to first token; per-step latency |
| Traffic | Requests per second | Tokens per minute |
| Errors | 5xx rate | Provider 429s and timeouts, tool failures, guardrail blocks, invalid structured output |
| Saturation | CPU, memory | Rate-limit headroom, queue length |

Add two that classic services do not have:

- **Cost**: per request, per tenant, per day.
- **Quality**: online evaluation scores and user feedback.

## Structured logging

Log JSON, one event per line, with the same fields every time:

```json
{"ts":"2026-11-20T10:15:00Z","level":"info","event":"chat_completed",
 "request_id":"r_91","trace_id":"t_5c","tenant_id":"t_42","model":"fast",
 "input_tokens":812,"output_tokens":164,"cost_usd":0.0009,
 "latency_ms":1840,"ttft_ms":420,"cache_hit":false,"status":"ok"}
```

Rules: never log secrets or full user messages in plain application logs;
always include a request id and the trace id so you can jump to the trace.

## Service level objectives

- **SLI**: a measured indicator, for example "share of requests answered in
  under 5 seconds".
- **SLO**: the target, for example "95% over 30 days".
- **Error budget**: what is left before you break the SLO. When it is nearly
  spent, stop shipping features and fix reliability.

Example SLOs for a support agent:

| SLI | SLO |
| --- | --- |
| Availability | 99.5% of requests return a non-error response |
| Time to first token | 95% under 1.5 s |
| Full response | 95% under 8 s |
| Groundedness (sampled) | 95% of sampled answers pass |

## Alerts

Alert on symptoms users feel, not on every cause.

| Alert | Condition |
| --- | --- |
| High error rate | Over 2% for 5 minutes |
| Slow responses | p95 over 8 s for 10 minutes |
| Provider down | Fallback rate over 20% |
| Cost spike | Hourly spend over twice the weekly average |
| Quality drop | Online groundedness below threshold for an hour |
| Runaway agent | A run exceeds the step or token limit |

Every alert needs an owner and a short runbook: what it means, first three
things to check, how to mitigate.

## Dashboards

One screen, top to bottom: traffic, error rate, latency percentiles, time to
first token, cost per hour, cache hit rate, fallback rate, quality score.

## Do

1. Emit structured JSON logs with request and trace ids.
2. Publish custom metrics: latency, time to first token, tokens, cost.
3. Build a CloudWatch (or Grafana) dashboard with the eight panels above.
4. Create three alarms: error rate, p95 latency, cost spike. Send them to
   email.
5. Trigger each alarm on purpose and confirm the email arrives.
6. Write a one-page runbook for "high error rate".
7. Write two SLOs in the README.

## Phase 3 wrap-up

- README for project 3: architecture diagram, deployed URL, pipeline badge,
  dashboard screenshot, cost and latency table, runbook link.
- Add AWS, Docker, CI/CD, Kubernetes (basics) and monitoring to your CV with
  the project link.
- Write an English post: "Deploying an LLM gateway on AWS: what it cost and
  what broke".
- **Start applying to remote and overseas roles from next week.**

## Checkpoint

- [ ] Dashboard shows live traffic.
- [ ] Alarms fire and reach you.
- [ ] Runbook and SLOs are in the repository.
- [ ] CV updated.

## Interview questions

1. Which metrics would you monitor for an LLM service that a normal API does
   not have?
2. What is an error budget?
3. Your p95 latency doubled overnight. How do you investigate?
4. How do you detect a quality regression in production?
