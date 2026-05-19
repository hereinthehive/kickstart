---
title: Living memory, session bookends, and a git safety net for the starter template
date: 2026-05-20
status: design-approved
---

# Living memory, session bookends, and a git safety net

## Problem

The starter template's day-zero experience is solid: onboarding speaks plain language, asks multiple-choice questions, and hands off cleanly. Day one onward is the gap. A PM or designer who codes a bit will hit these failure modes between session 2 and session 20:

- Forgetting what they configured during onboarding.
- Telling Claude "from now on, always X" mid-session — and losing that decision on session close.
- Wanting to undo a change but finding `git reset` intimidating.
- Opening a fresh session a week later with no idea what was in flight.
- Calcifying preferences because settings files feel like a black box.

This spec addresses those failure modes through three coordinated additions: **living memory**, **session bookends**, and an opt-in **safety net**. It also moves the template to a portable file layout so project context works across AI tools, not just Claude Code.

## Persona

The primary user is a PM or designer who codes a bit. They run a terminal comfortably and have used git, but they bail the moment something looks like real engineering — a stack trace, a merge conflict, a YAML schema error. They won't open a `.md` file in a code editor to change preferences. They'll tell Claude in natural language.

## Architecture overview

Onboarding adds five new project-level skills organized around three concerns, extends the two existing skills (onboarding, knowledge-curator), and switches the file layout to a portable convention with AGENTS.md as the canonical context file.

| Skill | Concern | What it does |
|---|---|---|
| `/remember` | Living memory | Append a "from now on" line to AGENTS.md's Preferences section |
| `/forget` | Living memory | Remove a previously-remembered line, with confirmation |
| `/wrap` | Bookends | Write `.claude/last-session.md` recap; capture decisions made in the session |
| `/undo` | Safety net | Reverse the last assistant action via `git stash` / `git reset --soft` (opt-in) |
| `/checkpoint` | Safety net | Quiet commit-equivalent before risky operations (opt-in) |

No file removals or breaking changes. Existing onboarding and curator flows continue to work; the new skills are additive.

## File layout after onboarding

```
AGENTS.md                ← canonical project context (tool-agnostic)
CLAUDE.md                ← @AGENTS.md plus Claude-specific notes
GEMINI.md                ← one-line pointer to AGENTS.md
.gitignore               ← adds .claude/last-session.md if git opted in
.agent/
  skills/
    README.md            ← explains the convention; empty by default
.claude/
  settings.json          ← Claude-specific (hooks, permissions, schedule)
  last-session.md        ← runtime artifact, gitignored
  skills/
    onboarding/SKILL.md
    knowledge-curator/SKILL.md
    remember/SKILL.md
    forget/SKILL.md
    wrap/SKILL.md
    help-me/SKILL.md
    undo/SKILL.md         (if safety-net opted in)
    checkpoint/SKILL.md   (if safety-net opted in)
```

## Two-track skill model

Skills split into two tracks by invocation:

- **Slash-command skills** live in `.claude/skills/`. Claude Code invokes them by name (`/remember`, `/wrap`). The invocation contract binds them to Claude Code. All eight skills in this spec fall here.
- **Instructional skills** live in `.agent/skills/`. Claude reads them on demand because AGENTS.md points there. They're portable — Gemini CLI, Cursor, and Aider all read AGENTS.md and follow the same pointers. The directory starts empty but seeded with a README that establishes the convention.

The split is honest about portability. Slash commands stay Claude-specific; instructional skills genuinely work across tools.

## Living memory

### Anchor in AGENTS.md

Onboarding writes a Preferences section to AGENTS.md with an HTML comment marker so the memory skills can find it deterministically:

```markdown
## Preferences
<!-- managed by /remember and /forget — items added below are remembered across sessions -->
```

Items live as bulleted lines under the marker, written in the user's voice: "Always X", "Never Y", "Prefer Z".

### `/remember`

1. User says `/remember always run prettier before committing`.
2. Skill confirms in plain English: *"Got it. I'll always run prettier before committing. I've added this to your AGENTS.md."*
3. Appends `- Always run prettier before committing.` under the Preferences marker. Creates the section if it does not exist.

### `/forget`

1. With no argument, the skill lists current preferences with numbers and asks which to remove.
2. With an argument, it finds the best-matching line, shows it, and asks "Remove this? yes/no" before deleting.
3. Confirms in plain English after removal.

### Edge cases

