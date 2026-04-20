# crackedpm-skills

> Generic starter skills for operators running [Claude Code](https://claude.com/claude-code). Copy, fork, adapt.

Each skill is a plain markdown file you drop into `~/.claude/commands/` (for user-scope) or `<project>/.claude/commands/` (for project-scope). No build step. No plugin manifest. Just slash commands Claude picks up on session start.

These are genericized versions of skills that power a working operator setup. They assume **nothing** about your folder structure, knowledge-base tooling, or note-taking app — drop them in and they work.

## Install

Pick whichever fits your workflow.

### Clone + copy

```bash
git clone https://github.com/scottd3v/crackedpm-skills ~/dev/crackedpm-skills
mkdir -p ~/.claude/commands
cp -i ~/dev/crackedpm-skills/commands/*.md ~/.claude/commands/
```

### Symlink (get updates on `git pull`)

```bash
git clone https://github.com/scottd3v/crackedpm-skills ~/dev/crackedpm-skills
mkdir -p ~/.claude/commands
for f in ~/dev/crackedpm-skills/commands/*.md; do
  ln -sf "$f" ~/.claude/commands/$(basename "$f")
done
```

### Verify

Start Claude Code in any directory and type `/`. You should see the imported commands in the picker.

## Skills

| Skill | What it does |
|---|---|
| [`/log-session`](commands/log-session.md) | Writes a session journal to `log/` (or wherever you configure) with a copy-pasteable resume prompt for your next session. The pattern behind [`/preflight/claude-code/cc-12`](https://crackedpm.ai/preflight/claude-code/cc-12-sessions-and-parallel). |

_More starter skills coming as they're genericized from their project-specific originals: `/new-skill`, `/ingest`, `/lint`._

## Configure per-project

Each skill works with zero config, but you can tune behavior by adding lines to your project's `CLAUDE.md`:

```markdown
## Session logs
- Write to `docs/sessions/` instead of `log/`
- Prefix filenames with branch name, not date
```

Skills read `CLAUDE.md` conventions at session start and respect them. Override defaults to taste.

## Contribute

Pull requests welcome. Criteria for accepting a skill:

1. **Generic by default** — works without assuming a specific vault, tool, or folder structure.
2. **One job** — resist bundling multiple concerns into one skill.
3. **Configurable via `CLAUDE.md`** — don't hardcode paths.
4. **Under 150 lines** — skills should be skimmable, not epic.
5. **Include an `examples/` sample** showing the artifact the skill produces.

## License

MIT. See [LICENSE](LICENSE).

## Credit

Extracted from patterns developed in [reed-fwm](https://github.com/scottd3v/reed-fwm) (private) — the working operator vault where these skills started. The lesson behind the pattern: [crackedpm.ai/preflight/claude-code/cc-12-sessions-and-parallel](https://crackedpm.ai/preflight/claude-code/cc-12-sessions-and-parallel).
