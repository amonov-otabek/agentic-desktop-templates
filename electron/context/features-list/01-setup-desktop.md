# Feature 01 — Desktop Shell & Scaffold

- Feature ID: FEAT-01
- Status: draft
- Depends on: none

> Source of truth for the **desktop foundation** of this Electron app. It stands up the
> runnable shell — main process, preload contextBridge, electron-vite build, renderer
> HashRouter + layout shell, theme/tokens, the typed envelope/JWT API client with secure
> OS-backed token storage, i18n, TanStack Query providers, and electron-builder packaging.
> It writes **no auth or business logic** — identity lands in FEAT-02.
>
> HONEST STATE: the shell is **partly built**. What already runs: main process
> (`src/main/index.ts` — `BrowserWindow` 1200×800, `setWindowOpenHandler` → `shell.openExternal`,
> dev/`file://` load split), preload bridge (`src/preload/index.ts` exposing `electron` +
> an **empty** `api = {}`; `index.d.ts` types `api: unknown`), renderer entry
> (`main.tsx` → StrictMode → `Providers` → `RouterProvider`), `createHashRouter` with only
> `/` → `AppShell` → `HomePage`, `Providers` holding **only** a `QueryClient`, `AppShell`
> using raw `neutral-*` Tailwind classes, and `globals.css` holding only
> `@import "tailwindcss"`. What REMAINS to build is everything below not marked “exists”.

## Goal

Finish the application skeleton so feature code can be written: a strict-TypeScript
Electron 42 + electron-vite app whose main/preload/renderer build clean, whose renderer
boots through a HashRouter with auth/dashboard layout shells, themes light/dark from
`@theme` tokens, talks to the backend through one typed envelope-aware API client, stores
the refresh token via OS-backed secure storage over a typed IPC bridge, localizes through
i18next, and packages to `.dmg` / NSIS / AppImage via electron-builder.

## Scope

- **Stack & tooling (exists)** — Electron 42, electron-vite 5 (Vite 7) driving
  main/preload/renderer, React 19 (StrictMode), strict TS with two project refs
  (`tsconfig.node.json` for main+preload, `tsconfig.web.json` for renderer), `@/*` →
  `src/renderer/src/*` alias in `tsconfig.web.json` + `electron.vite.config.ts`, ESLint 10
  flat + Prettier + Husky + lint-staged, pnpm canonical (`pnpm-lock.yaml`, no second lockfile).
- **Env config (to build)** — backend origin from `VITE_API_URL` (renderer-public; default
  `http://localhost:3000`); no secrets bundled in the renderer; `.env.local` gitignored,
  `.env.example` holds the public var.
- **Main process (exists, extend)** — `createWindow()`, app lifecycle, external-link handler
  already present. To build: register **named, allowlisted** `ipcMain.handle` channels for
  secure token storage (see below) and validate every payload before acting.
- **Preload bridge (to build)** — replace the empty `api = {}` with a typed, explicit
  surface exposed via `contextBridge.exposeInMainWorld("api", …)`; declare it in
  `src/preload/index.d.ts` (`Window.api` typed precisely, not `unknown`). Bridge surface for
  this feature: `api.token.get()` / `api.token.set(token)` / `api.token.clear()` backed by
  `ipcRenderer.invoke` of the main-side secure-storage channels. Keep it small and validated.
  IPC channels: `token:get`, `token:set`, `token:clear`.
- **Secure token storage (to build)** — main process persists the **refresh token** via
  Electron `safeStorage` (OS keychain), encrypted at rest on disk; the renderer reaches it
  only through the preload `api.token.*` bridge. The **access token** lives in renderer memory
  only. No refresh token in `localStorage`/`sessionStorage`.
- **Typed API client (to build)** (`shared/libs/api.ts`) — `api.get/getPaginated/post/put/patch/delete`;
  prepends `VITE_API_URL`, attaches the in-memory bearer token, unwraps the
  `{ success, data, meta? }` envelope, and throws a typed `ApiError(status, code, message, fields?)`
  on non-2xx / `{ success:false }` / parse failure. `204` → `data: undefined`. Pagination via
  `?page&limit` → `meta`. On `401` it refreshes once (using the bridge-stored refresh token)
  and retries; a failed refresh clears auth and routes to sign-in. Defines the contract
  identity (FEAT-02) consumes; makes no real calls itself.
- **i18n (to build)** (`src/renderer/src/shared/i18n`) — i18next + react-i18next; bundled
  `messages/{en,ru,uz}.json` (no browser detector — desktop reads a persisted locale);
  `routing.ts` locale list; every user-facing string resolves through `t(...)`.
