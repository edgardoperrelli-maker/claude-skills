# claude-skills

Single source of truth for my personal [Claude Code](https://claude.com/claude-code) skills.
Edit a skill here once; every project wired with the sync hook picks up the change
on its next Claude Code on the web session.

## Layout

```
manifest.txt            # list of skill files to sync (one path per line, relative to skills/)
skills/<name>/SKILL.md  # the skills themselves
hook/sync-skills.sh     # the sync hook to drop into any project
hook/settings.json      # the settings snippet: registers the hook + the plugins below
```

## Skills

| Skill | What it does | Upstream |
|-------|--------------|----------|
| **ponytail** | Laziest solution that actually works (YAGNI, stdlib first, shortest diff). | [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) |
| **caveman** | Ultra-compressed terse output to cut tokens, technical accuracy intact. | [JuliusBrussee/caveman](https://github.com/juliusbrussee/caveman) |
| **handoff** | Compresses & summarizes the conversation into a paste-ready HANDOFF.md to resume in another chat. | custom (this repo) |
| **grilling** | Interviews you relentlessly about a plan, decision, or idea to stress-test it before you build. Walks the design tree in rounds, asking the whole unblocked frontier at once and dispatching sub-agents for any fact it could look up itself. Also the default ticket type of `wayfinder`. | [mattpocock/skills](https://github.com/mattpocock/skills) |
| **goal** | Long-running goal continuation: give an objective and it auto-advances round by round (via `/loop`) until a completion audit passes. Multi-file; depends on the `/loop` skill. Chinese-language. | [limin112/claude-goal-skill](https://github.com/limin112/claude-goal-skill) |
| **wayfinder** | Multi-session "fog of war" planning: charts work too big for one session as a map of decision tickets on the issue tracker, then resolves them one at a time. User-invoked only (`/wayfinder`). Ships with the three skills it calls (below). | [mattpocock/skills](https://github.com/mattpocock/skills) |
| **domain-modeling** | Actively sharpens a project's domain model: challenges fuzzy terms, keeps `CONTEXT.md` as a pure glossary, writes ADRs sparingly. Wayfinder's default ticket type calls it alongside `grilling`. | [mattpocock/skills](https://github.com/mattpocock/skills) |
| **research** | Delegates reading legwork to a background agent, against primary sources only, and captures the findings as a cited Markdown file. Resolves wayfinder's `research` tickets. | [mattpocock/skills](https://github.com/mattpocock/skills) |
| **prototype** | Builds throwaway code that answers a design question: a single-file HTML state-machine demo, or several switchable UI variations on one route. Resolves wayfinder's `prototype` tickets. | [mattpocock/skills](https://github.com/mattpocock/skills) |
| **impeccable** | Frontend design language: 23 `/impeccable` commands (craft, shape, audit, critique, polish, animate, …) with per-command references, design detectors, and anti-slop rules. Multi-file (108 files, Apache 2.0). | [pbakaus/impeccable](https://github.com/pbakaus/impeccable) |
| **hallmark** | Anti-AI-slop design skill: makes generated UIs look made, not generated. One default design flow plus `audit` / `redesign` / `study` verbs, 20-theme catalog, 21 macrostructures, 58-gate slop test. Multi-file (108 files incl. the 24-theme OKLCH `tokens.css`; MIT). | [Nutlope/hallmark](https://github.com/Nutlope/hallmark) |
| **browser-harness** | Drives a real Chrome directly over CDP: coordinate clicks, screenshots, Python helpers, no selector hunting. **Needs the `browser-harness` CLI installed per machine** (see below); the markdown alone is inert. | [browser-use/browser-harness](https://github.com/browser-use/browser-harness) |


`wayfinder` is the one skill here with dependencies: it resolves each ticket type by
calling the Skill tool for `grilling`, `domain-modeling`, `research` or `prototype`, so
those travel with it. Two things it expects are deliberately *not* bundled: the map lives
on the repo's issue tracker, and upstream's `/setup-matt-pocock-skills` (which writes the
tracker's "Wayfinding operations" doc) is not installed — so wayfinder falls back to the
local-markdown tracker, whose operations ship as `skills/wayfinder/issue-tracker-local.md`.
Point it at a real tracker by writing that section yourself and referencing it from the
project's `CLAUDE.md`.

`browser-harness` is the only skill here that is **not self-contained**. Its SKILL.md
drives a `browser-harness` CLI that this hook does not install, so the skill is inert
until you run the one-time install on a machine that has a real Chrome to attach to:

```bash
uv tool install --python 3.12 --upgrade --force browser-harness
browser-harness <<'PY'
print(page_info())
PY
```

That is a local-machine step. Web sessions have no logged-in Chrome to drive, so the
synced markdown just sits there unused — harmless, but don't expect it to work there.
Upstream generates the skill body with `browser-harness skill`, which prints the copy
packaged into the installed CLI; `skills/browser-harness/SKILL.md` is that same file
vendored from the repo, so re-vendor it when you upgrade the CLI. Note its trigger is
deliberately broad ("always use browser-harness for any web interaction"), and the tool
executes Python against your real logged-in browser session.

## Plugins

Skills ride `manifest.txt` into `~/.claude/skills`. **Plugins don't** — they are declared
in settings and Claude Code installs them itself from their marketplace. `hook/settings.json`
carries both, so the wire snippet below gives a project the skills *and* the plugins.

| Plugin | What it does | Upstream |
|--------|--------------|----------|
| **open-code-review** | Two slash commands driving Alibaba's `ocr` review CLI: `/review` (OCR reviews with its own LLM, Claude filters the comments and applies the fixes) and `/delegate-review` (OCR picks the files and supplies the rules, Claude does the reviewing). | [alibaba/open-code-review](https://github.com/alibaba/open-code-review) (Apache 2.0) |

The plugin is *only* those two command files — the actual work is the `ocr` CLI, which it
does not bundle. Both commands install it on first use (`npm i -g @alibaba-group/open-code-review`),
so a fresh web session pays that install once, the first time you run one.

From there the two commands need different things:

- `/delegate-review` — **works as-is, no API key.** OCR only selects the files and hands
  over its review rules; the review itself runs on Claude's own model.
- `/review` — needs an LLM provider configured for OCR itself (`ocr config provider`;
  Anthropic, OpenAI or Bedrock protocols, also settable from the environment). That config
  is written on the machine, so it does not survive a web container. Treat `/review` as
  local-only unless you push the provider settings into the web environment yourself.

Upstream also ships a standalone `open-code-review` skill, but it lives outside the Claude
Code plugin (it targets Codex/Cursor) and covers the same ground as `/delegate-review`, so
it is deliberately not in `manifest.txt`.

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
```

No `chmod +x`: the settings entry below invokes the hook as an argument to `bash`,
so the file never needs the executable bit. That is deliberate — a copy landing
without it is the normal case, not the exception. `curl -o` clears it, and a file
committed through the GitHub contents API is always mode `100644`, because that
API has no way to set the bit. Under the old `chmod +x` wiring both of those
produce a hook that dies with *permission denied* at session start.

Then merge this into `.claude/settings.json`. The hook is named
`sync-skills.sh` (not `session-start.sh`) so it never clashes with a project's
own startup hook — if `.claude/settings.json` already has a `SessionStart` array,
just append this object as an extra element instead of replacing it:

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [ { "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/sync-skills.sh\"" } ] }
    ]
  },
  "extraKnownMarketplaces": {
    "open-code-review": {
      "source": { "source": "github", "repo": "alibaba/open-code-review" }
    }
  },
  "enabledPlugins": {
    "open-code-review@open-code-review": true
  }
}
```

Projects wired before this used `"command": "$CLAUDE_PROJECT_DIR/.claude/hooks/sync-skills.sh"`
and relied on `chmod +x`. Those keep working — their hook file does carry the bit — so there is
nothing to migrate. Use the `bash` form for anything wired from here on.

`extraKnownMarketplaces` + `enabledPlugins` are what replace typing `/plugin marketplace add`
and `/plugin install` in every project: Claude Code registers the marketplace and enables the
plugin on its own once the folder is trusted. Both keys take effect at **startup**, so a
project that just got them picks the plugin up on its next session, not the current one.

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

For the plugins, the per-project block above is only worth it in web sessions. Locally,
put the same `extraKnownMarketplaces` + `enabledPlugins` keys in `~/.claude/settings.json`
once and every project on the machine gets them — which is also the one place where
`/plugin marketplace add alibaba/open-code-review` and
`/plugin install open-code-review@open-code-review` do the same job by hand.
