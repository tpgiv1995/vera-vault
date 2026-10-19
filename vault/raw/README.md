# raw/ - Source documents

This is the **source layer** of the vault. Every file here is an original input: a meeting
transcript, an email, a calendar event, a chat export, a web clip, a note. These are the ground
truth that the wiki is built from.

**Claude never modifies files in `raw/`.** It only reads them and synthesizes the `wiki/` layer.
If you want to correct something, edit the wiki page, not the raw source.

## Which folder for which medium

| Medium | Goes in |
|---|---|
| Meeting transcripts (from a meeting service) | `meetings/` |
| Meeting transcripts (from a local AI notetaker) | `meeting-assistant/` |
| Older local audio recordings | `audio-transcripts/` |
| Email (inbound + sent) | `emails/` |
| Team/DM chat exports | `chats/` |
| Calendar events | `calendar/` |
| Daily notes and manual entries | `notes/` |
| Web articles and reference clips | `clips/` |

Some folders are **auto-pulled** by the data pipeline (meetings, emails, calendar, chats);
others are **manual** drops (notes, clips, meeting-assistant exports). See each folder's
own README for naming conventions and whether it is automatic or manual.
