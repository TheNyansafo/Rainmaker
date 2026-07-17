# Rainmaker — Product Detail

## The five stages

1. **Onboarding → Business DNA** (`dna.py`). Captured once, refined over time; the
   single source of tenant truth. Fields: brand voice, USPs, target keywords,
   competitors, and — importantly — `authorized_channels` (only channels the
   client has connected via official APIs).

2. **Market Intelligence Loop** (`market_pulse.py`). Scheduled `gather()` pulls
   from compliant sources (RSS/Atom, official platform APIs, licensed news) and
   emits a `MarketPulse` of `Signal`s tagged `news | competitor | lead`. Run it as
   a scheduled core worker (`rainmaker.market_pulse`).

3. **Content & Asset Generation** (`content.py`). `generate_asset(dna, type, brief)`
   produces an `Asset`, then `quality_check` applies rule-based gates (length
   bounds per asset type, keyword presence). Only `approved` assets are eligible
   to distribute. Generation itself is provider-agnostic — swap `_generate` for
   your LLM/image client.

4. **Distribution Engine** (`distribution/`). One `DistributionAdapter` interface;
   concrete adapters wrap official APIs / sanctioned schedulers. Ships `Ayrshare`
   and `Buffer` adapters (stubbed network calls, real call shapes documented). No
   browser auto-poster exists by design.

5. **Performance & Optimization** (build next). Ingest channel analytics, A/B test
   headlines/CTAs/visuals, auto-promote winners, surface KPIs to the owner.

## Worker kinds

| Kind | Payload | Result |
|---|---|---|
| `rainmaker.generate_asset` | `{dna, asset_type, brief}` | asset body + `approved` + `quality_notes` |
| `rainmaker.distribute` | `{adapter, channels, text, media_urls?, schedule_at?}` | adapter result (`ok`, `external_id`, ...) |

Both are registered (with prices) by `pipeline.register(...)`. Because they run on
the core, they get queueing, retries, per-action billing, and audit automatically.

## Quality gate rules

Defined in `content.QUALITY_RULES` per asset type (`blog`, `linkedin`, `email`,
`ad_copy`) as `min_words` / `max_words`, plus a target-keyword-presence check.
Extend with tone/readability/claims checks as needed. An asset that fails the gate
is returned un-approved with `quality_notes` explaining why — it never reaches
distribution.

## Adding a distribution channel

Implement `DistributionAdapter` (see `distribution/base.py`): `supported_channels()`
and `publish(channels, text, media_urls, schedule_at)`. Wrap an **official** API
or a sanctioned scheduler only. Register the adapter in `pipeline._ADAPTERS`.

## Compliance guardrails baked in

- Distribution targets abstractions over official APIs; there is no code path that
  drives a logged-in browser session to post.
- `BusinessDNA.authorized_channels` is the allowlist — distribution should refuse
  channels a tenant hasn't explicitly connected.
- Market intelligence sources are feeds/official APIs/licensed data, not scrapes
  of sites that forbid it.
