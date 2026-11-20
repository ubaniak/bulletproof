# Manifests & scaffolding (MEDIUM)

A **manifest** is a small, hand-edited YAML file that describes a domain, or how processes are assembled, so a generator can scaffold and wire it. It is optional. The rules in `rules/` apply identically with or without it; a manifest only automates what the rules would otherwise have you do by hand.

Manifests are language-neutral. A generator reads them and emits idiomatic code per language file.

## manifest-domain

`<domain>/domain.yaml`, one per domain. (Named `domain`, not `service`, because in this skill `service` is only the business layer.)

```yaml
name: orders
purpose: Order lifecycle from cart to fulfilment    # one purpose (structure-single-purpose)

storage:
  - port: OrderStorage
    engine: postgres
    aggregates: [Order]

gateways:
  - port: PaymentGateway
    impl: stripe                  # third-party provider
  - port: CustomerGateway
    impl: customersrpc            # peer running in another process (contract-remote-peer-is-gateway)
    peer: customers

peers:                            # same-process dependencies: other domains' service ports
  - payments

publishes: [OrderPlaced, OrderCancelled]
consumes:
  - event: PaymentSucceeded
    from: payments

exposes:
  rest:    { spec: api/openapi/orders/orders.yaml, prefix: /v1/orders }
  grpc:    { proto: api/proto/orders/v1 }
  consumer: {}
  cmd:     [ { name: "orders reindex" } ]
```

Every field maps to a rule: `storage`/`gateways` to ports and adapters, `peers` to `structure-cross-domain-imports`, `publishes` to `consistency-outbox`, `exposes` to `structure-transport-layer`.

## manifest-process

`bulletproof.yaml` at the repo root says which domains and transports each process mounts. This is how one codebase serves a modular monolith, several processes, or one domain per binary:

```yaml
processes:
  api:    { domains: [orders, payments, customers], transports: [rest, grpc] }
  worker: { domains: [orders, payments],            transports: [consumer, cron] }
  cli:    { domains: [orders],                      transports: [cmd] }
```

Microservice-per-binary is the same file with one domain per process. A domain listed in a process but whose peer is not listed there gets the `remote` gateway adapter for that peer (`contract-remote-peer-is-gateway`).

## manifest-scaffold-outputs

From the manifests a generator emits:

| Output | Notes |
|---|---|
| `entities`, `errors`, `service` skeletons | created once (hand-owned) |
| port files for each `storage`/`gateway` entry | created once |
| adapter + `_dto` stubs, memory fake, **contract-suite stub** per port | created once; the contract suite is the conformance gate |
| `transport/<protocol>/<domain>` + `_dto` for each `exposes` entry | created once; handlers wired to generated stubs |
| `publisher` port and outbox adapter for `publishes` | created once |
| migrations directory for the domain | created once |
| boundary-lint entries for the new domain | generated |
| registration in `app/` for each process listing the domain | generated (see below) |

## manifest-hand-owned-files

Two kinds of file, never mixed:

- **Generated** (header "generated, do not edit"): regenerated on every run, safe to overwrite.
- **Hand-owned** (everything with logic: invariants, rules, translator mapping bodies, handler bodies): written **only if missing**, never overwritten. Re-running the generator after editing a manifest adds new files and reports, but does not touch, files you own.

If a generated file needs a behaviour change, change the manifest or the template.

## manifest-generated-wiring

`app/` may be partly generated. The generated part contains **only** construction and registration (build platform, call each domain's `setup`, build each transport, mount it) driven by `bulletproof.yaml`. Route prefixes may come from the manifest. Middleware order, lifecycle, and graceful shutdown stay in hand-written `app/` code. All rules still apply: no env or feature-flag branching in generated wiring (`layer-setup-wiring-only`), and `setup` itself remains hand-written.

## manifest-consistency-check

CI verifies that manifests and code agree: every port in `domain.yaml` exists, every `exposes` entry has a transport folder, every transport folder has a manifest entry, and no domain is registered in a process that its manifest does not allow. Drift between manifest and code fails the build, the same way generated-code drift does.
