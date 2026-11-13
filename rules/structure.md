# Boundaries & structure (CRITICAL)

## structure-cross-domain-imports

Domain A may import only B's `entities`, `errors`, and `useCase` interface. Never B's `storage`, `gateway`, `transport`, `setup`, or any `_dto`.

**Wrong**
```go
import "app/payments/storage/mysql" // orders reaching into payments persistence
```

**Right**
```go
import (
    "app/payments"          // useCase interface + errors
    "app/payments/entities"
)

type OrdersUseCase struct {
    payments payments.UseCase // injected by app/
}
```

Cross-domain dependencies are injected by `app/`: it calls `payments.Setup(...)`, then passes the returned useCase into `orders.Setup(...)`.

## structure-public-api

A domain's public surface is exactly `entities`, `errors`, and the `useCase` interface. Make everything else unimportable:

- Go: put adapters under `X/internal/` (e.g. `X/internal/storage/mysql`), or enforce with `go-arch-lint`.
- TS: `X/index.ts` re-exports only the public surface; lint forbids deep imports (`X/storage/**`) from outside `X`.

If the compiler or linter does not stop it, the rule will be broken.

## structure-platform-layer

`platform/` holds infrastructure every domain needs and no domain owns:

- config loading, secret access
- logger, tracer, metrics registry
- DB pool, `TxManager` implementation
- clock, ID generator
- base error kinds (`NotFound`, `Conflict`, `Invalid`, `Unauthorized`, `Forbidden`, `Unavailable`)
- context helpers (principal, tenant, request ID)
- HTTP/gRPC server plumbing, middleware primitives

Rules:
- `platform/` **never** imports a domain.
- No business rules in `platform/`. If it mentions an order, a user, or a timesheet, it belongs in a domain.
- Keep it small. A growing `platform/utils` is a smell — find the owning domain.

## structure-app-root

`app/` is the single composition root for the process:

```go
func Run(ctx context.Context, cfg platform.Config) error {
    p := platform.New(cfg)                         // db, logger, tx, clock...
    paymentsUC, paymentsT := payments.Setup(p, cfg.Stripe)
    ordersUC, ordersT := orders.Setup(p, paymentsUC)
    _ = ordersUC

    srv := p.HTTPServer(
        middleware.Recover, middleware.RequestID, middleware.Logging, middleware.Authn(p.Auth),
    )
    paymentsT.Mount(srv.Router("/v1/payments"))
    ordersT.Mount(srv.Router("/v1/orders"))
    return srv.Run(ctx) // owns graceful shutdown
}
```

- Route prefixes, middleware order, server start/stop, worker pools, and consumer subscriptions are decided here.
- Per-domain `setup` builds the domain; `app` decides how the process exposes it.
- `cmd/<binary>/main.go` only loads config and calls `app.Run`. Multiple binaries (api, worker) can share `app` helpers but mount different transports.

## structure-data-ownership

- Each domain owns its tables/collections and their migrations.
- No other domain reads, writes, or joins those tables — not even "just a quick read-only query".
- Need data from B? Call B's useCase. Need it at report scale? Build a read model (`api-read-models`).
- Foreign keys across domain tables are discouraged; store the ID and treat the other domain as the source of truth.

## structure-single-purpose

One domain = one purpose. Split when a domain has two independent reasons to change (e.g. billing and notifications).

### When to nest a sub-domain

Nest `X/Y/` only when all hold:
- `Y` has its own lifecycle (independent CRUD, separate persistence)
- `Y` is only meaningful inside `X`
- external callers never reference `Y` directly

Otherwise keep it flat as a sibling domain.

## structure-no-cycles

If A needs B and B needs A:
1. Extract the shared concept into its own domain both depend on, or
2. Invert one direction with events: A publishes, B subscribes (see `consistency-outbox`).

Never break a cycle by letting one side import the other's storage.

## Application coordinators

When one workflow genuinely orchestrates several domains (e.g. "close pay period" touches timesheets, payroll, and exports), create a dedicated **coordinator domain** (`payperiodclose/`) whose useCase depends on the other domains' useCase interfaces and owns no tables of its own beyond its workflow state. It follows every rule above. This is the fix for a God useCase.
