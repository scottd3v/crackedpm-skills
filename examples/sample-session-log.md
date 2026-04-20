---
title: "Session: oauth refactor"
date: 2026-04-19
started: "2026-04-19 14:20"
ended: "2026-04-19 16:05"
branch: feat/oauth-refactor
resume: "claude --resume oauth-refactor"
---

## What happened

Refactored the OAuth token lifecycle to use per-client IDs instead of the shared LLAT. Migrated three call sites, added a token-rotation helper, and wrote integration tests for the happy path + one failure mode (token expiry mid-request). Tests green. PR not opened yet.

## Files changed

- `src/auth/oauth-client.ts` — introduced `PerClientTokenManager`; deprecated `SharedLLATProvider`
- `src/auth/rotation.ts` — new helper for quarterly rotation
- `tests/auth/oauth-client.test.ts` — 8 new cases
- `docs/auth-migration.md` — one-pager for other contributors

## Decisions made

- Kept the old `SharedLLATProvider` behind a `--legacy-auth` flag for the one-release deprecation window. Removal tracked in issue #412.
- Chose per-client OAuth over per-user because three of our services are multi-tenant. Per-user doesn't scale.

## Improvement suggestions

- Token-rotation tests took ~40s to run. Mock the clock next time instead of using real `setTimeout`.
- Had to look up the Cognito error codes three separate times. Worth a small `docs/error-codes.md` cheat sheet.
- `/log-session` catches this stuff — use it more at end of every day.

## Next Session Prompt

> Copy-paste this to start your next Claude Code session:

```
Hey Claude — resuming from oauth-refactor.

Read these first:
1. log/2026-04-19-oauth-refactor.md
2. src/auth/oauth-client.ts
3. docs/auth-migration.md

The state at end of last session: per-client OAuth implemented, tests green, PR not opened yet.

Next up: open the PR, request review from the auth team, and start migrating the fourth call site (the admin dashboard).
```
