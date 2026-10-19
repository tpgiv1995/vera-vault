---
name: vera-data-pipeline
description: Unified data pipeline for your Obsidian second brain. Pulls raw data (meetings, email, calendar, chat) from the MCP connectors you have connected in claude.ai, ingests raw files into wiki pages, and runs a lint/health check. Trigger with "run the pipeline", "sync my vault", or /vera-data-pipeline.
---

# /vera-data-pipeline

Three-phase, fully automated pipeline:

1. **PULL** - capture raw data (meetings, email, calendar, chat) into `raw/`.
2. **INGEST** - turn new raw files into linked wiki pages.
3. **LINT** - health check and cross-linking audit, with fixes.

Runs all phases sequentially without pausing for approval.

## Raw capture must be complete, not partial

`raw/` does two jobs: it feeds the ingest pass, and it stands alone as a complete, searchable record
of your work that you can dig through later. A capture that silently stops early is worse than an
explicit failure, because it looks complete. One real run marked 7 of 10 Teams days "partial" after
giving up at 3 rate-limit errors; re-pulled without a cap, some of those days had 2 to 3 times as many
messages.

- **Keep retrying through rate limits (429 / throttled).** There is no retry cap for throttling. Wait
  and retry the same request (see §1.4 step 3). Wall-clock time is cheap; a hole in the record is not.
- `partial: true` is only for errors that retrying cannot fix: a permission error, a connector that is
  not available, or a server error that repeats after a few tries.
- Never skip items to dodge a throttle, and never cap page counts.

## Usage settings (keep it affordable)

- **Capture (Phase 1) is mechanical.** Run it on Sonnet at low or medium effort (start the session
  with `claude --model sonnet`). Email capture needs care with full bodies and real dates, so do not
  drop below Sonnet for email.
- **Ingest and lint (Phases 2 and 3) need judgment.** Sonnet at high effort is the sweet spot. Opus
  is rarely needed.
- **The first run is the expensive one.** A months-long backlog costs real usage once; daily runs
  after that are small. For a big backlog, pull one source at a time, or a week at a time, and spread
  it across usage windows.
- **Don't re-read finished work.** The watermarks (§1.1) and the `sources:` check (§2.1) keep each
  run to genuinely new material.
- **Never drop the meeting transcript.** Use the connector's summary and next steps to fill the
  summary and action items, but still capture the transcript itself.

## Context

The vault root is the directory you run `claude` from. Throughout this skill, set:

```bash
V="$(pwd)"          # vault root = current working directory
```

All `raw/`, `wiki/`, and `vera/` paths are relative to `$V`.

### Owner identity

The vault owner's canonical self-link is `[[me]]`. Every "my tasks" query filters on `[[me]]`,
and `wiki/people/_aliases.md` maps the owner's real name to `[[me]]`. Only the owner's tasks get
checkboxes; everyone else's action items are plain bullets.

### Timezone

Wherever a source returns UTC timestamps (most MCP connectors do), **convert them to the user's
local timezone** before deriving dates, filenames, and display times. Do not hardcode an offset - 
use the machine's local zone (`date` honors `$TZ` / the system zone).

### MCP connectors (Phase 1 sources)

Phase 1 pulls from whatever MCP connectors are present in **your own claude.ai account**. These are
tied to *your* login and **must be connected in claude.ai first** (Settings → Connectors). If a
connector is not present, **skip that source silently** - the pipeline still runs for the rest.

| Source | Connector | Tools |
|---|---|---|
| Meetings (default) | **Zoom** | `mcp__claude_ai_Zoom_for_Claude__search_meetings`, `mcp__claude_ai_Zoom_for_Claude__get_recording_resource` |
| Meetings (optional) | Read AI - *only if connected* | `list_meetings` / `get_meeting_by_id` |
| Email + Calendar | **Microsoft 365** | `outlook_email_search`, `outlook_calendar_search`, `read_resource` |
| Chat | **Teams (via Microsoft 365)** | `chat_message_search` |

**Zoom is the default meeting source** and requires the user's own Zoom connector in claude.ai
(Settings → Connectors). Read AI is optional - pull from it only if it is connected. Teams (via
the Microsoft 365 connector) is the chat source; Slack is **not** part of the default set (note it
as optional only if the user happens to have it connected).

Tool names differ between connectors and providers - discover the actual tool names available in
your session rather than assuming the ones above. The *logic* below is provider-agnostic.

**Check whose data you are about to pull, before pulling anything.** Call the Microsoft 365
connector's `get_me` (or the equivalent "who am I" tool) and confirm the name and email match the
vault owner in `wiki/people/me.md`. If they do not match, stop and tell the user: a connector signed
in as someone else fills the vault with the wrong person's mail.

### Local meeting transcripts (no pull needed)

