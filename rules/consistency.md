# Consistency: transactions & events (HIGH)

## consistency-tx-port

When one useCase writes through several storage ports atomically, use a transaction port. useCase never holds a connection, session, unit-of-work implementation, or transaction object.

```
port TxManager                                  # declared in platform
  withinTx(ctx, work: fn(ctx) -> T) -> T

useCase.approve(ctx, id)
  tx.withinTx(ctx, fn(ctx):
    timesheets.save(ctx, ts)
    audit.append(ctx, entry))
```

Storage adapters pick up the active transaction from ctx (or an ambient scope, per language) and fall back to the pool.

**Wrong**
```
approve(ctx, tx: SqlTransaction, id)          # driver type in useCase signature
@Transactional approve(ctx, id)               # framework annotation on useCase
```

Framework transaction helpers are fine **inside** the `TxManager` adapter.

## consistency-one-domain-per-tx

A database transaction never spans two domains. If A's useCase calls B's useCase inside A's `withinTx`, B must not join that transaction. Cross-domain consistency is eventual, via events.

## consistency-outbox

Side effects that must happen after a commit — notifying another domain, calling a third party, syncing to an accounting system — go through an outbox:

1. In the same transaction as the state change, useCase calls `publisher.publish(ctx, event)`; the adapter writes to an `outbox` table.
2. A relay (worker registered in `app/`) reads the outbox and delivers to the bus or invokes the target.
3. Consumers are transports (`transport/consumer`) that call their own useCase.

**Wrong**
```
tx.withinTx(ctx, fn(ctx):
  storage.save(ctx, invoice)
  quickbooks.createInvoice(ctx, invoice))     # remote call inside a DB tx; partial failure on rollback
```

Events are named in past tense (`TimesheetApproved`), owned by the publishing domain, and carry IDs plus the minimum data consumers need. Event payload types are part of the publisher's public API, versioned like a wire contract.

Direct useCase call vs. event:
- Need the answer now to continue → call the other domain's useCase.
- Reacting to something that happened, or breaking a cycle → event.

## consistency-idempotency

Every entry point that can be retried must be idempotent:
- Webhooks (providers redeliver), queue consumers (at-least-once), client POSTs with retry.
- Use an idempotency key (provider event ID, message ID, `Idempotency-Key` header) stored with a unique constraint by the receiving domain's storage.
- useCase checks/records the key; transport only extracts it.

Long multi-step workflows across domains (sagas) are owned by a coordinator domain that persists its workflow state and issues compensating actions on failure.
