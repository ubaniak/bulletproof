# Boundaries & structure (CRITICAL)

## structure-cross-domain-imports

Domain A may depend only on B's `entities`, `errors`, and `useCase` port. Never B's `storage`, `gateway`, `transport`, `setup`, or any translator.

**Wrong**
```
# orders/useCase
import payments.storage.mysql          # reaching into another domain's persistence
```

**Right**
```
# orders/useCase
import payments.useCase as PaymentsUseCase   # port only
import payments.entities, payments.errors

useCase OrdersUseCase(payments: PaymentsUseCase, ...)   # injected by app/
```

`app/` builds payments first, then passes its useCase into `orders.setup(...)`.

## structure-public-api

A domain's public surface is exactly `entities`, `errors`, the `useCase` port, and any published event types. Everything else must be **unreachable** from outside the domain — by the language's visibility rules where possible, otherwise by a boundary linter in CI. If neither the compiler nor a linter stops it, the rule will be broken. See the language file for the mechanism.

## structure-platform-layer

`platform/` holds infrastructure every domain needs and no domain owns:

- config loading, secret access
- logger, tracer, metrics
- DB pool, `TxManager` implementation
- clock, ID generator
- error kinds (`Invalid`, `NotFound`, `Conflict`, `Unauthorized`, `Forbidden`, `RateLimited`, `Unavailable`, `Internal`)
- ctx helpers (principal, tenant, request ID)
- server plumbing, middleware primitives

Rules:
- `platform/` **never** depends on a domain.
- No business rules in `platform/`. If it mentions an order, a user, or a timesheet, it belongs in a domain.
- Keep it small. A growing `platform/utils` is a smell — find the owning domain.

## structure-app-root

`app/` is the single composition root for the process:

```
app.run(config):
  p = platform.build(config)                        # db, logger, tx, clock, ...
  paymentsUC, paymentsT = payments.setup(p, config.stripe)
  ordersUC,   ordersT   = orders.setup(p, paymentsUC)

  server = p.httpServer(middleware: [recover, requestId, logging, authn(p.auth)])
  server.mount("/v1/payments", paymentsT)
  server.mount("/v1/orders",   ordersT)
  server.run()                                      # owns graceful shutdown
```

- Route prefixes, middleware order, server start/stop, worker pools, and consumer subscriptions are decided here.
- Per-domain `setup` builds the domain; `app` decides how the process exposes it.
- The entrypoint only loads config and calls `app.run`. Multiple processes (api, worker) share `app` helpers but mount different transports.
- With a DI container, `app/` and `setup` are the only places that register or resolve components.

## structure-data-ownership

- Each domain owns its tables/collections and their migrations.
- No other domain reads, writes, or joins those tables — not even "just a quick read-only query".
- Need data from B? Call B's useCase. Need it at report scale? Build a read model (`api-read-models`).
- Foreign keys across domain tables are discouraged; store the ID and treat the other domain as the source of truth.
- ORMs: a domain's ORM models/relations must not reference another domain's models.

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

Never break a cycle by letting one side reach into the other's storage.

## Application coordinators

When one workflow genuinely orchestrates several domains (e.g. "close pay period" touches timesheets, payroll, and exports), create a dedicated **coordinator domain** (`payperiodclose/`) whose useCase depends on the other domains' useCase ports and owns no tables beyond its own workflow state. It follows every rule above. This is the fix for a God useCase.
