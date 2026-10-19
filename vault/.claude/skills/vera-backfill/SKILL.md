---
name: vera-backfill
description: Generate a session journal for every Claude Code session that has happened since the last time this skill was invoked. Use when the user says "backfill the journals", "catch up vera", "journal all recent sessions", "vera-backfill", or otherwise asks to generate session journals in batch for sessions that don't have one yet. Reads transcripts from $HOME/.claude/projects/*/*.jsonl and writes journals to the Obsidian vault's vera/sessions/ folder using the session-journal format documented in the vault's CLAUDE.md.
---

# vera-backfill

Generate session journals (in the format documented in the vault's CLAUDE.md "Session Journal" section) for every Claude Code session on this machine that has happened since this skill was last run. Uses a state marker for incrementality and a manifest so already-journaled sessions are never duplicated.

## Paths

The Obsidian vault root is the directory where you run `claude` (your current working directory, `$PWD`). Journals live under `vera/sessions/` inside that vault.

- **Transcripts root:** `$HOME/.claude/projects/` (each subdir = one project, each `*.jsonl` = one session)
- **Journal output:** `<your vault>/vera/sessions/`
- **State marker:** `$HOME/.claude/state/vera-backfill-last-run.txt` (ISO-8601 UTC string)
- **Manifest:** `$HOME/.claude/state/vera-backfill-manifest.json` (`{session_uuid: {journal_path, project_slug, journaled_at}}`)

## Procedure

### 1. Determine the cutoff

Read the marker file. If it doesn't exist, default the cutoff to **7 days before now**. Print the cutoff to the user so they see the window.

### 2. Find candidate sessions

Glob `$HOME/.claude/projects/*/*.jsonl`. For each file:

- Use the file's mtime as a coarse first filter (mtime > cutoff).
- Then open the JSONL and read the first line's `timestamp` field - that's the **session start time**. Sessions are journaled by start time (matches the `YYYY-MM-DD-HH-slug.md` session-journal naming convention).
- Extract `session_uuid` from the filename (`<uuid>.jsonl`).
- Read the manifest. If `session_uuid` is already in it, skip this session.

There is **no noise filter**: every uncovered session gets a journal, even single-turn ones. The one
exception is this skill's own workers: skip any session whose first user message begins with
"You are writing a session journal for a Claude Code session you did NOT participate in". Otherwise
every backfill run creates new sessions that the next run journals, forever.

**Windows note:** in Claude Code's Bash tool, `$HOME/.claude/projects` resolves to
`C:\Users\<you>\.claude\projects`. Transcripts from every folder Claude Code has been used in live
there, not only this vault's.

### 3. Sort and report

Sort the surviving sessions by start time ascending. Tell the user how many sessions will be journaled and roughly how many tokens that's likely to cost. Wait for confirmation only if the count is **> 25**; otherwise proceed.

### 4. Fan out (throttled parallel, or sequential)

Spawn the per-session workers on **Haiku** - reading a transcript and writing a summary journal is mechanical, and each worker ingests a *full* (often large) transcript, so a cheap model keeps this affordable on a small plan. If your harness has a `Workflow` / parallel sub-agent tool, use **throttled** `parallel()` - chunk into batches (~8-10 concurrent) via a `for` loop of `parallel(chunk)`, NOT one giant `parallel(all)`. Each agent is independent (no shared state), which is exactly what `parallel()` is for. The real limit is the API rate wall: a wide-open fan-out (~16+) tends to **rate-limit hard** with 429s, so stay comfortably under it (~8-10) and back off a notch if you see 429 retries in the logs. Don't carelessly fan out ~30 at once. **If you have no parallel sub-agent tool, just process the sessions one at a time** - slower, but the result is identical. Each session's worker gets a prompt of this shape:

```
You are writing a session journal for a Claude Code session you did NOT participate in.

Read the transcript at: <absolute path to the .jsonl>
Session UUID: <uuid>
Project slug: <slug>
Start time: <ISO timestamp>

Read every line of the JSONL. Each line is one event (user message, assistant message, tool call, tool result). Build a mental model of what the user and the assistant were trying to accomplish, what got done, what failed, and what's still open.

Write a journal file to:
  <your vault>/vera/sessions/YYYY-MM-DD-HH-<slug>.md

Where:
- YYYY-MM-DD-HH comes from the session start time (local time).
- <slug> is a short kebab-case descriptor of the primary work (e.g. "path-fix", "import-cleanup"). Derive from the actual content, NOT the project name.

Frontmatter MUST include:
  ---
  title: "Brief description of what was done"
  type: session-journal
  created: YYYY-MM-DD
  project: <project-slug-or-better-name>
  session_id: <uuid>
  tags: [session-journal, <relevant tags>]
  ---

Then write the standard session-journal sections: ## Status, ## What Was Done, ## Decisions Made, ## What Failed / Dead Ends, ## Open Items, ## Lessons / Gotchas, ## Next Session Should.

Be specific. Include file paths and line numbers where the transcript mentions them. Capture WHY decisions were made, not just what was done. If the session was trivial (one question, one answer), keep the journal short but still complete the sections (use "N/A" where genuinely empty).

Return ONLY the absolute path of the journal file you wrote. No other text.
```

### 5. Update state

After all workers finish:

1. For each success, append `{session_uuid: {journal_path, project_slug, journaled_at: <now>}}` to the manifest JSON.
2. Write the current UTC ISO timestamp to the marker file.
3. Summarize to the user: count journaled, count failed (if any), output directory.

If a worker failed (returned null or threw), do NOT add it to the manifest - it'll get retried on the next run.

## Gotchas

- **Don't trust mtime alone for start time.** A long session's mtime is the last write, not the start. Always read the first JSONL line for the true start time, which is what the filename convention requires.
- **Frontmatter `session_id` is new.** Existing pre-skill journals don't have it. On the first run, the manifest is empty so old sessions in the 7-day window will be re-journaled even if a journal already covered them. If you spot a recent journal in the sessions folder that overlaps the cutoff window, either: (a) extend the manifest by hand for those UUIDs, or (b) accept one round of duplicates and clean up. There is no good auto-dedup without `session_id` in the old files.
- **Project slug encoding.** Project dir names are path-encoded (e.g. `-home-user-dev-my-app`). Decode them to a readable project name (`my-app`) for the frontmatter `project:` field rather than dumping the slug raw.
- **The vault path is the vault root.** Always write journals under `vera/sessions/` inside the Obsidian vault (your current working directory) - never to a `vera/sessions/` folder somewhere else.
- **Empty / corrupt JSONL.** Skip silently and don't update the manifest for that session.
