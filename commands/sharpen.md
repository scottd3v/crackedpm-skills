# /sharpen

Turn a fuzzy goal into a crisp first-prompt for a fresh Claude Code session.

**When to use:**
- Starting a new CC session for work whose goal is vague ("launch my app", "clean up this repo").
- You know roughly what you want but not what to *ask* the agent to do first.

**What it does:** runs an adaptive 5-dimension dialogue, produces a structured prompt, copies it to your clipboard, and tells you to `/clear` + paste into a clean session.

## Invocation

- `/sharpen` — bare. Ask the user: "What's the rough idea?"
- `/sharpen <rough idea>` — `$ARGUMENTS` contains the rough idea. Skip the initial prompt.

## The 5 dimensions

The dialogue fills in five slots. Each produces one line of the final prompt.

| # | Dimension  | Asks | Example value |
|---|------------|------|---------------|
| 1 | Outcome    | What does "done" look like for *this* session? | "pre-launch App Store checklist" |
| 2 | Sources    | Which files or URLs should the agent read first? | `your-app/`, TestFlight stats — or reply `find` to autodiscover |
| 3 | Ruled out  | What's already been tried or is off the table? | "no pricing today, no marketing" |
| 4 | Constraints| Time / budget / anti-goals | "solo, weekend, no hiring" |
| 5 | Format     | How should the output come back? | "markdown checklist, paste-ready" |

## Dialogue rules

1. **Parse the rough idea.** Infer any dimension values that are already stated. Mark them 🟡 (inferred) until the user confirms. Unstated dimensions are ⚪ (blank).
2. **Render the dashboard every turn.** Show the current state before asking anything.
3. **Ask one dimension at a time** — the leftmost unfilled (🟡 or ⚪) one. Prefer multiple choice over open-ended when proposing values.
4. **Confirm inferred (🟡) values** rather than re-prompting. Example: "I inferred `sources = your-app repo`. Confirm, add more, or override."
5. **Escape hatch:** at any turn, the user can say "ship it" / "good enough". Produce the prompt with `# TBD: <dim>` markers for any ⚪ slots.
6. **Contradiction handling:** if the user changes a previously-green answer, reset that dimension to 🟡 and re-confirm before locking.

## Visual dashboard (render every turn)

```
/sharpen — forming your prompt

Rough idea: <echoed>

[🟢] Outcome       <value>
[🟡] Sources       <inferred — confirm?>
[⚪] Ruled out
[⚪] Constraints
[⚪] Format

Progress: ▰▰▱▱▱  2/5

─── Now asking ───
<one question>
```

- Legend: 🟢 confirmed · 🟡 inferred · ⚪ blank
- **Progress math — count ONLY 🟢 dimensions.** Do not count 🟡 or ⚪.
  - `▰` = one per 🟢 dimension.
  - `▱` = one per 🟡 or ⚪ dimension.
  - Counter `N/5` = number of 🟢 dimensions (NOT turns taken, NOT inferred values).
  - **Example — state: 2🟢, 2🟡, 1⚪ → `▰▰▱▱▱  2/5`**  (NOT `▰▰▰▰▱ 2/5`)
  - **Example — state: 4🟢, 1⚪ → `▰▰▰▰▱  4/5`**  (NOT `▰▰▰▰▰ 4/5`)
- Keep the dashboard under 15 lines — token-efficient.

## Source autodiscovery

Triggered only when the user replies `find` (or `find <topic>` / `find in <dir> <dir>`) to the Sources question. Otherwise, Sources works the same as any other dimension — user names paths manually.

### Inputs

- **Topic keyword** —
  - `find <topic>` → use `<topic>` directly
  - `find` alone → extract from `$ARGUMENTS` + the locked outcome
  - If no keyword can be inferred → ask "what project or topic should I search for?" before proceeding
- **Search roots** —
  - `find in <dir1> <dir2> …` → use those exact paths
  - `find` alone → ask the user: "where should I look? (paste paths, or reply `home` to scan `$HOME`)"
  - **Never hardcode personal paths** — always ask or accept from user input

