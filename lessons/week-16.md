# Week 16: Transformers

General knowledge; the architecture basics are stable.

## Goal

Explain how a transformer turns text into the next token, and what the
generation settings you already use actually change.

## The pipeline

```
text -> tokens -> token ids -> embeddings (+ position)
     -> N transformer blocks [ self-attention -> feed-forward ]
     -> logits over the vocabulary -> probabilities -> next token
```

## Pieces

### Tokenisation

Text is split into sub-word tokens (BPE, WordPiece, SentencePiece). Cost and
context limits are counted in tokens, not characters. Bangla text often uses
more tokens per word than English, which affects cost and context budget.

### Embeddings

Each token id maps to a vector. Similar meanings end up near each other.
Position information is added because attention itself has no sense of order.

### Self-attention

For each token, the model asks "which other tokens matter for me?" and mixes
their information in.

- Each token produces a **query**, a **key** and a **value** vector.
- The match between one token's query and every token's key gives attention
  weights (after softmax).
- The output is the weighted sum of the values.

```
Attention(Q, K, V) = softmax( Q · Kᵀ / sqrt(d_k) ) · V
```

- **Multi-head**: several attention computations in parallel, each free to
  focus on a different relationship.
- **Causal mask**: in a decoder, a token may only attend to earlier tokens.

### The block

Attention, then a feed-forward network, each wrapped with a residual
connection and layer normalisation. Stack N blocks.

## Three families

| Family | Sees | Good at | Examples |
| --- | --- | --- | --- |
| Encoder-only | Whole input, both directions | Classification, embeddings, retrieval | BERT-style models |
| Decoder-only | Only the past | Text generation, chat, agents | GPT-style models, Llama |
| Encoder-decoder | Encoder reads, decoder writes | Translation, summarisation | T5-style models |

Your embedding model is an encoder. Your chat model is a decoder.

## Generation settings

| Setting | What it changes |
| --- | --- |
| Temperature | Sharpens (low) or flattens (high) the probability distribution before sampling |
| Top-p | Sample only from the smallest set of tokens whose probabilities sum to p |
| Top-k | Sample only from the k most likely tokens |
| Max tokens | Hard stop on output length |
| Stop sequences | Strings that end generation |

Temperature 0 picks the most likely token each time: more repeatable, not
guaranteed identical.

## Why context windows cost what they cost

Attention compares every token with every other token, so compute grows
roughly with the square of the sequence length. Long prompts are slow and
expensive, which is why retrieval, summarising history and prompt caching
matter. Inference servers cache the keys and values of earlier tokens (the
KV cache) so each new token does not recompute the whole prefix.

## How LLMs are trained, in three stages

1. **Pre-training**: predict the next token on huge text. Learns language
   and facts.
2. **Fine-tuning / instruction tuning**: train on prompt and response pairs
   so it follows instructions.
3. **Alignment and preference tuning**: shape behaviour with human or model
   feedback.

## Do

1. Tokenise the same sentence in English and Bangla with a Hugging Face
   tokenizer; compare token counts.
2. Implement scaled dot-product attention in about 15 lines of PyTorch and
   check the output shape.
3. Follow Karpathy's "Let's build GPT" video and train a tiny character
   model.
4. Generate from one prompt at temperature 0, 0.7 and 1.5; describe the
   difference.
5. Write a one-page explanation of attention in your own words, with a small
   worked example.

## Checkpoint

- [ ] You can draw the pipeline from text to next token.
- [ ] You can explain query, key and value.
- [ ] You can say when to use an encoder and when a decoder.

## Interview questions

1. Explain self-attention to a backend engineer.
2. Why do transformers need positional information?
3. What is the difference between BERT-style and GPT-style models?
4. What do temperature and top-p do?
5. Why is a long prompt more expensive than a short one?
