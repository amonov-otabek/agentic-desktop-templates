# Feature 03 — Analytics Dashboard (Desktop)

- Feature ID: FEAT-03
- Status: draft
- Depends on: FEAT-01 (shell), FEAT-02 (auth — this is a guarded dashboard surface)

> Source of truth for the authenticated **analytics** surface: a summary dashboard plus a
> paginated event explorer with charts, inside the desktop `DashboardLayout`. Consumes the
> backend `analytic` domain.
>
> HONEST STATE: **to build.** `src/renderer/src/features/analytic/*` holds only `.gitkeep`.

## Goal

Give authenticated users an analytics overview on `/analytics`: headline stat cards, a
time-series chart of events, and a paginated event list — all read through TanStack Query
against the backend analytic API via the typed client.

## Scope

- **Analytics page** (`features/analytic/pages/analytics-page`, route `/analytics` under
  `DashboardLayout`) — orchestrates the summary + chart + event list and owns data fetching.
- **Summary stat cards** (`components/stat-cards`) — headline counts from the dashboard summary.
- **Events chart** (`components/events-chart`) — time-series visualization using the chart
  token ramp (`--chart-1…5`); presentational, receives series via props.
- **Event list** (`components/event-list`) — paginated table of analytics events
  (`?page&limit`) using `shared/hooks/use-pagination` + the `pager` widget.
- **Queries** (`queries/analytics.queries`) — `useDashboard` (summary) and `useEvents`
  (paginated); typed via `types/analytics.types` (`DashboardSummary`, `AnalyticsEvent`,
  `EventListQuery`); data fetched via `api/analytics.api`.
- Every panel handles loading (skeleton) / error / empty / success states.
- Out of scope: writing/emitting analytics events (server-side), CSV export, custom date-range
  pickers, OS-native export dialogs.

## Consumed API

Base `/api/v1/analytics`, envelope `{ success, data, meta? }`; access token attached by the
client (Bearer in memory).

- `GET /analytics/dashboard` → `DashboardSummary` (headline metrics). **`admin`-only on the
  backend** — the page is role-gated client-side via `use-permission` (UX only; the backend
  enforces and returns `403` for non-admins).
- `GET /analytics/events?page&limit` → paginated `AnalyticsEvent[]` with `meta`.

## UI / components used

- `features/analytic/components/{stat-cards,events-chart,event-list}`.
- `shared/components/ui/*` (`card`, `chart`, `table`, `skeleton`, `empty`, `badge`); widgets
  `pager`; `shared/hooks/use-pagination`; `shared/hooks/use-permission` (gating);
  `DashboardLayout`; `lucide-react` icons.

## Acceptance criteria

- [ ] `/analytics` renders behind the auth guard (unauthenticated → `/sign-in`) and is
      role-gated via `use-permission` (the dashboard endpoint is `admin`-only on the backend).
- [ ] Stat cards render headline metrics from `GET /analytics/dashboard`.
- [ ] The events chart renders a time series using only chart tokens (`--chart-1…5`) — no
      hardcoded colors.
- [ ] The event list paginates via `?page&limit` and reads counts from `meta`
      (`total/page/limit`); the pager reflects `totalPages`.
- [ ] Loading (skeleton), error, and empty states render on every panel.
- [ ] Data flows through `queries/analytics.queries` + `api/analytics.api` only — no `fetch`
      in components; responses typed via `types/analytics.types`.
- [ ] All labels localized (en/ru/uz).
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
