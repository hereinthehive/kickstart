# Claude Code best practices baseline

_Refreshed: 2026-05-23_

## What good looks like

### CLAUDE.md
- Include project purpose, stack, run/test/build commands, code conventions, architectural decisions. Exclude things Claude can infer from the files.
- Keep under 200 lines. Test each line: "Would removing this cause Claude to make mistakes?" If not, cut it.
- Use `@path/to/file` import syntax for supplementary context instead of inlining.
- Include `## Preferences` with HTML comment marker so `/remember`/`/forget` work.
- Include `## Constraints` for hard stops — off-limits files/areas, things Claude must never do.
- Project root + checked into git. Personal overrides → `~/.claude/CLAUDE.md`.

### settings.json
- `allow` — specific tools/commands the project genuinely needs. Precise `Bash(<cmd> *)` over blanket `Bash`.
- `deny` — highest precedence, cannot be overridden. Use for secrets, destructive ops, package installs.
- `ask` — commands to review before running.
- `defaultMode` — top-level only (`"default"` / `"auto"` / `"plan"`). Nested `permissions.defaultMode` is deprecated.
- Hook events: `SessionStart`, `Stop`, `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Notification`.
- `SessionStart` stdout injected as Claude context — keep fast.
- `Stop` fires on response completion — wrap signaling, notifications.

### Skills
- Location: `.claude/skills/<name>/SKILL.md` (project) or `~/.claude/skills/<name>/SKILL.md` (personal).
- Key fields: `description`, `disable-model-invocation`, `allowed-tools`.
- `disable-model-invocation: true` — manual-only trigger; removes from Claude's ambient context.
- `user-invocable: false` — Claude applies it automatically; not shown in `/` menu.
- Supporting files sit alongside SKILL.md; keep SKILL.md under 500 lines.

### Subagents
- Location: `.claude/agents/<name>.md` (project) or `~/.claude/agents/<name>.md` (personal).
- Key frontmatter: `name`, `description`, `tools`, `model`.
- `description` drives auto-delegation — use keywords matching natural user phrasing.
- `tools` restricts what the subagent can use.
- Use subagent over skill when: autonomous multi-step, floods context with intermediate results, called by other skills.

### Recurrence and scheduling
- `/loop weekly /update` — run once to schedule weekly audits.
