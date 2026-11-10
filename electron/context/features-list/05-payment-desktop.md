# Feature 05 — Payments & Invoices (Desktop)

- Feature ID: FEAT-05
- Status: draft
- Depends on: FEAT-01 (shell), FEAT-02 (auth — guarded dashboard surface)

> Source of truth for the authenticated **payments** surface: a payments list with create +
> refund, and an invoices list with a pay action, inside the desktop `DashboardLayout`.
> Consumes the backend `payment` domain.
>
> HONEST STATE: **to build.** `src/renderer/src/features/payment/*` holds only `.gitkeep`.

## Goal

Give authenticated users a payments surface on `/payments`: list payments (filter by
status), create a payment, refund a payment, and list + pay invoices — all through
TanStack Query against the backend payment API via the typed client.

## Scope

- **Payments page** (`features/payment/pages/payments-page`, route `/payments` under
  `DashboardLayout`) — orchestrates the payments list, the create form, and the invoices
  list; owns data fetching.
- **Payment list** (`components/payment-list`) — paginated, filterable by `status`; per-row
  refund action.
- **Payment create form** (`components/payment-create-form`) — Zod-validated
  (`validators/payment.validator`) amount/currency/description; create mutation.
- **Invoice list** (`components/invoice-list`) — paginated, filterable by `status`; per-row
  Pay action.
- **Queries** (`queries/payment.queries`) — `usePayments`, `useCreatePayment`,
  `useRefundPayment`, `useInvoices`, `usePayInvoice`; mutations invalidate the relevant
  lists. Typed via `types/payment.types` (`Payment`, `Invoice`, `CreatePaymentDto`,
  `PaymentListQuery`, `InvoiceListQuery`); fetched via `api/payment.api`.
- Loading / error / empty / success states; status badges via existing semantic tokens.
- Out of scope: a real payment-gateway integration, checkout/redirect flows (any external
  redirect opens in the OS browser via `shell.openExternal`), webhooks (server-side).

## Consumed API

Base `/api/v1`, envelope `{ success, data, meta? }`; Bearer access token attached by the client.

- `GET /payments?page&limit&status` → paginated `Payment[]` with `meta`.
- `POST /payments` → `Payment`.
- `POST /payments/:id/refund` (`{ amount }`) → `Payment`.
- `GET /invoices?page&limit&status` → paginated `Invoice[]`.
- `PATCH /invoices/:id/pay` → `Invoice`.

## UI / components used

- `features/payment/components/{payment-list,payment-create-form,invoice-list}`.
- `shared/components/ui/*` (`table`, `card`, `input`, `field`, `select`, `button`,
  `dialog`/`alert-dialog` for refund/pay confirm, `badge`, `skeleton`, `empty`, `toaster`);
  widgets `pager`; `shared/hooks/usePagination`; `DashboardLayout`; `lucide-react` icons.

## Acceptance criteria

- [ ] `/payments` renders behind the auth guard (unauthenticated → `/sign-in`).
- [ ] The payment list paginates and filters by `status` (counts from `meta`).
- [ ] Creating a payment validates with Zod, posts to `POST /payments`, invalidates the list,
      and shows success/error via toast; server field errors map onto inputs.
- [ ] Refunding a payment confirms, calls `POST /payments/:id/refund`, and refetches.
- [ ] The invoice list paginates/filters by `status`; Pay calls `PATCH /invoices/:id/pay` and
      refetches.
- [ ] Loading, error, and empty states render on every list/form.
- [ ] All HTTP flows through `api/payment.api` + `queries/payment.queries`; responses typed via
      `types/payment.types`; no `fetch` in components.
- [ ] All labels localized (en/ru/uz).
- [ ] Tokens-only styling, a11y, secure IPC (contextIsolation on / nodeIntegration off), lint,
      typecheck, build, and tests pass.

## Tasks

<!-- Left for /parse-features to decompose. -->
