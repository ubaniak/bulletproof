# Testing (HIGH)

The layout makes each layer testable in isolation. Use that.

| Layer | Test type | Real deps? |
|---|---|---|
| entities | unit | none |
| useCase | unit with fakes | none |
| storage/gateway adapters | contract suite | real DB (container) / provider sandbox or recorded HTTP |
| transport | handler tests | fake useCase |
| setup/app | few e2e | real DB, fake gateways |

## testing-usecase-fakes

useCase tests run against in-memory fakes of its ports — fast, no DB, no network.

- Prefer hand-written fakes (`storage/memory`) over mocking frameworks. A fake that implements the port is itself an adapter and must pass the contract suite.
- Small ports (ISP) keep fakes small. If a fake needs 20 stubs, the port is too fat.
- Inject the platform clock and ID generator so tests are deterministic.

## testing-adapter-contract

For every port, write one contract suite and run it against every adapter, including the in-memory fake:

```
contract UserStorageContract(newStorage: fn() -> UserStorage)
  test "getById on missing id fails NotFound"
    expect newStorage().getById(ctx, "nope") fails users.NotFound
  test "tenant isolation"
    ...

run UserStorageContract(newStorage: () -> MySQLUserStorage(testDb))
run UserStorageContract(newStorage: () -> MemoryUserStorage())
```

This is how Liskov substitution is **enforced**, not just documented. A new adapter is not done until it passes the suite. Gateway contracts assert error translation (vendor failure → domain error).

In dynamically typed languages this suite is the main guarantee an adapter conforms to its port; treat it as mandatory.

## testing-transport

Handler tests use a fake useCase and assert:
- wire → entity translation, including shape validation failures
- entity → wire response
- error → response mapping for each domain error kind
- authn middleware rejects missing/invalid credentials

## testing-e2e-through-setup

A thin set of end-to-end tests builds the real graph through `setup`/`app` with a real DB and fake gateways, covering critical paths only. If most bugs are only caught here, the lower layers are under-tested.

A domain is not done without useCase tests and contract suites for its ports.
