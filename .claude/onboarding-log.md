# Onboarding log

## 2026-05-23 — Initial setup

### What was done

- Replaced template CLAUDE.md with screenplay-specific content
- Created `~/.claude/CLAUDE.md` with user communication preferences
- Built `.claude/skills/track/SKILL.md` — character tracker
- Built `.claude/skills/review/SKILL.md` — screenplay review notes
- Built `.claude/skills/coverage/SKILL.md` — formal studio coverage
- Activated safety net: copied checkpoint + undo from skill-templates
- Created `.gitignore` with session state entries
- Updated `.claude/settings.json`: removed deprecated nested `permissions.defaultMode`, trimmed developer-only deny rules
- Wrote `.claude/knowledge.md` and `.claude/curator-recommendations.md`

### Project context captured

- Type: Feature screenplay
- Format: Markdown files
- Stage: Structure and character descriptions complete; moving into scene drafting
- Stack: None (pure writing project)
- Team: Solo

### Decisions on optional features

- Safety net (/checkpoint, /undo): **yes**

### Questions answered (skip on re-run)

- User is a writer, not a developer — use plain language
- Communication style: brief overall, but explain what you're doing and why
- Workflow: write scenes, then track/review; occasional studio coverage
- Versioning: yes, wants to track changes via git
- Constraints: never rewrite dialogue or alter structure/character files without being asked
