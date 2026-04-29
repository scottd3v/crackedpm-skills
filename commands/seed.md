---
description: Bootstrap the /handoff + /resume cycle in a fresh repo. Detects project type, drops a noise filter, writes a starter next-session.md.
---

# /seed

One-shot setup for the `/handoff` ↔ `/resume` cycle in a new project. Detects project type, drops a project-aware `.handoff/noise-filter`, generates a starter `next-session.md` so [`/resume`](resume.md) works from day one.

Pairs with [`/handoff`](handoff.md) (write side) and [`/resume`](resume.md) (read side). `/seed` is the install step.

## When to use it

- First time in a repo where you want the `/handoff` ↔ `/resume` cycle to work.
- After cloning a project a teammate was using `/handoff` on, to install your own per-project tuning.

Idempotent — re-running skips files that already exist. Designed as a one-time setup; for ongoing snapshots, use `/handoff`.

## Arguments

$ARGUMENTS

Ignored. Reserved for future flags.

## Instructions

### 1. Detect repo

```
REPO=$(git rev-parse --show-toplevel)
```

Halt if not in a git repo: *"seed requires a git repository — `cd` to a project root and re-run."*

### 2. Don't clobber existing setup

Check for prior `/seed` evidence. If any of these exist, summarize what's there and skip with a notice:
- `<REPO>/.handoff/noise-filter`
- `<REPO>/docs/next-session.md` or `<REPO>/next-session.md`

Default behavior: skip files that already exist; only create what's missing. Tell the user explicitly which files were created vs skipped.

### 3. Detect project type

Walk top-level files only (fast). Match the FIRST hit; if multiple match, list both as multi-language.

| Signal file(s) | Project type |
|---|---|
| `*.xcodeproj`, `*.xcworkspace`, `Package.swift`, `Podfile` | swift-ios-macos |
| `package.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb` | node-web |
| `pyproject.toml`, `setup.py`, `requirements.txt`, `Pipfile` | python |
| `Cargo.toml` | rust |
| `go.mod` | go |
| `Gemfile` | ruby |
| `pom.xml`, `build.gradle*` | jvm |
| `mix.exs` | elixir |
| `*.csproj`, `*.sln` | dotnet |
| (no match) | generic |

Also detect:
- Test surface: `test/`, `tests/`, `*Tests/`, `__tests__/`
- `docs/` directory presence
- `CLAUDE.md` / `AGENTS.md` / `README.md` presence

### 4. Generate `.handoff/noise-filter`

Universal section (all projects):

```
# Universal — OS / editor / scratch noise
^.. \.DS_Store$
^.. \.idea/
^.. \.vscode/
^.. \.swp$
^.. \.tmp$
^.. \.log$
```

Append per-type rules based on detection:

- **swift-ios-macos:** `xcuserdata/`, `DerivedData/`, `\.build/`, `Package\.resolved$`, `Pods/`
- **node-web:** `node_modules/`, `\.next/`, `dist/`, `build/`, `\.cache/`, `coverage/`
- **python:** `__pycache__/`, `\.venv/`, `venv/`, `\.pytest_cache/`, `\.mypy_cache/`, `dist/`, `.*\.egg-info/`
- **rust:** `target/`
- **go:** `vendor/`, `bin/`
- **ruby:** `\.bundle/`, `vendor/bundle/`
- **jvm:** `target/`, `build/`, `\.gradle/`
- **elixir:** `_build/`, `deps/`
- **dotnet:** `bin/`, `obj/`

Each rule is prefixed with `^.. ` to match `git status --short` output. Tag the file with a header comment naming the detected project type so future `/seed` re-runs (or human readers) know what generated it.

### 5. Generate the starter `next-session.md`

Output path: `<REPO>/docs/next-session.md` if `docs/` exists, else `<REPO>/next-session.md`.

```markdown
# Next session — pick up here

**Session ended:** YYYY-MM-DD · commit `<SHA>` · seeded by `/seed`
**Branch:** `<branch>`

## Where we left off

`/seed` ran on this repo on YYYY-MM-DD. No prior session-handoff exists. Project orientation below; first real handoff happens at the end of the next session that runs `/handoff`.

## Project orientation (auto-detected by /seed)

- **Type:** <swift-ios-macos | node-web | python | ... | generic>
- **Top-level dirs:** <list of non-dotfile dirs>
- **Test surface:** <"present at <path>" if found, else "none detected">
- **Existing orientation files:** <CLAUDE.md / AGENTS.md / README.md — list which exist>
- **Build / package manifest:** <Package.swift, package.json, pyproject.toml, etc.>

## Next action

If you came in with a goal, state it. If not — read the orientation files (in order: AGENTS.md → CLAUDE.md → README.md, whichever exist) and pick the first concrete TODO from there.

## In-flight (uncommitted)

<filtered git-status-short via the new noise-filter, one line per file>

## Recent commits (newest 10)

- `<sha>` <commit subject>

## Skipping ahead — read first

<bullet list of orientation files that exist, in priority order>
```

### 6. Optionally suggest tier-2 scaffolds

If `/handoff`'s richer state would help (release-blocker tracking, feature inventory), point at the optional formats with one sentence each:

- `docs/bug-log.md` with `## Release blockers` / `## Nice to fix` / `## Closed` sections + `### B-NNN · severity · title` stanzas → unlocks the Release-gate signal + Open work sections of `/handoff`'s output.
- `docs/feature-inventory.md` with a dashboard at the top → unlocks the inventory snapshot in `/handoff`.

Don't auto-create them. Mention once; let the user decide.

### 7. Final summary

```
/seed complete in <REPO>

Created:
  · .handoff/noise-filter        (<project-type> defaults)
  · <docs/>next-session.md       (orientation; /resume reads this)

Skipped (already present):
  <list, if any>

How to use:
  · End each session: /handoff
  · Start each session: /resume
  · Edit .handoff/noise-filter when in-flight clutter creeps back in

Recommend committing the new files now — they're project artifacts, not transient state.
```

Don't auto-commit. The user reviews + commits.

## Configuring per-project

Add this section to your project's `CLAUDE.md` to tune behavior post-seed:

```markdown
## Handoff

- Output: docs/next-session.md   # default: docs/ if it exists, else root
- Noise filter: .handoff/noise-filter   # generated by /seed; edit to taste
```

## What this skill does NOT do

- Doesn't auto-commit. New files are left for the user's review.
- Doesn't run `/init`. If the project has no `CLAUDE.md`, `/seed` notes that in the orientation block — generating one is `/init`'s job.
- Doesn't modify existing files. Skips them with a notice.
- Doesn't add `.handoff/` to `.gitignore`. By default the noise-filter is committed alongside the project so the convention travels.

## Failure modes

| Situation | Behavior |
|---|---|
| Not in a git repo | Halt with the message above. |
| Repo has no commits yet | Skip "Recent commits" section in the seed handoff; everything else still works. |
| Multiple project types match (e.g. `package.json` + `Cargo.toml` for a Tauri app) | Combine the per-type rule sets, deduplicate. Tag both types in orientation. |
| `<REPO>/.handoff/noise-filter` exists | Read it, summarize, ask whether to replace, append-and-merge, or keep as-is. Default to keep-as-is. |
| `<REPO>/docs/next-session.md` exists | Skip writing it; tell the user. The starter handoff is only for fresh repos. |
