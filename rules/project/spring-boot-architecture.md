---
alwaysApply: false
paths:
  - "**/*.java"
---

# Spring Boot architecture conventions

## Layering

Always follow **controller -> service -> repository**. Never skip a layer.

- **Controller** - HTTP concerns only: request mapping, validation annotations, response
  status. No business logic, no repository access.
- **Service** - business logic and transaction boundaries.
- **Repository** - data access only.
- Controllers never inject repositories. Services never build HTTP responses.
- Map between layers with DTOs. Never return JPA entities from a controller.

## Dependency injection

- Use **constructor injection** through Lombok `@RequiredArgsConstructor`. Declare every
  collaborator as a `private final` field.
- **Never use `@Autowired`.** Not on fields, not on setters, not on constructors - on a
  constructor it is redundant noise.
- If `@Autowired` appears to be the only option, stop and raise it as a trade-off before
  writing it. It is never a silent default.
- **Circular dependencies are a design smell.** Fix the design - extract a third service,
  invert the dependency, or publish an event - rather than hiding the cycle behind field
  injection or `@Lazy`.
- When a constructor needs `@Qualifier` or has more than a handful of parameters, write the
  constructor explicitly instead of relying on Lombok.

## Transactions (ACID)

- Put `@Transactional` on the **service** method that owns the unit of work - never on the
  controller and never on the repository.
- Mark pure reads with `@Transactional(readOnly = true)`.
- Default to `Propagation.REQUIRED`. Use `REQUIRES_NEW` only for audit or logging rows that
  must survive a rollback.
- Keep transactions short: no remote calls, file I/O, messaging publishes, or long loops
  inside a transaction.
- Pick the isolation level deliberately. If it is not `READ_COMMITTED`, state why.
- Roll back on checked exceptions explicitly with `rollbackFor = Exception.class` where the
  default behaviour is wrong.

## Exception handling

- Define domain exceptions extending `RuntimeException`, one per meaningful failure category.
- Centralise handling in a single `@RestControllerAdvice` with one `@ExceptionHandler` per
  category.
- Return one standard error envelope everywhere: timestamp, code, message, path, and field
  errors when validation fails.
- Never swallow an exception. Never return `null` to signal a failure.
- Log once at the handling boundary, not at every layer.

## Async

- Use `@Async` for work that should not block the response: notifications, report generation,
  exports, outgoing integrations.
- Never call an `@Async` method as part of an open transaction - the work may run before the
  transaction commits.
- Return `CompletableFuture<T>` when the caller needs the result.
- Use the framework default executor. Do not add a custom `ThreadPoolTaskExecutor` unless
  the user asks for one.
- Deferred trade-off, not yet decided: the default `SimpleAsyncTaskExecutor` creates a new
  thread per task and is unbounded. Raise it if async volume grows or thread count becomes
  a problem.

## Custom annotations

- Create a custom annotation only when the same cross-cutting concern appears in three or
  more places - auditing, masking, tenancy, rate limiting.
- Keep the annotation itself free of logic; put behaviour in an AOP aspect.
