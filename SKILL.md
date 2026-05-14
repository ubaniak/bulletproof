---
name: bulletproof
description: Bulletproof domain-driven architecture pattern. Use when scaffolding a new domain, organizing backend services, reviewing layering/import boundaries, or when user mentions "bulletproof", "domain layout", "usecase", "gateway", "storage adapter", or asks how to structure a service with entities/storage/gateway/usecase/transport layers.
---

# Bulletproof Architecture

A domain-driven layout for backend services. Each domain is a self-contained unit with clear inbound/outbound boundaries.

## Layout

```
<domain>/
    entities/
        entities              # plain domain types, no IO
    storage                   # interface (port)
    storage/
        mysql                 # adapter
        mysql_dto             # row <-> entity translation
        pg                    # adapter
        mongodb               # adapter
    gateway                   # interface (port)
    gateway/
        api_dto               # external api wire types
        api                   # api adapter
        rest_dto              # external rest wire types
        rest                  # rest adapter
    usecase | service         # interface + implementation
    transport                 # interface (port)
    transport/
        rest_dto              # inbound wire types
        rest                  # rest handler
        grpc                  # grpc handler
    setup                     # constructs and wires usecase + transport
```

## Rules (hard)

1. **Cross-domain imports**: DomainA may import only `entities` and `usecase` of DomainB. Never storage/gateway/transport of another domain.
2. **DTOs are translators**: DTOs exist only to convert between public `entities` and an underlying service (DB row, API payload, wire format). Never leak DTOs across the layer boundary.
3. **Usecase = business logic**: usecase/service holds business rules only. No SQL, no HTTP, no protocol details.
4. **Single purpose**: One domain = one purpose. Nesting allowed when a sub-concept earns its own lifecycle.
5. **Storage = persistence**: Owns durable state. Implements the domain's `storage` interface.
6. **Gateway = third-party**: Outbound calls to external APIs. Implements the domain's `gateway` interface.
7. **Transport = inbound**: REST/gRPC handlers translate wire → entity, call usecase, translate entity → wire.
8. **Setup wires the graph**: Constructs concrete storage + gateway + usecase + transport. Returns usecase (for sibling domains) and transport (for the server).

## Dependency direction

```
transport ──▶ usecase ──▶ storage (interface) ◀── storage/mysql
                  │
                  └─────▶ gateway (interface) ◀── gateway/rest
                  │
                  └─────▶ entities
```

Interfaces defined in the domain. Adapters implement them. Usecase depends on interfaces only.

## Scaffold checklist

When creating a new domain `X`:

1. `X/entities/` — define core types. No imports from other layers.
2. `X/storage.go` (or equivalent) — declare interface the usecase needs.
3. `X/storage/<engine>/` — implement. Add `<engine>_dto` for row mapping.
4. `X/gateway.go` — declare interface for each external dependency.
5. `X/gateway/<provider>/` — implement. Add `<provider>_dto` for wire types.
6. `X/usecase.go` — declare interface + struct. Constructor takes storage + gateway interfaces.
7. `X/transport.go` — declare transport interface.
8. `X/transport/<proto>/` — implement handler. Take usecase interface in constructor.
9. `X/setup.go` — build concrete graph, return `(Usecase, Transport)`.

## Review checklist

When reviewing code against this pattern, flag:

- Domain importing another domain's `storage/`, `gateway/`, or `transport/` package
- DTO types appearing in usecase signatures
- SQL/HTTP/protocol code inside usecase
- usecase importing concrete adapter instead of interface
- Entity referencing DTO type
- Missing `setup` — usecase constructed in `main` directly
- Multi-purpose domain (split it)
- Cyclic dep between domains (extract shared entities domain or invert)

## Anti-patterns

- **Anemic entities**: entities reduced to bags of fields with no invariants. Push validation into entity constructors.
- **Leaky DTO**: returning a `mysql_dto` row from usecase. Translate at the storage boundary.
- **God usecase**: one usecase orchestrating many domains. Split or introduce an application-layer coordinator.
- **Transport shortcut**: handler calling storage directly to "save a round trip". Always go through usecase.
- **Shared mutable entity**: two domains mutating the same entity type. Each domain owns its entities; translate at the seam.

## When to nest a sub-domain

Nest `X/Y/` when:
- `Y` has its own lifecycle (independent CRUD, separate persistence)
- `Y` is only meaningful inside `X` (otherwise lift to sibling)
- External callers never reference `Y` directly

Otherwise, keep flat.

## Quick reference

| Layer     | Role               | Direction | Owns          |
|-----------|--------------------|-----------|---------------|
| entities  | domain types       | —         | invariants    |
| storage   | persistence port   | outbound  | durable state |
| gateway   | 3rd-party port     | outbound  | external IO   |
| usecase   | business logic     | center    | rules         |
| transport | inbound port       | inbound   | protocol IO   |
| setup     | composition root   | —         | wiring        |
