# Rainmaker

**Multi-tenant Business-in-a-Box.** Runs a client's marketing/ops as an autonomous
unit: Business DNA → continuous market intelligence → on-brand content → human
approval gate → compliant multi-channel distribution. One operator, many client
tenants. Lowest-risk, fastest-to-revenue of the three systems.

> One of three independent passive-cashflow systems. Sibling products: **Tollgate**
> (API-as-a-Service) and **Ratchet** (compute broker). Each is self-contained and
> embeds its own copy of the shared `core` engine.

## Status

The client flow runs end to end: onboard a tenant, derive its Business DNA, pull
market intelligence, generate on-brand assets, park them for human review, and
distribute on an approved schedule. **26 tests passing.**

What is real vs. scaffolded:

| Stage | State |
|---|---|
| Onboarding → Business DNA | Built |
| Market pulse (intelligence) | Built |
| Content generation + quality gate | Built |
| Human approval gate (park / approve+schedule / reject) | Built |
| Distribution — dry-run adapter | Built; produces the real request shape without a network call |
| Distribution — Ayrshare / Buffer adapters | **Scaffolds.** Correct call shape, no live request yet |
| Performance loop (analytics feeding back into generation) | **Not built** |

That last row is the honest gap: the loop is designed but the analytics ingest
does not exist yet, so today the system is open-loop.

### Provider-agnostic, keyless-capable LLM

Generation targets any OpenAI-compatible endpoint (Gemini, Groq, OpenRouter,
DeepSeek, or a custom base URL), selected by environment variable over stdlib
HTTP, with Anthropic back-compat retained. The lead configuration is **in-house
Ollama** — no key, no per-token cost. When no provider is reachable it falls back
to templates rather than failing. See `FREE_LLM_SETUP.md`.

## Layout

```
src/core/          shared settlement + queue + worker engine (product-agnostic)
src/rainmaker/     onboarding, dna, market_pulse, content (+ quality gate),
                   generation, review, orchestrator, pipeline, cli,
                   distribution/ (base, dryrun, ayrshare, buffer)
docs/              architecture (00-overview, 01-core, 02-rainmaker),
                   03-client-flow, ADRs
tests/             26 tests
```

## Quickstart

```bash
pip install -e ".[dev]"
pytest
```

Distribution goes through **official APIs / sanctioned schedulers only** (Ayrshare,
Buffer, platform APIs). See `docs/00-overview.md`, `docs/02-rainmaker.md`,
`docs/03-client-flow.md`, and `docs/adr/0001` (compliant distribution). Every stage
runs as a `core` worker kind, inheriting queueing, retries, billing, and audit.

## Next steps

1. **Make one distribution adapter real.** Ayrshare is the shortest path — the call
   shape is already written; it needs credentials and live error handling.
2. **Close the performance loop.** Ingest post-level analytics and feed them back
   into the market-pulse and content stages. This is the largest missing piece and
   the one the pitch leans on hardest.
3. **Run a real tenant end to end** rather than fixtures — onboarding through to a
   scheduled, published post.
4. **Multi-tenant scheduling.** A runner exists for sweeping all tenants; it needs
   to be committed and put on a schedule.

---

_Source code is private. This repository is a public write-up of what the project does, how it's built, and the decisions behind it — not the implementation._
