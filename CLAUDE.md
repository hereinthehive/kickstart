# Kickstart

A Claude Code starter template with built-in onboarding, living memory, session bookends, an opt-in git safety net, and a unified `/update` audit.

## Getting Started

Run `/onboarding` to configure this project for your needs. It will:
- Learn about your project and how you like to work
- Detect git and offer an opt-in safety net for undoing changes
- Create a tailored CLAUDE.md and settings
- Wire up the curator (external best practices) and caretaker (internal integrity) subagents
- Hand off with clear instructions on what was created and what to do next

## Available Commands

After onboarding, you'll have these. You can use slash commands or just talk to Claude in natural language.

| Command | Natural-language phrase | Purpose |
|---|---|---|
| `/onboarding` | — | Set up or refresh Kickstart for this project |
| `/help-me` | *"What can you do?"* | Show available commands in plain language |
| `/remember X` | *"Remember to X"* | Save a preference across sessions |
| `/forget` | *"Forget about X"* | Remove a saved preference |
| `/wrap` | *"Wrap up"* / *"Recap this session"* | Recap the session for next time |
| `/update` | *"Update my setup"* | Audit external best practices + internal integrity |
| `/checkpoint` (opt-in) | *"Save a checkpoint"* | Snapshot before risky changes |
| `/undo` (opt-in) | *"Undo that"* | Reverse the last change |

## How it works

**Onboarding** is the orchestrator — it interviews you, reads your project, builds the structure, then asks the curator subagent to review everything before handing off.

**`/update`** is the recurring audit. It runs the curator (checks for outdated patterns against current Claude Code docs) and the caretaker (checks for preference rot, unused skills, broken hooks) in parallel and gives you a plain-language summary.

For automatic weekly audits, run this once:

`/loop weekly /update`

That's it — `/update` will run for you every week. Nothing changes without your approval.
