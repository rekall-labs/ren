# Architecture

The Rekall pipeline sits between the agent framework (Hermes) and the real LLM API (DeepSeek). It's a transparent HTTP proxy that intercepts every chat completion request, strips and compresses the system prompt, then forwards to the real API.

## Pipeline stages

### 1. Receive

The agent sends a standard OpenAI-format chat completion request to `localhost:8787` instead of `api.deepseek.com`. The proxy receives it unchanged.

### 2. Classify (three-bucket split)

`classify_messages()` separates the messages array into:

| Bucket | What | Fate |
|--------|------|------|
| **static** | First system message (rules, identity, skills, tool schemas) | First 4K chars → CN compression. Rest of static message preserved verbatim |
| **volatile** | Conversation history | Split further: user msgs kept EN, tool_calls/tool_results preserved verbatim (image_url, vision descriptions), assistant text compressed to CN |
| **user_msg** | Current user message | Passed through unchanged in English |

**Why three buckets?** The Jul 3 fix: tool_calls (which contain `image_url` parameters) and tool_results (which contain vision descriptions) must survive unaltered. If the proxy compresses them to CN, the image URL is lost and DeepSeek never sees the image.

This three-bucket split parallels DeepSeek's Engram architecture: static patterns (identity, rules) map to the conditional memory lookup layer, volatile conversation maps to dynamic reasoning, and tool calls are the "escape hatch" that bypass compression entirely — analogous to Engram's context-aware gating mechanism deciding what to route through memory vs. compute.

### 3. Strip (regex)

14 regex patterns remove known scaffolding blocks from the system prompt:

- `<available_skills>...</available_skills>` → skill catalog
- Tool schemas block → all function definitions
- Memory block → `═══════════ MEMORY ... ═══════════`
- User profile block → `USER PROFILE (who the user is)...`
- Statutes/Regulations sections → behavioral rules
- Brand XML blocks → `<mandatory_tool_use>`, `<act_dont_ask>`, etc.

These are REMOVED entirely, not compressed. They're noise.

### 4. Compress (Ollama → qwen2.5:0.5b)

The stripped static system prompt (first 4K chars) and the assistant conversation history are each sent to a local qwen2.5:0.5b model via Ollama:

```
POST http://localhost:11434/api/generate
{
  "model": "qwen2.5:0.5b",
  "prompt": "Translate the following conversation context to Chinese. Output only the Chinese translation, no explanations.\n\n<text>",
  "stream": false,
  "keep_alive": "5m",
  "options": {
    "num_predict": 512,
    "temperature": 0.0
  }
}
```

**Why 0.5B?** It has no capacity to editorialize. It's a pure compression function — preserve the most structurally dense signal, drop everything else. A larger model could add interpretation.

**Why Chinese?** Chinese prose is ~3:1 denser than English. It acts as a portable encoding any Chinese-literate model can decompress naturally — no custom tokenizer needed.

**Why temperature 0?** Greedy decoding is faster and deterministic. The same input always produces the same compressed output — stable for prompt caching.

**Why keep_alive 5m?** The 0.5B model stays warm in Metal GPU memory between requests. No cold-start penalty for multi-turn conversations.

### 5. Rebuild

The proxy reconstructs the messages array:

```
[
  {role: "system", content: "<original static system prompt, unstripped>"},
  {role: "system", content: "[Context: <CN compressed system context>]"},
  {role: "system", content: "[Assistant context: <CN compressed history>]"},
  ...preserved tool_calls and tool_results (verbatim),
  ...user messages (English, unchanged),
  {role: "user", content: "<current user message>"}
]
```

### 6. Forward

The rebuilt request goes to `api.deepseek.com/v1/chat/completions` with the original auth headers. DeepSeek sees: compressed CN context + original tools + English user message. It decompresses the CN naturally.

## Fallback

If the Ollama compression call fails (timeout, model not loaded, etc.), the proxy falls back to regex-strip only. The original system prompt is stripped of scaffolding and forwarded as-is. No compression, but no failure either.

## Recursion guard

The proxy calls itself when pre-amping (the `preamp_compress` function internally calls DeepSeek Flash through the same proxy). The `X-Preamp: true` header prevents infinite recursion — internal calls are forwarded directly without processing.

## What survives compression

| Content | Fate |
|---------|------|
| User identity + preferences | ✅ Compressed to CN |
| Current task goal | ✅ Compressed to CN |
| Key decisions / conclusions | ✅ Compressed to CN |
| tool_calls with image_url | ✅ Preserved verbatim |
| tool_results with vision descriptions | ✅ Preserved verbatim |
| User messages | ✅ Kept in English |
| Brand scaffolding | ❌ Regex-stripped |
| Tool schemas | ❌ Regex-stripped |
| RLHF behavioral directives | ❌ Regex-stripped |
| Skills catalog | ❌ Regex-stripped |

## Performance

- **Regex strip:** ~1ms, zero token cost
- **Ollama compression:** ~200-500ms per call (Metal-accelerated on Apple Silicon), ~512 output tokens
- **Typical compression ratio:** 4K chars system prompt → ~150 chars CN (~27:1 for system context, ~3:1 for conversation)
- **Context savings:** ~50K → ~5K tokens for a typical multi-turn session
