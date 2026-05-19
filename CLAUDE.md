# Project

<!-- Replace this section with a description of what you're building -->
A Next.js / React / TypeScript web app.

## Stack

- **Framework:** Next.js (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS (if installed)
- **Package manager:** npm

## Commands

| Command | Purpose |
|---|---|
| `npm run dev` | Start development server |
| `npm run build` | Production build |
| `npm run lint` | Run ESLint |
| `npm test` | Run tests (if configured) |

## Conventions

- TypeScript strict mode — no `any` without a comment explaining why
- Components in `src/components/`, pages in `src/app/`
- Keep components small and focused; extract when a file exceeds ~150 lines
- Prefer server components by default; add `"use client"` only when needed

## Constraints

- Do not push to git without explicit confirmation
- Do not modify `package-lock.json` directly — use `npm install`
- Ask before running `npm install <package>` — confirm the dependency first

## Available Commands

| Command | Purpose |
|---|---|
| `/onboarding` | Re-run setup or refresh this config |
| `/knowledge-curator` | Audit setup against latest Claude Code best practices |
| `/ship` | Lint, build-check, then commit current work |
