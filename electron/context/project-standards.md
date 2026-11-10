# Project Standards

Implementation rules for all code. Derived from the `desktop/electron` template.
Electron + React desktop project. Treat as locked.

## Engineering mindset

1. **Think first** — read the task block and relevant context before writing code.
2. **Scope is sacred** — build what the task says; nothing more, nothing less.
3. **Every change is verifiable** — if it can't be tested or observed, it isn't done.
4. **Clean over clever** — boring, readable code wins.
5. **One thing at a time** — never mix refactoring with feature work.
6. **Failures are expected** — handle errors as first-class flow (network, IPC, auth).
7. **Fix root causes** — never patch symptoms or wrap broken code.

## TypeScript

- **File names (HARD):** PascalCase for pages, components, layouts and providers; camelCase for hooks (`useX.ts`) and every other module; lowercase/kebab-case folders; generated and framework-reserved files keep their names — full table in `context/ui-rules.md` → File naming.
- Strict mode (`strict: true`), `noUnusedLocals`, `noUnusedParameters`,
  `verbatimModuleSyntax`, `isolatedModules` all on (both project refs).
- No `any` — use precise types or `unknown` + narrowing. No `as` to silence errors.
- Renderer uses the `@/*` alias (→ `src/renderer/src`) — not deep `../../..` chains.
  Main/preload use relative imports (they are in the node project, no `@` alias).
- Use `import type { … }` for type-only imports (`verbatimModuleSyntax` requires it).
- Explicit return types on exported functions and hooks.
- Validate all external input (API responses, IPC payloads, form input) at the boundary.

## Process separation (Electron)

- **Main process** (`src/main`) owns Node, OS access, windows, and IPC handlers. It is
  the only place `electron`, `node:*`, and `fs` may be imported.
- **Preload** (`src/preload`) is the single bridge: it exposes a typed, explicit `api`
  surface via `contextBridge.exposeInMainWorld`. Keep it small and validated.
- **Renderer** (`src/renderer`) is sandboxed UI — no Node, no `electron`. It reaches
  the OS only through `window.api` / `window.electron` (typed in `preload/index.d.ts`).
- Never set `nodeIntegration: true` or `contextIsolation: false`. Never call
  `ipcRenderer` directly from React components.

## Secure IPC

- Every IPC capability is a named method on the preload `api` object and a typed
  entry in `preload/index.d.ts`. No anonymous channels.
- Channel names are validated allowlists on the main side; reject unknown channels.
- Validate every IPC payload in the main process before acting on it — treat renderer
  input as untrusted.
- Prefer `ipcRenderer.invoke` / `ipcMain.handle` (request/response) over fire-and-forget
  `send`/`on` unless a one-way event genuinely fits.

## Renderer component structure

- **Pages** (`features/<name>/pages`) are route-level: they compose sections, own data
  fetching via query hooks, and handle loading/empty/error/success states.
- **Components** (`features/<name>/components`, `shared/components`) render from props.
  No data fetching, no business logic in leaf components.
- Shared primitives live in `shared/components/ui` (tokens-only); cross-feature
  composites in `shared/components/widgets`.
- No business logic in components — derive/transform in hooks, services, or the
  query/api layer.

## Data fetching

- Server state goes through **TanStack Query** (`features/<name>/queries`):
  `useXxxQuery` / `useXxxMutation` hooks wrapping `features/<name>/api` calls.
- The API client unwraps the response envelope (`{ success, data }`), throws typed
  errors on `{ success: false }`, and never returns raw `unknown` to the UI.
- Local/UI state goes through **Zustand** stores or React state — never duplicate
  server state into a Zustand store.
- Handle every async UI state explicitly: loading, empty, error, success.

## Styling

- **Tokens only.** Every color/spacing/radius/font value comes from a token in
  `context/ui-context.md` (Tailwind `@theme` in `styles/globals.css`). No raw hex/px.
- Compose conditional classes with `cn()` (`shared/libs/utils.ts`) — `clsx` +
  `tailwind-merge`. Do not hand-concatenate class strings.
- Icons come from `lucide-react`; do not add another icon set.

## Accessibility

- Semantic HTML first; labels on inputs; visible focus styles; full keyboard reach.
- Text contrast ≥ 4.5:1; respect `prefers-reduced-motion`.
- Interactive controls disabled while their async action runs.

## Errors

- Network/API failures, IPC failures, and auth expiry are expected flow — handle
  them, surface a user-friendly message, never a stack trace.
- No empty catch blocks; log full context to the console/main-process logger; show
  the user a localized, friendly message.
- Never leak tokens, secrets, or raw error internals into the UI.

## Testing

- **Where tests live (HARD):** every test file goes in `test/` (`main`, `app`, `setup`, `helpers`, `e2e/`), at the path that
  mirrors its source file — see `context/architecture.md` → **Test structure**. Never beside a
  source file, never inside a module/feature folder. `setup` keeps the suite
  self-contained: tests run on a clean checkout with no developer `.env`.
- No test runner is wired yet. When a task needs tests, add Vitest +
  React Testing Library for the renderer; note the new dependency in QA notes.
- New behavior ships with a happy-path test plus at least one edge/error case once a
  runner exists; pre-existing tests stay green.

## Comments, TODOs, dependencies

- Comments explain *why*, where the code can't. No TODO/FIXME/HACK without a linked task.
- Prefer existing dependencies; a new one requires developer approval (note it in QA notes).
- Before using any third-party library, load its installed skill if present or verify
  against the installed version's docs — APIs drift, training data goes stale.
- pnpm is the package manager (see `context/architecture.md`); do not introduce a second lockfile.
