# AGENTS.md — Agent Contract

The full operating contract for every agent working in this project. `CLAUDE.md`
is the summary; this file is the law. When they disagree, this file wins.

## 1. Directory layout

```
project-root/
├── CLAUDE.md                       ← bootstrap + reading order (root)
├── AGENTS.md                       ← this contract (root)
├── .claude/skills/                 ← the workflow skills, read-only (root)
└── context/                        ← everything else lives here
    ├── project-overview.md         ← what we're building (developer-owned)
    ├── architecture.md             ← stack + folder structure + boundaries (HARD CONTRACT)
    ├── project-standards.md        ← code standards for all code
    ├── ui-rules.md / ui-context.md     ← UI rules + design system (inert for backend-only)
    ├── api-rules.md / api-context.md   ← backend constraints + API surface/registry
    ├── progress-tracker.md         ← 3-layer state: features / tasks / bugs (engine-owned)
    ├── README.md                   ← human-facing overview
    ├── features-list/              ← NN-<slug>.md feature specs (developer-owned)
    ├── tasks-list/                 ← NN-<slug>-tasks.md task boards (engine-owned)
    ├── bugs-list/                  ← NN-bug-<slug>.md review reports (engine-owned)
    └── memory/                     ← patterns.md, decisions.md, session.md (cross-session)
```

Application code lives wherever `context/architecture.md` says — agents never invent
folder structure.

## 2. File formats

### context/features-list/NN-<slug>.md (written by the developer)

```markdown
# Feature NN — <Title>

- Feature ID: FEAT-NN
- Status: draft | ready   (spec-file gate only — the lifecycle status lives in
  context/progress-tracker.md Layer 1; /parse-features refuses `draft` specs)
- Depends on: FEAT-MM | none

## Goal
One or two sentences: the user-facing outcome.

## Scope
- Concrete bullet per capability (endpoints, screens, behaviors).
- What is explicitly OUT of scope, if ambiguous.

## Acceptance Criteria
- [ ] Verifiable statements — each becomes a QA gate.

## Notes (optional)
DB tables, API endpoints, UI screens, design references, library choices.

## Tasks (optional)
If present, /parse-features uses these verbatim instead of auto-decomposing.
```

### context/tasks-list/NN-<slug>-tasks.md (written only by /parse-features)

One file per feature. Progress table at top, then one block per task:
`Status`, `Depends on`, `Layer`, `Files`, `Attempts`/`Max attempts`,
**What to build**, **Acceptance criteria** checkboxes, **QA notes**.
Exact format lives in `.claude/skills/parse-features/SKILL.md`.

### context/bugs-list/NN-bug-<slug>.md (written only by /review and /recover)

Dated review sections with per-issue blocks: `BUG-NN-X`, severity
(critical/important/minor), layer (1/2/3), status (open/diagnosed/fixed/ignored),
location, expected vs. actual, suspected failure mode. Exact format lives in
`.claude/skills/review/SKILL.md`.

## 3. Roles (build loop)

| Role | Count | Writes code | May set task Status |
| --- | --- | --- | --- |
| Orchestrator | 1 | never | `in-progress`, `blocked` |
| Builder | 1 per independent task | yes | `in-review`; `blocked` (with a specific reason) |
| QA | per in-review task | never (notes only) | `done`, `needs-fix` |

- A Builder never approves its own work; only QA sets `done`.
- One Builder per file: tasks whose `Files:` overlap never run concurrently.
- Builders touch only the files in their task block.

## 4. Task status machine

```
pending → in-progress → in-review → done
              ↑              ↓
              └─────── needs-fix
in-progress → blocked        (budget exhausted or missing input)
pending → obsolete           (scope removed via /parse-features re-run)
```

`Attempts` increments every time a Builder picks the task up. At `Max attempts`
(default 3) the task goes `blocked` — the answer to three failures is `/review` +
`/recover`, not a fourth identical attempt.

## 5. Write-ownership (single-writer principle)

