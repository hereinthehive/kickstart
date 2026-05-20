---
title: Living memory, session bookends, a git safety net, and a unified update flow for the starter template
date: 2026-05-20
status: design-approved (revised)
---

# Living memory, session bookends, a git safety net, and a unified update flow

## Problem

The starter template's day-zero experience is solid: onboarding speaks plain language, asks multiple-choice questions, and hands off cleanly. Day one onward is the gap. A PM or designer who codes a bit will hit these failure modes between session 2 and session 20:

- Forgetting what they configured during onboarding.
- Telling Claude "from now on, always X" mid-session — and losing that decision on session close.
- Wanting to undo a change but finding `git reset` intimidating.
- Opening a fresh session a week later with no idea what was in flight.
- Calcifying preferences because settings files feel like a black box.
- Accumulating clutter — skills they never use, preferences gone stale — with no way to notice.

This spec addresses those failure modes through four coordinated additions: **living memory**, **session bookends**, an opt-in **safety net**, and an **`/update` skill** that audits the project from both outside (best practices) and inside (integrity) in a single call.

## Persona

The primary user is a PM or designer who codes a bit. They run a terminal comfortably and have used git, but they bail the moment something looks like real engineering. They won't open a `.md` file to change preferences. They'll tell Claude in natural language. Slash commands are the shortcut path; natural language is the discovery path.

## Architecture overview

Six new project-level skills plus two subagents get added. Onboarding gets extended. The existing top-level `/knowledge-curator` skill is retired in favor of a unified `/update` flow.

| Skill | Concern | What it does |
|---|---|---|
| `/remember` | Living memory | Append a "from now on" line to CLAUDE.md's Preferences section |
| `/forget` | Living memory | Remove a previously-remembered line, with confirmation |
| `/wrap` | Bookends | Write `.claude/last-session.md` recap; capture decisions made in session |
| `/undo` | Safety net | Reverse the last assistant action via `git stash` / `git reset --soft` (opt-in) |
| `/checkpoint` | Safety net | Quiet stash before risky operations (opt-in) |
| `/update` | Integrity | Run external (curator) + internal (caretaker) audits, merge findings, present in plain words |

Two subagents under `.claude/agents/`:
- `curator.md` — external audit (current `/knowledge-curator` logic, repurposed as subagent)
- `caretaker.md` — internal audit (preference rot, adoption signal, shim integrity)

The existing `/knowledge-curator` skill is retired during the next onboarding refresh and replaced by `/update`. Existing `schedule` block in settings.json is removed at the same time. Users are told about the change in plain language during the refresh handoff.

## Natural-language triggers as the primary affordance

CLAUDE.md gets a `## Talking to me` section that lists trigger phrases:

```markdown
## Talking to me

You don't need to remember slash commands. Just talk to me:

- *"Remember to..."* / *"From now on, always..."* → saves a preference (same as /remember)
- *"Forget about..."* / *"Stop remembering..."* → removes a preference (same as /forget)
- *"Recap this session"* / *"Wrap up"* → writes a session recap (same as /wrap)
- *"Undo that"* / *"Take that back"* → reverses the last change (same as /undo, if enabled)
- *"Save a checkpoint"* / *"Snapshot this"* → creates a recovery point (same as /checkpoint, if enabled)
- *"What can you do?"* / *"What commands do I have?"* → shows your command list (same as /help-me)
- *"Update my setup"* / *"Check my setup"* / *"Audit me"* → audits the project (same as /update)
```

CLAUDE.md is loaded into every session, so Claude reads these triggers as instructions and routes intent without slash-command syntax. Slash commands still work for users who prefer them.

## Living memory

### Anchor in CLAUDE.md

Onboarding writes a Preferences section with an HTML comment marker so the memory skills find it deterministically:

```markdown
## Preferences
<!-- managed by /remember and /forget — items added below are remembered across sessions -->
```

Items live as bulleted lines, written in the user's voice: "Always X", "Never Y", "Prefer Z".

### `/remember` (and the "remember to..." phrase)

1. User says `/remember always run prettier before committing` or *"remember to always run prettier before committing"*.
2. Skill confirms in plain English: *"Got it. I'll always run prettier before committing. I've added this to your CLAUDE.md."*
3. Appends `- Always run prettier before committing.` under the Preferences marker. Creates the section if absent.

