# Progress Tracker

Engine-maintained. Single source of truth for build state across three layers:
features → tasks → bugs. Every skill that changes state updates this file before
finishing. Sessions resume from this file, not from chat history.

---

## Next

> The single next command to run. Every skill that changes state updates this
> line as its last action — sessions resume from here.

- Fill `context/project-overview.md` and `context/architecture.md`, then write
  `context/features-list/01-<name>.md` and run `/parse-features 01`.

---

## Layer 1 — Features

| Feature | Title | Status | Depends on | Tasks file | Updated |
| ------- | ----- | ------ | ---------- | ---------- | ------- |
| _(none yet — first row added by /parse-features)_ |

Feature statuses:
`spec-draft` → `tasks-ready` (/parse-features) → `building` (/start-build) →
`built` (orchestrate, all tasks done) → `review-clean` or `review-found-issues`
(/review) → `complete` (developer sign-off).

## Layer 2 — Tasks

One row per feature with a generated task file. Counts mirror the Progress table
inside each `context/tasks-list/NN-*-tasks.md`.

| Feature | total | done | in-review | in-progress | needs-fix | blocked | pending |
| ------- | ----- | ---- | --------- | ----------- | --------- | ------- | ------- |
| _(none yet)_ |

## Layer 3 — Bugs

One row per review report in `context/bugs-list/`. `Ignored` counts issues the developer
marked `Status: ignored` directly in the bug file (intentional behavior).

| Feature | Report | Open | Diagnosed | Fixed | Ignored | Last review |
| ------- | ------ | ---- | --------- | ----- | ------- | ----------- |
| _(none yet)_ |

---

## Session notes

Append-only, newest first. One dated line per significant event
(build started/resumed/finished, review run, recovery action, scope change).

- _(none yet)_

## Open questions

Unresolved ambiguities any agent has surfaced. Cleared by the developer.

- _(none)_
