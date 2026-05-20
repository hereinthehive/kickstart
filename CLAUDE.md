# Kickstart

A Claude Code starter template with built-in onboarding and knowledge curation.

## Getting Started

Run `/onboarding` to configure this project for your needs. It will:
- Learn about your project and how you like to work
- Create a tailored CLAUDE.md and settings
- Have the Knowledge Curator review the setup against current best practices
- Hand off with clear instructions on what was created and what to do next

## Available Commands

| Command | Purpose |
|---|---|
| `/onboarding` | Set up or refresh Claude Code for this project |
| `/knowledge-curator` | Review the current setup against latest Claude Code best practices |

## How it works

**Onboarding** is the orchestrator — it interviews you, reads your project, builds the structure, then calls the **Knowledge Curator** to review everything before handing off to you.

The **Knowledge Curator** can also run independently to audit your setup as Claude Code evolves (new features, changed best practices, etc.). It runs automatically on a weekly schedule to keep your setup current.
