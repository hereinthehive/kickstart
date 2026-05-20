# Kickstart

A Claude Code starter template. Fork it, run `/onboarding`, and Claude Code will configure itself for your project — interview you about what you're building, write a tailored `CLAUDE.md`, set sensible permissions, and hand off cleanly.

## Quick start

1. Fork or clone this repo
2. Open it in Claude Code
3. Run `/onboarding`

The first run takes around 5–10 minutes — most of that is a friendly discovery conversation about your project, how you work, and what you'd like Claude to help with most. Re-run any time to refresh.

After onboarding finishes, close and reopen Claude Code once so the new hooks take effect.

## What you can talk to me about

You don't need to memorize slash commands. Just say what you want in plain English. The slash commands in parens are shortcuts if you prefer.

| Phrase | Command | What it does |
|---|---|---|
| *"What can you do?"* | `/help-me` | Reminds you what commands you have |
| *"Remember to..."* | `/remember X` | Saves a preference across sessions |
| *"Forget about..."* | `/forget` | Removes a saved preference |
| *"Wrap up"* | `/wrap` | Recaps the session for next time |
| *"Update my setup"* | `/update` | Checks your setup against current best practices |
| *"Save a checkpoint"* (opt-in) | `/checkpoint` | Saves a recoverable snapshot before risky changes |
| *"Undo that"* (opt-in) | `/undo` | Reverses the last change |

The two opt-in commands appear only if you say yes to the safety net during onboarding (requires git).

## Memory across sessions

Kickstart writes a short recap at the end of each working session — automatically when you close cleanly, or by saying *"wrap up"*. The next time you open Claude Code, you'll see a brief *"Where we left off"* note so you can pick up without re-explaining context.

If you tell Claude *"remember to..."* or *"from now on, always..."*, the preference goes into `CLAUDE.md` and survives across sessions.

## Keeping the setup healthy

Run this once to enable weekly audits:

```
/loop weekly /update
```

`/update` runs two checks in parallel:

- A **curator** that researches current Claude Code best practices and writes them to `.claude/knowledge.md` (the baseline).
- A **caretaker** that audits your project against the baseline and against itself — looking for outdated patterns, stale preferences, and skills you never use.

You get a plain-language summary. Nothing changes without your approval.

## What's inside

- `.claude/skills/` — slash-command skills (the ones in the table above)
- `.claude/agents/` — the curator and caretaker subagents `/update` calls
- `.claude/skill-templates/` — the safety-net skills, staged until onboarding activates them
- `.claude/settings.json` — permissions and the SessionStart / Stop hooks that drive the recap pattern

You don't need to know any of this to use Kickstart — but it's all here if you want to peek under the hood.
