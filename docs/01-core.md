# The Shared Core (settlement + queue + worker)

Every product in this family embeds its own copy of the `core` package. The core
is deliberately product-agnostic: it knows how to authenticate callers, hold a
prepaid balance, queue work, run idempotent workers, settle payment, and keep an
audit trail. It knows nothing about *what* the work is — each product registers
its own worker kinds and price rules at boot.

This is the single reusable asset across all three passive-cashflow systems. Get
it right once and each new product is "just" a set of workers on top.

## Modules

| Module | Responsibility |
|---|---|
| `config.py` | Env-driven settings (DB path, retry policy). Per-product env prefix. |
| `db.py` | SQLite schema + connection. WAL with graceful fallback. `transaction()` context manager for atomic units of work. |
| `auth.py` | Account creation, API-key issuance (SHA-256 hashed, shown once), authentication. |
| `ledger.py` | Prepaid credit in integer **millicents** (no float drift). Append-only ledger; every credit/debit/refund is one immutable row inside a transaction. |
| `pricing.py` | Price matrix + tiered volume discounts. Ships EMPTY; products call `register_price(kind, rule)`. |
| `queue.py` | Persisted priority queue in SQLite. `submit_request` (with idempotency dedupe + soft affordability check) and `claim_next` (atomic claim, safe across workers). |
| `worker.py` | Worker registry + `process_request`: runs the kind's callable, charges on success, retries transient failures with exponential backoff, marks permanent failures without charging. |
| `audit.py` | Append-only audit log; one row per meaningful event. |
| `gateway.py` | FastAPI edge: submit / poll / balance, plus a dev-only account-provisioning route. |

## Request lifecycle

1. **Intake** — caller hits the gateway (or calls `queue.submit_request`) with an
   API key, a `kind`, a `payload`, an optional `priority` (1–9) and an optional
   `idempotency_key`.
2. **Auth + dedupe** — the key resolves to an account. If the idempotency key was
   seen before, the original request is returned; no duplicate work, no double
   charge.
3. **Soft affordability** — intake rejects (HTTP 402) if the account clearly can't
   afford the estimated price. The authoritative charge happens later.
4. **Queue** — the request is persisted `queued` with a computed compute weight.
5. **Claim** — a worker atomically flips one `queued` row to `running`
   (`UPDATE ... WHERE status='queued'`), so two workers never take the same job.
6. **Execute** — the worker callable runs. On success the account is debited the
   priced cost *inside the same flow*, the result is stored, status → `succeeded`.
7. **Failure** — `PermanentError` (bad input) fails immediately with no charge.
   Anything else retries up to `max_retries` with `base * factor^(n-1)` backoff;
   on exhaustion the request fails, still uncharged, and a `webhook.notify` event
   is recorded for the caller.
8. **Audit** — every transition writes an audit row.

## Money model

- All amounts are integer **millicents** (1 cent = 1000 millicents) so sub-cent
  per-call pricing never rounds badly.
- Balance is a cached integer backed by an append-only `ledger` table — the
  ledger is the source of truth and is fully reconstructable.
- **Failed work is never billed.** Funds are only captured on success, so there
  is no refund path to get wrong.

## Why idempotent + stateless workers

Workers take everything from `payload` and return everything in their result.
That makes retries safe, lets you run any number of workers in parallel, and
means a crashed worker mid-job costs nothing — the row is simply re-claimed.

## Scaling pathway

- **More throughput:** run more worker processes; `claim_next` already serializes
  claims safely. In production, swap the two SQLite queue helpers for Redis or a
  real broker (RabbitMQ/SQS) without touching callers.
- **More data:** the schema shards cleanly by `account_id`; lift onto Postgres
  when a single SQLite file is the bottleneck.
- **Autoscale signals:** queue depth, worker error rate, and request latency are
  the three metrics to drive scaling rules.

## Deliberate scaffold simplifications (production TODOs)

- The gateway processes jobs inline via a background thread for demo convenience.
  In production the gateway only enqueues; a separate worker fleet drains the queue.
- Crypto settlement (`SETTLEMENT_MODE=crypto`) is a documented seam, not wired.
- Webhook delivery is a recorded event stub — point it at the account's URL.
