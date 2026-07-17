# Rainmaker

**Multi-tenant Business-in-a-Box.** Runs a client's marketing/ops as an autonomous
unit: Business DNA → continuous market intelligence → on-brand content → compliant
multi-channel distribution → performance loop. One operator, many client tenants.
Lowest-risk, fastest-to-revenue of the three systems.

> One of three independent passive-cashflow systems. Sibling products: **Tollgate**
> (API-as-a-Service) and **Ratchet** (compute broker). Each is self-contained and
> embeds its own copy of the shared `core` engine.

## Layout

```
src/core/          shared settlement + queue + worker engine (product-agnostic)
src/rainmaker/     dna, market_pulse, content (+ quality gate), distribution/, pipeline
docs/              architecture (00-overview, 01-core, 02-rainmaker) + ADRs
tests/             4 tests
```

## Quickstart

```bash
pip install -e ".[dev]"
pytest
```

Distribution goes through **official APIs / sanctioned schedulers only** (Ayrshare,
Buffer, platform APIs). See `docs/00-overview.md`, `docs/02-rainmaker.md`, and
`docs/adr/0001` (compliant distribution). Every stage runs as a `core` worker kind,
inheriting queueing, retries, billing, and audit.

---

_Source code is private. This repository is a public write-up of what the project does, how it's built, and the decisions behind it — not the implementation._
