---
title: Person Alias Registry
type: hub
domain: meta
created: 2026-10-19
updated: 2026-10-19
sources: []
tags: []
---

# Person Alias Registry

Claude reads this during ingestion and lint so every mention of a person resolves to one page.

## How to use

- An alias (case-insensitive) maps to one canonical `[[wikilink]]`.
- Include full names, unique first names, nicknames, email usernames and alternate spellings.
- Mark a first name **AMBIGUOUS** when two people share it; those need a last name to resolve.
- The owner always maps to `[[me]]`. `/vera-onboard` fills in the owner's rows.

## Alias Table

| Alias | Canonical Link |
| ----- | -------------- |
| (owner's full name) | [[me]] |
