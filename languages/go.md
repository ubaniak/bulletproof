# Bulletproof in Go

How to express the rules in `rules/` idiomatically in Go. Rules win over anything here.

## Layout

```
cmd/api/main.go                 # load config, app.Run
app/app.go
platform/{config,log,db,tx,errs,ctxkeys,clock}/
transport/                      # inbound adapters: protocol, then domain
    rest/timesheet/             rest/timesheet_dto/
    webhook/timesheet/          webhook/timesheet_dto/
    cmd/timesheet/              cmd/timesheet_dto/      # CLI commands
timesheet/
    entities/timesheet.go
    errors.go
    storage.go                  # port(s)
    gateway.go
    publisher.go
    service.go                 # Service port + impl
    setup.go
    internal/
        storage/mysql/            storage/mysql_dto/
        storage/memory/
        storage/contract/         # contract suite (test helper)
        gateway/quickbooks/       gateway/quickbooks_dto/
```

Storage and gateway adapters live under `internal/` so only `timesheet` (and its `setup`) can import them. Transports live outside the domain, so they can import only what `timesheet` exports: `Service`, entities, errors.

## Ports

```go
type Storage interface {
    GetByID(ctx context.Context, id string) (*entities.Timesheet, error)
    List(ctx context.Context, f Filter) (platform.Page[*entities.Timesheet], error)
    Save(ctx context.Context, ts *entities.Timesheet) error
}
```

Declared in the domain package (consumer side). Keep them small; split per aggregate/capability.

## Public API enforcement

- `internal/` for every adapter and translator.
- Exported identifiers in the domain root are the public API: `Service`, entities, errors, event types.
- `go-arch-lint` for the rest (below).

## Boundary lint

`.go-arch-lint.yml` (adjust globs to your module):
```yaml
version: 3
workdir: .
components:
  platform: { in: platform/** }
  app:      { in: app/** }
  transport: { in: transport/** }
  domain:   { in: [timesheet, payroll] }    # list domain root packages explicitly; "*" would overlap app, platform, transport
  entities: { in: [timesheet/entities, payroll/entities] }
  adapters: { in: [timesheet/internal/**, payroll/internal/**] }
deps:
  platform: { mayDependOn: [] }
  entities: { mayDependOn: [platform] }
  domain:   { mayDependOn: [platform, entities, domain] }
  adapters: { mayDependOn: [platform, entities, domain] }
  transport: { mayDependOn: [platform, entities, domain] }   # domain = public root package only (internal/ is unreachable)
  app:      { mayDependOn: [platform, transport, domain, entities] }
```

This is a sketch: shared components cannot say "a transport may import only *its own* domain". Enforce that by giving each domain its own components (`timesheet`, `timesheet_transport`, ...) or in review, and add each new domain to the lists above.

`golangci-lint` with `depguard` to deny IO packages in domain root and entities:
```yaml
depguard:
  rules:
    domain-core:
      files: ["**/entities/*.go", "**/service.go"]
      deny:
        - { pkg: database/sql, desc: "use a storage port" }
        - { pkg: net/http,     desc: "use a gateway/transport" }
        - { pkg: github.com/stripe, desc: "vendor SDKs only in gateway adapters" }
```

## Errors

Error values with kinds, checked with `errors.Is`/`errors.As`.

```go
// platform/errs
type Kind int
const (Invalid Kind = iota + 1; NotFound; Conflict; Unauthorized; Forbidden; RateLimited; Unavailable; Internal)

type Error struct{ Kind Kind; Code, Msg string; Err error }
func (e *Error) Error() string { return e.Code + ": " + e.Msg }
func (e *Error) Unwrap() error { return e.Err }
func (e *Error) Is(t error) bool { o, ok := t.(*Error); return ok && o.Code == e.Code }
func (e *Error) Wrap(err error) *Error { c := *e; c.Err = err; return &c }

// payments/errors.go
var ErrCardDeclined = &errs.Error{Kind: errs.Invalid, Code: "card_declined", Msg: "card was declined"}

// gateway/stripe_dto
func TranslateError(err error) error {
    var se *stripe.Error
    if errors.As(err, &se) && se.Code == stripe.ErrorCodeCardDeclined {
        return payments.ErrCardDeclined.Wrap(err)
    }
    return errs.Unavailable(err)
}
```

Transport: one `writeError(w, err)` per protocol in `platform/rest` that switches on `Kind`; every `transport/rest/<domain>` handler calls it.

## Context

`context.Context` as the first parameter of every port method and service method. Typed keys in `platform/ctxkeys`; accessors `platform.PrincipalFrom(ctx)`, `platform.TenantFrom(ctx)`.

## Transactions

```go
type TxManager interface {
    WithinTx(ctx context.Context, fn func(ctx context.Context) error) error
    Detached(ctx context.Context) context.Context   // ctx without the active *sql.Tx
}
```
The adapter stores `*sql.Tx` in ctx; storage adapters call `db.Conn(ctx)` which returns the tx if present. The service never sees `*sql.Tx`.

## Wiring

Manual constructor injection. No DI framework needed. `wire`/`fx` are acceptable only inside `setup`/`app`.

```go
func Setup(p *platform.Platform, payments payments.Service) Service {
    var store Storage = mysql.New(p.DB)
    store = cached.New(store, p.Cache)
    return newService(store, payments, p.Tx, p.Clock)
}

// app/app.go
svc := timesheet.Setup(p, paymentsSvc)
mux.Handle("/v1/timesheets", timesheetrest.NewHandler(svc))   // transport/rest/timesheet
```

## Testing

- Hand-written fakes in `internal/storage/memory`.
- Contract suite as `func RunStorage(t *testing.T, newStore func(t *testing.T) Storage)` in `internal/storage/contract`, called from each adapter's `_test.go`.
- Real DB via testcontainers-go for adapter tests.

## Naming & tooling

- Packages lowercase, no underscores except translator packages (`mysql_dto`). Files `snake_case.go`.
- `gofmt`/`goimports`, `golangci-lint` (`depguard`, `errorlint`, `wrapcheck`, `contextcheck`), `go-arch-lint`, `go mod tidy` check in CI.

## Traps

- Returning `sql.ErrNoRows` or `*pq.Error` from storage — translate.
- Package-level `var db *sql.DB` globals — inject.
- `init()` doing wiring — wiring belongs in `setup`.
