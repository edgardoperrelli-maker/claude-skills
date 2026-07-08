# claude-skills

Single source of truth for my personal [Claude Code](https://claude.com/claude-code) skills.
Edit a skill here once; every project wired with the sync hook picks up the change
on its next Claude Code on the web session.

## Layout

```
manifest.txt            # list of skill files to sync (one path per line, relative to skills/)
skills/<name>/SKILL.md  # the skills themselves
hook/session-start.sh   # the sync hook to drop into any project
hook/settings.json      # the settings snippet that registers the hook
```

## Skills

| Skill | What it does | Upstream |
|-------|--------------|----------|
| **ponytail** | Laziest solution that actually works (YAGNI, stdlib first, shortest diff). | [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) |
| **caveman** | Ultra-compressed terse output to cut tokens, technical accuracy intact. | [JuliusBrussee/caveman](https://github.com/juliusbrussee/caveman) |
| **handoff** | Compresses & summarizes the conversation into a paste-ready HANDOFF.md to resume in another chat. | custom (this repo) |

## How it works

- **This repo must stay PUBLIC.** The hook fetches files via `raw.githubusercontent.com`
  with no auth (Claude Code on the web blocks the GitHub API and repo tarballs, so raw
  files are the only reliable transport).
- Each project carries a small **SessionStart hook** (`hook/session-start.sh`). On every
  web session it reads `manifest.txt` from this repo and downloads each listed skill into
  `~/.claude/skills/`, making them available across that session.
- The hook is a **no-op locally** (`CLAUDE_CODE_REMOTE` guard) and **fails gracefully**
  if this repo is unreachable, so it never blocks a session.

## Add a new skill

1. Add `skills/<name>/SKILL.md` (plus any extra files the skill needs).
2. Add each file's path (relative to `skills/`) to `manifest.txt`.
3. Commit + push. Every wired project gets it next session.

## Wire a new project (cloud)

Copy two files into the project's repo and commit them:

```bash
mkdir -p .claude/hooks
curl -sSL https://raw.githubusercontent.com/edgardoperrelli-maker/claude-skills/main/hook/session-start.sh -o .claude/hooks/session-start.sh
chmod +x .claude/hooks/session-start.sh
# then merge hook/settings.json into the project's .claude/settings.json
```

`.claude/settings.json` (create or merge):

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [ { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/session-start.sh" } ] }
    ]
  }
}
```

## Local setup (once per machine)

Locally `~/.claude/skills` is persistent, so you don't need the hook — just seed it once:

```bash
for s in ponytail caveman handoff; do
  mkdir -p ~/.claude/skills/$s
  curl -sSL https://raw.githubusercontent.com/edgardoperrelli-maker/claude-skills/main/skills/$s/SKILL.md -o ~/.claude/skills/$s/SKILL.md
done
```

Re-run to pull the latest. Restart Claude Code to load newly added skills.
