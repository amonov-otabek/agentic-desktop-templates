# Agentic Features — Build Kit

Feature-based, semi-automated build workflow. The developer writes feature specs by
hand; agents parse them into tasks, build them in a supervised loop, review the
result, and recover from failures with diagnosis-first discipline. The developer
stays in control between every stage.

## Session start (mandatory reading order)

1. `CLAUDE.md` — this file
2. `AGENTS.md` — the full agent contract (roles, state model, file formats)
3. `context/progress-tracker.md` — current feature / task / bug state
4. `context/project-overview.md` and `context/architecture.md` — what we're building and how
5. `context/memory/session.md` — if it exists, last session's handoff (`/remember restore`);
   `context/memory/patterns.md` + `context/memory/decisions.md` — knowledge from approved tasks

Read context files on demand per task type: `context/ui-rules.md` + `context/ui-context.md` for UI
work, `context/api-rules.md` + `context/api-context.md` for backend work, `context/project-standards.md`
for all code.

## The pipeline

```
developer writes        agent generates           agents build            agent verifies
context/features-list/      →   context/tasks-list/           →   (orchestrated loop) →   context/bugs-list/
01-auth.md              01-auth-tasks.md          code + tests            01-bug-auth.md
   /parse-features 01      /start-build 01           /review 01           /recover 01 (if needed)
```

| Command | What it does |
| --- | --- |
| `/parse-features NN` | Feature spec → per-feature task file in `context/tasks-list/` |
| `/start-build NN` | Orchestrator loop: parallel Builders + QA until done/blocked |
| `/resume-build NN` | Re-enter the loop from current task states (never resets) |
| `/review NN` | 3-layer feature verification → report in `context/bugs-list/` |
| `/recover NN` | Diagnose failure mode first, then targeted fix / hard reset / rethink |
| `/remember save\|restore` | Session handoff memory |

Utility skills (invoke when relevant): `qa` (single-task gate check),
`code-structure`, `code-structure-cleanup`, `source-code-context`,
`grep-loop-review-workflow`.

## Hard rules

- No stage auto-advances into the next: parse, build, review, and recover are each
  started by the developer. Agents finish their stage and stop.
- `context/progress-tracker.md` must match reality after every state change — the next
  session resumes from files, not from chat history.
- Manual edits by the developer are a feature, not an error. If files contradict
  tracked state, surface the drift and ask — never silently overwrite.
- Fill `context/project-overview.md` and `context/architecture.md` before the first
  `/parse-features` run. Tasks never guess the stack.
- If the same problem persists after one corrective prompt — stop immediately
  and run `/recover`. Patching a failed fix with another fix is how sessions
  spiral (failure mode 2); diagnosis is cheaper than a third attempt.
