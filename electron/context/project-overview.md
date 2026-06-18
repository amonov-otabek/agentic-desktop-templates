# Project Overview

A concise, high-level summary of this project. Every agent reads this before any
feature work. Derived from the `desktop/electron` template — a cross-platform
desktop client that consumes the backend monolith API.

## What we are building

A production-grade **Electron desktop application** with a **React + Vite +
Tailwind** renderer. A single packaged app that runs on macOS, Windows, and Linux,
talks to the shared backend HTTP API over the network, and presents the same domain
modules (identity, message, audit, notification, payment, analytic) as the web and
mobile clients — through a native desktop shell with secure process separation.

## Goals

1. Ship a hardened desktop shell first: secure main/renderer separation
   (`contextIsolation` on, `nodeIntegration` off), a typed preload bridge, routing,
   data-fetching, and a tokens-driven design system — production-grade from the
   first commit.
2. Build feature areas (identity, message, …) one at a time against the backend API,
   keeping each feature isolated under `src/renderer/src/features/<name>` so they can
   be developed, tested, and reasoned about independently while shipping as one app.

## Platforms

| Surface | Tech | Who uses it |
| --- | --- | --- |
| macOS | Electron 42 → `.dmg` (electron-builder) | desktop end users |
| Windows | Electron 42 → NSIS installer | desktop end users |
| Linux | Electron 42 → AppImage | desktop end users |
| Backend API | consumed over HTTPS (`/api/v1`) | the app, via the typed API client |

## Core flows

1. A user opens the app; the main process creates a `BrowserWindow` and loads the
   renderer (Vite dev server in dev, built `index.html` over `file://` in prod).
2. The renderer authenticates against the backend (email/phone + password, OAuth,
   or 2FA) and receives a JWT access + refresh token pair, stored securely.
3. Authenticated views fetch data through the typed API client (TanStack Query);
   the response envelope (`{ success, data, meta? }`) is unwrapped into typed models.
4. App-specific OS capabilities (window controls, native dialogs, file access) are
   reached only through the preload `contextBridge` over validated IPC channels —
   never via `nodeIntegration` in the renderer.

## Roles

| Role | Description |
| --- | --- |
| user | standard authenticated account |
| admin | elevated views (admin/audit features) gated by backend role |
| _(RBAC)_ | the backend owns roles/permissions; the renderer hides/disables UI by role |

## Key constraints

- **Security first:** `contextIsolation: true`, `nodeIntegration: false`; the renderer
  has no Node access — only the explicit preload bridge surface.
- External links open in the OS browser (`setWindowOpenHandler` → `shell.openExternal`),
  never inside an app window.
- Tokens stored via secure OS-backed storage (planned — `safeStorage` / keychain),
  never in plain `localStorage` for refresh tokens.
- All visual values come from design tokens; no hardcoded colors/spacing.

## Scaffold status (honest)

- **Shell in place, features planned.** The desktop shell exists and runs: main
  process (`src/main/index.ts`), preload bridge (`src/preload/index.ts`, currently an
  empty `api = {}` surface), renderer entry, HashRouter, `AppShell` layout, a
  placeholder `HomePage`, the `Providers` (TanStack Query) tree, and Tailwind v4 wired
  in.
- **Every feature folder is an empty scaffold.** `src/renderer/src/features/<name>/`
  exists for all six modules with `api/ components/ pages/ queries/ types/ validators/`
  subfolders containing only `.gitkeep`. No working screens, no auth, no API client
  yet — these are the planned milestones.
- The first feature area becomes `context/features-list/01-<name>.md`; auth/identity is
  the natural first build against the backend API.

## Out of scope

- The backend itself (a separate project — this app only consumes its API; see
  `context/api-context.md`).
- Auto-update infrastructure and code signing beyond the electron-builder targets
  already configured (`.dmg` / NSIS / AppImage).
