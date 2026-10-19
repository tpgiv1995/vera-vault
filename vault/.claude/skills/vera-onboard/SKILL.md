---
name: vera-onboard
description: Personalize a fresh vault for its owner. Use when the user runs /vera-onboard, says "set me up", "onboard me", "personalize the vault", or when wiki/people/me.md still says "(not set yet)". Asks a few questions, then fills in the owner page, the alias registry and the category vocabulary in CLAUDE.md. Safe to re-run.
---

# /vera-onboard

Fill in who owns this vault and how their work divides up, so ingestion files everything correctly.
Keep it short and friendly: a handful of questions, then make the edits.

## Before you start

Confirm you are in the vault folder (`CLAUDE.md`, `wiki/` and `raw/` exist here). If not, tell the
user to start Claude Code from the vault folder and stop.

## 1. Learn who the owner is

If a Microsoft 365 connector is available, call its `get_me` tool first and use the name, email and
job title it returns as the defaults, so the user only has to confirm.

Then ask (use AskUserQuestion where it helps; free text is fine):

1. **Name, email and role** (confirm the `get_me` values if you have them).
2. **Manager and team**, in a few words.
3. **Main split:** "What is the main way your work divides up? For example clients, offices, lines of
   business, teams or products." Then: "List the main ones, 3 to 8, comma-separated."
4. **Personal notes too?** Whether the vault should also hold personal (non-work) material.

## 2. Make the edits

Change only these regions; do not rewrite whole files.

1. **`wiki/people/me.md`**: set `name`, `email`, `role` and `category` in the frontmatter, set the
   `title` to the owner's full name, set `updated` to today, and fill the Role, Manager and team, and
   What I work on sections with what they told you. Remove the "Run /vera-onboard" line.
2. **`wiki/people/_aliases.md`**: replace the placeholder row with rows mapping the owner's full name,
   first name, email address and email username to `[[me]]`.
3. **`CLAUDE.md`**, between `<!-- CUSTOMIZE:start category -->` and `<!-- CUSTOMIZE:end category -->`
   (keep the markers so a re-run can find the block): name the dimension ("the owner's main split:
   **Offices**"), and rebuild the table as `All`, one row per value they listed (short Title Case keys,
   no spaces), then `Admin` and `Other`. Keep the multi-category and inheritance rules.
4. **`CLAUDE.md` top note**: replace the "First time here?" block with
   `> Personalized for <name> (<role>) on <date>. Re-run /vera-onboard to change it.`
5. **`wiki/overview.md`**: one line under the title naming the owner and their role.
6. **`wiki/log.md`**: add `## [YYYY-MM-DD] onboard | Personalized for <name> (<main split>)` at the top
   of the entries.

Do not invent people, projects or meetings. The vault stays empty until real data arrives.

## 3. Check the data sources

Look at which connector tools exist in this session and report in one short list:

- **Microsoft 365** (email, Teams chats, calendar): tools like `outlook_email_search`,
  `chat_message_search`, `outlook_calendar_search`.
- **Meetings:** Read AI (`list_meetings`) or Zoom (`search_meetings`, `get_recording_resource`).

For anything missing, tell the user exactly where to turn it on: **claude.ai > Settings >
Connectors**, connect it, then start a new Claude Code session in the vault. Everything that is
connected already works; missing sources are simply skipped.

## 4. Wrap up

Summarize what you changed (the files above), then offer the first real run:
*"Want me to pull the last 3 days of your email, Teams, calendar and meetings and build the first
wiki pages? That's `/vera-data-pipeline 3`."*
