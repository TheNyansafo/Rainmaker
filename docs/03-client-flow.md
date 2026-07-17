# Rainmaker — The Client Flow (working)

This is the end-to-end path an operator runs per client. It works out of the box:
generation falls back to an offline template if no LLM key is set, market research
is network-free unless you opt in, and distribution runs in a safe **dry-run**
mode until real credentials are connected. Nothing here can touch a live social
account by accident.

## The cycle (with the human approval gate)

```
onboard_client(dna, credit)                      # core account + DNA + prepaid credit
        │
run_cycle(tenant)                                # orchestrator.run_cycle
        ├─ gather()            Market Pulse from compliant RSS/Atom sources
        ├─ default_plan()      content plan from DNA (+ a reactive post if the pulse fired)
        ├─ rainmaker.generate_asset   (core queue: billed, retried, audited)
        ├─ quality_check()     keep only assets that pass the gate
        └─ PARK as pending_review   ← nothing is published yet
        │
   ┌────┴──────── HUMAN GATE (you) ────────────┐
   │  pending / show                            │  review what was generated
   │  approve --asset ID  --at <when>           │  schedule publishing
   │  approve --all       --at <when>           │  (or reject --asset ID)
   └────┬───────────────────────────────────────┘
        │
   rainmaker.distribute (core queue) -> scheduled | distributed
```

Generation and distribution are separate billable steps: a cycle bills only the
**generation** it did; **distribution is billed at approval**, so content you
reject never costs a distribution charge. Nothing reaches a channel without your
explicit approval.

## Run it (CLI)

```bash
pip install -e ".[dev]"            # add ".[gen]" to enable Claude generation

python -m rainmaker onboard --tenant acme --name "Acme Analytics" \
    --industry SaaS --audience "CTOs at Series A startups" \
    --value "ship features 2x faster" \
    --keywords "developer velocity,ship faster" \
    --usps "one-day setup,no-code pipelines" \
    --channels "linkedin,email" --credit 500000

# 1) generate + quality-gate, then PARK for review (publishes nothing)
python -m rainmaker run     --tenant acme

# 2) review the queue
python -m rainmaker pending --tenant acme
python -m rainmaker show    --tenant acme --asset asset_XXXX

# 3) approve with a publish schedule, or reject
python -m rainmaker approve --tenant acme --asset asset_XXXX --at +2h
python -m rainmaker approve --tenant acme --all --at "2026-07-12T09:00"
python -m rainmaker reject  --tenant acme --asset asset_YYYY --reason "off-brand"

# 4) status: assets grouped by state
python -m rainmaker report  --tenant acme
```

### Schedule strings (`--at`)

| Value | Meaning |
|---|---|
| `now` (default) | publish immediately (status → `distributed`) |
| `+30m` / `+2h` / `+3d` | relative offset from now (status → `scheduled`) |
| `2026-07-12T09:00` | ISO-8601 datetime |
| `1783708537` | raw epoch seconds |

Example: approving with `--at +2h` returns `status: "scheduled"` and the exact
publish time; the schedule is passed straight through to the distribution adapter
(Buffer/Ayrshare `scheduleDate` in production).

`--credit` and all money are in **millicents** (1 cent = 1000). Generation is
20,000 per asset; distribution is 2,000 per approved asset.

## Turning on the real integrations

| Capability | How to enable | Where |
|---|---|---|
| **LLM generation** | set `LLM_PROVIDER` + `LLM_API_KEY` (free: gemini/groq/openrouter) — see `FREE_LLM_SETUP.md` | `generation.get_generator()` uses any OpenAI-compatible provider |
| **Live market research** | pass `--sources feeds.json` and call `gather(..., live=True)` | `market_pulse.fetch_feed` |
| **Real distribution** | set `AYRSHARE_API_KEY` / `BUFFER_ACCESS_TOKEN`, run with `--adapter ayrshare` | `distribution/*_adapter.py` |

`feeds.json` shape:

```json
[{"type": "rss", "url": "https://example.com/feed.xml",
  "source": "rss:example", "kind": "news"}]
```

## Safety properties baked into the flow

- **Human approval gate** — a cycle publishes nothing. Assets wait as
  `pending_review` until you approve them; you choose the publish time per asset
  (or reject).
- **Dry-run by default** — even after approval, no live account is touched until
  you pick a real adapter (`--adapter ayrshare`).
- **Strict channel allowlist** — an asset can only target a channel in the
  tenant's `authorized_channels`; an asset with no authorized channel cannot be
  approved (it is auto-rejected), never silently redirected.
- **Compliant research** — RSS/Atom and official APIs only; parsing is a pure
  function, and network fetches are opt-in.
- **No charge for rejected work** — distribution is billed at approval, so
  rejected content incurs no distribution charge; nothing is billed for failed work.

## What to build next on top of this

1. Schedule `run_cycle` per tenant (cron → the core queue) for hands-off operation.
2. Performance loop: ingest channel analytics, A/B test variants, auto-promote
   winners into `default_plan`.
3. Approval gate: hold `approved` assets for optional human review before
   distribution (a status between `approved` and `distributed`).
4. Richer `default_plan` driven by pulse signal *kind* (lead vs. competitor vs.
   news) and buyer-journey stage.
