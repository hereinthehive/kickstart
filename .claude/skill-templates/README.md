# Skill templates

This directory holds skills that aren't active by default. Onboarding copies
them into `.claude/skills/` when the user opts in to the relevant feature
(e.g., the safety net).

Claude Code does not auto-discover skills in this directory, so unactivated
skills don't appear as commands.

To activate a skill manually, copy its folder to `.claude/skills/`:

`cp -r .claude/skill-templates/undo .claude/skills/`

Then restart your session so the new skill is picked up.
