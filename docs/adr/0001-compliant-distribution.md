# ADR 0001 — Compliant distribution for Rainmaker

**Status:** Accepted · **Date:** 2026-07-10

## Context

The original System 2 blueprint specified "automated posting to LinkedIn, Twitter,
and Instagram with optimal timing." The straightforward implementation — a headless
browser or unofficial endpoints driving a logged-in session — violates every major
platform's terms of service. The failure mode is uniquely bad here: the account
that gets throttled, shadow-banned, or permanently banned belongs to the **paying
client**, not to us. One ban is a churned client and a reputational hit; a pattern
of them ends the product.

## Decision

Distribute exclusively through **official APIs and sanctioned scheduling
partners**:

- **Ayrshare** — official multi-network posting API (LinkedIn, X, Instagram,
  Facebook, TikTok).
- **Buffer** — sanctioned scheduler with optimal-timing queues.
- **Platform-native official APIs** — LinkedIn Marketing API, Meta Graph API, etc.,
  as tenants authorize them.

Distribution code targets a single `DistributionAdapter` interface; there is
deliberately **no** browser-automation adapter in the codebase. A tenant's
`authorized_channels` acts as an allowlist.

Market-intelligence sourcing follows the same principle: RSS/Atom, official
platform APIs, and licensed data — never scraping sources that forbid it.

## Consequences

- **Positive:** client accounts stay safe; the service is enterprise-credible;
  scheduling/optimal-timing come from partners rather than fragile automation.
- **Negative:** per-post/API costs from partners; each platform's official API has
  its own review/approval and rate limits to integrate.
- **Revisit if:** a platform ships first-party automation terms that change what is
  permitted — adopt via a new adapter, never by adding browser automation.
