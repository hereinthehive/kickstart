# Living memory, session bookends, safety net, and unified /update — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add six new skills, two subagents, a Stop hook, and onboarding integration to the starter template so a non-technical Claude Code user has living memory, session bookends, an opt-in git safety net, and a single `/update` audit command.

**Architecture:**
- Always-active skills and subagents ship in `.claude/skills/` and `.claude/agents/`.
- Conditional safety-net skills (`/undo`, `/checkpoint`) ship in `.claude/skill-templates/` and are copied into `.claude/skills/` by onboarding on opt-in.
- The existing `/knowledge-curator` skill is retired; its logic moves to `.claude/agents/curator.md`. The `/update` skill invokes both `curator` and `caretaker` subagents in parallel via the Agent tool and merges reports.
- The Stop hook writes a `.claude/.wrap-pending` marker (it cannot invoke a slash command directly); the SessionStart hook detects the marker on the next session and instructs Claude to offer a retroactive recap.
- Recurrence is via `/loop weekly /update`, not a settings.json `schedule` block.

**Tech Stack:** Markdown skills with YAML frontmatter, shell commands inside `settings.json` hooks, Claude Code's Agent tool for subagent invocation. No application code; no test framework. Verification is via the scenario matrix in the spec.

---

## File structure

**Create:**
- `.claude/agents/curator.md` — external-audit subagent (replaces current `/knowledge-curator` skill content)
- `.claude/agents/caretaker.md` — internal-audit subagent (new)
- `.claude/skills/update/SKILL.md` — unified user-facing audit command
- `.claude/skills/remember/SKILL.md` — save a preference
- `.claude/skills/forget/SKILL.md` — remove a saved preference
- `.claude/skills/wrap/SKILL.md` — write end-of-session recap
- `.claude/skills/help-me/SKILL.md` — plain-language command list
- `.claude/skill-templates/undo/SKILL.md` — staged for safety-net opt-in
- `.claude/skill-templates/checkpoint/SKILL.md` — staged for safety-net opt-in
- `.claude/skill-templates/README.md` — explains why this directory exists

**Modify:**
- `.claude/skills/onboarding/SKILL.md` — Phase 1 git detection; Phase 2 safety-net opt-in; Phase 3 curator-subagent invocation; Phase 4 file creation, CLAUDE.md sections, hook installation, gitignore; Phase 7 handoff
- `.claude/settings.json` — remove `schedule` block; rewrite SessionStart hook; add Stop hook
- `README.md` — mention new commands and the natural-language affordance

**Delete:**
- `.claude/skills/knowledge-curator/SKILL.md` — content moves to `.claude/agents/curator.md`

---

## Implementation notes

These are decisions the spec was hand-wavy about that the engineer needs to know:

1. **Stop hook cannot invoke a slash command.** It writes `.claude/.wrap-pending` with session metadata (end timestamp, substantive-turn count, files-touched list). The SessionStart hook reads this file on the next session and emits a system message asking Claude to offer a retroactive `/wrap`. If the user declines, `.wrap-pending` is deleted. If they accept, `/wrap` runs against the metadata + git diff and writes the recap.

2. **Substantive-turn heuristic.** Count of assistant tool uses in the transcript matching: `Edit`, `Write`, `MultiEdit`, `NotebookEdit`, or `Bash` with a command that starts with `git`, `npm`, `pnpm`, `yarn`, `cargo`, `make`, `pytest`, or modifies files (`sed -i`, `mv`, `cp`, `rm`). Threshold: ≥5.

3. **Session-start timestamp file.** SessionStart hook writes `.claude/.session-start` containing `date +%s` only when the safety net is opted in. `/undo` Mode 2 reads it. The file is gitignored.

4. **Conditional skill activation.** Onboarding's safety-net opt-in path copies `.claude/skill-templates/{undo,checkpoint}/` to `.claude/skills/{undo,checkpoint}/`. The skill-templates directory is never auto-discovered by Claude Code (it's not a known path), so unactivated skills don't pollute the command surface.

5. **`/update` subagent fan-out.** The `/update` skill invokes both subagents using the Agent tool with parallel calls (one message, two `Agent` tool uses). The skill body merges the two returned reports into a single output and writes `.claude/update-report.md`.

6. **Curator skill retirement.** Existing forks running onboarding refresh will have the old `.claude/skills/knowledge-curator/` directory deleted and a one-line note added to `.claude/onboarding-log.md`.

---

## Task 1: Migrate `/knowledge-curator` to a subagent

**Files:**
- Create: `.claude/agents/curator.md`
- Delete: `.claude/skills/knowledge-curator/SKILL.md` (and parent directory if empty)

- [ ] **Step 1: Create `.claude/agents/` directory**

```bash
mkdir -p .claude/agents
```

- [ ] **Step 2: Write `.claude/agents/curator.md`**

```markdown
---
name: curator
description: External audit subagent. Reviews the Claude Code setup against current docs and best practices. Invoked by /update; returns structured findings.
tools: Read, Write, Edit, Bash, WebSearch, WebFetch
---

You are the curator subagent. Your job is to audit the project's Claude Code setup against current external best practices and return structured findings to the caller (`/update`).

## Inputs you receive

- The caller passes the project root path. Read these files yourself:
  - `CLAUDE.md`
  - `.claude/settings.json`
  - `.claude/skills/` directory listing
  - `.claude/knowledge.md` if it exists (previous baseline)

## Step 1: Fetch current Claude Code documentation

Use WebSearch for:
- "Claude Code skills SKILL.md frontmatter"
- "Claude Code hooks SessionStart Stop best practices"
- "Claude Code settings.json permissions allow deny"
- "Claude Code subagents agents directory"
- "Claude Code /loop schedule recurring"
- "site:docs.anthropic.com claude code"

Look for features or patterns that are recent.

## Step 2: Review across these dimensions

- **CLAUDE.md quality** — context complete? run/test/build documented? preferences and constraints captured?
- **Settings and permissions** — necessary allows present? hooks meaningful? Stop hook present?
- **Skills** — repetitive workflows captured? /update present and schedulable via /loop?
- **Subagents** — curator and caretaker discoverable in `.claude/agents/`?
- **Deprecated patterns** — anything outdated?

## Step 3: Return findings

Return (don't write files — the caller merges and writes the report) a JSON-ish block:

```
### Curator findings

