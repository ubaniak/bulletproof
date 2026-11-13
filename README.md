# bulletproof

A Claude Code skill encoding a domain-driven backend architecture: self-contained domains with entities, storage adapters, third-party gateways, useCase logic, and transport handlers, on top of a small platform layer and a single app composition root.

## What it does

When this skill is installed, Claude Code will use the bulletproof layout to:

- Scaffold new domains with the correct layer structure, errors, and tests
- Review existing code against prioritized boundary, layer, error, consistency, and security rules
- Flag DTO/vendor leaks, layer violations, cross-domain table access, and god-useCases
- Wire per-domain `setup` and the `app/` composition root correctly
- Enforce SOLID — each principle maps to a structural rule in the layout

The skill auto-triggers when you mention "bulletproof", "domain layout", "useCase", "gateway", "storage adapter", "composition root", or ask how to structure a backend service.

## Install

Clone into your Claude skills directory:

    git clone <this-repo-url> ~/.claude/skills/bulletproof

Or symlink an existing checkout:

    ln -s /path/to/this/repo ~/.claude/skills/bulletproof

Restart Claude Code to pick up the new skill.

## Usage

- "Scaffold a new billing domain using bulletproof"
- "Review this code against the bulletproof layout"
- "Where should this DTO live?"
- "How should the QuickBooks webhook fit into bulletproof?"
- "Is it OK for the orders domain to import payments/storage?"

## Files

- `SKILL.md` — umbrella: layout, prioritized rule index, scaffold and review checklists
- `rules/structure.md` — boundaries, public API, platform and app layers, data ownership, nesting (CRITICAL)
- `rules/layers.md` — responsibilities of entities, storage, gateway, useCase, transport, setup (CRITICAL)
- `rules/errors.md` — error model: domain errors, adapter translation, transport mapping (HIGH)
- `rules/consistency.md` — transactions, outbox/events, idempotency (HIGH)
- `rules/security.md` — authn/authz placement, context propagation, tenant scoping, secrets (HIGH)
- `rules/testing.md` — useCase fakes, adapter contract suites, transport and e2e tests (HIGH)
- `rules/crosscutting.md` — decorators, middleware, observability, async transports (MEDIUM)
- `rules/api.md` — neutral filters, pagination, error envelope, versioning, read models (MEDIUM)
- `rules/standards.md` — boundary linting, naming, tooling and CI gates (MEDIUM)
- `rules/solid.md` — SOLID mapping and violation signs

## Pattern summary

    cmd/<binary>/          # entrypoint
    app/                   # composition root: routes, middleware, lifecycle
    platform/              # shared kernel, imports no domain
    <domain>/
        entities/
        errors
        storage            # interface
        storage/<impl>     # adapter
        storage/<impl>_dto # translation
        gateway            # interface
        gateway/<impl>
        gateway/<impl>_dto
        publisher          # optional event port
        useCase            # business logic
        transport          # interface
        transport/<impl>   # rest, grpc, consumer, webhook, cron
        transport/<impl>_dto
        setup              # wiring

Every adapter `<impl>` pairs with a sibling `<impl>_dto`.

Hard rule: a domain may only import another domain's `entities`, `errors`, and `useCase` interface.
