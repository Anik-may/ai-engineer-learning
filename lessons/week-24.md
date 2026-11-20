# Week 24: Your Story, Behavioural Questions and ML Theory Review

## Goal

Tell the Omnichan story in five minutes with numbers, answer behavioural
questions with structure, and pass a quick ML theory screen.

## Part 1: the five-minute project walkthrough

Almost every interview starts with "tell me about a project". Prepare one
flagship answer (Omnichan) and two shorter ones (portfolio projects).

Structure:

| Part | Seconds | Content |
| --- | --- | --- |
| Context | 30 | What the product does and who uses it |
| Your role | 20 | What you personally built |
| Architecture | 60 | The main components and data flow |
| Hardest problem | 90 | One real difficulty, what you tried, what worked |
| Trade-offs | 45 | A decision you made and what you gave up |
| Results | 30 | Numbers |
| Next | 25 | What you would improve |

For Omnichan, good "hardest problem" candidates from your CV:

- usage-based billing that never loses or double-counts usage
  (idempotency keys, local save before sending, daily reconciliation),
- one agent answering across web, WhatsApp, Messenger and email,
- tenant isolation with a separate database per customer,
- handing off from the AI agent to a human in one shared inbox.

Pick the one you understand most deeply. Expect follow-ups three levels down.

Get the numbers now, from real data: tenants, conversations per day, share
resolved without a human, response time, cost per conversation. If you
cannot share exact figures, give ranges or relative changes. Do not invent.

Likely follow-ups to prepare:

1. Why LangGraph and not a simple chain?
2. How do you know the agent's answers are correct?
3. What happens when the model provider is down?
4. How do you stop one tenant seeing another's data?
5. What was your biggest mistake on this project?
6. What would break at ten times the traffic?

## Part 2: behavioural questions

Use **STAR**: Situation, Task, Action, Result. Keep to two minutes. Say "I",
not "we", for your own actions.

Prepare one true story for each:

| Theme | Example question |
| --- | --- |
| Failure | Tell me about a bug you caused in production |
| Conflict | A time you disagreed with a teammate or manager |
| Ownership | A time you went beyond your assigned task |
| Ambiguity | A time requirements were unclear |
| Learning | Something you learned quickly for a project |
| Deadline | A time you had to cut scope |
| Feedback | Criticism you received and what you changed |
| Impact | The work you are most proud of |

Also prepare:

- **Why this company?** One specific thing about their product.
- **Why leave your current job?** Positive framing: growth, scope. Never
  criticise your employer.
- **Salary expectation**: research the range; give a range; anchor on value.
- **Questions for them** (always ask): What does the first 90 days look
  like? How do you evaluate your AI features? What is the hardest technical
  problem the team has now? How are decisions made between product and
  engineering?

## Part 3: ML and LLM theory quick review

Answer each in under a minute.

Fundamentals:

1. Bias versus variance.
2. Overfitting and three ways to reduce it.
3. Precision, recall, F1.
4. Train, validation and test sets.
5. What a loss function and gradient descent are.
6. What an embedding is.
7. Cosine similarity and why it is used for retrieval.

LLMs:

8. How a transformer generates text.
9. What attention does.
10. Tokens and context windows.
11. Temperature and top-p.
12. Why models hallucinate and how you reduce it.
13. RAG versus fine-tuning.
14. What LoRA is.
15. What quantization is.

Systems:

16. How vector search works (approximate nearest neighbour, HNSW at a high
    level).
17. Hybrid search and reranking.
18. Function calling: how the model "calls" a tool.
19. Structured output and how to guarantee valid JSON.
20. Prompt injection and defences.
21. How you evaluate an LLM application.
22. How you reduce cost and latency.
23. What MCP is.
24. Agent versus workflow.

## Do

1. Write the Omnichan walkthrough; cut it until it fits five minutes.
2. Record it three times. Listen for filler and vague claims.
3. Write eight STAR stories as bullet points, not scripts.
4. Record answers to the 24 theory questions; redo any over one minute.
5. Write three questions to ask each company you apply to.

## Checkpoint

- [ ] Five-minute walkthrough recorded, with numbers.
- [ ] Eight STAR stories ready.
- [ ] All 24 theory answers under a minute each.
