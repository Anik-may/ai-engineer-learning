# LangSmith 07: Production Monitoring, Feedback and Prompts

Most of this file is **From memory**. The concepts are stable; confirm exact
method names and UI locations in the docs before using them.

## What to watch in production

| Signal | Why |
| --- | --- |
| Trace volume | Traffic, and whether tracing is working |
| Error rate | Failed model calls, tool errors, timeouts |
| Latency p50 / p95 / p99 | User experience; p95 matters more than the average |
| Time to first token | Perceived speed for streamed replies |
| Tokens and cost per trace | Budget; catches runaway loops |
| Feedback scores | Real quality as users see it |
| Tool-call failure rate | Agent reliability |

LangSmith provides built-in dashboards per project and lets you build custom
charts grouped by metadata (for example cost per tenant, latency per model).

## User feedback

Attach a score to the run that produced the answer.

```python
from langsmith import Client
client = Client()

client.create_feedback(
    run_id=run_id,              # id of the root run for that answer
    key="user_thumb",
    score=1,                    # 1 = up, 0 = down
    comment="Solved my problem",
)
```

To know the `run_id`, generate one yourself and pass it in the config when
you invoke the chain (Verified: a custom `run_id` set at the root is used as
the trace id), then store it with the chat message so the thumbs button can
send it back.

```python
import uuid
run_id = uuid.uuid4()
graph.invoke(inputs, config={"run_id": run_id})
```

## Online evaluation

Evaluators that run automatically on production traces, with no reference
answer available.

- Configure in the UI on a tracing project: pick a filter (for example root
  runs with tag `prod`), a sampling rate, and an LLM-judge or code evaluator.
- Scores are attached to the traces as feedback.
- Good online checks: faithfulness to retrieved context, answer relevance,
  toxicity, "did the bot refuse correctly", language matches the user's.
- Sample (for example 5% to 10%) to control cost.

## Automation rules

Rules act on traces that match a filter:

- add them to a dataset (for example all thumbs-down traces),
- add them to an annotation queue for human review,
- run an online evaluator,
- trigger a webhook.

A useful loop: thumbs-down trace → annotation queue → human writes the correct
answer → example added to the regression dataset.

## Annotation queues

A review inbox. Reviewers see a trace, score it against a rubric and can edit
the output into a reference answer. Use it to build datasets and to check
whether your LLM judge agrees with humans.

## Alerts

Set alerts on error rate, latency or a feedback score dropping, delivered by
webhook or a pager integration.

## Prompt management

Store prompts in LangSmith instead of in code so they can change without a
deploy and so every trace records which prompt version produced it.

```python
from langsmith import Client
client = Client()

prompt = client.pull_prompt("support-system-prompt")          # latest
prompt = client.pull_prompt("support-system-prompt:prod")     # a tagged version
client.push_prompt("support-system-prompt", object=my_prompt_template)
```

- Each save creates a commit; tags such as `prod` and `staging` point at
  commits.
- The Playground lets you run a prompt against a dataset and compare versions
  side by side before promoting one.

## Production checklist

1. Separate projects for dev, staging and production.
2. `tenant_id`, `channel`, `app_version`, `prompt_version` in metadata.
3. Personal data masked before it leaves your servers.
4. Sampling set if volume is high.
5. Thumbs up and down wired to `create_feedback`.
6. One online evaluator for faithfulness on a sample.
7. Rule: thumbs-down traces go to an annotation queue.
8. Alert on error rate and p95 latency.
9. Tracing failures must never break the request: LangSmith sends traces in
   the background, but wrap your own feedback calls in try/except.

## Exercise

1. Add thumbs up and down to your chat widget and send feedback to LangSmith.
2. Create an online faithfulness evaluator on 10% of production-tagged traces.
3. Create a rule that sends thumbs-down traces to an annotation queue.
4. Review 20 of them and add the corrected ones to your regression dataset.
