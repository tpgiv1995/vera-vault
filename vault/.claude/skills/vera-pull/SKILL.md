---
name: vera-pull
description: Phase 1 of the data pipeline ONLY - capture ALL raw source data (meetings, emails, calendar, Teams chats) into raw/, with no filtering and no wiki changes. Use when the user says "pull raw data", "run phase 1", "just grab the data", "capture my emails/meetings/chats/calendar", "sync raw", or /vera-pull.
---

# /vera-pull - Phase 1 (capture) only

Run **only Phase 1 (PULL)** of the `vera-data-pipeline`. Capture raw source data into `raw/` and
stop. Do **not** ingest into the wiki and do **not** lint - those are `/vera-ingest` and `/vera-lint`.

## Principle (do not get this wrong)

Phase 1 is a **dumb, comprehensive capture**. Pull **every** available item from every connected
source - every email (inbox **and** sent), every calendar event, every Teams message, every meeting
 - and write it verbatim into `raw/`. **Do NOT skip, filter, sample, or judge importance.** Paginate
every source to exhaustion; never cap page counts. The only thing to avoid is re-capturing the *same*
item twice (use the watermarks). When in doubt, capture it. Deciding what matters is Phase 2's job.

**Two fidelity rules that are easy to get wrong (see §1.3 in `vera-data-pipeline`):**
- **Stamp each file with the item's OWN date, never today.** Email/meeting/chat filenames and their
  `created`/`received`/`sent` fields come from the message's real timestamp. Every file ending up with
  today's date = the capture is worthless.
- **Capture full bodies, not previews.** Email search returns only a truncated preview - call
  `read_resource` per message to get the complete body (forwarded thread, headers, attachments).

## Procedure

Open the `vera-data-pipeline` skill's `SKILL.md` and execute its **`# PHASE 1: PULL`** section
exactly - steps 1.1 (watermarks) through 1.6 (session-journal backfill) - for whatever connectors are
available. Do ONLY that phase; do not proceed into Phase 2 or Phase 3.

**Argument:** `/vera-pull [today | yesterday | YYYY-MM-DD | YYYY-MM-DD YYYY-MM-DD | N]` - same date
semantics as `vera-data-pipeline` (default: watermark-based per source).

**Usage tip:** capture is mechanical - run it on a light model (e.g. `claude --model sonnet`, low
effort). For a large backlog, capture one source or one week at a time to stay under usage limits.

When done, print the Phase 1 summary (counts captured per source) and remind the user that these raw
files become wiki pages only after they run **`/vera-ingest`**.