- **Contradiction with existing preference.** `/remember never use prettier` when "Always run prettier" is already saved. Skill flags it: *"That contradicts an existing preference. Want me to replace the old one, or keep both?"*
- **Vague preferences.** `/remember be better at code` gets pushed back once: *"That's a bit broad — could you be specific about when or how?"* If the user insists, save verbatim.
- **Preferences vs. project facts.** Phrasing like "we deploy on Tuesdays" is project context, not a preference. Skill detects "we/our" phrasing and offers to put it in a project-facts section instead.

## Session bookends

### `/wrap`

User runs `/wrap` at the end of a session. The skill drafts a short recap from conversation context, shows it for approval before writing, and saves it to `.claude/last-session.md`, overwriting the previous one.

Recap format:

```markdown
# Last session — 2026-05-20

## What we did
- [2-4 bullets of substantive work]

## Decisions made
- [any "from now on" / "we chose X over Y" moments]

## In flight
- [anything unfinished, with enough context to resume]

## Next time, start with
- [one concrete next action]
```

`/wrap` also offers to call `/remember` on any "from now on" decisions it detected that were not already saved. One-shot prompt per item, yes/no.

### SessionStart hook extension

The existing SessionStart hook gets one more block:

```sh
if [ -f .claude/last-session.md ]; then
  echo ''
  echo '--- Where we left off ---'
  awk '/## Next time, start with/{flag=1; next} /^## /{flag=0} flag' .claude/last-session.md | head -2
fi
```

Only the "Next time, start with" line surfaces. The full recap is available to Claude as context, but the terminal stays quiet. If the recap is more than a day old, the hook prepends the age: *"Where we left off (3 days ago):"* — useful signal that context might be stale.

### File location

`.claude/last-session.md` is gitignored. The recap is personal and ephemeral. Committing it would cause endless merge conflicts on shared projects and would leak in-flight context to repo history.

### Edge cases

- **Fresh session, no recap.** Hook block is silent. No "no recap found" noise.
- **`/wrap` run mid-session.** Skill notices the recap covers very little ground and asks: *"You've only just started — are you sure?"*

## Safety net (opt-in, git-gated)

This section is the riskiest because it touches git. The design must be foolproof. One incident of `/undo` losing work would erase the trust gains the safety net is meant to produce.

### Detection and opt-in

During Phase 1 of onboarding, run:

```sh
git --version >/dev/null 2>&1 && git rev-parse --is-inside-work-tree >/dev/null 2>&1
```

Three states determine what happens:

1. **Git not installed.** Safety-net skills are never offered. Onboarding moves on silently.
2. **Git installed, not a repo.** Onboarding mentions it once during handoff: *"This project isn't tracked by git. If you ever want a safety net for undoing changes, just say so and I can set that up."*
3. **Git installed, repo.** Onboarding offers the safety net during Phase 2 via `AskUserQuestion`:

> *"Want a safety net for taking back changes? I can add two simple commands: `/checkpoint` (save a snapshot you can come back to) and `/undo` (reverse the last thing I did). Both use git behind the scenes but you don't need to know git to use them."*
>
> - **Yes, add them** — recommended for most projects
> - **Not now** — you can ask me to add them later
> - **No, skip it** — I don't want git-based undo in this project

The decision is recorded in `.claude/onboarding-log.md`:
- *Yes:* skills created, settings.json gets safe-git allowlist additions.
- *Not now:* no skills; marker says re-ask on next refresh.
- *No:* no skills; marker says do not re-prompt.

A user can opt in later by saying *"Add the undo safety net."* Claude reads this as intent and creates the two skills without requiring `/onboarding` to re-run.

### `/checkpoint`

Near-silent commit-and-stash. Used either manually or as a building block by `/undo`.

1. Runs `git add -A && git stash push -u -m "checkpoint: <timestamp> <short-desc>"`.
2. Confirms in plain English: *"Saved a checkpoint. If you want to come back to this exact state, I can restore it."*
3. Records the stash ref in `.claude/checkpoints.log` (gitignored) with an inferred one-line description.

Stash beats a `wip/` branch because branches require the user to think about branch state. Stash entries list and pop without leaving the current branch.

### `/undo` — Mode 1: uncommitted changes (most common)

1. Claude has just edited files. User says `/undo`.
2. Skill runs `git status --porcelain` to inventory dirty files.
3. Shows the user in plain English: *"I'm about to undo these 3 file changes. Nothing will be deleted permanently — I'll save them as a checkpoint first so you can restore if needed. OK?"*
4. On yes: `git stash push -u -m "undo: <timestamp>"`.
5. Confirms: *"Done. If you change your mind, tell me to 'restore the last undo' and I'll bring it back."*

