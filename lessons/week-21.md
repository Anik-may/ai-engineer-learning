# Week 21: AI System Design, Part 1: The Method and RAG

Phase 5 begins: interviews. General knowledge of how these interviews run.

## Goal

Have a repeatable method for any AI system design question, and a full
answer for "design a RAG system".

## The method: seven steps, 40 minutes

| Step | Minutes | What you do |
| --- | --- | --- |
| 1. Clarify | 5 | Users, scale, languages, latency, accuracy bar, data sensitivity, budget |
| 2. Define success | 3 | Quality metric, latency target, cost target |
| 3. High-level design | 7 | Draw the boxes and the data flow |
| 4. Deep dive | 12 | The two or three hardest parts |
| 5. Evaluation | 5 | Offline dataset, online monitoring, feedback loop |
| 6. Reliability, safety, cost | 5 | Failure modes, guardrails, scaling, cost levers |
| 7. Trade-offs and next steps | 3 | What you would do with more time |

Rules:

- Ask before you draw. Wrong assumptions sink the answer.
- State numbers, even rough ones, and say they are assumptions.
- Say the simple design first, then improve it.
- Always include evaluation. Most candidates forget it; it is what
  distinguishes an AI engineer from a backend engineer.
- Name trade-offs out loud: "this raises recall and costs latency".

## Clarifying questions to memorise

1. Who uses it and how many requests per day?
2. What languages?
3. What is an acceptable response time?
4. What happens when the answer is wrong? (Sets the accuracy bar and the
   need for human review.)
5. How large is the data and how often does it change?
6. Is the data sensitive? Per-user permissions?
7. Is there a budget per request?
8. Must it cite sources?

## Worked example: design a RAG system for company documents

### Requirements (assumed after asking)

10,000 employees, 500,000 documents, updated daily, English and Bangla,
answers in under 5 seconds with citations, documents have per-department
access rules.

### Ingestion pipeline

```
Sources -> Parse -> Clean -> Chunk -> Embed -> Vector store (+ metadata + ACL)
                                   \-> Keyword index (BM25)
```

Deep-dive points:

- **Parsing**: PDFs with tables and scans need layout-aware parsing or OCR.
- **Chunking**: by structure (headings, paragraphs) with overlap; keep
  title and section path in each chunk. Chunk size is a tunable trade-off:
  small chunks retrieve precisely but lose context.
- **Metadata**: source, date, department, access list, language.
- **Incremental updates**: hash documents; re-embed only what changed;
  delete removed documents.
- **Embedding model**: must handle Bangla; test it on your own queries.

### Query pipeline

```
Query -> (rewrite) -> Hybrid retrieve (vector + BM25, ACL filter)
      -> Rerank top 50 to top 5 -> Build prompt -> LLM -> Answer + citations
      -> Guardrail checks -> User
```

Deep-dive points:

- **Access control at retrieval time**: filter by the user's permissions in
  the vector query. Never retrieve first and filter after generation.
- **Hybrid search**: vector for meaning, keyword for exact terms such as ids
  and product names.
- **Reranking**: a cross-encoder reorders candidates; large quality gain for
  moderate latency.
- **Query rewriting**: resolve follow-up questions using chat history.
- **"I don't know"**: instruct the model to refuse when the context lacks
  the answer; test this explicitly.
- **Citations**: return chunk ids with the answer and verify they exist.

### Evaluation

- Offline: dataset of question, correct document, reference answer.
  Retrieval hit rate and MRR; faithfulness and correctness by a validated
  judge.
- Online: sampled faithfulness scoring, thumbs feedback, "no answer" rate.
- Loop: failures go to an annotation queue and then into the dataset.

### Reliability, safety, cost

- Timeouts and fallback model; cached answers for repeated questions (key
  includes the user's permission group).
- Prompt-injection defence for document content; PII handling in logs.
- Cost levers: smaller model for easy questions, fewer and shorter chunks,
  prompt caching, reranker to cut context size.
- Latency budget: retrieval 200 ms, rerank 300 ms, generation the rest;
  stream the answer.

### Trade-offs to mention

Chunk size, top-k, reranker on or off, one index versus per-department
indexes, freshness versus indexing cost.

## Do

1. Answer this question aloud in 40 minutes, recording yourself. Draw on
   paper or a whiteboard tool.
2. Watch the recording. Note where you were vague or forgot evaluation.
3. Repeat once with a changed constraint: "answers must be under 1 second".
4. Write the design as a one-page document.

## Checkpoint

- [ ] You can list the seven steps from memory.
- [ ] You delivered the RAG design in 40 minutes, including evaluation.