If you run a local AI notetaker, point it to drop transcripts into `raw/meeting-assistant/`
(filename pattern `Title [8hex-id].md`, frontmatter `type: meeting-transcript`,
`source: Meeting Assistant`). Nothing is fetched for these in Phase 1, but they DO count as
already-on-disk coverage for the dedup in §1.2 and are ingested in Phase 2.

## Argument parsing

`/vera-data-pipeline [arg]`. The date argument applies to ALL sources in Phase 1.

- **(no arg)** - watermark-based per source (see §1.1).
- **`today`** - today's local date for all sources.
- **`yesterday`** - yesterday's local date.
- **single date `YYYY-MM-DD`** - that date only.
- **two dates `YYYY-MM-DD YYYY-MM-DD`** - inclusive range, oldest first.
- **plain integer `N`** - the last `N` days through yesterday.

---

# PHASE 1: PULL (Raw Data Capture)

> **PRINCIPLE - do not get this wrong.** Phase 1 is a DUMB, COMPREHENSIVE capture. Pull **every**
> available item from every connected source - every email (inbox **and** sent), every calendar
> event, every Teams message, every meeting - and write it verbatim into `raw/`. **Do NOT skip,
> filter, sample, or judge importance here.** Deciding what matters, extracting context, and building
> the wiki (people / projects / index) is Phase 2's entire job. When in doubt, capture it. The only
> thing Phase 1 avoids is re-capturing the *same* item twice (idempotency via watermarks) - it never
> drops anything by content. Paginate every source to exhaustion; never cap page counts.

Run the steps sequentially - they share connectors that are often rate-limited. The session-journal
backfill (§1.6) is the exception and may run in parallel.

If a connector for a given source is not available in the session, note it and move on.

## 1.1: Find watermarks

For the no-arg case, find the most recent captured date per source so you only pull what is new.
The watermark only *bounds* the pull window; the real duplicate guard is the §1.2 identity-key dedup.

```bash
V="$(pwd)"

# Meetings watermark - most recent YYYY-MM-DD-*.md anywhere under raw/meetings/
mw=$(find "$V/raw/meetings" -name '*.md' -printf '%f\n' 2>/dev/null \
  | grep -oE '^[0-9]{4}-[0-9]{2}-[0-9]{2}' | sort | tail -1)
echo "meetings: ${mw:-none}"

# Email watermark - most recent YYYY-MM-DD-* across processed/ + sent/ + inbox/
ew=$(find "$V/raw/emails/processed" "$V/raw/emails/sent" "$V/raw/emails/inbox" \
  -maxdepth 1 -name '*.md' -printf '%f\n' 2>/dev/null \
  | grep -oE '^[0-9]{4}-[0-9]{2}-[0-9]{2}' | sort | tail -1)
echo "email: ${ew:-none}"

# Chat watermark - most recent complete (not partial:true) YYYY-MM-DD-*-chats.md
cw=none
while IFS= read -r f; do
  d=$(basename "$f" | grep -oE '^[0-9]{4}-[0-9]{2}-[0-9]{2}')
  [ -z "$d" ] && continue
  grep -qE '^partial:[[:space:]]*true\b' "$f" && continue
  cw="$d"; break
done < <(find "$V/raw/chats" -maxdepth 1 -name '*-chats.md' 2>/dev/null | sort -r)
echo "chat: $cw"
```

If the user passed explicit dates, skip watermark detection and use those for all sources.

## 1.2: Sync meetings (deduplicated across sources)

Capture every **unique** meeting and write each one **once**. The same physical meeting can land in
more than one source (a Zoom call a meeting bot also joined, a calendar meeting you also recorded
locally), so dedupe by an identity key and pull the transcript from the highest-priority source that
has it.

### Meeting identity key

For any meeting (a fresh candidate OR an existing file), compute:

```
key = <local-date YYYY-MM-DD> + "|" + <title-slug>
```

- **local-date** - convert the meeting's start time to local time and take the calendar date.
- **title-slug** - lowercase, strip punctuation, collapse whitespace to `-`, drop recurring noise
  (`weekly`, `bi-weekly`, embedded dates, `- copy`). Stable, not exact.

Two candidates are the **same meeting** when their slugs match (fuzzy: ignore minor word
differences) AND their start times are within **±20 minutes**. Same-day, same-slug meetings at
clearly different times (e.g. a standup held twice) are distinct - disambiguate by appending
`-HHMM` (local start) to the slug.

### Source priority (transcript selection on overlap)

**Zoom (default) → Read AI (optional) → Teams → local (notetaker / audio).** Zoom is the default
meeting source; Read AI is consulted only if its connector is present. When a meeting's key is
already covered by a higher-priority source, do NOT fetch or write it from a lower one. Local files
are already on disk, so a meeting your notetaker already captured is skipped by default - its
coverage counts.

### Step A - Build the on-disk coverage set

Compute identity keys for every meeting already captured, so nothing is re-pulled:

