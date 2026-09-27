# omap-stack: agent instructions

## Agent skills

### Issue tracker

Issues live in Linear: team **Engineering** (`ENG`), project **omap-stack**. See `docs/agents/issue-tracker.md`.

### Triage labels

Triage and Canceled are Linear states; `needs-info`, `ready-for-agent`, `ready-for-human` are labels. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Working rules

- No em-dashes in anything you write: use colons, semicolons or parentheses.
- No AI attribution in commits or PR bodies.
- Commit identity `Malthe Poulsen <malthe@grundtvigsvej.dk>`, unsigned. Conventional commits (release-please reads them).
- One PR per issue, CI green. Merging needs the human.
