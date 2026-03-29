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
- flag data consistency risks, lost updates, duplicate writes, stale reads, or partial commits
- flag logic that assumes connection, session, or transaction behavior in a fragile way
- flag persistence changes that may require coordinated migration, backfill, or compatibility handling
- flag unsafe assumptions around batching, retries, or pooled connection reuse when they affect correctness
- flag changes that may bypass tenant/audit/versioning/soft-delete rules if such patterns are present

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
- flag template changes that may break the rendered output for existing data models or null/missing values
- flag missing null-safe operators (`!`, `??`, `!''`) that may cause template errors when model values are absent
- flag `<#assign>` or `<#global>` directives that shadow or overwrite variables from the data model unintentionally
- flag inline logic or complex expressions in templates that should be computed in the backing Java code instead
- flag template includes or macros (`<#include>`, `<#macro>`) that reference paths which may not exist in all environments
- flag unescaped user-supplied values rendered with `${...}` that may introduce XSS or injection risks in HTML output
- flag changes to shared macros, layouts, or base templates that may silently affect other pages or email templates
- flag hardcoded text or URLs in templates that should be externalized via message bundles or configuration

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
- flag missing tests for changed business behavior
- flag BDD-style JUnit tests that describe scenarios well but do not assert the real business outcome strongly enough
- flag tests that cover only happy paths when the code changes affect edge cases, retries, failures, or duplicate delivery
- flag missing integration coverage where Spring wiring, transactions, Kafka interaction, MongoDB persistence, Oracle persistence, or configuration behavior is central to correctness
- flag Karate tests that miss contract drift, validation changes, error semantics, or backward compatibility risks

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
