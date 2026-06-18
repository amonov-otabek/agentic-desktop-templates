# API Context — Backend Surface (Consumed)

Machine-readable contract for the **backend API this desktop app consumes**: the
envelope it must unwrap, the auth model it implements, shared conventions, and the
living registry of endpoints the app actually calls. `context/api-rules.md` says how to
talk to the backend; this file IS the contract. This app does not own the backend —
it is a client. Keep this consistent with `context/architecture.md`.

> The IPC bridge (`window.api`, preload `contextBridge`) is a **separate internal
> surface** between renderer and main process — it is not this HTTP contract. IPC
> channels are documented in `src/preload/index.d.ts`, not in this registry.

## Response envelope (every endpoint, no exceptions)

The backend returns this envelope; the API client (`shared/libs`) unwraps it and the
renderer code only ever sees typed `data` or a thrown typed error.

```ts
// success
interface SuccessResponse<T> {
  success: true;
  data: T;
  meta?: PaginationMeta;          // present on paginated lists
}

// error
interface ErrorResponse {
  success: false;
  error: { code: string; message: string; fields?: Record<string, string[]> };
}

interface PaginationMeta { page: number; limit: number; total: number; totalPages: number }
```

The error is an **object** (`{ code, message, fields? }`), not a bare string. UI
copy/branching keys off `error.code`, not the message text (messages are i18n).

## Auth model

- Mechanism: **JWT** — the app sends the access token as a **Bearer** token in the
  `Authorization` header. Access token is short-lived; a refresh token mints new
  access tokens. Login supports email/phone + password, OAuth, and 2FA (the backend
  drives the flow; the renderer renders the steps).
- **Secure token storage:** refresh tokens are stored via OS-backed secure storage
  (Electron `safeStorage` / OS keychain, accessed through the preload bridge), **not**
  in plain `localStorage`/`sessionStorage`. The access token may live in memory.
- **Refresh handling:** the API client intercepts `401` (expired access token),
  refreshes once, and retries; a failed refresh clears auth and routes to login.
- **Authorization:** the backend enforces roles/permissions. The renderer hides or
  disables UI by role for UX only — it never relies on the client for enforcement.

## Pagination convention

- Query params: `?page=1&limit=20`. `page` ≥ 1 (default 1), `limit` 1–100 (default 20).
  List screens extend this with their own validated filters.
- Response: `meta: { page, limit, total, totalPages }`. Query hooks page from day one
  (infinite or paged), never load-all.

## Validation convention

- The app validates **form input before sending** and **API responses on receipt**
  (defensive parsing into the feature's `types`). Schemas live in
  `features/<name>/validators`.
- On a `400`, surface field errors from `error.fields` next to the matching form
  fields, and `error.message` as the summary.

## Networking conventions

- Base URL comes from config/env (e.g. `VITE_API_URL`), never hardcoded per call.
- All calls go through the shared typed API client — no bare `fetch` in components.
- Standard status handling: `200/201` success, `400` validation, `401` re-auth,
  `403` forbidden, `404` not found, `409` conflict, `5xx` generic retry/toast.

## Consumed-endpoint registry (living document)

Updated by Builders with every backend endpoint the app calls; checked before adding
a new call (reuse the existing client method > duplicate); audited by /review. One
row per backend `/api/v1` endpoint this app consumes.

| Method | Path | Auth | Feature | Used by (query/page) |
| --- | --- | --- | --- | --- |
| _(empty — first row added by the first feature that calls the API)_ |

## Invariants

1. No backend call outside the typed API client; no call that isn't in this registry.
2. No renderer code consumes the raw envelope — the client unwraps to typed `data`
   or throws a typed error.
3. Types shared between the api / queries / pages layers are defined once in the
   feature's `types/` and imported — never duplicated by hand.
4. Auth tokens are never logged and refresh tokens never touch plain web storage.
