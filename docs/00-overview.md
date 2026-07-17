# Rainmaker — Overview

**System 2: multi-tenant Business-in-a-Box.** Rainmaker runs the lifecycle of a
client's digital marketing/ops as an autonomous unit: onboard a client, learn
their "Business DNA," continuously research their market, generate on-brand
content, and distribute it across channels — with the human owner reviewing KPIs
rather than doing the work. One operator can run many client tenants at once.

This is the lowest-risk, fastest-to-revenue of the three systems.

## What changed from the original blueprint

The blueprint is kept almost intact. The one substantive change is the
**distribution engine** (see `adr/0001-compliant-distribution.md`):

Direct automated posting to LinkedIn / Instagram / X via headless browsers or
unofficial endpoints violates those platforms' terms and gets **client** accounts
throttled or banned — catastrophic for a service you sell to clients. Rainmaker
distributes only through **official APIs and sanctioned schedulers** (Ayrshare,
Buffer, and the platforms' own APIs). Same multi-channel outcome, without putting
client accounts at risk.

Market intelligence is likewise constrained to compliant sources (RSS/Atom,
official platform APIs, licensed news/data) rather than scraping sites that
forbid it.

## Product shape

- `core/` — shared settlement + queue + worker engine (see `01-core.md`). Every
  Rainmaker stage runs as a core worker kind, so it inherits queueing, retries,
  billing, and audit for free.
- `rainmaker/dna.py` — the persistent per-tenant **Business DNA** profile.
- `rainmaker/market_pulse.py` — compliant market-intelligence loop → "Market Pulse."
- `rainmaker/content.py` — content generation + a rule-based quality gate.
- `rainmaker/distribution/` — official-channel adapters behind one interface.
- `rainmaker/pipeline.py` — wires the stages and registers the worker kinds
  (`rainmaker.generate_asset`, `rainmaker.distribute`).

## Monetization

- **Subscription:** $500–$2,000 / client / month for the automated unit.
- **Target:** 20 clients × $500 = $10k MRR.
- **Margin:** primary cost is generation/distribution API spend, well under 10%
  of revenue at scale.
- **White-label** upside: license an instance to other agencies.

## Multi-tenancy

Each client is an isolated tenant keyed by `tenant_id`: its own Business DNA,
content calendar, authorized channels, and (via the core) its own billing account.
Scaling 1 → 100 clients is adding worker capacity, not rearchitecting.

## Run it

```bash
pip install -e ".[dev]"     # LLM generation needs no extra deps (stdlib HTTP)
pytest                       # tests

# Full client flow from the CLI:
python -m rainmaker onboard --tenant acme --name "Acme" --industry SaaS \
    --audience "CTOs" --value "ship faster" --keywords "ship faster" --credit 500000
python -m rainmaker run    --tenant acme
python -m rainmaker report --tenant acme
```

See `03-client-flow.md` for the working end-to-end path.

## Status

**A first working client flow is built** (`onboarding.py`, `orchestrator.py`,
`cli.py`): onboard → research → generate → quality-gate → schedule, all through the
core queue with billing and audit. Generation uses any OpenAI-compatible LLM
(free options: Gemini / Groq / OpenRouter — see `FREE_LLM_SETUP.md`) when
`LLM_PROVIDER` + `LLM_API_KEY` are set, and an offline template otherwise;
distribution is safe **dry-run** until real credentials are connected.

A **human approval gate** now sits between generation and publishing: a cycle
parks quality-passed assets as `pending_review`; you approve each (choosing a
publish time) or reject them via the CLI. See `03-client-flow.md`.

## Next build steps

1. Schedule `run_cycle` per tenant (cron → core queue) so the generate-and-park
   step runs on a cadence and simply fills your review queue.
2. Performance loop: ingest channel analytics, A/B test, auto-promote winners.
3. Wire the LinkedIn/Meta official APIs alongside Ayrshare/Buffer.
4. Reddit/news official-API sources in addition to RSS.
5. Owner dashboard / weekly KPI report.
