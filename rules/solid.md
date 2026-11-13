# SOLID enforcement

Bulletproof is opinionated SOLID. Each principle maps to a structural rule the layout already enforces.

## S — Single Responsibility

Each layer has one reason to change:

- **entities** change when domain invariants change
- **storage** changes when persistence engine/schema changes
- **gateway** changes when a third-party contract changes
- **useCase** changes when business rules change
- **transport** changes when a wire protocol changes
- **setup** changes when domain wiring changes
- **app** changes when process composition changes (routes, middleware, workers)
- **platform** changes when shared infrastructure changes

**Violation signs**
- useCase handling both billing and notifications → split into two domains or a coordinator
- storage adapter formatting JSON for an HTTP response → that belongs in transport
- entity opening DB connections → entity is pure data + invariants
- one file holding useCase + storage + transport → split into layers

## O — Open / Closed

Add a new adapter, never modify existing ones:

- New DB? Add `storage/<newengine>/` + `<newengine>_dto/`. Don't edit `mysql/`.
- New protocol or API version? Add `transport/<newproto>/` + `<newproto>_dto/`. Don't edit `rest/`.
- New external provider? Add `gateway/<newprovider>/` + `<newprovider>_dto/`. Don't edit `stripe/`.
- New cross-cutting behaviour? Add a decorator adapter (`storage/cached/`). Don't edit the wrapped adapter.

useCase depends on the interface, so adding adapters never touches business logic.

**Violation signs**
- if/else in useCase switching on `db.kind == "mysql"`
- shared mutable adapter base class everyone extends
- changing the storage interface every time a new adapter joins (interface should be stable; adapters conform)

## L — Liskov Substitution

Every adapter must be a true substitute for its interface:

- `storage/mysql` and `storage/pg` return identical **domain** errors for identical inputs
- `gateway/stripe` and `gateway/braintree` translate vendor failures into the same domain errors (see errors.md); one cannot throw on a missing field while the other silently defaults
- Adapter-specific behaviour (timeouts, transparent retries that respect the contract) is fine; differing semantics is not

Enforced by: the per-port contract suite every adapter must pass (testing.md).

**Violation signs**
- useCase tests pass with one adapter and fail with another
- docs say "only use `mysql` adapter for X" — a leaked contract; fix the interface
- adapter returns an error type not declared by the domain
- vendor error types reaching useCase

## I — Interface Segregation

Keep layer interfaces small and role-specific. One fat `Repository` interface is an anti-pattern.

- Split storage by aggregate: `UserStorage`, `OrderStorage`, not `Storage`
- Split gateway by capability: `PaymentGateway`, `RefundGateway`, not `StripeClient`
- Split transport by handler group; no god-handler interface

useCase depends only on the methods it calls.

**Violation signs**
- useCase imports an interface with 20 methods and uses 2
- fakes/mocks with stubs for unused methods
- adding a method to an interface breaks unrelated useCases

## D — Dependency Inversion

The whole layout exists to enforce this. High-level (useCase) depends on abstractions declared in the domain. Low-level (mysql, stripe, rest) implements them.

- Interfaces live with the **consumer** (the domain), not with the adapter
- Adapters import the domain to implement the interface
- useCase never imports `storage/mysql` — only `storage` (interface)
- `setup` is the only place concrete and abstract meet; `app` is the only place domains meet

**Violation signs**
- useCase imports `database/sql` or `net/http`
- domain interface lives in the adapter package
- useCase constructor takes a `*sql.DB` or `*sql.Tx`
- circular import between domain and adapter (interface on the wrong side)

## SOLID review checklist

- **SRP**: layer doing two jobs, file doing two layers' work
- **OCP**: type switches on adapter kind, modifying an existing adapter to add a feature
- **LSP**: adapter semantics differ from the contract; adapter not run against the contract suite
- **ISP**: interfaces with methods no caller uses; fat repository
- **DIP**: useCase importing a concrete adapter or driver, interface declared in adapter package, concrete types in useCase constructor
