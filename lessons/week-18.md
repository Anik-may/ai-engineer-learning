# Week 18: Fine-Tuning (Project 4, Part 1)

General knowledge. Fine-tuning libraries move quickly: treat the code below
as a pattern and check the current `transformers` and `peft` documentation
for exact arguments before running it.

## Goal

Fine-tune a small model on your dataset and beat the week 17 baselines.

## When to fine-tune

Try in this order. Stop at the first one that works.

| Step | Use when |
| --- | --- |
| 1. Better prompt | Always first |
| 2. Few-shot examples | The task needs a style or format |
| 3. RAG | The model lacks knowledge |
| 4. Fine-tuning | You need consistent behaviour, lower latency or cost, a narrow task, or a small model to match a big one |

Fine-tuning teaches **behaviour**, not facts. For facts, use RAG.

## Two fine-tuning routes for this project

| Route | Model | Result |
| --- | --- | --- |
| A. Encoder classifier | A BERT-style multilingual or Bangla encoder | Tiny, very fast, very cheap; best for pure classification |
| B. Small LLM with LoRA | A small open instruction-tuned model | Flexible; can output label plus reason |

Do A first. It usually wins for classification and trains on a free GPU in
minutes.

## Full fine-tuning versus LoRA

- **Full fine-tuning** updates every weight. Costly for large models; fine
  for small encoders.
- **LoRA** freezes the model and trains small low-rank matrices added to
  some layers. Trains well under 1% of the parameters, needs far less
  memory, and produces a small adapter file.
- **QLoRA** loads the base model in 4-bit precision and trains LoRA adapters
  on top, so larger models fit on small GPUs.

## Route A: encoder classifier

```python
import numpy as np
from datasets import load_dataset
from sklearn.metrics import accuracy_score, f1_score
from transformers import (AutoTokenizer, AutoModelForSequenceClassification,
                          TrainingArguments, Trainer, DataCollatorWithPadding)

name = "<multilingual-or-bangla-encoder>"
tok = AutoTokenizer.from_pretrained(name)
ds = load_dataset("<your-dataset>")
ds = ds.map(lambda b: tok(b["text"], truncation=True, max_length=128), batched=True)

model = AutoModelForSequenceClassification.from_pretrained(name, num_labels=5)

def metrics(p):
    preds = np.argmax(p.predictions, axis=1)
    return {"accuracy": accuracy_score(p.label_ids, preds),
            "macro_f1": f1_score(p.label_ids, preds, average="macro")}

args = TrainingArguments(
    output_dir="out",
    learning_rate=2e-5,
    per_device_train_batch_size=16,
    num_train_epochs=4,
    weight_decay=0.01,
    eval_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    metric_for_best_model="macro_f1",
)

trainer = Trainer(model=model, args=args,
                  train_dataset=ds["train"], eval_dataset=ds["validation"],
                  data_collator=DataCollatorWithPadding(tok),
                  compute_metrics=metrics)
trainer.train()
```

## Route B: LoRA on a small LLM (pattern)

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=16,                 # rank: capacity of the adapter
    lora_alpha=32,        # scaling
    lora_dropout=0.05,
    target_modules=["q_proj", "v_proj"],   # depends on the model architecture
    task_type="CAUSAL_LM",
)
model = get_peft_model(base_model, config)
model.print_trainable_parameters()
```

Format each example as a chat with the model's own chat template: system
instruction, user message, assistant label. Train with a supervised
fine-tuning trainer (for example `trl`'s `SFTTrainer`). Check the docs for
your model's target module names.

## Hyperparameters and what they do

| Parameter | Typical start | Too high | Too low |
| --- | --- | --- | --- |
| Learning rate | 2e-5 (encoder), 1e-4 to 2e-4 (LoRA) | Loss unstable | Learns slowly |
| Epochs | 3 to 5 | Overfits | Underfits |
| Batch size | 8 to 32 | Out of memory | Noisy, slow |
| LoRA rank `r` | 8 to 16 | Overfits, larger adapter | Cannot learn the task |
| Max length | 128 to 512 | Slow, memory | Truncates input |

Change one at a time and log every run.

## Experiment tracking

Record for each run: commit, dataset version, hyperparameters, validation
metrics, training time. Use MLflow or Weights & Biases, or a simple CSV.
Unlogged experiments cannot be compared or reproduced.

## Common failures

| Symptom | Likely cause |
| --- | --- |
| Validation loss rises while training loss falls | Overfitting: fewer epochs, more data, more dropout |
| Predicts one class always | Class imbalance or learning rate too high |
| Excellent validation, poor test | Leakage between splits |
| Out of memory | Lower batch size or max length; use gradient accumulation; QLoRA |
| LLM forgets general ability | Too many epochs or too high a rate; mix in general examples |

## Compute

Use the free GPUs on Google Colab or Kaggle. Save checkpoints to the Hub or
Drive because sessions disconnect.

## Do

1. Fine-tune the encoder (route A). Log three runs with different learning
   rates.
2. Report validation accuracy and macro F1 against the baselines.
3. Plot the confusion matrix; read 30 errors and categorise them.
4. Fix the top error category (usually bad labels or a missing class) and
   retrain.
5. Optional: route B with LoRA on a small LLM.
6. Evaluate **once** on the frozen test set.

## Checkpoint

- [ ] Fine-tuned model beats the classical and zero-shot baselines.
- [ ] Runs are logged and reproducible.
- [ ] You did an error analysis, not only a score.

## Interview questions

1. When would you fine-tune instead of using RAG?
2. What is LoRA and why is it cheaper?
3. How do you know your model is overfitting?
4. Why evaluate on the test set only once?
