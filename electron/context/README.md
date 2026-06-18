# Agentic Features

A feature-based, semi-automated build workflow for Claude Code. You write feature
specs by hand; agents parse them into tasks, build them in a supervised
multi-agent loop, review the result, and recover from failures with
diagnosis-first discipline. You stay in control between every stage.

Unlike fully-automated pipelines (docs in → app out), this kit keeps the human
checkpoints that matter: you lock the architecture, you review the generated
tasks, you decide what gets fixed.

## Quick start

1. Copy the contents of this repo into your project root.
2. Fill `context/project-overview.md` and `context/architecture.md` (stack, folder structure,
   commands) — agents refuse to work while these are templates.
3. Set the design tokens in `context/ui-context.md` and the API contract in `context/api-context.md`.
4. Copy `context/features-list/_template.md` to `context/features-list/01-<name>.md`, fill it,
   and set `Status: ready` (parse-features refuses `draft` specs).
5. Run the pipeline:

```
/parse-features 01     # spec → context/tasks-list/01-<name>-tasks.md (review it by hand)
/start-build 01        # orchestrated loop: parallel Builders + QA per task
/review 01             # 3-layer verification → context/bugs-list/01-bug-<name>.md
/recover 01            # if issues: diagnose first, then fix / reset / rethink
```

`/resume-build NN` continues an interrupted build. `/review all` is the
project-wide release gate. `/remember save` / `restore` hand off between sessions.

## How it works

```
you write               agent generates          agents build             agent verifies
context/features-list/    →     context/tasks-list/        →     code + tests       →     context/bugs-list/
01-auth.md              01-auth-tasks.md         (Builders + QA loop)     01-bug-auth.md
```

- **Roles:** an Orchestrator schedules; Builders write code (one task each, never
  self-approve); QA verifies against blocking gates (lint, typecheck, native
  build, tests, acceptance criteria, standards).
- **State lives in files**, not chat history: `context/progress-tracker.md` (features /
  tasks / bugs, plus a `## Next` resume hint), per-feature task boards, and
  `context/memory/` (patterns + decisions QA records after each approved task). All of
  it is intentionally committed to git — it is the source of truth for resuming
  sessions.
- **Failure discipline:** 3 attempts max per task, then blocked. One failed
  corrective prompt in a session → `/recover`, which diagnoses the failure mode
  (isolated bug / polluted session / wrong foundation) before touching code.

## Layout

| Path | Purpose |
| --- | --- |
| `CLAUDE.md` / `AGENTS.md` | agent bootstrap + the full operating contract |
| `context/project-overview.md`, `context/architecture.md` | what you're building, on what stack (you own these) |
| `context/project-standards.md`, `context/ui-rules.md`, `context/api-rules.md` | code / UI / API rules |
| `context/ui-context.md`, `context/api-context.md` | design tokens + component registry, API envelope + endpoint registry |
| `context/features-list/` → `context/tasks-list/` → `context/bugs-list/` | the pipeline artifacts |
| `context/progress-tracker.md`, `context/memory/` | build state + cross-session knowledge |
| `.claude/skills/` | the 12 workflow skills (11 slash commands + 1 internal orchestrator) |

## Skills

`parse-features`, `start-build`, `resume-build`, `orchestrate`, `qa`, `review`,
`recover`, `remember`, plus utilities: `code-structure`,
`code-structure-cleanup`, `source-code-context`, `grep-loop-review-workflow`.

Built for Claude Code (skills = slash commands). Distilled from several earlier
agentic-workflow projects; the hard-won rules — scaffold smoke checks, attempt
budgets, read-modify-write file discipline, tokens-only UI, diagnosis-before-fix
— are baked into the skill texts.
