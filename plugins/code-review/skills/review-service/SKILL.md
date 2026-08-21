---
name: review-service
description: Repo-type checklist for backend service and API PRs (REST, gRPC, workers). Layer on top of review-core. Covers API contracts, auth, database and migrations, concurrency, resilience, and observability.
---

# Backend service / API review checklist

Layered on `review-core`. Severity definitions and the output template come from there — this file only adds *what to look for*.

## API contract

- **Breaking change without versioning.** Removed or renamed field, narrowed type, changed enum value, required request field added, status code changed for an existing case → **BLOCKER (API-CONTRACT)** unless the PR shows a version bump or a deprecation window. Say who breaks.
- Response shape inconsistent with sibling endpoints (error envelope, pagination style, casing) → MAINTAINABILITY SHOULD-FIX.
- New endpoint without pagination on a collection that can grow → PERF SHOULD-FIX (BLOCKER if backed by an unbounded table).
- Input validation missing or partial on a new endpoint — especially numeric bounds, string length, and enum membership → SHOULD-FIX; BLOCKER if the unvalidated value reaches a query, a filesystem path, or a shell.
- Non-idempotent handler for an operation clients will retry (payments, provisioning) → **BLOCKER (CORRECTNESS)**.

## AuthN / AuthZ

- Any new route, queue consumer, or admin action: confirm the authz check exists **and** that it scopes to the caller's tenant/org/user. A route that reads an ID from the request and fetches it without an ownership check → **BLOCKER (SECURITY)** — this is IDOR and it is the single most common real finding.
- Authorisation logic duplicated inline rather than going through the existing middleware/policy layer → SECURITY SHOULD-FIX.
- Permissive CORS (`*` with credentials), disabled TLS verification, or a debug/bypass flag readable from a request → **BLOCKER**.
- Secrets: hardcoded credentials, a real key in a test fixture, or a token/PII field written to logs → **BLOCKER (SECURITY)**.

## Database and migrations

- **Destructive or irreversible migration** (drop column/table, type narrowing, backfill with no batching) → **BLOCKER (DATA)** unless the PR includes a rollback path and a two-phase deploy plan.
- Migration that locks a large table (adding a non-null column with default, adding an index without `CONCURRENTLY` on Postgres) → **BLOCKER (DATA)** on any table with meaningful row count; say which.
- Code deployed before the migration it depends on (or vice versa) — check ordering assumptions → BLOCKER.
- New query without an index on its filter/join columns → PERF SHOULD-FIX; BLOCKER on a hot path.
- **N+1**: a query inside a loop or inside a per-item serialiser → PERF BLOCKER if the collection is unbounded, SHOULD-FIX otherwise.
- Missing transaction around a multi-write operation that must be atomic → **BLOCKER (DATA)**.
- Read-modify-write without optimistic locking or `SELECT … FOR UPDATE` where concurrent callers are realistic → **BLOCKER (DATA)**.
- String-interpolated SQL → **BLOCKER (SECURITY)**.

## Concurrency and resilience

- New outbound call with no timeout → **BLOCKER (PERF)**. No retry policy, or retries without backoff/jitter → SHOULD-FIX. Retries on a non-idempotent operation → BLOCKER.
- Removal of an existing timeout, rate limit, circuit breaker, or bulkhead → BLOCKER.
- Unbounded concurrency (`Promise.all` over an unbounded list, goroutine per item with no semaphore, thread pool with unbounded queue) → PERF BLOCKER.
- Shared mutable state touched from concurrent handlers without synchronisation → CORRECTNESS BLOCKER.
- Background work started without a shutdown path (in-flight work dropped on deploy) → SHOULD-FIX; BLOCKER if it drops user-visible writes.
- Queue consumer with no dead-letter path or no poison-message handling → SHOULD-FIX.

## Error handling

- Swallowed exception (`catch {}`, `except: pass`, `_ = err`) → CORRECTNESS SHOULD-FIX; BLOCKER if it hides a failed write.
- Internal error detail (stack trace, SQL, upstream body) returned to the caller → SECURITY SHOULD-FIX.
- Wrong status class — a client error returned as 500, or a failure returned as 200 → SHOULD-FIX.

## Observability

- New failure path with no log, no metric, and no trace span → OBSERVABILITY SHOULD-FIX. Ask: *during an incident, could I tell this happened?*
- Log lines without a correlation/request ID where the rest of the service has one → NIT.
- Unbounded-cardinality metric label (user ID, request ID) → SHOULD-FIX.
- New feature with no way to disable it (no flag) where the repo otherwise uses flags → PROCESS SHOULD-FIX.

## Tests

- New business logic branch with no test → TESTS SHOULD-FIX.
- Migration with no test or no verified rollback → SHOULD-FIX.
- Test asserting only on a mock being called, never on the resulting behaviour → NIT.
- Any test deleted, skipped, or given a widened tolerance in this PR → **BLOCKER (TESTS)** unless the PR explains why.

## Repo consistency

Compare against the rest of the service: error-handling helper, logging wrapper, config loading, DB access layer, validation library. A PR that introduces a second pattern for an already-solved concern is a MAINTAINABILITY SHOULD-FIX — name the existing one.
