# AI providers

Scalar talks to whichever model vendor your installation chooses, and to none if you prefer.

**Requests go straight from your installation to the provider you name.** Nothing passes through infrastructure operated by Scalar's maintainers, because there is no such infrastructure. Your key, your endpoint, your logs.

## Scalar without AI

The default. Leave every `AI_*` setting blank and Scalar runs: tasks, projects, calendar, timeline, inbox, planning, focus and search all work exactly as they do with a provider configured. Only Ask Scalar is unavailable, and it says so rather than failing oddly. Settings shows "No AI provider is configured".

AI enhances Scalar. It does not make Scalar function.

## Configuration

| Variable | Meaning |
| --- | --- |
| `AI_PROVIDER` | `anthropic`, `openai`, `ollama` or `openai_compatible` |
| `AI_MODEL` | Model name. Each provider has a sensible default |
| `AI_BASE_URL` | Origin of an OpenAI-compatible API, with or without `/v1` |
| `AI_API_KEY` | Omitted for a local server that does not want one |

### Anthropic

```env
AI_PROVIDER=anthropic
AI_API_KEY=sk-ant-…
AI_MODEL=claude-opus-5
```

### OpenAI

```env
AI_PROVIDER=openai
AI_API_KEY=sk-…
AI_MODEL=gpt-5
```

### Ollama, or any local model

```env
AI_PROVIDER=ollama
AI_BASE_URL=http://ollama:11434
AI_MODEL=llama3.1
```

`AI_BASE_URL` defaults to `http://localhost:11434/v1`, so a plain `AI_PROVIDER=ollama` works when Ollama runs on the same machine as the API. No key is needed: a local model has no key, and demanding one would make the simplest self-hosted setup the most annoying.

The infra compose file ships an optional profile:

```bash
docker compose --profile ollama up -d
```

Then pull a model and point the API at it:

```bash
docker compose --profile ollama exec ollama ollama pull llama3.1
```

### Anything else OpenAI-compatible

vLLM, llama.cpp, LM Studio, OpenRouter, a colleague's gateway:

```env
AI_PROVIDER=openai_compatible
AI_BASE_URL=https://your-endpoint/v1
AI_MODEL=your-model
AI_API_KEY=…
```

## What to expect from a local model

Scalar asks a model to interpret a sentence and to choose tools. It never asks a model to work out whether an hour is free: that is the [planner](../architecture/planner.md), which is ordinary deterministic code. So a smaller local model degrades the *interpretation*, not the correctness of your schedule.

In practice, on a local model of the size that runs comfortably on a laptop:

- Simple questions ("what is due tomorrow?") work well.
- Multi-step tool use is less reliable; the loop may need more turns or give up.
- Structured output is the weakest point. Scalar validates it and reports a clear error rather than acting on something malformed.

Nothing a model returns is trusted: tool input is validated against a schema, writes are classified and require explicit approval, and the planner is unaffected by which model you use. See [ai-safety.md](../architecture/ai-safety.md).

## Upgrading from `ANTHROPIC_API_KEY`

It still works. An installation with only `ANTHROPIC_API_KEY` set continues to use Anthropic, because upgrading Scalar should not silently turn off someone's assistant. Setting `AI_PROVIDER` takes precedence, and the old variable is deprecated.

## When it is wrong

Misconfiguration is logged once at boot and then treated as no provider at all. Scalar does not refuse to start: that would take tasks, calendar and planning down over an optional feature.

| What you see | Usually means |
| --- | --- |
| `AI_PROVIDER=openai needs AI_API_KEY` at boot | A provider that needs a key has none |
| Command returns 503, Settings says not configured | Nothing set, or the configuration was rejected at boot. Check the API logs |
| "could not reach the model provider" | Wrong `AI_BASE_URL`, or the local server is not running |
| "has no `<model>` at `<url>`" | `AI_MODEL` is not pulled or not spelled the way the server spells it |
| "rejected the API key" | Wrong or expired key |
