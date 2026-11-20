---
name: bulletproof
description: Bulletproof domain-driven backend architecture with enforced SOLID design, for any backend language (Go, TypeScript, Python, Java/Kotlin, C#, ...). Use when scaffolding a new domain or service, reviewing layering/import boundaries, placing a file, designing errors, transactions, events, auth, tests, or API conventions for a backend, or when the user mentions "bulletproof", "domain layout", "useCase", "gateway", "storage adapter", "transport", "port", "adapter", "composition root", "SOLID", "SRP", "DIP", or asks how to structure a backend service.
---

# Bulletproof Backend Architecture

An opinionated, language-agnostic blueprint for backend services. Code is split into self-contained **domains**, each with strict inbound/outbound boundaries, sitting on a small **platform** layer and assembled by one **app** composition root. **The layout structurally enforces SOLID** — see [rules/solid.md](rules/solid.md).

This file is the umbrella: vocabulary, layout, rule index, and checklists. Detailed rules live in `rules/` (neutral, with pseudocode). Idiomatic code, tooling, and folder shapes per language live in `languages/`.

## Step 0 — detect the stack

Before scaffolding or reviewing, detect the language and load its file:

| Marker | Load |
|---|---|
| `go.mod` | [languages/go.md](languages/go.md) |
| `package.json` + `tsconfig.json` (or `.ts` sources) | [languages/typescript.md](languages/typescript.md) |
| `pyproject.toml`, `setup.cfg`, `requirements*.txt` | [languages/python.md](languages/python.md) |
| `pom.xml`, `build.gradle(.kts)` | [languages/java-kotlin.md](languages/java-kotlin.md) |
| `*.csproj`, `*.sln` | [languages/csharp.md](languages/csharp.md) |

No match: apply the rules using the pseudocode in `rules/`, and name the closest idiomatic equivalent for the language in use (how ports are expressed, how visibility is enforced, how errors and context flow). Rules in `rules/` win over language idioms; language files only say **how**, never **whether**.

## Vocabulary

| Term | Meaning |
|---|---|
| **domain** | A module with one purpose and one public surface |
| **entity** | Domain type + invariants; no IO |
| **port** | Abstract contract declared *by the domain* (interface, protocol, abstract class, trait, function type) |
| **adapter** | Concrete implementation of a port (`storage/<impl>`, `gateway/<impl>`, `transport/<impl>`) |
| **translator** (`<impl>_dto`) | Converts between an adapter's native format (row, vendor payload, wire message) and entities/domain errors |
| **useCase** | Business rules and orchestration; depends only on ports |
| **setup** | Per-domain composition root: builds the domain's object graph |
| **app** | Process composition root: builds platform, calls every setup, mounts transports, owns lifecycle |
| **platform** | Shared kernel of infrastructure primitives; depends on no domain |
| **ctx** | Request context: principal, tenant, request/trace ID, active transaction, cancellation |
| **domain error** | Typed failure with a **kind** (Invalid, NotFound, Conflict, ...) and a stable code — expressed as error values, exceptions, or `Result` depending on language |
| **IO library** | Any DB driver/ORM, HTTP client/server, message-broker SDK, cloud SDK, or vendor SDK |

## Project layout (conceptual)

```
<entrypoint>                  # load config, call app.run — nothing else
app/                          # composition root: routes, middleware, workers, startup/shutdown
platform/                     # config, logger, tracer, db pool, tx manager, clock, id generator,
                              # error kinds, ctx helpers. Depends on NO domain.
<domain>/
    entities                  # domain types + invariants, no IO
    errors                    # domain errors (public API)
    storage                   # port(s), split per aggregate
    storage/<impl>            # adapter (mysql, pg, mongodb, cached, memory, ...)
    storage/<impl>_dto        # row/doc <-> entity translator
    gateway                   # port(s), split per capability
    gateway/<impl>            # adapter (stripe, quickbooks, ...)
    gateway/<impl>_dto        # vendor wire <-> entity translator
    publisher                 # port for outbound domain events (optional)
    useCase                   # port + implementation — business rules only
    transport                 # port
    transport/<impl>          # handler (rest, grpc, consumer, webhook, cron)
    transport/<impl>_dto      # request AND response wire <-> entity translator
    setup                     # wires this domain; returns (useCase, transport)
```

Each entry is a **unit of code ownership**: it may be a directory, package, module, namespace, or file, as the language file specifies. What is fixed:

- Ports sit at the domain root; adapters and translators sit beneath their port.
- Every adapter `<impl>` has a sibling translator `<impl>_dto` (casing per language file).
- The business layer is always called **useCase**. Do not mix `service` and `useCase`.
- A domain's **public API** is exactly: `entities`, `errors`, the `useCase` port (and published event types). Everything else must be made unreachable from outside the domain by the language's visibility rules or a boundary linter.

## Dependency direction

```
            app ──▶ <domain>/setup ──▶ adapters (storage/<impl>, gateway/<impl>, transport/<impl>)
                                         │ implement
transport ──▶ useCase ──▶ storage | gateway | publisher (ports) ──▶ entities, errors
                 │
                 └──▶ other domain's useCase port / entities / errors

platform ◀── usable by everyone; depends on nothing above it
```

Ports are declared by the consumer (the domain). Adapters depend on the domain to implement them. `setup` is the only place concrete and abstract meet; `app` is the only place domains meet each other.

## Framework rule

Frameworks (Spring, NestJS, Express, Django, FastAPI, ASP.NET, Gin, ...) are adapters.

- Framework annotations, decorators, base classes, and request/response types are allowed in **transport, adapters, setup, app** only.
- **Never** in entities or useCase: no `@Transactional`, `@Injectable`, ORM model base classes, `Request`/`Response` objects, or framework exceptions.
- DI containers are configured only in `setup`/`app`. useCase gets its ports via constructor/parameters, never by looking them up.

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
- `structure-cross-domain-imports` — Domain A may depend only on B's `entities`, `errors`, `useCase` port.
- `structure-public-api` — Each domain exposes one public surface; everything else is unreachable from outside.
- `structure-platform-layer` — Infrastructure primitives live in `platform/`; platform depends on no domain.
- `structure-app-root` — `app/` is the single composition root: routes, middleware, lifecycle.
- `structure-data-ownership` — A domain owns its tables and migrations. No cross-domain joins or table reads.
- `structure-single-purpose` — One domain, one purpose. Nest only per the nesting rules.
- `structure-no-cycles` — No cyclic deps between domains; extract a shared domain or invert via events.

### 2. Layer responsibilities (CRITICAL) — [rules/layers.md](rules/layers.md)
- `layer-entities-pure` — Entities hold invariants, no IO, never reference translators or framework types.
- `layer-storage-neutral` — Storage persists and retrieves; no business rules in method names or queries.
- `layer-gateway-dumb` — Gateway calls third parties and translates; it decides nothing.
- `layer-usecase-rules-only` — Business rules only; depends on no IO library and no framework.
- `layer-transport-shape-only` — Transport validates shape/presence, calls useCase, maps errors; no domain validation.
- `layer-setup-wiring-only` — Setup constructs and wires; no env/flag branching.
- `layer-dto-no-leak` — Translator types never cross their adapter boundary.

### 3. Errors (HIGH) — [rules/errors.md](rules/errors.md)
- `errors-domain-defined` — Each domain declares its errors with a kind and stable code.
- `errors-adapters-translate` — Adapters translate vendor/driver failures into domain errors; vendor error types never reach useCase.
- `errors-usecase-decides` — Retry/fallback/compensation decisions live in useCase, made on domain errors.
- `errors-transport-maps` — Each transport maps domain errors to protocol responses in exactly one place.
- `errors-expected-vs-unexpected` — Expected errors are control flow; unexpected errors are logged once at the boundary.

### 4. Consistency (HIGH) — [rules/consistency.md](rules/consistency.md)
- `consistency-tx-port` — Transactions via a `TxManager` port; useCase never holds a connection/session/transaction object.
- `consistency-one-domain-per-tx` — A DB transaction never spans domains.
- `consistency-outbox` — Post-commit side effects (other domains, third parties) go through an outbox.
- `consistency-idempotency` — Every retryable inbound entry point is idempotent.

### 5. Security & context (HIGH) — [rules/security.md](rules/security.md)
- `security-authn-in-transport` — Authentication is transport middleware.
- `security-authz-in-usecase` — Authorization is a business rule; it lives in useCase.
- `security-context-propagation` — Principal, tenant, request ID travel in ctx via `platform` helpers.
- `security-tenant-scoping` — Storage requires tenant scope on every multi-tenant query; fail closed.
- `security-secrets-config` — Secrets come from config/secret store, injected via setup; never literals.

### 6. Testing (HIGH) — [rules/testing.md](rules/testing.md)
- `testing-usecase-fakes` — useCase unit tests run against in-memory fakes of its ports.
- `testing-adapter-contract` — One contract suite per port; every adapter must pass it (enforces LSP).
- `testing-transport` — Transport tests cover wire ↔ entity and error → response mapping.
- `testing-e2e-through-setup` — A thin layer of end-to-end tests built through `setup`/`app`.

### 7. Cross-cutting concerns (MEDIUM) — [rules/crosscutting.md](rules/crosscutting.md)
- `crosscutting-decorators` — Caching, metrics, tracing, transient retries are decorator adapters wired in setup.
- `crosscutting-middleware` — Request logging, authn, rate limiting, recovery are transport middleware registered in `app`.
- `crosscutting-observability` — Logger/tracer/metrics injected from platform; structured, ctx-aware.
- `crosscutting-async-transports` — Queue consumers, webhooks, cron jobs are transports like REST/gRPC.

### 8. Query & API design (MEDIUM) — [rules/api.md](rules/api.md)
- `api-neutral-filters` — Storage accepts neutral filter objects; useCase supplies the business values.
- `api-pagination` — Cursor pagination with one shared shape.
- `api-error-envelope` — One error response envelope across all endpoints.
- `api-versioning` — Breaking wire changes get a new transport version, not an edited one.
- `api-read-models` — Cross-domain reporting goes through a read-model domain fed by events.

### 9. Standards & tooling (MEDIUM) — [rules/standards.md](rules/standards.md)
- `standards-boundary-lint` — Dependency rules are enforced by a tool in CI, not only by review.
- `standards-naming` — Concept names are fixed; casing follows the language file.
- `standards-strict-tooling` — Formatter, linter, strict type checking, locked deps, pre-commit, CI gates.

SOLID mapping and violation signs: [rules/solid.md](rules/solid.md).

## Scaffold checklist

When creating a new domain `X` (concrete file names and syntax from the language file):

1. `X/entities` — core types with constructor/factory-enforced invariants. Depends only on platform primitives.
2. `X/errors` — the domain's errors (kind + stable code).
3. `X/storage` — one small port per aggregate the useCase needs.
4. `X/storage/<impl>` + `<impl>_dto` — implement; translate rows and driver failures.
5. `X/gateway` — one small port per external capability.
6. `X/gateway/<impl>` + `<impl>_dto` — implement; translate vendor payloads and failures.
7. `X/publisher` (if X emits events) — port; adapter writes to the outbox.
8. `X/useCase` — port + implementation; receives only ports.
9. `X/transport` + `X/transport/<impl>` + `<impl>_dto` — handlers receive the useCase port; one error-mapping function per transport.
10. `X/setup` — build the concrete graph from injected platform deps; return `(useCase, transport)`.
11. Tests: useCase with fakes, contract suite per port (run against every adapter incl. the fake), transport mapping tests.
12. Migrations for X's tables owned by X.
13. Register X in `app/` (call setup, mount transport). Add X to the boundary-lint config.

## Review checklist

Flag:

**Boundaries**
- Domain depending on another domain's `storage`, `gateway`, `transport`, `setup`, or translators
- Domain reading or joining another domain's tables
- `platform/` depending on a domain
- Routes/middleware registered outside `app/`; useCase constructed in the entrypoint instead of `setup`
- Cyclic dependency between domains; multi-purpose domain

**Layers**
- Translator types in useCase signatures or entities
- useCase or entities depending on any IO library or framework type (DB driver/ORM, HTTP, broker SDK, vendor SDK, framework annotations)
- useCase receiving a connection, client, session, or transaction object instead of a port
- useCase depending on a concrete adapter instead of a port
- Storage method names encoding business rules (`getEligibleUsers`)
- Gateway making retry/fallback decisions based on business meaning
- Transport doing domain validation (approved email domains, plan limits)
- Setup branching on env/feature flags; DI container lookups inside useCase

**Errors**
- Vendor/driver error or exception types reaching useCase
- Error → response mapping scattered across handlers
- Expected errors logged as failures; one error logged at multiple layers

**Consistency & security**
- Transaction object passed into useCase; transaction spanning two domains; framework transaction annotation on useCase
- Remote call inside a DB transaction; post-commit side effect without outbox
- Webhook/consumer handler that is not idempotent
- Authorization in transport; query missing tenant scope; secret literal in code

**Tests**
- New adapter not run against its port's contract suite
- useCase test requiring a real DB or network

## Anti-patterns

- **Anemic entities** — bags of fields with no invariants. Validate in entity constructors/factories.
- **Leaky translator / leaky vendor** — returning a DB row type or a vendor exception from an adapter. Translate at the boundary.
- **God useCase** — one useCase orchestrating many domains. Split, or create a coordinator domain ([rules/structure.md](rules/structure.md)).
- **Transport shortcut** — handler calling storage directly. Always go through useCase.
- **Shared mutable entity** — two domains mutating the same entity type. Each domain owns its entities.
- **Framework creep** — ORM models as entities, `@Transactional`/`@Injectable` on useCase, request objects passed into useCase.
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

## Pseudocode conventions (used in `rules/`)

```
port Name                      # abstract contract
  method(ctx, arg: Type) -> Result | ErrorName
adapter Impl implements Name   # concrete implementation
fail ErrorName                 # return/raise a domain error (language decides how)
try / catch VendorError        # adapter-side translation
```

Translate to the idiom in the language file; do not copy pseudocode literally.