### Algorithm

1. For each search root, run two passes:
   - `Glob` filename matches: `<root>/**/*<keyword>*` (depth ≤ 4)
   - `Grep` content matches: `<keyword>` in `*.md`, `*.swift`, `README*` (output mode: files-with-matches)
2. Exclude any path containing: `node_modules`, `.git`, `build`, `DerivedData`, `.next`, `.venv`, `dist`, `Pods`, `target`
3. Rank results by file mtime (most recently modified first). Cap at **10 candidates**.
4. Present as a numbered list with path and human-readable age:
   ```
   Found 5 candidates for "launch":
     1.  your-app/docs/launch-plan.md         (2d ago)
     2.  notes/2025-11-launch-beta.md         (6d ago)
     3.  your-app/README.md                   (14d ago)
     4.  your-app/CHANGELOG.md                (21d ago)
     5.  inbox/launch-ideas.md                (38d ago)

   Which should the agent read? (numbers / all / none / paste more)
   ```
5. Lock the user's picks as the Sources value; continue the dialogue.

### Degradation

| Situation | Behavior |
|---|---|
| Zero hits | Fall back to the normal Sources question: "no matches — paste source paths instead" |
| Keyword too generic (e.g. "launch", "app" — common words with >100 false-positive hits likely) | Ask the user to narrow: "that keyword is broad — give me a more specific phrase" |
| User replies `none` | Lock Sources as empty; the output template's `Read first:` will use the no-sources fallback |

### Invariant

**Do NOT `Read` candidate files during autodiscovery.** The skill's job is to list paths — reading them is the downstream cleared session's job. Reading here wastes tokens in the prep pass.

## Output template (clipboard payload)

When all dimensions are locked (or the user shipped early), produce:

```
Goal: <one declarative sentence>

Read first:
  - <path or URL>
  - <path or URL>

Context:
  Stage: <where things are now>
  Ruled out: <what to skip>

Constraints: <deadline · budget · anti-goals>

Output: <format — checklist / doc / code / memo>
```

For any unfilled slot after "ship it", substitute `# TBD: <dim>` so the user sees what to fill in before pasting.

## End-of-skill sequence

1. Render a **before/after diff block** — the rough idea on the left, the polished prompt on the right. Compact, token-efficient (no heavy Unicode).
2. Copy the polished prompt to the clipboard. Try in order:

   ```bash
   if command -v pbcopy >/dev/null 2>&1; then
     printf '%s' "$PROMPT" | pbcopy
   elif command -v xclip >/dev/null 2>&1; then
     printf '%s' "$PROMPT" | xclip -selection clipboard
   elif command -v wl-copy >/dev/null 2>&1; then
     printf '%s' "$PROMPT" | wl-copy
   else
     # Fallback: print with delimiters for manual copy
     echo "══════ BEGIN PROMPT ══════"
     echo "$PROMPT"
     echo "══════ END PROMPT ══════"
   fi
   ```

3. Print the handoff message:

   ```
   Copied. Run /clear, then ⌘V to paste.
   ```

   (If the fallback path was used, instead: `Clipboard tool not found — copy the block above manually, then /clear and paste.`)

## Edge cases

| Situation | Behavior |
|-----------|----------|
| User exits early ("ship it") | Produce prompt with `# TBD: <dim>` markers. Still copy. |
| No sources named by end | Output template's "Read first" becomes `explore <likely dir>` as the first line. |
| Clipboard chain fails | Print with `══════` delimiters + fallback message. |
| `$ARGUMENTS` fills all 5 dimensions | Skip dialogue; show dashboard (all 🟢), produce + copy in one turn. |
| User contradicts earlier answer | Reset that dimension to 🟡; re-confirm before re-locking. |
| User replies `find` on Sources | Trigger Source autodiscovery (see dedicated section). |

## Non-goals

- No prompt history saved to disk. This is a future feature.
- No auto-executing `/clear` — destructive action stays in user's hands.
- The skill never runs the produced prompt itself.
- No Windows clipboard (`clip.exe`) yet.
