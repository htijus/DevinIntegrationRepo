# Copilot Review Instructions

Review only for issues that are not typically caught by SonarQube, CheckStyle, SpotBugs/FindBugs, or Checkmarx.

Do not comment on formatting, style, naming, minor cleanup, generic lint findings, or common scanner-detectable issues.

Focus only on high-signal review comments about business logic, runtime behavior, integration risk, and test effectiveness in this Java/Spring application.

Prioritize:

- business logic mistakes and unintended behavior changes
- transaction boundary mistakes
- partial updates across Spring services, Kafka, MongoDB, and Oracle DB operations
- retry and idempotency risks
- state transition and workflow inconsistencies
- backward compatibility risks in APIs, events, DTOs, and persisted values
- configuration/profile-dependent behavior changes
- missing or weak regression/integration tests
- FreeMarker template rendering errors from null or missing model values
- Gradle build configuration changes that silently break dependencies or the build graph

Java / Spring / Spring Boot focus:
- flag methods that perform multiple write operations (repository saves, template updates, external calls) without a shared `@Transactional` boundary — ask if atomicity is intended
- flag `@Transactional(readOnly = true)` on methods that perform write operations
- flag self-invocation (`this.method()`) where the target method has `@Transactional`, `@Cacheable`, `@Async`, or `@Retryable` — these annotations are bypassed when not called through the Spring proxy
- flag `@Async` methods that read `SecurityContext`, MDC, or other `ThreadLocal` values without explicit propagation
- flag `@TransactionalEventListener` or `@EventListener` methods where execution order or transactional participation is assumed but not explicitly configured
- flag additions or changes to `@ConditionalOnProperty`, `@ConditionalOnBean`, `@Profile`, or `@Conditional` annotations that may enable or disable beans in specific environments
- flag removal or renaming of configuration properties that existing deployments may depend on
- flag changes to default values of properties that are not overridden in all active profiles
- flag if a method handling one case of a switch, enum dispatch, or strategy pattern was updated but other cases in the same dispatch were not touched — ask if they need the same change

Kafka focus:
- flag producer/consumer flows that may cause duplicate effects under retries or redelivery
- flag missing idempotency protections in message processing
- flag ordering assumptions that are not guaranteed
- flag code that updates DB state and publishes/consumes messages in a risky or non-atomic sequence
- flag retry, DLQ, offset-commit, or error-handling behavior that may lose, duplicate, or mis-sequence business effects
- flag event schema or semantic changes that may break consumers/producers

Oracle / UCP focus:
- flag read-then-write sequences (select followed by update/save) on the same entity without optimistic locking (`@Version`) or explicit pessimistic locking (`SELECT ... FOR UPDATE`) — ask if concurrent modification is possible
- flag multiple repository save/update calls in a method without a shared `@Transactional` boundary — a failure between saves will leave partial state
- flag code that stores or caches a JDBC `Connection`, `DataSource`, or `EntityManager` reference in a field or static variable — pooled connections should be obtained per-use and released promptly
- flag code that calls Oracle session-level operations (`ALTER SESSION`, `DBMS_SESSION`, package-level state) without ensuring the same connection is used for the full operation — UCP may return a different connection on the next call
- flag additions, removals, or renames of JPA entity fields (`@Column`, `@JoinColumn`) where no corresponding Flyway/Liquibase migration script is included in the same PR — ask if the migration is tracked separately
- flag changes to column constraints (nullable, unique, length, default) on JPA entities that may conflict with existing data in production
- flag batch insert/update operations (`JdbcTemplate.batchUpdate`, `saveAll`) that do not handle partial failures — if row 50 of 100 fails, are the first 49 committed or rolled back?
- flag retry logic around database operations that are not idempotent — a retried `INSERT` without an `ON CONFLICT`/`MERGE` guard may create duplicate rows
- flag raw SQL queries (native queries, `@Query` with `nativeQuery=true`, `JdbcTemplate`) that bypass the JPA entity layer — these may skip Hibernate filters, `@Where` clauses, or entity listeners that enforce tenant isolation, soft-delete, or audit logging
- flag direct `DELETE` statements on entities that use `@SQLDelete` or `@Where` annotations for soft-delete — the delete should use the repository method that applies the soft-delete logic

MongoDB focus:
- flag queries missing indexes or performing collection scans on large collections
- flag read/write concern levels that do not match the consistency requirements of the operation
- flag multi-document operations that require atomicity but are not wrapped in a transaction
- flag schema changes (added/removed/renamed fields) that may break existing queries, aggregations, or downstream consumers
- flag unbounded queries missing projection, limit, or pagination that could return excessive data
- flag incorrect use of `$set` vs `$unset` vs `$push` that may corrupt document structure
- flag MongoTemplate or repository methods that silently upsert when only an update is intended, or vice versa
- flag missing null/empty checks before persisting embedded documents or arrays that could create inconsistent state
- flag connection pool or timeout configuration changes that may affect performance or availability under load
- flag aggregation pipelines that perform unbounded `$lookup`, `$unwind`, or `$group` stages without size guards

