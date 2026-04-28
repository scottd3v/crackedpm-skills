---
description: Pick up cold from a prior session's /handoff. Reads next-session.md and orients to where work left off.
---

# /resume

Pick up cold from a prior session's [`/handoff`](handoff.md). Reads the project's `next-session.md` and orients to where work left off, without re-deriving context from git history.

Pairs with [`/handoff`](handoff.md). End of session → `/handoff` writes the snapshot. Start of next session → `/resume` reads it.

## When to use it

- First action when opening Claude Code in a project that has been touched recently and may have a `next-session.md` waiting.
- After `/clear` mid-flight when you want to reload the most recent state without scrolling back.

## Arguments

$ARGUMENTS

Ignored.

## Instructions

### 1. Detect repo + handoff file

```
REPO=$(git rev-parse --show-toplevel)
```

Halt if not in a git repo: *"resume requires a git repository — `cd` to a project root and re-run."*

Look for the handoff file in this exact order. Stop at the first one that exists:
1. `<REPO>/docs/next-session.md`
2. `<REPO>/next-session.md`

If none found, halt with: *"No `next-session.md` found at `<REPO>/docs/` or `<REPO>/`. The prior session didn't `/handoff`, or this is a fresh project. Run `/handoff` at the end of any non-trivial session — or [`/seed`](seed.md) to bootstrap."*

### 2. Sanity-check freshness

Compare the handoff file's `**Session ended:**` date against today:
- Same day or yesterday → fresh, proceed silently.
- 2–7 days old → proceed but note the gap: *"Handoff is N days old — git history since then may have moved past it."*
- 8+ days old → flag prominently: *"Handoff is N days old. Verify against `git log --since='<handoff-date>' --oneline` before trusting the 'next action'."*

### 3. Read it in full + orient

Read `<handoff-file>` end-to-end. Then to the user, in this exact order:

1. **One sentence: where we left off.** Quote or paraphrase the "Where we left off" section.
2. **Release-gate state, if present.** 🟢 CLEAR or 🔴 N blockers.
3. **Next action, verbatim from the file.** Don't synthesize. Just paste.
4. **Ask the user how they want to proceed:**
   - *"a) execute the next action now"*
   - *"b) inspect first — show me the file and I decide"*
   - *"c) something else (you describe)"*

Don't auto-execute. The user signs off. They may have come in with a different priority than yesterday's handoff anticipated.

### 4. Cross-check (silent unless something's off)

Pull `git log <handoff-commit>..HEAD --oneline` and `git status --short`. If either has unexpected content (commits past the handoff that aren't yours, uncommitted changes the handoff didn't list), surface it. Otherwise stay quiet.

### 5. Hand off to the user

Once they pick a path, drop into their chosen mode. `/resume`'s job ends here — it doesn't execute the next action itself.

## Configuring per-project

`/resume` reads whatever path `/handoff` writes to. If you've configured a non-default path in `CLAUDE.md`:

```markdown
## Handoff

- Output: docs/sessions/next-session.md
```

`/resume` checks the same locations in the same priority order as `/handoff` writes.

## What this skill does NOT do

- Doesn't auto-execute the next action. The user decides.
- Doesn't update the handoff file. (`/handoff` writes; `/resume` reads.)
- Doesn't summarize from git history. The handoff file is canonical for "what happened last session."
- Doesn't fetch / pull / sync. Local handoff file is the source of truth.

## Failure modes

| Situation | Behavior |
|---|---|
| No `next-session.md` anywhere | Halt with the message above. Suggest `/handoff` at end of next session, or `/seed` for first-time setup. |
| `next-session.md` exists but is empty / malformed | Show what's there + offer to git-log-derive a quick orientation as fallback. |
| Handoff date is in the future (clock skew, file edited manually) | Flag, treat as fresh. |
| Fresh `/clear` mid-session with an open `next-session.md` from this same session | Read normally. The handoff snapshots the most recent commit boundary; pre- and post-clear are continuous. |
