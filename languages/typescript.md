# Bulletproof in TypeScript (Node)

How to express the rules in `rules/` idiomatically in TypeScript. Rules win over anything here.

## Layout

```
src/
  main.ts                         # load config, app.run()
  app/app.ts
  platform/{config,logger,db,tx,errors,context,clock}.ts
  transport/                      # inbound adapters: protocol, then domain
    rest/timesheet/  rest/timesheet-dto/
    webhook/timesheet/  webhook/timesheet-dto/
    cmd/timesheet/  cmd/timesheet-dto/
  timesheet/
    index.ts                      # PUBLIC API: re-exports entities, errors, Service type, events
    entities/timesheet.ts
    errors.ts
    storage.ts                    # port(s)
    gateway.ts
    publisher.ts
    service.ts                   # Service port + implementation
    setup.ts
    storage/mysql/  storage/mysql-dto/  storage/memory/  storage/contract.ts
    gateway/quickbooks/  gateway/quickbooks-dto/
```

Other domains and `transport/**` import only `@/timesheet` (the `index.ts`), never deep paths.

## Ports

```ts
export interface TimesheetStorage {
  getById(ctx: Ctx, id: string): Promise<Timesheet>;          // throws NotFound
  list(ctx: Ctx, f: TimesheetFilter): Promise<Page<Timesheet>>;
  save(ctx: Ctx, ts: Timesheet): Promise<void>;
}
```

Use `interface`/`type`, not abstract classes. Declared in the domain.

## Public API enforcement

- `index.ts` re-exports only `entities`, `errors`, `Service` (type), event types.
- `@/` path alias; deep imports into another domain forbidden by lint.
- `package.json` `exports` per domain if domains are workspace packages.

## Boundary lint

`dependency-cruiser` (`.dependency-cruiser.cjs`, rules are path regexes; `$1`/`$2` refer to groups captured in `from`):
```js
const DOMAIN = "^src/(?!platform/|app/|transport/)([^/]+)/";
module.exports = {
  options: { tsConfig: { fileName: "tsconfig.json" }, tsPreCompilationDeps: true, doNotFollow: { path: "node_modules" } },
  forbidden: [
    { name: "platform-depends-on-no-domain", severity: "error",
      from: { path: "^src/platform/" }, to: { path: "^src/(?!platform/)" } },
    { name: "domains-use-only-others-public-api", severity: "error",
      from: { path: DOMAIN },
      to: { path: "^src/(?!platform/|app/|transport/)(?!$1/)[^/]+/", pathNot: "^src/[^/]+/index\\.ts$" } },
    { name: "no-domain-depends-on-transport", severity: "error",
      from: { path: DOMAIN }, to: { path: "^src/transport/" } },
    { name: "transport-uses-own-domain-public-api-only", severity: "error",
      from: { path: "^src/transport/[^/]+/([^/]+?)(-dto)?/" },
      to: { path: "^src/(?!platform/|transport/)", pathNot: "^src/$1/index\\.ts$" } },
    { name: "transports-do-not-depend-on-each-other", severity: "error",
      from: { path: "^src/transport/([^/]+)/([^/]+?)(-dto)?/" },
      to: { path: "^src/transport/", pathNot: "^src/transport/$1/$2(-dto)?/" } },
    { name: "entities-and-service-import-no-io", severity: "error",
      from: { path: "^src/[^/]+/(entities/|service\\.ts$)" },
      to: { dependencyTypes: ["npm"], path: "^node_modules/(pg|mysql2|@?prisma|typeorm|axios|express|@nestjs|stripe)/" } },
  ],
};
```
Run `depcruise src --config .dependency-cruiser.cjs` in CI. Extend the IO list with your own drivers and vendor SDKs. `app/` and `<domain>/setup.ts` are unrestricted by these rules; `app` may import any domain's `setup`, and nothing else may (rule 2 blocks other domains, rule 4 blocks transports).

`eslint-plugin-boundaries` is an alternative, but its current element descriptors classify folders only, so single-file units (`index.ts`, `service.ts`, `setup.ts`) are silently left unchecked. If you use it, make each unit a folder first, and confirm with a deliberate violation that the rule fires.

## Errors

Exceptions with a kind and code.

```ts
// platform/errors.ts
export type Kind = "Invalid" | "NotFound" | "Conflict" | "Unauthorized" | "Forbidden" | "RateLimited" | "Unavailable" | "Internal";
export class DomainError extends Error {
  constructor(readonly kind: Kind, readonly code: string, message: string, options?: { cause?: unknown }) { super(message, options); }
}

// payments/errors.ts
export class CardDeclined extends DomainError { constructor(cause?: unknown) { super("Invalid", "card_declined", "card was declined", { cause }); } }

// payments/gateway/stripe-dto/errors.ts
export function translateError(e: unknown): DomainError {
  if (e instanceof Stripe.errors.StripeCardError) return new CardDeclined(e);
  if (e instanceof Stripe.errors.StripeRateLimitError) return new RateLimited(e);
  return new Unavailable(e);
}
```

`Result` types (neverthrow, Effect) are fine if used consistently; the same kinds apply. Transport: one error handler/middleware per protocol (in `platform/rest`, registered in `app`) mapping `kind` → status.

## Context

Either an explicit `ctx: Ctx` first parameter (preferred: visible and testable) or `AsyncLocalStorage` behind `platform/context.ts` accessors. Pick one per codebase.

## Transactions

```ts
export interface TxManager {
  withinTx<T>(ctx: Ctx, work: (ctx: Ctx) => Promise<T>): Promise<T>;
  detached(ctx: Ctx): Ctx;   // same ctx without the active transaction
}
```
The adapter puts the driver/ORM transaction into ctx (or ALS); storage adapters read it. Prisma `$transaction`, Knex `transaction`, TypeORM `QueryRunner` live only in the adapter.

## Contracts & codegen (optional)

`api/` + `gen/` at the repo root. `openapi-typescript` (or `orval`) for OpenAPI, `@bufbuild/protobuf` with `buf` for protobuf; `buf breaking` and `oasdiff` in CI. Extend the `dependency-cruiser` rules so only `src/transport/**`, `src/*/gateway/**`, and publisher/consumer adapters may import `gen/`. Drift check: `npm run gen && git diff --exit-code`.

## Wiring

Manual constructor injection in `setup.ts`. DI containers (tsyringe, Inversify) only in `setup`/`app`.

**NestJS**: modules, `@Injectable`, `@Controller` belong to `setup` and `transport` only. A service is a plain class registered via a factory provider in the module; no Nest decorators or imports in `service.ts` or entities. Guards are coarse authn; authz stays in the service.

## Testing

- In-memory fakes in `storage/memory`.
- Contract suite as `export function timesheetStorageContract(make: () => Promise<TimesheetStorage>)` run with `describe` from each adapter's test file.
- Real DB via testcontainers.

## Naming & tooling

- Files `kebab-case.ts`, types/classes `PascalCase`, translator folders `<impl>-dto`.
- `strict: true`, `noUncheckedIndexedAccess`; no `any` in domain code.
- Biome or ESLint + Prettier, `dependency-cruiser`, `tsc --noEmit` in CI; pnpm/npm with lockfile and exact versions; husky + lint-staged.

## Traps

- Prisma/TypeORM entities used as domain entities — keep ORM models in `storage/<impl>`, translate.
- Express `req`/`res` passed into service.
- Zod schemas for **business** rules in transport — Zod checks shape; business rules go in service/entities.