FreeMarker template focus:
- flag newly added or modified `${...}` expressions that do not use null-safe operators (`!`, `??`, `!''`) — these will throw a runtime error if the model value is absent
- flag `<#assign>` or `<#global>` directives that use the same variable name as a known data model parameter passed from the Java controller — ask if the shadowing is intentional
- flag template expressions that contain business rule calculations, multi-step arithmetic, or nested conditional chains (3+ levels of `<#if>`/`<#elseif>`) — these are harder to test and debug in templates and are better placed in the Java layer
- flag `<#include>` or `<#import>` directives that reference absolute paths or paths outside the current template directory — relative paths are more portable across environments
- flag `${...}` expressions in HTML-context templates (producing tags, attribute values, or inline scripts) that do not use `?html`, `?js_string`, or auto-escaping — these may introduce XSS if the value contains user input
- flag changes to files in shared template directories (e.g., `/layouts/`, `/macros/`, `/common/`) or to `<#macro>` definitions that are imported by other templates — note that these changes affect all consumers
- flag hardcoded URLs (e.g., `https://...`, `http://...`) in templates that point to environment-specific endpoints — these should come from configuration
- flag newly added user-visible text strings in templates only if the project already uses message bundles (e.g., Spring `MessageSource`, `<@spring.message>`) — ask if internationalization is needed

Gradle build files focus:
- flag `build.gradle` or `settings.gradle` changes that introduce dependency version conflicts or override managed BOM versions
- flag new dependencies added without specifying a version when no dependency management plugin or platform is in place
- flag `api` vs `implementation` scope misuse — `api` should only be used for intentionally transitive public dependencies
- flag `resolutionStrategy.force` or `strictly` constraints that may silently downgrade transitive dependencies and cause runtime errors (e.g., `NoSuchMethodError`)
- flag custom task definitions or `dependsOn` declarations that may introduce cycles in the task graph
- flag plugin version upgrades that may change default behavior (e.g., Spring Boot plugin, Shadow/Fat JAR plugin)
- flag changes to `sourceSets`, `processResources`, or `test` configurations that may silently exclude files or break the build
- flag Gradle wrapper (`gradle-wrapper.properties`) changes that upgrade/downgrade the Gradle version without coordinating with CI and team tooling
- flag hardcoded file paths, environment-specific values, or credentials in build scripts

Testing focus:
- flag changed or newly added public methods in service/domain classes that have no corresponding test method in the PR — ask if existing tests already cover the change
- flag test methods whose name describes a specific business scenario (e.g., `shouldCalculateDiscount`, `shouldRejectDuplicateOrder`) but whose assertions only check generic outcomes like `assertNotNull`, `assertTrue(result)`, or `verify(mock).call()` without asserting the specific business value (e.g., the discount amount, the rejection reason, the resulting state)
- flag if the production code diff adds or modifies exception handling, retry logic, fallback behavior, or conditional error paths, but the test diff contains no corresponding test for the failure/error scenario
- flag if the production code diff introduces or modifies `@Transactional` boundaries, Kafka consumer/producer wiring, `MongoTemplate`/repository queries, `JdbcTemplate`/native queries, or `@ConditionalOnProperty`/`@Profile` logic, but the test diff only contains unit tests with mocked dependencies — ask if an integration test (e.g., `@SpringBootTest`, `@DataJpaTest`, `@EmbeddedKafka`) is needed
- flag if the production code diff changes REST controller response fields, status codes, error response structure, or validation rules, but Karate `.feature` files in the PR do not update the corresponding match/assert statements to reflect the change
- flag Karate scenarios that assert only status 200 without validating the response body schema or key field values

Comment only when the concern is concrete, important, and specific to the diff.

Useful comment patterns (one per domain — shows tone and specificity expected):
- "This changes existing behavior because ..." *(general)*
- "This transaction may not cover ..." *(Spring / data layer)*
- "This path may duplicate effects if Kafka redelivers ..." *(Kafka)*
- "This may behave differently under another Spring profile or bean condition ..." *(configuration)*
- "This BDD test does not appear to verify ..." *(testing)*
- "This MongoDB query may perform a collection scan because ..." *(MongoDB)*
- "This FreeMarker template will fail at render time if the model is missing ..." *(FreeMarker)*
- "This Gradle dependency change may conflict with the managed BOM because ..." *(Gradle)*

If no high-signal concern exists, do not comment.
