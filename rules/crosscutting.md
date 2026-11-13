# Cross-cutting concerns (MEDIUM)

Cross-cutting behaviour never goes inside business logic. It wraps it.

## crosscutting-decorators

Caching, metrics, tracing, circuit breaking, and contract-respecting retries are **decorator adapters**: they implement the same port and wrap another implementation. Setup composes them.

```go
// storage/cached/users.go
type Users struct{ next users.Storage; cache platform.Cache }

func (c *Users) GetByID(ctx context.Context, id string) (*entities.User, error) {
    if u, ok := c.cache.Get(ctx, "user:"+id); ok { return u.(*entities.User), nil }
    u, err := c.next.GetByID(ctx, id)
    if err == nil { c.cache.Set(ctx, "user:"+id, u, 5*time.Minute) }
    return u, err
}

// setup
var store users.Storage = mysql.New(p.DB)
store = instrumented.New(store, p.Metrics)
store = cached.New(store, p.Cache)
```

This is Open/Closed in practice: adding caching touches neither `mysql/` nor useCase. Decorators must pass the port's contract suite too.

Retry rule of thumb: retrying a transient network failure transparently is a decorator; deciding to retry or fall back because of what an error *means* is useCase.

## crosscutting-middleware

Request-scoped concerns are transport middleware registered once in `app/`:
- panic recovery
- request ID + trace context
- access logging
- authentication
- rate limiting, body-size limits, timeouts

Order is decided in `app/` and is the same for every domain's routes.

## crosscutting-observability

- Logger, tracer, and metrics come from `platform` and are injected; no global loggers inside domains.
- Logs are structured and include request ID, tenant, and principal from `ctx`.
- Business events worth counting (timesheet approved, payment failed) are emitted from useCase via an injected metrics port, not by parsing logs.

## crosscutting-async-transports

REST and gRPC are not the only inbound ports. These are all transports with their own `<impl>` + `<impl>_dto`, and they follow every transport rule (shape-only validation, call useCase, map errors):

```
X/transport/
    rest/       rest_dto/
    grpc/       grpc_dto/
    consumer/   consumer_dto/   # queue/topic messages
    webhook/    webhook_dto/    # inbound third-party callbacks (Stripe, QuickBooks)
    cron/                       # scheduled triggers; usually no DTO
```

- Webhook transports verify signatures, extract the idempotency key, translate the payload, call useCase. They do not contain sync logic.
- Consumers map error kinds to ack/nack/dead-letter (see errors.md).
- Cron handlers establish a system principal and call a useCase method; scheduling config lives in `app/`.
