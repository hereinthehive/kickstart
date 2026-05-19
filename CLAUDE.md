# Design Portfolio

A personal portfolio site showcasing design work. Built with Next.js and Tailwind CSS, deployed on Vercel.

## Project overview

This is a static-style portfolio site. Pages include a home/hero, project gallery, individual project case studies, and a contact section. The goal is clean, image-forward design with fast load times.

## Tech stack

- **Next.js 15** (App Router)
- **Tailwind CSS** for styling
- **Vercel** for deployment

## Getting started

If the project hasn't been scaffolded yet:

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
```

## Commands

```bash
npm run dev      # start local dev server (http://localhost:3000)
npm run build    # production build
npm run lint     # run eslint
```

## Code conventions

- Use the App Router (`app/` directory), not the Pages Router
- Components go in `src/components/`
- Keep components small and focused — one responsibility per file
- Use Tailwind utility classes directly; avoid custom CSS unless necessary
- Images go in `public/` or use Next.js `<Image>` component for optimization
- TypeScript is preferred but not required for simple additions

## Working style notes

- The user is a designer, not a developer — explain what you're doing briefly in plain language
- Prioritize visual quality and clean layout over engineering complexity
- When adding a new page or section, describe what it will look like before writing code
- Prefer simple, readable solutions over clever ones

## Available commands

| Command | Purpose |
|---|---|
| `/knowledge-curator` | Review the current setup against latest Claude Code best practices |
