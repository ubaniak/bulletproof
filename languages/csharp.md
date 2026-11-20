# Bulletproof in C# (.NET)

How to express the rules in `rules/` idiomatically in .NET. Rules win over anything here.

## Layout

Either one project with folders + namespaces, or one project per layer when compile-time enforcement is wanted:

```
src/
  Acme.App/                         # Program.cs, composition root
  Acme.Platform/
  Acme.Timesheet/                   # PUBLIC: Entities, Errors, ITimesheetUseCase, events, ports
    Entities/  Errors.cs  Ports/  UseCase/  Setup/
    Storage/MySql/  Storage/MySqlDto/  Storage/Memory/
    Gateway/QuickBooks/  Gateway/QuickBooksDto/
    Transport/Rest/  Transport/RestDto/  Transport/Webhook/
```

## Ports

```csharp
public interface ITimesheetStorage
{
    Task<Timesheet> GetById(Ctx ctx, string id);              // throws NotFound
    Task<Page<Timesheet>> List(Ctx ctx, TimesheetFilter f);
    Task Save(Ctx ctx, Timesheet ts);
}
```

## Public API enforcement

- Adapters, translators, and use-case implementations are `internal`; the port, entities, errors are `public`.
- `InternalsVisibleTo` only for the domain's own test project.
- NetArchTest / ArchUnitNET tests in CI.

## Boundary lint

```csharp
Types.InAssembly(typeof(Timesheet).Assembly)
    .That().ResideInNamespace("Acme.Timesheet.Entities").Or().ResideInNamespace("Acme.Timesheet.UseCase")
    .ShouldNot().HaveDependencyOnAny("System.Data", "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore", "Stripe", "System.Net.Http")
    .GetResult().IsSuccessful.Should().BeTrue();

Types.InAssembly(typeof(Payroll).Assembly)
    .ShouldNot().HaveDependencyOnAny("Acme.Timesheet.Storage", "Acme.Timesheet.Gateway", "Acme.Timesheet.Transport")
    .GetResult().IsSuccessful.Should().BeTrue();
```

## Errors

Exception hierarchy (or a `Result<T, DomainError>` library — pick one per codebase).

```csharp
public enum Kind { Invalid, NotFound, Conflict, Unauthorized, Forbidden, RateLimited, Unavailable, Internal }
public class DomainError(Kind kind, string code, string message, Exception? inner = null) : Exception(message, inner)
{ public Kind Kind { get; } = kind; public string Code { get; } = code; }
public sealed class CardDeclined(Exception? inner = null) : DomainError(Kind.Invalid, "card_declined", "card was declined", inner);

// gateway adapter
catch (StripeException e) when (e.StripeError?.Code == "card_declined") { throw new CardDeclined(e); }
catch (StripeException e) { throw new Unavailable(e); }
```

Transport: one `IExceptionHandler` / exception middleware mapping `Kind` → `ProblemDetails` status.

## Context

Explicit `Ctx` parameter (preferred, carries `CancellationToken`), or `AsyncLocal<T>` behind `Platform.Context` accessors. `HttpContext` never leaves transport.

## Transactions

```csharp
public interface ITxManager { Task<T> WithinTx<T>(Ctx ctx, Func<Ctx, Task<T>> work); }
```
Adapter implemented with EF Core `Database.BeginTransactionAsync` or `TransactionScope`; `DbContext` lives only in storage adapters.

## Wiring

One `IServiceCollection` extension per domain in `Setup` (`services.AddTimesheet(config)`) registering adapters against ports. `Program.cs` (app) calls each extension and maps endpoints. useCase classes never take `IServiceProvider`.

EF Core entity classes are storage models (`Storage/Ef`), not domain entities.

## Testing

- In-memory fakes implementing ports.
- Contract suite as an abstract xUnit class (`public abstract class TimesheetStorageContract { protected abstract ITimesheetStorage Create(); ... }`) inherited per adapter.
- Testcontainers for DB adapters; architecture tests in the normal test run.

## Naming & tooling

- `PascalCase` everything, `I` prefix on ports, translator folders `<Impl>Dto`.
- `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, `dotnet format`, analyzers, Central Package Management with locked versions.

## Traps

- EF `DbContext` injected into use cases.
- `[Authorize(Roles=...)]` as the only authorization — fine as a coarse guard, business authz stays in useCase.
- MediatR handlers mixing transport and business logic — a handler is a useCase method; keep HTTP types out.
