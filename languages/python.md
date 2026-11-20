# Bulletproof in Python

How to express the rules in `rules/` idiomatically in Python. Rules win over anything here.

## Layout

```
src/<project>/
  main.py                          # load config, app.run()
  app/app.py
  platform/{config,logging,db,tx,errors,context,clock}.py
  timesheet/
    __init__.py                    # PUBLIC API via __all__: entities, errors, UseCase, events
    entities.py
    errors.py
    ports.py                       # storage/gateway/publisher/transport protocols
    use_case.py
    setup.py
    storage/mysql.py  storage/mysql_dto.py  storage/memory.py  storage/contract.py
    gateway/quickbooks.py  gateway/quickbooks_dto.py
    transport/rest.py  transport/rest_dto.py  transport/webhook.py  transport/webhook_dto.py
```

## Ports

```python
class TimesheetStorage(Protocol):
    async def get_by_id(self, ctx: Ctx, id: str) -> Timesheet: ...   # raises NotFound
    async def list(self, ctx: Ctx, f: TimesheetFilter) -> Page[Timesheet]: ...
    async def save(self, ctx: Ctx, ts: Timesheet) -> None: ...
```

`typing.Protocol` (structural) or `abc.ABC`. Declared in the domain.

## Public API enforcement

Python cannot hide modules, so enforcement is tooling:
- `__all__` in the domain `__init__.py` defines the public API.
- `import-linter` contracts in CI (below).
- Leading underscore for adapter-internal helpers.

## Boundary lint

`import-linter` (`.importlinter`):
```ini
[importlinter]
root_package = project

[importlinter:contract:platform-independent]
name = platform depends on no domain
type = forbidden
source_modules = project.platform
forbidden_modules = project.timesheet, project.payroll

[importlinter:contract:no-adapter-reach]
name = domains use only each other's public API
type = forbidden
source_modules = project.payroll
forbidden_modules = project.timesheet.storage, project.timesheet.gateway, project.timesheet.transport, project.timesheet.setup

[importlinter:contract:pure-core]
name = entities and use cases import no IO
type = forbidden
source_modules = project.timesheet.entities, project.timesheet.use_case
forbidden_modules = sqlalchemy, psycopg, httpx, requests, fastapi, django, stripe
```

## Errors

Exception hierarchy with kind and code.

```python
class Kind(StrEnum):
    INVALID = "invalid"; NOT_FOUND = "not_found"; CONFLICT = "conflict"; FORBIDDEN = "forbidden"
    RATE_LIMITED = "rate_limited"; UNAVAILABLE = "unavailable"; INTERNAL = "internal"

class DomainError(Exception):
    kind: Kind; code: str

class CardDeclined(DomainError):
    kind, code = Kind.INVALID, "card_declined"

# gateway/stripe_dto.py
def translate_error(e: stripe.error.StripeError) -> DomainError:
    if isinstance(e, stripe.error.CardError): return CardDeclined()
    if isinstance(e, stripe.error.RateLimitError): return RateLimited()
    return Unavailable()

# adapter: `except stripe.error.StripeError as e: raise translate_error(e) from e`
```

Transport: one exception handler per app (FastAPI `exception_handler(DomainError)`, Django middleware) mapping `kind` → status.

## Context

`contextvars.ContextVar` behind `platform/context.py` accessors, or an explicit `ctx` parameter. Set at the transport boundary; works across `async` tasks.

## Transactions

```python
class TxManager(Protocol):
    def within_tx(self, ctx: Ctx) -> AbstractAsyncContextManager[Ctx]: ...

async with self.tx.within_tx(ctx) as ctx:
    await self.timesheets.save(ctx, ts)
    await self.audit.append(ctx, entry)
```
SQLAlchemy `Session`/`AsyncSession` lives only in the adapter, carried via ctx/contextvar.

## Wiring

Manual construction in `setup.py`. `dependency-injector` or FastAPI `Depends` only in `setup`/`app`/transport — `Depends` never in useCase signatures.

**Django**: models are storage adapters, not entities; views/DRF serializers are transport + translators; business logic leaves `models.py`/views into `use_case.py`.

## Testing

- In-memory fakes in `storage/memory.py`.
- Contract suite as a base test class or a `pytest` fixture parametrized over adapters (`@pytest.fixture(params=["memory", "mysql"])`).
- **Mandatory** in Python: there is no compiler checking port conformance.
- Real DB via testcontainers.

## Naming & tooling

- Modules/files `snake_case`, classes `PascalCase`, translators `<impl>_dto.py`.
- `ruff` (lint + format), `mypy --strict` or `pyright` strict on the domain package, `import-linter`, `uv`/`poetry` lockfile, pre-commit.

## Traps

- SQLAlchemy/Django models used as entities.
- Pydantic models from transport passed into useCase — translate to entities/primitives.
- Module-level singletons (`db = Database()`) imported by useCase — inject.
