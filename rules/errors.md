# Errors (HIGH)

Errors cross every layer, so they need one model. It is defined in terms of **kinds**, not language mechanics: Go returns error values, Java/C#/Python/TS throw exceptions, some codebases use `Result` types. The language file says which; the rules are the same.

```
driver/vendor failure ──(adapter translator)──▶ domain error ──(useCase decides)──▶ domain error
                                                                       │
                                  transport maps once ──▶ HTTP status / gRPC code / ack-nack
```

## errors-domain-defined

`platform` defines the kinds. Each domain declares its errors in `X/errors`, each with a kind and a stable machine-readable code.

```
platform.ErrorKind = Invalid | NotFound | Conflict | Unauthorized | Forbidden
                   | RateLimited | Unavailable | Internal

payments.errors:
  CardDeclined  = DomainError(kind: Invalid,     code: "card_declined")
  RateLimited   = DomainError(kind: RateLimited, code: "provider_rate_limited")
```

`errors` is part of the domain's public API: other domains may check for `payments.CardDeclined`.

- Error-value languages: sentinel values / typed errors + an "is" check.
- Exception languages: one base `DomainError` exception carrying `kind` and `code`; subclasses per error.
- Result languages: a sealed/union error type per domain.

## errors-adapters-translate

Storage and gateway adapters translate driver/vendor failures in their translator into domain errors. Vendor error/exception types never reach useCase.

**Wrong**
```
charge(ctx, c)
  return stripeClient.createCharge(...)     # StripeError propagates into useCase
```

**Right**
```
stripe_dto.translateError(e: StripeError) -> DomainError
  match e.code
    "card_declined" -> payments.CardDeclined(cause: e)
    "rate_limit"    -> payments.RateLimited(cause: e)
    otherwise       -> platform.Unavailable(cause: e)
```

Both `gateway/stripe` and `gateway/braintree` now fail the same way for the same situation — the LSP requirement. Keep the original as the cause for logs; never expose it to callers.

In exception languages, the adapter catches **only** vendor/driver exceptions and rethrows domain ones. Do not catch-all in useCase.

## errors-usecase-decides

Retry, fallback, and compensation are business decisions. The useCase makes them on domain errors:

```
result = primary.charge(ctx, c)
on payments.RateLimited:
  result = fallback.charge(ctx, c)
```

## errors-transport-maps

Each transport has exactly one error → response mapping (a function, an exception filter/handler, or middleware):

| Kind | HTTP | gRPC | Consumer |
|---|---|---|---|
| Invalid | 400/422 | InvalidArgument | ack + dead-letter |
| NotFound | 404 | NotFound | ack + dead-letter |
| Conflict | 409 | AlreadyExists/Aborted | ack (idempotent) |
| Unauthorized | 401 | Unauthenticated | dead-letter |
| Forbidden | 403 | PermissionDenied | dead-letter |
| RateLimited | 429 | ResourceExhausted | nack + backoff |
| Unavailable | 503 | Unavailable | nack + backoff |
| Internal / unknown | 500 | Internal | nack + backoff |

Never leak internal messages or stack traces in responses.

## errors-expected-vs-unexpected

- **Expected** (Invalid, NotFound, Conflict, Forbidden): normal control flow. Return/throw them; log at debug/info at most.
- **Unexpected** (Unavailable, Internal): wrap with context at the adapter, log **once** at the transport boundary with request ID, return a generic message.
- Do not log and rethrow at every layer — one error, one log line.
