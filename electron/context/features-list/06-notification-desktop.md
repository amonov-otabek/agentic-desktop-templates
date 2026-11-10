# Feature 06 — Notifications (Desktop)

- Feature ID: FEAT-06
- Status: draft
- Depends on: FEAT-01 (shell), FEAT-02 (auth — guarded dashboard surface)

> Source of truth for the authenticated **notifications** surface: an in-app notification
> feed with unread tracking + mark-read actions, a preferences form, and optional native OS
> notifications. Consumes the backend `notification` domain.
>
> HONEST STATE: **to build.** `src/renderer/src/features/notification/*` holds only `.gitkeep`.

## Goal

Give authenticated users a notifications surface on `/notifications`: read the feed, see the
unread count, mark one or all read, manage delivery preferences, and (optionally) surface
new notifications as native OS notifications — all through TanStack Query against the backend
notification API via the typed client.

## Scope

- **Notifications page** (`features/notification/pages/notifications-page`, route
  `/notifications` under `DashboardLayout`) — orchestrates the feed + preferences; owns
  data fetching.
- **Notification list** (`components/notification-list`) — paginated feed with an unread-only
  filter; per-item Mark-read and a Mark-all-read action; visual read/unread state.
- **Unread count** — `useUnreadCount` drives a badge (dashboard header / user nav) and
  refreshes after mark-read.
- **Preferences form** (`components/preferences-form`) — channel/category toggles; loads
  current preferences and persists via PUT (optimistic where safe).
- **Native notifications (optional, if scoped)** — when an unseen notification arrives, the
  renderer requests a native OS notification through a typed preload bridge method
  (`window.api.notify.show({ title, body })` → main-side `new Notification(...)` over the
  `notification:show` IPC channel, payload validated in main). Clicking it can focus the
  window via `notification:focus`. Falls back silently when unsupported. The HTTP feed
  remains the source of truth; native banners are a presentation layer only.
- **Queries** (`queries/notification.queries`) — `useNotifications`, `useUnreadCount`,
  `usePreferences`, `useUpdatePreferences`, `useMarkRead`, `useMarkAllRead`; mutations
  invalidate the feed + unread count. Typed via `types/notification.types`
  (`AppNotification`, `NotificationPreferences`, `NotificationListQuery`); fetched via
  `api/notification.api`.
- Loading / error / empty / success states.
- Out of scope: real-time push/socket delivery (not wired in this template), per-notification
  deep links, badge counts on the OS dock/taskbar.

## Consumed API

Base `/api/v1/notifications`, envelope `{ success, data, meta? }`; Bearer access token
attached by the client. Native-notification display is a local IPC bridge, not an HTTP call.

- `GET /notifications?page&limit&unread_only` → paginated `AppNotification[]` with `meta`.
- `GET /notifications/unread-count` → `{ unread }`.
- `GET /notifications/preferences` → `NotificationPreferences`.
- `PUT /notifications/preferences` → `NotificationPreferences`.
- `PATCH /notifications/:id/read` → `AppNotification`.
- `PATCH /notifications/read-all` → `{ updated }`.

## UI / components used

- `features/notification/components/{notification-list,preferences-form}`.
- `shared/components/ui/*` (`card`, `table`/`item`, `switch`, `checkbox`, `badge`, `button`,
  `tabs`, `skeleton`, `empty`, `toaster`); widgets `pager`; `shared/hooks/usePagination`;
  `DashboardLayout`; `lucide-react` icons.

## Acceptance criteria

- [ ] `/notifications` renders behind the auth guard (unauthenticated → `/sign-in`).
- [ ] The feed paginates, supports an `unread_only` filter, and shows read/unread state.
- [ ] The unread-count badge reflects `GET /notifications/unread-count` and updates after
      mark-read / mark-all-read.
- [ ] Mark-read and mark-all-read call their endpoints and invalidate the feed + unread count.
- [ ] The preferences form loads current prefs and persists via `PUT`; success/error toast;
      optimistic update reverts on error.
- [ ] If native notifications are scoped: a new unseen notification triggers a native OS
      notification via the typed `window.api.notify.show` bridge (payload validated in main);
      it degrades gracefully when unsupported; the renderer never imports `electron` directly.
- [ ] Loading, error, and empty states render.
- [ ] All HTTP flows through `api/notification.api` + `queries/notification.queries`; responses
      typed via `types/notification.types`; no `fetch` in components.
- [ ] All labels localized (en/ru/uz).
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
