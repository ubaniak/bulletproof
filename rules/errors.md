# Errors (HIGH)

Errors cross every layer, so they need one model. The flow:

```
driver/vendor error ──(adapter _dto translates)──▶ domain error ──(useCase decides)──▶ domain error
                                                                       │
                                     transport maps once ──▶ HTTP status / gRPC code / ack-nack
```

## errors-domain-defined

Each domain declares its errors in `X/errors`, built on `platform` error kinds so transports can map generically.

```go
// platform/errors.go
type Kind int
const (KindInvalid Kind = iota + 1; KindNotFound; KindConflict; KindUnauthorized; KindForbidden; KindRateLimited; KindUnavailable; KindInternal)
type Error struct{ Kind Kind; Code, Msg string; Err error }

// payments/errors.go
var (
    ErrCardDeclined = platform.New(platform.KindInvalid, "card_declined", "card was declined")
    ErrRateLimited  = platform.New(platform.KindRateLimited, "provider_rate_limited", "payment provider busy")
)
```

`errors` is part of the domain's public API: other domains may check `errors.Is(err, payments.ErrCardDeclined)`.

## errors-adapters-translate

Storage and gateway adapters translate driver/vendor errors in their `_dto` (or adapter) into domain errors. Raw vendor types never reach useCase.

**Wrong**
```go
return nil, err // *stripe.Error escapes into useCase
```

**Right**
```go
// gateway/stripe_dto/errors.go
func TranslateError(err error) error {
    var se *stripe.Error
    if errors.As(err, &se) {
        switch se.Code {
        case stripe.ErrorCodeCardDeclined: return payments.ErrCardDeclined.Wrap(err)
        case stripe.ErrorCodeRateLimit:    return payments.ErrRateLimited.Wrap(err)
        }
    }
    return platform.Unavailable(err)
}
```

Both `gateway/stripe` and `gateway/braintree` now fail the same way for the same situation — the LSP requirement. Wrap the original error for logs; never expose it to callers.

## errors-usecase-decides

Retry, fallback, and compensation are business decisions. The useCase makes them, on domain errors:

```go
res, err := u.primary.Charge(ctx, c)
if errors.Is(err, payments.ErrRateLimited) {
    res, err = u.fallback.Charge(ctx, c)
}
```

## errors-transport-maps

Each transport has exactly one function mapping errors to its protocol:

```go
func (h *Handler) writeError(w http.ResponseWriter, err error) {
    status, body := httperr.From(err) // Kind -> status, Code/Msg -> envelope; unknown -> 500 + generic msg
    h.writeJSON(w, status, body)
}
```

| Kind | HTTP | gRPC | Consumer |
|---|---|---|---|
| Invalid | 400/422 | InvalidArgument | ack + dead-letter |
| NotFound | 404 | NotFound | ack + dead-letter |
| Conflict | 409 | AlreadyExists/Aborted | ack (idempotent) |
| Unauthorized | 401 | Unauthenticated | dead-letter |
| Forbidden | 403 | PermissionDenied | dead-letter |
| RateLimited | 429 | ResourceExhausted | nack + backoff |
| Unavailable | 503 | Unavailable | nack + backoff |
| Internal/unknown | 500 | Internal | nack + backoff |

Never leak internal messages or stack traces in responses.

## errors-expected-vs-unexpected

- **Expected** (Invalid, NotFound, Conflict, Forbidden): normal control flow. Return them; log at debug/info at most.
- **Unexpected** (Unavailable, Internal): wrap with context at the adapter, log **once** at the transport boundary with request ID, return a generic message.
- Do not log and return at every layer — one error, one log line.