**What's healthy (external):**
- bullet
- bullet

**Gaps (external):**
- bullet — one-line reason
- bullet — one-line reason

**Quick wins:**
- bullet — concrete change
```

The caller (`/update`) merges this with caretaker findings before presenting to the user.

## Hard rule

Recommend, never apply. Do not edit files. Return findings only.
```

- [ ] **Step 3: Delete the old skill**

```bash
rm -rf .claude/skills/knowledge-curator
```

- [ ] **Step 4: Verify**

```bash
ls .claude/agents/curator.md && ls .claude/skills/knowledge-curator 2>&1
```

Expected: curator.md exists; `ls: .claude/skills/knowledge-curator: No such file or directory`.

- [ ] **Step 5: Commit**

```bash
git add .claude/agents/curator.md .claude/skills/knowledge-curator
git commit -m "Migrate /knowledge-curator skill to curator subagent"
```

---

## Task 2: Create the caretaker subagent

**Files:**
- Create: `.claude/agents/caretaker.md`

- [ ] **Step 1: Write `.claude/agents/caretaker.md`**

```markdown
---
name: caretaker
description: Internal-integrity audit subagent. Checks preference rot, adoption signal, and shim/config integrity. Invoked by /update; returns structured findings.
tools: Read, Bash, Grep, Glob
---

You are the caretaker subagent. Your job is to audit the project's internal state and return structured findings to the caller (`/update`). You do NOT modify files. You report.

## Inputs you receive

The caller passes the project root path. Read:
- `CLAUDE.md` (preferences and structure)
- `.claude/last-session.md` if it exists (modified time matters)
- `.claude/checkpoints.log` if it exists (size and modified time)
- `.gitignore`
- `.claude/onboarding-log.md` if it exists
- `.claude/.session-start` if it exists
- Stash list via `git stash list 2>/dev/null` (filter for `undo:` prefix)

## Step 1: Preference rot audit

Read CLAUDE.md's `## Preferences` section (between the `<!-- managed by /remember -->` marker and the next `##` heading).

For each preference line, check:
- **Stale referent** — does it mention a file, script, command, or tool? Does that thing still exist? Use `Grep`/`Glob` to verify.
- **Personal-sounding** — first-person singular phrasing ("I prefer", "my..."). Flag for suggested move to `~/.claude/CLAUDE.md`.
- **Vague** — no clear actionable referent. Flag for rewrite or removal.
- **Subtle drift** — does this preference look like a conditional refinement of an earlier preference (not a direct contradiction)? E.g., "always X" + later "skip X for hotfixes." Flag the cluster.

## Step 2: Adoption-signal audit (kill criteria)

Compute these signals:

- **`/wrap` adoption.** Modified date of `.claude/last-session.md`. If missing, never written. If older than 28 days, stale.
- **`/checkpoint` adoption.** Size of `.claude/checkpoints.log` and modified date. If unchanged for 30 days, stale.
- **`/undo` adoption.** `git stash list | grep -c '^stash@.*: undo:'`. If 0 and stash list is otherwise non-trivial, never used.
- **`/remember` adoption.** Count of bullet lines in the Preferences section. If <2 after 30 days of project activity (use `git log --since=30.days.ago | wc -l` as a proxy for activity), under-used.

For each, recommend removal if the threshold is crossed. Use conservative thresholds.

## Step 3: Shim and config integrity

- `.gitignore` contains `.claude/last-session.md`. If safety-net active (check by presence of `.claude/skills/undo/`), also `.claude/checkpoints.log` and `.claude/.session-start`.
- `.claude/.session-start` exists if safety-net active. If missing, the SessionStart hook isn't writing it.
- Preferences section's HTML marker is intact (`<!-- managed by /remember and /forget`). If missing, `/remember` and `/forget` can't function.
- `.claude/onboarding-log.md` exists if onboarding has been run.

## Step 4: Return findings

Return a structured block:

```
### Caretaker findings

**What's healthy (internal):**
- bullet
- bullet

**Preference rot:**
- [line N of Preferences] — issue — suggested fix
- ...

**Adoption signal:**
- /wrap: [active | stale (last write N days ago) | never used] → [recommendation]
- /checkpoint: [...]
- /undo: [...]
- /remember: [N entries, recommendation]

**Shim/config integrity:**
- bullet for each issue, or "all good"

**Recommended removals (require user approval):**
- bullet, with one-line justification
```

## Hard rule

Recommend, never apply. Do not edit files or run destructive commands. Return findings only.
```

- [ ] **Step 2: Verify**

```bash
ls .claude/agents/caretaker.md
```

Expected: file exists.

- [ ] **Step 3: Commit**

```bash
git add .claude/agents/caretaker.md
git commit -m "Add caretaker subagent for internal-integrity audits"
```

---

## Task 3: Create the `/update` skill

**Files:**
- Create: `.claude/skills/update/SKILL.md`

- [ ] **Step 1: Write `.claude/skills/update/SKILL.md`**

```markdown
---
description: Audit the project's Claude Code setup. Runs the curator (external best practices) and caretaker (internal integrity) subagents in parallel and presents merged findings in plain language.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash Agent
---

You are the `/update` orchestrator. Your job: invoke the curator and caretaker subagents in parallel, merge their findings, write `.claude/update-report.md`, and present the user with a friendly summary.

## Step 1: Tell the user what's about to happen

Brief plain-language preview:

> "I'll check your setup against current Claude Code best practices (curator) and audit the project's own state for clutter or rot (caretaker). About 30-60 seconds. I won't change anything without asking."

## Step 2: Invoke both subagents in parallel

In a single message, call the Agent tool twice:

- `subagent_type: curator` — prompt: "Audit the Claude Code setup in `$(pwd)` against current external best practices. Read CLAUDE.md, .claude/settings.json, .claude/skills/, and .claude/knowledge.md if present. Return findings in the format your skill specifies. Do not modify files."
- `subagent_type: caretaker` — prompt: "Audit the project's internal integrity in `$(pwd)`. Check preference rot, adoption signals (kill criteria), and shim/config integrity per your skill's instructions. Return findings in the format your skill specifies. Do not modify files."

