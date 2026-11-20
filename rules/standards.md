# Standards & tooling (MEDIUM)

Concrete tools and configs are in the language file. The requirements are the same everywhere.

## standards-boundary-lint

Every dependency rule must be checked by a tool in CI. Review alone does not hold. The linter config must encode at least:

| From | May depend on |
|---|---|
| `platform` | nothing in the project |
| `<domain>/entities`, `<domain>/errors` | `platform` |
| `<domain>` ports, `useCase` | `platform`, own entities/errors, other domains' public API |
| `<domain>/storage|gateway|transport/*` adapters | `platform`, own domain |
| `<domain>/setup` | `platform`, own domain, own adapters, other domains' public API |
| `app` | `platform`, every domain's public API and `setup` |

Plus a deny-list: `entities` and `useCase` may not depend on any IO library or web/ORM framework package.

Prefer language visibility (internal packages, module exports, access modifiers) **and** a linter: visibility stops accidental imports, the linter catches the rest. Add each new domain to the config as part of the scaffold.

## standards-naming

Concept names are fixed across languages; casing follows the language file.

| Concept | Fixed name |
|---|---|
| business layer | `useCase` (never mixed with `service`) |
| ports | role names: `UserStorage`, `PaymentGateway`, `Publisher` — never vendor names |
| adapters | technology names: `mysql`, `stripe`, `rest`, `memory`, `cached` |
| translators | `<impl>_dto` (or the language's casing of it) |
| errors | named after the condition: `NotFound`, `CardDeclined` |
| events | past tense: `TimesheetApproved` |

## standards-strict-tooling

- **Formatting**: the language's standard formatter, enforced, never argued in review.
- **Linting**: standard linter plus boundary rules and error-handling checks.
- **Types**: strictest available type checking. Dynamic languages require a type checker in strict mode on domain code.
- **Dependencies**: locked versions; automated update PRs.
- **Pre-commit**: format + lint + boundary check on staged files.
- **CI gates**: build, lint, boundary lint, type check, unit + contract tests, migration check. Merge blocked on failure.
- **Migrations**: forward-only, reviewed, owned per domain.