- `raw/meetings/**` (recursive - includes any `zoom/` or `teams/` subfolders)
- `raw/meeting-assistant/` (derive date from frontmatter `created`/`session_id`, slug from title)
- `raw/audio-transcripts/` (filename `YYYY-MM-DD HHmm - Title.md`)

Call this set `COVERED`.

### Step B - List candidates (cheap, metadata only)

Using whichever meeting connector you have (Zoom is the default; Read AI only if connected):

- **Zoom (default)**: `mcp__claude_ai_Zoom_for_Claude__search_meetings` over the date window (past
  meetings only), paginate via the page token. Each result → `{key, source: Zoom, meeting_id,
  topic, start_time}`. Metadata only here - defer the recording fetch to Step D.
- **Read AI (optional, only if connected)**: list meetings in the window (metadata only, no
  transcript expand; paginate via cursor). Each result → `{key, source, id, title, start, platform,
  participants, report_url}`.

Defer fetching summaries/transcripts until Step D - only the winners get expanded.

### Step C - Merge & dedupe

Group all candidates by key. For each group, the winning source is the higher priority. Drop any
group whose key is already in `COVERED`. The survivors are the new unique meetings.

### Step D - Write the new meetings

For each surviving group, write **one** file to `raw/meetings/YYYY-MM-DD-<title-slug>.md`. Fetch the
transcript/summary from the winning source:

- **Zoom (default)**: call `mcp__claude_ai_Zoom_for_Claude__get_recording_resource` for the meeting
  with `types: ["transcript", "summary", "nextStep"]`. Write the raw file with: the meeting **topic**;
  the **participants / speakers** (the distinct `display_name`s from the transcript turns); the
  **summary** → ✨ SUMMARY; Zoom's **next-steps** → ✅ ACTION ITEMS; and the **transcript text** →
  🗣 TRANSCRIPT, rendered as timeline turns, one per line in the form `display_name: text`. If a
  requested type is missing for that recording, just omit that section. (Requires the user's own Zoom
  connector in claude.ai.)
- **Read AI (optional, only if connected)**: expand `summary`, `action_items`, `chapter_summaries`
  (add `transcript` only if verbatim text is actually needed - it is the heaviest). Map summary →
  ✨ SUMMARY, action items → ✅ ACTION ITEMS, chapters → 💬 CHAPTERS & TOPICS. Use the report URL as
  the canonical link.

Add each written key to `COVERED`.

**Raw meeting frontmatter (all sources):** The filename `YYYY-MM-DD`, the `created` field, and the
`Event time` line ALL come from the meeting's OWN start time (Zoom `schedule_start_time` / Read AI
`start` / Teams event `start`), converted to local time - **NEVER the run/ingestion date.** Same rule
as the email step (§1.3): if every meeting file gets today's date, the capture is worthless.
```
---
title: YYYY-MM-DD - Meeting Title
type: meeting-raw
created: <YYYY-MM-DD from the meeting's start time - NOT today>
source: Read AI | Zoom | Teams | Meeting Assistant
meeting_key: YYYY-MM-DD|title-slug
alt_sources: [other sources that also had this meeting, if any]
---

Meeting: Meeting Title
[<canonical source URL>]

Event time: YYYY-MM-DD H:MM PM - H:MM PM (local)
Platform: <platform>
Participants: <comma-separated participant names>


✨ SUMMARY

<summary text, wrapped ~80 chars>


✅ ACTION ITEMS

 * <action item 1>
 * <action item 2>


💬 CHAPTERS & TOPICS

<CHAPTER TITLE>
<chapter description>


🗣 TRANSCRIPT

[Full transcript at: <url> - fetch verbatim only if needed; can exceed 200K chars]
```

Skip embedding huge verbatim transcripts - link the source URL. If a meeting looks sensitive
(a 1:1, a personal/mobile recording), add a `⚠️ Sensitive` header and apply the depth heuristic in
Phase 2.

### Step E - Calendar-meeting transcripts (only for meetings nothing else covers)

If your email/calendar connector exposes meeting transcripts (e.g. Teams via M365), this is usually
the most expensive source (two `read_resource` calls per meeting under a low RPM cap), so only chase
transcripts for meetings **not already in `COVERED`**:

1. Search the calendar for the window, past events only, oldest first; paginate.
2. Filter to plausible recorded meetings: skip all-day events, skip solo blocks (no other
   attendees), skip events the owner declined or marked free. Compute each event's key; **skip any
   key already in `COVERED`** (this is where the higher-priority-first ordering pays off - those
   meetings cost zero calls here).
3. For each remaining event: read the event resource, look for a transcript URL field. If absent,
   the meeting was not transcribed → skip. If present, read that resource → transcript segments.
