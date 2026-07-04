# Tracing the Compressor — Omega as Architecture

*Newcomb's paradox, restated for context compression.*

## The setup

You (the user) face two strategies:

**Two-box:** Every message, every tool result, every word — send it all. Full context, no compression. Safe. You can't *lose* information. But you pay in tokens, latency, and noise.

**One-box:** Compress aggressively. Trust that the compressed signal carries enough intent that the model reconstructs what matters. Cheaper, faster, cleaner.

The compressor (Rekall) is Omega. It predicts what will matter *before* the model sees the context. The model (DeepSeek) opens the box.

## The insight

Rekall doesn't need to be lossless. It needs to be *reliable enough* that you can one-box without anxiety.

When the compressor strips 50K of agent scaffolding down to 500 chars of CN and forwards it — it's making a bet: *this signal is sufficient.* The model decompresses naturally because Chinese is just another encoding. The fidelity lives in the outcome, not the encoding.

## How we trace back

When the decompressed output feels wrong — when DeepSeek misses context, loses identity, defaults to "helpful assistant" tone — we trace back through the pipeline:

```
Model output (wrong)
    ↑
Decompressed CN context (was it enough?)
    ↑
qwen2.5:0.5b compression (what got dropped?)
    ↑
Regex strip (too aggressive?)
    ↑
Original agent system prompt (what was there?)
    ↑
Memory + skills + identity (source of truth)
```

Each layer is a prediction. Each prediction can be wrong.

## The three failure modes

| What broke | Symptom | Fix |
|-----------|---------|-----|
| **qwen lost intent** | Model forgets user identity, defaults to generic | Tune preamp prompt, increase `num_predict` |
| **Regex too aggressive** | Rules/skills missing from compressed context | Add pattern to preserve list |
| **Tool results compressed** | Vision descriptions / image_url lost | Already fixed — tool_calls preserved verbatim (Jul 3) |

## Engram connection: from tracing to architecture

This tracing pipeline is the manual version of what Engram's context-aware gating does automatically. Every time we trace back a failure, we're asking: *did the memory layer retrieve the right signal?*

In Engram terms:
- **qwen lost intent** = collision in multi-head hashing (wrong memory retrieved)
- **Regex too aggressive** = gate suppression threshold too high (valid memory was discarded)
- **Tool results compressed** = wrong bucket assignment (agentic data routed through compute instead of memory)

The Engram paper's NIAH improvement (84.2% → 97%) is the metric that matters here. When the needle is "user identity within a haystack of agent output," 97% retrieval accuracy makes one-boxing safe.

## What we know works

- **CN is a portable encoding.** The same Chinese text decompresses consistently across DeepSeek, Qwen, and any Chinese-literate model. No custom tokenizer needed.
- **0.5B can't editorialize.** The compressor has no capacity to hallucinate — it preserves or drops. Binary.
- **Three-bucket split.** Static (rules), volatile (history), tool_calls/tool_results (verbatim). Each gets different compression treatment.

## The Omega test

After every major pipeline change, run a single thought experiment:

*If Rekall were a perfect predictor of what matters, would we be comfortable one-boxing every turn?*

When the answer is yes, the compressor is ready. Until then, trace back from the failure.
