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

## Architecture Principles (12-Factor)

When generating new services, controllers, or configuration:

- **Config via environment:** Never hardcode connection strings, credentials, API URLs, feature flags, or environment-specific values. Use Spring `@Value("${...}")`, `@ConfigurationProperties`, or environment variables. Place defaults in `application.yml` with profile-specific overrides in `application-{profile}.yml`.
- **Stateless processes:** Do not store user sessions, request context, or business state in instance fields or static variables. Use external stores (Redis, DB) for shared state. Spring beans should be stateless singletons unless explicitly scoped otherwise.
- **Port binding / self-contained:** Services should be self-contained Spring Boot applications with embedded servers. Do not assume external servlet containers.
- **Disposability:** Design for fast startup and graceful shutdown. Use `@PreDestroy` for cleanup. Do not rely on in-memory state surviving restarts. Kafka consumers should handle rebalancing gracefully.
- **Dev/prod parity:** Use the same backing services (Oracle, MongoDB, Kafka) in all environments. Avoid H2 or embedded databases in tests if the production database is Oracle — use Testcontainers instead where feasible.
- **Logs as streams:** Use structured logging (SLF4J + Logback) to stdout. Do not write log files directly. Do not use `System.out.println` or `e.printStackTrace()`.
- **Admin processes:** One-off tasks (migrations, backfills) should be runnable as separate Spring Boot commands or Flyway migrations, not embedded in the main application startup.

## Security & Privacy

When generating code that handles data, authentication, or external input:

- **Never log sensitive data.** Do not log passwords, tokens, API keys, SSNs, credit card numbers, or email addresses in plain text. If logging request/response payloads, redact fields that may contain PII. Use a `@ToString.Exclude` or custom serializer for sensitive entity fields.
- **Never hardcode secrets.** Do not place passwords, API keys, tokens, or certificates in source code, `application.yml`, or `build.gradle`. Use environment variables, Spring Cloud Config, or a secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager).
- **Validate all external input.** Use Bean Validation (`@Valid`, `@NotNull`, `@Size`, `@Pattern`) on controller request bodies and path/query parameters. Do not trust client-supplied IDs, roles, or permissions without server-side verification.
- **Use parameterized queries.** Never concatenate user input into SQL or MongoDB query strings. Use JPA named parameters, `JdbcTemplate` placeholders (`?`), or Spring Data query methods. For native queries, always use bind parameters.
- **Escape output in templates.** In FreeMarker templates, use `?html` for HTML context, `?js_string` for JavaScript context. Enable auto-escaping where possible. Never render user-supplied values with raw `${...}` in HTML output.
- **Apply least privilege.** Database connections should use service-specific accounts with minimal required permissions. Do not use admin/root credentials for application connections.
- **Protect APIs.** REST endpoints that modify data or access sensitive resources should require authentication. Use Spring Security with appropriate method-level or URL-level authorization (`@PreAuthorize`, `SecurityFilterChain`).

## Observability & Resilience

When generating service code, external integrations, or infrastructure:

- **Structured logging.** Use SLF4J with MDC for correlation IDs. Include `traceId`, `spanId`, and business context (e.g., `orderId`, `customerId`) in log messages for traceability. Use log levels appropriately: ERROR for failures requiring attention, WARN for recoverable issues, INFO for business events, DEBUG for troubleshooting.
- **Timeouts on all external calls.** Every `RestTemplate`, `WebClient`, `HttpClient`, or JDBC call to an external system must configure both a connection timeout and a read timeout. Never use unbounded calls — they can hang threads indefinitely and exhaust the thread pool.
- **Circuit breakers for external dependencies.** When calling external HTTP services, apply a circuit breaker pattern (e.g., Resilience4j `@CircuitBreaker`) to prevent cascade failures. Configure sensible thresholds for failure rate, slow call rate, and wait duration.
- **Retry with backoff.** Use Spring Retry (`@Retryable`) or Resilience4j retry for transient failures. Always configure max attempts, backoff interval, and retryable exception types. Never retry non-idempotent operations without an idempotency guard.
- **Health checks.** Spring Boot Actuator health endpoints (`/actuator/health`) should include checks for all critical dependencies (database, Kafka, external APIs). Use custom `HealthIndicator` implementations for non-standard dependencies.
- **Graceful degradation.** When an optional dependency (e.g., a notification service, analytics API) is unavailable, the primary business flow should continue. Use fallback methods, default values, or feature flags rather than failing the entire request.
- **Do not swallow exceptions.** Never catch and ignore exceptions with an empty catch block. At minimum, log the exception. For recoverable errors, handle them explicitly. For unrecoverable errors, let them propagate to the global exception handler.