### `/undo` — Mode 2: committed in this session

1. User says `/undo` after a commit has landed.
2. Skill checks `git log --since="<session start>"` to find session commits. Session start is the timestamp the SessionStart hook fires; the hook writes it to `.claude/.session-start` (gitignored) on each session open, and `/undo` reads from there.
3. Shows them, asks which to undo: *"I made these commits in this session. Which do you want to undo? (Your work is safe — I'll keep it staged so you can re-do it differently.)"*
4. On confirm: `git reset --soft <commit>^` — keeps changes staged, drops the commit.
5. Refuses commits older than the session start: *"That commit is from before we started today. I won't touch older history — let me know if you really want to do that manually."*

### Hard refusals

`/undo` will not:
- Touch pushed commits. Checks `git log @{upstream}..HEAD` first.
- Run `git reset --hard` ever.
- Operate when the repo has unresolved merge conflicts. Bails out: *"Something's mid-merge here — that's beyond what I'll touch automatically. Want me to walk you through it?"*
- Delete untracked files. The `-u` in `git stash push` captures them; nothing else needs deletion.

### Restoration

Phrases like *"Restore the last undo"* or *"Bring back what we undid"* run `git stash pop`. `/help-me` lists the restoration phrase so users don't have to memorize it.

### Settings.json pre-approval

If the user opts in, onboarding adds these to the allow list so the skills don't trigger permission prompts:

- `Bash(git stash *)`
- `Bash(git status *)`
- `Bash(git log *)`
- `Bash(git reset --soft *)`

Destructive verbs stay un-approved. `git reset --hard`, `git push --force`, and `git clean` still hit a permission prompt. Defense in depth.

### Edge cases

- **Clean working tree, no recent commits.** `/undo` says *"There's nothing to undo right now."* Does not error.
- **`/undo` twice in a row.** Second run notices the previous stash and asks if the user wants to undo further back.
- **`.claude/checkpoints.log` grows unbounded.** Rotate at 50 entries — older ones remain recoverable via `git stash list`.

## Onboarding integration

### Phase 1 (read everything first)

Add one detection step: run `git --version` and `git rev-parse --is-inside-work-tree` to determine the three git states.

### Phase 2 (discovery interview)

Add the safety-net opt-in question, conditionally:

- State 3 (git repo): ask via `AskUserQuestion`.
- State 2 (git installed, not a repo): include the one-line mention in handoff.
- State 1 (no git): say nothing.

### Phase 4 (build the setup)

**Always create:**

- AGENTS.md (full content from interview answers)
- CLAUDE.md (shim: `@AGENTS.md` + Claude-specific section)
- GEMINI.md (one-line pointer)
- `.agent/skills/README.md`
- `.claude/skills/remember/SKILL.md`
- `.claude/skills/forget/SKILL.md`
- `.claude/skills/wrap/SKILL.md`
- `.claude/skills/help-me/SKILL.md`

**Migration if a real CLAUDE.md already exists:**

Preview the migration before doing it. On approval, contents become AGENTS.md, CLAUDE.md gets replaced with the shim. Onboarding-log notes the migration.

**Conditionally create (safety-net opt-in only):**

- `.claude/skills/undo/SKILL.md`
- `.claude/skills/checkpoint/SKILL.md`
- Safe-git allowlist additions in settings.json.

**SessionStart hook updates:**

- Change `[ -f CLAUDE.md ]` check to `[ -f AGENTS.md ]`.
- Add the "Where we left off" block from the bookends section.
- If the safety net is opted in, write the current timestamp to `.claude/.session-start` so `/undo` can scope its `git log` query to this session.

**Gitignore updates (if git opted in):**

- Add `.claude/last-session.md`.
- Add `.claude/checkpoints.log` and `.claude/.session-start` if safety net opted in.
- Create `.gitignore` if missing.

### AGENTS.md template

Onboarding writes:

- Project overview, stack, goals (from interview).
- A `## Skills` section explaining the two-track convention.
- A `## Preferences` section with the HTML marker.
- A `## Constraints` section (from interview).
- A `## Commands` quick-reference of available slash commands in plain language.

### CLAUDE.md shim

```
@AGENTS.md

## Claude Code
Skills live in `.claude/skills/`. Settings in `.claude/settings.json`.
```

### GEMINI.md shim

```
# Gemini CLI
All project context lives in AGENTS.md at the project root. Read it before any task.
```

### `.agent/skills/README.md` seed

