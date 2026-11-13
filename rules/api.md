# Query & API design (MEDIUM)

## api-neutral-filters

Storage must not encode business rules, but it also must not force filtering in memory. Use neutral criteria objects declared in the domain:

```go
type UserFilter struct {
    Status         *string
    Plan           *string
    LastLoginAfter *time.Time
    Page           platform.PageRequest
}
type UserStorage interface {
    List(ctx context.Context, f UserFilter) (platform.Page[*entities.User], error)
}
```

- The useCase owns the **values** ("eligible = active + premium + seen in 30 days").
- Storage owns the **translation** to SQL/indexes.
- If a query needs a business-named method for performance, name it after the data shape, not the rule (`ListByStatusAndPlan`, not `GetEligibleUsers`).

## api-pagination

- Cursor-based pagination for lists, with one shared shape from `platform`:
  - request: `limit`, `cursor`
  - response: `items`, `next_cursor` (absent at end)
- Enforce a max `limit` in transport.
- Never return unbounded lists.

## api-error-envelope

All HTTP endpoints return errors in one envelope produced by the transport error mapper:

```json
{ "error": { "code": "card_declined", "message": "card was declined", "request_id": "..." } }
```

`code` is the stable, machine-readable domain error code. `message` is safe for clients. No internals.

## api-versioning

- Wire contracts (`transport/<impl>_dto`) are versioned by route prefix (`/v1`) or package (`rest/v1`, `rest/v2`).
- Breaking change → add a new version adapter alongside the old one (Open/Closed). Both call the same useCase.
- Additive changes (new optional fields) do not need a new version.
- Published event payloads follow the same rules.

## api-read-models

Cross-domain reporting (e.g. "hours by project by payroll status") must not break `structure-data-ownership` with cross-domain joins. Options, in order:

1. Compose in a coordinator useCase by calling each domain's useCase (fine for small result sets).
2. A dedicated **read-model domain** (`reporting/`) that subscribes to events and maintains its own denormalized tables. It owns those tables and exposes query useCases.

Reporting domains are read-only from the rest of the system's perspective; they never write back into source domains.
