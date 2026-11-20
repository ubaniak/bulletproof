# Bulletproof in TypeScript (Node)

How to express the rules in `rules/` idiomatically in TypeScript. Rules win over anything here.

## Layout

```
src/
  main.ts                         # load config, app.run()
  app/app.ts
  platform/{config,logger,db,tx,errors,context,clock}.ts
  timesheet/
    index.ts                      # PUBLIC API: re-exports entities, errors, UseCase type, events
    entities/timesheet.ts
    errors.ts
    storage.ts                    # port(s)
    gateway.ts
    publisher.ts
    use-case.ts                   # UseCase interface + implementation
    transport.ts
    setup.ts
    storage/mysql/  storage/mysql-dto/  storage/memory/  storage/contract.ts
    gateway/quickbooks/  gateway/quickbooks-dto/
    transport/rest/  transport/rest-dto/  transport/webhook/  transport/webhook-dto/
```

Other domains import only `@/timesheet` (the `index.ts`), never deep paths.

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

- `index.ts` re-exports only `entities`, `errors`, `UseCase` (type), event types.
- `@/` path alias; deep imports into another domain forbidden by lint.
- `package.json` `exports` per domain if domains are workspace packages.

## Boundary lint

`eslint-plugin-boundaries` (sketch):
```js
settings: {
  "boundaries/elements": [
    { type: "platform", pattern: "src/platform/*" },
    { type: "app",      pattern: "src/app/*" },
    { type: "public",   pattern: "src/*/index.ts", mode: "file", capture: ["domain"] },
    { type: "core",     pattern: "src/*/(entities|errors.ts|use-case.ts|storage.ts|gateway.ts|publisher.ts|transport.ts)", capture: ["domain"] },
    { type: "adapter",  pattern: "src/*/(storage|gateway|transport)/*", capture: ["domain"] },
    { type: "setup",    pattern: "src/*/setup.ts", mode: "file", capture: ["domain"] },
  ],
},
rules: {
  "boundaries/element-types": ["error", { default: "disallow", rules: [
    { from: "platform", allow: [] },
    { from: "core",    allow: ["platform", ["core", { domain: "${from.domain}" }], "public"] },
    { from: "adapter", allow: ["platform", ["core", { domain: "${from.domain}" }]] },
    { from: "setup",   allow: ["platform", ["core", { domain: "${from.domain}" }], ["adapter", { domain: "${from.domain}" }], "public"] },
    { from: "app",     allow: ["platform", "public", "setup"] },
  ]}],
  "no-restricted-imports": ["error", { patterns: ["@/*/storage/*", "@/*/gateway/*", "@/*/transport/*"] }],
}
```
Add an override for `entities/**` and `use-case.ts` restricting `pg`, `mysql2`, `prisma`, `typeorm`, `axios`, `express`, `@nestjs/*`, vendor SDKs. `dependency-cruiser` is an equivalent alternative.

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

`Result` types (neverthrow, Effect) are fine if used consistently; the same kinds apply. Transport: one error handler/middleware per transport mapping `kind` → status.

## Context

Either an explicit `ctx: Ctx` first parameter (preferred: visible and testable) or `AsyncLocalStorage` behind `platform/context.ts` accessors. Pick one per codebase.

## Transactions

```ts
export interface TxManager { withinTx<T>(ctx: Ctx, work: (ctx: Ctx) => Promise<T>): Promise<T>; }
```
The adapter puts the driver/ORM transaction into ctx (or ALS); storage adapters read it. Prisma `$transaction`, Knex `transaction`, TypeORM `QueryRunner` live only in the adapter.

## Wiring

Manual constructor injection in `setup.ts`. DI containers (tsyringe, Inversify) only in `setup`/`app`.

**NestJS**: modules, `@Injectable`, `@Controller` belong to `setup` and `transport` only. useCase is a plain class registered via a factory provider in the module; no Nest decorators or imports in `use-case.ts` or entities. Guards are coarse authn; authz stays in useCase.

## Testing

- In-memory fakes in `storage/memory`.
- Contract suite as `export function timesheetStorageContract(make: () => Promise<TimesheetStorage>)` run with `describe` from each adapter's test file.
- Real DB via testcontainers.

## Naming & tooling

- Files `kebab-case.ts`, types/classes `PascalCase`, translator folders `<impl>-dto`.
- `strict: true`, `noUncheckedIndexedAccess`; no `any` in domain code.
- Biome or ESLint + Prettier, boundaries plugin, `tsc --noEmit` in CI; pnpm/npm with lockfile and exact versions; husky + lint-staged.

## Traps

- Prisma/TypeORM entities used as domain entities — keep ORM models in `storage/<impl>`, translate.
- Express `req`/`res` passed into useCase.
- Zod schemas for **business** rules in transport — Zod checks shape; business rules go in useCase/entities.
