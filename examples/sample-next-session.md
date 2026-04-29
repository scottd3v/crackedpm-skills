# Next session — pick up here

**Session ended:** 2026-04-27 · commit `a3f1c92`
**Branch:** `feat/payments-retry`

## Where we left off

Wired the exponential-backoff retry into the Stripe webhook handler; integration tests passing locally, CI still red on the flaky timeout case.

## Release-gate signal

🔴 FAIL — 1 release-blocker open

## Open work

### Release-blockers (1)
- **B-014** — Webhook retries double-fire on 5xx · `src/webhooks/stripe.ts:142` · investigation start: check whether `idempotency_key` is being regenerated on retry

### Nice-to-fix (2)
- **B-015** — Retry log lines lack request ID (one-line)
- **B-018** — Backoff cap of 30s feels short for off-hours traffic (one-line)

## Next action

B-014 — webhook retries double-fire on 5xx. Investigation start: check whether `idempotency_key` is being regenerated on retry instead of preserved across the retry chain.

## In-flight (uncommitted)

```
 M src/webhooks/stripe.ts
 M test/webhooks/stripe.spec.ts
?? .handoff/noise-filter
```

## Recent commits this session (newest first)

- `a3f1c92` feat(webhooks): exponential backoff with jitter
- `7e2d401` test(webhooks): add 5xx retry coverage
- `1b8a09c` refactor(webhooks): extract retry policy into module

## Skipping ahead — read first

- `docs/bug-log.md` — the canonical blocker list
- `docs/feature-inventory.md` — release dashboard
- `CLAUDE.md` — project conventions
