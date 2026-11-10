# Architecture Context

HARD CONTRACT: file placement, stack choices, and boundary rules here are
non-negotiable for Builders — QA rejects violations. Derived from the
`desktop/electron` template; keep it locked.

## Stack

| Layer                 | Technology                                              | Role               |
| --------------------- | ------------------------------------------------------- | ------------------ |
| Desktop runtime       | Electron 42 (Chromium renderer + Node main process)     | app shell, OS access |
| Language              | TypeScript ~6.0 — strict, two project refs (node + web) | all source         |
| Bundler / dev         | electron-vite 5 (Vite 7, esbuild) — main / preload / renderer | build + HMR    |
| Renderer framework    | React 19 (`react-jsx`, StrictMode)                      | UI                 |
| Routing               | react-router-dom 7 — **HashRouter** (`createHashRouter`) | renderer routing  |
| Data fetching         | TanStack Query 5 (`@tanstack/react-query`)              | server-state cache |
| Client state          | Zustand 5                                               | local/UI state     |
| Styling               | Tailwind CSS v4 (`@tailwindcss/vite`, `@import "tailwindcss"`) | tokens + utilities |
| Class utility         | `clsx` + `tailwind-merge` (`cn` in `shared/libs/utils.ts`) | conditional classes |
| Icons                 | `lucide-react`                                          | iconography        |
| IPC / preload bridge  | `contextBridge` via `@electron-toolkit/preload` (`electronAPI`) | main↔renderer |
| Main helpers          | `@electron-toolkit/utils` (`is`, `electronApp`, `optimizer`) | dev/prod, shortcuts |
| Packaging             | electron-builder 26 (`electron-builder.yml`)            | `.dmg` / NSIS / AppImage |
| Quality               | ESLint 10 + typescript-eslint, Prettier, Husky + lint-staged | lint/format/hooks |

Package manager: **pnpm** is canonical (`pnpm-lock.yaml`, `pnpm.onlyBuiltDependencies`,
`.npmrc`). Do not introduce a second lockfile or mix package managers.

> Testing is not yet wired (no test runner in `package.json`). When a feature needs
> tests, add Vitest + React Testing Library for the renderer (and note it in QA
> notes per `context/project-standards.md`); do not assume a runner exists today.

## Commands

| Action | Command |
| --- | --- |
| Dev (all 3 processes + HMR) | `pnpm dev` (`electron-vite dev`) |
| Build (compile main/preload/renderer to `out/`) | `pnpm build` (`electron-vite build`) |
| Preview built app | `pnpm start` (`electron-vite preview`) |
| Typecheck (native gate) | `pnpm typecheck` (`typecheck:node` + `typecheck:web`) |
| Lint | `pnpm lint` (`eslint .`) |
| Format check | `pnpm format:check` |
| Package (unpacked) | `pnpm build:unpack` (`electron-builder --dir`) |
| Package macOS | `pnpm build:mac` |
| Package Windows | `pnpm build:win` |
| Package Linux | `pnpm build:linux` |

QA and Builders run exactly these — keep them accurate. `pnpm typecheck && pnpm lint
&& pnpm build` is the native verification gate.

## Folder structure

Path alias (renderer only, `tsconfig.web.json` + `electron.vite.config.ts`):
`@/*` → `src/renderer/src/*`. Main and preload (`tsconfig.node.json`) use relative
imports — they run in the Node context and have no `@` alias.

