# Week 22: AI System Design, Part 2: Agents, Gateways and More

## Goal

Three more complete designs and a bank of practice questions.

## Design 1: a multi-tenant customer support agent

You built this at Canduit. Use real experience.

Clarify: tenants, channels (web, WhatsApp, Messenger, email), volume, which
actions the agent may take, human handoff rules, languages.

```
Channels -> Ingest / normalise -> Conversation service
   -> Agent (LangGraph): intent -> retrieve -> tools -> respond
        |-> Tools (via MCP): CRM, orders, tickets
        |-> Knowledge base per tenant (vector store, tenant filter)
        |-> Handoff to human inbox
   -> Guardrails -> Reply -> Channel
Side: tracing, usage metering, billing, evaluation
```

Deep dives:

| Topic | Points |
| --- | --- |
| Tenant isolation | Tenant id on every query, cache key, trace and tool call; separate data stores or strict filters |
| State | Conversation state in a checkpointer keyed by conversation; resumable |
| Tool safety | Read tools free; write tools need confirmation or policy check; idempotency keys |
| Handoff | Triggers: low confidence, negative sentiment, explicit request, repeated failure; pass a summary to the human |
| Reliability | Retries with backoff, timeouts, fallback model, step limit |
| Async channels | Webhooks in, queue, worker, reply; deduplicate provider retries |
| Billing | Meter per conversation or token with idempotent usage events |

Evaluation: task success rate, tool-selection accuracy, handoff precision,
resolution without a human, cost per conversation, injection test set.

## Design 2: an LLM gateway

You built this in week 12.

```
Apps -> Gateway [auth -> rate limit -> cache -> router -> provider adapters]
            |-> usage log / cost         |-> fallback chain
            |-> guardrails (optional)    |-> streaming pass-through
```

Deep dives: virtual keys and budgets per tenant; routing by alias and by
policy; retries with backoff and jitter on 429 and 5xx; circuit breaker per
provider; cache safety; streaming without buffering; observability; high
availability (stateless gateway, Redis for limits and cache).

## Design 3: an agent platform with human approval

```
Request -> Planner -> Step executor (tools via MCP) -> Approval gate -> Continue
              ^                |                              |
              +---- state store / checkpoints ----------------+
```

Deep dives:

- **Durable execution**: checkpoint after every step so a run survives
  restarts and waits days for approval.
- **Approvals**: show the exact action and arguments; record who approved.
- **Idempotency**: every side effect has a key; resuming never repeats it.
- **Failure handling**: retry, compensate (undo), or escalate.
- **Limits**: steps, tokens, wall time and spend per run.
- **Audit**: full trace of decisions and tool calls.
- **Permissions**: the agent acts with the requesting user's rights, not a
  super-user's.

## Short designs to practise (10 minutes each)

1. Semantic search over 10 million products.
2. Document extraction: invoices to structured JSON with validation.
3. Text-to-SQL analytics assistant that must not run destructive queries.
4. Meeting summariser with speaker labels in Bangla and English.
5. Content moderation pipeline with a human review queue.
6. Voice agent: speech-to-text, LLM, text-to-speech, under 1 second.
7. Code review assistant for pull requests.
8. Recommendation of help articles from a support conversation.

## Trade-off phrases to have ready

| Decision | Trade-off |
| --- | --- |
| Bigger model | Quality up; cost and latency up |
| More retrieved chunks | Recall up; cost up, risk of distraction |
| Reranker | Precision up; adds latency |
| Agent versus fixed workflow | Flexibility up; predictability and testability down |
| Fine-tune versus prompt | Consistency and cost at scale; upfront effort and maintenance |
| Cache | Cost and latency down; staleness and leakage risk |
| Streaming | Perceived speed up; harder guardrails on partial output |
| Human in the loop | Safety up; throughput down |

A point worth making in every agent design: **use a fixed workflow where the
steps are known, and an agent only where they are not.** Deterministic code
is cheaper, faster and testable.

## Do

1. Deliver each of the three designs aloud in 40 minutes, recorded.
2. Do four of the short designs in 10 minutes each.
3. For each, write the three hardest trade-offs on one line each.
4. Do one full mock with another person or a recorded self-review.

## Checkpoint

- [ ] Three designs delivered within time, each with evaluation and
      failure handling.
- [ ] You can explain when not to use an agent.
