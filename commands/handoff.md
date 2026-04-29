---
description: End-of-session snapshot. Writes a single `next-session.md` so tomorrow's session picks up cold.
---

# /handoff

End the current Claude Code session by snapshotting where you left off. Writes one file — `next-session.md` — that tomorrow's session reads first to be up to speed without re-deriving context from git history.

Pairs with [`/seed`](seed.md) (first-time setup) and [`/resume`](resume.md) (start-of-session reader).

## When to use it

- You're wrapping up a work session and want the next one to pick up cold.
- The session produced enough state that re-grepping git in the morning would be wasteful.

Don't run mid-session. This is a wrap-up artifact; the file is overwritten each run, so the most recent handoff is the only one that survives.

## Arguments

$ARGUMENTS

Optional: a one-line override of the "where we left off" sentence. If empty, derive it from the topmost commit subject.

## Instructions

### 1. Detect repo + decide output path

```
REPO=$(git rev-parse --show-toplevel)
```

Halt if not in a git repo: *"handoff requires a git repository — `cd` to a project root and re-run."*

Output path, in priority order:
1. `<REPO>/next-session.md` if it already exists (preserve the user's chosen location)
2. `<REPO>/docs/next-session.md` if it already exists, OR if `<REPO>/docs/` exists for a fresh repo
3. `<REPO>/next-session.md` otherwise

### 2. Probe optional inputs

Read each only if present — no errors, no placeholders for absent files:

- `<REPO>/.handoff/noise-filter` — line-separated regexes for `git status` paths to exclude. Falls back to a sensible default (`\.DS_Store$`, `\.swp$`, `\.tmp$`, `\.log$`).
- `<REPO>/docs/bug-log.md` — if it has `## Release blockers` / `## Nice to fix` / `## Closed` sections + a `**Open bugs:** N release-blocker · M nice-to-fix` header line, treat as tier-2.
- `<REPO>/docs/feature-inventory.md` — if it has a dashboard at the top, pull the first ~6 lines as a tier-2 input.
- Newest `<REPO>/test-results/*.json` — if `summary` is parseable via `jq`, include it.

### 3. Pull state

**Tier 1 (always available):**
- Recent commits this session: find the start-of-session marker via `git log --since=midnight --oneline`, falling back to last 10 commits with an "(boundary inferred)" note.
- Branch: `git branch --show-current`
- Working tree filtered through the noise-filter: `git status --short | grep -vE -f <REPO>/.handoff/noise-filter`

**Tier 2 (only if the corresponding file exists):**
- Bug-log header: `grep '^\*\*Open\|^\*\*Closed\|^\*\*Last' <REPO>/docs/bug-log.md`
- Open release-blockers: `awk '/^## Release blockers/{p=1;next} /^## /{p=0} p && /^### B-/' <REPO>/docs/bug-log.md`. Don't use awk's `,` range — it includes the start line and matches nothing useful.
- Open nice-to-fix: same pattern scoped to `## Nice to fix`.
- Inventory dashboard: first ~6 lines of `<REPO>/docs/feature-inventory.md`.
- Latest test summary: `jq '.summary' <newest test-results JSON>`.

**The single next action:** topmost open release-blocker (if tier-2 bug-log present), or topmost nice-to-fix, or — if neither exists — quote the topmost commit subject verbatim with the literal suffix " — continue from here."

If the topmost commit is a closure (subject contains `killed`, `shipped`, `closed`, `done`, `fixed`, `resolved`) and the user stated an explicit forward-looking next in the conversation ("save X for tomorrow", "next is Y"), quote that instead.

### 4. Write `next-session.md`

Overwrite, don't append. Last session wins. Sections without data are **omitted entirely** — no `_(none)_` filler.

```markdown
# Next session — pick up here

**Session ended:** YYYY-MM-DD · commit `<SHA>`
**Branch:** `<branch>`

## Where we left off

<one sentence — paraphrase the topmost commit's subject, or use $ARGUMENTS if provided>

## Release-gate signal               (omit if no bug-log)

🟢 CLEAR  |  🔴 FAIL — N release-blocker(s) open

## Open work                         (omit if no bug-log)

### Release-blockers (N)
- **B-NNN** — title · code site · investigation start point

### Nice-to-fix (N)
- **B-NNN** — title (one-line)

## Next action

<topmost open blocker or nice-to-fix verbatim from bug-log; otherwise "continue from <topmost commit subject>">

## In-flight (uncommitted)           (omit if working tree is clean)

<filtered git-status-short, one line per file>

## Recent commits this session (newest first)

- `<sha>` <commit subject>

## Skipping ahead — read first       (only present if these files exist)

- `<REPO>/docs/bug-log.md` — if present
- `<REPO>/docs/feature-inventory.md` — if present
- Project orientation: `AGENTS.md` / `CLAUDE.md` / `README.md` (whichever exist)

## Latest test summary               (omit if no test-state file)

<jq summary block from the newest test-results JSON>
```

### 5. Confirm to user

Print to stdout:

```
handoff snapshot written → <output-path>
  · last commit: <SHA>
  · open release-blockers: N (or "no bug-log present")
  · next action: <one-line>
```

Don't auto-commit. The user reviews + commits per their preference.

## Configuring per-project

Add this section to your project's `CLAUDE.md` to tune behavior:

```markdown
## Handoff

- Output: docs/next-session.md   # default: docs/ if it exists, else root
- Noise filter: .handoff/noise-filter   # one regex per line; comments start with #
```

Or just create the files directly:

```bash
mkdir -p .handoff
cat > .handoff/noise-filter <<'EOF'
# Project-specific git-status noise to exclude from In-flight
^.. node_modules/
^.. \.DS_Store$
EOF
```

Use [`/seed`](seed.md) to bootstrap both at once for a new project.

## Want a Slack/Discord TL;DR?

Out of scope for this skill — fork it and add a step 4b that posts the `Where we left off` + `Next action` sections to whatever channel you want. The local file is the source of truth either way.

## What this skill does NOT do

- Doesn't auto-commit. Stages nothing.
- Doesn't push.
- Doesn't synthesize prose creatively. Quote the topmost open bug verbatim if available; otherwise quote the commit subject.
- Doesn't update any project memory file.
