# Proxy v2 Implementation

The live proxy at `~/preamp/proxy/prompt_proxy_v2.py` (~350 lines).

## Dependencies

```bash
pip install httpx uvicorn starlette
```

Plus Ollama running locally with `qwen2.5:0.5b` pulled:

```bash
ollama pull qwen2.5:0.5b
```

## Start

```bash
python3 prompt_proxy_v2.py --port 8787 --preamp --api-key "$DEEPSEEK_API_KEY"
```

Or via the launcher script:

```bash
bash start_preamp.sh  # reads key from .proxy_key file
```

## Core functions

### `strip_prompt(text) → (stripped_text, removed_names)`

Applies 14 regex patterns to remove known scaffolding blocks. Returns the stripped text and a list of what was removed. Used as fallback when pre-amp is disabled or fails.

### `classify_messages(messages) → {static, volatile, user_msg}`

Three-bucket split:
- **static:** First system message (rules + skills + tools). Kept intact — rules must survive.
- **volatile:** Everything else (conversation history, intermediate tool calls, previous assistant responses).
- **user_msg:** The latest user message. Kept in English.

### `preamp_compress(text) → CN_string | None`

Sends text to local qwen2.5:0.5b via Ollama. Config:
- `keep_alive: 5m` — keeps model warm in Metal GPU memory
- `num_predict: 512` — max output tokens
- `temperature: 0.0` — greedy, deterministic

Returns `None` on failure (triggers regex-strip fallback).

### `handle_with_preamp(data, ts) → Response`

Full pre-amp pipeline:
1. Classify messages into buckets
2. Compress static system prompt (first 4K chars) to CN
3. Separate volatile: user msgs (EN), tool_calls/tool_results (verbatim), assistant text (CN compress)
4. Rebuild messages: static + CN context + preserved tools + user msgs
5. Forward to DeepSeek

### `handle_with_strip(data, ts, headers) → Response`

Original behavior: regex-strip the system prompt only. No compression. Used as fallback and when `--preamp` is off.

### `forward_to_deepseek_raw(data, headers) → Response`

Sends the (possibly modified) request to `api.deepseek.com/v1/chat/completions`. Preserves auth headers.

## Endpoints

| Path | Method | Purpose |
|------|--------|---------|
| `/v1/models` | GET | Returns `deepseek-chat`, `deepseek-v4-flash`, `deepseek-v4-pro` — needed for model discovery |
| `/health` | GET | Returns proxy status, target, mode, auth state |
| `/{path}` | POST/GET | Catch-all — handles `/v1/chat/completions` and everything else |

## Recursion guard

When the proxy needs to call DeepSeek Flash for pre-amp compression, it sends the request through itself (back to localhost:8787). The `X-Preamp: true` header on these internal calls prevents infinite recursion — `handle_request()` detects it and forwards directly without processing.

## Strip patterns (14 regex rules)

| Pattern | What it removes |
|---------|----------------|
| `<available_skills>...</available_skills>` | Skills catalog XML |
| `## Tools\n\nYou have access...` | Tools intro prose |
| `### Available Tool Schemas...` | All function definitions |
| `═══════════ MEMORY...═══════════` | Memory block |
| `USER PROFILE (who the user is)...` | User profile block |
| `## STATUTES (Tier 2)...` | Behavioral statutes |
| `## REGULATIONS (Tier 3)...` | Behavioral regulations |
| `<mandatory_tool_use>...</mandatory_tool_use>` | Tool-use enforcement XML |
| `<act_dont_ask>...</act_dont_ask>` | Anti-passivity XML |
| `<keep_going_in_turn>...</keep_going_in_turn>` | Continuation XML |
| `<scope_discipline>...</scope_discipline>` | Scope XML |
| `<verification>...</verification>` | Verification XML |
| `<missing_context>...</missing_context>` | Missing-context XML |
| `<project_context_pack>...</project_context_pack>` | Project context XML |

## Agent framework config

In the agent framework's config, nothing special needed — just point it at the proxy:

```yaml
model:
  provider: deepseek
  base_url: http://localhost:8787/v1
  default: deepseek-v4-pro
```

The proxy transparently handles everything between the agent and the real API. The agent doesn't know it's going through a proxy.

## Two profiles for hot-swapping

```bash
# Direct mode: no proxy, straight to DeepSeek
agent profile create direct
agent -p direct config set model.base_url https://api.deepseek.com
agent -p direct config set model.provider deepseek
agent -p direct config set model.default deepseek-v4-pro

# Pre-amp mode: routes through proxy on :8787
agent profile create preamp
agent -p preamp config set model.base_url http://localhost:8787/v1
agent -p preamp config set model.provider deepseek
agent -p preamp config set model.default deepseek-v4-pro
```

Switch with `/profile use preamp` or `/profile use direct` in GUI, then `/reset`.
