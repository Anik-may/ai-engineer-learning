# Week 17: The Hugging Face Ecosystem and Your Dataset

General knowledge; library APIs change, so check the current Hugging Face
docs. Model names are examples to look up, not guarantees.

## Goal

Run and compare open models on a Bangla task, and build the labelled dataset
for project 4.

## The pieces

| Piece | What it is |
| --- | --- |
| Hub | Website hosting models, datasets and demos |
| `transformers` | Load and run models |
| `tokenizers` | Fast tokenisation |
| `datasets` | Load, process and share datasets |
| `evaluate` | Standard metrics |
| `peft` | Parameter-efficient fine-tuning (LoRA) |
| `accelerate` | Device placement and mixed precision |
| Model card | The README of a model: intended use, data, limits, licence |

Always read the model card and **licence** before using a model commercially.

## Two ways to run a model

Pipeline, quickest:

```python
from transformers import pipeline

clf = pipeline("text-classification", model="<a-sentiment-model>")
print(clf("সেবাটি খুব ভালো ছিল"))
```

Manual, for control:

```python
import torch
from transformers import AutoTokenizer, AutoModelForSequenceClassification

name = "<a-sentiment-model>"
tok = AutoTokenizer.from_pretrained(name)
model = AutoModelForSequenceClassification.from_pretrained(name)

batch = tok(["সেবাটি খুব ভালো ছিল"], return_tensors="pt", padding=True, truncation=True)
with torch.no_grad():
    logits = model(**batch).logits
print(logits.softmax(dim=-1))
```

## Classification metrics

| Metric | Meaning | Use when |
| --- | --- | --- |
| Accuracy | Share correct | Classes are balanced |
| Precision | Of predicted positives, share truly positive | False alarms are costly |
| Recall | Of true positives, share found | Misses are costly |
| F1 | Balance of precision and recall | Classes are imbalanced |
| Macro F1 | Average F1 across classes, equal weight | Rare classes matter |
| Confusion matrix | Counts of true versus predicted class | Finding which classes get mixed up |

With imbalanced intents (most messages are "order status"), accuracy
misleads. Report macro F1 and the confusion matrix.

## Building the dataset for project 4

Task: classify a customer message into an **intent** (order status, refund,
complaint, product question, other) or a **sentiment**.

1. **Define labels** with one-sentence definitions and two examples each.
   Ambiguous labels ruin a dataset.
2. **Collect** 1,000 to 3,000 messages. Use public Bangla review or support
   data, or write and paraphrase your own. Do not use real customer messages
   from your employer without permission.
3. **Label** with an LLM first, then review.
4. **Verify by hand** at least 200. Measure how often you disagree with the
   LLM labels. Fix the label definitions and relabel if it is high.
5. **Split** 80 / 10 / 10 into train, validation, test, stratified by label.
   Make sure paraphrases of the same message do not land in different splits.
6. **Freeze the test set.** Do not look at it while tuning.
7. **Document**: source, label definitions, class counts, known problems.

Load and split:

```python
from datasets import load_dataset

ds = load_dataset("csv", data_files="intents.csv")["train"]
ds = ds.class_encode_column("label")
split = ds.train_test_split(test_size=0.2, stratify_by_column="label", seed=0)
```

## Compare three baselines before any fine-tuning

| Baseline | How |
| --- | --- |
| API model, zero-shot | Prompt with the label definitions |
| Open multilingual encoder | Look for a model fine-tuned for a similar task |
| Classical | TF-IDF + logistic regression (scikit-learn) |

The classical baseline is often surprisingly strong and is the number your
fine-tuned model has to beat.

## Do

1. Run three open models on 20 Bangla sentences; compare outputs.
2. Write label definitions.
3. Build and verify the dataset; record class counts and label agreement.
4. Compute accuracy and macro F1 for the three baselines on the validation
   set.
5. Push the dataset to a private Hub repository with a dataset card.

## Resource

The free Hugging Face LLM course (first four chapters this week).

## Checkpoint

- [ ] Dataset with train, validation and test splits exists.
- [ ] At least 200 labels verified by hand.
- [ ] Baseline table filled in.

## Interview questions

1. Why is accuracy misleading on imbalanced data?
2. How do you check the quality of LLM-generated labels?
3. What is data leakage between splits and how do you prevent it?
4. What do you read on a model card before using a model?
