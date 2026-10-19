# raw/notes/

Daily notes and manual entries you write yourself.

- **Source:** manual.
- **Naming:** `YYYY-MM-DD-daily.md` for daily notes, or `YYYY-MM-DD-[descriptive-slug].md` for
  standalone notes.

During ingestion, Claude reads each note, extracts action items and decisions, and creates or
updates the relevant wiki pages. If a note is vague or purely informational, it gets flagged in
`wiki/inbox.md` for review rather than forcing a page. Use the daily-note template in
`templates/` to start a new daily entry.
