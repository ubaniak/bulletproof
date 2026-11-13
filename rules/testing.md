# Testing (HIGH)

The layout makes each layer testable in isolation. Use that.

| Layer | Test type | Real deps? |
|---|---|---|
| entities | unit | none |
| useCase | unit with fakes | none |
| storage/gateway adapters | contract suite | real DB (container) / sandbox or recorded HTTP |
| transport | handler tests | fake useCase |
| setup/app | few e2e | real DB, fake gateways |

## testing-usecase-fakes

useCase tests run against in-memory fakes of its ports — fast, no DB.

- Prefer hand-written fakes (`storage/memory`) over mock frameworks; a fake that implements the interface is itself an adapter and must pass the contract suite.
- Small interfaces (ISP) keep fakes small. If a fake needs 20 stubs, the interface is too fat.
- Inject `platform.Clock` and ID generator so tests are deterministic.

## testing-adapter-contract

For every port, write one contract suite and run it against every adapter, including the in-memory fake:

```go
// storage/contract/contract.go
func RunUserStorage(t *testing.T, newStore func(t *testing.T) users.Storage) {
    t.Run("GetByID missing returns ErrNotFound", func(t *testing.T) {
        _, err := newStore(t).GetByID(ctx, "nope")
        require.ErrorIs(t, err, users.ErrNotFound)
    })
    t.Run("tenant isolation", func(t *testing.T) { /* ... */ })
}

// storage/mysql/mysql_test.go
func TestMySQL(t *testing.T) { contract.RunUserStorage(t, newMySQLStore) }
// storage/memory/memory_test.go
func TestMemory(t *testing.T) { contract.RunUserStorage(t, memory.New) }
```

This is how Liskov substitution is **enforced**, not just documented. A new adapter is not done until it passes the suite. Gateway contracts assert error translation (vendor error → domain error).

## testing-transport

Handler tests use a fake useCase and assert:
- wire → entity translation (including shape validation failures)
- entity → wire response
- error → status mapping for each domain error kind
- authn middleware rejects missing/invalid credentials

## testing-e2e-through-setup

A thin set of end-to-end tests builds the real graph through `setup`/`app` with a real DB and fake gateways, covering the critical paths only. If most bugs are only caught here, the lower layers are under-tested.

Add a test step to every scaffold: a domain is not done without useCase tests and contract suites for its ports.