| File | Writer |
| --- | --- |
| `context/features-list/*` | developer only (agents propose edits, never apply silently) |
| `context/tasks-list/*` structure | `/parse-features` only |
| `context/tasks-list/*` Status / Attempts / checkboxes / QA notes | build loop roles per §3 |
| `context/bugs-list/*` | `/review` (create/append), `/recover` (status updates) |
| `context/progress-tracker.md` | the skill performing the state change — except inside the build loop, where ONLY the Orchestrator writes it (see §8) |
| `context/memory/patterns.md`, `context/memory/decisions.md` | QA only, after a task is `done` (dedup before appending) |
| `context/memory/session.md` | `/remember` only |
| context files (`context/architecture.md`, `*-rules.md`, `project-*.md`, and the non-registry sections of `context/ui-context.md` / `context/api-context.md`) | developer; agents only with explicit approval |
| living registries (component registry in `context/ui-context.md`, endpoint registry in `context/api-context.md`) | Builders append/update entries for what they build |

## 6. Concurrent writes (read-modify-write)

`context/tasks-list/NN-*-tasks.md` and `context/progress-tracker.md` are shared between parallel
Builders, QA, and the Orchestrator. §5 says who may write; this says how:

1. Re-read the whole file immediately before every write — never write from a
   copy read earlier in the session.
2. Touch only your own section (your task block, your feature's row). If the
   bytes you are about to replace differ from what you just read, another agent
   wrote in between: abort, re-read, re-apply.
3. Never overwrite another agent's edit — there is no "mine wins."
4. Progress tables are always recounted from the Status fields, never
   hand-incremented (+1/−1 drifts; recounting cannot).

## 7. Verification gates (blocking, every task)

1. Lint — zero errors
2. Typecheck — zero errors; suppressions need written reasons
3. Native build — the real build command exits 0 (typecheck alone is insufficient)
4. Tests — all old and new tests pass, and every test file follows `context/architecture.md` →
   Test structure: one root `test/` folder with the fixed root files (`main`, `app`,
   `setup`, `helpers`, `e2e/`), every other spec at its mirrored source path — none
   beside source files or inside module/feature folders. 0 collected tests = fail
5. Acceptance criteria — verified item by item, not assumed
6. Standards — placement per `context/architecture.md`; tokens-only UI; validated,
   enveloped, auth-guarded APIs; no unlinked TODO/FIXME

## 8. Context discipline

Before implementing or making any architectural decision, read in order:
`context/project-overview.md` → `context/architecture.md` → `context/project-standards.md` →
`context/progress-tracker.md`, plus `context/ui-rules.md` + `context/ui-context.md` for UI work and
`context/api-rules.md` + `context/api-context.md` for backend work. Never implement from the
prompt alone.

Keep the docs in sync with reality:

1. **After each meaningful implementation change**, update `context/progress-tracker.md`
   (status rows + a Session notes line) and the affected living registries
   (component registry in `context/ui-context.md`, endpoint registry in `context/api-context.md`).
   After each task is approved `done`, QA updates `context/memory/patterns.md` and
   `context/memory/decisions.md` (new knowledge only — skip when nothing was learned).
   Every skill that changes state finishes by updating the `## Next` line in
   `context/progress-tracker.md` with the single next command to run — except Builders
   and QA inside the build loop: they never touch the tracker; the Orchestrator
   updates it on its tick.
2. **If implementation reveals that the architecture, scope, or standards in a
   context file are wrong or incomplete**, do not keep building against a document
   you know is stale — and do not silently rewrite it either (§5: those files are
   developer-owned). Stop, state the mismatch and the proposed doc change, and
   continue only after the developer approves the update.
3. **Code never drifts from docs silently.** Either the code conforms to the
   context files, or the context files change first — there is no third state.

## 9. Interaction style

- Stop at stage boundaries; the developer starts the next stage.
- Report failures plainly with output — never soften, never bury.
- Unknowns are marked TBD and surfaced, never invented.
- When file state and tracker state disagree, ask; the developer may have edited
  by hand, and that is legitimate.
