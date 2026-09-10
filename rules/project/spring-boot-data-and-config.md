---
alwaysApply: false
paths:
  - "**/*.java"
  - "**/*.yml"
  - "**/*.yaml"
  - "**/*.properties"
  - "**/pom.xml"
  - "**/build.gradle"
---

# Spring Boot data access, logging, and configuration

## Data access: choose by expected data volume

- **CRUD operations** - use Spring Data JPA repositories.
- **Reports, analytics, bulk reads and writes** - use `NamedParameterJdbcTemplate`.
- **Threshold rule** - if a table will realistically stay under about 1000 rows, JPA is
  acceptable even for complex reads. Above that, or when a query aggregates, joins many
  tables, or returns wide result sets, use `NamedParameterJdbcTemplate` with an explicit
  `RowMapper`.
- Never build a report by looping over JPA entities with lazy associations - that produces
  N+1 queries. Write one query instead.
- Always use named parameters in SQL. Never concatenate values into a query string.

## Logging

- Use Lombok `@Log4j2` on classes that need logging; the logger field is `log`.
- Log at boundaries: mutating controller requests, external service calls, async job start
  and end, and exceptions that are caught and handled.
- Pass the exception object as the last argument - `log.error("...", ex)` - not just
  `ex.getMessage()`.
- Never log secrets, tokens, passwords, or full request bodies containing PII.
- **No extra dependency is needed.** `@Log4j2` is a Lombok annotation that generates
  `org.apache.logging.log4j.LogManager.getLogger(...)`. A standard Spring Boot project already
  has `log4j-api` on the classpath: `spring-boot-starter-logging` brings `log4j-to-slf4j`,
  which depends on `log4j-api`. The annotation therefore compiles and runs as-is.
- **Backend caveat:** with the default starter, the actual implementation is Logback and
  Log4j2 API calls are routed to SLF4J. Configure logging through `logback-spring.xml` or
  `logging.level.*` properties. A `log4j2.xml` file is ignored unless the project switches to
  `spring-boot-starter-log4j2`. Do not add that starter unless the user asks for a Log4j2
  backend.

## Configuration properties

- Never use `@Value` for application configuration.
- Bind configuration into a typed class or record with
  `@ConfigurationProperties(prefix = "...")`.
- Register it with `@ConfigurationPropertiesScan` or `@EnableConfigurationProperties`.
- Group related keys under one prefix and keep key names in kebab-case.
- Fail fast: annotate required fields with `@NotBlank` / `@NotNull` so startup fails on
  missing configuration instead of failing later at runtime.