- **Theme / tokens + dark mode (to build)** — add the `@theme` token block + `.dark` overrides
  from `context/ui-context.md` to `src/renderer/src/styles/globals.css` (today it is only
  `@import "tailwindcss"`); dark toggles via the `dark` class on `<html>`; window chrome
  respects OS theme. Tokenize the placeholder `AppShell` / `HomePage` (replace raw `neutral-*`).
- **Router (exists, extend)** (`src/renderer/src/app/router.tsx`) — `createHashRouter` is
  present with `/` → `AppShell` → `HomePage`. Extend to mount layout groups: `AuthLayout`
  (`/sign-in`, `/sign-up`, `/forgot-password`, `/reset-password`, `/two-factor`,
  `/verify-email`) and `DashboardLayout` (`/dashboard`, `/messages`, `/users`,
  `/notifications`, `/audit`, `/analytics`, `/payments`, `/settings`). Each dashboard route
  carries a `handle.titleKey`. HashRouter is locked (production loads over `file://`).
- **Layout shell (exists placeholder, extend)** (`src/renderer/src/layout/*`) — `AppShell`
  exists. To build: `AuthLayout`, `DashboardLayout` (sidebar + header + user nav + content),
  and a `coming-soon` placeholder for unbuilt routes.
- **Route guard (to build)** — dashboard routes check `isAuthenticated()` and redirect
  unauthenticated users to `/sign-in`; authenticated users skip the auth group. Guards are
  UX-only — the backend remains the authz source of truth (`401/403` clears the session).
- **Providers (exists, extend)** (`src/renderer/src/shared/components/provider/providers.tsx`)
  — today holds only a `QueryClient`. Extend to compose in order i18n → theme → TanStack
  Query → toaster; stays mounted in `main.tsx` wrapping `<RouterProvider/>`.
- **Production build & packaging (exists, verify)** — `pnpm build` (`electron-vite build` →
  `out/main`, `out/preload`, `out/renderer`); `package.json#main` → `./out/main/index.js`;
  `electron-builder.yml` produces `.dmg` (mac) / NSIS (win) / AppImage (linux);
  `.github/workflows/{ci-checking,cd-release}.yml`.
- Out of scope: any auth/business logic, feature data wiring, tests (FEAT-08).

## Consumed API

- None directly. This feature defines the **client** of the backend envelope contract
  (`/api/v1`, `{ success, data, meta? }`) consumed by every later feature, and the secure
  token-storage IPC bridge it relies on. It makes no real endpoint calls itself.

## UI / components used

- Layout shells (`src/renderer/src/layout/*`), `shared/components/provider/providers.tsx`,
  `shared/components/ui/*` (tokens-only primitives), `shared/components/widgets/*`
  (locale-switch, theme-toggle, pager, sidebar-nav), `shared/hooks/*`, `shared/libs/{utils,api}.ts`,
  tokens in `src/renderer/src/styles/globals.css`, icons from `lucide-react`.

## Acceptance criteria

- [ ] `pnpm install` succeeds; only one lockfile (`pnpm-lock.yaml`) present.
- [ ] `@/*` alias resolves identically in `electron.vite.config.ts` and `tsconfig.web.json`;
      main/preload stay in the node project (relative imports, `types: ["node"]`).
- [ ] `shared/libs/api.ts` prepends `VITE_API_URL`, attaches the in-memory bearer token,
      unwraps `{ success, data, meta? }`, throws typed `ApiError` on failure, resolves `204` to
      `undefined`, refreshes once on `401` then retries, and clears auth on refresh failure.
- [ ] The preload `api` surface is a typed, explicit object (`api.token.get/set/clear`) typed
      precisely in `index.d.ts` (no `api: unknown`); the renderer never calls `ipcRenderer`
      directly and never imports `electron`/`node:*`.
- [ ] The refresh token is stored via main-process `safeStorage` over allowlisted, payload-
      validated IPC channels; the access token lives only in renderer memory; no refresh token
      in web storage; no secrets bundled in the renderer (only `VITE_API_URL`).
- [ ] i18next loads `en/ru/uz`, the locale switch persists, and every visible string resolves
      through `t(...)`.
- [ ] Light/dark theme toggles via the `dark` class (light default); all visual values come
      from `globals.css` `@theme` tokens (no hardcoded hex/px); `AppShell`/`HomePage` are
      tokenized (no raw `neutral-*`).
- [ ] `createHashRouter` mounts the existing `/` plus the auth and dashboard layout groups
      with the routes in Scope; dashboard routes carry `handle.titleKey`.
- [ ] Dashboard routes redirect unauthenticated users to `/sign-in`; authenticated users are
      kept out of the auth group.
- [ ] Providers compose i18n → theme → query → toaster once and mount in `main.tsx`.
- [ ] `pnpm build` builds main/preload/renderer to `out/`; `pnpm build:mac|win|linux` produce
      `.dmg` / NSIS / AppImage via electron-builder.
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
