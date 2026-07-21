# claude-skills

Single source of truth for my personal [Claude Code](https://claude.com/claude-code) skills.
Edit a skill here once; every project wired with the sync hook picks up the change
on its next Claude Code on the web session.

## Layout

```
manifest.txt            # list of skill files to sync (one path per line, relative to skills/)
skills/<name>/SKILL.md  # the skills themselves
hook/sync-skills.sh     # the sync hook to drop into any project
hook/settings.json      # the settings snippet that registers the hook
```

## Skills

| Skill | What it does | Upstream |
|-------|--------------|----------|
| **ponytail** | Laziest solution that actually works (YAGNI, stdlib first, shortest diff). | [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) |
| **caveman** | Ultra-compressed terse output to cut tokens, technical accuracy intact. | [JuliusBrussee/caveman](https://github.com/juliusbrussee/caveman) |
| **handoff** | Compresses & summarizes the conversation into a paste-ready HANDOFF.md to resume in another chat. | custom (this repo) |
| **goal** | Long-running goal continuation: give an objective and it auto-advances round by round (via `/loop`) until a completion audit passes. Multi-file; depends on the `/loop` skill. Chinese-language. | [limin112/claude-goal-skill](https://github.com/limin112/claude-goal-skill) |
| **impeccable** | Frontend design language: 23 `/impeccable` commands (craft, shape, audit, critique, polish, animate, …) with per-command references, design detectors, and anti-slop rules. Multi-file (108 files, Apache 2.0). | [pbakaus/impeccable](https://github.com/pbakaus/impeccable) |

## How it works

- **This repo must stay PUBLIC.** The hook fetches it with no auth. Primary transport is
  one shallow `git clone` of this repo — a single network call for the whole skill set,
  which matters now that the manifest is 100+ files (impeccable). If the clone fails it
  falls back to per-file `raw.githubusercontent.com` fetches, parallelized and retried
  (the GitHub API and repo tarballs are blocked in web sessions, and raw fetches
  rate-limit with HTTP 429 under bursty access — hence clone first).
- Each project carries a small **SessionStart hook** (`hook/sync-skills.sh`). On every
  web session it reads `manifest.txt` from this repo and copies each listed skill into
  `~/.claude/skills/`, making them available across that session. Commenting a line out
  of the manifest stops syncing that file.
- The hook is a **no-op locally** (`CLAUDE_CODE_REMOTE` guard) and **fails gracefully**
  if this repo is unreachable, so it never blocks a session.
- **Projects keep their own copy of the hook.** After the hook changes in this repo,
  re-run the wire snippet below in each project to refresh it. Old copies keep working
  against the current manifest — they just fetch serially over raw, so session start is
  slower until re-wired.

## Add a new skill

1. Add `skills/<name>/SKILL.md` (plus any extra files the skill needs).
2. Add each file's path (relative to `skills/`) to `manifest.txt`.
3. Commit + push. Every wired project gets it next session.

## Wire a new project (cloud)

Copy the hook into the project's repo and register it:

```bash
mkdir -p .claude/hooks
curl -sSL https://raw.githubusercontent.com/edgardoperrelli-maker/claude-skills/main/hook/sync-skills.sh -o .claude/hooks/sync-skills.sh
chmod +x .claude/hooks/sync-skills.sh
```

Then add this SessionStart entry to `.claude/settings.json`. The hook is named
`sync-skills.sh` (not `session-start.sh`) so it never clashes with a project's
own startup hook — if `.claude/settings.json` already has a `SessionStart` array,
just append this object as an extra element instead of replacing it:

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [ { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/sync-skills.sh" } ] }
    ]
  }
}
```

## Local setup (once per machine)

Locally `~/.claude/skills` is persistent, so you don't need the hook — just seed
it once. This reads `manifest.txt`, so it handles multi-file skills too:

```bash
BASE=https://raw.githubusercontent.com/edgardoperrelli-maker/claude-skills/main
curl -fsSL "$BASE/manifest.txt" | grep -v '^\s*#' | grep -v '^\s*$' | while read -r rel; do
  mkdir -p ~/.claude/skills/"$(dirname "$rel")"
  curl -fsSL --retry 4 --retry-delay 2 "$BASE/skills/$rel" -o ~/.claude/skills/"$rel"
done
```

Re-run to pull the latest. Restart Claude Code to load newly added skills.