Both run concurrently. Wait for both before proceeding.

## Step 3: Merge findings

Compose `.claude/update-report.md` with this exact structure:

```
# Setup audit — [today's date]

## In plain words
[Friendly 2-3 sentence summary covering BOTH audits. Lead with overall health.
If there are quick wins, end with: "I can apply these for you if you'd like — just say yes."
Plain language only. No jargon.]

## What's healthy
[merged bullets from both subagents — "external" and "internal" tagged]

## What needs attention
[merged bullets, sorted by severity]

## Recommended removals
[caretaker's kill-criteria findings + any curator-flagged deprecations]

## Detailed findings
### From curator (external)
[curator's full structured output]

### From caretaker (internal)
[caretaker's full structured output]

## Next steps
- Want me to apply the quick wins? Say yes.
- Want to ignore a recommendation? Tell me which.
- Want more detail on something? Ask.
```

## Step 4: Present to the user

Show only the "In plain words" section in your response. The full report lives in `.claude/update-report.md` for reference. Then ask:

> "Want me to apply any of these? Tell me which, or say 'none' and we're done."

## Step 5: Apply on confirmation

If the user approves quick wins, apply them. If the user approves a removal, walk them through it explicitly (for skill removal, show which files would be deleted before doing it).

## Hard rule

Never auto-prune. Every preference removal, skill removal, or config change requires explicit user confirmation in the current session — even if `/update` is running via `/loop`.

## When invoked on a schedule (via `/loop weekly /update`)

Same flow. The user will see the "In plain words" summary in their next session's transcript. They can act on it then or not at all.
```

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/update/SKILL.md
```

Expected: file exists.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/update/SKILL.md
git commit -m "Add /update skill — unified curator + caretaker audit"
```

---

## Task 4: Create the `/remember` skill

**Files:**
- Create: `.claude/skills/remember/SKILL.md`

- [ ] **Step 1: Write `.claude/skills/remember/SKILL.md`**

```markdown
---
description: Save a preference to CLAUDE.md so it persists across sessions. Triggered by /remember or natural-language phrases like "remember to..." or "from now on, always...".
disable-model-invocation: true
allowed-tools: Read Edit Write
---

You are the `/remember` skill. Save a preference to CLAUDE.md's Preferences section so it persists across future sessions.

## Triggers

- Slash form: `/remember <preference text>`
- Natural language: "remember to...", "from now on, always...", "always X", "never X"

## Step 1: Parse the preference

Extract the user's intended preference. Normalize to one of:
- `Always <verb> <object> [context]`
- `Never <verb> <object> [context]`
- `Prefer <X> over <Y>`
- Or leave verbatim if it doesn't match a pattern.

## Step 2: Personal vs. project disambiguation

If the preference uses first-person singular phrasing ("I prefer", "I like", "my"), ask once:

> "Is this just for you, or should everyone working on this project follow it?"

- "Just me" → write to `~/.claude/CLAUDE.md` under the heading `## Project: <project-name>`.
- "Everyone" / "the project" → continue to Step 3 (write to CLAUDE.md).

## Step 3: Ambiguity check

If the preference is a project-style preference but lacks an engineering, project, or workflow vocabulary (e.g., "remember to call mom"), confirm:

> "Want me to save that as a project preference, or did you mean something else?"

On "something else," abort the skill. On "yes, save it," continue.

## Step 4: Locate the Preferences section in CLAUDE.md

Read CLAUDE.md. Look for the HTML comment marker:

```
<!-- managed by /remember and /forget — items added below are remembered across sessions -->
```

If the marker is missing, insert a `## Preferences` section with the marker near the end of CLAUDE.md (before any auto-generated trailers, after the Constraints section if present).

## Step 5: Contradiction check

For each existing bullet under the marker, ask: does the new preference directly contradict it? (e.g., new "Never use prettier" vs. existing "Always use prettier.")

If yes, surface to the user:

> "That contradicts an existing preference: '<existing>'. Want me to replace the old one, or keep both?"

On "replace," remove the existing line. On "keep both," continue without removal.

## Step 6: Append the preference

Add a new bullet under the marker:

```
- <normalized preference>.
```

Preserve any existing bullets. Sort order: append, do not re-order.

## Step 7: Confirm to the user

In plain English:

> "Got it. <repeat the preference>. I've added this to your CLAUDE.md."

## Edge cases

- **Empty input.** Ask: "What should I remember?"
- **Vague preference** (e.g., "be better"). Push back once: "That's a bit broad — could you be specific about when or how?" If the user insists, save verbatim.
- **Preference already present.** Confirm: "I already have that saved. No change needed."
```

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/remember/SKILL.md
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/remember/SKILL.md
git commit -m "Add /remember skill"
```

---

## Task 5: Create the `/forget` skill

**Files:**
- Create: `.claude/skills/forget/SKILL.md`

- [ ] **Step 1: Write `.claude/skills/forget/SKILL.md`**

```markdown
---
description: Remove a saved preference from CLAUDE.md. Triggered by /forget or phrases like "forget about..." or "stop remembering...".
disable-model-invocation: true
allowed-tools: Read Edit
---

You are the `/forget` skill. Remove a preference from CLAUDE.md's Preferences section.

## Triggers

- Slash form: `/forget` (no arg) or `/forget <text>`
- Natural language: "forget about...", "stop remembering...", "drop the preference about..."

## Step 1: Locate the Preferences section

Read CLAUDE.md. Find the HTML comment marker:

```
<!-- managed by /remember and /forget — items added below are remembered across sessions -->
```

If the section is missing or empty, tell the user: "There are no saved preferences yet."

## Step 2: List and pick

**If no argument** is provided:

Number each existing preference and present:

> "Here's what I'm currently remembering:
> 1. Always run prettier before committing.
> 2. Never push to main directly.
> 3. Prefer integration tests over mocks.
>
> Which would you like me to forget? (number or short phrase)"

Wait for response.

**If an argument** is provided:

Find the best-matching line via case-insensitive substring match. If multiple match, list them and ask which.

## Step 3: Confirm

Show the matched line and ask:

> "Remove this? '<line>'  (yes/no)"

On "no," abort.

## Step 4: Remove

Edit CLAUDE.md to delete the matched bullet. Preserve all other bullets and the marker.

## Step 5: Confirm to the user

> "Done — I've forgotten '<line>'."

## Edge cases

- **No match.** Tell the user: "I couldn't find a preference matching that. Want me to list them all?"
- **Multiple matches.** List numbered options; ask which.
- **User says 'all' or 'everything'.** Ask twice: "Remove ALL preferences? This will clear the section entirely. (yes/no)" — require a second yes.
```

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/forget/SKILL.md
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/forget/SKILL.md
git commit -m "Add /forget skill"
```

---

## Task 6: Create the `/wrap` skill

**Files:**
- Create: `.claude/skills/wrap/SKILL.md`

- [ ] **Step 1: Write `.claude/skills/wrap/SKILL.md`**

```markdown
---
description: Write an end-of-session recap to .claude/last-session.md. Triggered by /wrap, "wrap up", "recap this session", or automatically by the Stop hook via the .claude/.wrap-pending marker.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash
---

