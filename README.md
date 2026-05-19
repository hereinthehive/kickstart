# Starter

A Claude Code starter template with built-in onboarding and knowledge curation. Fork it, run `/onboarding`, and Claude Code will configure itself for your project.

## Quick start

1. Fork or clone this repo
2. Open it in Claude Code
3. Run `/onboarding`

That's it. Onboarding will ask you a few questions, read your project, set up `CLAUDE.md` and `settings.json`, then have the Knowledge Curator check everything against current best practices before handing off.

## Built-in skills

| Skill | What it does |
|---|---|
| `/onboarding` | Interviews you, builds the setup, runs a best-practice review, hands off clearly |
| `/knowledge-curator` | Audits the current setup against the latest Claude Code docs — runs weekly automatically (web) |

## How the weekly review works

On claude.ai, `/knowledge-curator` runs automatically each week and surfaces anything outdated or missing. In the CLI or IDE, run it manually or use `/loop weekly /knowledge-curator` to get the same cadence.

## What gets created

After onboarding:
- `CLAUDE.md` — project context, conventions, and your working preferences
- `.claude/settings.json` — permissions, hooks, and schedule tailored to your stack
- `.claude/skills/` — any project-specific slash commands that fit your workflow
