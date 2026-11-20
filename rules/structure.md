# Boundaries & structure (CRITICAL)

## structure-cross-domain-imports

Domain A may depend only on B's `entities`, `errors`, and `service` port. Never B's `storage`, `gateway`, `setup`, or any translator. No domain depends on `transport/` at all.

**Wrong**
```
# orders/service
import payments.storage.mysql          # reaching into another domain's persistence
```

**Right**
```
# orders/service
import payments.service as PaymentsService   # port only
import payments.entities, payments.errors

service OrdersService(payments: PaymentsService, ...)   # injected by app/
```

`app/` builds payments first, then passes its service into `orders.setup(...)`.

## structure-public-api

A domain's public surface is exactly `entities`, `errors`, the `service` port, and any published event types. Transports consume this surface and nothing else. Everything else must be **unreachable** from outside the domain — by the language's visibility rules where possible, otherwise by a boundary linter in CI. If neither the compiler nor a linter stops it, the rule will be broken. See the language file for the mechanism.

## structure-transport-layer

Transport is not part of a domain. It sits at the project root beside `platform/` and `app/`, grouped by protocol and then by domain:

```
transport/
    rest/<domain>       rest/<domain>_dto
    cmd/<domain>        cmd/<domain>_dto        # CLI commands
    grpc/<domain>       grpc/<domain>_dto
    consumer/<domain>   webhook/<domain>   cron/<domain>   ...
```

- `transport/<protocol>/<domain>` may depend on `platform` and **that domain's public API only** (`entities`, `errors`, `service` port). It never touches `storage`, `gateway`, `setup`, or another domain.
- A handler that needs several domains belongs to a coordinator domain's transport (see Application coordinators), not to one that reaches across.
- No domain depends on `transport/`. Deleting `transport/` must leave every domain compiling and its tests green.
- Transports do not depend on each other. Shared wire primitives for a protocol (error mapper, error envelope, pagination shape, middleware primitives) live in `platform/<protocol>`, e.g. `platform/rest`; they know error kinds, never domains.
- An endpoint or command that composes several domains (a dashboard, a combined view) belongs to a **coordinator domain** with its own `transport/<protocol>/<coordinator>`. A thin, read-only coordinator that owns no tables is expected and fine; the alternative is a transport reaching across domains, which is not allowed.
- Adding a protocol or a domain's exposure means adding a folder under `transport/`; no domain changes.

**Wrong**
```
orders/transport/rest        # transport inside the domain
```

**Right**
```
transport/rest/orders        # depends on orders' service port, entities, errors, platform
```

## structure-platform-layer

`platform/` holds infrastructure every domain needs and no domain owns:

- config loading, secret access
- logger, tracer, metrics
- DB pool, `TxManager` implementation
- clock, ID generator
- error kinds (`Invalid`, `NotFound`, `Conflict`, `Unauthorized`, `Forbidden`, `RateLimited`, `Unavailable`, `Internal`)
- ctx helpers (principal, tenant, request ID)
- server plumbing, middleware primitives, and per-protocol wire primitives (`platform/rest`: error mapper, error envelope, pagination shape)

Rules:
- `platform/` **never** depends on a domain.
- No business rules in `platform/`. If it mentions an order, a user, or a timesheet, it belongs in a domain.
- Keep it small. A growing `platform/utils` is a smell — find the owning domain.

## structure-app-root

`app/` is the single composition root for the process:

```
app.run(config):
  p = platform.build(config)                        # db, logger, tx, clock, ...
  paymentsSvc = payments.setup(p, config.stripe)
  ordersSvc   = orders.setup(p, paymentsSvc)

  server = p.httpServer(middleware: [recover, requestId, logging, authn(p.auth)])
  server.mount("/v1/payments", transport.rest.payments.handler(paymentsSvc))
  server.mount("/v1/orders",   transport.rest.orders.handler(ordersSvc))
  server.run()                                      # owns graceful shutdown
```

- Route prefixes, middleware order, server start/stop, worker pools, and consumer subscriptions are decided here.
- Per-domain `setup` builds the domain and returns its service; `app` decides how the process exposes it by building each `transport/<protocol>/<domain>` handler from that service.
- The entrypoint only loads config and calls `app.run`. Multiple processes (api, worker, cli) share `app` helpers but mount different transports (`transport/rest`, `transport/consumer`, `transport/cmd`).
- With a DI container, `app/` and `setup` are the only places that register or resolve components.

## structure-data-ownership

- Each domain owns its tables/collections and their migrations.
- No other domain reads, writes, or joins those tables — not even "just a quick read-only query".
- Need data from B? Call B's service. Need it at report scale? Build a read model (`api-read-models`).
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

When one workflow genuinely orchestrates several domains (e.g. "close pay period" touches timesheets, payroll, and exports), create a dedicated **coordinator domain** (`payperiodclose/`) whose service depends on the other domains' service ports and owns no tables beyond its own workflow state. It follows every rule above. This is the fix for a God service.
