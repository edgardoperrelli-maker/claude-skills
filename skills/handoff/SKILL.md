---
name: handoff
description: >
  Compress and summarize the current conversation into a paste-ready handoff
  document so a fresh chat (or another agent) can resume the work with full
  context — no re-explaining, no re-discovering. Use when the user says
  "handoff", "fai un handoff", "riprendi in un'altra chat", "passa il lavoro",
  "salva la sessione", "sto per finire il contesto", "continue later",
  "resume elsewhere", or when a long session is wrapping up or context is low.
---

# Handoff

Turn everything learned in this conversation into a compact, self-contained
document the next session can start from. The next chat receives ONLY this file,
so it must stand on its own.

## When to run
- The user asks for a handoff or to continue the work in another chat.
- Context is running low, or a long session is pausing / wrapping up.
- Switching to another agent or ending for the day.

## Steps
1. **Mine THIS conversation end to end** — you are the only one who has it.
   Extract: the goal, what's done, what was tried, what failed and *why*,
   decisions and the alternatives rejected, key files/paths, exact numbers and
   commands, and open questions. If you're skimming, stop and re-read: the
   details are the value.
2. **Grab repo state** if this is a git project:
   `git branch --show-current`, `git log --oneline -10`, `git status -s`,
   `git diff --stat`.
3. **Continue the chain.** If a `HANDOFF.md` already exists, read it first and
   note what changed since — don't start from zero.
4. **Write** the handoff to `HANDOFF.md` in the project root using the template
   below.
5. **Report** the file path and give the user a one-line paste prompt for the
   next chat.

## Output template (HANDOFF.md)
```
# Handoff — <short title> (<date>)

## Goal
<what we're ultimately trying to achieve, 1-3 lines>

## Current status
<where things stand right now>

## Done
- <completed items, with the files/functions touched>

## In progress / not yet done
- <remaining work — concrete next actions first>

## What worked
- <approaches that succeeded — reuse these>

## What did NOT work (and why)
- <failed approaches + the reason — the most valuable section; stops the next
  session re-discovering the same dead ends>

## Key decisions
- <decision → why, and the alternative that was rejected>

## Key files & commands
- `path/to/file` — <why it matters>
- `command` — <what it does>

## Open questions
- <anything unresolved or needing the user>

## Next step
<the single most important thing to do next>
```

## Rules
- **Compress hard, but never drop a failed approach or an exact number/command**
  — those are the expensive things to rediscover. Prose is cheap to cut;
  evidence is not.
- **Self-contained.** No "as discussed above" — the next chat has no history.
- **Don't invent progress.** If something is uncertain, say so.
- Preserve the user's language: write the handoff in the language of the session.

## Paste prompt
Finish by telling the user, verbatim, what to paste into the new chat:

> Read `HANDOFF.md` and continue from "Next step".
