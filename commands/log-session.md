---
description: Write a session journal so the next session can pick up without re-explaining.
---

# /log-session

End the current Claude Code session by writing a markdown file that captures what changed, what's next, and the exact prompt to paste into your next session.

## When to use it

- You're wrapping up a work session and want to leave yourself a trail.
- You're about to switch machines, branches, or projects.
- You hit a natural stopping point and want to preserve context before it evaporates.

Don't use it for every conversation — use it when the session produced something worth resuming from.

## Arguments

$ARGUMENTS

Optional: a short summary phrase to use in the filename. If empty, derive one from what actually happened.

## Instructions

### 1. Determine the log location

Default path: `log/` in the current working directory.

Override: if `CLAUDE.md` in this project (or `~/.claude/CLAUDE.md` globally) specifies a different directory under a `## Session logs` section, use that instead.

If `log/` doesn't exist, create it.

### 2. Determine the filename

Format: `<date>-<slug>.md` where:
- `<date>` is today in `YYYY-MM-DD`
- `<slug>` is kebab-case, 2-4 words, derived from the work done (or the `$ARGUMENTS` if provided)

Examples: `2026-04-19-oauth-bugfix.md`, `2026-04-19-shipping-v2.md`.

### 3. Write the frontmatter

```yaml
---
title: "Session: <summary>"
date: YYYY-MM-DD
started: "YYYY-MM-DD HH:MM"
ended: "YYYY-MM-DD HH:MM"
branch: <current git branch, if in a git repo>
resume: "claude --resume <session-name>"
---
```

`started` — estimate from the beginning of the conversation.
`ended` — now.
`resume` — only meaningful if the user named the session with `/rename` or `claude --name <name>`.

### 4. Write the three sections that matter

```markdown
## What happened

Two or three sentences. What did we work on? What's the state at the end?

## Files changed

- `path/to/file.ts` — one line on what changed
- `another/file.md` — one line

(Skip if no files changed — conversations can be worth logging too.)

## Decisions made

Tradeoffs or choices that aren't obvious from the diff. Link to any
ADR / design doc / PR you created. Skip this section if everything
was obvious.
```

### 5. Write the improvement suggestions section

```markdown
## Improvement suggestions

- What was slow or high-friction this session?
- What got repeated that could be automated or turned into a skill?
- What worked well and should be kept?
```

This section is the compounding layer. Every future session gets a little better because past sessions wrote down what hurt.

### 6. Write the next-session prompt

This is the section that matters most. It's a copy-paste block for future you.

```markdown
## Next Session Prompt

> Copy-paste this to start your next Claude Code session:

\`\`\`
Hey Claude — resuming from <session-name>.

Read these first:
1. <log file you're writing now>
2. <most important file or doc from this session>
3. <second-most-important if relevant>

The state at end of last session: <one sentence>.

Next up: <one sentence on the first thing to do>.
\`\`\`
```

Keep it under 10 lines. Longer prompts don't get pasted; they get re-read and edited.

### 7. Save the file

Write the markdown to the path from step 1.

### 8. Commit (optional)

If the current project is a git repo and the user's convention is to commit logs:

```bash
git add <log-file>
git commit -m "log: <summary>"
```

Skip commit if the user hasn't indicated they want session logs in git (check `CLAUDE.md` for a convention; when in doubt, don't commit).

### 9. Report

Tell the user:
- Path of the file you wrote
- (If committed) the commit hash
- Hand them the resume-prompt block so they can copy it immediately if they want

## Configuring per-project

Add this section to your project's `CLAUDE.md` to tune behavior:

```markdown
## Session logs

- Location: docs/sessions/   # default is log/
- Commit: yes                # default is no
- Naming: <branch>-<slug>    # default is <date>-<slug>
```

Respect whatever the user's `CLAUDE.md` says. If it says nothing, use defaults.

## What not to do

- Don't write a log for trivial conversations ("fix this typo"). Not every session needs one.
- Don't fabricate timestamps you don't know — estimate honestly.
- Don't assume the reader (future you or someone else) has the same vocabulary; write so a cold reader can pick up the thread.
- Don't push to a remote. Commit locally if the project convention calls for it; let the user push.

## Canonical pattern reference

The lesson behind this skill: [crackedpm.ai/preflight/claude-code/cc-12-sessions-and-parallel](https://crackedpm.ai/preflight/claude-code/cc-12-sessions-and-parallel).
