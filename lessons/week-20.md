# Week 20: Publish Project 4 and Close Phase 4

## Goal

Project 4 is public and your CV shows model-level ML with evidence.

## Publish the model

1. Push the model to the Hugging Face Hub
   (`trainer.push_to_hub()` or `huggingface-cli upload`).
2. Write the **model card**. This is what reviewers read.

Model card sections:

| Section | Content |
| --- | --- |
| Summary | One sentence: what it does and for which language |
| Intended use | The task; who should use it |
| Out of scope | What it must not be used for |
| Training data | Source, size, label definitions, how labels were verified |
| Training procedure | Base model, method (full or LoRA), key hyperparameters |
| Evaluation | Test-set metrics, confusion matrix, baseline comparison |
| Limitations and bias | Where it fails: dialects, code-mixed text, short messages |
| How to use | A five-line code example |
| Licence | Yours, and compatible with the base model's |

## Publish the repository

README:

1. What and why, in three sentences.
2. The comparison table from week 19.
3. Dataset: how it was built and verified.
4. How to reproduce: one command to train, one to evaluate.
5. Error analysis: three failure categories with examples.
6. How to run the API with Docker.
7. Links: model on the Hub, dataset, blog post.

Reproducibility checklist:

- [ ] Fixed random seeds.
- [ ] Pinned dependency versions.
- [ ] Training script takes a config file.
- [ ] Dataset version recorded.
- [ ] Test set never used for tuning.

## Connect it to project 1

Use the classifier as the router in your RAG assistant or gateway: classify
intent first, then choose the path (FAQ answer, order lookup tool, human
handoff). Measure the cost saved compared with using an LLM call for routing.
This turns two separate projects into one system and is a strong interview
story.

## Write the post

"When fine-tuning a small model beats calling an API", 500 to 700 words:

1. The task and why it matters in a support product.
2. The four options and the table.
3. One surprise from the error analysis.
4. What you would ship and why.

## Update the CV

Skills: add PyTorch, Hugging Face Transformers, fine-tuning (LoRA),
quantization, model serving.

Project entry, pattern to follow:

> **Bangla intent classifier** | PyTorch, Hugging Face, ONNX, FastAPI
> Fine-tuned a multilingual encoder on a 2,400-message dataset I built and
> verified; reached X macro F1 against Y for a zero-shot API model, at Z ms
> p95 on CPU and a fraction of the cost per 1,000 requests.

Replace X, Y and Z with your measured numbers. Never estimate them.

## Phase 4 self-test

Answer each aloud in under two minutes, without notes:

1. Walk through a training loop.
2. What is attention?
3. Encoder versus decoder models.
4. Fine-tuning versus RAG versus prompting: when each?
5. What is LoRA?
6. What is quantization and what does it trade?
7. Precision, recall, F1: when does each matter?
8. How did you build and validate your dataset?
9. What were your model's main failure cases?
10. How would you serve it at scale?

Any answer you cannot give means going back to that week's lesson.

## Checkpoint

- [ ] Model on the Hub with a full model card.
- [ ] Repository public and reproducible.
- [ ] Classifier wired into project 1 as a router.
- [ ] Post published.
- [ ] CV updated with measured numbers.