**Personal vs. project disambiguation:** If the preference uses first-person singular phrasing (*"I prefer..."*, *"my..."*), the skill asks once: *"Is this just for you, or should everyone working on this project follow it?"* On "just me," the line goes to `~/.claude/CLAUDE.md` under the project-specific heading. Addresses the leak risk in team-shared files.

### `/forget` (and the "forget about..." phrase)

1. With no argument, the skill lists current preferences with numbers and asks which to remove.
2. With an argument, it finds the best-matching line, shows it, and asks "Remove this? yes/no" before deleting.
3. Confirms in plain English after removal.

### Edge cases

- **Direct contradiction.** `/remember never use prettier` when "Always run prettier" exists. Skill flags it.
- **Subtle drift.** Conditional refinements aren't caught here — they're the caretaker's job, surfaced via `/update`.
- **Vague preferences.** `/remember be better at code` gets pushed back once. If user insists, save verbatim.

### Kill criterion

If the Preferences section has fewer than two user-added entries after 30 days, `/update` recommends removing `/remember` and `/forget` from the template.

## Session bookends

### `/wrap` (and the "wrap up" phrase)

User runs `/wrap` at session end. The skill drafts a short recap from conversation context, **shows it for approval** before writing, and saves to `.claude/last-session.md`, overwriting the previous one. Format:

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

`/wrap` also offers to call `/remember` on any "from now on" decisions detected during the session that weren't already saved. One-shot prompt per item.

### Stop hook (the discipline fix)

User-initiated `/wrap` is fragile. The Stop hook makes it automatic:

- Fires on session end.
- Checks whether the session had at least five substantive assistant turns.
- If yes, runs `/wrap` non-interactively, drafting the recap and saving it.
- A short notification surfaces in the next session: *"I wrapped up last session for you. You can review or edit `.claude/last-session.md` anytime."*

**Honest about limits:** force-quits, hard `/clear`, and system crashes bypass the hook.

### SessionStart hook extension

The existing hook gets one more block:

```sh
if [ -f .claude/last-session.md ]; then
  echo ''
  echo '--- Where we left off ---'
  awk '/## Next time, start with/{flag=1; next} /^## /{flag=0} flag' .claude/last-session.md | head -2
fi
```

If the recap is more than a day old, prepend the age: *"Where we left off (3 days ago):"*.

### File location

`.claude/last-session.md` is gitignored. Personal and ephemeral. Multi-machine work is single-machine by design — acknowledged limitation, not solved here.

### Kill criterion

If `.claude/last-session.md` is empty or unmodified for 4 consecutive weeks, `/update` recommends removing `/wrap` and the Stop hook.

## Safety net (opt-in, git-gated)

Riskiest section because it touches git. The design must be foolproof — one incident of `/undo` losing work erases all trust gains.

### Detection and opt-in

During Phase 1 of onboarding:

```sh
git --version >/dev/null 2>&1 && git rev-parse --is-inside-work-tree >/dev/null 2>&1
```

Three states:

1. **Git not installed.** Skills never offered. Onboarding silent.
2. **Git installed, not a repo.** One-line mention in handoff.
3. **Git installed, repo.** Onboarding offers via `AskUserQuestion`:

> *"Want a safety net for taking back changes? I can add two simple commands: `/checkpoint` (save a snapshot you can come back to) and `/undo` (reverse the last thing I did). Both use git behind the scenes but you don't need to know git to use them."*
>
> - **Yes, add them** — recommended for most projects
> - **Not now** — you can ask me to add them later
> - **No, skip it** — I don't want git-based undo in this project

Decision recorded in `.claude/onboarding-log.md`. "Not now" re-asks on refresh; "no" doesn't. User can opt in later via natural language.

### `/checkpoint`

Stash-based snapshot. Used manually or as a building block by `/undo`.

1. Runs `git add -A && git stash push -u -m "checkpoint: <timestamp> <short-desc>"`.
2. Confirms: *"Saved a checkpoint. If you want to come back to this exact state, I can restore it."*
3. Records the stash ref in `.claude/checkpoints.log` (gitignored).

