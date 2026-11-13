---
name: bulletproof
description: Bulletproof domain-driven backend architecture with enforced SOLID design. Use when scaffolding a new domain or service, reviewing layering/import boundaries, placing a file, designing errors, transactions, events, auth, tests, or API conventions for a backend, or when the user mentions "bulletproof", "domain layout", "useCase", "gateway", "storage adapter", "transport", "composition root", "SOLID", "SRP", "DIP", or asks how to structure a backend service.
---

# Bulletproof Backend Architecture

An opinionated blueprint for backend services. Code is split into self-contained **domains**, each with strict inbound/outbound boundaries, sitting on a small **platform** layer and assembled by one **app** composition root. **The layout structurally enforces SOLID** — see [rules/solid.md](rules/solid.md).

This file is the umbrella: layout, hard rules, and checklists. Detailed rules with wrong/right examples live in `rules/`. Load the rule file for the topic at hand rather than guessing.

Examples are Go. The layout is language-neutral — see [Language mapping](#language-mapping).

## Project layout

```
cmd/<binary>/main.go          # entrypoint: load config, call app.Run
app/                          # composition root: builds platform, calls every domain's setup,
                              # mounts transports, registers middleware, owns startup/shutdown
platform/                     # shared kernel: config, logger, tracer, db pool, tx manager, clock,
                              # id generator, base error types. Imports NO domain.
<domain>/
    entities/                 # domain types + invariants, no IO
    errors                    # domain error values/types (part of the public API)
    storage                   # interface (port), split per aggregate
    storage/
        <impl>/               # adapter (mysql, pg, mongodb, cached, ...)
        <impl>_dto/           # row/doc <-> entity translation
    gateway                   # interface (port), split per capability
    gateway/
        <impl>/               # adapter (stripe, quickbooks, ...)
        <impl>_dto/           # external wire types <-> entity translation
    publisher                 # interface (port) for outbound domain events (optional)
    useCase                   # interface + implementation — business rules only
    transport                 # interface (port)
    transport/
        <impl>/               # handler (rest, grpc, consumer, webhook, cron)
        <impl>_dto/           # request AND response wire types <-> entity translation
    setup                     # wires this domain's graph; returns (UseCase, Transport)
```

- Every adapter `<impl>` has a sibling `<impl>_dto`. Uniform across `storage/`, `gateway/`, `transport/`.
- The business-logic layer is always called **`useCase`**. Do not mix `service` and `useCase` in one codebase.
- A domain's **public API** is exactly: `entities`, `errors`, and the `useCase` interface. Nothing else is importable from outside.

## Dependency direction

```
            app ──▶ <domain>/setup ──▶ adapters (storage/<impl>, gateway/<impl>, transport/<impl>)
                                         │ implement
transport ──▶ useCase ──▶ storage | gateway | publisher (interfaces) ──▶ entities, errors
                 │
                 └──▶ other domain's useCase interface / entities / errors

platform ◀── imported by everyone; imports nothing above it
```

Interfaces are declared by the consumer (the domain). Adapters import the domain to implement them. `setup` is the only place concrete and abstract meet; `app` is the only place domains meet each other.

## Rules by priority

| Priority | Category | Impact | Prefix |
|---|---|---|---|
| 1 | Boundaries & structure | CRITICAL | `structure-` |
| 2 | Layer responsibilities | CRITICAL | `layer-` |
| 3 | Errors | HIGH | `errors-` |
| 4 | Consistency (transactions, events) | HIGH | `consistency-` |
| 5 | Security & context | HIGH | `security-` |
| 6 | Testing | HIGH | `testing-` |
| 7 | Cross-cutting concerns | MEDIUM | `crosscutting-` |
| 8 | Query & API design | MEDIUM | `api-` |
| 9 | Standards & tooling | MEDIUM | `standards-` |

### 1. Boundaries & structure (CRITICAL) — [rules/structure.md](rules/structure.md)
- `structure-cross-domain-imports` — Domain A may import only B's `entities`, `errors`, `useCase` interface.
- `structure-public-api` — Each domain exposes one public surface; everything else is internal (Go `internal/`, TS `index.ts` + lint).
- `structure-platform-layer` — Cross-cutting infrastructure lives in `platform/`; platform never imports a domain.
- `structure-app-root` — `app/` is the single composition root: routes, middleware, lifecycle.
- `structure-data-ownership` — A domain owns its tables and migrations. No cross-domain joins or table reads.
- `structure-single-purpose` — One domain, one purpose. Nest only per the nesting rules.
- `structure-no-cycles` — No cyclic deps between domains; extract a shared entities domain or invert via events.

### 2. Layer responsibilities (CRITICAL) — [rules/layers.md](rules/layers.md)
- `layer-entities-pure` — Entities hold invariants, no IO, never reference DTOs.
- `layer-storage-neutral` — Storage persists and retrieves; no business rules in method names or queries.
- `layer-gateway-dumb` — Gateway calls third parties and translates; it decides nothing.
- `layer-usecase-rules-only` — Business rules only; no SQL, HTTP, or protocol types.
- `layer-transport-shape-only` — Transport validates shape/presence, calls useCase, maps errors; no domain validation.
- `layer-setup-wiring-only` — Setup constructs and wires; no env/flag branching.
- `layer-dto-no-leak` — DTOs never cross their adapter boundary.

### 3. Errors (HIGH) — [rules/errors.md](rules/errors.md)
- `errors-domain-defined` — Each domain declares its error types in `errors`.
- `errors-adapters-translate` — Adapters translate vendor/driver errors into domain errors; raw vendor errors never reach useCase.
- `errors-usecase-decides` — Retry/fallback/compensation decisions live in useCase, made on domain errors.
- `errors-transport-maps` — Each transport maps domain errors to status codes in one place.
- `errors-expected-vs-unexpected` — Expected errors are returned, not logged as failures; unexpected errors are wrapped and logged once.

### 4. Consistency (HIGH) — [rules/consistency.md](rules/consistency.md)
- `consistency-tx-port` — Transactions via a `TxManager` port carried in `ctx`; useCase never sees `*sql.Tx`.
- `consistency-one-domain-per-tx` — A DB transaction never spans domains.
- `consistency-outbox` — Cross-domain or external side effects after a commit go through an outbox / domain events.
- `consistency-idempotency` — Every inbound retryable entry point (webhook, consumer, POST with retries) is idempotent.

### 5. Security & context (HIGH) — [rules/security.md](rules/security.md)
- `security-authn-in-transport` — Authentication is transport middleware.
- `security-authz-in-usecase` — Authorization is a business rule; it lives in useCase.
- `security-context-propagation` — Principal, tenant, request ID travel in `ctx` via `platform` helpers.
- `security-tenant-scoping` — Storage requires tenant scope on every multi-tenant query.
- `security-secrets-config` — Secrets come from config/secret store, injected via setup; never literals.

### 6. Testing (HIGH) — [rules/testing.md](rules/testing.md)
- `testing-usecase-fakes` — useCase unit tests run against in-memory fakes of its ports.
- `testing-adapter-contract` — One contract suite per port; every adapter must pass it (this is how LSP is enforced).
- `testing-transport` — Transport tests cover wire ↔ entity and error → status mapping.
- `testing-e2e-through-setup` — A thin layer of end-to-end tests built through `setup`/`app`.

### 7. Cross-cutting concerns (MEDIUM) — [rules/crosscutting.md](rules/crosscutting.md)
- `crosscutting-decorators` — Caching, metrics, tracing, retries-with-backoff are decorator adapters wired in setup.
- `crosscutting-middleware` — Request logging, auth, rate limiting, recovery are transport middleware registered in `app`.
- `crosscutting-async-transports` — Queue consumers, webhooks, cron jobs are transports like REST/gRPC.

### 8. Query & API design (MEDIUM) — [rules/api.md](rules/api.md)
- `api-neutral-filters` — Storage accepts neutral filter/criteria objects; useCase supplies the business values.
- `api-pagination` — Cursor pagination with one shared shape.
- `api-error-envelope` — One error response envelope across all endpoints.
- `api-versioning` — Breaking wire changes get a new transport version, not an edited one.
- `api-read-models` — Cross-domain reporting goes through a dedicated read-model/query domain fed by events.

### 9. Standards & tooling (MEDIUM) — [rules/standards.md](rules/standards.md)
- `standards-boundary-lint` — Import rules are enforced by a linter in CI, not only by review.
- `standards-naming` — File, package, and type naming conventions.
- `standards-strict-tooling` — Formatter, linter, strict types, pinned deps, pre-commit hooks, CI gates.

SOLID mapping and violation signs: [rules/solid.md](rules/solid.md).

## Scaffold checklist

When creating a new domain `X`:

1. `X/entities/` — core types with constructor-enforced invariants. Imports only `platform` primitives.
2. `X/errors` — declare the domain's error values/types.
3. `X/storage` — one small interface per aggregate the useCase needs.
4. `X/storage/<impl>/` + `<impl>_dto/` — implement; translate rows and driver errors.
5. `X/gateway` — one small interface per external capability.
6. `X/gateway/<impl>/` + `<impl>_dto/` — implement; translate wire types and vendor errors.
7. `X/publisher` (if X emits events) — interface; adapter writes to the outbox.
8. `X/useCase` — interface + struct; constructor takes only interfaces.
9. `X/transport` + `X/transport/<impl>/` + `<impl>_dto/` — handlers take the useCase interface; one error-mapping function per transport.
10. `X/setup` — build the concrete graph from injected platform deps; return `(UseCase, Transport)`.
11. Tests: useCase with fakes, contract suite for each port, transport mapping tests.
12. Migrations for X's tables live under X (or are namespaced to X).
13. Register X in `app/` (call setup, mount transport). Add X to the boundary-lint config.

## Review checklist

Flag:

**Boundaries**
- Domain importing another domain's `storage/`, `gateway/`, `transport/`, or adapter DTOs
- Domain reading or joining another domain's tables
- `platform/` importing a domain
- Routes/middleware registered outside `app/`; useCase constructed in `main` instead of `setup`
- Cyclic dependency between domains; multi-purpose domain

**Layers**
- DTO types in useCase signatures or entities
- SQL/HTTP/protocol code or `database/sql`, `net/http` imports in useCase
- useCase importing a concrete adapter, or constructor taking `*sql.DB` / `*http.Client`
- Storage method names encoding business rules (`GetEligibleUsers`)
- Gateway making retry/fallback decisions based on business meaning
- Transport doing domain validation (approved email domains, plan limits)
- Setup branching on env/feature flags

**Errors**
- Vendor/driver error types (Stripe, pq, Mongo) reaching useCase
- Status-code mapping scattered across handlers
- Expected errors (not found, validation) logged at error level; errors logged at more than one layer

**Consistency & security**
- `*sql.Tx` passed into useCase; transaction spanning two domains
- External call or cross-domain write after commit without outbox/idempotency
- Webhook/consumer handler that is not idempotent
- Authorization in transport; query missing tenant scope; secret literal in code

**Tests**
- New adapter not run against the port's contract suite
- useCase test requiring a real DB

## Anti-patterns

- **Anemic entities** — bags of fields with no invariants. Validate in entity constructors.
- **Leaky DTO / leaky vendor** — returning a `mysql_dto` row or a `*stripe.Error` from an adapter. Translate at the boundary.
- **God useCase** — one useCase orchestrating many domains. Split, or create a coordinator domain ([rules/structure.md](rules/structure.md)).
- **Transport shortcut** — handler calling storage directly. Always go through useCase.
- **Shared mutable entity** — two domains mutating the same entity type. Each domain owns its entities.
- **Cross-domain join** — reading another domain's tables for a report. Use a read model.
- **Remote call inside a DB transaction** — use the outbox.
- **Growing `platform/utils`** — business logic hiding in the shared kernel. Find the owning domain.

## Quick reference

| Layer | Role | Direction | Owns |
|---|---|---|---|
| platform | shared kernel | — | infra primitives |
| entities | domain types | — | invariants |
| errors | domain errors | — | error contract |
| storage | persistence port | outbound | durable state |
| gateway | 3rd-party port | outbound | external IO |
| publisher | event port | outbound | emitted events |
| useCase | business logic | center | rules, authz, retries |
| transport | inbound port | inbound | protocol IO, error mapping |
| setup | domain wiring | — | domain graph |
| app | composition root | — | process lifecycle |

## Language mapping

| Concept | Go | TypeScript |
|---|---|---|
| Port | `storage.go` interface in domain package | `storage.ts` exported `interface` |
| Adapter | `storage/mysql/` package | `storage/mysql/` folder |
| Public API | non-`internal/` packages | `index.ts` re-exports only |
| Boundary lint | `go-arch-lint`, `depguard` | `eslint-plugin-boundaries`, `dependency-cruiser` |
| Context | `context.Context` | `AsyncLocalStorage` or explicit `ctx` param |