```
src/
├── main/
│   └── index.ts              ← main process: createWindow(), app lifecycle, IPC handlers
├── preload/
│   ├── index.ts              ← contextBridge: exposes electronAPI + app `api` surface
│   └── index.d.ts            ← global Window typings for the exposed bridge
└── renderer/
    ├── index.html            ← renderer entry HTML
    └── src/
        ├── main.tsx          ← React root: StrictMode → Providers → RouterProvider
        ├── app/
        │   ├── router.tsx    ← createHashRouter route tree
        │   └── HomePage.tsx  ← placeholder landing page
        ├── layout/
        │   └── AppShell.tsx  ← <Outlet/> chrome (the desktop frame)
        ├── features/<name>/  ← one folder per domain feature (see Feature shape)
        ├── shared/
        │   ├── components/
        │   │   ├── provider/ ← Providers (TanStack Query client)
        │   │   ├── ui/       ← design-system primitives (tokens-only)
        │   │   └── widgets/  ← cross-feature composite components
        │   ├── hooks/        ← shared React hooks
        │   ├── i18n/         ← localization
        │   └── libs/         ← utils.ts (cn), api client (planned)
        └── styles/
            └── globals.css   ← Tailwind v4 entry (@import "tailwindcss"); @theme tokens

test/                          ← mirrors src/ (see Test structure)
electron.vite.config.ts        ← electron-vite: main / preload / renderer configs
electron-builder.yml           ← appId, productName, mac/win/linux targets
tsconfig.{json,node,web}.json  ← root refs + node (main/preload) + web (renderer)
.github/workflows/             ← ci-checking.yml, cd-release.yml
```

### Feature shape (every `features/<name>/`)

```
pages/         ← route-level screens; orchestrate sections, own data fetching
components/    ← presentational components for this feature (render only)
queries/       ← TanStack Query hooks (useXxxQuery / useXxxMutation)
api/           ← typed API-client calls for this feature's endpoints
types/         ← TS interfaces for this feature's models / DTOs
validators/    ← input/output schemas (form + API response shapes)
```

Every file a Builder creates must fit this tree; test files follow
**Test structure** below. New top-level folders require
developer approval first. (Today these folders hold only `.gitkeep` — the scaffold.)

## Test structure (HARD)

Every project built from this template uses this exact test layout. QA rejects a task whose test files break any rule below.

```
test/
├── main.spec.ts               ← main process: createWindow, lifecycle, IPC
├── app.spec.tsx               ← renderer shell: Providers, router, app-shell
├── setup.ts                   ← self-contained test env (runner setup file)
├── helpers.tsx                ← render-with-providers, fixtures, mocks
├── preload/                   ← mirrors src/preload (exposed bridge surface)
├── renderer/src/              ← mirrors src/renderer/src: features/<name>/, shared/, layout/
└── e2e/                       ← <flow>.e2e-spec.ts
```

Source → spec mapping:

```
src/main/index.ts                                  → test/main.spec.ts (main process)
src/preload/index.ts                               → test/preload/index.spec.ts
src/renderer/src/app/router.tsx                    → test/renderer/src/app/router.spec.tsx
```

