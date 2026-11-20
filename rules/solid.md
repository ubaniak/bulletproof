# SOLID enforcement

Bulletproof is opinionated SOLID. Each principle maps to a structural rule the layout already enforces.

## S — Single Responsibility

Each layer has one reason to change:

- **entities** change when domain invariants change
- **storage** changes when persistence engine/schema changes
- **gateway** changes when a third-party contract changes
- **service** changes when business rules change
- **transport** changes when a wire protocol changes
- **setup** changes when domain wiring changes
- **app** changes when process composition changes (routes, middleware, workers)
- **platform** changes when shared infrastructure changes

**Violation signs**
- service handling both billing and notifications → split into two domains or a coordinator
- storage adapter formatting JSON for an HTTP response → that belongs in transport
- entity opening DB connections → entity is pure data + invariants
- one file holding service + storage + transport → split into layers

## O — Open / Closed

Add a new adapter, never modify existing ones:

- New DB? Add `storage/<newengine>/` + `<newengine>_dto/`. Don't edit `mysql/`.
- New protocol or API version? Add `transport/<newproto>/<domain>` + `<domain>_dto`. Don't edit `transport/rest/`.
- New external provider? Add `gateway/<newprovider>/` + `<newprovider>_dto/`. Don't edit `stripe/`.
- New cross-cutting behaviour? Add a decorator adapter (`storage/cached/`). Don't edit the wrapped adapter.

service depends on the port, so adding adapters never touches business logic.

**Violation signs**
- if/else in service switching on `db.kind == "mysql"`
- shared mutable adapter base class everyone extends
- changing the storage port every time a new adapter joins (port should be stable; adapters conform)

## L — Liskov Substitution

Every adapter must be a true substitute for its port:

- `storage/mysql` and `storage/pg` return identical **domain** errors for identical inputs
- `gateway/stripe` and `gateway/braintree` translate vendor failures into the same domain errors (see errors.md); one cannot throw on a missing field while the other silently defaults
- Adapter-specific behaviour (timeouts, transparent retries that respect the contract) is fine; differing semantics is not

Enforced by: the per-port contract suite every adapter must pass (testing.md).

**Violation signs**
- service tests pass with one adapter and fail with another
- docs say "only use `mysql` adapter for X" — a leaked contract; fix the port
- adapter returns/throws an error type not declared by the domain
- vendor error or exception types reaching service

## I — Interface Segregation

Keep ports small and role-specific. One fat `Repository` port is an anti-pattern.

- Split storage by aggregate: `UserStorage`, `OrderStorage`, not `Storage`
- Split gateway by capability: `PaymentGateway`, `RefundGateway`, not `StripeClient`
- Split handlers by group; no god-handler

service depends only on the methods it calls.

**Violation signs**
- service depends on a port with 20 methods and uses 2
- fakes/mocks with stubs for unused methods
- adding a method to a port breaks unrelated services

## D — Dependency Inversion

The whole layout exists to enforce this. High-level (service) depends on abstractions (ports) declared in the domain. Low-level (mysql, stripe, rest) implements them.

- Ports live with the **consumer** (the domain), not with the adapter
- Adapters depend on the domain to implement the port
- service never depends on `storage/mysql` — only on the `storage` port
- `setup` is the only place concrete and abstract meet; `app` is the only place domains meet

**Violation signs**
- service or entities depend on an IO library or framework (DB driver/ORM, HTTP, vendor SDK, framework annotations)
- domain port declared in the adapter's module
- service receives a connection, client, session, or transaction object
- circular dependency between domain and adapter (port on the wrong side)

## SOLID review checklist

- **SRP**: layer doing two jobs, file doing two layers' work
- **OCP**: type switches on adapter kind, modifying an existing adapter to add a feature
- **LSP**: adapter semantics differ from the contract; adapter not run against the contract suite
- **ISP**: ports with methods no caller uses; fat repository
- **DIP**: service depending on a concrete adapter or IO library, port declared in the adapter module, concrete types in service constructor