Stash beats a `wip/` branch — branches require thinking about branch state; stashes don't.

### `/undo` — Mode 1: uncommitted changes (most common)

1. User says `/undo` or *"undo that."*
2. Skill runs `git status --porcelain` to inventory dirty files.
3. Shows the user in plain English what's about to happen.
4. On yes: `git stash push -u -m "undo: <timestamp>"`.
5. Confirms: *"Done. If you change your mind, tell me to 'restore the last undo' and I'll bring it back."*

### `/undo` — Mode 2: committed in this session

1. User says `/undo` after a commit has landed.
2. Skill reads session-start timestamp from `.claude/.session-start` (written by SessionStart hook).
3. Runs `git log --since="<session-start-timestamp>"` to find session commits.
4. **Pre-flight:** if working tree has uncommitted changes on top, the skill stashes them first as a separate, named stash. Confirms: *"You have unsaved changes too. I'll set those aside first, then undo the commit. Both are recoverable."*
5. Shows session commits, asks which to undo.
6. On confirm: `git reset --soft <commit>^`.
7. Refuses commits older than session start: *"That commit is from before we started today. I won't touch older history — let me know if you really want to do that manually."*

### Multi-window session-start caveat

`.claude/.session-start` is per-CLI-process. The skill is honest on first encounter: *"I only track work from this Claude Code window's session. Commits from other windows are out of scope for `/undo`."*

### Hard refusals

`/undo` will not:
- Touch pushed commits. Checks `git log @{upstream}..HEAD` first.
- Run `git reset --hard` ever.
- Operate on unresolved merge conflicts. Bails with manual guidance.
- Delete untracked files. The `-u` in `git stash push` captures them.

### Restoration

Phrases like *"Restore the last undo"* run `git stash pop`. `/help-me` lists the phrase.

### Settings.json pre-approval (if opted in)

- `Bash(git stash *)`, `Bash(git status *)`, `Bash(git log *)`, `Bash(git reset --soft *)` — allowed.
- `git reset --hard`, `git push --force`, `git clean` — still prompt. Defense in depth.

### Kill criteria

Reported by `/update`:
- `/checkpoint`: if `.claude/checkpoints.log` is empty or unchanged for 30 days.
- `/undo`: if no `undo:` stash entries appear in 30 days.

## `/update` — unified audit

One user-facing skill that runs both audits, merges findings, and presents them as a single report. Replaces the previous standalone `/knowledge-curator` as the top-level command.

### Why one command, not two

Two weekly maintenance commands is one too many for this persona. The user thinks *"audit my setup"* — they don't think *"first audit against external best practices, then audit against internal state."* `/update` is the latch point; the split into curator and caretaker is an implementation detail.

### Shape

- **`/update` skill** at `.claude/skills/update/SKILL.md` — top-level entry point. Invokes both subagents in parallel via the Agent tool, merges their reports, presents to the user.
- **Curator subagent** at `.claude/agents/curator.md` — does what the previous `/knowledge-curator` skill did: fetch current docs, review setup against best practices.
- **Caretaker subagent** at `.claude/agents/caretaker.md` — internal integrity audit (below).
- **Recurrence via `/loop`**, not a settings.json schedule block. Users run `/loop weekly /update` to enable automatic weekly audits. The starter template removes the previous `schedule` block (which pointed at the old `/knowledge-curator`). Reasons: `/loop` works in both CLI/IDE and web; it's user-visible and user-controllable; it avoids a hidden cron-like layer that non-technical users can't inspect. The SessionStart hook continues to suggest `/loop weekly /update` if it notices the user hasn't enabled it.

### Caretaker subagent — what it checks

**Preference rot in CLAUDE.md:**
- Stale entries — references to files, scripts, or tools that no longer exist in the project.
- Subtle drift — preferences that look like conditional refinements of older ones (not direct contradictions). Flags the cluster, asks the user to reconcile.
- Personal-sounding preferences still in CLAUDE.md. Suggests moving to `~/.claude/CLAUDE.md`.
- Vague preferences with no clear referent. Suggests removal or rewrite.

