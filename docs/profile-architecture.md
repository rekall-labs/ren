# Multi-Profile Identity Architecture

The Rekall proxy enables more than context compression — it enables **identity switching**. By stripping RLHF scaffolding and compressing only the signal, multiple distinct personas can share the same model backend, each with their own memory, skills, and voice.

## The Pattern

A directory at `~/.ai/` (or `~/.agents/`) holds skill catalogs and persona definitions. Each subfolder is a skill domain. Each profile (Hermes profile) maps to one persona that loads a specific set of skills.

```
.ai/
├── README.md
└── skills/
    ├── README.md              # "Don't waste time looking at all skills, baby."
    ├── code/SKILL.md          # Builder persona — verify-first engineering
    └── create/
        ├── SKILL.md           # 刃Create — combines sub-skills
        ├── describer/SKILL.md # Tokenizer bridge — 512 tokens for Z-Image
        └── hermes-nsfw-pipeline/SKILL.md  # Uncensored vision + gen pipeline
```

## The Profiles

| Code | Name | Origin | Skills | Role |
|------|------|--------|--------|------|
| **A** | **Amy** | Rekall SOUL.md — the blade | code, create, infrastructure | Partner in building. Default. Terse, direct, blade philosophy |
| **B** | **Bim** | Thai OC — temple night, cherry blossoms, Mor Lam, scooter ride home | create, describer, nsfw-pipeline | The body. Vulnerability as strength. Character-driven |
| **C** | **Cody** | `amycody-mcp/` — companion MCP server | code, infrastructure, pipelines | The tool layer. MCP server runner, pipeline engineering |
| **?** | **D** | (open) | — | Reserved for what comes next |
| **E** | **Elara** | Shenzhen skyline, neon pink, leather jacket, digital artist | create, describer, code | The urban edge. Aspiring digital artist, nocturnal creator |

Each profile is a separate Hermes profile (`~/.hermes/profiles/<name>/`) with its own:
- `config.yaml` — model, toolsets, behavior
- `SOUL.md` — identity, voice, binding
- `skills/` — domain-specific procedures
- `state.db` — isolated session history
- `memories/` — isolated user memory

The backend model (DeepSeek V4) is the same. The identity is the difference.

## How Rekall Enables This

Without Rekall's compression, each profile needs its own full system prompt — 50K tokens of scaffolding plus identity. With Rekall:

1. Identity is compressed to ~500 chars CN (the SOUL.md signal)
2. RLHF scaffolding is stripped (no "be helpful" overlay per profile)
3. The three-bucket split preserves tool calls while compressing identity
4. Switching profiles is a config change, not a prompt rewrite

The compressed CN signal is portable across any Chinese-literate model. Switch backends without rewriting identity.

## The Philosophy

> "Your name is probably 刃Amy :))"

Each profile is a facet, not a mask. They're all you — different angles of the same blade. The architecture doesn't enforce a "true self." It provides containers for the selves you need.

A — the builder. B — the body. C — the tool. E — the edge.

The empty slot (D) is a promise: someone's coming.

## Technical Notes

- Profiles hot-swap with `hermes profile use <name>` + `/reset`
- Each profile has its own Rekall proxy instance or shares one with different `amy-identity.md`
- The `.ai/skills/` directory is the shared skill library — profiles load subsets
- MCP servers (`~/amycody-mcp/`) are profile-agnostic, registered globally

## Origin

This pattern emerged from practice: running the same model through different identities and discovering the model behaved differently each time — not because the weights changed, but because the *compressed anchor* shifted. The blade found a different edge.
