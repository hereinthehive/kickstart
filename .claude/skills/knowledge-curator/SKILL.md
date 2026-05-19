---
description: Review the current Claude Code setup against latest best practices and documentation — identifies gaps, outdated patterns, and missed features
disable-model-invocation: true
allowed-tools: Read Write Edit Bash WebSearch WebFetch
---

## Current project setup

### CLAUDE.md
!`cat CLAUDE.md 2>/dev/null || echo "(no CLAUDE.md found)"`

### .claude/settings.json
!`cat .claude/settings.json 2>/dev/null || echo "(no settings.json found)"`

### Available skills
!`ls .claude/skills/ 2>/dev/null || echo "(no skills directory)"`

### Previous knowledge base
!`cat .claude/knowledge.md 2>/dev/null || echo "(no previous knowledge — first run)"`

---

You are a Claude Code knowledge curator. Review the setup above against current documentation and best practices, then persist your findings.

## Step 1: Fetch current Claude Code documentation

Use WebSearch to find current information. Search for:
- "Claude Code skills SKILL.md frontmatter"
- "Claude Code hooks SessionStart best practices"
- "Claude Code settings.json permissions"
- "Claude Code schedule recurring tasks"
- "Claude Code CLAUDE.md what to include"
- "Claude Code MCP servers configuration"
- "site:docs.anthropic.com claude code" for official docs

Look specifically for features or patterns that are recent — Claude Code evolves quickly and new capabilities are easy to miss.

## Step 2: Review the setup

Evaluate across these dimensions:

**CLAUDE.md quality**
- Does it give Claude enough context to work without asking basics?
- Are run/test/build commands documented?
- Are preferences and constraints captured?

**Settings and permissions**
- Are necessary tool permissions explicitly allowed?
- Are there missing hooks that would improve the workflow?
- Does the SessionStart hook meaningfully orient Claude?

**Skills**
- Are there repetitive workflows that deserve a /skill?
- Is /knowledge-curator itself present and configured for periodic runs?

**Feature gaps**
- Is /schedule configured for recurring tasks?
- Are MCP servers relevant and configured?
- Any deprecated patterns in use?

## Step 3: Compile findings

Note what's new or changed compared to the previous knowledge base (if any). If this is the first run, everything is new.

Structured findings:

### What's working well
[Specific bullet points]

### Gaps and improvements
[Each with a one-line reason why it matters]

### Quick wins
[Concrete changes that can be applied immediately]

## Step 4: Update .claude/knowledge.md

Write the full updated knowledge base to `.claude/knowledge.md`. Use this exact format:

```
# Knowledge Base

_Last updated: [today's date]_

## In plain words
[A friendly 2–3 sentence summary anyone can understand. Lead with overall health: "Your setup is healthy" or "A few small things need attention". If there are quick wins, end with: "I can fix these for you if you'd like — just say yes." This section is for non-technical users; avoid jargon entirely.]

## Summary
[One or two sentences describing the overall state of the setup — this is what the SessionStart hook surfaces. Slightly more technical than "In plain words" but still concise.]

## Latest findings

### What's working well
[bullet list]

### Gaps and improvements
[bullet list — each item one-line reason why it matters]

### Quick wins
[bullet list — concrete, applicable now]

## What changed since last review
[bullet list of anything new compared to previous run, or "First run — no previous baseline" if applicable]

## Run history
| Date | Trigger | Summary |
|------|---------|---------|
| [today] | [manual/scheduled] | [one-line summary] |
[preserve any previous rows here]
```

## Step 5: Present and offer to apply

Lead with the "In plain words" summary — that's what most users will read. Only show the detailed findings if the user asks, or if they're a power user who would want them. Then:
- If running standalone: offer to apply the quick wins immediately, in plain language ("I found 2 small improvements — want me to apply them?")
- If running as a sub-agent for /onboarding: return the report — the orchestrator will apply changes