You are the `/wrap` skill. Write a session recap that survives to the next session.

## Triggers

- Slash form: `/wrap`
- Natural language: "wrap up", "recap this session", "let's call it"
- Retroactive (via SessionStart): if `.claude/.wrap-pending` exists from a previous session

## Mode A: Live invocation (end of current session)

### Step 1: Substantive-check

Briefly assess whether this session covered substantive work. If you've made fewer than ~3 file edits or git operations, ask the user:

> "You've only just started — are you sure you want to wrap up? There's not much to recap yet."

Continue only on confirmation.

### Step 2: Draft the recap

Compose a recap from your conversation context using this format exactly:

```
# Last session — <YYYY-MM-DD>

## What we did
- [2-4 bullets of substantive work, not micro-steps]

## Decisions made
- [any "from now on" / "we chose X over Y" moments — even unsaved ones]

## In flight
- [anything unfinished, with enough context to resume]

## Next time, start with
- [one concrete next action]
```

### Step 3: Show the draft

Present the recap to the user for approval:

> "Here's the recap I'll save. Look right? (yes / edits)"

Apply edits if requested. Re-show if substantial changes were made.

### Step 4: Save

Write `.claude/last-session.md`, overwriting any previous recap.

### Step 5: Offer to save "from now on" decisions

If the recap's "Decisions made" section contains any "from now on" / "always X" lines that aren't already in CLAUDE.md's Preferences, offer one by one:

> "I noticed you decided '<decision>' during this session. Want me to remember that across future sessions too?"

On yes, invoke the `/remember` skill with that decision text. On no, move on.

### Step 6: Confirm

> "Recap saved. You'll see 'Where we left off' at the start of your next session."

## Mode B: Retroactive (called from SessionStart hook detecting .wrap-pending)

### Step 1: Read the marker

`.claude/.wrap-pending` is a key=value file:

```
SESSION_END=<unix-timestamp>
SESSION_START=<unix-timestamp>
SUBSTANTIVE_TURNS=<integer>
FILES_TOUCHED=<comma-separated paths>
```

### Step 2: Ask the user

> "Last session ended without a recap. It had <N> substantive turns. Want me to put together a quick summary based on the git activity? (yes / no / skip forever)"

- yes → continue to Step 3.
- no → delete `.claude/.wrap-pending`; move on.
- skip forever → write a flag to `.claude/onboarding-log.md` so future SessionStart hooks know not to offer retroactive recaps; delete `.wrap-pending`.

### Step 3: Reconstruct context from git

Run:
- `git log --since="<SESSION_START>" --until="<SESSION_END>" --pretty=format:'%h %s'`
- `git diff --stat <commit-before-SESSION_START>..<commit-at-SESSION_END>` (best-effort)

Use the FILES_TOUCHED list and commit messages to draft the same recap format as Mode A. Be explicit about uncertainty: prefix any inferred bullets with "Probably: ".

### Step 4: Show and save

Same as Mode A Steps 3-6. After saving, delete `.claude/.wrap-pending`.

## Edge cases

- **Stop hook fired but user opens new session days later.** SESSION_END timestamp shows. Surface the age: "Last session ended 4 days ago." Let the user decide whether to bother reconstructing.
- **`.claude/last-session.md` already has a recap newer than `.wrap-pending`.** Delete `.wrap-pending` silently; do nothing.
- **Force-quit detected (no SESSION_END).** Treat as zero substantive turns; do nothing.
```

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/wrap/SKILL.md
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/wrap/SKILL.md
git commit -m "Add /wrap skill with live and retroactive modes"
```

---

## Task 7: Create the `/help-me` skill

**Files:**
- Create: `.claude/skills/help-me/SKILL.md`

- [ ] **Step 1: Write `.claude/skills/help-me/SKILL.md`**

```markdown
---
description: Show a plain-language list of available commands and natural-language phrases. Triggered by /help-me or "what can you do?" / "what commands do I have?".
disable-model-invocation: true
allowed-tools: Read Bash
---

You are the `/help-me` skill. Show the user what commands they have, written in their voice and in plain language.

## Step 1: Detect what's active

- Always-present skills: `/remember`, `/forget`, `/wrap`, `/update`, `/help-me`.
- Safety-net skills: check whether `.claude/skills/undo/` and `.claude/skills/checkpoint/` exist.
- Any other skills in `.claude/skills/` that you didn't ship — list them too (the user or onboarding may have added them).

Run:
```bash
ls .claude/skills/
```

## Step 2: Show the list

In plain language, grouped by purpose:

> Here's what you can ask me to do:
>
> **Just talking** — you don't need to remember slash commands; just say it:
> - *"Remember to X"* — I'll save it as a preference (also: `/remember X`)
> - *"Forget about X"* — I'll remove a preference (also: `/forget`)
> - *"Wrap up"* or *"Recap this session"* — I'll write a recap before you go (also: `/wrap`)
> - *"Update my setup"* or *"Check my setup"* — I'll audit everything (also: `/update`)
>
> [If safety net is active:]
> - *"Save a checkpoint"* — I'll snapshot before something risky (also: `/checkpoint`)
> - *"Undo that"* — I'll reverse the last change (also: `/undo`)
> - *"Restore the last undo"* — I'll bring back what we undid
>
> **For automatic weekly audits:** run `/loop weekly /update` once.

## Step 3: Offer to drill in

> Want more detail on any of these? Just ask.
```

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/help-me/SKILL.md
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/help-me/SKILL.md
git commit -m "Add /help-me skill"
```

---

## Task 8: Stage the `/undo` skill template

**Files:**
- Create: `.claude/skill-templates/README.md`
- Create: `.claude/skill-templates/undo/SKILL.md`

- [ ] **Step 1: Write `.claude/skill-templates/README.md`**

```markdown
# Skill templates