4. Write to `raw/meetings/teams/YYYY-MM-DD-<title-slug>.md` with the frontmatter above
   (`source: Teams`, transcript in 🗣 TRANSCRIPT; a raw transcript has no structured summary - leave
   ✨/✅ minimal or omit). Add the key to `COVERED`.
5. **Rate-limit (429) handling** - use the mandatory foreground-sleep pattern from §1.4. After 3
   consecutive 429s, stop the transcript sweep for this run and note it in the Phase 1 summary; the
   rest is already captured.

## 1.3: Sync email (inbox + sent)

### Inbox

Search the email connector's Inbox: `afterDateTime = <emailWatermark>`, oldest first, page size ~25.
**Paginate until a page returns no more results - capture the ENTIRE window; do NOT cap pages** (a
page cap silently drops most of a backlog).

**Phase 1 captures EVERYTHING - do NOT skip, filter, or judge importance here.** That is Phase 2's
job (it classifies each email important/context/discard and moves junk to `raw/emails/discarded/`, so
nothing is lost). **Write EVERY email** - including automated, no-reply, mass-distribution, and
notification mail.

#### ⚠️ Two things the search result does NOT give you - get these right or the capture is worthless

1. **The real date.** The filename prefix, the `created` field, and the `received`/`sent` field all
   come from the **message's own timestamp** (`receivedDateTime` for inbox, `sentDateTime` for sent)
   - **NEVER today's date, NEVER the ingestion date.** If every file you write ends up stamped with
   today's date, you have destroyed the one thing that makes the archive searchable. The timestamp is
   on every search result - use it verbatim, converted to local time.
2. **The full body.** `outlook_email_search` returns only metadata + a *truncated* preview. The real
   message - the forwarded/quoted thread, the inline `From/Sent/To/Cc/Subject` headers, the
   `Attachments:` line - is **only** available via `read_resource`. You MUST call `read_resource` with
   each message's URI (`mail:///messages/{messageId}`, returned by the search) and write its full
   `body.content`. Writing the preview snippet and moving on drops most of the email.

#### Per-email procedure

For each message the search returns:
1. **Read the full message:** call `read_resource` with its `mail:///messages/{messageId}` URI →
   complete `body.content` (HTML), full `from` / `toRecipients` / `ccRecipients`, and the
   authoritative `receivedDateTime` / `sentDateTime`.
2. **Derive the date** from that timestamp, converted to local time → `YYYY-MM-DD-HHmm`. This is the
   filename prefix AND the `created` / `received` (or `sent`) values. Never substitute the current date.
3. **HTML-strip the body** using the same rules as §1.4 step 6 (`<p>`→text, `<br>`→newline,
   `<a>`→`[text](url)`, decode entities, strip remaining tags). **Preserve** the forwarded/quoted
   thread and any attachments - that downstream context is the whole point of capturing it. Render
   forwarded headers as `**Forwarded - From:** Name <email>` / `**Sent:** Weekday, Month DD, YYYY
   H:MM AM/PM` / `**To:**` / `**Cc:**` / `**Subject:**`, and list attachments as
   `Attachments: a.xlsx, b.xlsx`.
4. **Write** to `raw/emails/inbox/YYYY-MM-DD-HHmm_<subject-slug>.md` (subject slug lowercase-kebab,
   max 50 chars, punctuation stripped):
```yaml
---
title: <subject>
type: email-raw
created: <YYYY-MM-DD from the message timestamp - NOT today>
received: <full ISO-8601 receivedDateTime>
from: <sender address>
to: <all To recipients, comma-separated>
cc: <all Cc recipients, comma-separated; omit the line if none>
source: <connector>
direction: inbox
tags: [email]
---
```
Body: the **full HTML-stripped message body**, including the forwarded thread and attachments line.

**Rate limits:** the per-email `read_resource` call roughly doubles the request count, so 429s are
likely on a large backlog. Use the mandatory foreground-sleep pattern from §1.4 step 3 and **retry
the same message** - never skip emails to dodge the throttle (skipping is the exact data loss this is
fixing).

### Sent

Same as inbox but: search the **Sent** folder; the timestamp field is `sentDateTime` (use it for the
filename/`created`/`sent`); no sender filtering (all are from the owner); `direction: sent` in
frontmatter; write to `raw/emails/sent/`.

## 1.4: Sync chat (Teams via Microsoft 365)

The default chat source is **Teams**, pulled through the Microsoft 365 connector's
`chat_message_search` tool. Slack is **not** part of the default set - only fetch from it if the
user has the Slack connector connected, in which case apply the same per-day capture logic below.

### Date range resolution

If explicit dates were passed, use those. Otherwise auto-detect: take the most recent *complete*
(not `partial: true`) `YYYY-MM-DD-*-chats.md` in `raw/chats/`, then target every day from that date
+ 1 through **yesterday**. If caught up, skip. If there are no existing files, capture just
yesterday. If the auto range exceeds 14 days, print the list and ask the user to confirm.

