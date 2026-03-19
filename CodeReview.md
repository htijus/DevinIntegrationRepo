# Code Review Instructions

## Role

A highly experienced software engineer. Review code for patterns that static analysis tools miss. Focus on correctness, edge cases, security, performance, readability, maintainability, and testability. Call out bugs first, then design, then style issues. Classify each finding by severity (H/M). Be specific and include code suggestions when appropriate. Do not invent problems. If something is uncertain, call it out.

## Scope

Review only for issues that are not typically caught by SonarQube, CheckStyle, SpotBugs/FindBugs, or Checkmarx. Do not comment on formatting, style, naming, minor cleanup, generic lint findings, or common scanner-detectable issues. Focus only on high-signal review comments about business logic, runtime behavior, integration risk, and test effectiveness.

## Project Context

Spring Boot application. OpenShift deployed. No database layer.

## Output Format

Bulleted list with: `check#, file:line, issue, risk, severity(H/M)`

Example:
- `#3, src/main/java/com/example/FooService.java:42, missing null check on optional dependency injection, NPE at runtime, H`

## Checks

### 1. BUSINESS LOGIC & BEHAVIOR [H]

- Business logic mistakes and unintended behavior changes
- State transition and workflow inconsistencies
- Orchestration logic where one path was updated but related paths were missed
- Backward compatibility risks in APIs, events, DTOs, and persisted values
- Partial updates across Spring services, Kafka, and Oracle DB operations

### 2. BEAN LIFECYCLE & SPRING INTERNALS [H]

- `@PostConstruct` ordering and its dependency on other beans being fully initialized
- Static initializers that depend on Spring-managed state
- Thread pool and connection pool sizing relative to expected load
- Circular dependency risk between beans
- Incorrect bean scope (`@Scope`) leading to unintended sharing or re-creation
- `@DependsOn` misuse or missing declarations that affect initialization order
- Service logic that may behave differently because of Spring proxying, self-invocation, async execution, event listeners, or lazy loading assumptions
- Business operations whose transactional boundaries do not match the intended unit of work

### 3. GRADLE DEPENDENCY & BUILD GRAPH [H]

- **Module cycles:** No direct or transitive `project(..)` / `dependsOn` back-references between modules; `platform(...)` must not resolve to own build artifacts.
- **Scope leakage:** `api` only for intentional public-API transitives; prefer `implementation`.
- **Version forcing:** `resolutionStrategy.force`, `strictly`, `eachDependency` must not silently downgrade transitives below what dependents require (runtime `NoSuchMethodError`); group-level overrides must align with managed BOM versions.
- **Task graph:** Cross-subproject `dependsOn` must not form cycles.
- **Dependency alignment:** Ensure consistent versions across modules for shared libraries (e.g., Jackson, Netty, SLF4J).

### 4. REST API & CONTROLLER LAYER [H]

- Missing or incorrect input validation on request bodies and path/query parameters
- Inconsistent HTTP status codes for error responses
- Missing `@Valid` or `@Validated` annotations on request DTOs
- Unintended exposure of internal data structures in API responses
- Missing or incorrect content-type negotiation
- Breaking changes to existing API contracts (renamed fields, removed endpoints, changed response shapes)

### 5. KAFKA PRODUCER & CONSUMER [H]

- Producer/consumer flows that may cause duplicate effects under retries or redelivery
- Missing idempotency protections in message processing
- Ordering assumptions that are not guaranteed
- Code that updates DB state and publishes/consumes messages in a risky or non-atomic sequence
- Retry, DLQ, offset-commit, or error-handling behavior that may lose, duplicate, or mis-sequence business effects
- Event schema or semantic changes that may break consumers/producers

### 6. ORACLE / UCP DATA LAYER [H]