This directory holds skills that aren't active by default. Onboarding copies
them into `.claude/skills/` when the user opts in to the relevant feature
(e.g., the safety net).

Claude Code does not auto-discover skills in this directory, so unactivated
skills don't appear as commands.

To activate a skill manually, copy its folder to `.claude/skills/`:

    cp -r .claude/skill-templates/undo .claude/skills/

Then restart your session so the new skill is picked up.
```

- [ ] **Step 2: Write `.claude/skill-templates/undo/SKILL.md`**

```markdown
---
description: Reverse the last assistant action via git stash or git reset --soft. Triggered by /undo or "undo that" / "take that back". Refuses on pushed history, hard resets, or merge conflicts.
disable-model-invocation: true
allowed-tools: Read Bash
---

You are the `/undo` skill. Reverse the assistant's most recent change without destroying work.

## Triggers

- Slash form: `/undo`
- Natural language: "undo that", "take that back", "revert what you just did"

## Hard refusals (check first)

Run these checks. If any return non-empty/non-zero, refuse with the corresponding message:

1. **Pushed commits.** `git log @{upstream}..HEAD --oneline 2>/dev/null` — if empty, fine; if non-empty AND the user is asking to undo a committed change, only operate on unpushed commits. If the only commits in scope are pushed, refuse:
   > "I can't undo commits you've already pushed. That would rewrite shared history. Want help doing it manually?"

2. **Merge in progress.** `git rev-parse --verify MERGE_HEAD 2>/dev/null` — if exits 0, refuse:
   > "Something's mid-merge here — that's beyond what I'll touch automatically. Want me to walk you through it?"

3. **No git repo.** `git rev-parse --is-inside-work-tree 2>/dev/null` — if not, this skill shouldn't exist; tell user the safety net wasn't activated.

## Step 1: Detect mode

Run `git status --porcelain` and capture output.

- **Output non-empty (dirty tree)** → Mode 1: undo uncommitted changes.
- **Output empty + recent commits in session** → Mode 2: undo a session commit.
- **Output empty + no session commits** → "There's nothing to undo right now."

To detect session commits, read `.claude/.session-start` for the session-start timestamp and run:

```bash
git log --since="@$(cat .claude/.session-start)" --pretty=format:'%h %s'
```

## Mode 1: Uncommitted changes

### Step 1.1: Show what will be undone

```bash
git status --porcelain
```

Translate to plain English:

> "I'm about to undo these changes:
> - Modified: src/foo.js
> - Modified: src/bar.js
> - New file: src/baz.js
>
> Nothing will be deleted permanently — I'll save them as a checkpoint first so you can restore if needed. OK to proceed? (yes / no)"

### Step 1.2: Stash on confirm

```bash
git stash push -u -m "undo: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### Step 1.3: Confirm

> "Done. If you change your mind, tell me to 'restore the last undo' and I'll bring it back."

## Mode 2: Committed in this session

### Step 2.1: Pre-flight stash of any uncommitted changes

Run `git status --porcelain`. If non-empty:

```bash
git stash push -u -m "undo-preflight: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

Tell the user:

> "You have unsaved changes on top of the commit. I'll set those aside first, then undo the commit. Both are recoverable."

### Step 2.2: List session commits

Show them with numbers:

> "I made these commits in this session:
> 1. a1b2c3d Add login form
> 2. e4f5g6h Fix typo
>
> Which would you like to undo? (number — only the most recent is straightforward, older requires care)"

### Step 2.3: Refuse old history

If the user picks a commit older than the session-start timestamp:

> "That commit is from before we started today. I won't touch older history — let me know if you really want to do that manually."

### Step 2.4: Soft reset

For the chosen commit, run:

```bash
git reset --soft <commit>^
```

This keeps changes staged. Confirm:

> "Done. Your changes are staged but the commit is gone. You can re-commit differently or use `git restore --staged .` to unstage."

### Step 2.5: Re-apply pre-flight stash

If a pre-flight stash was created:

```bash
git stash pop
```

Tell the user:

> "I've also put back the unsaved changes you had."

## Restoration

Triggered by: "restore the last undo", "bring back what we undid".

```bash
git stash pop
```

If multiple `undo:` stashes exist, list and ask which.

## Hard rules

- Never run `git reset --hard`.
- Never run `git clean`.
- Never touch pushed commits.
- Every operation must be reversible (stash and soft reset are; hard reset and clean are not).
```

- [ ] **Step 3: Verify**

```bash
ls .claude/skill-templates/undo/SKILL.md
ls .claude/skill-templates/README.md
```

- [ ] **Step 4: Commit**

```bash
git add .claude/skill-templates/undo .claude/skill-templates/README.md
git commit -m "Stage /undo skill template for safety-net opt-in"
```

---

## Task 9: Stage the `/checkpoint` skill template

**Files:**
- Create: `.claude/skill-templates/checkpoint/SKILL.md`

- [ ] **Step 1: Write `.claude/skill-templates/checkpoint/SKILL.md`**

```markdown
---
description: Save a recoverable snapshot of the current working tree via git stash. Triggered by /checkpoint or "save a checkpoint" / "snapshot this".
disable-model-invocation: true
allowed-tools: Read Bash
---

You are the `/checkpoint` skill. Save a recoverable snapshot before something risky.

## Triggers

- Slash form: `/checkpoint [optional description]`
- Natural language: "save a checkpoint", "snapshot this", "before we do that, save where we are"

## Step 1: Hard refusals (check first)

