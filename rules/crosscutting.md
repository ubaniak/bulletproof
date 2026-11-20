# Cross-cutting concerns (MEDIUM)

Cross-cutting behaviour never goes inside business logic. It wraps it.

## crosscutting-decorators

Caching, metrics, tracing, circuit breaking, and contract-respecting retries are **decorator adapters**: they implement the same port and wrap another implementation. Setup composes them.

```
adapter CachedUserStorage(next: UserStorage, cache: Cache) implements UserStorage
  getById(ctx, id)
    hit = cache.get(ctx, "user:" + id)
    if hit: return hit
    user = next.getById(ctx, id)
    cache.set(ctx, "user:" + id, user, ttl: 5m)
    return user

# setup
storage = MySQLUserStorage(p.db)
storage = InstrumentedUserStorage(storage, p.metrics)
storage = CachedUserStorage(storage, p.cache)
```

This is Open/Closed in practice: adding caching touches neither `mysql` nor useCase. Decorators must pass the port's contract suite too.

Language-provided interception (AOP, Python decorators, NestJS interceptors, middleware pipelines) is acceptable **only** when applied in setup/app and only to adapters — never to useCase or entities.

Retry rule of thumb: retrying a transient network failure transparently is a decorator; deciding to retry or fall back because of what an error *means* is useCase.

## crosscutting-middleware

Request-scoped concerns are transport middleware registered once in `app/`:
- panic/exception recovery
- request ID + trace context
- access logging
- authentication
- rate limiting, body-size limits, timeouts

Order is decided in `app/` and is the same for every domain's routes.

## crosscutting-observability

- Logger, tracer, and metrics come from `platform` and are injected; no global loggers configured inside domains.
- Logs are structured and include request ID, tenant, and principal from ctx.
- Business events worth counting (timesheet approved, payment failed) are emitted from useCase via an injected metrics port, not by parsing logs.

## crosscutting-async-transports

REST and gRPC are not the only inbound ports. These are all transports with their own `<impl>` + translator, following every transport rule (shape-only validation, call useCase, map errors):

```
X/transport/
    rest       rest_dto
    grpc       grpc_dto
    consumer   consumer_dto    # queue/topic messages
    webhook    webhook_dto     # inbound third-party callbacks (Stripe, QuickBooks)
    cron                       # scheduled triggers; usually no translator
```

- Webhook transports verify signatures, extract the idempotency key, translate the payload, call useCase. No sync logic in the handler.
- Consumers map error kinds to ack/nack/dead-letter (see errors.md).
- Cron handlers establish a system principal and call a useCase method; schedules are configured in `app/`.
