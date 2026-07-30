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
distribute on an approved schedule. **39 tests passing.**

What is real vs. scaffolded:

| Stage | State |
|---|---|
| Onboarding → Business DNA | Built |
| Market pulse (intelligence) | Built |
| Content generation + quality gate | Built |
| Human approval gate (park / approve+schedule / reject) | Built |
| Distribution — dry-run adapter | Built; produces the real request shape without a network call |
| Distribution — Ayrshare adapter | **Built (live).** Real `POST /api/post` + analytics; degrades to `ok=False` with no key |
| Distribution — X (direct) adapter | **Built (live).** Direct `POST /2/tweets`, text posts — no aggregator subscription (In-House Roadmap Phase 4) |
| Distribution — Buffer adapter | Scaffold. Correct call shape, no live request yet |
| Performance loop (analytics feeding back into generation) | **Built.** `analytics.py` store + `ingest-metrics` → per-type engagement summary → generation `pulse_context` |

The system is now **closed-loop**: published posts are registered, their metrics
pulled back via `rainmaker ingest-metrics`, aggregated per asset type, and fed
into the next cycle's generation as a performance hint. A tenant with no measured
posts yet produces an empty hint, so early cycles behave exactly as before.
Ayrshare is the shortest path to going live (one key, every network); the direct
X adapter is the zero-subscription path for text posts. Buffer remains a scaffold.

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
                   generation, review, orchestrator, pipeline, cli, analytics
                   (performance loop),
                   distribution/ (base, dryrun, ayrshare, x, buffer, _http)
docs/              architecture (00-overview, 01-core, 02-rainmaker),
                   03-client-flow, ADRs
tests/             39 tests
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

1. ~~Make one distribution adapter real.~~ **Done** — Ayrshare (live multi-network)
   and a direct X adapter both make real API calls with graceful degradation.
2. ~~Close the performance loop.~~ **Done** — `analytics.py` + `ingest-metrics`
   pull post metrics back and feed a per-type performance hint into generation.
3. **Run a real tenant end to end** with live credentials — onboarding through to a
   scheduled, published post, then `ingest-metrics` on a schedule to warm the loop.
4. **Multi-tenant scheduling.** A runner exists for sweeping all tenants; it needs
   to be committed and put on a schedule (pairs naturally with periodic
   `ingest-metrics`).
5. **Broaden direct adapters.** X covers text today; port Relay's media-upload flow
   for image/video posts, and add direct LinkedIn/Meta to further shed Ayrshare.

---

_Source code is private. This repository is a public write-up of what the project does, how it's built, and the decisions behind it — not the implementation._
