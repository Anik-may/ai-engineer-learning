# Week 8: Model Routing, Caching and Write-Up

Phase 2, final week. General knowledge, not checked against docs today;
confirm library APIs before use.

## Goal

Cut the cost and latency of project 1 without losing quality, prove it with
numbers, and publish both phase 2 projects.

## Concepts

### Model routing

Not every request needs your strongest model.

| Strategy | How it decides | Good for |
| --- | --- | --- |
| Rule-based | Length, keywords, intent label, tenant plan | Simple, predictable |
| Classifier | A small model labels difficulty | More accurate, small extra cost |
| Cascade | Try the cheap model; escalate if confidence is low or a check fails | Best savings; adds latency on escalations |
| Fallback | Second provider when the first errors or times out | Reliability, not cost |

A cascade needs an escalation signal: a failed JSON validation, a low
self-reported confidence, a failed groundedness check, or "I don't know".

### Caching

| Type | Key | Hit when | Risk |
| --- | --- | --- | --- |
| Exact-match | Hash of model + full prompt | Identical request | Low |
| Semantic | Embedding of the question | Similar meaning above a threshold | Wrong answer for a near-but-different question |
| Provider prompt caching | Stable prefix of the prompt | Same system prompt and tools | None for correctness; saves input cost |
| Retrieval cache | Query text | Same query | Stale documents |

Rules:

- Never share a cache across tenants or users when answers depend on their
  data. Put tenant id (and user id if needed) in the key.
- Do not cache answers that depend on live data (order status).
- Set a time-to-live and invalidate when documents change.
- For semantic caching, start with a high similarity threshold and measure
  the wrong-hit rate on your dataset.
- For provider prompt caching, keep the stable parts (system prompt, tool
  definitions) at the start and the changing parts at the end.

### An LLM gateway

One internal API in front of all providers: routing, fallback, caching, rate
limits, cost tracking and key management in one place. LiteLLM is a common
open-source choice. You will build one in week 12.

## Do

1. Take the 100-question dataset from week 3.
2. Record the baseline: quality score, average cost per request, p50 and p95
   latency, all on the strong model.
3. Add a rule-based router: short factual questions go to a small model.
4. Run the dataset again. Fill in this table:

   | Setup | Quality | Cost / request | p95 latency |
   | --- | --- | --- | --- |
   | Strong model only | | | |
   | Small model only | | | |
   | Rule-based router | | | |
   | Cascade | | | |
   | Router + exact cache | | | |

5. Add an exact-match cache in Redis with tenant id in the key. Measure the
   hit rate on a replay of repeated questions.
6. Try a semantic cache. Count wrong hits. Decide whether to keep it.
7. Add a fallback provider and test it by breaking the primary key.

## Write-up

Finish both READMEs (project 1 and project 2):

- what it does, in two sentences,
- architecture diagram,
- how to run it,
- evaluation table with before and after,
- cost and latency table from this week,
- security results from week 6,
- limitations, honestly.

Publish one English post, 400 to 600 words: "What I learned measuring my RAG
agent". Lead with one number.

## Checkpoint

- [ ] Router cuts cost with a quality drop you can state.
- [ ] Cache hit rate measured; tenant isolation tested.
- [ ] Fallback works when the primary provider fails.
- [ ] Both repositories public with complete READMEs.
- [ ] Post published.

## Interview questions

1. How would you cut LLM cost by half without hurting quality?
2. When is semantic caching dangerous?
3. How does a cascade decide to escalate?
4. What belongs in a cache key for a multi-tenant app?
