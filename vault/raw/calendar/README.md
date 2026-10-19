# raw/calendar/

Calendar events pulled from your calendar. One file per event, overwritten on updates.

- **Source:** auto-pulled by the data pipeline.
- **Naming:** `YYYY-MM-DD-HHmm_Event-Subject.md`
- **Frontmatter:** includes start/end/organizer/status/response.

Calendar events do **not** get their own wiki pages. Instead Claude uses them to enrich meeting
pages (pre-populate attendees, scheduled times, location), update people pages (meeting
frequency), and feed daily/weekly digests. When both a calendar event and a meeting transcript
exist for the same meeting (match on date + title), they are cross-linked.