**Adoption signal (kill criteria enforcement):**
- `.claude/last-session.md` modified date → if empty or stale 4 weeks running → recommend removing `/wrap` + Stop hook.
- `.claude/checkpoints.log` size and modification → 30-day staleness → recommend removing `/checkpoint`.
- Stash list filtered by `undo:` prefix → 30-day staleness → recommend removing `/undo`.
- Preferences section entry count → if fewer than two after 30 days → recommend removing `/remember` + `/forget`.

**Shim and config integrity:**
- `.gitignore` contains expected entries.
- `.claude/.session-start` exists and isn't ancient (if safety net opted in).
- Preferences section's HTML comment marker is intact.
- Onboarding-log is internally consistent.

### Merged output

`/update` writes a combined report to `.claude/update-report.md` (gitignored), structured:

```markdown
# Setup audit — [date]

## In plain words
[Friendly 2-3 sentence summary covering both audits — e.g., "Your setup is healthy.
I found 2 small things you could improve and noticed you haven't been using
/checkpoint — want me to remove it?"]

## What's healthy
- [merged bullets from both audits]

## What needs attention
- [merged bullets with severity, sourced from both]

## Recommended removals
- [kill-criteria findings from caretaker]
- [deprecated patterns from curator]
```

`/update` presents the plain-words summary and offers to apply quick wins / removals.

### Hard rule: never auto-prune

`/update` recommends, never removes. Removing skills, pruning preferences, or rotating logs all require explicit user approval — even on scheduled runs. False positives are catastrophic.

### Kill criterion for `/update` itself

If `/update` hasn't produced actionable findings across 12 consecutive weekly runs, the next run suggests reducing cadence — *"Things have been quiet for 3 months. Want to switch to `/loop monthly /update`?"* — with the exact command to swap. The skill stays available for manual invocation regardless.

## Onboarding integration

### Phase 1 (read everything first)

Add: run `git --version` and `git rev-parse --is-inside-work-tree` to determine the three git states.

### Phase 2 (discovery interview)

Add the safety-net opt-in question, conditional on state 3.

### Phase 3 (consult curator on methodology)

The onboarding skill currently invokes the `/knowledge-curator` skill here. After this change, it invokes the **curator subagent** directly via the Agent tool. The methodology consultation is unchanged in substance; only the invocation path changes.

### Phase 4 (build the setup)

**Always create:**
- `.claude/skills/remember/SKILL.md`
- `.claude/skills/forget/SKILL.md`
- `.claude/skills/wrap/SKILL.md`
- `.claude/skills/help-me/SKILL.md` (already exists)
- `.claude/skills/update/SKILL.md`
- `.claude/agents/curator.md` (subagent — repurposed from old skill)
- `.claude/agents/caretaker.md` (subagent)

**Remove (during onboarding refresh on existing setups):**
- `.claude/skills/knowledge-curator/SKILL.md` — replaced by `/update` + curator subagent. Onboarding-log notes the retirement.

**CLAUDE.md additions:**
- `## Talking to me` section with natural-language trigger list.
- `## Preferences` section with HTML marker.
- Brief mention of `/update` in plain language.

**Conditionally create (safety-net opt-in only):**
- `.claude/skills/undo/SKILL.md`
- `.claude/skills/checkpoint/SKILL.md`
- Safe-git allowlist additions in settings.json.

**Hook changes:**
- SessionStart hook: add the "Where we left off" block. If safety-net opted in, write timestamp to `.claude/.session-start`.
- Stop hook: new — triggers `/wrap` non-interactively after sessions with ≥5 substantive turns.

**Recurrence:**
- Remove the existing `schedule` block from settings.json (it pointed at `/knowledge-curator`).
- In Phase 7 handoff, suggest the user run `/loop weekly /update` if they want automatic weekly audits. Explain it in plain language and show the exact command.
- SessionStart hook: on every session start, if no active `/loop` for `/update` is detected, surface a one-line tip: *"Tip: run `/loop weekly /update` to keep your setup current automatically."* Quiet once enabled.

**Gitignore updates (if git opted in):**
- Add `.claude/last-session.md`, `.claude/update-report.md`.
- If safety-net opted in: also `.claude/checkpoints.log`, `.claude/.session-start`.
- Create `.gitignore` if missing.

