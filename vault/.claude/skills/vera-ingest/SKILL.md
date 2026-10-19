---
name: vera-ingest
description: Phase 2 of the data pipeline ONLY - process everything already captured in raw/ into the wiki (decide what's important, build people/projects/concepts/meetings/decisions pages, extract tasks, update the index). Does NOT pull from connectors and does NOT lint. Use when the user says "ingest raw", "run phase 2", "process my raw files", "build the wiki from raw", or /vera-ingest.
---

# /vera-ingest - Phase 2 (raw → wiki) only

Run **only Phase 2 (INGEST)** of the `vera-data-pipeline`. Process the files already in `raw/` into
wiki pages. Do **not** pull from any connector (that's `/vera-pull`) and do **not** lint (that's
`/vera-lint`).

## What this phase does

This is where you **decide what matters** and build the knowledge base from the raw capture:

- Go through every file under `raw/` that isn't already listed in some wiki page's `sources:`
  (meetings, notes, emails, calendar, chats, meeting-assistant, audio-transcripts, clips).
- **Emails:** classify each **important / context / discard**. Important → a `wiki/emails/` page;
  context → merge into the relevant project/people pages; discard → move to `raw/emails/discarded/`
  (kept, not deleted). Move processed files to `raw/emails/processed/`.
- **Meetings / chats / notes:** extract decisions and action items; create or update the right wiki
  pages - add new people to `wiki/people/`, update project pages, create concept/decision pages.
- Extract the owner's tasks in Obsidian Tasks format (only the owner's get checkboxes).
- Classify every page with `domain` (+ the work slicing field). Resolve names through
  `wiki/people/_aliases.md`. Record each ingested raw path in the page's `sources:`. Update
  `wiki/index.md` and append to `wiki/log.md`.

## Procedure

Open the `vera-data-pipeline` skill's `SKILL.md` and execute its **`# PHASE 2: INGEST`** section
exactly (including the page-classification step). Do ONLY that phase.

**Usage tip:** this is the synthesis-heavy phase - give it more reasoning (higher effort), but it
doesn't need Opus; Sonnet at high effort is the sweet spot. Report files processed, pages
created/updated, emails important/context/discarded, and tasks extracted.