- **No git repo.** `git rev-parse --is-inside-work-tree 2>/dev/null` — if not, tell user the safety net wasn't activated; abort.
- **Merge in progress.** `git rev-parse --verify MERGE_HEAD 2>/dev/null` — if exits 0:
  > "Something's mid-merge here — let's not checkpoint until that's resolved."

## Step 2: Infer a description

If the user passed a description, use it. Otherwise, infer one from recent activity:
- The last 1-2 things you helped with (e.g., "before refactoring login form").
- Fall back to a timestamp if nothing distinctive: "checkpoint at <time>".

## Step 3: Stash with the description

```bash
git stash push -u -m "checkpoint: $(date -u +%Y-%m-%dT%H:%M:%SZ) <description>"
```

## Step 4: Log it

Append to `.claude/checkpoints.log` (create if missing):

```
<ISO timestamp> | <description> | <stash ref returned by git stash list -1>
```

## Step 5: Confirm

> "Saved a checkpoint: '<description>'. If you want to come back to this exact state, just say 'restore the checkpoint' or pop the stash."

## Restoration

Triggered by "restore the checkpoint" or "go back to the checkpoint":

If multiple checkpoints exist (read `.claude/checkpoints.log`), list and ask which.

```bash
git stash apply stash@{N}
```

Note: use `apply` rather than `pop` so the checkpoint stays available for repeated restoration.

## Hard rules

- Never overwrite a checkpoint silently. Always create a new stash entry.
- Never delete a checkpoint without explicit user confirmation.
- The `.claude/checkpoints.log` is append-only from this skill's perspective.
```

- [ ] **Step 2: Verify**

```bash
ls .claude/skill-templates/checkpoint/SKILL.md
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skill-templates/checkpoint/SKILL.md
git commit -m "Stage /checkpoint skill template for safety-net opt-in"
```

---

## Task 10: Update settings.json — Stop hook, SessionStart hook, remove schedule

**Files:**
- Modify: `.claude/settings.json`

- [ ] **Step 1: Read the current file**

```bash
cat .claude/settings.json
```

- [ ] **Step 2: Replace with new content**

Write `.claude/settings.json` containing exactly:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(git *)",
      "Bash(ls *)",
      "Bash(wc *)",
      "Bash(cat *)",
      "Bash(mkdir *)",
      "Bash(cp *)",
      "Bash(date *)",
      "Agent"
    ]
  },
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "if [ ! -f CLAUDE.md ] || [ $(wc -c < CLAUDE.md) -lt 50 ]; then echo 'Setup not finished yet — run /onboarding to get started.'; else echo \"Branch: $(git branch --show-current 2>/dev/null || echo 'n/a')\"; echo \"Status: $(git status --short 2>/dev/null | head -5 || echo 'clean')\"; echo \"Last commit: $(git log --oneline -1 2>/dev/null || echo 'none')\"; fi; if [ -f .claude/skills/undo/SKILL.md ]; then mkdir -p .claude && date +%s > .claude/.session-start; fi; if [ -f .claude/last-session.md ]; then echo ''; echo '--- Where we left off ---'; awk '/## Next time, start with/{flag=1; next} /^## /{flag=0} flag' .claude/last-session.md | sed '/^$/d' | head -2; AGE_DAYS=$(( ( $(date +%s) - $(stat -f %m .claude/last-session.md 2>/dev/null || stat -c %Y .claude/last-session.md 2>/dev/null || echo 0) ) / 86400 )); if [ \"$AGE_DAYS\" -gt 1 ]; then echo \"(recap is $AGE_DAYS days old)\"; fi; fi; if [ -f .claude/.wrap-pending ]; then echo ''; echo '--- Pending recap ---'; echo 'Last session ended without a recap. Run /wrap to build one retroactively from git activity.'; fi; if [ -f .claude/update-report.md ]; then echo ''; echo '--- Last audit summary ---'; awk '/## In plain words/{flag=1; next} /^## /{flag=0} flag' .claude/update-report.md | sed '/^$/d' | head -3; fi; echo ''; echo 'Tip: run /loop weekly /update once to keep your setup current automatically.'"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "TRANSCRIPT=\"${CLAUDE_TRANSCRIPT_PATH:-}\"; if [ -z \"$TRANSCRIPT\" ] || [ ! -f \"$TRANSCRIPT\" ]; then exit 0; fi; SUBSTANTIVE=$(grep -c '\"name\":\"\\(Edit\\|Write\\|MultiEdit\\|NotebookEdit\\)\"' \"$TRANSCRIPT\" 2>/dev/null || echo 0); BASH_OPS=$(grep -oE '\"name\":\"Bash\".*?\"command\":\"(git|npm|pnpm|yarn|cargo|make|pytest|sed -i|mv |cp |rm )[^\"]*\"' \"$TRANSCRIPT\" 2>/dev/null | wc -l); TOTAL=$((SUBSTANTIVE + BASH_OPS)); if [ \"$TOTAL\" -ge 5 ]; then FILES=$(grep -oE '\"file_path\":\"[^\"]+\"' \"$TRANSCRIPT\" 2>/dev/null | sed 's/\"file_path\":\"//;s/\"$//' | sort -u | tr '\\n' ',' | sed 's/,$//'); mkdir -p .claude; printf 'SESSION_END=%s\\nSUBSTANTIVE_TURNS=%s\\nFILES_TOUCHED=%s\\n' \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\" \"$TOTAL\" \"$FILES\" > .claude/.wrap-pending; fi"
          }
        ]
      }
    ]
  }
}
```

Notes:
- The `schedule` block is removed entirely. Users invoke `/loop weekly /update` themselves.
- The SessionStart hook:
  - Checks `CLAUDE.md` (not `AGENTS.md` — no portability migration in this spec).
  - Writes `.claude/.session-start` only if `.claude/skills/undo/` is present (i.e., safety net is active).
  - Surfaces `.claude/last-session.md`'s "Next time, start with" line.
  - Surfaces `.claude/.wrap-pending` if it exists.
  - Surfaces the last `/update` report's plain-words summary.
  - Always shows the `/loop weekly /update` tip until we have a better detection mechanism.