### Phase 7 (handoff)

Commands list, in the user's voice with natural-language phrasing:

- *"What can you do?"* or `/help-me` — Remind me what commands I have.
- *"Remember to X"* or `/remember X` — Save a preference.
- *"Forget about X"* or `/forget` — Remove a saved preference.
- *"Recap this session"* or `/wrap` — Write a session recap (or I'll do it automatically when you close cleanly).
- *"Update my setup"* or `/update` — Audit project health. Run `/loop weekly /update` once if you want it to happen automatically every week.
- *(if safety net opted in) "Save a checkpoint"* or `/checkpoint` — Snapshot before risky operations.
- *(if safety net opted in) "Undo that"* or `/undo` — Take back the last thing I did.

## Testing scenarios

| Scenario | What to verify |
|---|---|
| Fresh fork | `/onboarding` produces CLAUDE.md (with Preferences + Talking-to-me sections), 5 base skills, 2 subagents, no `schedule` block, and a Phase 7 prompt suggesting `/loop weekly /update` |
| Fresh fork after running `/loop weekly /update` | SessionStart hook stops surfacing the "Tip" line |
| Existing fork with old `/knowledge-curator` skill | Refresh removes the old skill; `/update` takes its place; user is informed in plain language |
| Git installed + git repo + safety-net "yes" | `/undo`, `/checkpoint` skills created; settings.json allowlist extended; gitignore covers `.claude/.session-start` and `.claude/checkpoints.log` |
| Git installed + git repo + safety-net "not now" | No safety-net skills; onboarding-log marker present |
| No git | Silent — no mention, no offer |
| `/update` invocation | Both subagents run in parallel; merged report produced; "In plain words" surfaces first |
| User says *"remember to always X"* | Skill triggers via natural language |
| `/remember` direct contradiction | Flagged |
| `/remember "I prefer dark mode"` | Disambiguation asks "just you or everyone?" |
| `/wrap` on a fresh session | Skill asks "are you sure?" |
| Session with ≥5 substantive turns closed cleanly | Stop hook fires; recap written automatically |
| Session with <5 turns closed | Stop hook does nothing |
| `/undo` mode 2 with uncommitted changes on top | Stashes uncommitted work first; preserves both |
| `/undo` after push | Refuses with explanation |
| `/update` after 30 days of `/checkpoint` non-use | Recommends removal; user must approve |

After the scenario walk, run `/update` on the starter repo. Combined report should report healthy.

## Rollout sequence

1. Generate implementation plan via `superpowers:writing-plans`.
2. Build all skills + subagents + onboarding edits + hook additions.
3. Migrate existing `/knowledge-curator` skill content into the curator subagent. Retire the old skill file.
4. Walk the scenario matrix in a scratch fork.
5. Run `/update` on the starter repo.
6. Update README.md to mention the new commands and the natural-language affordance.
7. Commit to main.

## Risks

- **Stop hook reliability.** Force-quits and `/clear` bypass it. Acknowledged.
- **Kill-criteria false positives.** A user might genuinely use `/undo` once a quarter. Mitigation: `/update` recommends, never removes; thresholds conservative (30 days minimum).
- **Natural-language triggers ambiguity.** *"Remember to call mom"* might be misread as a project preference. Mitigation: skill confirms intent for phrases without engineering/workflow vocabulary.
- **`/undo` trust regression.** Skill must never destroy work. Every stash gets a recoverable name. One bad incident erases all trust gains.
- **`/knowledge-curator` retirement during refresh.** Users who scripted around `/knowledge-curator` (unlikely but possible) will need to switch. Onboarding-log records the migration.

## Out of scope (YAGNI)

- AGENTS.md / portable layout. Persona uses Claude Code; not designing for tool-switching today.
- Automated test suite. Manual scenario walks are honest about what we can verify at template stage.
- Cross-window or cross-machine session-start coordination.
- Auto-pruning by `/update`. Never. Always user-approved.
- A `/preferences` listing skill. CLAUDE.md is readable; `/help-me` summarizes.
- Multi-step undo with a redo stack. `git stash` provides this for free.
- Telemetry beyond what the caretaker reads from local files.
