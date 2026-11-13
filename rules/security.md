# Security & context (HIGH)

## security-authn-in-transport

Authentication (who is calling) is transport middleware registered in `app/`. It verifies the token/signature and puts a `Principal` in `ctx`. Webhook transports verify the provider signature before anything else.

## security-authz-in-usecase

Authorization (may they do this) is a business rule, so it lives in useCase.

**Wrong** — role check in the handler; a second transport (gRPC, consumer) silently skips it:
```go
if !principal.HasRole("admin") { http.Error(w, "forbidden", 403); return }
```

**Right**
```go
func (u *UseCase) ApproveTimesheet(ctx context.Context, id string) error {
    p := platform.PrincipalFrom(ctx)
    ts, err := u.storage.Get(ctx, id)
    if err != nil { return err }
    if !p.Can("timesheet:approve") || ts.CompanyID != p.CompanyID {
        return timesheets.ErrForbidden
    }
    // ...
}
```

Complex policies can sit behind an `authz` port declared by the domain; the useCase still decides when to call it.

## security-context-propagation

Principal, tenant ID, request ID, and trace context travel in `ctx`, set and read only through `platform` helpers:

```go
platform.WithPrincipal(ctx, p) / platform.PrincipalFrom(ctx)
platform.WithTenant(ctx, id)   / platform.TenantFrom(ctx)
platform.RequestID(ctx)
```

- Never pass these as loose string parameters through every method.
- Background jobs and consumers establish a system principal and tenant explicitly at the transport boundary.

## security-tenant-scoping

In a multi-tenant system, every storage query on tenant data is scoped:

- Storage methods take the tenant from `ctx` (or an explicit `tenantID` parameter) and **fail closed** if missing.
- No storage method returns rows across tenants unless it is explicitly a system/admin method, named as such and reachable only from a system principal.
- Contract tests include a cross-tenant leak case.

## security-secrets-config

- Secrets come from `platform` config backed by env/secret manager, injected into adapters via setup.
- Never hard-code keys, never log them, never put them in DTOs that get serialized.
- Input from transports is untrusted: bound request body sizes, validate shape, use parameterized queries in storage only.
