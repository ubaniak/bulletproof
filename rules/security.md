# Security & context (HIGH)

## security-authn-in-transport

Authentication (who is calling) is transport middleware registered in `app/`. It verifies the token/signature and puts a `Principal` in ctx. Webhook transports verify the provider signature before anything else.

## security-authz-in-usecase

Authorization (may they do this) is a business rule, so it lives in useCase.

**Wrong** — role check in the handler (or a framework guard/annotation on the route) as the only check; a second transport (gRPC, consumer, cron) silently skips it:
```
if not principal.hasRole("admin"): return 403
```

**Right**
```
approveTimesheet(ctx, id)
  p  = platform.principal(ctx)
  ts = storage.get(ctx, id)
  if not p.can("timesheet:approve") or ts.companyId != p.companyId:
    fail timesheets.Forbidden
  ...
```

Coarse route guards in transport are fine as defence in depth, never as the only check. Complex policies can sit behind an `Authz` port declared by the domain; useCase still decides when to call it.

## security-context-propagation

Principal, tenant ID, request ID, trace context, cancellation, and the active transaction travel in **ctx**, set and read only through `platform` helpers:

```
platform.withPrincipal(ctx, p)  / platform.principal(ctx)
platform.withTenant(ctx, id)    / platform.tenant(ctx)
platform.requestId(ctx)
```

How ctx is carried is a language choice (explicit parameter, async-local storage, context variables, request-scoped DI) — see the language file. Rules that hold everywhere:
- Never thread these as loose string parameters through every method.
- Never read them from a global/static that is not request-scoped.
- Background jobs and consumers establish a system principal and tenant explicitly at the transport boundary.

## security-tenant-scoping

In a multi-tenant system, every storage query on tenant data is scoped:

- Storage takes the tenant from ctx (or an explicit `tenantId` parameter) and **fails closed** if missing.
- No storage method returns rows across tenants unless it is explicitly a system/admin method, named as such and reachable only from a system principal.
- ORM global query filters are acceptable as the mechanism, provided they fail closed.
- Contract tests include a cross-tenant leak case.

## security-secrets-config

- Secrets come from `platform` config backed by env/secret manager, injected into adapters via setup.
- Never hard-code keys, never log them, never put them in translator types that get serialized.
- Input from transports is untrusted: bound request body sizes, validate shape, use parameterized queries only.
