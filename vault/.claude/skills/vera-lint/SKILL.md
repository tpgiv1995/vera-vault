---
name: vera-lint
description: Phase 3 of the data pipeline ONLY - wiki health check and cross-linking for the pages the latest ingest touched (links, task fields, broken sources), with safe fixes. With --full, audits the whole wiki (orphans, stale projects, missing concept pages, contradictions). Does NOT pull or ingest. Use when the user says "lint the vault", "run phase 3", "health check", "clean up the wiki", or /vera-lint.
---

# /vera-lint: Phase 3 (health check) only

Run **only Phase 3 (LINT)** of `vera-data-pipeline`. Do not pull from connectors (`/vera-pull`) and do
not ingest raw files (`/vera-ingest`).

## Scope

- **`/vera-lint`** (default): the batch set is the pages created or updated by the most recent ingest.
  Find them from the newest ingest entry in `wiki/log.md`; if that entry does not list pages, use the
  wiki pages whose `updated` date is on or after that entry's date. Write only to those pages.
- **`/vera-lint --full`**: the whole wiki, including the full-audit checks in §3.4.

## Procedure

Open `.claude/skills/vera-data-pipeline/SKILL.md` and execute its **`# PHASE 3: LINT`** section
exactly, with the scope above. Anything that needs the owner's judgment goes to `wiki/inbox.md`.
Append a summary to `wiki/log.md` and report what changed.

**Usage tip:** lint reads the compact wiki, not the raw corpus, so high effort is worthwhile and
cheap. Sonnet at high effort is a good default.