1. **One location only.** Every test file lives under `test/` at the project root — never next to a source file, never inside a module/feature folder, never in a second test folder.
2. **Fixed root files — nothing else at the root.** `main.spec.ts` covers the main process only: `createWindow()`, app lifecycle, IPC handler registration. `app.spec.tsx` covers the renderer shell only: providers, hash router, `app-shell`. `setup.ts` makes the suite self-contained: electron + IPC mocks for the node context, a `window.api` bridge mock and DOM matchers for the renderer — loaded by both runner projects. `helpers.tsx` holds render-with-providers (router, query client, i18n, theme), fixtures and mocks — no assertions. Besides these four, the root holds only mirrored folders and `e2e/`.
3. **Mirror the source tree.** Every other spec sits at the path of the file it tests: same folder names, same file name, plus the `.spec.ts` / `.spec.tsx` suffix (`*.test.*` is not used).
4. **One spec per source file.** Moving or renaming a source file moves or renames its spec in the same task.
5. **No catch-all spec.** Logic shared across the app is tested at the mirrored path of the file that owns it — never in a "global" / "misc" / "utils" spec.
6. **Specs import only source code and the helpers file** — never another spec, never a fixture defined inside another spec.
7. **End-to-end tests** live only in `test/e2e/` (Playwright `_electron`, launching the built app), named `<flow>.e2e-spec.ts`. Unit/integration specs never go there; the e2e folder may hold one `helpers` file of its own (route/bridge mocks, launch fixtures).
8. **Runner config follows the folder.** No test runner is wired yet. When one is added (Vitest with a `node` project for `test/main.spec.ts` + `test/preload/` and a `jsdom` project for `test/app.spec.tsx` + `test/renderer/`; Playwright for e2e): `setupFiles: ['test/setup.ts']`, exclude `test/e2e/**`, coverage measures `src/`; then add the commands to the table above. If the config disagrees with this section, fix the config in the same task.
9. **Same tree in every template.** The root file names (`main`, `app`, `setup`, `helpers`, `e2e/`) are identical across all stacks, so a developer moving between a TS and a Python API finds the same layout. A change to this rule is applied to every template at once.
10. **Legacy layout is drift, not precedent.** If existing code or a feature spec places tests elsewhere (e.g. a module's own `tests/` folder or `*.test.ts` files), new and modified tests still follow this section; moving the old ones is its own task — surface the drift to the developer, never silently mass-move files.

## Boundary rules (HARD)

1. **Process separation is sacred.** Main-process code (Node, OS, `electron`) lives
   only under `src/main`. Renderer code never imports `electron`, `node:*`, or `fs`.
2. **`contextIsolation: true`, `nodeIntegration: false`.** The renderer reaches the
   OS only through the preload `contextBridge`. Never enable `nodeIntegration` or
   disable isolation to "make something work".
3. **Typed IPC bridge.** Every capability the renderer needs is an explicit, named
   method on the preload `api` surface (`src/preload/index.ts`), typed in
   `src/preload/index.d.ts`. No `ipcRenderer` calls scattered in the renderer.
4. **Pages orchestrate, components render.** A page composes sections and owns data
   fetching (queries); presentational components receive props and render — no inline
   section JSX in pages, no fetching inside leaf components.
5. **Tokens-only styling.** Every visual value comes from a token defined in
   `context/ui-context.md` (Tailwind `@theme`). No hardcoded hex/px/magic spacing.
6. **Typed boundaries.** API responses unwrap through the typed client into models in
   `features/<name>/types`; never pass raw `unknown`/`any` payloads into the UI.
7. **Immutability everywhere.** Never mutate state objects — return new copies
   (React state, Zustand stores, query data).
8. **Features are isolated.** A feature does not import another feature's internals;
   shared building blocks live in `shared/`.

## Cross-cutting invariants

- **Renderer routing is hash-based** (`createHashRouter`) because production loads
  over `file://`, where a browser/history router cannot resolve deep links.
- **External navigation** always goes to the OS browser via `shell.openExternal`
  (`setWindowOpenHandler` denies in-app new windows).
- **Two TS project refs:** `tsconfig.node.json` (main + preload, `types: ["node"]`)
  and `tsconfig.web.json` (renderer, `types: ["vite/client"]`, DOM libs). Keep
  main/preload code out of the web project and vice versa.
- **Response envelope** the renderer consumes (from the backend, see
  `context/api-context.md`):
  - success: `{ success: true, data, meta? }`
  - error: `{ success: false, error: { code, message, fields? } }`
- **Build output** lands in `out/` (main/preload/renderer); `package.json#main`
  points at `./out/main/index.js`.

## Decisions log

Locked decisions agents must not re-litigate.

- **electron-vite over electron-forge** — one config drives all three processes with
  Vite HMR; `externalizeDepsPlugin` keeps native deps out of the bundle.
- **HashRouter, not BrowserRouter** — production runs over `file://`.
- **TanStack Query for server state, Zustand for local state** — Query owns the
  cache/refetch lifecycle for API data; Zustand holds ephemeral UI/app state. Do not
  duplicate server state into Zustand.
- **contextBridge-only IPC** — the renderer is sandboxed from Node; capabilities are
  an explicit, typed bridge surface, never ambient Node access.
- **Tailwind v4 with `@theme` tokens** — the single source of truth for visual values.
