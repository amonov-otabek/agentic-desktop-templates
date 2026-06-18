# Feature 08 — Desktop Testing & QA

- Feature ID: FEAT-08
- Status: draft
- Depends on: FEAT-01, FEAT-02, FEAT-03, FEAT-04, FEAT-05, FEAT-06, FEAT-07

> Source of truth for the **test harness**: renderer unit/component tests (Vitest + React
> Testing Library), main/preload unit tests, and Playwright-for-Electron end-to-end flows
> against the packaged shell, plus accessibility checks.
>
> HONEST STATE: **to add — no test runner exists today.** `package.json` ships no `test`
> script and no test dependencies; there is no `vitest.config`, no `playwright.config`, and
> no `src/test/` helpers. This feature installs and wires the entire harness. The native
> gate today is only `pnpm typecheck && pnpm lint && pnpm build`.

## Goal

Stand up a working test suite for the desktop app: Vitest + React Testing Library for
renderer component/unit tests (mocked API client + mocked query hooks, and a mocked
`window.api` IPC bridge), Vitest unit tests for the typed API client / auth service / preload
bridge contract, Playwright-for-Electron e2e for critical flows against the built app (with
intercepted backend responses), and automated a11y assertions — all wired to pnpm scripts.

## Scope

### Tooling & config to install

- `vitest` (`jsdom` env for renderer; `node` env for main/preload unit tests) +
  `@vitest/coverage-v8`.
- `@testing-library/react`, `@testing-library/user-event`, `@testing-library/jest-dom`.
- `@playwright/test` (drives Electron via `_electron.launch({ args: ["."] })`) +
  `@axe-core/playwright` for a11y.
- `vitest.config.ts` (jsdom project for `src/renderer`, node project for `src/main` +
  `src/preload`, `setupFiles: src/test/setup.ts`); `playwright.config.ts` (single Electron
  project; builds the app with `pnpm build` then launches the packaged `out/main/index.js`).
- `src/test/setup.ts` — registers `@testing-library/jest-dom`, clears storage between tests,
  polyfills `matchMedia`, and stubs `window.api` + `window.electron` (the preload bridge) so
  renderer components run under jsdom without a real main process.
- `src/test/render.tsx` — `renderWithProviders` mounting components with i18n, React Query
  (fresh client, retries off), an in-memory `MemoryRouter`, and the stubbed IPC bridge.

### Scripts (`package.json`)

`test` (Vitest run), `test:watch`, `test:coverage`, `test:e2e` (Playwright-for-Electron),
`test:e2e:ui` — added alongside the existing `typecheck` / `lint` / `build` gate.

### API & IPC mocking

- Component tests mock the feature query hooks (`vi.mock("../queries/*.queries")`) and the
  toast hook, asserting on rendered envelope-shaped data and on `ApiError.fields` without a
  live backend.
- `shared/libs/api.ts` is unit-tested directly against a mocked `fetch` (envelope unwrap,
  typed `ApiError`, 204, pagination `meta`, 401-refresh-then-retry path).
- The auth service is unit-tested with a **mocked `window.api.token.*` bridge** (set/get/clear
  refresh token; rotation; pending-2FA in memory) — no real `safeStorage`.
- The preload bridge contract is unit-tested: the exposed `api` shape matches
  `src/preload/index.d.ts` and each method forwards to the expected `ipcRenderer.invoke`
  channel name.
- E2E mocks the backend at the network layer (Playwright route interception) returning the
  `{ success, data, meta? }` envelope (`e2e/helpers/api-mock.ts`).

### Unit / component tests (Vitest)

- `shared/libs/api.test.ts` — envelope unwrap, typed `ApiError`, 204, pagination `meta`,
  401 → refresh → retry.
- `identity/services/auth.service.test.ts` — token persist/clear via mocked bridge, refresh
  rotation, pending-2FA.
- `preload/bridge.test.ts` — exposed `api` surface matches the declared types + channel names.
- Zod validators: `identity/validators/auth.validator.test.ts`,
  `message/validators/message.validator.test.ts`,
  `payment/validators/payment.validator.test.ts`.
- One+ representative component per feature domain (identity sign-in/sign-up forms; analytic
  stat-cards; audit alert-table resolve; payment payment-list refund; notification list
  mark-read; message compose/list) against mocked hooks/client.

### E2E (Playwright-for-Electron)

- `e2e/launch.spec.ts` — the app launches, the main window appears, and the renderer mounts
  (HashRouter `#/`) + axe (no serious/critical violations).
- `e2e/route-guard.spec.ts` — unauthenticated deep link to a dashboard route redirects to
  `/sign-in` (hash route).
- `e2e/auth-flow.spec.ts` — sign-in happy path lands on `/dashboard` and persists the refresh
  token through the bridge; sign-out returns to `/sign-in` and clears it. Backend mocked via
  route interception.
- `e2e/preferences.spec.ts` — locale switch persists across relaunch; theme toggle persists
  across relaunch.

## Consumed API

- None live. Component tests mock query hooks + `shared/libs/api.ts` and stub `window.api`;
  e2e intercepts the `{ success, data, meta? }` envelope at the network layer.

## UI / components used

- Test utilities under `src/test/*` (`renderWithProviders` wrapping i18n / Query / router /
  stubbed IPC bridge); e2e helpers under `e2e/helpers/*`; existing feature components/pages
  and the preload bridge as units under test.

## Acceptance criteria

- [ ] Vitest, React Testing Library, Playwright-for-Electron, and axe are installed and
      configured; `test` / `test:watch` / `test:coverage` / `test:e2e` / `test:e2e:ui` scripts
      run.
- [ ] `renderWithProviders` (i18n + Query + router + stubbed `window.api`) and
      `src/test/setup.ts` exist and are used by component tests.
- [ ] `shared/libs/api.ts` is unit-tested (envelope, `ApiError`, 204, `meta`, 401-refresh) and
      the auth service is tested against a mocked token bridge (no real `safeStorage`).
- [ ] The preload bridge contract test asserts the exposed `api` shape matches `index.d.ts`
      and forwards to the expected IPC channels.
- [ ] At least one component test per feature (identity, analytic, audit, payment,
      notification, message) runs against mocked hooks/client.
- [ ] Playwright-for-Electron launches the built app and covers: launch + mount, route-guard
      redirect, sign-in → dashboard + sign-out, locale/theme persistence — backend mocked.
- [ ] Axe runs on at least the launched main window (no serious/critical violations).
- [ ] Coverage is wired via `test:coverage` (threshold target ≥80%; note in QA whether it is
      enforced or measured only).
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
