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
        pg_dto                # row <-> entity translation
        mongodb               # adapter
        mongodb_dto           # doc <-> entity translation
    gateway                   # interface (port)
    gateway/
        api                   # api adapter
        api_dto               # external api wire types
        rest                  # rest adapter
        rest_dto              # external rest wire types
    usecase | service         # interface + implementation
    transport                 # interface (port)
    transport/
        rest                  # rest handler
        rest_dto              # inbound rest wire types
        grpc                  # grpc handler
        grpc_dto              # inbound grpc wire types
    setup                     # constructs and wires usecase + transport
```

**DTO naming**: every adapter `<impl>` has a sibling `<impl>_dto` for translation. Pattern is uniform across `storage/`, `gateway/`, `transport/`.

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
3. `X/storage/<impl>/` — implement. Add sibling `X/storage/<impl>_dto/` for row mapping.
4. `X/gateway.go` — declare interface for each external dependency.
5. `X/gateway/<impl>/` — implement. Add sibling `X/gateway/<impl>_dto/` for wire types.
6. `X/usecase.go` — declare interface + struct. Constructor takes storage + gateway interfaces.
7. `X/transport.go` — declare transport interface.
8. `X/transport/<impl>/` — implement handler. Add sibling `X/transport/<impl>_dto/` for inbound wire types. Take usecase interface in constructor.
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
- Storage method names encoding business rules (e.g. `GetEligibleUsers`) — split into neutral query + usecase composition
- Gateway interpreting error codes for retry/fallback — return raw, decide in usecase
- Transport doing domain validation (e.g. approved-email-domain checks) — only check shape/presence
- Usecase reaching past its storage interface to call `db.Query` directly
- Setup branching on env/feature flags — pass deps in instead

## Layer rules — examples

### Storage

**Allowed** — pure data retrieval:

```go
func (r *MySQLUserRepo) GetByID(ctx context.Context, id string) (*dto.UserRow, error) {
    return r.db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id)
}
```

**Violation** — business logic in storage:

```go
// Bad: "active premium users only" is a business rule, not a query concern
func (r *MySQLUserRepo) GetEligibleUsers(ctx context.Context) ([]*dto.UserRow, error) {
    return r.db.Query("SELECT * FROM users WHERE status = 'active' AND plan = 'premium' AND last_login > NOW() - INTERVAL 30 DAY")
}
```

Fix: expose `GetByStatus(status string)` and let the usecase compose the eligibility check.

### Gateway

**Allowed** — pure external call, raw response:

```go
func (g *StripeGateway) ChargeCard(ctx context.Context, req ChargeRequest) (*ChargeResponse, error) {
    return g.client.Post("/charges", req)
}
```

**Violation** — decision-making in gateway:

```go
// Bad: retry based on business meaning of error codes is a business rule
func (g *StripeGateway) ChargeCard(ctx context.Context, req ChargeRequest) (*ChargeResponse, error) {
    resp, err := g.client.Post("/charges", req)
    if err != nil && isRetryableBusinessError(err) {
        // retry with fallback payment method
    }
    return resp, err
}
```

Fix: return the raw error; let usecase decide whether to retry or fall back.

### Transport

**Allowed** — parse request, call usecase, format response:

```go
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var dto CreateUserDTO
    json.NewDecoder(r.Body).Decode(&dto)
    user, err := h.usecase.CreateUser(r.Context(), dto.Name, dto.Email)
    json.NewEncoder(w).Encode(toResponse(user))
}
```

**Violation** — business validation in transport:

```go
// Bad: "email must be from approved domain" is a business rule
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var dto CreateUserDTO
    json.NewDecoder(r.Body).Decode(&dto)
    if !strings.HasSuffix(dto.Email, "@company.com") {
        http.Error(w, "only company emails allowed", 400)
        return
    }
    // ...
}
```

Fix: move the domain check into usecase. Transport only validates field presence and well-formedness (not empty, valid email syntax).

### Usecase / Service

**Allowed** — orchestrates via interfaces, contains the business rule:

```go
func (u *UserUsecase) CreateUser(ctx context.Context, name, email string) (*entities.User, error) {
    if !isApprovedDomain(email) {
        return nil, ErrUnauthorizedDomain
    }
    existing, _ := u.storage.GetByEmail(ctx, email)
    if existing != nil {
        return nil, ErrAlreadyExists
    }
    return u.storage.Save(ctx, &entities.User{Name: name, Email: email})
}
```

**Violation** — raw DB call in usecase:

```go
// Bad: usecase directly querying the database
func (u *UserUsecase) CreateUser(ctx context.Context, name, email string) (*entities.User, error) {
    row := u.db.QueryRow("SELECT id FROM users WHERE email = ?", email)
    // ...
}
```

Fix: add `GetByEmail` to the storage interface and call that instead.

### Setup / Wiring

**Allowed** — pure dependency injection:

```go
func NewUserDomain(db *sql.DB, httpClient *http.Client) UserUsecaseInterface {
    storage := mysql.NewUserRepo(db)
    gateway := stripe.NewGateway(httpClient)
    return usecase.NewUserUsecase(storage, gateway)
}
```

**Violation** — logic in setup:

```go
// Bad: setup making a runtime decision
func NewUserDomain(db *sql.DB, env string) UserUsecaseInterface {
    if env == "prod" {
        // different logic path
    }
}
```

Fix: pass the right dependencies in from outside; setup just wires them.

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