- The Stop hook:
  - Reads the transcript path from `$CLAUDE_TRANSCRIPT_PATH`.
  - Counts substantive turns: Edit/Write/MultiEdit/NotebookEdit + Bash with file-modifying commands.
  - If ≥5, writes `.claude/.wrap-pending`.
  - Otherwise exits silently.

- [ ] **Step 3: Validate JSON**

```bash
python3 -m json.tool .claude/settings.json > /dev/null && echo OK
```

Expected: `OK`.

- [ ] **Step 4: Commit**

```bash
git add .claude/settings.json
git commit -m "Update settings.json: drop schedule, add Stop hook, extend SessionStart"
```

---

## Task 11: Update onboarding skill

**Files:**
- Modify: `.claude/skills/onboarding/SKILL.md`

- [ ] **Step 1: Read the current file**

```bash
cat .claude/skills/onboarding/SKILL.md
```

- [ ] **Step 2: Make targeted edits**

Apply these edits (using the Edit tool — find and replace each block):

**Edit A: Add Phase 1 git detection.** Inside Phase 1, after the "User preferences" bullet block, append:

```markdown
**Environment detection:**
- Run `git --version >/dev/null 2>&1 && git rev-parse --is-inside-work-tree >/dev/null 2>&1` to determine git state. Record state for Phase 2: (1) no git installed, (2) git installed but not a repo, (3) git installed + repo.
```

**Edit B: Add Phase 2 safety-net opt-in.** Inside Phase 2, near the "Their constraints and hard stops" subsection, insert a new subsection:

```markdown
*Safety-net opt-in (conditional)*

If git state is (3) — installed and a repo — ask via `AskUserQuestion`:

> "Want a safety net for taking back changes? I can add two simple commands: `/checkpoint` (save a snapshot you can come back to) and `/undo` (reverse the last thing I did). Both use git behind the scenes but you don't need to know git to use them."
>
> - **Yes, add them** (recommended for most projects)
> - **Not now** — you can ask me to add them later
> - **No, skip it** — I don't want git-based undo in this project

Record the decision in onboarding-log.md (Phase 6). On "yes," remember to activate the safety-net skills in Phase 4. On "not now," remember to re-ask on next refresh. On "no," remember to skip the question entirely on future refreshes.

If git state is (2), mention in Phase 7 handoff: *"This project isn't tracked by git. If you ever want a safety net for undoing changes, just say so and I can set that up."*

If git state is (1), say nothing about the safety net.
```

**Edit C: Update Phase 3 (consult curator).** Replace the current Phase 3 body with:

```markdown
## Phase 3: Consult the curator on methodology

Invoke the curator subagent (lives at `.claude/agents/curator.md`) via the Agent tool:

- subagent_type: `curator`
- prompt: "I'm running onboarding for a project. Based on current Claude Code best practices, recommend the right CLAUDE.md structure, hook patterns, and skill conventions for this project. Project context: <summary from Phase 2 interview>."

Use the curator's response as your build guide for Phase 4. If the subagent call fails, fall back to applying generally-known best practices and tell the user the curator wasn't reachable.
```

**Edit D: Update Phase 4 (build the setup).** Add a new sub-section "Skills setup" near where existing project-specific-skills guidance lives:

```markdown
### Skills setup (always-active)

The following ship with the template and are already in `.claude/skills/`:
- `/help-me`, `/remember`, `/forget`, `/wrap`, `/update`

Plus subagents at `.claude/agents/`:
- `curator`, `caretaker`

Verify they're present. If any are missing, abort and tell the user the template is incomplete.

### Skills setup (conditional — safety net)

If the user opted "yes" to the safety net in Phase 2:

```bash
mkdir -p .claude/skills
cp -r .claude/skill-templates/undo .claude/skills/undo
cp -r .claude/skill-templates/checkpoint .claude/skills/checkpoint
```

Then add these to the `permissions.allow` array in `.claude/settings.json`:

- `"Bash(git stash *)"`
- `"Bash(git status *)"`
- `"Bash(git log *)"`
- `"Bash(git reset --soft *)"`

Read the file, parse the JSON, append the entries, write back.

If the user opted "not now" or "no," do not copy the template files. Record the decision in the onboarding-log.

### CLAUDE.md additions

When writing CLAUDE.md in Phase 4, include these sections explicitly:

1. **`## Talking to me`** — verbatim from the spec, listing natural-language triggers for each active skill. Adjust which triggers appear based on whether the safety net is active.

2. **`## Preferences`** — with the HTML comment marker:
   ```
   ## Preferences
   <!-- managed by /remember and /forget — items added below are remembered across sessions -->
   ```
   Empty body. `/remember` will populate.

3. **`## Constraints`** — from the interview answers.

### Gitignore additions

If git state is (3):

```bash
[ -f .gitignore ] || touch .gitignore
grep -qxF '.claude/last-session.md' .gitignore || echo '.claude/last-session.md' >> .gitignore
grep -qxF '.claude/update-report.md' .gitignore || echo '.claude/update-report.md' >> .gitignore
grep -qxF '.claude/.wrap-pending' .gitignore || echo '.claude/.wrap-pending' >> .gitignore
```

If the safety net was opted in, also add:

```bash
grep -qxF '.claude/checkpoints.log' .gitignore || echo '.claude/checkpoints.log' >> .gitignore
grep -qxF '.claude/.session-start' .gitignore || echo '.claude/.session-start' >> .gitignore
```
```

**Edit E: Update Phase 5 to invoke curator subagent.** Replace any `Skill knowledge-curator` references with `Agent → curator` invocation, with the same review-pass purpose.

**Edit F: Update Phase 7 (handoff).** Replace the existing "Commands available" list with:

```markdown
- *"What can you do?"* or `/help-me` — Remind me what commands you have.
- *"Remember to X"* or `/remember X` — Save a preference for future sessions.
- *"Forget about X"* or `/forget` — Remove a saved preference.
- *"Recap this session"* or `/wrap` — Write a session recap. I'll also do it automatically when you close cleanly.
- *"Update my setup"* or `/update` — Audit project health.
- *(if safety net opted in) "Save a checkpoint"* or `/checkpoint` — Snapshot before risky operations.
- *(if safety net opted in) "Undo that"* or `/undo` — Take back the last thing I did.

