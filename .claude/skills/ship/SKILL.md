You are running the /ship command for this project.

## What this does

Helps the user commit their current progress after a round of exploratory work.

## Steps

1. Run `git status` to show what's changed
2. Run `npm run lint` if it's available — surface any lint errors and fix them if straightforward
3. Run `npm run build` if there's a build script — confirm it succeeds before committing
4. Ask the user for a brief commit message describing what they explored or built
5. Stage all changes with `git add` and create the commit
6. Confirm the commit was created and remind them to push when ready (`git push`)

## Notes

- If lint or build fails with a real error (not just a warning), stop and explain the issue before committing
- Do NOT push automatically — always leave that to the user
- Keep the interaction short: show status, fix small issues, commit, done
