# Copilot Code Generation Instructions

These instructions guide GitHub Copilot during code writing (Chat, Completions, Inline Suggestions). They apply to all code generated in this Java/Spring Boot application.

For PR review instructions, see `copilot-reviewinstructions.md` in the repo root.

## Project Stack

- Java 17+, Spring Boot, Spring Framework
- Kafka (producers and consumers)
- Oracle DB with UCP connection pooling, JPA/Hibernate
- MongoDB with Spring Data Mongo
- FreeMarker templates for rendering
- Gradle build system
- Karate for API/contract testing, JUnit 5 for unit/integration tests

## Configuration & Environment

- Never hardcode connection strings, credentials, API URLs, feature flags, or environment-specific values. Use Spring `@Value("${...}")`, `@ConfigurationProperties`, or environment variables.
- Place defaults in `application.yml` with profile-specific overrides in `application-{profile}.yml`.
- Use Spring profiles (`@Profile`, `spring.profiles.active`) for environment-specific behavior. Do not use `if/else` blocks checking environment names in application code.
- Use structured logging (SLF4J + Logback) to stdout. Do not write log files directly. Do not use `System.out.println` or `e.printStackTrace()`.
- Use `@PreDestroy` for cleanup of resources. Do not rely on in-memory state surviving restarts.

## Security & Privacy

- Never log passwords, tokens, API keys, SSNs, credit card numbers, or email addresses in plain text. If logging request/response payloads, redact fields that may contain PII. Use `@ToString.Exclude` or a custom serializer for sensitive entity fields.
- Never place passwords, API keys, tokens, or certificates in source code, `application.yml`, or `build.gradle`. Use environment variables or a secrets manager.
- Use Bean Validation (`@Valid`, `@NotNull`, `@Size`, `@Pattern`) on controller request bodies and path/query parameters. Do not trust client-supplied IDs, roles, or permissions without server-side verification.
- Never concatenate user input into SQL or MongoDB query strings. Use JPA named parameters, `JdbcTemplate` placeholders (`?`), or Spring Data query methods. For native queries, always use bind parameters.
- In FreeMarker templates, use `?html` for HTML context, `?js_string` for JavaScript context. Never render user-supplied values with raw `${...}` in HTML output.
- REST endpoints that modify data or access sensitive resources should require authentication. Use Spring Security with `@PreAuthorize` or `SecurityFilterChain`.

## Observability & Resilience

- Use SLF4J with MDC for correlation IDs. Include `traceId`, `spanId`, and business context (e.g., `orderId`, `customerId`) in log messages. Use log levels appropriately: ERROR for failures requiring attention, WARN for recoverable issues, INFO for business events, DEBUG for troubleshooting.
- Every `RestTemplate`, `WebClient`, `HttpClient`, or JDBC call to an external system must configure both a connection timeout and a read timeout. Never use unbounded calls — they can hang threads and exhaust the thread pool.
- When calling external HTTP services, apply a circuit breaker (e.g., Resilience4j `@CircuitBreaker`) to prevent cascade failures.
- Use Spring Retry (`@Retryable`) or Resilience4j retry for transient failures. Always configure max attempts, backoff interval, and retryable exception types. Never retry non-idempotent operations without an idempotency guard.
- Never catch and ignore exceptions with an empty catch block. At minimum, log the exception. For unrecoverable errors, let them propagate to the global exception handler.
- When an optional dependency is unavailable, the primary business flow should continue. Use fallback methods or feature flags rather than failing the entire request.

## Spring Boot Conventions

- Use constructor injection (not field injection with `@Autowired`) for all dependencies. Use Lombok `@RequiredArgsConstructor` if the project uses Lombok.
- Use `@ConfigurationProperties` for type-safe configuration classes. Prefer Java records for immutable config binding in Spring Boot 3+.
- Use `@ControllerAdvice` with `@ExceptionHandler` for centralized error handling. Return consistent error response structures with meaningful error codes and messages.

## Kafka Conventions

- Design consumers to be idempotent — processing the same message twice should produce the same result. Use deduplication keys, upserts, or idempotency tokens.
- Configure a `DefaultErrorHandler` with a `DeadLetterPublishingRecoverer` for messages that fail after retries. Do not silently drop failed messages.
- Use backward-compatible schema changes for event DTOs. Add new optional fields rather than renaming or removing existing ones.
- For critical business operations where message loss is unacceptable, prefer manual offset commit (`AckMode.MANUAL_IMMEDIATE` or `RECORD`) over auto-commit.

## Database Conventions

- All Oracle schema changes must have corresponding Flyway migration scripts in `db/migration/`. Never rely on Hibernate auto-DDL (`ddl-auto`) in production.
- Use `@Version` on JPA entities that may be concurrently modified to prevent lost updates.
- All list/search queries should support pagination (`Pageable`). Never return unbounded result sets.
- Do not store or cache JDBC `Connection` or `EntityManager` references. Obtain them per-use from the pool and release promptly.
- When adding new MongoDB query patterns, ensure corresponding indexes exist.

## Testing Conventions

- Assert specific business values (e.g., `assertEquals(expectedPrice, result.getPrice())`), not just non-null or generic success.
- For every happy-path test, consider adding a failure-path test (invalid input, timeout, exception, duplicate).
- Use `@SpringBootTest`, `@DataJpaTest`, or `@EmbeddedKafka` to test real Spring wiring, transactions, and persistence for code that depends on framework behavior.
- API tests (Karate) should validate response body schema and key field values, not just status codes.
- For data layer integration tests, prefer Testcontainers (Oracle, MongoDB, Kafka) over in-memory fakes when the team's test infrastructure supports it — this catches driver-specific issues that H2 or embedded alternatives miss.
