# Feature 07 — Messages (Desktop)

- Feature ID: FEAT-07
- Status: draft
- Depends on: FEAT-01 (shell), FEAT-02 (auth — guarded dashboard surface)

> Source of truth for the authenticated **messages** surface: compose an outbound message
> (email/sms channel) and browse the message list with delivery status, inside the desktop
> `DashboardLayout`. Consumes the backend `message` domain (async delivery).
>
> HONEST STATE: **to build.** `src/renderer/src/features/message/*` holds only `.gitkeep`.

## Goal

Give authenticated users a messages surface on `/messages`: compose a message to a chosen
channel and watch its delivery status in a paginated list — all through TanStack Query
against the backend message API via the typed client.

## Scope

- **Messages page** (`features/message/pages/messages-page`, route `/messages` under
  `DashboardLayout`) — orchestrates the compose form + message list; owns data fetching.
- **Compose form** (`components/message-compose-form`) — Zod-validated
  (`validators/message.validator`): channel (`email` | `sms`), recipient, subject/body;
  create mutation enqueues an async delivery job server-side.
- **Message list** (`components/message-list`) — paginated, filterable by `status`; shows
  channel, recipient, and delivery status (queued/sent/failed) badges.
- **Queries** (`queries/message.queries`) — `useMessages` (paginated), `useMessage` (detail),
  `useCreateMessage` (invalidates the list). Typed via `types/message.types` (`Message`,
  `CreateMessageDto`, `MessageListQuery`); fetched via `api/message.api`.
- Loading / error / empty / success states; status badges via existing semantic tokens.
- Out of scope: live status streaming, provider/template configuration (server-side),
  threaded conversations.

## Consumed API

Base `/api/v1/messages`, envelope `{ success, data, meta? }`; Bearer access token attached by
the client.

- `GET /messages?page&limit&status` → paginated `Message[]` with `meta`.
- `GET /messages/:id` → `Message`.
- `POST /messages` → `Message` (enqueues `email.jobs` / `sms.jobs` server-side).

## UI / components used

- `features/message/components/{message-compose-form,message-list}`.
- `shared/components/ui/*` (`card`, `table`, `input`, `textarea`, `field`, `select`,
  `button`, `badge`, `skeleton`, `empty`, `toaster`); widgets `pager`;
  `shared/hooks/use-pagination`; `DashboardLayout`; `lucide-react` icons.

## Acceptance criteria

- [ ] `/messages` renders behind the auth guard (unauthenticated → `/sign-in`).
- [ ] The compose form validates with Zod, posts to `POST /messages` with the chosen channel,
      invalidates the list, and shows success/error via toast; server field errors map onto
      inputs.
- [ ] The message list paginates and filters by `status` (counts from `meta`) and renders
      channel + delivery-status badges.
- [ ] Loading, error, and empty states render on the form and list.
- [ ] All HTTP flows through `api/message.api` + `queries/message.queries`; responses typed via
      `types/message.types`; no `fetch` in components.
- [ ] All labels localized (en/ru/uz).
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
