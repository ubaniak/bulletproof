# Layer responsibilities (CRITICAL)

## layer-entities-pure

Entities are domain types plus invariants. No IO, no DTO references, no framework tags if avoidable.

**Wrong** — anemic bag of fields, validated somewhere else:
```go
type Timesheet struct{ Hours float64; WorkerID string }
```

**Right** — constructor enforces invariants:
```go
func NewTimesheet(workerID string, hours float64) (*Timesheet, error) {
    if workerID == "" { return nil, ErrInvalid("workerID required") }
    if hours < 0 || hours > 24 { return nil, ErrInvalid("hours out of range") }
    return &Timesheet{workerID: workerID, hours: hours}, nil
}
```

Two domains never mutate the same entity type. Each owns its own; translate at the seam.

## layer-storage-neutral

Storage persists and retrieves. It owns durable state and implements the domain's `storage` interface.

**Right** — pure data retrieval:
```go
func (r *MySQLUserRepo) GetByID(ctx context.Context, id string) (*entities.User, error) {
    var row dto.UserRow
    err := r.db.QueryRowContext(ctx, "SELECT id, name, email FROM users WHERE id = ?", id).Scan(&row.ID, &row.Name, &row.Email)
    if errors.Is(err, sql.ErrNoRows) { return nil, users.ErrNotFound }
    if err != nil { return nil, platform.Unavailable(err) }
    return dto.ToEntity(row), nil
}
```

**Wrong** — business rule baked into the query/method name:
```go
// "active premium users seen in 30 days" is a business rule
func (r *MySQLUserRepo) GetEligibleUsers(ctx context.Context) ([]*entities.User, error) {
    return r.query("SELECT * FROM users WHERE status='active' AND plan='premium' AND last_login > NOW() - INTERVAL 30 DAY")
}
```

Fix: a neutral filter (see `api-neutral-filters`) — the useCase decides the values, storage builds the SQL:
```go
users, err := u.storage.List(ctx, users.Filter{Status: "active", Plan: "premium", LastLoginAfter: u.clock.Now().AddDate(0, 0, -30)})
```

Do **not** "fix" this by loading every row and filtering in memory.

## layer-gateway-dumb

Gateway makes outbound calls to third parties and translates. It decides nothing.

**Right** — call, translate wire types and errors into domain terms:
```go
func (g *StripeGateway) Charge(ctx context.Context, c entities.Charge) (*entities.ChargeResult, error) {
    resp, err := g.client.Charges.New(dto.ToStripeParams(c))
    if err != nil { return nil, dto.TranslateError(err) } // -> payments.ErrCardDeclined, ErrRateLimited, ErrUnavailable...
    return dto.ToEntity(resp), nil
}
```

**Wrong** — business decision in the gateway:
```go
if err != nil && isRetryableBusinessError(err) {
    // retry with fallback payment method  <- business decision
}
```

**Also wrong** — returning the raw `*stripe.Error` to useCase. That leaks the vendor into the domain and makes `gateway/stripe` and `gateway/braintree` non-substitutable (LSP).

Transport-level resilience that respects the contract (timeouts, transparent retry on connection reset) is allowed, preferably as a decorator (`crosscutting-decorators`).

## layer-usecase-rules-only

useCase holds business rules and orchestration via interfaces. No SQL, HTTP, or protocol types.

**Right**
```go
func (u *UserUseCase) CreateUser(ctx context.Context, name, email string) (*entities.User, error) {
    if err := u.authz.Can(ctx, "user:create"); err != nil { return nil, err }
    if !isApprovedDomain(email) { return nil, users.ErrUnauthorizedDomain }
    if _, err := u.storage.GetByEmail(ctx, email); err == nil {
        return nil, users.ErrAlreadyExists
    } else if !errors.Is(err, users.ErrNotFound) {
        return nil, err
    }
    user, err := entities.NewUser(name, email)
    if err != nil { return nil, err }
    return user, u.storage.Save(ctx, user)
}
```

**Wrong**
```go
row := u.db.QueryRow("SELECT id FROM users WHERE email = ?", email) // useCase touching the DB
```

Fix: add `GetByEmail` to the storage interface.

## layer-transport-shape-only

Transport (REST, gRPC, consumers, webhooks, cron) translates wire → entity, calls useCase, translates entity → wire, maps errors.

**Right**
```go
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var in dto.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&in); err != nil || in.Email == "" {
        h.writeError(w, platform.Invalid("name and email required"))
        return
    }
    user, err := h.useCase.CreateUser(r.Context(), in.Name, in.Email)
    if err != nil { h.writeError(w, err); return } // single mapping function, see errors.md
    h.writeJSON(w, http.StatusCreated, dto.ToUserResponse(user))
}
```

**Wrong** — domain validation in transport:
```go
if !strings.HasSuffix(in.Email, "@company.com") { http.Error(w, "only company emails", 400); return }
```

Transport checks only shape and presence: required fields, types, syntax (valid email format, UUID format). `_dto` holds both request **and** response types.

Never call storage directly from a handler "to save a round trip".

## layer-setup-wiring-only

**Right** — pure dependency injection, deps passed in:
```go
func Setup(p *platform.Platform, payments payments.UseCase) (UseCase, Transport) {
    storage := mysql.NewOrderRepo(p.DB)
    uc := NewUseCase(storage, payments, p.Tx, p.Clock)
    return uc, rest.NewHandler(uc)
}
```

**Wrong**
```go
if env == "prod" { /* different logic path */ }
```

Fix: `app/` chooses which adapter to pass (e.g. a fake gateway in local dev); setup just wires.

## layer-dto-no-leak

DTOs exist only to translate between entities and an underlying representation (row, vendor payload, wire format). A `mysql_dto` type in a useCase signature, an entity field typed as a DTO, or a vendor struct returned from a gateway are all leaks. Translate at the adapter boundary.
