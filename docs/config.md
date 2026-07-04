# Configuration

## Required components

### 1. Ollama (local)

Running on `localhost:11434` with `qwen2.5:0.5b`:

```bash
# Install Ollama (macOS)
brew install ollama

# Start the service
ollama serve

# Pull the model
ollama pull qwen2.5:0.5b
```

The 0.5B model uses ~400MB VRAM. On Apple Silicon, it runs via Metal with negligible latency.

### 2. DeepSeek API key

Set in environment or `.proxy_key` file:

```bash
export DEEPSEEK_API_KEY="sk-..."
# or
echo -n "sk-..." > ~/preamp/proxy/.proxy_key
```

### 3. Python dependencies

```bash
pip install httpx uvicorn starlette
```

## Ollama tuning

The proxy hardcodes these qwen settings for deterministic, fast compression:

| Parameter | Value | Why |
|-----------|-------|-----|
| `num_predict` | 512 | Max compressed output — enough for CN signal, not enough for rambling |
| `temperature` | 0.0 | Greedy decoding — deterministic, same input → same output |
| `keep_alive` | 5m | Keep model warm in Metal GPU between requests |

## Agent framework profiles

Two profiles for instant switching:

### Direct profile (no proxy)

```bash
agent profile create direct
agent -p direct config set model.base_url https://api.deepseek.com
agent -p direct config set model.provider deepseek
agent -p direct config set model.default deepseek-v4-pro
```

### Preamp profile (through Rekall proxy)

```bash
agent profile create preamp
agent -p preamp config set model.base_url http://localhost:8787/v1
agent -p preamp config set model.provider deepseek
agent -p preamp config set model.default deepseek-v4-pro
```

Switch in-session: `/profile use preamp` → `/reset`.

## Static identity file

An optional static identity file can be injected as a system message after the compressed context. This is a workaround — the upgrade path is to inject compressed *lived session context* dynamically instead.

## Compression behavior by message type

| Content | Pre-amp path | Strip-only path |
|---------|-------------|-----------------|
| System prompt (first 4K chars) | Compressed to CN | Regex-stripped, passed as-is |
| System prompt (beyond 4K) | Preserved verbatim | Regex-stripped, passed as-is |
| Assistant text responses | Compressed to CN | Passed as-is |
| Tool calls (with image_url) | Preserved verbatim | Passed as-is |
| Tool results (vision descriptions) | Preserved verbatim | Passed as-is |
| User messages | Kept in English | Passed as-is |
| Skills catalog | Regex-stripped | Regex-stripped |
| Brand XML blocks | Regex-stripped | Regex-stripped |

## Startup checklist

1. Start Ollama: `ollama serve`
2. Verify model: `ollama list | grep qwen2.5:0.5b`
3. Start proxy: `bash ~/preamp/proxy/start_preamp.sh`
4. Verify: `curl http://localhost:8787/health`
5. In agent framework: `/profile use preamp` → `/reset`

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `Connection refused :8787` | Proxy not running | Start proxy |
| `⚠️ Local pre-amp failed (5xx)` | Ollama not running | `ollama serve` |
| `⚠️ Local pre-amp failed (4xx)` | Model not pulled | `ollama pull qwen2.5:0.5b` |
| Pre-amp enabled but no compression | `--preamp` flag missing | Add `--preamp` to launch command |
| Vision/image calls lost | Pre-amp compressing tool msgs | Fixed in v2 — tool msgs preserved verbatim |
| Greeting/claude-speak in preamp | RLHF rules getting through | Increase strip regex coverage or tune qwen prompt |
