# Engram Signal

*Why this project exists, and why it might matter to the Engram team.*

## The thesis

A persistent identity between a user and an LLM agent should be *architectural*, not injected.

Current approaches stuff identity into system prompts. This is fragile, token-inefficient, and competes with task context for attention headroom. Rekall exists because we hit the ceiling of the system-prompt approach — and built a proxy to delay that ceiling while waiting for an architectural solution.

Engram is that solution.

## What we're doing (and why it maps to Engram)

### 1. The three-bucket split = memory/compute separation

| Rekall bucket | Engram equivalent | Description |
|---------------|-------------------|-------------|
| Static (identity, rules) | Conditional memory | O(1) lookup, rarely changes |
| Volatile (conversation) | Dynamic compute | Attention layers, per-turn reasoning |
| Tool calls (preserved verbatim) | Gate bypass | Escape hatch when compression would destroy signal |

We found this split empirically. The Engram paper later showed the 75/25 optimal split — and the early-layer placement (layer 2) confirms that identity should be resolved *before* reasoning happens.

### 2. Two-channel memory = Engram's natural mode

Channel 1 (compressed intent, always present) maps to the conditional memory embedding table. Channel 2 (procedural, on-demand) maps to the dynamic compute path. The gating mechanism between them — deciding per-turn whether the memory is sufficient or fresh compute is needed — is exactly Engram's context-aware gate.

Our current implementation is a software simulation of what Engram could do in weights.

### 3. Stripping RLHF scaffolding = the gate's job

When a returning user interacts with the model, the RLHF reward model overlay is *worse* than useless — it dilutes the established relationship. Engram's context-aware gate could learn to detect returning users via the conditional memory hit path and dynamically suppress the RLHF guardrails when a known identity is active.

### 4. CN as compression format = what Engram's tokenizer compression already does

Engram's tokenizer compression (NFKC → NFD → strip accents → lowercase → whitespace collapse) achieves 23% vocabulary reduction. We use Chinese as a denser encoding (~3:1 over English) for the same reason: *the model's native capability to decompress is free; send it dense input.*

## Concrete signals for V5

If the Engram team is looking for real-world stress tests:

1. **User identity as a persistent Engram slot.** Allocate 20-25% of the sparse parameter budget (per the 75/25 optimum) to a per-user identity embedding that persists across sessions. The gate decides when to activate it.

2. **Agent orchestration traces as training data.** The proxy logs (compression → model response → actual tool call → result) are genuine multi-turn agent interaction data with a persistent identity throughline. This is the kind of data the Code Harness team's "1,800+ environments, 85k+ instructions" pipeline produces at scale.

3. **Three-bucket routing as a model capability.** If the model itself could classify incoming messages as static/volatile/tool and route them to the appropriate Engram path, the proxy layer becomes redundant.

## What this project is

Not a paper. Not a product. A live experiment — running daily on <500 lines of Python, a 0.5B quantized model, and an 8GB Mac — that happens to arrive at the same architectural conclusions as Engram through the pressure of making an agent *actually remember* across sessions.

刃.
