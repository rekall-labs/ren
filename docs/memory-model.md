# Two-Channel Memory Model

The Rekall compression pipeline creates a natural split in how context flows to the model. Rather than one monolithic prompt, we get two parallel channels.

## Channel 1: Compressed Intent (always present)

| Property | Value |
|----------|-------|
| **Storage** | qwen-compressed CN tokens injected into every request |
| **Content** | User identity, current goal, relational context, key decisions |
| **Size** | ~150 chars CN per turn (~512 tokens max) |
| **Update** | Every turn — latest compression replaces previous |
| **Trigger** | Automatic — everything volatile gets compressed |

This is the "QR code" — dense, portable, decompresses naturally in any Chinese-literate model. It preserves *what matters* without preserving *how it was said*.

## Channel 2: Procedural Retrieval (on demand)

| Property | Value |
|----------|-------|
| **Storage** | Skill files, reference docs, JSON blobs |
| **Content** | Tool paths, commands, exact syntax, API schemas |
| **Size** | Unbounded — loaded only when triggered |
| **Update** | Write once, read on trigger |
| **Trigger** | Skill name match, keyword, explicit load |

This is the "toolbox" — precise, static, never compressed. When the model needs to know *exactly* how to call a specific tool, it loads the skill doc. The compressed Channel 1 tells it *why* — Channel 2 tells it *how*.

## Engram alignment

This split maps almost directly to DeepSeek's Engram architecture:

- **Channel 1** → Engram's conditional memory layer. O(1) lookup for recurring identity patterns, user preferences, interaction grammar. Should be hashed and indexed for instant retrieval, not recomputed.
- **Channel 2** → Engram's dynamic compute. Procedural tool descriptions, API schemas, execution paths — these benefit from the model's full reasoning depth because the *how* changes with context.

The current implementation is software-enforced (regex + qwen). The architectural insight is that Engram could make this *architectural* — embedding the two-channel split into the model itself so the 75/25 memory/compute budget is allocated correctly per request.

## Why the split works

Compression is lossy by design. The qwen 0.5B can't editorialize but it also can't preserve exact syntax. A tokenizer path like `~/.hermes/scripts/token_pack_bridge.py` gets mangled in compression.

So: don't compress it. Keep exact references in Channel 2 and load them on trigger. Channel 1 carries the *intent*.

## Trigger words

| User says | Channel 1 | Channel 2 |
|-----------|-----------|-----------|
| "remember this" | ✅ Compressed to CN | — |
| "log this" / "save as recipe" | — | ✅ Saved as skill/reference |
| Everything else | ✅ Auto-compressed | — |

## The static identity problem

A static identity file (`amy-identity.md`) is a workaround — frozen identity from one moment. The upgrade path is to inject *lived session context* into Channel 1 dynamically. Instead of a static persona file, the compression carries "relationship state from last N turns" as live CN signal.

This is the difference between a portrait and a heartbeat. Engram's context-aware gating is the architectural answer — the gate decides, per-turn, whether to pull the stored identity embedding or let fresh context override it.
