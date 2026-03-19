# Copilot Review Instructions

Review only for issues that are not typically caught by SonarQube, CheckStyle, SpotBugs/FindBugs, or Checkmarx.

Do not comment on formatting, style, naming, minor cleanup, generic lint findings, or common scanner-detectable issues.

Focus only on high-signal review comments about business logic, runtime behavior, integration risk, and test effectiveness in this Java/Spring application.

Prioritize:

- business logic mistakes and unintended behavior changes
- transaction boundary mistakes
- partial updates across Spring services, Kafka, and Oracle DB operations
- retry and idempotency risks
- state transition and workflow inconsistencies
- backward compatibility risks in APIs, events, DTOs, and persisted values
- configuration/profile-dependent behavior changes
- missing or weak regression/integration tests

Java / Spring / Spring Boot focus:
- flag business operations whose transactional boundaries do not match the intended unit of work
- flag service logic that may behave differently because of Spring proxying, self-invocation, async execution, event listeners, or lazy loading assumptions
- flag changes in defaults, properties, bean conditions, or profiles that may silently alter runtime behavior
- flag orchestration logic where one path was updated but related paths were missed

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

Testing focus:
- flag missing tests for changed business behavior
- flag BDD-style JUnit tests that describe scenarios well but do not assert the real business outcome strongly enough
- flag tests that cover only happy paths when the code changes affect edge cases, retries, failures, or duplicate delivery
- flag missing integration coverage where Spring wiring, transactions, Kafka interaction, Oracle persistence, or configuration behavior is central to correctness
- flag Karate tests that miss contract drift, validation changes, error semantics, or backward compatibility risks

Comment only when the concern is concrete, important, and specific to the diff.

Useful comment patterns:
- "This changes existing behavior because ..."
- "This transaction may not cover ..."
- "This path may duplicate effects if Kafka redelivers ..."
- "This sequence looks non-atomic if DB update succeeds but message publish fails ..."
- "This consumer appears to rely on ordering that may not be guaranteed ..."
- "This may behave differently under another Spring profile or bean condition ..."
- "This BDD test does not appear to verify ..."
- "This Karate scenario may miss a regression for ..."

If no high-signal concern exists, do not comment.