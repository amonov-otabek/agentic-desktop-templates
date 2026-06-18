# Feature 02 — Identity & Auth (Desktop)

- Feature ID: FEAT-02
- Status: draft
- Depends on: FEAT-01 (shell: API client, secure token bridge, providers, HashRouter, layouts, theme, i18n)

> Source of truth for the **desktop identity layer**: the full auth surface, session +
> token handling backed by OS-secure storage, profile/settings, and the RBAC admin
> user-management surface. It wires `features/identity/*` to the backend identity API; it
> builds no backend.
>
> HONEST STATE: **to build.** `src/renderer/src/features/identity/{api,components,pages,queries,types,validators}`
> currently hold only `.gitkeep`. The secure-token IPC bridge and typed API client it
> depends on are delivered by FEAT-01.

## Goal

Make sign-in / sign-up and the full identity surface work end-to-end against the backend
`/api/v1` identity API: access token in renderer memory + refresh token in OS-secure
storage with silent rotation, guarded dashboard routes, email verification, password
reset, 2FA, profile/settings, and a role-gated admin user-management surface.

## Scope

- **Auth pages** (`AuthLayout` group, `features/identity/pages/*`):
  - `sign-in-page` — identifier (email/phone) + password, `remember_me`, `trust_device`;
    on `requires_two_fa` stash the pending token and route to `/two-factor`.
  - `sign-up-page` — email/phone + password + names, terms/privacy acceptance.
  - `verify-email-page` — consume an email-link token (`/auth/verify-email-link`).
  - `forgot-password-page` — request a reset email.
  - `reset-password-page` — submit token + new password.
  - `two-factor-page` — complete the 2FA challenge with the pending token.
- **Settings page** (`features/identity/pages/settings-page`, `/settings` under
  `DashboardLayout`) — profile edit, password change, 2FA setup/enable/disable, active-session
  list + revoke / revoke-all.
- **Admin users page** (`features/identity/pages/users-page`, `/users`) — paginated user
  list, detail, toggle-active, role assignment, delete; roles list. Gated by role via
  `shared/hooks/use-permission`.
- **Session / token handling** (`services/auth.service.ts`) — access token in renderer
  memory; refresh token persisted via the FEAT-01 preload bridge (`window.api.token.*` →
  main `safeStorage`); `loadSession()` hydrates on startup by reading the stored refresh
  token and minting an access token; silent refresh-rotation; forced sign-out (clear stored
  token + redirect `/sign-in`) on `401/403`; pending-2FA token kept in renderer memory only.
- **Client auth state** (`store/auth.store.ts`, Zustand) — `user`, `status`, `setUser`,
  `clear`; `useMe` hydrates the store on load; the route guard reads `isAuthenticated()`.
- **Validators** (`validators/auth.validator.ts`, Zod) — sign-in/up, forgot/reset, 2FA,
  profile, change-password; server field errors (`error.fields`) map onto inputs.
- **Queries** (`queries/{auth,settings,admin}.queries.ts`) — `useMe`, `useSignIn`,
  `useSignUp`, `useSignOut`, `useVerifyTwoFactor`, `useForgotPassword`, `useResetPassword`,
  `useVerifyEmail`, profile/password/2FA/session hooks, admin user hooks; mutations
  invalidate the relevant query keys.
- Out of scope: OAuth providers, devices/security center, address book (add as token
  follow-ups only if the backend exposes them — not in the base template).

## Consumed API

Base `/api/v1`, envelope `{ success, data, meta? }`. Branch on `error.code` / `error.fields`.
Secure token persistence is local via the preload bridge, not an HTTP endpoint.

- **Auth:** `POST /auth/sign-in` (2FA → `requires_two_fa` + `pending_token`, else `tokens`)
  · `POST /auth/sign-up` · `POST /auth/refresh` (rotate pair) · `POST /auth/logout`.
- **Verify / password:** `POST /auth/verify-email-link` · `POST /auth/forgot-password`
  · `POST /auth/reset-password` · `PATCH /auth/password`.
- **2FA:** `GET /2fa/status` · `POST /2fa/setup` (secret + QR) · `POST /2fa/enable`
  (backup codes) · `POST /2fa/verify` (challenge → tokens) · `DELETE /2fa/disable`.
- **Profile:** `GET /profile/me` · `PATCH /profile/me`.
- **Sessions:** `GET /sessions` · `DELETE /sessions/:id` · `DELETE /sessions`.
- **Admin (role-gated):** `GET /admin/users` (paginated) · `GET /admin/users/:id`
  · `PATCH /admin/users/:id/toggle-active` · `PATCH /admin/users/:id/role` ·
  `DELETE /admin/users/:id` · `GET /admin/roles`.

## UI / components used

- `features/identity/components/*`: `sign-in-form`, `sign-up-form`, `two-factor-form`,
  `forgot-password-form`, `reset-password-form`, `verify-email-view`, `auth-banner`,
  `profile-form`, `password-form`, `two-factor-card`, `sessions-table`, `users-table`,
  `roles-table`.
- `shared/components/ui/*` primitives (button, card, input, field, label, tabs, table,
  dialog, alert-dialog, badge, input-otp, pagination, toaster), `shared/hooks`
  (`use-permission`, `use-pagination`, `use-toast`), `AuthLayout` / `DashboardLayout`,
  `lucide-react` icons.

## Acceptance criteria

- [ ] Sign-up and sign-in work against the live API; access token in renderer memory, refresh
      token in OS-secure storage (main `safeStorage` via the preload bridge) and rehydrated by
      `loadSession()` on startup.
- [ ] Refresh rotation works: expired access → silent refresh via the stored refresh token;
      revoked/invalid refresh → forced sign-out, cleared secure storage, redirect to `/sign-in`.
- [ ] On `requires_two_fa`, the pending token is stashed (in memory) and `/two-factor`
      completes auth.
- [ ] Protected dashboard routes redirect unauthenticated users to `/sign-in`; authenticated
      users skip the auth group.
- [ ] Email-link verification, forgot → reset, and change-password flows work.
- [ ] 2FA full cycle: status → setup (QR) → enable (backup codes) → sign-in challenge verify
      → disable.
- [ ] Settings page edits profile, lists + revokes sessions (single + all), and manages 2FA.
- [ ] Admin users page lists (paginated), shows detail, toggles active, assigns a role, and
      deletes — gated by `use-permission`; non-admins cannot reach it.
- [ ] Server field errors (`error.fields`) map onto the right form inputs; assertions branch on
      `error.code`, never message text.
- [ ] All copy + validation errors localized (en/ru/uz); auth state in the Zustand store;
      mutations invalidate query keys; no `fetch` in components; no token ever written to web
      storage or logged.
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
