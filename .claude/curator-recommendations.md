# Curator recommendations

_Generated: 2026-05-23_

## In plain words

The Kickstart template is well-structured and aligns with current Claude Code best practices. Onboarding has now layered screenplay-specific content on top of it. The three custom skills (track, review, coverage) are in place. A few small structural items were addressed: the deprecated nested `permissions.defaultMode` was removed, `.gitignore` was created, and the developer-only deny rules were trimmed for a pure writing project.

## What's healthy

- `settings.json` has well-structured `allow`, `ask`, and `deny` rules.
- Sensitive paths (`.env`, `secrets/`) are denied.
- Destructive commands (`rm`, `git push`, `git clean`) are in `deny`.
- `SessionStart` hook injects git context and session recap.
- `Stop` hook tracks substantive sessions and flags them for wrap.
- Both subagents (`curator`, `caretaker`) are present with correct frontmatter.
- All always-active skills are present (`help-me`, `remember`, `forget`, `update`, `wrap`).
- Safety net (`checkpoint`, `undo`) is active.
- `## Preferences` and `## Constraints` sections are present in CLAUDE.md.
- `.gitignore` covers session state files.

## Applied during onboarding

- Replaced template CLAUDE.md with screenplay-specific content.
- Built three screenplay skills: `/track`, `/review`, `/coverage`.
- Activated safety net: `/checkpoint` and `/undo`.
- Created `.gitignore` with session state entries.
- Removed deprecated nested `permissions.defaultMode: "ask"` from `settings.json`.
- Trimmed developer-only deny rules (package managers) that added noise for a writing project.
- Wrote user-level `~/.claude/CLAUDE.md` with communication style preferences.

## Things to watch

- As the screenplay grows, consider adding `@characters.md` or `@structure.md` references in CLAUDE.md so Claude loads them automatically each session.
- If you rename or reorganise your scene files, update CLAUDE.md to reflect the new layout.
- Run `/update` occasionally to keep skills and settings current.
