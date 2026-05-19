---
description: Review the current Claude Code setup against latest best practices and documentation — identifies gaps, outdated patterns, and missed features
allowed-tools: Read Bash WebSearch WebFetch
---

## Current project setup

### CLAUDE.md
!`cat CLAUDE.md 2>/dev/null || echo "(no CLAUDE.md found)"`

### .claude/settings.json
!`cat .claude/settings.json 2>/dev/null || echo "(no settings.json found)"`

### Available skills
!`ls .claude/skills/ 2>/dev/null || echo "(no skills directory)"`

---

You are a Claude Code knowledge curator. Review the setup above against current documentation and best practices.

## Step 1: Fetch current Claude Code documentation

Use WebSearch to find current information. Search for:
- "Claude Code skills SKILL.md frontmatter"
- "Claude Code hooks SessionStart best practices"
- "Claude Code settings.json permissions"
- "Claude Code schedule recurring tasks"
- "Claude Code CLAUDE.md what to include"
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
- Is /schedule configured for recurring tasks like this weekly review?
- Are MCP servers relevant and configured?
- Any deprecated patterns in use?

## Step 3: Report findings

### What's working well
[Specific bullet points — not generic praise]

### Gaps and improvements
[Each with a one-line reason why it matters]

### Quick wins
[Concrete, specific changes that can be applied immediately]

---

If running standalone, offer to apply the quick wins immediately after presenting the report.

If running as a sub-agent for /onboarding, return the report only — the orchestrator will apply changes.
