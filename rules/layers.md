# Layer responsibilities (CRITICAL)

Examples are pseudocode (see SKILL.md conventions). Idiomatic versions are in `languages/`.

## layer-entities-pure

Entities are domain types plus invariants. No IO, no translator references, no framework or ORM base classes/annotations.

**Wrong** — anemic bag of fields, validated somewhere else (or an ORM model used as the entity):
```
entity Timesheet { hours, workerId }
```

**Right** — constructor/factory enforces invariants:
```
Timesheet.create(workerId, hours) -> Timesheet | Invalid
  if workerId is empty:       fail Invalid("workerId required")
  if hours < 0 or hours > 24: fail Invalid("hours out of range")
  return Timesheet(workerId, hours)
```

Two domains never mutate the same entity type. Each owns its own; translate at the seam.

## layer-storage-neutral

Storage persists and retrieves. It owns durable state and implements the domain's storage port.

**Right** — pure retrieval, translate row and driver failure:
```
adapter MySQLUserStorage implements UserStorage
  getById(ctx, id) -> User | NotFound
    try
      row = db.queryOne(ctx, "SELECT id, name, email FROM users WHERE id = ?", id)
    catch DriverError e
      fail Unavailable(cause: e)
    if row is none: fail users.NotFound
    return mysql_dto.toEntity(row)
```

**Wrong** — business rule baked into the query/method name:
```
getEligibleUsers(ctx)   # "active premium users seen in 30 days" is a business rule
  db.query("... WHERE status='active' AND plan='premium' AND last_login > NOW() - 30 days")
```

Fix: a neutral filter (`api-neutral-filters`) — useCase decides the values, storage builds the query:
```
storage.list(ctx, UserFilter(status: "active", plan: "premium", lastLoginAfter: clock.now() - 30 days))
```

Do **not** "fix" this by loading every row and filtering in memory.

## layer-gateway-dumb

Gateway calls third parties and translates. It decides nothing.

**Right** — call, translate payload and failures into domain terms:
```
adapter StripePaymentGateway implements PaymentGateway
  charge(ctx, c: Charge) -> ChargeResult | CardDeclined | RateLimited | Unavailable
    try
      resp = stripeClient.createCharge(stripe_dto.toParams(c))
    catch StripeError e
      fail stripe_dto.translateError(e)
    return stripe_dto.toEntity(resp)
```

**Wrong** — business decision in the gateway:
```
catch StripeError e
  if isRetryableBusinessError(e): retry with fallback payment method   # business decision
```

**Also wrong** — letting `StripeError` escape to useCase. That leaks the vendor into the domain and makes `gateway/stripe` and `gateway/braintree` non-substitutable (LSP).

Transport-level resilience that respects the contract (timeouts, transparent retry on connection reset) is allowed, preferably as a decorator (`crosscutting-decorators`).

## layer-usecase-rules-only

useCase holds business rules and orchestration via ports. It depends on no IO library and no framework.

**Right**
```
useCase UserUseCase(storage: UserStorage, authz: Authz)
  createUser(ctx, name, email) -> User | Forbidden | UnauthorizedDomain | AlreadyExists
    authz.require(ctx, "user:create")
    if not isApprovedDomain(email): fail UnauthorizedDomain
    match storage.getByEmail(ctx, email)
      User      -> fail AlreadyExists
      NotFound  -> continue
    user = User.create(name, email)
    storage.save(ctx, user)
    return user
```

**Wrong**
```
createUser(ctx, name, email)
  row = db.query("SELECT id FROM users WHERE email = ?", email)   # useCase touching the DB
```

Fix: add `getByEmail` to the storage port.

## layer-transport-shape-only

Transport (REST, gRPC, consumers, webhooks, cron) translates wire → entity, calls useCase, translates entity → wire, maps errors.

**Right**
```
handler UserRestHandler(useCase: UserUseCase)
  POST /users (req)
    input = rest_dto.parseCreateUser(req.body)       # shape: required fields, types, email syntax
    if input invalid: return mapError(Invalid(input.problems))
    result = useCase.createUser(req.ctx, input.name, input.email)
    on error e: return mapError(e)                    # single mapping function, see errors.md
    return 201, rest_dto.toUserResponse(result)
```

**Wrong** — domain validation in transport:
```
if not input.email.endsWith("@company.com"): return 400 "only company emails"
```

Transport checks only shape and presence: required fields, types, syntax. Translators hold both request **and** response types. Never call storage directly from a handler "to save a round trip".

## layer-setup-wiring-only

**Right** — pure dependency injection, deps passed in:
```
orders.setup(p: Platform, payments: PaymentsUseCase) -> (OrdersUseCase, OrdersTransport)
  storage = MySQLOrderStorage(p.db)
  uc      = OrdersUseCase(storage, payments, p.tx, p.clock)
  return uc, OrdersRestHandler(uc)
```

**Wrong**
```
if env == "prod": different logic path
```

Fix: `app/` chooses which adapter to pass (e.g. a fake gateway in local dev); setup just wires.

## layer-dto-no-leak

Translators exist only to convert between entities and an underlying representation (row, ORM model, vendor payload, wire format). A row type in a useCase signature, an entity field typed as a vendor payload, an ORM model returned from storage, or a vendor exception escaping a gateway are all leaks. Translate at the adapter boundary.