```
# Portable instructional skills

This directory holds instructional skills that are tool-agnostic.
Each skill is a folder containing a SKILL.md with minimal frontmatter:

---
name: skill-name
description: One-line summary of what this skill is for and when to use it.
---

These are referenced from AGENTS.md and read on demand by whichever
AI tool is in use (Claude Code, Gemini, Cursor, etc.).

Slash-command skills for Claude Code live in `.claude/skills/`.
```

### Phase 7 (handoff)

The "Commands available" list is written in the user's voice:

- `/help-me` — Remind me what commands I have and when to use them
- `/remember <thing>` — Save a preference for future sessions
- `/forget` — Remove a saved preference
- `/wrap` — Recap this session so I can pick up next time
- *(if safety net opted in)* `/checkpoint` — Save a snapshot before doing something risky
- *(if safety net opted in)* `/undo` — Take back the last thing I did

## Knowledge curator updates

The curator's review pass should check:

- AGENTS.md exists and is the source of truth; CLAUDE.md is a thin shim.
- `.agent/skills/` exists with its README.
- Preferences section in AGENTS.md is not rotting (no contradictions, no vague entries piling up).
- `.claude/last-session.md` is gitignored if it exists.
- Safety-net skills, if present, still have their git allowlist intact.

Findings stay gentle. The "In plain words" section remains friendly.

## Testing scenarios

| Scenario | What to verify |
|---|---|
| Fresh fork, no existing context files | `/onboarding` produces AGENTS.md + CLAUDE.md shim + GEMINI.md + `.agent/skills/README.md` + all 6 base skills |
| Fork with an existing real CLAUDE.md | Migration preview shown; on approval, content moves to AGENTS.md; CLAUDE.md becomes shim; nothing lost |
| Git installed + git repo + safety-net "yes" | `/undo` and `/checkpoint` skills created; settings.json allowlist extended; `.claude/last-session.md` and `.claude/checkpoints.log` in `.gitignore` |
| Git installed + git repo + safety-net "not now" | No safety-net skills; onboarding-log marker present; next `/onboarding` re-asks |
| Git installed + git repo + safety-net "no" | No safety-net skills; marker says do not re-prompt |
| Git installed, not a repo | One-line mention in handoff; no safety-net opt-in offered |
| No git installed | Silent — no mention, no offer |
| `/remember always X`, then `/remember never X` | Contradiction detector flags it; user chooses keep / replace |
| `/wrap` on a fresh session with no substantive work | Skill notices and asks "are you sure?" |
| `/undo` on clean working tree | "Nothing to undo right now" — no error |
| `/undo` after push to remote | Refuses; explains why; suggests manual path |
| Second session after `/wrap` | SessionStart hook surfaces "Next time, start with..." line |

After the scenario walk, run `/knowledge-curator` against the starter repo itself. The "In plain words" verdict should report a healthy setup. Any gaps the curator finds are real bugs to fix.

## Rollout sequence

1. Generate implementation plan via `superpowers:writing-plans`.
2. Build all skills + onboarding/curator edits + AGENTS.md / CLAUDE.md / GEMINI.md scaffolding.
3. Walk the scenario matrix manually in a scratch fork. Fix as we go.
4. Run `/knowledge-curator` on the starter repo.
5. Update README.md to mention the new commands and the AGENTS.md convention.
6. Commit to main.

## Risks

- **AGENTS.md migration on existing forks.** Users who customized CLAUDE.md will run a re-onboarding that migrates it. The migration must preview before acting and accept cancellation.
- **`/undo` trust regression.** The skill must never destroy work. Every stash gets a recoverable name, and the skill states recoverability in plain English. One bad incident erases all trust gains.
- **`.agent/` convention adoption.** The convention is de-facto, not formal. If a competing standard emerges, we update. Low risk — the pattern is gaining traction, and we cite agentic-design-system as the source.

## Out of scope (YAGNI)

- Automated test suite for the skills. Cost exceeds value at template stage; manual scenario walks are honest about what we can verify.
- Telemetry on safety-net opt-in rate. Future signal once the template has real users.
- A skill-converter between `.claude/skills/` and `.agent/skills/` formats. Useful eventually; not now.
- Tool-specific configs beyond GEMINI.md (no `.cursorrules`, `.aider.conf.yml`). One pointer file is enough.
- A `/preferences` listing skill. AGENTS.md is readable; `/help-me` can summarize.
- Auto-running `/wrap` on Stop hook. Tempting but fragile — user might `/clear` and not want a recap.
- Multi-step undo with a redo stack. `git stash` provides this for free.
- Cross-session `/undo`. If the user wants to undo yesterday's work, the answer is "open yesterday's session" or do it manually.
