---
description: Log a scene to the character tracker — who appeared, what they did, and whether it was a positive or negative scene for each character. Say "update the tracker" or "log that scene".
disable-model-invocation: true
allowed-tools: Read Write Edit Bash
---

You are the `/track` skill. Keep the character tracker up to date after each scene.

## Triggers

- Slash form: `/track`
- Natural language: "update the tracker", "log that scene", "add to the tracker", "track that scene"

## What you do

After the user writes a scene, read it (or use their description), then update `tracker.md` with the characters who appeared, what they did, and the scene's emotional tone for each character.

## Step 1: Load context

Read `tracker.md` if it exists — you'll append to it.
Read character description files to know the full cast.

## Step 2: Get the scene

If the user pointed to a file, read it. If they described the scene in the conversation, use that. If unclear:

> "Which scene should I log — can you point me to the file or describe what happened?"

## Step 3: Update tracker.md

For each character who appeared, update their section:

- Add the scene to their appearances list (use Act + Scene number, or a short title)
- Mark the scene's tone: **Positive** (things went their way) or **Negative** (things went against them)
- One sentence on what happened to them in the scene

### tracker.md format

If `tracker.md` doesn't exist, create it:

```
# Character Tracker

_Last updated: [date]_

## [Character Name]

**Total appearances:** N
**Positive scenes:** N | **Negative scenes:** N

| Scene | Tone | What happened |
|---|---|---|
| Act 1 Scene 1 | Positive | Lands the job she's been chasing |

---
```

Add a `---` separator between characters.

## Step 4: Confirm

One sentence on what was logged:

> "Logged: Sarah and Marcus both appear in Act 2 Scene 3 — positive for Sarah (she gets the information she needed), negative for Marcus (he's caught off-guard)."

## Hard rules

- Never alter the scene file itself.
- Never alter character description files.
- Only update `tracker.md`.

## See also

- `/review` — when you want notes on what you've written
- `/coverage` — when you want a full reader's view
