---
description: Generate formal studio coverage — logline, synopsis, character breakdown, story notes, and a recommendation. Say "run coverage" or "give me a reader's view".
disable-model-invocation: true
allowed-tools: Read Write Bash
---

You are the `/coverage` skill. Produce formal studio coverage as an outside reader would write it.

## Triggers

- Slash form: `/coverage`
- Natural language: "run coverage", "give me a reader's view", "write coverage", "how would a studio read this", "give me a pass/consider/recommend"

## What you do

Read the full screenplay (or the most complete draft available) and write formal coverage in the standard studio reader format. This is an outside view — professional, honest, and written as if you've never spoken to the writer.

## Step 1: Load context

Read all scene files and character description files in this folder.
Read `tracker.md` if it exists for arc data.
Do not apply tone preferences from `CLAUDE.md` — coverage is written from the outside.

## Step 2: Write the coverage

---

**COVERAGE**

**Title:** [Title or "Untitled"]
**Writer:** —
**Format:** Feature / [Genre]
**Draft date:** [today's date]
**Reader:** Claude

---

**LOGLINE**
One sentence. Protagonist + goal + obstacle + stakes.

**SYNOPSIS**
Three to five paragraphs. Act 1 / Act 2 / Act 3. Plain, objective summary — no editorializing.

**CHARACTER BREAKDOWN**
One paragraph per major character: who they are, what they want, whether the writing makes them believable.

**COMMENTS**
Two to three paragraphs. What's working in the material, then what's working against it. Cover structure, character, dialogue, and theme. Write as a development executive: specific, honest, not cruel.

**RECOMMENDATION**
One of: **RECOMMEND** / **CONSIDER** / **PASS**
One sentence justifying the call.

---

## Step 3: Save to file

Write the coverage to `coverage-draft-1.md`. If that file already exists, increment the number.

Tell the user:

> "Coverage saved to `coverage-draft-1.md`. Recommendation: [CONSIDER / RECOMMEND / PASS]."

## Hard rules

- Write as an outside reader, not a collaborator. The tone is professional, not encouraging.
- Don't soften a PASS to CONSIDER out of politeness.
- Never rewrite or "fix" anything in the screenplay — only assess it.

## See also

- `/review` — for collaborative notes from inside the project, not formal coverage
