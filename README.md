# vera-vault

An Obsidian vault that Claude Code maintains for you: a second brain built from your own email,
Teams chats, calendar, meeting transcripts, notes and Claude sessions.

```
 email, Teams, calendar, meetings ──pull──▶  raw/        the complete record, never edited
 your notes, clips, transcripts ───drop──▶     │
                                               ▼
                                         ingest + lint   (Claude, following vault/CLAUDE.md)
                                               │
                                               ▼
                                             wiki/       people · projects · meetings · decisions
                                               │         your tasks · dense [[links]] back to raw/
                                               ▼
                              you ──ask──▶  "what did we decide about X?"  "what's due this week?"

 every Claude session ──/vera──▶ vera/sessions/   journals, read back by /acclimate next time
```

## Set it up

Install [Obsidian](https://obsidian.md), open Claude Code (the desktop app's Code tab or `claude` in a
terminal), and paste:

```text
Set up my Obsidian second brain from https://github.com/tpgiv1995/vera-vault. Read START-HERE.md in that repo and follow Part 2 from Step 1.
```

Claude downloads the kit, copies the vault into place, personalizes it, walks you through connecting
Microsoft 365 and your meeting tool in claude.ai, and runs the first pull. The full runbook it follows
is [START-HERE.md](START-HERE.md).

## What's in it

```
vault/                    the vault you open in Obsidian
├── CLAUDE.md             the rulebook Claude reads in every session: layout, page format, tasks, workflows
├── .claude/skills/       the commands (they load when Claude Code starts in the vault folder)
├── raw/                  meetings · emails (inbox, sent, processed, discarded) · calendar · chats · notes · clips
├── wiki/                 index · overview · log · inbox · tasks · sessions · people/ · projects/ · meetings/ ...
├── notes/                your loose notes
├── vera/sessions/        session journals
└── output/               generated files
```

| Command | What it does |
|---|---|
| `/acclimate` | Start of a session: reads recent journals and vault state, briefs you on what is open and due |
| `/vera-data-pipeline` | Daily: pull email, Teams, calendar and meetings, ingest into the wiki, tidy links and tasks, journal the run |
| `/vera-pull` · `/vera-ingest` · `/vera-lint` | The three pipeline phases on their own (`/vera-lint --full` audits everything) |
| `/vera` | End of a session: write the session journal |
| `/vera-backfill` | Journal past Claude Code sessions that do not have one yet |
| `/vera-onboard` | Personalize the vault (first run, re-runnable) |

No Obsidian plugins are required. Data sources are claude.ai connectors on your own account; any that
are missing are skipped.

## Credits

Built by Pat Gordon. MIT licensed, see [LICENSE](LICENSE).
