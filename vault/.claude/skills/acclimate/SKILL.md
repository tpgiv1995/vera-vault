---
name: acclimate
description: Get up to speed at the start of a session - recent session journals, what changed in the vault, open items and the owner's open tasks - then brief the owner. Use when the user runs /acclimate, says they're starting a new session, asks you to "get up to speed", "load context", "orient", "catch me up", or asks "where did we leave off". Optional argument narrows it to one topic or project, e.g. /acclimate budget-review.
---

# /acclimate

Build a working picture of where things stand so the next message can be productive. This vault's
CLAUDE.md is already in your context; do not re-read it.

If the current folder does not contain `CLAUDE.md` and `vera/sessions/`, say so in one line: Claude
needs to be started from the vault folder for these skills to work. Then stop.

## 1. Topic (optional)

If the user gave an argument (`$ARGUMENTS`), treat it as a topic or project slug. Otherwise acclimate
on the vault as a whole.

## 2. Recent journals

Journals are `vera/sessions/YYYY-MM-DD-HH-slug.md`, so sorting by filename sorts by date.

- With a topic: find journals whose `project:` frontmatter or filename contains the slug.
- Without one: take the newest journals regardless of project.

Read the 3 most recent in full. For the next 5 back, read only Status, Open Items and Next Session
Should. Go further back only when an open item points there. If there are no journals yet, say so
and continue.

## 3. Current state of the vault (run these in parallel)

- **Last pipeline run:** the newest entries at the top of `wiki/log.md`, and the newest file dates in
  `raw/emails/inbox`, `raw/emails/processed`, `raw/chats`, `raw/meetings` and `raw/calendar`. If the
  newest capture is more than a day old, the owner may want `/vera-data-pipeline`.
- **Waiting to be ingested:** raw files newer than the last ingest entry in `wiki/log.md` (a rough
  count is enough).
- **Needs the owner's judgment:** open items in `wiki/inbox.md`.
- **Open tasks:** search `wiki/` for lines starting `- [ ]` that contain `[[me]]`. Count them, and
  list the ones that are overdue or due in the next 3 days (the 📅 date).
- **Upcoming meetings:** events in `raw/calendar/` for today and tomorrow.
- With a topic: also read the matching project page in `wiki/projects/` and its most recent sources.

## 4. Brief

Give the owner a short briefing in plain prose or a few bullets:

- where things stand, from the newest journal;
- what is open or was promised next;
- tasks overdue or due soon, and today's meetings;
- whether the vault is behind (days since the last pull, files waiting for ingest);
- one or two gotchas from the journals that affect the next piece of work, if any.

Leave out anything that is empty rather than padding it. End by asking what they want to pick up,
or name the obvious next step if a journal states one.
