# Start here

This repo sets up an Obsidian vault that Claude Code keeps up to date for you. It pulls your email,
Teams chats, calendar and meeting transcripts into the vault, turns them into a linked wiki of
people, projects, meetings and decisions, tracks your tasks, and journals every Claude session so
nothing gets lost between sittings.

## Part 1: for you (two minutes)

You need **Obsidian** (https://obsidian.md) and **Claude Code**: either the Code tab in the Claude
desktop app, or `claude` in a terminal. Open Claude Code and paste this:

```text
Set up my Obsidian second brain from https://github.com/tpgiv1995/vera-vault. Read START-HERE.md in that repo and follow Part 2 from Step 1.
```

Claude does the rest. It stops to ask you a few questions, to have you click a couple of things in
Obsidian and claude.ai, and once to have you start a fresh Claude session in your new vault folder.
Answer in plain English; "explain that more simply" always works.

---

## Part 2: for Claude (the setup runbook)

You are setting this vault up for someone who may not be technical. Do every step you can yourself.
For the few things only they can click, give a short, exact click path and wait for them to say done.
Keep messages short, and say in one line what each step is for.

Steps 1 to 5 run in whatever session they started. Step 6 moves them into a session in the vault
folder, and Steps 7 to 10 run there.

### Step 1. Download the kit

The repo is public, so no sign-in is needed. On Windows (PowerShell):

```powershell
$kit = Join-Path $env:TEMP "vera-vault-kit"
Remove-Item $kit -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Path $kit | Out-Null
Invoke-WebRequest "https://github.com/tpgiv1995/vera-vault/archive/refs/heads/main.zip" -OutFile "$kit\kit.zip"
Expand-Archive "$kit\kit.zip" -DestinationPath $kit
Get-ChildItem -Force "$kit\vera-vault-main\vault"
```

On Mac or Linux, `git clone https://github.com/tpgiv1995/vera-vault` into a temp folder. The template
vault is the `vault` folder inside the kit.

### Step 2. Choose where the vault lives

Find out whether they already made a vault. Obsidian lists every vault it knows in
`%APPDATA%\obsidian\obsidian.json` on Windows (`~/Library/Application Support/obsidian/obsidian.json`
on Mac). Read it and look inside any vault it lists. Then ask one question:

- **No vault yet, or an empty one:** use a new folder. Suggest `Documents\Obsidian\Second Brain`.
- **A vault with notes in it:** offer to set up inside it. Their files stay where they are and nothing
  is overwritten. Their old notes can be ingested later if they move them into `notes/` (ask first).

### Step 3. Copy the vault into place

Copy the template's contents into the chosen folder **without overwriting anything that exists**:

```powershell
$dest = "<the folder from Step 2>"
robocopy "$kit\vera-vault-main\vault" $dest /E /XC /XN /XO /NFL /NDL
# robocopy exit codes 0 to 7 mean success; 8 or higher is a real error
```

On Mac or Linux: `rsync -a --ignore-existing <kit>/vault/ "<dest>/"`.

Confirm these exist: `<dest>\CLAUDE.md`, `<dest>\raw\emails\inbox`, `<dest>\wiki\index.md`,
`<dest>\.claude\skills\acclimate\SKILL.md`. If the folder already had its own `CLAUDE.md`, the copy
kept theirs: show them the difference and ask which to keep.

### Step 4. Open it in Obsidian (they click)

For a new folder, tell them:

1. Open Obsidian.
2. Click the vault switcher at the bottom left, then **Open folder as vault**.
3. Pick the folder from Step 2.

They should see `raw`, `wiki`, `notes`, `vera` and `output`. No plugins or settings are needed.
Wait for them to confirm.

### Step 5. Connect the data sources (they click)

Check which connector tools this session has:

- **Microsoft 365** (email, Teams chats, calendar): tools named like `outlook_email_search`,
  `chat_message_search`, `outlook_calendar_search`, `read_resource`, `get_me`.
- **Meetings:** Read AI (`list_meetings`, `get_meeting_by_id`) or Zoom (`search_meetings`,
  `get_recording_resource`), whichever they use.

For each one missing, tell them: go to **claude.ai > Settings > Connectors**, find it, click
**Connect**, and sign in with their work account. If it says an administrator must approve it, that is
a request for their IT team; carry on without it. Connectors belong to their claude.ai account, so a
newly connected one shows up in the next session (Step 6).

### Step 6. Move into the vault folder (they click)

The vault's commands (`/acclimate`, `/vera-data-pipeline` and the rest) load when a Claude Code
session **starts** in the vault folder. They were copied in after this session started, so a new
session is needed even if this one is already in `<dest>`. Tell them:

1. Start a new Claude Code session in `<dest>`. In the desktop app's Code tab, choose that folder for
   the new session. In a terminal: `cd "<dest>"`, then `claude`.
2. Paste this in the new session:

```text
Continue my vault setup from Step 7 of START-HERE.md in https://github.com/tpgiv1995/vera-vault.
```

Then stop. Everything from Step 7 on happens in the new session.

### Step 7. Personalize

Run `/vera-onboard`. It asks who they are and how their work divides up, then fills in
`wiki\people\me.md`, the alias registry and the category list in `CLAUDE.md`, and reports which
connectors are live.

Then, if Microsoft 365 is connected, call `get_me` and confirm the name and email match
`wiki\people\me.md`. A connector signed in as someone else would fill the vault with the wrong mail.

### Step 8. First run: pull and build the wiki

Run `/vera-data-pipeline 3` (the last 3 days). It pulls email, Teams, calendar and meetings into
`raw/`, builds wiki pages, cleans up links and tasks on those pages, and writes a session journal.

Check the capture before trusting it:

- Open 3 files in `raw\emails\inbox` and 1 in `raw\emails\sent`. Each must hold the **full** email body
  (not a one-line preview), and its filename must carry the email's **own** date, not today's.
- Open one `raw\chats\YYYY-MM-DD-chats.md` and check the messages look complete.

If anything is wrong, fix it and re-pull before going further. Then point them at two or three of the
new wiki pages in Obsidian: a person, a meeting, and `wiki\tasks.md`.

**More history (optional).** Offer to backfill further back, for example 4 weeks. Do it a week at a
time (`/vera-pull 2026-09-21 2026-09-27`, then the next week), then one `/vera-ingest` and one
`/vera-lint` at the end. The first backfill is the expensive part of the setup; daily runs after it
are small. Use Sonnet for this, not Opus.

**Past Claude sessions (optional).** If they have used Claude Code before, `/vera-backfill` writes a
journal for each past session on this machine.

### Step 9. Teach the daily rhythm

Tell them, briefly:

- **Always start Claude Code in the vault folder.** The commands only exist there.
- **Start of a session:** `/acclimate`. Claude reads the latest journals and tells them where things
  stand, what is open and what is due.
- **Once a day:** `/vera-data-pipeline`. It pulls everything new since the last run.
- **End of a session:** `/vera`, or just "wrap up". Claude writes the journal the next session reads.
- **Anytime:** plain-English questions ("what did we decide about X", "what's on my plate this
  week"). Drop files into `notes` and say "ingest my notes".

### Step 10 (optional). Run the pipeline every morning automatically

Ask if they want it. It needs the `claude` command (check with `claude --version` in PowerShell; the
desktop app alone does not provide it). If it is missing, install it for the terminal with
`irm https://claude.ai/install.ps1 | iex` in PowerShell (details at https://code.claude.com/docs),
then have them open a new PowerShell window and run `claude` once to sign in.

Then register a scheduled task (no admin rights needed). It runs at 7:30 AM while they are signed in
and writes a log to `output\pipeline-last-run.log`:

```powershell
$vault  = "<dest>"
$claude = (Get-Command claude -ErrorAction Stop).Source
$cmd    = "Set-Location -LiteralPath '$vault'; & '$claude' -p 'Run the vera-data-pipeline skill with no argument.' --permission-mode bypassPermissions *> '$vault\output\pipeline-last-run.log'"
$action  = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -WindowStyle Hidden -Command `"$cmd`""
$trigger = New-ScheduledTaskTrigger -Daily -At 7:30am
Register-ScheduledTask -TaskName "Second Brain daily pipeline" -Action $action -Trigger $trigger -Description "Pulls email, Teams, calendar and meetings into the Obsidian vault and updates the wiki"
Start-ScheduledTask -TaskName "Second Brain daily pipeline"
```

Keep the prompt on one line with no double quotes in it. Wait for the test run to finish (a run can
take 10 to 20 minutes), then confirm new files in `raw/` and a new journal in `vera/sessions/`. If the
log says bypass mode is disabled by an administrator, replace `--permission-mode bypassPermissions`
with `--permission-mode acceptEdits --allowedTools "mcp__*" "Bash" "Read" "Write" "Edit" "Glob" "Grep"`,
re-register, and test again.

### Done

Tell them in one or two lines: where the vault is, which sources are connected, how much the first run
pulled, and that every session now starts with `/acclimate`.

---

## If something goes wrong

- **`/acclimate` and the other commands are not recognized:** Claude Code was not started in the vault
  folder. Start a new session there.
- **A connector's tools are missing:** it is not connected in claude.ai, or the session started before
  it was connected. Connect it, then start a new session.
- **Emails have one-line bodies or today's date:** the capture skipped the full-message read. Delete
  those raw files and re-pull that day; the pipeline's email section spells out both rules.
- **Teams days look short:** rate limiting. The pipeline keeps retrying by design; re-run the pull for
  those dates.
- **A page looks wrong:** fix the wiki page, never the `raw/` file.
