# bulletproof

A language-agnostic Claude Code skill encoding a domain-driven backend architecture: self-contained domains with entities, storage adapters, third-party gateways, and service logic, plus a top-level `transport/` layer (one folder per protocol, then per domain) for inbound handlers, on top of a small platform layer and a single app composition root.

## What it does

When this skill is installed, Claude Code will use the bulletproof layout to:

- Scaffold new domains with the correct layer structure, errors, and tests
- Review existing code against prioritized boundary, layer, error, consistency, and security rules
- Flag DTO/vendor leaks, layer violations, transports reaching into storage or other domains, cross-domain table access, and god-services
- Place transports in `transport/<protocol>/<domain>` (`rest`, `cmd`, `grpc`, `consumer`, `webhook`, `cron`), outside the domain
- Wire per-domain `setup` and the `app/` composition root correctly: `setup` returns the service, `app` builds and mounts the transports
- Enforce SOLID — each principle maps to a structural rule in the layout

The skill auto-triggers when you mention "bulletproof", "domain layout", "service layer", "service port", "gateway", "storage adapter", "transport", "composition root", or ask how to structure a backend service.

## Install

Clone into your Claude skills directory:

    git clone https://github.com/ubaniak/bulletproof ~/.claude/skills/bulletproof

Or symlink an existing checkout:

    ln -s /path/to/this/repo ~/.claude/skills/bulletproof

Restart Claude Code to pick up the new skill.

## Usage

- "Scaffold a new billing domain using bulletproof"
- "Review this code against the bulletproof layout"
- "Where should this DTO live?"
- "How should the QuickBooks webhook fit into bulletproof?"
- "Is it OK for the orders domain to import payments/storage?"
- "Where does the REST handler for orders go?"
- "Add a CLI command for billing"

## Files

- `SKILL.md` — umbrella: layout, prioritized rule index, scaffold and review checklists
- `rules/structure.md` — boundaries, public API, top-level transport layer, platform and app layers, data ownership, nesting (CRITICAL)
- `rules/layers.md` — responsibilities of entities, storage, gateway, service, transport, setup (CRITICAL)
- `rules/errors.md` — error model: domain errors, adapter translation, one shared error mapper per protocol (HIGH)
- `rules/consistency.md` — transactions (including `tx.detached` for cross-domain calls), outbox/events, idempotency (HIGH)
- `rules/security.md` — authn/authz placement, context propagation, tenant scoping, secrets (HIGH)
- `rules/testing.md` — service fakes, adapter contract suites, transport and e2e tests (HIGH)
- `rules/crosscutting.md` — decorators, middleware, observability, async transports (MEDIUM)
- `rules/api.md` — neutral filters, pagination, error envelope, versioning, read models (MEDIUM)
- `rules/standards.md` — boundary-lint dependency table (domains, transport, app), naming, tooling and CI gates (MEDIUM)
- `rules/contracts.md` — optional: contract-first specs in `api/`, generated code in `gen/`, remote peers as gateways, events (MEDIUM)
- `rules/manifest.md` — optional: `domain.yaml` and `bulletproof.yaml` manifests, scaffold outputs, hand-owned vs generated files (MEDIUM)
- `rules/solid.md` — SOLID mapping and violation signs
- `languages/*.md` — idiomatic layout, ports, errors, context, transactions, wiring, boundary-lint config, and framework traps for Go, TypeScript, Python, Java/Kotlin, and C#

Rules in `rules/` are language-neutral (pseudocode). Claude detects the stack from `go.mod`, `package.json`, `pyproject.toml`, `pom.xml`/`build.gradle`, or `*.csproj` and loads the matching language file. To add a language, copy an existing `languages/*.md`, answer the same sections, and add its marker to the table in `SKILL.md`.

## Optional: contracts and manifests

Teams that define OpenAPI/protobuf specs first and generate code can add `api/` (specs you edit) and `gen/` (generated, never edited). Generated types stay at the edge: only transports, gateways and event adapters import them, and entities are never generated from a wire spec. A peer that runs in another process is a gateway behind the consumer's own port. A `domain.yaml` per domain and a `bulletproof.yaml` per repo let a generator scaffold domains and wire processes (modular monolith, several processes, or one domain per binary). See `rules/contracts.md` and `rules/manifest.md`.

## Pattern summary

    <entrypoint>           # load config, call app.run
    app/                   # composition root: routes, middleware, lifecycle
    platform/              # shared kernel, imports no domain
        <protocol>/        # per-protocol wire primitives: error mapper, envelope, pagination (e.g. rest/)
    transport/             # inbound adapters: protocol, then domain
        rest/<domain>      # handler, calls the domain's service port
        rest/<domain>_dto  # wire <-> entity translation
        cmd/<domain>       # CLI commands (also grpc, consumer, webhook, cron)
    <domain>/
        entities/
        errors
        storage            # port
        storage/<impl>     # adapter
        storage/<impl>_dto # translation
        gateway            # port
        gateway/<impl>
        gateway/<impl>_dto
        publisher          # optional event port
        service            # business logic
        setup              # wiring, returns the service

Every adapter `<impl>` pairs with a sibling `<impl>_dto`; transport sits beside `platform`, not inside a domain. Each entry is a unit of code ownership: directory, package, module, or namespace, per the language file.

Hard rules:

- A domain may only depend on another domain's `entities`, `errors`, and `service` port.
- No domain depends on `transport/`. A transport depends only on `platform` and its own domain's public API (`entities`, `errors`, `service` port), never on `storage`, `gateway`, `setup`, another domain, or another transport.
- `setup` returns the service; `app` builds each transport from it and mounts it.
- An endpoint spanning several domains belongs to a thin coordinator domain with its own transport.

## License

MIT. See `LICENSE`.
