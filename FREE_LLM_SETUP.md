# Connecting a free LLM (or any OpenAI-compatible provider)

Rainmaker's content generation is provider-agnostic. It talks the **OpenAI
chat-completions format**, which nearly every provider now supports — so switching
from one LLM to another is **two env vars, no code change**. With no provider set,
it falls back to the offline template generator, so everything still runs.

## Fully in-house: local Ollama (recommended if you already run it)

If you have Ollama on the machine (as you do for Brimstone), this is the best
option: no key, no bill, no rate limits, and no data ever leaves your computer.

```bash
LLM_PROVIDER=ollama
LLM_MODEL=llama3.1      # set to a model you've actually pulled — run: ollama list
```

That's the whole config — no API key needed (the local server ignores auth).
Rainmaker will call `http://localhost:11434/v1`. Pick any model you've pulled
(`llama3.1`, `mistral`, `qwen2.5`, etc.); for marketing copy an 8B+ instruct model
is plenty.

> Note: run the Rainmaker CLI directly on the same machine as Ollama (not inside a
> container). If you ever containerize it, point `LLM_BASE_URL` at
> `http://host.docker.internal:11434/v1` instead of localhost.

## Recommended free HOSTED options (verified July 2026)

| Provider | Free? | Why | `LLM_PROVIDER` | Default model | Get a key |
|---|---|---|---|---|---|
| **Google Gemini** | ✅ free tier, no card | Best free limits (~1,500 req/day on 2.5 Flash), strong writing | `gemini` | `gemini-2.5-flash` | aistudio.google.com |
| **Groq** | ✅ free, no card | Extremely fast; open models (Llama 3.3 70B, Gemma) | `groq` | `llama-3.3-70b-versatile` | console.groq.com |
| **OpenRouter** | ✅ free models | One key, many models; `openrouter/free` auto-routes | `openrouter` | `openrouter/free` | openrouter.ai |
| **DeepSeek** | 💲 cheap, not free | Very low cost fallback if free limits pinch | `deepseek` | `deepseek-chat` | platform.deepseek.com |

> Free tiers and model names change often — if a call fails with "model not found",
> check the provider's current model list and set `LLM_MODEL`.

## Setup (pick one)

**Gemini (recommended):**
```bash
LLM_PROVIDER=gemini
LLM_API_KEY=your_gemini_key
```

**Groq:**
```bash
LLM_PROVIDER=groq
LLM_API_KEY=your_groq_key
```

**OpenRouter:**
```bash
LLM_PROVIDER=openrouter
LLM_API_KEY=your_openrouter_key
# LLM_MODEL=meta-llama/llama-3.3-70b-instruct:free   # pin a specific free model if you like
```

That's it. Run a cycle and the copy comes from your chosen provider:
```bash
python -m rainmaker run --tenant <your-tenant>
```

## Overrides & auto-detection

- **Model:** set `LLM_MODEL` to override the provider default.
- **Base URL:** for `LLM_PROVIDER=custom` (e.g. a local **Ollama** at
  `http://localhost:11434/v1`, or any self-hosted OpenAI-compatible server), set
  `LLM_BASE_URL` + `LLM_MODEL` + `LLM_API_KEY`.
- **Auto-detect:** if you set a provider-specific key (`GEMINI_API_KEY`,
  `GROQ_API_KEY`, `OPENROUTER_API_KEY`, `DEEPSEEK_API_KEY`) and no `LLM_PROVIDER`,
  the matching provider is selected automatically.

## Priority order

`get_generator()` chooses, in order: **LLM_\* provider** → legacy `ANTHROPIC_API_KEY`
(still supported) → **offline template**. So configuring a free provider immediately
takes over from Claude with no other change.

## Fully in-house option

For zero external dependency, run a local model with **Ollama** (or any local
OpenAI-compatible server) and point `custom` at it:
```bash
LLM_PROVIDER=custom
LLM_BASE_URL=http://localhost:11434/v1
LLM_MODEL=llama3.1
LLM_API_KEY=ollama            # any non-empty string; local servers ignore it
```
Now generation runs on your own hardware — no provider, no bill, no data leaving.
