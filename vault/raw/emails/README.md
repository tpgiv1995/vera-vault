# raw/emails/

Email captured from your mailbox. Four subfolders track each email's lifecycle:

| Folder | Contents |
|---|---|
| `inbox/` | Unprocessed inbound email. Frontmatter includes from/to/created. |
| `sent/` | Unprocessed sent email. Frontmatter includes `direction: sent`. |
| `processed/` | Email already ingested into the wiki (important or context). Both inbound and sent end up here after processing. |
| `discarded/` | Junk/noise email, kept for audit but excluded from processing. |

- **Source:** auto-pulled by the data pipeline (inbox + sent); processed/discarded are filled by Claude during ingestion.
- **Naming:** `YYYY-MM-DD-subject-slug.md`

During ingestion each email is classified **important** (gets a full page in `wiki/emails/`),
**context** (merged into existing project/people pages), or **discard** (moved to `discarded/`).
After processing, important/context emails move to `processed/`. Do not edit these by hand.
