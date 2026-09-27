# omap-stack: agent instructions

## Agent skills

### Issue tracker

Issues live in Linear: team **Engineering** (`ENG`), project **omap-stack**. See `docs/agents/issue-tracker.md`.

### Triage labels

Triage and Canceled are Linear states; `needs-info`, `ready-for-agent`, `ready-for-human` are labels. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Engineering conventions

- **Adapters are in-tree code.** Each Adapter is a Go package implementing the provider interface. A Provider in `omap.yaml` picks an Adapter by name and passes it the config that Adapter's schema declares. No Provider behaviour is defined in yaml.
- **Schema first.** Every interface a process or person touches has a schema, and code is generated from it: protobuf for the queen-ant protocol (ConnectRPC), JSON Schema for `omap.yaml` and each Adapter's config, sqlc for SQL, OpenAPI for any other HTTP API.
- **Event-sourced by default** where state has a lifecycle worth replaying (who did what, when, and why this row looks like this). Follow `docs/patterns/event-sourcing.md`; apply it per aggregate, not everywhere (`docs/patterns/go-architecture-guidelines.md` says when not to).
- **Postgres is the only broker.** The event log, the task queue and every consumer run on Postgres (LISTEN/NOTIFY plus polling, `FOR UPDATE SKIP LOCKED`). No NATS, JetStream or other message broker.
- **Test throughout.** Unit tests for pure logic; integration tests against real dependencies via testcontainers (PostGIS, Martin, a real queen for ant tests) for everything that touches I/O. Tests own their containers, never a dev database. Every change lands with its tests.

## Working rules

- No em-dashes in anything you write: use colons, semicolons or parentheses.
- No AI attribution in commits or PR bodies.
- Commit identity `Malthe Poulsen <malthe@grundtvigsvej.dk>`, unsigned. Conventional commits (release-please reads them).
- One PR per issue, CI green. Merging needs the human.
