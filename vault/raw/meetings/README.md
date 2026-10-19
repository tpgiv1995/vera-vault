# raw/meetings/

Meeting transcripts and summaries pulled from a meeting service (e.g. a recording/notetaker
connector).

- **Source:** auto-pulled by the data pipeline.
- **Naming:** `YYYY-MM-DD-Meeting-Title.md`
- **Frontmatter:** typically includes the meeting date, attendees, and a transcript body.

During ingestion, Claude creates or updates a page in `wiki/meetings/`, extracts action items
(only the owner's tasks get checkboxes), and cross-links the matching calendar event when one
exists. Do not edit these files by hand - fix the wiki page instead.
