---
title: Session Journals
type: hub
domain: meta
created: 2026-10-19
updated: 2026-10-19
sources: []
tags: []
---

# Session Journals

Every Claude Code session writes a journal to `vera/sessions/`: what was done, decisions made, dead
ends, and handoff notes for the next session. The newest journal is the handoff; `/acclimate` reads
it for you at the start of a session.

```query
path:"vera/sessions"
```
