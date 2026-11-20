# Week 19: Quantization and Local Serving (Project 4, Part 2)

General knowledge. Serving tools change quickly; confirm commands in the
Ollama, vLLM and llama.cpp documentation.

## Goal

Make your fine-tuned model small and fast, serve it behind an API, and
compare it fairly with an API model.

## Quantization

Model weights are normally 32-bit or 16-bit floats. Quantization stores them
with fewer bits.

| Precision | Size vs 16-bit | Quality | Use |
| --- | --- | --- | --- |
| FP16 | 1× | Reference | GPU serving |
| 8-bit | about ½ | Usually near-identical | Safe default |
| 4-bit | about ¼ | Small loss, task-dependent | Laptops, small GPUs |

- **Post-training quantization**: convert after training. Simple.
- **Quantization-aware training**: simulate low precision during training.
  Better quality, more work.
- Always **measure** accuracy after quantizing. Do not assume.

Related ways to shrink a model:

| Technique | Idea |
| --- | --- |
| Pruning | Remove weights that matter least |
| Distillation | Train a small "student" to imitate a large "teacher" |
| Smaller architecture | Pick a smaller model and fine-tune it |

## Formats and runtimes

| Format / runtime | Used for |
| --- | --- |
| `safetensors` | Standard safe weight format on the Hub |
| GGUF + llama.cpp | CPU and laptop inference; many quantization levels |
| Ollama | Easy local model runner built on llama.cpp; simple HTTP API |
| vLLM | High-throughput GPU server with an OpenAI-compatible API |
| ONNX Runtime | Portable optimised inference, good for encoders |
| TensorRT | NVIDIA-specific optimisation |

Which to use:

- Encoder classifier → ONNX Runtime (CPU is fine) behind FastAPI.
- Small LLM on your laptop → Ollama.
- Small LLM on a GPU server with many users → vLLM.

## Serving concepts

| Term | Meaning |
| --- | --- |
| Throughput | Requests or tokens per second across all users |
| Latency | Time for one request |
| Time to first token | Delay before streaming starts |
| Batching | Processing several requests together for throughput |
| Continuous batching | Adding new requests into a running batch (vLLM does this) |
| KV cache | Reusing computation for earlier tokens |
| Cold start | Time to load the model into memory |

Throughput and latency trade against each other: bigger batches raise
throughput and can raise latency.

## Serve the classifier with FastAPI

```python
from fastapi import FastAPI
from pydantic import BaseModel
from transformers import pipeline

app = FastAPI()
clf = pipeline("text-classification", model="./out/best")   # loaded once at start

class Req(BaseModel):
    text: str

@app.post("/classify")
def classify(req: Req):
    r = clf(req.text, truncation=True)[0]
    return {"label": r["label"], "score": float(r["score"])}

@app.get("/health")
def health():
    return {"ok": True}
```

Load the model once at startup, never per request.

## The comparison that makes the project

Run all four options on the **same frozen test set**:

| Option | Macro F1 | p50 latency | p95 latency | Cost per 1,000 requests | Model size |
| --- | --- | --- | --- | --- | --- |
| API model, zero-shot | | | | | n/a |
| Open model, not fine-tuned | | | | | |
| Fine-tuned | | | | | |
| Fine-tuned + quantized | | | | | |

For local models, estimate cost from the hourly price of the machine divided
by requests per hour at your measured throughput. State your assumptions.

Measure latency properly: warm up first, run at least 200 requests, report
percentiles, and state the hardware.

## Do

1. Export the encoder to ONNX and quantize to 8-bit. Re-measure macro F1.
2. Serve it with FastAPI in Docker.
3. If you did route B: convert to GGUF, run in Ollama, call its API.
4. Fill in the comparison table.
5. Load-test the endpoint (for example with `hey` or `locust`) and record
   throughput.
6. Decide, in two sentences, which option you would ship and why.

## Checkpoint

- [ ] Quantized model's accuracy loss is measured and stated.
- [ ] Endpoint runs in Docker.
- [ ] Comparison table is complete with hardware noted.

## Interview questions

1. What is quantization and what does it cost you?
2. When does a small fine-tuned model beat a large API model?
3. What is the difference between throughput and latency?
4. How would you serve an LLM to 100 concurrent users?