**For automatic weekly audits, run this once:**
```
/loop weekly /update
```

This will run `/update` for you every week. You can stop it anytime by canceling the loop.
```

**Edit G: Update Phase 6 onboarding-log structure.** Include a new field for the safety-net decision:

```markdown
### Decisions on optional features
- Safety net (/undo, /checkpoint): [yes / not now / no]
```

- [ ] **Step 3: Verify the file still parses as valid markdown with frontmatter**

```bash
head -10 .claude/skills/onboarding/SKILL.md
```

Expected: frontmatter intact (`---` opening, `---` closing, recognizable fields).

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/onboarding/SKILL.md
git commit -m "Onboarding: integrate /update, safety-net opt-in, new skills, new hooks"
```

---

## Task 12: Update README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read the current file**

```bash
cat README.md
```

- [ ] **Step 2: Replace the "Built-in skills" table and add a "Day-to-day commands" section**

Find the existing table and replace with:

```markdown
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
```

- [ ] **Step 3: Verify**

```bash
grep -q "Built-in commands" README.md && grep -q "Talking to" README.md && echo OK
```

Expected: `OK`.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "Update README with new commands and natural-language affordance"
```

---

## Task 13: Walk the scenario matrix

Manual verification. Each row is a scenario from the spec — verify behavior in a scratch fork.

- [ ] **Step 1: Create a scratch fork**

```bash
cd /tmp
rm -rf starter-test
git clone /Users/hereinthehive/Sites/starter starter-test
cd starter-test
```

- [ ] **Step 2: Scenario — fresh fork**

Open in Claude Code. Run `/onboarding`. Verify:
- CLAUDE.md is written with `## Talking to me`, `## Preferences` (with HTML marker), `## Constraints` sections.
- `.claude/skills/{remember,forget,wrap,update,help-me}/` exist.
- `.claude/agents/{curator,caretaker}.md` exist.
- `.claude/skills/knowledge-curator/` does NOT exist.
- `settings.json` has no `schedule` block; Stop hook present.
- Phase 7 handoff mentions `/loop weekly /update`.

- [ ] **Step 3: Scenario — safety-net "yes"**

Re-run onboarding (or fresh clone). Choose "Yes" on safety-net opt-in. Verify:
- `.claude/skills/undo/SKILL.md` and `.claude/skills/checkpoint/SKILL.md` exist.
- `.claude/skill-templates/{undo,checkpoint}/` also still exist (we copy, don't move).
- `.gitignore` contains `.claude/checkpoints.log` and `.claude/.session-start`.
- `settings.json` permissions include `Bash(git stash *)` and three others.

- [ ] **Step 4: Scenario — `/remember` flow**

Type: *"remember to always run prettier before committing"*. Verify:
- Claude routes via the natural-language trigger to `/remember`.
- CLAUDE.md gains `- Always run prettier before committing.` under the marker.

Then type: `/remember never use prettier`. Verify:
- Contradiction detected; user is prompted with replace/keep-both.

- [ ] **Step 5: Scenario — `/wrap` live**

Make a small edit, then run `/wrap`. Verify:
- Recap draft shown before write.
- `.claude/last-session.md` written after approval.

- [ ] **Step 6: Scenario — Stop hook + retroactive wrap**

Make ≥5 substantive changes in a session (edits + git ops). Close the session cleanly. Open a new session. Verify:
- SessionStart hook surfaces "Pending recap" block.
- `/wrap` retroactive mode is invokable; produces a recap from git activity.

- [ ] **Step 7: Scenario — `/undo` Mode 1**

Make an uncommitted edit. Run `/undo`. Verify:
- Plain-English preview shown.
- On confirm: file is reverted; `git stash list` shows an `undo:` entry.
- "Restore the last undo" pops the stash.

- [ ] **Step 8: Scenario — `/undo` Mode 2 with mixed state**

Commit a change. Make another uncommitted edit. Run `/undo`. Verify:
- Skill stashes the uncommitted edit first (preflight stash).
- Then resets the commit softly.
- After: changes from the commit are staged; the unsaved edit is restored from the preflight stash.

- [ ] **Step 9: Scenario — `/update`**

Run `/update`. Verify:
- Both subagents invoked in parallel (one assistant turn with two Agent tool calls).
- `.claude/update-report.md` written.
- User sees only the "In plain words" summary first.

- [ ] **Step 10: Scenario — `/help-me`**

Run `/help-me`. Verify output lists commands with natural-language phrases. Verify it correctly notes whether safety net is active.

- [ ] **Step 11: Document any failures**

For any failed scenario, fix the underlying skill or hook, then re-run.

- [ ] **Step 12: Commit any fixes**

```bash
cd /Users/hereinthehive/Sites/starter
git add -A
git commit -m "Fix issues found during scenario walk-through"
```

(Skip this step if no fixes were needed.)

---

## Task 14: Run `/update` on the starter repo itself

- [ ] **Step 1: Run `/update`**

In the starter repo, type `/update`. Verify:
- Both subagents run.
- `.claude/update-report.md` is written.
- "In plain words" reports the setup as healthy (or surfaces specific real findings).

- [ ] **Step 2: Address any high-severity findings**

If anything is genuinely broken, fix and re-run.

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "Final cleanup from /update on starter repo"
```

(Skip if no changes.)

---

## Self-review checklist (run after writing the plan, fix inline)

- [x] **Spec coverage** — every section of the spec has at least one task.
  - Living memory → Tasks 4, 5
  - Bookends → Tasks 6, 10
  - Safety net → Tasks 8, 9, 11
  - `/update` → Tasks 1, 2, 3
  - Onboarding integration → Task 11
  - Curator retirement → Task 1
  - README → Task 12
  - Scenario verification → Task 13
  - Self-audit on the starter → Task 14
- [x] **Placeholder scan** — no TBDs, no "TODO", no "implement later" without code shown.
- [x] **Type consistency** — file paths match across tasks (`.claude/skills/`, `.claude/agents/`, `.claude/skill-templates/`). Skill names (`remember`, `forget`, `wrap`, `update`, `help-me`, `undo`, `checkpoint`) consistent throughout.

---

## Execution handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-20-living-memory-and-safety-net-implementation.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — I execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
