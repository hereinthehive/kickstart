# Kickstart

A Claude Code starter template with built-in onboarding, a knowledge curator, an internal caretaker, and a unified `/update` audit. Fork it, run `/onboarding`, and Claude Code will configure itself for your project.

## Quick start

1. Fork or clone this repo
2. Open it in Claude Code
3. Run `/onboarding`

Onboarding will ask you a few questions, read your project, set up `CLAUDE.md` and `settings.json`, and hand off cleanly. Once it's done, you'll have a small set of plain-language commands you can talk to.

## Built-in commands

After running `/onboarding`, you'll have these:

| Command | Plain-language phrase | What it does |
|---|---|---|
| `/help-me` | *"What can you do?"* | Reminds you what commands you have |
| `/remember X` | *"Remember to X"* | Saves a preference across sessions |
| `/forget` | *"Forget about X"* | Removes a saved preference |
| `/wrap` | *"Wrap up"* | Recaps the session for next time |
| `/update` | *"Update my setup"* | Audits your setup against best practices |
| `/undo` (opt-in) | *"Undo that"* | Reverses the last change |
| `/checkpoint` (opt-in) | *"Save a checkpoint"* | Saves a recoverable snapshot |

**You don't need to memorize slash commands** — just talk to Claude using the phrases above. Slash commands are the shortcut path; natural language is the discovery path.

## Keeping your setup current

Run this once to enable weekly audits:

```
/loop weekly /update
```

`/update` runs two checks in parallel — external best practices (curator) and internal project integrity (caretaker) — and gives you a plain-language summary. Nothing changes without your approval.

## How onboarding works

`/onboarding` interviews you, reads your project, writes a tailored `CLAUDE.md` and `.claude/settings.json`, optionally activates the git safety net, and hands off cleanly. The first run might take 2–3 minutes. Re-run it any time to refresh.
