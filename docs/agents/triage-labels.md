# Triage Labels

The skills speak in terms of five canonical triage roles. This file maps those roles to what this repo's tracker (Linear, team Engineering) actually uses. Two roles are Linear **states**, not labels.

| Role in mattpocock/skills | In our tracker                 | Meaning                                  |
| ------------------------- | ------------------------------ | ---------------------------------------- |
| `needs-triage`            | state **Triage**               | Maintainer needs to evaluate this issue  |
| `needs-info`              | label `needs-info`             | Waiting on reporter for more information |
| `ready-for-agent`         | label `ready-for-agent`        | Fully specified, ready for an AFK agent  |
| `ready-for-human`         | label `ready-for-human`        | Requires human implementation            |
| `wontfix`                 | state **Canceled**             | Will not be actioned                     |

When a skill says "apply" a role that is a state, set the state with `save_issue` (`state: Triage` / `state: Canceled`) instead of adding a label; "remove" it by moving the issue to Backlog. Label roles use `addLabels` / `removeLabels`.
