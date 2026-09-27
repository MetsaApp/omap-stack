# Issue tracker: Linear

Issues and specs for this repo live in Linear: workspace `metsaapp`, team **Engineering** (key `ENG`), project **omap-stack** (`P-ENG-21`). Use the Linear MCP tools (`mcp__linear__*`) for all operations. Every issue you create gets `team: Engineering` and `project: omap-stack`.

## Conventions

- **Create an issue**: `save_issue` with `team`, `project`, `title`, `description` (Markdown with real newlines), and a `milestone` when one fits.
- **Read an issue**: `get_issue` (with `includeRelations: true` for blocking), then `list_comments` for its comments.
- **List issues**: `list_issues` with `project: omap-stack`, filtered by `label`, `state`, `parentId` or `assignee`.
- **Comment on an issue**: `save_comment` with `issueId` and `body`.
- **Apply / remove labels**: `save_issue` with `id` and `addLabels` / `removeLabels`.
- **Close**: `save_comment` with the reason, then `save_issue` with `id` and `state: Done` (or `Canceled` when ruled out).
- **Cross-references**: Linear turns a bare `#NNN` into a link to an unrelated MetsaApp PR, and auto-links `shp.zip`. Refer to issues by identifier (`ENG-230`) or by name with its URL; write a GitHub issue as "issue NNN" with a full URL.

## Pull requests as a triage surface

**PRs as a request surface: no.** PRs live on GitHub (`MetsaApp/omap-stack`); name the Linear identifier in the branch or PR title so Linear links them.

## When a skill says "publish to the issue tracker"

Create a Linear issue in project omap-stack.

## When a skill says "fetch the relevant ticket"

`get_issue` with the identifier, then `list_comments`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **sub-issues** as tickets.

- **Map**: a single issue in project omap-stack labelled `wayfinder:map`, holding the Destination / Notes / Decisions-so-far / Not yet specified / Out of scope body.
- **Child ticket**: a sub-issue of the map (`save_issue` with `parentId: <map identifier>`), labelled `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, assigned to the driving dev.
- **Blocking**: Linear's native relation, visible in the UI. `save_issue` with `id: <child>` and `blockedBy: [<blocker identifiers>]` (append-only; `removeBlockedBy` to drop an edge). A ticket is unblocked when every blocker is Done or Canceled.
- **Frontier query**: `list_issues` with `parentId: <map>`, open states only; for each, `get_issue` with `includeRelations: true` and drop any with an open blocker or an assignee. First by creation order wins.
- **Claim**: `save_issue` with `id` and `assignee: "me"`, the session's first write.
- **Resolve**: `save_comment` with the answer, then `save_issue` with `state: Done`, then append a context pointer (name linked to its URL, plus a one-line gist) to the map's Decisions-so-far with a `save_issue` `patch`.
