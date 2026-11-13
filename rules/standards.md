# Standards & tooling (MEDIUM)

## standards-boundary-lint

Every boundary rule must be checked by a tool in CI. Review alone does not hold.

Go — `go-arch-lint` (`.go-arch-lint.yml`), sketch:
```yaml
version: 3
workdir: .
components:
  platform:  { in: platform/** }
  app:       { in: app/** }
  domain:    { in: "*/" }                 # each domain root (entities, errors, useCase, ports)
  adapters:  { in: "*/{storage,gateway,transport}/**" }
  setup:     { in: "*/setup/**" }
deps:
  platform: { mayDependOn: [] }
  domain:   { mayDependOn: [platform, domain] }   # only public surface of other domains, see below
  adapters: { mayDependOn: [platform, domain] }
  setup:    { mayDependOn: [platform, domain, adapters] }
  app:      { mayDependOn: [platform, domain, setup] }
```
Combine with `internal/` packages so other domains physically cannot import adapters. `depguard` can forbid `database/sql`, `net/http` in useCase packages.

TypeScript — `eslint-plugin-boundaries` or `dependency-cruiser`:
- forbid imports of `<domain>/(storage|gateway|transport|setup)/**` from outside `<domain>`
- forbid `platform/**` importing any domain
- forbid `pg`, `mysql2`, `axios`, `express` etc. in `<domain>/useCase*`

Add each new domain to the config as part of the scaffold.

## standards-naming

| Thing | Go | TypeScript |
|---|---|---|
| Domain folder | lowercase, singular-ish noun (`timesheet`, `payroll`) | kebab-case (`pay-period`) |
| Files | `snake_case.go` | `kebab-case.ts` |
| Business layer | `useCase` (file `use_case.go`), type `UseCase` | `use-case.ts`, `UseCase` |
| Ports | `Storage`, `PaymentGateway`, `Publisher` — role names, not vendor names | same |
| Adapters | folder = technology (`mysql`, `stripe`, `rest`) | same |
| DTO folders | `<impl>_dto` | `<impl>-dto` |
| Errors | `ErrXxx` values | `XxxError` classes or `{ code }` constants |
| Events | past tense `TimesheetApproved` | same |

Pick `useCase` everywhere. Do not mix with `service`.

## standards-strict-tooling

- **Formatting**: `gofmt`/`goimports`; Biome or Prettier for TS. Never argued about in review.
- **Linting**: `golangci-lint` (incl. `depguard`, `errorlint`, `wrapcheck`) / Biome or ESLint with boundaries.
- **Types**: TS `strict: true`, no `any` in domain code.
- **Dependencies**: pinned/locked versions (`go.sum`, exact versions + lockfile); renovate/dependabot for updates.
- **Pre-commit**: format + lint + boundary check on staged files.
- **CI gates**: build, lint, boundary lint, unit + contract tests, migration check. Merge blocked on failure.
- **Migrations**: forward-only, reviewed, namespaced per domain.
