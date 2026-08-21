---
name: review-gateway
description: Repo-type checklist for API gateway and edge PRs — rate limiting and quotas, API key lifecycle, header trust boundaries, edge caching, and routing defaults. Layer on top of review-core AND review-service.
---

# API gateway / edge review checklist

**Load `review-service` as well.** A gateway is still a service: its API contract, auth, resilience, and observability rules all apply. This file adds the concerns that are specific to sitting at the edge, where a mistake is exposed to the whole internet and affects every downstream service at once.

The framing question for this profile: **what does this diff let an unauthenticated stranger, or one abusive customer, do to everything behind the gateway?**

## Rate limiting and quotas

- **A new publicly reachable route with no rate limit** → **BLOCKER (SECURITY/PERF)**. Name the existing limit tier it should join. "It's low traffic" is not a reason; the limit exists for the abusive case, not the normal one.
- Removing, disabling, or raising an existing limit with no stated capacity reasoning → **BLOCKER**.
- Limit keyed on IP alone for an authenticated route → SHOULD-FIX. Key on API key / tenant / account, or one customer behind a NAT starves the rest and one attacker rotating IPs bypasses it entirely.
- Limit held in per-instance memory in a multi-replica deployment → SHOULD-FIX, because the effective limit is silently `N × configured`. **BLOCKER** when it is the only abuse control on that route.
- Expensive endpoints (search, export, report generation, anything LLM-backed) sharing a bucket with cheap ones → SHOULD-FIX. Recommend a cost-weighted bucket and say roughly what the expensive call costs.
- No `429` path, or `429` returned without `Retry-After` and the rate-limit headers the rest of the gateway sets → SHOULD-FIX. Clients that can't see the limit will hammer it.
- Limit applied *after* the expensive work (auth lookup, DB read, upstream call) rather than before → PERF SHOULD-FIX; the limiter has to be cheap or it becomes the amplifier.
- No global backstop — per-tenant quota, concurrency cap, or circuit breaker to the upstream — for a new route that fans out → SHOULD-FIX.
- Burst/window semantics changed (fixed window → sliding, or window length altered) without noting the effect on existing clients → SHOULD-FIX.

## API keys and credentials

- **Keys stored in plaintext or reversibly encrypted** → **BLOCKER (SECURITY)**. Store a hash; keep a short non-secret prefix for lookup and display (`sk_live_a1b2…`).
- Key compared with `==` / non-constant-time equality → SECURITY SHOULD-FIX (timing oracle).
- **Key in a URL path or query string** → **BLOCKER (SECURITY)**. It lands in access logs, proxy logs, browser history, and `Referer` headers. Keys go in a header.
- Key echoed into logs, traces, metrics labels, or an error response body → **BLOCKER (SECURITY)**. Check the *error* paths specifically; that's where keys leak.
- Key generated with a non-cryptographic RNG, or with insufficient entropy → **BLOCKER (SECURITY)**.
- **No revocation on the hot path** — a revoked key keeps working until a cache TTL expires, with no invalidation signal → **BLOCKER**. Revocation that takes effect "within 5 minutes" is not revocation during an incident.
- No rotation path: the customer cannot hold two active keys and retire the old one → SHOULD-FIX. Without overlap, every rotation is an outage, so nobody rotates.
- No expiry and no last-used timestamp → SHOULD-FIX. You cannot clean up what you cannot see is dead.
- A new key type that grants full access where scoped permissions already exist → SHOULD-FIX; scope it.
- Key validated at the edge but the identity it establishes is not passed downstream in a *trusted* way (see header trust below) → SHOULD-FIX.
- No per-key usage attribution in metrics → OBSERVABILITY SHOULD-FIX. When traffic spikes at 3am, "which key" is the first question.

## Header and identity trust boundary

This is where gateways get breached, so check it on every routing change.

- **Client-supplied headers that downstream services trust** (`X-User-Id`, `X-Tenant-Id`, `X-Forwarded-For`, `X-Api-Key-Scopes`, `X-Internal-*`) not stripped and re-set at the edge → **BLOCKER (SECURITY)**. If a caller can send `X-User-Id: 1` and reach an upstream that believes it, the whole auth model is bypassed.
- `X-Forwarded-For` appended to rather than anchored at the trusted proxy hop, then used for rate limiting or geo/IP allowlisting → **BLOCKER (SECURITY)** — spoofable limiter key.
- A new route added into an **allowlist-by-default** config, or auth applied by denylist rather than allowlist → **BLOCKER**. The default for a new route must be "authenticated" and require an explicit opt-out.
- Internal/admin/debug routes reachable through the public listener → **BLOCKER (SECURITY)**.
- mTLS or signed-request verification between gateway and upstream weakened or bypassed → BLOCKER.

## Edge caching

- Response caching enabled on an **authenticated** route without identity in the cache key (or a correct `Vary`) → **BLOCKER (DATA/SECURITY)**. This serves one customer's data to another and is very hard to detect after the fact.
- `Cache-Control` set at the edge that contradicts what the upstream sets → SHOULD-FIX; say which wins.
- Cacheable error responses (a 500 cached for the TTL) → SHOULD-FIX.
- No cache-busting or purge path for a newly cached resource → SHOULD-FIX.

## Routing, limits, resilience

- New upload or body-accepting route without a request size limit → SHOULD-FIX; **BLOCKER** if it streams straight to an upstream or to disk.
- Gateway timeout longer than the upstream's → orphaned work and connection pileup; shorter than the upstream's → premature `504` while the upstream still commits the write. Either mismatch → SHOULD-FIX; **BLOCKER** when the upstream operation is non-idempotent (the client retries a write that actually succeeded).
- Retries at the gateway on a non-idempotent upstream method → **BLOCKER (CORRECTNESS)**.
- CORS origins widened, especially with `Allow-Credentials` → **BLOCKER (SECURITY)** unless the origin list is explicit and justified.
- New route with no upstream health check, no circuit breaker, or no fallback where siblings have one → SHOULD-FIX.
- Routing config change with no staged rollout or rollback note, on a shared edge → PROCESS SHOULD-FIX. Everything is behind this thing.

## Observability specific to the edge

- New route not covered by the gateway's standard request metrics (status class, latency histogram, per-key counter) → OBSERVABILITY SHOULD-FIX.
- Rate-limit rejections not counted separately from other `4xx` → SHOULD-FIX; you can't distinguish "abuse blocked" from "clients broken".
- Trace/correlation ID not generated at the edge and propagated downstream → SHOULD-FIX. The gateway is the only place it can be minted reliably.
