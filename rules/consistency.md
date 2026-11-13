# Consistency: transactions & events (HIGH)

## consistency-tx-port

When one useCase writes through several storage interfaces atomically, use a transaction port. The useCase never sees `*sql.Tx`.

```go
// platform
type TxManager interface {
    WithinTx(ctx context.Context, fn func(ctx context.Context) error) error
}

// useCase
err := u.tx.WithinTx(ctx, func(ctx context.Context) error {
    if err := u.timesheets.Save(ctx, ts); err != nil { return err }
    return u.audit.Append(ctx, entry)
})
```

Storage adapters pull the active tx from `ctx` (`platform.TxFrom(ctx)`) and fall back to the pool.

**Wrong**
```go
func (u *UseCase) Approve(ctx context.Context, tx *sql.Tx, id string) error // driver type in useCase
```

## consistency-one-domain-per-tx

A database transaction never spans two domains. If A's useCase calls B's useCase inside A's `WithinTx`, B must not join that transaction. Cross-domain consistency is eventual, via events.

## consistency-outbox

Side effects that must happen after a commit — notifying another domain, calling a third party, syncing to QuickBooks — go through an outbox:

1. In the same transaction as the state change, the useCase calls `publisher.Publish(ctx, event)`; the adapter writes to an `outbox` table.
2. A relay (worker in `app/`) reads the outbox and delivers to the bus or invokes the target.
3. Consumers are transports (`transport/consumer`) that call their own useCase.

**Wrong**
```go
u.tx.WithinTx(ctx, func(ctx context.Context) error {
    u.storage.Save(ctx, invoice)
    return u.quickbooks.CreateInvoice(ctx, invoice) // remote call inside a DB tx; partial failure on rollback
})
```

Events are named in past tense (`TimesheetApproved`), owned by the publishing domain, and carry IDs plus the minimum data consumers need. Event payload types are part of the publisher's public API (versioned like a wire contract).

When to use a direct useCase call vs. an event:
- Need the answer now to continue → call the other domain's useCase.
- Reacting to something that happened, or breaking a cycle → event.

## consistency-idempotency

Every entry point that can be retried must be idempotent:
- Webhooks (providers redeliver), queue consumers (at-least-once), POSTs from clients with retry.
- Use an idempotency key (provider event ID, message ID, `Idempotency-Key` header) stored with a unique constraint by the receiving domain's storage.
- The useCase checks/records the key; transport only extracts it.

Long multi-step workflows across domains (sagas) are owned by a coordinator domain that persists its workflow state and issues compensating actions on failure.
