# API Rules

Human-readable constraints for everything server-side. The machine-readable API
surface (envelope shape, auth model, endpoint registry) lives in `context/api-context.md`
— this file says how to build it. Builders read both before any backend task; QA
enforces both.

## The golden rules

1. **Validate at the boundary.** Every input (body, params, query, headers) is
   schema-validated before any logic runs. Invalid input fails fast with a clear
   message. Never trust external data.
2. **One envelope.** Every response uses the exact envelope defined in
   `context/api-context.md` — success and error alike. No bespoke shapes per endpoint.
3. **Auth before logic.** Protected operations check authentication AND ownership/
   role before touching business logic. Defaults to protected; public endpoints
   are explicitly marked.
4. **Thin handlers.** Route handlers parse, validate, authorize, delegate to the
   service layer, and shape the response. Business logic lives in services; SQL
   lives in the data layer.

## Resource design

- Nouns for resources, plural: `/api/projects`, `/api/projects/:id/members`.
- Standard verbs: GET (read), POST (create), PATCH (partial update), DELETE.
- List endpoints paginate from day one; define the scheme in `context/api-context.md`.
- Filtering/sorting via query params with validated allowlists.

## Status codes

| Code | Use |
| --- | --- |
| 200 / 201 | success / created |
| 400 | validation failure (details in error message) |
| 401 | not authenticated |
| 403 | authenticated but not allowed |
| 404 | resource not found (also for hiding others' resources) |
| 409 | conflict (duplicates, state machine violations) |
| 500 | unexpected — log details server-side, return generic message |

## Errors

- Client gets a user-friendly message; server log gets the full context
  (prefixed: `[feature/operation]`).
- Error messages never leak internals: no stack traces, SQL, or secrets in responses.
- Expected failures (validation, not-found, conflict) are flow, not exceptions.

## Database access

- Parameterized queries / ORM only — never string-built SQL.
- Mutations that span multiple writes run in transactions.
- Migrations are numbered, forward-only, and reviewed; schema changes always go
  through migrations, never manual edits.
- No raw DB records to the client — map to explicit response types.

## Do not

- Return different envelope shapes from different endpoints.
- Put business logic in route handlers or DB queries in UI code.
- Log request bodies that may contain credentials or personal data.
- Add an endpoint without registering it in `context/api-context.md`.