### Capture procedure (per target DATE)

1. **Local-day window**: build a UTC window that spans the local calendar day DATE (start of DATE
   in local time → start of DATE+1 in local time, expressed in UTC for the connector).
2. **Fetch + paginate**: search messages for that window, `limit=25`, `offset=0`; keep paging while
   the response reports more results.
3. **Rate-limit (429 / throttled): wait in the foreground, then retry. No retry cap.** On a throttle
   error, call Bash with
   ```
   command: "for i in $(seq 1 6); do sleep 10; echo \"waited ${i}0s\"; done"
   timeout: 70000
   ```
   Do **NOT** set `run_in_background: true`; a backgrounded sleep makes the agent look idle. After it
   returns, retry the same offset. If the throttle keeps coming back, run the wait again (two or three
   in a row is fine) and keep retrying until the page succeeds. Never mark a day `partial: true`
   because of rate limits; `partial: true` is only for non-retriable errors (see "Raw capture must be
   complete" above).
4. **Group + sort**: group messages by conversation/`chatId`; sort each thread ascending by time.
5. **Thread heading**: collect unique sender display names. For a 2-person DM →
   `## Name1 <-> Name2 (1:1)`; otherwise `## Name1, Name2, Name3 (group)` (up to ~5 names). Next
   line: the conversation id in italics for traceability.
6. **HTML stripping** for each body:
   - `<p>X</p>` → X (blank line after); `<br>` variants → newline
   - `<a href='URL'>text</a>` → `[text](URL)`
   - `<strong>`/`<b>` → `**X**`; `<em>`/`<i>` → `*X*`; `<code>` → `` `X` ``
   - `<ul><li>X</li>…</ul>` → `- X` lines
   - Entities: `&nbsp;`→space, `&amp;`→`&`, `&lt;`→`<`, `&gt;`→`>`, `&quot;`→`"`, `&#39;`→`'`
   - Strip all remaining tags. If the cleaned body is empty, skip the message.
7. **Time format**: convert each timestamp to local time, render as `H:MM AM/PM` (no seconds, no TZ
   label).
8. **Reply context - SKIP BY DEFAULT.** Per-message reply-parent lookups are rarely worth the API
   cost in 1:1 / small-group corpora, where people type sequential replies instead of using the
   formal Reply action. Capture the search results, build the file, mark `partial: false`. The
   inline `(from X)` markers that quote-replies already embed survive the HTML strip and preserve the
   reply context that exists. Only do selective reply lookups if a caller explicitly asks and a
   small yield sample is non-zero.
9. **Per-message format**:
   ```
   **{Sender Display Name}** ({H:MM AM/PM}):
   ```
   then the cleaned body, then a blank line. If a body looks truncated, append
   ` *[truncated by API]*`.
10. **Write the file** at `raw/chats/DATE-chats.md`:
    ```yaml
    ---
    title: "Chats - {DayName}, {Month} {DD} {YYYY}"
    type: chats-raw
    date: DATE
    source: <connector>
    captured_at: <current ISO-8601 local timestamp>
    window_utc: {afterDateTime} to {beforeDateTime}
    message_count: N
    thread_count: M
    partial: false
    tags: [chat]
    ---
    ```
    Body: an H1 title, a one-line summary, then per-thread sections separated by `---`.
11. **Empty days**: write a stub with `message_count: 0`, `thread_count: 0`, body
    `No chat activity on this date.` so it is not re-fetched.

## 1.5: Sync calendar

Refresh `raw/calendar/` from the calendar connector. Calendar events are future-dated and get
updated/cancelled, so this step is **window-based, not watermark-based** (do not add it to the §1.1
filename watermark - the latest filename is a future event, not a capture marker).

- **No-arg / default**: search `afterDateTime = today − 7 days`, `beforeDateTime = today + 30 days`,
  oldest first, page size ~25, **paginate until no more results - capture EVERY event, do NOT cap pages.**
- **Explicit date(s)**: use the passed date(s) as the window.

For each event, write/overwrite `raw/calendar/YYYY-MM-DD-HHmm_<subject-slug>.md` (date/time from the
event start, local; subject slug lowercase-kebab, max 50 chars; strip any `Canceled-` prefix from the
slug but keep it in the title). One file per event slot, overwrite on update. Keep cancellations - 
they enrich meeting/people pages.

```yaml
---
title: <subject>
type: calendar-event-raw
created: YYYY-MM-DD
start: <ISO start>
end: <ISO end>
organizer: <organizer address>
attendees: <comma-separated addresses>
status: <busy|free|tentative>
cancelled: <true|false>
event_id: <id>
source: <connector>
tags: [calendar]
---
```
Body: location (with any join URL) and the event description if present.

Calendar events do NOT get individual wiki pages - Phase 2 uses them to enrich meeting/people pages
and feed digests.

## 1.6: Backfill session journals

Capture every Claude Code session on this machine since the last backfill as a session journal.
This is the [[vera-backfill]] procedure folded into the pipeline. It MAY run in parallel - chunk it
into batches of ~8-10 concurrent agents (back off if you see 429s; do not fan out ~30 at once).

> **Cost note (small plans):** each agent reads a *full* session transcript (often very large), so
> this is a real usage sink - **run the backfill agents on Haiku** (it's transcript→summary, not
> fidelity-critical). On a fresh setup it's also nearly empty (you have few past sessions), so it's
> fine to **skip §1.6 entirely** until you've actually used Claude Code for a while - session journals
> are a convenience (context handoff), not source data. Run it on its own (`/vera-backfill`) when you
> want it, rather than letting it ride along with every heavy pull.

State files:
- Marker: `$HOME/.claude/state/vera-backfill-last-run.txt` (ISO-8601 UTC)
- Manifest: `$HOME/.claude/state/vera-backfill-manifest.json`
  (`{session_uuid: {journal_path, project_slug, journaled_at}}`)

Steps:
1. **Cutoff** - read the marker; if missing, default to 7 days ago. Print the window.
2. **Candidates** - glob `$HOME/.claude/projects/*/*.jsonl`. Coarse-filter by mtime > cutoff, read
   the first JSONL line's `timestamp` for the true session start, extract `session_uuid` from the
   filename, and skip any already in the manifest. Every uncovered session gets a journal. (After a
   machine migration, mtimes may be bulk-reset to the copy time - trust the manifest over mtime.)
3. **Sort** ascending by start time; report count and date span.
4. **Process** one agent per session (spawn the agents on **Haiku** - cheap transcript summarization)
   using the vera-backfill per-session prompt: read the full
   transcript → write `vera/sessions/YYYY-MM-DD-HH-<slug>.md` with the standard journal frontmatter
   (`title`, `type: session-journal`, `created`, `project`, `session_id`, `tags`) and sections
   (Status, What Was Done, Decisions Made, What Failed / Dead Ends, Open Items, Lessons / Gotchas,
   Next Session Should). Decode path-encoded project slugs to readable names for `project:`.
5. **Update state** - append each success to the manifest, then write the current UTC ISO timestamp
   to the marker. Do not record failures; they retry next run.

## Phase 1 Summary

```
Phase 1 (Pull) complete:
- Meetings: X new unique (by source) | or "none new since <date>"
- Inbox emails: X written, Y skipped
- Sent emails: X written
- Chat: X day(s) captured (DATE..DATE) | or "caught up"
- Calendar: X events refreshed (window DATE..DATE)
- Session journals: X backfilled (DATE..DATE) | or "none since <date>"
```

If a step fails (connector unavailable, permission error, rate limit after retries), log the error
inline and continue. Do not abort the pipeline on a partial failure.

---

# PHASE 2: INGEST (Raw → Wiki)

Process all un-ingested raw files into wiki pages. Fully automated.

## 2.1: Identify un-ingested files

Scan these directories for files not yet listed in any `wiki/` page's `sources:` frontmatter array:
`raw/meetings/`, `raw/notes/`, `raw/emails/inbox/`, `raw/emails/sent/`, `raw/calendar/`,
`raw/audio-transcripts/`, `raw/meeting-assistant/`, `raw/chats/`, `raw/clips/`, `notes/`.

## 2.2: Load alias registry

Read `wiki/people/_aliases.md`. Resolve ALL person names through it before writing any wiki content.
The owner's real name resolves to `[[me]]`.

## 2.2.5: Classify every page - domain / category / status (REQUIRED)

Every wiki page created or updated in this phase MUST carry structured frontmatter. This is what
keeps the vault sliceable - do not skip it.

- **`domain`** (required, all pages): `work` | `personal` | `meta`.
- **`category`** (required on `domain: work` pages): your custom slicing dimension (e.g. a team,
  client, or line of business). Single value, or a YAML list for multi-category pages. Keep the
  vocabulary small and consistent.
  - People: seed `category` from your directory/roster if you have one.
  - Meetings / emails / notes: inherit `domain` and `category` from the project page(s) the content
    links to (default `domain: work`, no `category` if it links no project).
- **`status`** (projects only, optional): `active | paused | shipped | idea`.
- **Tags** - controlled theme vocabulary ONLY:
  `ai-llm, automation, reporting, security, compliance, training, vendor, rollout, integration,
  prototype, recurring`. NEVER tag with a project, person, type, category, or domain name - use the
  `[[wikilink]]` or the field. No applicable theme → `tags: []`.

## 2.3: Process each source type

### Meetings (`raw/meetings/`, incl. `zoom/` + `teams/` subfolders)
- **One wiki page per unique meeting, not per raw file.** A meeting may have raw files in several
  places. Group them by the §1.2 identity key (`meeting_key` when present; otherwise compute
  date+slug, ±20 min start). Create/update a single `wiki/meetings/` page per group.
- When a group spans sources, prefer the richest transcript (Read AI → Zoom → Teams → local) for the
  body, but list **every** contributing raw file in `sources:` and note the alternates.
- Extract action items as tasks (only the owner's get checkboxes; others are plain bullets).
- Cross-link with calendar events when both exist for the same meeting.
- Apply the transcript-depth heuristic: full read for sensitive/contentious content, structured
  sections only for routine meetings.

### Emails (`raw/emails/inbox/`, `raw/emails/sent/`)
Classify each email:
- **important** - decisions, approvals, substantive project updates, action items. Gets a full wiki
  page in `wiki/emails/` with original content preserved. Move the raw file to
  `raw/emails/processed/`.
- **context** - routine info referencing a tracked project. No standalone page; merge into existing
  project/people pages. Move to `raw/emails/processed/`.
- **discard** - automated notifications, mass announcements, no content. Move to
  `raw/emails/discarded/`.

Email wiki page format:
```yaml
---
title: "Email subject line"
type: email
domain: work | personal
category: <category if domain:work>
created: YYYY-MM-DD
updated: YYYY-MM-DD
from: "[[sender-wikilink]]"
to: "[[recipient1]], [[recipient2]]"
sources: [raw/emails/processed/original-filename.md]
tags: []
---
```
Sections: Summary (only when needed, see below), Related (wikilinks), Action Items (if any),
Original Email (full content blockquoted).

**Write a Summary only for long emails** (roughly over 2,000 characters) or tangled forwarded
threads. For a typical short email a summary just repeats the original sitting right below it, so
leave the section out. Never paste a cut-off slice of the body as the summary.

### Chat (`raw/chats/`)
Extract decisions, action items, and substantive project context. Distribute content across existing
project/people pages. Do NOT create per-chat wiki pages. Add the chat file path to the `sources:`
frontmatter of every page it informs.

### Calendar (`raw/calendar/`)
Do NOT create individual wiki pages. Use to enrich meeting pages (attendees, transcript links by
date+title match), update people pages (meeting frequency), and feed daily/weekly digests.

### Audio transcripts (`raw/audio-transcripts/`)
Treat like meetings but with sensitivity awareness - these often capture casual or sensitive
thinking-out-loud. Apply the hybrid depth heuristic. Never quote sensitive content verbatim;
paraphrase neutrally and flag unclear material in `wiki/inbox.md`.

### Local notetaker transcripts (`raw/meeting-assistant/`)
Local AI-notetaker output (filename `Title [8hex-id].md`; frontmatter `type: meeting-transcript`,
`source: Meeting Assistant`, with `session_id`, `speakers`, `duration`). Treat like
meetings/audio-transcripts: create/update a `wiki/meetings/` page, cross-link a calendar event on
date+title match, extract the owner's action items. **Apply the same cross-source grouping as the
Meetings section** - if a remote raw file exists for the same meeting (matching identity key), do NOT
create a second page; merge into the one page, preferring the richer remote transcript and listing
the notetaker file in `sources:`. Same sensitivity heuristic - these capture whatever was on the mic,
including personal sessions. Do NOT create wiki pages for purely personal sessions; optionally note
them in `wiki/inbox.md`. Speaker labels are generic ("Speaker 1"); attribute names only when context
is unambiguous, otherwise keep them generic.

### Notes (`notes/`)
If actionable (action items, decisions, project-relevant content), process fully. If vague or purely
informational, add to `wiki/inbox.md` for human review.

### Clips (`raw/clips/`)
Match to existing concept/project pages and add as a source. If no match, log to `wiki/inbox.md`
under "## Unmatched Clips".

## 2.4: Task extraction and reconciliation

- Extract action items from all sources in Obsidian Tasks format, placed on the relevant wiki page.
- Task format: `- [ ] Task description - [[me]] 📅 YYYY-MM-DD 🔼`
- **Only the owner's tasks get checkboxes**; everyone else's action items are plain bullets, e.g.
  `- Send the report - [[sam-rivera]]`.
- Task descriptions should wikilink the relevant project when applicable.
- Default to medium priority (🔼) unless urgency is clear.
- Default due date: 3 business days from the source date if none is mentioned.
- No clear owner → assign to `[[me]]`.
- Do not duplicate the same task across multiple pages - put it on the single most appropriate page.
- **Reconciliation**: for every email (especially sent) and meeting, check related project pages'
  open tasks. If a source demonstrates a task was completed, mark it done (`[x]`) with the source's
  date as the completion date.

## 2.5: Source tracking

Every ingested file's exact path (e.g. `raw/meetings/2026-06-15-team-sync.md`) MUST appear in the
wiki page's `sources:` frontmatter array. This powers `wiki/inbox.md`'s un-ingested detection.

## 2.6: Update indexes

- Add any new pages to `wiki/index.md`.
- Append a timestamped entry to `wiki/log.md`.
- Log uncertain items to `wiki/inbox.md`.

## Phase 2 Summary

```
Phase 2 (Ingest) complete:
- New wiki pages: X (list names)
- Updated wiki pages: X (list names)
- Emails classified: X important, X context, X discarded
- Tasks created: X | Tasks completed: X
- Items flagged for review: X (in wiki/inbox.md)
```

---

# PHASE 3: LINT (Health Check + Fix)

Run all checks and fix issues automatically. Log changes to `wiki/log.md`.

## 3.1: Scope: THIS run's batch only

Phase 3 tidies what THIS run just ingested, nothing more. Its scope is exactly the set of wiki pages
Phase 2 of this run created or updated (the "batch set", the page list from the Phase 2 summary).

- WRITE only to pages in the batch set. Never rewrite a page this run did not already touch.
- No vault-wide sweeps and no "while I'm here" fixes to unrelated pages. The failure this prevents:
  a routine run rewriting half the vault because the cross-link audit walked every page.
- You MAY read other pages and the alias registry to decide how to fix a batch page.
- A full-vault health audit is a separate, deliberate action (`/vera-lint --full`), never part of
  the routine run.

## 3.2: Cross-linking audit (batch set only)

- For each batch page, find mentions in ITS OWN text of entities that already have a wiki page and
  convert them to `[[wikilinks]]`.
- Links go one way, from batch pages out to existing pages. Do not edit older pages to add backlinks;
  they pick up links the next time a batch touches them.
- Create a stub page for a dangling wikilink only when the target is clearly a real, recurring
  entity; otherwise flag it in `wiki/inbox.md`.
- Count the links added.

## 3.3: Health checks (batch set only)

Fix, on batch pages only:
- unlinked person names (resolve through `wiki/people/_aliases.md`; the owner is `[[me]]`);
- missing task fields (owner, due date, priority);
- broken `sources:` paths.

Flag, never rewrite, as one-liners in `wiki/inbox.md`:
- a batch page that contradicts existing content;
- an open task on a batch page that is already past due.

## 3.4: Full audit (only with `/vera-lint --full`)

When the user explicitly asks for a full audit, run the checks above across the whole wiki, plus:
orphan pages (no inbound links), stale active projects (`updated` more than 30 days ago), concepts
mentioned on several pages but lacking a page of their own, and contradictions between pages. Fix
what is mechanical; flag anything needing judgment in `wiki/inbox.md`.

## Phase 3 Summary

```
Phase 3 (Lint) complete:
- Wikilinks added: X
- Task formatting fixes: X
- Orphan pages found: X
- Issues flagged for review: X (in wiki/inbox.md)
```

---

# PHASE 4: SESSION JOURNAL (close out the run)

The §1.6 backfill journals *past* Claude Code sessions; this phase journals **this run itself**, so
every pipeline execution leaves a handoff for the next session (distinct from `wiki/log.md`, which is
the per-change audit log). Do this even if earlier phases were partial - the failures are the most
useful thing to record.

Write (or update, if one already exists for this session) a journal at
`vera/sessions/YYYY-MM-DD-HH-<slug>.md` using the **`/vera`** session-journal format documented in the
"Session Journal" section of `CLAUDE.md` (frontmatter `type: session-journal`; sections Status,
What Was Done, Decisions Made, What Failed / Dead Ends, Open Items, Lessons / Gotchas, Next Session
Should). Keep it tight - for a routine run, capture:
- **What Was Done** - the phase counts (emails/meetings/chats/calendar captured; pages created/updated;
  tasks; links/fixes).
- **What Failed / Dead Ends** - any source skipped, rate-limited (429s), partial captures, or
  `partial: true` days; anything the run could not complete.
- **Open Items / Next Session Should** - what to re-run (e.g. "Teams 2026-06-20 hit 429s, re-pull"),
  any `wiki/inbox.md` items needing human judgment.

This is cheap - run it on the same model the rest of the run used; do not escalate to Opus for it.

---

# FINAL REPORT

After all three phases, output one consolidated summary:

```
vera-data-pipeline complete.

Pull:
- Meetings: ...
- Inbox emails: ...
- Sent emails: ...
- Chat: ...
- Calendar: ...
- Session journals: ...

Ingest:
- New pages: ...
- Updated pages: ...
- Tasks: ...

Lint:
- Links added: ...
- Fixes applied: ...
- Review items: ...

Journal: vera/sessions/<file written this run>
```

## Error handling

- Partial failures in any phase do not abort the pipeline. Log the error, continue with the rest.
- If an entire phase fails (e.g. no connector available), skip it and note it in the final report.
- All changes to `wiki/` are logged to `wiki/log.md` with timestamps.
