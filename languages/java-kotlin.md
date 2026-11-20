# Bulletproof in Java / Kotlin (JVM)

How to express the rules in `rules/` idiomatically on the JVM. Rules win over anything here. Examples are Kotlin; Java is the same shape.

## Layout

```
com.acme.app                       # Application/main, composition root
com.acme.platform.{config,logging,db,tx,errors,context,clock}
com.acme.timesheet                 # PUBLIC: entities, errors, UseCase, events, ports
com.acme.timesheet.entities
com.acme.timesheet.setup
com.acme.timesheet.storage.mysql       com.acme.timesheet.storage.mysqldto
com.acme.timesheet.gateway.quickbooks  com.acme.timesheet.gateway.quickbooksdto
com.acme.timesheet.transport.rest      com.acme.timesheet.transport.restdto
```

Package names cannot contain `_`; translators are `<impl>dto` (or a `dto` subpackage of the adapter).

## Ports

```kotlin
interface TimesheetStorage {
    fun getById(ctx: Ctx, id: String): Timesheet          // throws NotFound
    fun list(ctx: Ctx, f: TimesheetFilter): Page<Timesheet>
    fun save(ctx: Ctx, ts: Timesheet)
}
```

## Public API enforcement

- Kotlin `internal` on adapters if domains are separate Gradle modules; Java: JPMS `module-info.java` exporting only the domain root package, or package-private adapters.
- ArchUnit tests in CI for everything else.

## Boundary lint

ArchUnit:
```kotlin
@ArchTest val platformIndependent = noClasses().that().resideInAPackage("..platform..")
    .should().dependOnClassesThat().resideInAnyPackage("com.acme.timesheet..", "com.acme.payroll..")

@ArchTest val pureCore = noClasses().that().resideInAnyPackage("..entities..")
    .or().haveSimpleNameEndingWith("UseCaseImpl")
    .should().dependOnClassesThat().resideInAnyPackage(
        "java.sql..", "jakarta.persistence..", "org.springframework..", "com.stripe..", "okhttp3..")

@ArchTest val noAdapterReach = noClasses().that().resideOutsideOfPackage("com.acme.timesheet..")
    .should().dependOnClassesThat().resideInAnyPackage(
        "com.acme.timesheet.storage..", "com.acme.timesheet.gateway..", "com.acme.timesheet.transport..")
```

## Errors

Exception hierarchy (or Kotlin sealed `Result` types — pick one per codebase).

```kotlin
enum class Kind { INVALID, NOT_FOUND, CONFLICT, UNAUTHORIZED, FORBIDDEN, RATE_LIMITED, UNAVAILABLE, INTERNAL }
open class DomainError(val kind: Kind, val code: String, msg: String, cause: Throwable? = null) : RuntimeException(msg, cause)
class CardDeclined(cause: Throwable? = null) : DomainError(Kind.INVALID, "card_declined", "card was declined", cause)

// gateway adapter
try { client.charges().create(params) }
catch (e: CardException) { throw CardDeclined(e) }
catch (e: RateLimitException) { throw RateLimited(e) }
catch (e: StripeException) { throw Unavailable(e) }
```

Transport: one `@RestControllerAdvice` / Ktor `StatusPages` mapping `kind` → status.

## Context

Explicit `ctx` parameter (preferred), or Kotlin coroutine context elements. Avoid raw `ThreadLocal` with coroutines/virtual threads unless wrapped by `platform/context`. Spring request-scoped beans only in transport/setup.

## Transactions

```kotlin
interface TxManager { fun <T> withinTx(ctx: Ctx, work: (Ctx) -> T): T }
```
Adapter implemented with Spring `TransactionTemplate` or jOOQ `transactionResult`. **No `@Transactional` on useCase** — the annotation couples business logic to Spring and hides boundaries.

## Wiring

Spring: `@Configuration` class per domain in `setup` (`@Bean fun timesheetUseCase(...)`) building plain classes. useCase classes carry **no** Spring annotations. Controllers (`@RestController`) are transport. Manual wiring or Koin works the same way.

JPA `@Entity` classes are storage-adapter models (in `storage.jpa`), never domain entities.

## Testing

- In-memory fakes implementing ports.
- Contract suite as an abstract JUnit class (`abstract class TimesheetStorageContract { abstract fun storage(): TimesheetStorage ... }`) extended per adapter.
- Testcontainers for DB adapters; ArchUnit in the normal test run.

## Naming & tooling

- `PascalCase` types, `camelCase` methods; `UseCase` suffix on the port.
- ktlint/detekt or Spotless + Checkstyle/Error Prone; Gradle version catalogs with locked versions; ArchUnit + tests as CI gates.

## Traps

- `@Transactional`, `@Service`, `@Autowired` on useCase.
- JPA entities with lazy relations leaking out of storage.
- Controllers containing business `if`s.
