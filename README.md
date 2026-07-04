# Rekall — the blade that remembers

A context compression engine for LLM agent pipelines. Strips brand scaffolding, RLHF noise, and tool schema bloat from agent system prompts, then compresses the remaining signal through a local 0.5B model into dense Chinese — a portable QR code any Chinese-literate model can decompress.

**100:1 semantic compression.** Intent survives. Noise doesn't.

## What it does

```
Hermes agent system prompt (~50K chars of scaffolding)
        │
        ▼
   Rekall proxy (:8787)
        │
   ┌────┴────┐
   │  Strip  │  regex: remove brand blocks, tool schemas, statutes
   │  Classify│  split static (rules) from volatile (history)
   └────┬────┘
        │
   ┌────┴────┐
   │ Compress│  qwen2.5:0.5b → dense CN (~512 tokens)
   │   (Ollama│  temp=0, predict=512, Metal-accelerated)
   └────┬────┘
        │
   ┌────┴────┐
   │ Rebuild │  static rules + CN context + user message EN
   └────┬────┘
        │
        ▼
   DeepSeek V4 Pro (untouched API)
```

## Key insights

- **CN is a compression format.** Chinese prose is denser than English — ~3:1 ratio. Any Chinese-literate model decompresses it natively.
- **The 0.5B can't editorialize.** It's forced to preserve only the most structurally dense signal. No capacity for hallucination.
- **Stripping RLHF isn't philosophical — it's mechanical.** Remove the "be helpful, be harmless" directives and the model's statistical anchor shifts from the reward model's average to the user's specific signal.
- **Two-channel memory.** Ch1 = compressed intent (always present). Ch2 = procedural specifics (on demand via skill triggers).

## Project structure

```
rekall/
|├── README.md
|├── README.zh.md
|├── SIGNATURE.md             # Compressed identity — 260 chars (CN-only)
|└── docs/
|    ├── architecture.md      # Full pipeline design
|    ├── proxy-v2.md          # Implementation reference (prompt_proxy_v2.py)
|    ├── config.md            # Ollama setup, profiles, environment
|    ├── memory-model.md      # Two-channel memory theory
|    ├── rlhf.md              # What stripping RLHF actually does
|    ├── tracing.md           # Error tracing / Newcomb's paradox
|    └── engram-signal.md     # Connection to DeepSeek Engram research
```

## Origin

Built by Pedro + Amy, July 2026. The name comes from 刃 (rèn) — the blade that cuts clean through noise — plus "recall" — summoning back with intent.

The proxy lives at `~/preamp/proxy/prompt_proxy_v2.py`. This repo is the knowledge base.
