# Knowledge Base

_Last updated: 2026-05-19_

## Summary
Setup is complete for a Next.js 15 + Tailwind CSS design portfolio. CLAUDE.md, user preferences, and permissions are all configured. Project scaffold still needs to be run.

## Latest findings

### What's working well
- SessionStart hook provides useful git + environment context each session
- Weekly /knowledge-curator schedule is configured to keep best practices current
- Permissions are scoped appropriately for Next.js development (npm, npx, node, git)
- User preferences captured at user level (not project level) as recommended

### Gaps and improvements
- Project hasn't been scaffolded yet — Next.js app needs to be initialized with `npx create-next-app`
- No project-specific skills created yet; consider adding one for common portfolio tasks (e.g. adding a new project case study) once the site structure is established

### Quick wins
- Run `npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"` to initialize the project
- Deploy to Vercel early (free, connects to GitHub) so previews are available as you build

## What changed since last review
First run — no previous baseline.

## Run history
| Date | Trigger | Summary |
|------|---------|---------|
| 2026-05-19 | onboarding | Initial setup for design portfolio (Next.js 15 + Tailwind, Vercel) |