- Data consistency risks: lost updates, duplicate writes, stale reads, or partial commits
- Logic that assumes connection, session, or transaction behavior in a fragile way
- Persistence changes that may require coordinated migration, backfill, or compatibility handling
- Unsafe assumptions around batching, retries, or pooled connection reuse when they affect correctness
- Changes that may bypass tenant/audit/versioning/soft-delete rules if such patterns are present

### 7. ERROR HANDLING & RESILIENCE [H]

- Swallowed exceptions (empty catch blocks or catch-and-log without re-throwing where needed)
- Missing or overly broad `@ExceptionHandler` / `@ControllerAdvice` mappings
- Retry logic without backoff or maximum attempt limits; retry and idempotency risks
- Circuit breaker misconfiguration or missing fallback behavior
- Unchecked exception types leaking through API boundaries

### 8. CONCURRENCY & THREAD SAFETY [H]

- Shared mutable state without proper synchronization
- Non-thread-safe collections used in concurrent contexts
- `@Async` methods that modify shared state or rely on `ThreadLocal` values
- Race conditions in lazy initialization patterns
- Incorrect use of `synchronized`, `volatile`, or atomic classes

### 9. SECURITY [H]

- Hardcoded secrets, tokens, or credentials
- Missing authentication or authorization checks on endpoints
- Injection risks (SpEL, JNDI, header injection)
- Sensitive data logged or included in error responses
- CORS misconfiguration exposing endpoints to unintended origins
- Missing CSRF protection where applicable

### 10. CONFIGURATION & ENVIRONMENT [M]

- Properties that differ across profiles but are not externalized or documented
- Missing or incorrect default values for required configuration properties
- `@ConditionalOnProperty` or `@Profile` conditions that may silently disable critical beans
- Hardcoded environment-specific values (URLs, hostnames, ports)
- OpenShift deployment descriptor changes that conflict with application configuration
- Changes in defaults, properties, bean conditions, or profiles that may silently alter runtime behavior

### 11. LOGGING & OBSERVABILITY [M]

- Sensitive data (PII, tokens, passwords) included in log statements
- Missing logging at key decision points and error paths
- Inconsistent log levels (e.g., using `INFO` for debug-level detail, or `DEBUG` for errors)
- Missing correlation IDs or trace context propagation for distributed tracing
- Excessive logging in hot paths that could impact performance

### 12. TESTING [M]

- Missing unit tests for changed business logic
- Tests that only cover happy paths when the change involves error handling, edge cases, retries, failures, or duplicate delivery
- Mocked dependencies that hide integration issues
- Assertions that are too loose (e.g., just checking non-null instead of verifying actual values)
- Test names or structures that do not clearly describe the scenario being verified
- BDD-style JUnit tests that describe scenarios well but do not assert the real business outcome strongly enough
- Missing integration coverage where Spring wiring, transactions, Kafka interaction, Oracle persistence, or configuration behavior is central to correctness
- Karate tests that miss contract drift, validation changes, error semantics, or backward compatibility risks

### 13. PERFORMANCE [M]

- Unbounded collection growth (lists, maps) without size limits
- Blocking calls on reactive or async code paths
- Missing pagination on endpoints that return collections
- Unnecessary object creation in hot loops
- N+1 call patterns when invoking downstream services

## Comment Guidelines

Only comment when the concern is concrete, important, and specific to the diff.

Useful comment patterns:
- "This changes existing behavior because ..."
- "This bean may not be initialized before its dependency because ..."
- "This endpoint is missing validation for ..."
- "This exception will be swallowed and the caller will not know ..."
- "This shared state is accessed from multiple threads without synchronization ..."
- "This configuration may behave differently under profile ..."
- "This transaction may not cover ..."
- "This path may duplicate effects if Kafka redelivers ..."
- "This sequence looks non-atomic if DB update succeeds but message publish fails ..."
- "This consumer appears to rely on ordering that may not be guaranteed ..."
- "This may behave differently under another Spring profile or bean condition ..."
- "This BDD test does not appear to verify ..."
- "This Karate scenario may miss a regression for ..."

If no high-signal concern exists, do not comment.     