## Spring Boot Conventions

When generating Spring Boot code:

- **Constructor injection.** Use constructor injection (not field injection with `@Autowired`) for all dependencies. Use Lombok `@RequiredArgsConstructor` if the project uses Lombok.
- **Immutable configuration.** Use `@ConfigurationProperties` with `@ConstructorBinding` for type-safe, immutable configuration classes.
- **Layer separation.** Keep business logic in `@Service` classes. Controllers should only handle HTTP concerns (request validation, response mapping). Repositories should only handle data access. Do not put business logic in controllers or repositories.
- **Exception handling.** Use `@ControllerAdvice` with `@ExceptionHandler` for centralized error handling. Return consistent error response structures with meaningful error codes and messages.
- **Profiles.** Use Spring profiles (`@Profile`, `spring.profiles.active`) for environment-specific behavior. Do not use `if/else` blocks checking environment names in application code.

## Kafka Conventions

When generating Kafka producer or consumer code:

- **Idempotent consumers.** Design consumers to be idempotent — processing the same message twice should produce the same result. Use deduplication keys, upserts, or idempotency tokens.
- **Error handling.** Configure a `DefaultErrorHandler` with a `DeadLetterPublishingRecoverer` for messages that fail after retries. Do not silently drop failed messages.
- **Schema evolution.** Use backward-compatible schema changes for event DTOs. Add new optional fields rather than renaming or removing existing ones. Consider schema registry for Avro/Protobuf schemas.
- **Manual offset commit.** Prefer manual offset commit (`AckMode.MANUAL_IMMEDIATE` or `RECORD`) over auto-commit for critical business operations to prevent message loss.

## Database Conventions

When generating JPA, JDBC, or MongoDB code:

- **Flyway for Oracle migrations.** All Oracle schema changes must have corresponding Flyway migration scripts in `db/migration/`. Never rely on Hibernate auto-DDL (`ddl-auto`) in production.
- **Optimistic locking.** Use `@Version` on JPA entities that may be concurrently modified. This prevents lost updates without the overhead of pessimistic locking.
- **Pagination.** All list/search queries should support pagination (`Pageable`). Never return unbounded result sets.
- **Connection pool awareness.** Do not store or cache JDBC `Connection` or `EntityManager` references. Obtain them per-use from the pool and release promptly.
- **MongoDB indexes.** When adding new query patterns, ensure corresponding indexes exist. Document index requirements in comments or migration scripts.

## Testing Conventions

When generating test code:

- **Test the business outcome.** Assert specific business values (e.g., `assertEquals(expectedPrice, result.getPrice())`), not just non-null or generic success (`assertNotNull(result)`).
- **Test failure paths.** For every happy-path test, consider adding a failure-path test (invalid input, timeout, exception, duplicate).
- **Integration tests for wiring.** Use `@SpringBootTest`, `@DataJpaTest`, or `@EmbeddedKafka` to test real Spring wiring, transactions, and persistence — not just mocked-out unit tests — for code that depends on framework behavior.
- **Karate for API contracts.** API tests should validate response body schema and key field values, not just status codes.
- **Testcontainers over mocks for data.** Prefer Testcontainers (Oracle, MongoDB, Kafka) over in-memory fakes for data layer integration tests. This catches driver-specific and query-specific issues that H2 or embedded alternatives miss.
