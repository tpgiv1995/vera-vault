---
name: vera
description: Write or update the session journal for this Claude Code session in vera/sessions/, capturing what was done, decisions and why, dead ends, open items and handoff notes for the next session. Use when the user runs /vera, says "wrap up", "save progress", "journal", "hand off", or when a long session is about to end or run out of context.
---

# /vera

Write or update a session journal capturing context for future Claude Code sessions.

**Target path:** `vera/sessions/` in the vault root (the folder you run `claude` from - the one with
`CLAUDE.md` in it). Write the journal there, not to a `vera/sessions/` folder inside some other project.

File name: `YYYY-MM-DD-HH-slug.md` (named by the session start time + a short descriptor of the
primary work, e.g. `2026-06-26-21-acme-kickoff-ingest.md`).

If a journal already exists for this session at that path, update it. Otherwise create one.

Use the session-journal format documented in `CLAUDE.md` (the "Session Journal" section). Frontmatter:

```yaml
---
title: "Brief description of what was done"
type: session-journal
created: YYYY-MM-DD
project: project-name-or-general
tags: [session-journal, relevant-tags]
---
```

Then these sections:

```
## Status
What state is the work in? Done, in progress, blocked?

## What Was Done
Bullet list of concrete accomplishments, with file paths.

## Decisions Made
What was decided and WHY. The reasoning matters more than the decision.

## What Failed / Dead Ends
What was tried and didn't work. Error messages, wrong approaches, why they failed.

## Open Items
What still needs to be done. File paths, the exact problem.

## Lessons / Gotchas
Hard-won knowledge not obvious from reading the files.

## Next Session Should
Explicit instructions for the next session. What to read, do, and avoid.
```

Rules:
- Be specific. "Fixed the dashboard" is useless. Include file paths and the actual changes.
- Capture the WHY. The files show what was done; the journal captures why and what was tried.
- Write it like a briefing for a smart colleague with zero context.
- Err on the side of too much detail.
- Don't just list files modified - that's a changelog, not a journal.
