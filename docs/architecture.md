# Architecture

Last updated: 2026-11-13

Interchange is the single governed junction between external partner systems and a project's core systems. This document describes how a request actually flows through the platform, how the connector plugin model works, how the job engine and dead-letter queue behave, the two-database model, and the dormant-by-default posture.

## Stack

Next.js 16 (App Router) on Vercel, Drizzle ORM, Neon serverless Postgres, TypeScript, zod for schema validation. There is no separate worker process and no Redis: the job queue lives in Postgres itself.

## The core idea

Partners never talk to a project's internal systems directly. They talk to Interchange. Interchange authenticates them, validates their payload, runs (or queues) the work, and logs everything. This gives a project one place to see what happened across every partner integration, instead of that logic being scattered across bespoke handlers.

## Inbound request lifecycle

An inbound request is a partner calling into the platform, for example to submit a record or notify of an event. By default it is handled synchronously: the caller is HTTP-waiting, so the connector's `execute()` runs inline. A partner that cannot hold a connection open may instead ask for the call to be accepted and run in the background (`Prefer: respond-async`, see "Async inbound (202 Accepted)" below): the platform then answers 202 and runs the job right after the response.

```
Partner                     Interchange                              Data
  |                             |                                      |
  | POST /api/inbound/          |                                      |
  |   <systemCode>/<endpoint>   |                                      |
  |----------------------------->                                      |
  |                             | 0. request id: honour X-Request-ID   |
  |                             |    or generate one; echoed back     |
  |                             |    and stored on job.request_id      |
  |                             | 0b. per-IP rate limit (bucket        |
  |         429 + Retry-After   |    ip:<addr>, default 250/min)       |
  |<-----------------------------                                      |
  |                             | 1. authenticatePartner(headers, code)|
  |                             |    - credential from x-api-key or    |
  |                             |      Authorization (Bearer / Basic)  |
  |                             |    - look up a live (not deleted)    |
  |                             |      external_system by URL code     |
  |                             |    - require status = ACTIVE         |
  |                             |    - find the credential by lookup   |
  |                             |      hash (or keyed username         |
  |                             |      lookup), verify timing safe     |
  |                             |    - reject revoked or expired       |
  |                             |    - stamp last_used_at on the match |
  |         401                 |  (all pre-auth failures: one body)   |
  |<-----------------------------                                      |
  |         403                 |  (confinement: owner in log only)    |
  |<-----------------------------                                      |
  |                             | 1a. service token, only when the     |
  |       401 / 403 / 503       |    type policy is partner+service    |
  |<-----------------------------    (X-Service-Token, HS256 JWT)      |
  |                             | 1b. per-system rate limit (bucket    |
  |         429 + Retry-After   |    sys:<code>, external_system       |
  |<-----------------------------    .rate_limit_rpm)                  |
  |                             | 1c. endpoint: normalise the segment; |
  |         404 / 429           |    if the system declares limits,    |
  |<-----------------------------    only those names pass, each with  |
  |                             |    its own bucket sys:<code>:<ep>    |
  |                             | 2. handler from the integration type |
  |                             |    (joined during auth),             |
  |                             |    resolveHandler()                  |
  |                             |    (static generated manifest)       |
  |                             | 2b. Idempotency-Key, then JSON body  |
  |         400                 |    (invalid key / bad JSON)          |
  |<-----------------------------                                      |
  |                             | 3. connector.inputSchema parsed with |
  |         422                 |    the top level strict (unknown     |
  |<-----------------------------    keys: names only in the reply)    |
  |                             | 4. createJob(...): INSERT ... ON     |
  |                             |    CONFLICT DO NOTHING on            |
  |                             |    (system_code, idempotency_key),   |
  |                             |    status=PROCESSING (inline) or     |
  |                             |    PENDING + accepted_async (async)  |
  |  replay, same key:          |------------------------------------->|
  |  200 result, 500 failed,    |<-------------------------------------|
  |  or 202 while still queued, |                                      |
  |  + Idempotent-Replayed: true|                                      |
  |<-----------------------------                                      |
  |                             | 5. connector.execute(ctx) runs INLINE|
  |                             | 6. markSuccess / markFailed          |
  |    200 (or 502 / 500)       |------------------------------------->|
  |<-----------------------------                                      |
  |                             | Async branch: the payload is sealed  |
  |  202 {jobId, PENDING} +     |    in job_payload_vault (same txn),  |
  |  Preference-Applied         |    the 202 goes out, then the job is |
  |<-----------------------------    claimed by id and run (5 and 6)   |
  |                             | Every exit above (incl. 401/429/4xx  |
  |                             | also writes an activity_log row      |
```

In order, the inbound chain is: request id, per-IP limit, partner auth, service token (by policy), per-system limit, endpoint allowlist and limit, handler, `Idempotency-Key`, JSON, strict parse, job. A cross-origin browser preflight (`OPTIONS`) takes a separate, shorter path: per-IP limit, then the per-system CORS decision (see `docs/security.md`).

Source: `src/app/api/inbound/[systemCode]/[endpoint]/route.ts`, `core/auth/with-partner-auth.ts`, `core/auth/service-token.ts`, `core/pure/endpoint-name.ts`, `core/pure/connector-input.ts`, `core/gateway/rate-limit.ts`, `core/gateway/log-call.ts`, `core/pure/redact.ts`, `core/connectors/resolver.ts`, `core/jobs/lifecycle.ts`, `core/pure/prefer.ts`, `core/jobs/payload-vault.ts`.

Properties worth calling out:

- **systemCode confinement.** The auth guard resolves the partner system named in the URL first, then identifies the presented credential by its lookup index across all systems. A valid credential for system A can never authenticate a call to system B's routes, even if both are ACTIVE: it is refused with a generic `403`, and the owning system is recorded only in the activity row. Revoked and expired credentials get `401`, and so does a credential whose owning system is deleted. A deleted system is an unknown system at the gateway. See `docs/security.md`.
- **Rate limiting is Postgres-backed and fails open.** Each check is one atomic `INSERT ... ON CONFLICT (key, window_start) DO UPDATE SET count = count + 1 RETURNING count` against `rate_limit_bucket` (one-minute fixed windows). Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; a rejected call gets 429 with `Retry-After`. If the database is unreachable the limiter logs a warning and lets the request through, because a limiter outage must not become a gateway outage. The client IP is taken from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, and must parse as an IP address (anything else shares an `unknown` bucket); on a host that does not overwrite `x-forwarded-for` that last fallback is client-controlled, so put a trusted proxy in front. The per-IP limit (`RATE_LIMIT_IP_RPM`, default 250 per minute) is checked before authentication, which throttles credential guessing; the per-system limit (`external_system.rate_limit_rpm`) is checked after, so strangers cannot burn a partner's quota. A system can also declare per-endpoint limits (bucket `sys:<code>:<endpoint>`); the response headers report the tightest bucket consumed. The "Rate limits" page shows the current window's gateway buckets and the last hour of 429s, with client IPs truncated to /24 and /48.
- **Idempotent retries.** A repeated `Idempotency-Key` from the same system returns the original job's outcome with `Idempotent-Replayed: true` instead of creating a second job or failing: 200 with the output for a `SUCCESS`, 500 for a `FAILED` or `DLQ` job, and, for an async job still `PENDING`, `PROCESSING` or `RETRY`, a 202 `{jobId, status}` that doubles as the status poll. The same key with a different payload is a 422; a duplicate that arrives while an inline original is still running is a 409. The key is trimmed (an empty key counts as absent), at most 200 printable ASCII characters (else 400), and stored with an `in:` prefix so a partner key can never collide with a scheduler key. Only true replays carry `Idempotent-Replayed: true`; the 422 does not.
- **Every call is logged.** `activity_log` gets one row per call the platform receives or makes, written by `logCall` (`core/gateway/log-call.ts`). `direction`, `isTest`, `request` and `response` are required by the TypeScript type, so a call site cannot forget one (use `{}` or `{ reason }` when there is no body). Inbound logging is total: every exit of the route is a `finish(...)` call and the handler body runs inside a try whose catch also finishes (a 500), so auth failures, rate limits, validation errors and unexpected throws each write a row. A throw in `await params` before the try is a framework fault and sits outside the audit's Rule D scope. Outbound rows come from the three network wrappers: `safeFetch` (`core/net/ssrf.ts`), `probeSftp` (`core/net/sftp-probe.ts`) and `qstashFetch` (`core/jobs/qstash.ts`). Each takes a required `OutboundLog` context (`systemCode`, `jobId`, `isTest`, `requestId`); health checks, SFTP tests and QStash calls are written with `isTest: true`, and a call that fails before any HTTP response logs response code 0 and a fixed error code. `logCall` never throws: a failed write is a warning, so the call being logged is never broken by its own bookkeeping. `scripts/audit-log-calls.ts` (a TypeScript AST audit, `npm run audit:log-calls`, run in CI) enforces this with three rules. Rule A: a network call sits in a function that calls `logCall`, or in a private helper whose every in-module caller does. Rule C: `ssh2`, `undici`, `tls`, `http` and `https`, and the connecting exports of `net`, are imported only in the wrapper files. Rule D: each exported non-`OPTIONS` inbound handler is declarations, then a try whose catch returns `finish(...)`, and every return is `finish(...)`, which calls `logCall`. Redaction runs before storage and clips run after it, so a stored preview never holds a secret. Key rules mask the value of a secret-looking key at any depth; value patterns replace secret-shaped spans inside any string (Bearer and the other Authorization schemes, Basic, JWT, PEM private keys, `keyword=value` pairs, URL userinfo, Luhn-valid card numbers); strings clip at 2,000 characters and a payload at 16,000 (a longer one is stored as `{truncated, originalChars, preview}`); the stored endpoint is masked and capped at 256 characters. Credentials and `Authorization` headers are never stored. The exact rules are in `docs/security.md`. Errors on the inbound catch, `logCall` and prune paths go through `safeErrorLog` (the error name, a SQLSTATE and a short message cut before any bound parameters, never the error object). The inbound row is written after the response is sent (`after()` on the Vercel runtime, inline under test) so logging adds no latency. Retention is controlled by the `app_setting` row `retention` (`{enabled, activityDays}`): `enabled: false` is the kill switch for every automatic delete, and with no row `ACTIVITY_RETENTION_DAYS` (default 30) applies. Two triggers prune in bounded batches: the tick (5,000 rows x 10 batches per run) and a throttled `after()` trigger on inbound traffic (`maybeRunRetention`, 1,000 x 5, at most once per ten minutes across instances through a compare-and-set on the `retention_last_run` row; a stored stamp in the future counts as stale). A row is never deleted while its job has an open (`PENDING_REVIEW`) DLQ item. This table is separate from `admin_audit_log`, which records admin actions. A replay of an inbound call (same `Idempotency-Key`) returns the stored `job.result`, which was redacted and clipped when it was written, so a replay returns the redacted, clipped output and not the original body (Deviation 2, by design).
- **Fail-fast on an unknown handler.** If `integration_type.handler_key` does not exist in the generated manifest, `resolveHandler()` throws `UnresolvedHandlerError` immediately rather than queuing a job that will never succeed.
- **Service token by policy.** Each integration type has an `auth_policy`: `partner` (default) or `partner+service`. Under `partner+service` the request must also carry an `X-Service-Token` JWT (HS256, `aud` `interchange-inbound`, lifetime at most 900 s). It is checked after the partner credential, answers 503 when `SERVICE_JWT_SECRET` is missing (fail closed), and is never accepted as a partner credential. `PartnerIdentity` carries the policy, handler key, direction and CORS origins from one join with `integration_type`, so the route no longer selects the type itself.
- **Endpoint allowlist.** The `[endpoint]` segment is normalised (NFC, lower-case, `^[a-z0-9._-]{1,64}$`). Declaring any endpoint limit turns the list into the system's endpoint allowlist: other names are 404 "unknown endpoint", after authentication.
- **Strict payloads.** The top level of every inbound payload and every manual `jobs.run` payload is parsed strict. Unknown keys give a 422 that names the keys and never echoes a value. The drain parses stored payloads leniently, so older rows are not dead-lettered by the new rule.
- **Soft delete and suspend.** A deleted system is an unknown system. Suspending or deleting a system stops its schedules with no catch-up fire, holds a suspended system's queued jobs, and fails a deleted system's queued jobs without retry (see "Outbound and scheduled job lifecycle"). Deleted system and integration type codes stay reserved, so a code is never reused for a different partner.

### Async inbound (202 Accepted)

A partner that cannot wait for `execute()` can send `Prefer: respond-async` (RFC 7240) with an `Idempotency-Key`. The gateway then accepts the call, answers 202 and runs the job in the background. Every one of these must hold; otherwise the preference is ignored (RFC 7240 lets a server do that) and the call runs inline exactly as before, with no `Preference-Applied` header:

- The header names `respond-async`. `prefersRespondAsync` (`core/pure/prefer.ts`) reads a comma list, matches the name case-insensitively, ignores parameters and values, and ignores a header over 1,024 characters whole.
- A valid `Idempotency-Key` is present, so a lost 202 can be recovered by replaying the call. An invalid key is still a 400.
- The connector's `handler.meta.ts` declares `asyncInbound: true`. The flag is the connector author's promise that `execute()` is idempotent, because a reclaimed async job runs again: delivery is at-least-once. Both inbound templates declare it, and the catalog build refuses it on an outbound connector.
- The validated payload is at most 64,000 JSON characters (`ASYNC_PAYLOAD_MAX_CHARS`), because a vault row is bounded, not a blob store.
- Sealing the payload succeeds. A seal failure (for example no master key) is logged and the call runs inline.

When they hold, one transaction creates the job `PENDING` with `accepted_async = true` and inserts its sealed payload into `job_payload_vault`, so the two commit together or not at all. The answer is 202 `{jobId, status: "PENDING"}` with `Preference-Applied: respond-async`. A replay of the same key while the job is still `PENDING`, `PROCESSING` or `RETRY` answers 202 `{jobId, status}` with `Idempotent-Replayed: true` (and `Preference-Applied` only if that call asked for it), so the replay is also the status poll; a `SUCCESS` replays as 200 with the output, a `FAILED` or `DLQ` job as 500, and a different body is a 422.

After the response, the deferred block claims exactly that job by id (`drainJobById`) and runs it, instead of the throttled `maybeDrain`. The claim uses the predicate and `FOR UPDATE SKIP LOCKED` of a batch claim, so a job a concurrent drain already holds, or one whose system is suspended, is simply not taken. Because the accepting call runs its own job, a deploy with no queue tick still runs it. A retryable failure goes to `RETRY` with the normal backoff and is drained by the after-response drain and the tick, like an outbound job.

- **The executable copy.** `job.payload` of every inbound job stays the redacted copy and is never executed. The original is stored in `job_payload_vault` (`core/jobs/payload-vault.ts`): AES-256-GCM under a purpose key that `encryptForPurpose` (`core/crypto/vault.ts`) derives from `CREDENTIAL_MASTER_KEY` with HKDF-SHA256 (purpose `job-payload`), with the job id as additional authenticated data and a `key_version` like a credential blob. The purpose key means a credential blob cannot be opened as a payload or the reverse, and the job id means a blob copied to another job fails its tag check. No new environment variable is added.
- **Short life.** A copy expires 24 hours after it is stored. It is deleted when the job succeeds, when its dead letter is discarded, when the job dead-letters because its copy is missing or expired (any other dead letter keeps the copy until its TTL, so a DLQ replay can still run it), when it expires (the tick's `purgePayloadVault` stage and the throttled retention run purge in bounded batches, independent of the retention kill switch, because expired ciphertext is not history) and by foreign-key cascade when the job is deleted.
- **When the copy is gone.** A missing, expired or undecryptable copy (and an inbound row that was never accepted async) fails the job non-retryable with `PAYLOAD_UNAVAILABLE` at attempt 1, and `execute()` is never called. An undecryptable copy is kept until its TTL rather than deleted, because it may only lack a key an operator can restore (`CREDENTIAL_MASTER_KEY_PREVIOUS`); after that, a DLQ replay runs it. The drain parses the sealed body once with the connector's schema, as the inline path does, so a schema with a transform or default behaves the same in both modes.
- **What the connector gets.** The drain gives an inbound connector `{ flags }` only, never outbound credentials, as the inline path does, and masks the payload's secret values (and the flag values) out of the stored result, error and log lines.
- **Reclaim.** A stuck async job retries like an outbound one (see "Stuck jobs are reclaimed" below); a stuck inline job still goes to `FAILED`.
- **Console.** The job detail page shows an "Execution" row for inbound jobs (Inline, or Accepted async (202)), and the Catalog shows an "Accepts async (202)" chip on a connector that opted in. The demo has no inbound gateway in the browser, so the 202 path is server-only there; a seeded job shows the async state.

## Edge proxy

Before any route handler runs, the Next proxy (`src/proxy.ts`, Node runtime; it sits in `src/` because the app directory is `src/app`) handles every request except static assets, icons and prefetches. It resolves the request id (step 0 above), generates the Content-Security-Policy nonce and sets the CSP, adds HSTS over https, and applies the env-list CORS rule to `/api/*` except inbound (denied unless `CORS_ALLOWED_ORIGINS` lists the origin, and never for `/api/admin/*`). `/api/inbound/*` passes through untouched: the inbound route has its own `OPTIONS` handler and decides per system, using the system's `cors_origins` list with `CORS_ALLOWED_ORIGINS` as a ceiling when it is set. Because a nonce needs a request, every page renders dynamically. The pure header builders are in `core/http/security-headers.ts`. See `docs/security.md` for the exact policy.

## Outbound and scheduled job lifecycle

Outbound work (calling out to a partner, or running a recurring task) does not run inline. It is written to the `job` table and picked up by a drain.

```
                       +-------------------+
   createJob(...)  --> |  job              |
   (status=PENDING)    |  status=PENDING   |
                       +-------------------+
                                |
                                | /api/jobs/tick or /api/jobs/process
                                | (POST), called by an armed QStash
                                | schedule; the request must carry a
                                | valid Upstash signature
                                v
                       +-----------------------------------+
                       | drainBatch()                       |
                       | core/jobs/drain.ts                 |
                       |                                     |
                       | SELECT ... FROM job                 |
                       | WHERE status IN ('PENDING','RETRY')  |
                       |   AND (nextAttemptAt IS NULL         |
                       |        OR nextAttemptAt <= now())    |
                       | ORDER BY coalesce(nextAttemptAt,     |
                       |          createdAt), id              |
                       | LIMIT 20                             |
                       | FOR UPDATE SKIP LOCKED                |
                       |   (claims a batch atomically; two     |
                       |    concurrent drains never race       |
                       |    on the same row)                   |
                       +-----------------------------------+
                                |
                                v
                       status -> PROCESSING
                                |
                for each claimed job:
                                |
                resolveSystemConfig(systemCode)
                (system row, then type default;
                 decrypts via a version-checked
                 cache, see docs/security.md)
                                |
                connector.execute(ctx)
                                |
              +-----------------+------------------+
              |                 |                   |
         SUCCESS            RETRY/thrown         non-retryable
              |             (transient)           FAILED
              v                 |                   |
      status=SUCCESS            v                   v
      result stored,    status=RETRY,        status=DLQ
      succeeded step    nextAttemptAt =      + job_dlq row
      (one transaction) now + delay(n)       (reason, lastError)
                         attempts < maxAttempts   attempts >= maxAttempts
                                |                   OR error.retryable=false
                                |
                        (next drain picks it
                         up again once due)

   delay(n) = max(backoff(n), Retry-After)
   Every failed attempt also stores the partner's masked answer in
   job.upstream_error (4xx and 5xx only).
```

Source: `core/jobs/drain.ts`, `core/jobs/lifecycle.ts`, `core/jobs/scheduler.ts`, `core/jobs/reclaim.ts`, `core/jobs/verify-qstash.ts`, `core/pure/retry-policy.ts`, `db/schema/operational.ts`.

Key mechanics:

- **The queue is not a separate table.** It is `job` rows with `status IN ('PENDING', 'RETRY')` whose `next_attempt_at` is due. This is a deliberate simplification: one source of truth for both "what ran" and "what's next".
- **Claiming is race-safe without Redis.** `FOR UPDATE SKIP LOCKED` inside a transaction lets multiple concurrent invocations of the drain (which can happen on serverless, since nothing guarantees single-flight) each claim a disjoint batch.
- **Claim order is oldest due first.** The claim sorts by `coalesce(next_attempt_at, created_at), id`, so a retry counts from when it became due and a stream of new PENDING jobs cannot starve an overdue RETRY. The `id` is the tiebreak.
- **Backoff is exponential, capped and jittered** (`core/pure/retry-policy.ts`). The full delay is `min(15 min, 5 s * 2^(n - 1))` for attempt `n`; equal jitter then keeps half of it fixed and randomises the other half (`full / 2 + random * full / 2`), so a burst of failures does not retry in lockstep. Attempt 1 waits 2.5 to 5 s, and the cap is first reached at attempt 9. The wait is the longer of that backoff and the partner's `Retry-After`.
- **Retry classification** (`classifyHttpStatus` and `outboundRequest` in `core/net/outbound-client.ts`; the table is in `core/pure/retry-policy.ts`):

  | Failure | Code | Retried |
  |---|---|---|
  | Network error (socket, reset) | `NETWORK` | yes |
  | Timeout, connect or TLS failure, temporary DNS | `TIMEOUT`, `CONNECT`, `TLS`, `DNS_TEMPORARY` | yes |
  | 5xx | `HTTP_5XX` | yes |
  | 429 | `HTTP_429` | yes, no sooner than `Retry-After` |
  | Other 4xx (408 included) | `HTTP_4XX` | no, dead-lettered at once |
  | 3xx | `REDIRECT` | no |
  | Blocked or invalid address, permanent DNS failure | `BLOCKED`, `INVALID_URL`, `DNS` | no |
  | A connector throws `NonRetryableError` | its own code | no |
  | A connector throws any other error | `HANDLER_THREW` | yes |
  | No connector for the handler key | `UNKNOWN_HANDLER` | no, fail-fast at attempt 1 |
  | The endpoint's auth label has no active credential | `CREDENTIAL_MISSING` | no, fail-fast at attempt 1 |
  | A stored credential does not decrypt, or cannot be a header value | `CREDENTIAL_INVALID` | no, fail-fast at attempt 1 |
  | The OAuth token endpoint is unreachable, times out, answers 5xx or 429, or another caller holds the refresh lease | `OAUTH_TOKEN_UNAVAILABLE` | yes, no sooner than its `Retry-After` |
  | The token endpoint answers 4xx or a redirect, the token URL is blocked or not allowed, or a 2xx body is not a usable token | `OAUTH_TOKEN_REJECTED` | no |
  | An async inbound job's executable payload is missing, expired or cannot be decrypted | `PAYLOAD_UNAVAILABLE` | no, fail-fast at attempt 1 |

- **Retry-After.** A 429 or 5xx answer's `Retry-After` header (delta-seconds or an HTTP-date) is parsed to a delay and clamped to 0 to 1 hour, so one partner cannot park a job for days. The next attempt is due at the later of the backoff and that delay.
- **Fail-fast.** A failure that no retry can fix goes to the DLQ at the first attempt: an unknown handler key (`UNKNOWN_HANDLER`), a payload that fails the connector's schema (`INVALID_PAYLOAD`), a deleted system (`SYSTEM_DELETED`), a missing or unusable credential (`CREDENTIAL_MISSING`, `CREDENTIAL_INVALID`), an async inbound payload that is no longer available (`PAYLOAD_UNAVAILABLE`) and a `NonRetryableError`. The job lists one actionable DLQ row instead of a retry cycle.
- **Retries config.** A job's `max_attempts` is the system's `endpoint_config.retries + 1` (`maxAttemptsFor`, `core/jobs/lifecycle.ts`), read when the job is created; a job already queued keeps the value it was created with. A system with no `endpoint_config` row gets the column default of 5 attempts. Jobs queued before P4a keep their stored `max_attempts` (5).
- **After-response drain.** A tick is not the only way a due job runs. `maybeDrain` (`core/jobs/drain.ts`) runs after the response of `jobs.list`, `jobs.get`, `dlq.list`, `dashboard.summary` and the inbound route (there only after an authenticated call answered under 400, and after the alerts step), through `deferWork`. An inbound call that answered 202 does not open the window: it runs exactly its accepted job instead (`drainJobById`, one job claimed by id). It is throttled to one window per 60 s (`DRAIN_THROTTLE_MS`) by a compare-and-set on the `app_setting` row `drain_last_run`, so concurrent reads and separate instances drain once per window. The winner first reclaims stuck jobs, then drains at most 3 jobs (`DRAIN_AFTER_LIMIT`), claiming one job per transaction and stopping once 15 s have passed (`DRAIN_BUDGET_MS`); the first claim is always made, and a stop never strands a claimed job. Worst case is the 15 s budget plus one 30 s connector timeout, inside the 60 s `maxDuration` the routes declare; an accepted 202 job runs alone, so its bound is one connector timeout. A `process_log` row (kind `drain`, trigger `after`) is written only when work was done (a job reclaimed or claimed), so an idle window leaves just the stamp. It never throws, and a switch (`core/jobs/drain-trigger.ts`, a module flag, not an env key) turns the call sites off for tests. The signed tick and `/api/jobs/process` are unchanged and drain a batch of up to 20. The queue dashboard read (`dashboard.queue`, P4d) is deliberately not on that list: it is a passive read and never opens a window.
- **Terminal writes (D9).** A job that already ran is never failed because the database blinked. The SUCCESS update and the `succeeded` step are one transaction, retried up to 3 times (100, 200 and 400 ms apart, `retryTerminalWrite`). If it still fails, or the failure write itself fails, the error is logged and the job stays `PROCESSING` for reclaim; it is never passed to the failure path a second time. The failure paths already write their step inside the same transaction as the status change.
- **Migration 0008.** `db/migrations/0008_p4a_engine.sql` adds the nullable column `job.upstream_error` (text, with a CHECK of at most 8000 characters) and an index on `notification (alert_id)`. It is additive, so the previous build stays compatible and rolling back is a redeploy.
- **Credential resolution and the cache (P4b1).** `resolveSystemConfig` (`core/config/resolve-config.ts`) builds a job's credentials per label from the system's own active row, then the active default of its integration type (`type_credential`), then nothing. The decrypted rows are cached per instance (`core/config/credential-cache.ts`: TTL 300 s, at most 500 systems), and an entry is served only while `external_system.credential_version` and `integration_type.credential_version` both match the values read at the start of the job. Every credential write, and every type delete and restore, adds 1 to the counter inside its own transaction, so a revoke or rotation takes effect on the next job on any instance; a job already past its version read finishes with the old value. The endpoint row is read fresh on every job. The full rules are in `docs/security.md`.
- **Outbound auth (P4b1).** `endpoint_config.auth_label` and `auth_header` bind a credential to an endpoint. `outboundRequest` adds the header (an `api_key` in `auth_header`, default `X-API-Key`; `oauth2`, `basic` and a fetched `oauth2_client` token as `Authorization`) only when the request origin equals the base URL origin and is in the credential's allowed origins (1 to 10 exact origins); otherwise it sends no auth and logs one fixed warning. A missing label gives `CREDENTIAL_MISSING`, an unusable or undecryptable credential `CREDENTIAL_INVALID`; both are non-retryable and dead-letter at attempt 1. The `flag` credential type is a secret switch read as `ctx.config.flags` (`flagEnabled`), never auth and with no origin pin; inbound connectors receive flags only.
- **Migration 0009.** `db/migrations/0009_p4b_credentials.sql` adds the `type_credential` table, `credential_version` (integer, default 0) on `external_system` and `integration_type`, `endpoint_config.auth_label` and `auth_header` (nullable), `system_credential.allowed_origins` (`text[]`, default empty) and the partial index `job_claim_due_idx` on the claim order of `job`. It is additive and guarded (`IF NOT EXISTS`, and a `DO` block for the foreign key). The rollback hazard is in `docs/deployment.md`.
- **OAuth 2.0 client credentials (P4b2).** An `oauth2_client` credential holds a token URL, a client id and secret, an optional scope and audience and an auth method (`basic` or `post`). When `outboundRequest` resolves such an auth, it first applies the same origin pin as for any credential, and only then asks the token client (`core/net/oauth-client.ts`) for a token: the stored one, or a fresh one fetched with a form `POST` through the same guarded `outboundRequest`. Tokens are cached encrypted in `oauth_token_cache` (one row per credential), reused until a margin before expiry (60 s, or half the lifetime under 120 s), and refreshed under a 30 s lease so one caller fetches while the others re-read the row 3 times, 500 ms apart. `expires_in` is clamped to 30 s to 24 h (300 s when absent). A rotate or revoke deletes the cached row in its own transaction, and a credential version change makes a stored token a miss. A resource `401` for a stored token drops it, fetches one new token and retries once. Token failures are `OAUTH_TOKEN_UNAVAILABLE` (retryable) and `OAUTH_TOKEN_REJECTED` (final), listed in the retry table above. The secret's rules, the origin checks and the leak controls are in `docs/security.md`.
- **Migration 0012.** `db/migrations/0012_p4b2_oauth.sql` creates `oauth_token_cache` with a foreign key to either `system_credential` or `type_credential` (cascade on delete, and a CHECK that exactly one is set), one unique index per owner column and the lease columns. It is additive and idempotent. See `docs/deployment.md`.
- **Migration 0013.** `db/migrations/0013_async_inbound.sql` adds `job.accepted_async` (boolean, default false) and creates `job_payload_vault`: one row per accepted async job, keyed by the job id with a cascade foreign key to `job`, the encrypted blob (`enc_iv`, `enc_ciphertext`, `enc_auth_tag`), `key_version`, `expires_at` and an index on `expires_at` for the purge. It is additive and idempotent. It must be applied before this version is deployed, and the rollback hazard is in `docs/deployment.md`.
- **The DLQ (`job_dlq`) is a queryable ledger of terminal failures**, each pointing back at the original `job.id`, with the reason (`retries exhausted` or `non-retryable failure`) and the last error payload. `replayDlq(jobId)` resets a DLQ'd job to `PENDING` with `attempts` cleared, so it re-enters the drain on the next pass. The console calls it through `POST /api/admin/dlq/:jobId/replay`, which requires `jobs:retry` (a job that is no longer replayable returns `CONFLICT`). A manual run is `POST /api/admin/jobs`, which requires `jobs:run`.
- **Schedules (`job_schedule`) fire by cron, exactly once.** Each enabled schedule has a `next_run_at`. A tick claims a due schedule with a compare-and-swap (`UPDATE ... WHERE id = ? AND next_run_at = ? RETURNING`), so two overlapping ticks cannot both fire it, then enqueues one PENDING outbound job and computes the next `next_run_at` from the cron expression (croner, UTC). The claim and the job insert share one transaction, so a failed insert rolls the claim back and the next tick retries the same slot. A schedule registers nothing with QStash: its cadence is its own cron, evaluated by the tick, so it can run no more often than the tick. `/api/jobs/dispatch` does only this enqueue step, and only for due schedules. Both of the tick's selects skip soft-deleted schedules and take at most 50 rows in a stable order (due ones by `next_run_at`). A schedule whose cron expression throws or can never fire again is disarmed by the tick (logged as `SCHEDULE_INVALID_CRON`), so one bad row cannot hold its place at the head of the queue and starve the rest.
- **`/api/jobs/tick` is the single background driver.** One signed QStash call per cadence runs, in order: stuck-job reclaim, due-schedule dispatch, a bounded `drainBatch()`, and a prune of expired rate-limit buckets and old `activity_log` rows (the activity prune reads the `retention` setting: a disabled kill switch makes it delete nothing, and rows still referenced by an open DLQ item are kept; with the tick dormant, the throttled after-response trigger prunes only after inbound traffic), then a `purgePayloadVault` stage (expired async payload copies, deleted whatever the retention setting says), then a `pruneJobs` stage that applies the same switch and guard to jobs, their step logs and process runs older than `jobDays`, then a `pruneAlerts` stage (resolved alerts and notifications older than `jobDays`, behind the same kill switch) and an `alerts` stage (the throttled alert evaluation, `maybeRunAlerts("qstash")`; see "Alerts, notifications and system health (P3c)"). The whole tick is recorded as one `process_log` row, and a second delivery of the same QStash message (same `Upstash-Message-Id`) answers 200 `{ok:true, duplicate:true}` and runs nothing. Each stage has its own try/catch, so one failure cannot skip the others; if any stage failed the response is 500 with every stage's result. There is exactly one QStash schedule, cadence `QSTASH_TICK_CRON` (default `*/5 * * * *`), created by the explicit "Arm tick" action on the Schedules page (`armTick()`, id stored in `app_setting`) and removed by "Disarm tick". It is dormant until armed.
- **Stuck jobs are reclaimed.** A job left `PROCESSING` with `locked_at` older than `JOB_STUCK_MINUTES` (default 10) lost its worker. An outbound job returns to `RETRY` with the same capped, jittered backoff while attempts remain (there is no `Retry-After` to honour), otherwise goes to the DLQ through the same path as a normal failure (`applyFailure`, code `RECLAIMED`); a reclaim leaves `job.upstream_error` as it was, because the worker that died may have stored the partner's last answer; this is at-least-once delivery, so outbound handlers and the partners they call must be idempotent. An inline inbound job is never re-executed: it goes to `FAILED`, and the partner's retry with the same `Idempotency-Key` receives that recorded failure. An async inbound job (`accepted_async`) is different: the partner already holds its 202 and is not waiting, so failing it would lose the accepted call. It retries like an outbound job (the same backoff, code `RECLAIMED`) and runs again from its vault copy, which is why `asyncInbound` is a promise that `execute()` is idempotent; past the 24 h vault TTL the re-run dead-letters as `PAYLOAD_UNAVAILABLE`.
- **Suspended and deleted systems do no background work.** The scheduler joins `external_system` on `system_code` and fires a schedule only for a live, `ACTIVE` system. Any other due schedule has its `next_run_at` advanced to the next cron slot with nothing enqueued (counted in `skipped`), so activating or restoring a system later never produces a catch-up fire. The drain's claim query excludes jobs of live, non-`ACTIVE` systems: a `SUSPENDED` or `SETUP` system's jobs are held (still pending, attempts unchanged, no batch slot taken from other systems) until the system is `ACTIVE`. A deleted system's job fails non-retryable ("system deleted") before the connector runs and lands in the DLQ. Restore leaves a system `SUSPENDED`, so held work waits for an explicit activation. This changed in P2b: before it, a suspended system's schedules fired and its queued jobs ran. A schedule whose `system_code` matches no system row at all is not gated.
- **QStash calls are signature-verified.** `/api/jobs/tick`, `/dispatch` and `/process` verify the `Upstash-Signature` header against the raw body with `QSTASH_CURRENT_SIGNING_KEY` / `QSTASH_NEXT_SIGNING_KEY`. The expected url is built from `APP_URL` (else `NEXTAUTH_URL`) plus the request path and passed to the verifier, so a signature minted for one endpoint cannot be replayed against another. With the keys unset they fail closed (503). The only bypass is an explicit `QSTASH_DEV_BYPASS=1`, honoured outside production only.
- **DLQ review state.** `job_dlq.status` is `PENDING_REVIEW`, `RETRIED` or `DISCARDED`; the dashboard count and the DLQ list share one helper (`core/jobs/dlq.ts`) and filter `PENDING_REVIEW`; the list pages by `offset` in pages of 50 ("Rows A to B of N" with Previous and Next), like the jobs list. `dlq.list` takes a `status` filter (`PENDING_REVIEW` by default, `RETRIED`, `DISCARDED` or `all`). `replayDlq()` marks the entry `RETRIED`; `dlq.discard` (`POST /dlq/:jobId/discard`, `jobs:discard`) marks it `DISCARDED` and leaves the job in `DLQ`, now eligible for retention. Both take a note, stored redacted in `job_dlq.notes` (required for a discard, optional for a replay); the discard audit diff is `{hasNotes:true}`, never the note. Replay and discard each run in one transaction that first moves the entry out of `PENDING_REVIEW`: no row to move answers `CONFLICT` (409), so a discarded item cannot be revived and a concurrent replay and discard have exactly one winner. The Jobs page has Jobs, Dead letters and Runs views; Retry renders only with `jobs:retry` and Discard only with `jobs:discard`.

## The connector plugin model

A connector is a self-contained folder under `src/connectors/<name>/` with exactly two files that matter to the platform:

- `index.ts` - default-exports an object implementing the `Connector` interface (`core/connectors/types.ts`): a `kind` (`"inbound"` or `"outbound"`), a zod `inputSchema`, and an `execute(ctx)` function.
- `handler.meta.ts` - exports a `meta: ConnectorMeta` object with a `handlerKey`, `direction`, `version`, and optional `description`; an inbound connector may also set `asyncInbound: true` to accept `Prefer: respond-async` (see `docs/connectors.md`).

Current connectors: `rest-inbound`, `rest-outbound`, `webhook-inbound`, `file-drop-sftp`. The file a `file-drop-sftp` job sends travels inside its payload (`{filename, contentBase64}`) through the existing `jobs.run`, with shared size and name rules; see `docs/connectors.md` and `docs/security.md`.

### Why codegen instead of runtime dynamic import

Vercel's serverless bundler (Node File Trace) needs to see every file a route might load at build time. A runtime `import(\`./connectors/${key}\`)` with a dynamic key is invisible to that analysis and would fail in production even though it works locally. `scripts/generate-connector-manifest.ts` runs at build time (wired as `prebuild` and `predev` in `package.json`) and:

1. Globs `src/connectors/*/` for folders containing both `handler.meta.ts` and `index.ts`.
2. Extracts each folder's `handlerKey` with a lightweight regex (the manifest itself needs no TypeScript evaluation; the catalog step in item 5 does).
3. Fails the build if two folders declare the same `handlerKey`.
4. Emits `src/connectors/generated/manifest.ts`: a file with genuine static `import` statements, one per connector, assembled into a `Record<string, Connector>`.
5. Emits `src/connectors/generated/catalog.ts` through `scripts/connector-catalog.ts`. This step does evaluate TypeScript: it imports each folder's `handler.meta.ts` and `index.ts` through `tsx`, turns the connector's zod `inputSchema` (made strict) into a JSON Schema and pairs it with the `sample` from the meta. The build fails if a sample does not pass the connector's own schema under the gateway's strict rule (`parseConnectorInput`), or if the meta `direction` disagrees with the connector `kind`.

The catalog is build time only. `zod-to-json-schema` is a devDependency imported only by `scripts/`, and an ESLint `no-restricted-imports` rule bans it from `src/` and `core/`, so it can never reach a bundle. App code reads the committed `catalog.ts` as plain data: the real server and the demo both import it, so `connectors.catalog` returns identical data on both. A drift test fails if `catalog.ts` differs from a fresh `buildCatalog()`, and another checks that each sample validates against its emitted schema.

At runtime, `core/connectors/resolver.ts` does nothing more than a synchronous object lookup into that generated map:

```ts
export function resolveHandler(handlerKey: string): Connector {
  const connector = connectorManifest[handlerKey];
  if (!connector) throw new UnresolvedHandlerError(handlerKey);
  return connector;
}
```

This is the mechanism behind the two-lever model:

- **Lever A (developer): add a connector type.** Create a new folder, implement `index.ts` and `handler.meta.ts`, run `npm run generate:connectors`, redeploy. One-time work per connector *type*.
- **Lever B (operator): register a connector instance.** Once a connector type exists, an operator registers unlimited partner *instances* of it through the admin UI (an `external_system` row pointing at an `integration_type` whose `handler_key` matches), with no code change and no redeploy.

See `docs/connectors.md` for the full walkthrough and a minimal connector example.

## Outbound network guard (`core/net`)

Every server-side HTTP request to a URL an administrator typed goes through `safeFetch` in `core/net/ssrf.ts`; outbound connectors must not call `fetch` themselves. The one non-HTTP path is the SFTP connection probe (`core/net/sftp-probe.ts`): it calls `resolvePublicHost` in the same file (the same classifier, one resolution, every answer must be public), dials the checked address itself and hands the open socket to `ssh2`, which never sees the host name. The guard has three layers: a pure address classifier shared with the demo (`core/pure/address-policy.ts`), a preflight that resolves the host and refuses it if any answer is blocked (https only in production), and a connect-time lookup on the HTTP client that judges the address actually dialled, so DNS rebinding fails. Redirects are never followed, every call has a timeout, and failures surface as one of seven fixed codes (`BLOCKED`, `INVALID_URL`, `DNS`, `DNS_TEMPORARY`, `TIMEOUT`, `CONNECT`, `TLS`) that never carry upstream text. `DNS_TEMPORARY` is a temporary resolver failure and is retried with backoff. Its consumers are the `rest-outbound` connector (which wires the system's `timeout_ms`) and the system health check. `safeFetch` and `probeSftp` take a required `log` context and write one outbound `activity_log` row per call, success or failure; the AST audit (Rules A and C) fails the build if a network call bypasses them. The ranges, the code-to-retry mapping and the rationale are in `docs/security.md`; the connector rules are in `docs/connectors.md`.

## The two-database model

Interchange's own schema (31 core tables plus one opt-in table) is the platform's single source of truth for partner registry, credentials, jobs, and audit history. This is always present.

Separately, a connector may need to read from a partner's or a project's own upstream system: a legacy database, an internal REST API, whatever the integration needs. Interchange models this as `upstream_data_source` (`db/schema/upstream.ts`), explicitly marked as **opt-in and not part of the core tables**:

- The core platform provides only the connection and credential plumbing (`connection_config`, encrypted credential fields, a `kind` enum of `postgres | mysql | rest | odbc_http_bridge`, and a `read_only` flag defaulting to `true`).
- Connectors that need upstream reads bring their own named queries, living in their own connector folder, never in core. Core does not know or care what SQL or API calls a connector issues against its upstream source.
- **The platform runs fully with zero upstream sources configured.** A deployment with no `upstream_data_source` rows is a complete, working Interchange instance; upstream integration is additive, not load-bearing.

This keeps the platform vendor-neutral: nothing in `core/` or the base schema assumes any particular partner's database shape.

## Dormant posture

A fresh deploy is fully built but inert by design:

- Every `external_system` row defaults to `status = 'SETUP'` (see `db/schema/config-auth.ts`). The inbound auth guard rejects all traffic for a system that is not `ACTIVE`, regardless of whether valid credentials exist for it.
- `job_schedule` rows default to `enabled = false` and carry no `qstash_schedule_id` until explicitly armed.
- `core/jobs/qstash.ts`'s `armTick()` is a documented no-op when `QSTASH_TOKEN` is unset: it returns `{ armed: false, reason: "QSTASH_TOKEN not set (dormant)" }` rather than throwing, so the platform is fully functional in a dormant deploy with no external scheduler configured.
- The job endpoints reject unsigned calls and answer 503 when the QStash signing keys are not configured, so a dormant deploy has no open trigger.
- Dormant means no timer tick, not no drain. With no QStash schedule armed, the throttled after-response drain (60 s window, 3 jobs, 15 s claim budget; see "After-response drain") still runs on console reads and authenticated inbound calls answered under 400, so a due job is processed when someone uses the console or a partner calls in. A job accepted with 202 needs no tick either: the call that accepted it runs it. A deploy with an empty queue does nothing: the drain writes only its `drain_last_run` stamp.
- The Queue page shows this posture live: whether the timer tick is armed, whether its signing keys are present, when the last drain window ran, and how many jobs are Queued, Due now, Held, Waiting or Stuck. See "Queue and dashboards (P4d)".
- No recurring schedule fires, and nothing recurring is billed, until a project explicitly flips a system to `ACTIVE` and/or arms a schedule.

See `docs/deployment.md` for the exact activation steps, and `docs/security.md` for why this is also a security property, not only a cost-control one.

## Admin API and demo

The admin console is one client UI (`src/features/*`, routed by `src/features/shell/ConsoleShell.tsx` from the table in `src/features/routes.ts`) over a single typed interface, `AdminApi` (`src/admin-api/contract.ts`). The UI never knows which implementation it talks to; that is what lets the live console and the public demo share every screen.

- **`HttpAdminApi`** (`src/admin-api/http/client.ts`) calls `/api/admin/*`. On the server, one catch-all route (`src/app/api/admin/[...path]/route.ts`) matches method and path against the endpoint registry (`src/admin-api/endpoints/`). Each endpoint declares its id, method, path, permission, zod input, output and audit action, so there are no per-endpoint route files. An unknown path is 404 and a known path with the wrong method is 405.
- **`DemoAdminApi`** (`src/admin-api/demo/`) runs entirely in the browser over an in-memory engine seeded with fictional data. It makes no `/api` calls, enforces the same permissions per role and returns the same error codes, so the demo behaves like the real console. It lives at `/demo`, which links into `/demo/console?as=admin|operator|viewer`; inside the console a "view as" switcher changes role and a reset button restores the seed.
- **Every `/api/admin` request runs through `withAdmin`** (`core/http/with-admin.ts`): request id, per-IP rate limit (before the CSRF guard and any session lookup), CSRF guard and JSON-only body on non-GET requests, `requirePermission` against the database before input is parsed, per-user rate limit (after authorization), a 64 KB body cap, zod validation, then the error envelope `{error:{code,message,requestId,issues?}}`. Over either limit the answer is 429 `RATE_LIMITED` with `Retry-After`; the limiter fails open. Responses carry `Cache-Control: no-store`. See `docs/security.md` for the full order and reasoning.
- **Param routes and the system detail page.** A route entry in `src/features/routes.ts` may declare a `pattern` such as `systems/:id`. `findRoute` matches it segment by segment (a deeper or shallower path is unknown) and returns the route with its params, which the console router passes to the page. `systems/:id` has no navigation entry and requires `systems:read`; `expiring-credentials` is a second navigation-less drill-down, and `jobs/:id` (the job detail page, which requires `jobs:read`) is a third. The navigation has eleven items: the original seven, Rate limits and Activity, plus System health (P3c) and Queue (P4d, in Monitor after Dashboard). Since the UX rework the sidebar groups them under four headings (Monitor, Operate, Configure, Govern, the `group` field of each route), hides a heading whose items the role cannot open, and the console home is the first item the role can open in that order. Each system code in the Systems list links to the detail page. The page (`?tab=overview|credentials|endpoints|cors|health|certificates|sftp`) shows credential counts (Active, Rotating, Expired, Revoked) that link to the Credentials tab filtered by that status, and the Credentials tab issues, rotates, revokes and (when enabled) reveals credentials, stores the Flag type and a list of allowed origins, and shows the type's defaults in a read-only "Inherited from type" table. A card count and its list total come from the same `credentials.list` call, so they cannot drift.
- **The registry has 77 endpoints.** P4d added 1 (76 before; see "Queue and dashboards (P4d)" below). P4c3 added 1 (75 before; see "Jobs filters and export (P4c3)" below). P4c1 added 6 (69 before; see "Schedule management (P4c1)" below). P4b1 added 5 (64 before; see "Default credentials of an integration type (P4b1)" below). P3c added 6 (58 before; see "Alerts, notifications and system health (P3c)" below). P3b added 4 (54 before; see "Jobs observability (P3b)" below). P3a added 6 (48 before; see "Activity log and retention (P3a)" below). P2c added 7 (41 before; see "Certificates, SFTP and the catalog" below). P2b added 14 (27 before) and extended three. New: `systems.update` (`POST /systems/:id/settings`, which avoids shadowing `POST /systems/import`), `systems.delete`, `systems.restore`, `systems.duplicate`, `systems.export`, `systems.import`, `systems.checkHealth`, `endpointConfig.get`, `endpointConfig.save`, `integrationTypes.setAuthPolicy`, `integrationTypes.delete`, `integrationTypes.restore`, `rateLimits.monitor` (`GET /rate-limits`) and `credentials.expiring` (`GET /credentials/expiring`). Extended: `systems.list` (`view`, `health` filters), `integrationTypes.list` (`view`) and `integrationTypes.create` (`authPolicy`). A registry test asserts that no two same-method endpoints can match one concrete path.
- **Screens added in P2b.** The Systems list gains a Deleted tab (with Restore), Duplicate, Export and Import. The system detail page gains Endpoints (config and limits), CORS and Health tabs, and a Delete action. The Catalog page (the route slug stays `connectors`) shows each type's auth policy to everyone and can delete and restore types (since the UX rework, delete and restore of systems and types need `systems:delete`, which only the admin holds); only an admin (permission `auth-policy:write`) can change a policy, after a confirmation. The Rate limits page (nav) and the Expiring credentials page (drill-down) are new. The dashboard gets a Gateway section with four cards (deleted systems, health failing, rate-limited in the last hour, credentials expiring in 30 days); each count comes from the helper behind its destination list, so a card and the list it opens cannot disagree. Since the UX rework the dashboard opens with a Needs attention section: a card for each non-zero count among pending dead letters, failed jobs, failing health checks, expiring credentials and open alerts, linking to the same list as the matching count card (so the numbers agree); it says All clear only when every count that applies is known and zero, and says so when the open alerts could not be loaded. Each Recent jobs row links to its job.
- **Certificates, SFTP and the catalog (P2c).** Seven endpoints: `certificates.list` (`GET /systems/:id/certificates`, optional `status`), `certificates.upload` (`POST` on the same path) and `certificates.revoke` (`POST /certificates/:certId/revoke`); `sftp.get` (`GET /systems/:id/sftp`), `sftp.save` (`POST /systems/:id/sftp`) and `sftp.test` (`POST /systems/:id/sftp/test`); and `connectors.catalog` (`GET /connectors/catalog`). Reads need `systems:read`; the mutations use the new `certificates:write`, `sftp:write` and `sftp:test` permissions, and each writes an audit row (`certificate.upload`, `certificate.revoke`, `sftp.save`, `sftp.test`).
  - **Data.** Migration `0004` adds `system_certificate` (label, PEM, the optional encrypted key `enc_key_*`, fingerprint, subject and issuer CN, validity window, `revoked_at`; partial unique indexes on active label and active fingerprint per system) and reshapes `sftp_config`: the password columns and `remote_path` give way to `auth_method`, an encrypted `enc_secret_*` blob, `host_key_sha256`, a `paths` JSON object and `updated_at`, with at most one row per system.
  - **Services.** `core/services/certificates.ts` and `core/services/sftp.ts` hold the rules; `core/pure/x509-lite.ts`, `core/pure/certificate-status.ts` and `core/pure/sftp-config.ts` are browser-safe, so the real services and the demo accept, refuse and classify identically. Certificate status is derived, never stored.
  - **Probe flow.** `testSftpConnection` checks the system is live, takes the per-user and per-system rate limit, loads the row and calls `probeSftp` (`core/net/sftp-probe.ts`): `preflight` (`resolvePublicHost`), `connect` (dial the checked address), `handshake`, `hostkey` (compare with the pinned fingerprint; no pin or a different key stops here, before authentication and before the secret is decrypted), `auth` (one attempt), `done`. The probe never throws and returns fixed messages inside a 10 second budget. `ssh2` is loaded with a dynamic import and listed in `serverExternalPackages`, so a load failure breaks only the test. A save follows the same row: a host or port change clears the pin, and a host, port or username change clears the stored secret, unless the same save supplies a new one; an omitted port keeps the stored port. Details and the reasoning are in `docs/security.md`.
  - **Screens.** The system detail page gains Certificates (five status cards that each open the list they count, upload, revoke) and SFTP (config form, Pin this key, Test connection) tabs. A viewer sees no mutation controls. The Catalog page lists each connector's direction, version, input schema and sample with copy buttons.
  - **Demo.** The demo engine implements all seven endpoints. Its SFTP test is a simulation (no socket, no limiter), and it stores only a boolean for a certificate key.
- **Activity log and retention (P3a).** Six endpoints: `activity.list` (`GET /activity`, `logs:read`), `activity.get` (`GET /activity/:id`, `logs:read` and `logs:payload`), `activity.export` (`GET /exports/activity`, `logs:read` and `logs:export`), `activity.cleanup` (`POST /activity/cleanup`, `logs:cleanup`), `retention.get` (`GET /settings/retention`, `logs:read`) and `retention.save` (`POST /settings/retention`, `settings:write`). The export is not `GET /activity/export` because `GET /activity/:id` would match that path (the registry test refuses it). Cleanup, export and retention save each write an audit row (`activity.cleanup`, `activity.export`, `retention.save`). Migration `0005` adds `job.is_test`, `job.payload_sha256` (the idempotency compare hashes the body before redaction, so two bodies that differ only in a secret do not look equal) and four indexes, and grants the new permissions to the seeded roles.
  - **Screen.** The Activity log page (nav label "Activity", after Rate limits, needs `logs:read`) keeps its filters in the URL through one shared parser (`direction`, `systemCode`, `codeClass`, `scope`, `from`, `to`, `requestId`, `offset`), also used by the endpoint and the demo engine. The list carries no payload and a truncated address, and shows "Showing N of total". A payload is loaded lazily by `activity.get` when the row drawer opens "Show payloads" (only with `logs:payload`) and is rendered as preformatted text, never HTML. Export CSV needs `logs:export`: it covers the newest 5,000 rows of the current filter, with spreadsheet formula neutralisation (`core/pure/csv.ts`). Clean up (`logs:cleanup`, admin only) deletes either `expired` rows, which honours the kill switch, or `test` rows older than one hour, in at most 5 batches of 1,000 per call, and reports `{deleted, more, disabled}`. The retention panel shows the setting to `logs:read` and edits it with `settings:write` (days 1 to 365).
  - **Query cost.** List and export run a `count(*)` per page like every other list, bounded by retention. An admin list `offset` above 1,000,000 is a validation error.
  - **Test scope on jobs and the DLQ.** `job.is_test` marks a test run (a manual run flagged "Test run", which the operator chooses and the audit row records). One helper per table (`core/services/scope.ts`) gives `live` (default), `test` or `all`, and the dashboard cards, the jobs list and the DLQ list share it, so a count and the list it opens agree. The Jobs page "Show test runs" toggle sets `?scope=` for both the jobs list and the DLQ view.
- **Jobs observability (P3b).** Four endpoints, five new columns, one new table and a reshaped Jobs page, so an operator can answer "what happened to this job" without the database. Migration `0006_p3b_jobs_observability.sql` adds `job.failure_code` and `job.entity`; `job_log.step`, `job_log.seq` (an identity column) and `job_log.duration_ms`; the `process_log` table; the indexes behind the retention cutoff, the DLQ status filter, step order and audit search; and grants `jobs:discard` to the stored `operator` role with an idempotent update.
  - **Endpoints.** `jobs.get` (`GET /jobs/:id`, `jobs:read`) returns job metadata, the step log (`seq`, `step`, `level`, `message`, `durationMs`, `hasMeta`) and the job's DLQ history, with no payload, result or step meta. `jobs.payload` (`GET /jobs/:id/payload`, `jobs:read` and `logs:payload`) returns the payload, result and step metas redacted at read, and writes an audit row `job.payload_view` strictly: a read whose audit row cannot be written fails. `dlq.discard` (`POST /dlq/:jobId/discard`, `jobs:discard`, audit `dlq.discard`). `processRuns.list` (`GET /process-runs`, `jobs:read`, pages of 50). Extended: `dlq.list` (`status`), `dlq.replay` (`notes`), `audit.list` (filters, offset, rendered diff), `retention.save` (`jobDays`), and `activity.get`, which now writes audit `activity.payload_view` the same strict way. No list carries a payload or result body.
  - **Failure codes.** `core/pure/failure-codes.ts` holds a closed enum and `toFailureCode`, which maps an enum member to itself, the legacy aliases `HTTP` and `HTTP4xx` to `HTTP_5XX` and `HTTP_4XX`, and anything else to `UNKNOWN`. The drain, the inbound route and the reclaimer write it to `job.failure_code` and to `job_dlq.last_error.code`. Rows written before `0006` show "Not recorded". The list is in `docs/connectors.md`.
  - **Step log.** `makeJobLogger` (`core/jobs/lifecycle.ts`) writes rows in `seq` order and flushes before the terminal job write, so a job's steps precede its final state. The platform sets the steps (`received`, `claimed`, `validated`, `executed` with its duration, `succeeded`, `retry_scheduled`, `dead_lettered`, `failed`, `reclaimed`, `replayed`, `discarded`); a connector's own lines are stored with step `connector`. A job keeps at most 200 rows (`JOB_LOG_MAX_ROWS`): connector lines stop at 150 rows and 20 per attempt, platform steps at 199, and one `truncated` marker counts the dropped lines. Since P4a the terminal step is written after that marker, so the marker follows every connector row and the terminal step is last. The `replayed` and `discarded` steps are written inside the replay or discard transaction, so neither action commits without its step and a losing concurrent replay writes none; moving a job to `DLQ` and inserting its DLQ row are one transaction too.
  - **Entity.** A connector may declare `ConnectorMeta.entityPath`, a dot path into the payload (`reference` for `rest-inbound`, `payload.orderId` for `webhook-inbound`). The string or number found there becomes `job.entity`, redacted and clipped to 128 characters. It is display only.
  - **Process log.** `recordRun` (`core/jobs/process-log.ts`) writes one `process_log` row per tick, dispatch, drain (a manual run or `/api/jobs/process`) and retention run, with status, duration and a counts-only summary. A signed QStash call keys its row on `Upstash-Message-Id`, so a duplicate delivery inserts nothing, answers 200 `{ok:true, duplicate:true}` and runs nothing. A `RUNNING` row older than 15 minutes is shown as "Did not finish". The dedupe is log hygiene, not a security control: the signature covers the body and url, not the `Upstash-*` headers. A failure to write the row never fails the run. A stage error is stored without the query's bound parameters (the `params:` tail is cut), and the tick's response and server log carry the same stripped text.
  - **Retention.** The `retention` setting gains `jobDays` (default 30). `runJobPrune` (`core/jobs/prune.ts`) deletes `job_log` rows, then the jobs in `SUCCESS`, `FAILED` or `DLQ` older than the cutoff (their closed DLQ rows go with them), then old `process_log` rows, in batches: 100 jobs by 5 batches after a response, 200 by 10 in the tick. A job with an open DLQ item is never deleted, and the kill switch stops every path.
  - **Audit search.** `audit.list` filters by `action`, `targetTable`, `targetId`, `actor`, `from` and `to`, kept in the URL, in pages of 50. Rows and total come from one `auditWhere`, so the footer equals what the pager reaches. The diff is rendered on the server by `renderAuditDiff` (`core/pure/audit-diff.ts`) from an allowlist and fails closed; see `docs/security.md`.
  - **Screens.** The Jobs page has Jobs, Dead letters and Runs views (`?view=`). The job detail page (`jobs/:id`) shows the metadata, the step timeline, the DLQ history with notes and, only with `logs:payload`, a "Show payload" button that loads `jobs.payload` on demand. A job whose dead letter still awaits review also offers Retry and Discard (`JobDlqActions`), with the same dialogs as the Dead letters view, then refetches to show its new status. A job already removed by retention shows "This job is no longer retained".
- **One contract suite, two implementations.** `test/contract/admin-api.contract.ts` defines the behaviour (counts match filtered lists, validation before side effects, viewers refused on every mutation, duplicate codes conflict) and runs against both `HttpAdminApi` (`test/contract/http.test.ts`) and `DemoAdminApi` (`test/contract/demo.test.ts`). A registry test checks that every endpoint has a permission and every mutation has an audit action.
- **Capability flag.** `AdminApi.capabilities` lets an implementation say what it supports. Since P3c notifications are `true` on both implementations (`HttpAdminApi` is now backed by the `notification` tables), and the bell and Notifications page still render only when the flag is set.
- **Alerts, notifications and system health (P3c).** Six endpoints (58 before): `notifications.list`, `notifications.markRead`, `notifications.markAllRead`, `alerts.list`, `alerts.evaluate` ("Evaluate now") and a system health summary. All are gated by the new `alerts:read` permission except `alerts.evaluate`, which needs `settings:write`; the per-system table of the health summary additionally needs `systems:read`. Migration `0007_p3c_alerts.sql` extends `alert`, creates `notification` and `notification_read`, and grants `alerts:read` to the stored `operator` and `viewer` roles with an idempotent update. The screens are System health (a nav item) and the Notifications page, with a bell in the shell.
  - **Rules.** One pure module, `core/pure/alert-rules.ts`, holds the rules, thresholds, messages and the reconcile plan. The server evaluator (`core/jobs/alerts.ts`) and the demo engine both call it, so the two cannot disagree about when an alert opens, escalates or resolves.

    | Rule | Entity | Fires when | Warning | Critical |
    |---|---|---|---|---|
    | `dlq_open` | system | live `PENDING_REVIEW` dead letters on a system | 1 or more | 10 or more |
    | `job_failure_rate` | system | failed (`FAILED` + `DLQ`) share of terminal jobs in the last 60 minutes, at least 10 terminal jobs | 25% | 50% |
    | `inbound_error_rate` | system | 5xx share of inbound calls in the last 15 minutes, at least 20 calls | 20% | 50% |
    | `cert_expiry` | certificate | a non-revoked certificate of a live system is near or past its end date | within 30 days | within 7 days, or expired |
    | `system_down` | system | an `ACTIVE`, non-deleted system whose stored health is `DOWN` | always | never |
    | `tick_stale` | platform | the tick is armed and has not run for 15 minutes (or has never run) | always | never |
    | `job_stuck` | job | a `PROCESSING` job locked longer than the stuck-job minutes setting (`JOB_STUCK_MINUTES`) | always | never |

  - **Fingerprint and lifecycle.** Each firing has the fingerprint `<rule>:<entity_type>:<entity_id>` (for example `dlq_open:system:<code>`); there is at most one OPEN alert per fingerprint. A firing with no open row opens one and writes a notification. A higher severity escalates the row and writes one notification. A lower severity or a changed message is a quiet update with no notification. An open row whose fingerprint no longer fires resolves. Each rule keeps at most 100 firing entities per run (`job_stuck`: 50); a rule cut by its cap is not auto-resolved in that run.
  - **Idempotency, three layers.** (1) A transaction-scoped advisory lock (`pg_try_advisory_xact_lock`): a run that does not get the lock writes nothing. (2) A partial unique index on the fingerprint, `alert_open_fingerprint_uq ... WHERE status = 'OPEN'`. (3) Every open is an `INSERT ... ON CONFLICT (fingerprint) WHERE status = 'OPEN' DO UPDATE`, whose update applies only when the severity or message `IS DISTINCT FROM` the stored values, so a repeat run changes nothing and notifies nobody.
  - **Bounded work.** Opens plus resolves are capped at `MAX_TRANSITIONS` (200) per run, opens first and worst first. The rest wait for the next run and the result reports `more`. Escalations and quiet updates touch existing rows and are not capped.
  - **Triggers.** (1) The tick's `alerts` stage. (2) A throttled `after()` run on admin reads and when an inbound call finishes, at most once per 300 seconds across instances through a compare-and-set on the `alerts_last_run` row of `app_setting` (a stamp more than 60 seconds in the future counts as stale); these are behind `alertTriggerEnabled()`, which is off under test. (3) The admin "Evaluate now" action, which ignores the throttle, still takes the advisory lock and stamps the throttle. Each run is recorded as an `alerts` row in `process_log`.
  - **Notifications.** Read state is per user (`notification_read`, removed with the notification or the user), so one operator reading an alert does not hide it from another. The list is capped at 1000 (`pruneNotificationsToCap`, 500 deletions per run, independent of the retention kill switch). Retention prunes resolved alerts and notifications older than `jobDays`; an OPEN alert is never pruned, and job and activity pruning keep any row an OPEN job alert names. See `docs/security.md`.
  - **Demo.** The demo engine re-evaluates synchronously after each mutation, with no throttle and no lock, and records an `alerts` run only on an explicit Evaluate now. `job_stuck` never fires in the demo (its snapshot has no stuck jobs).
- **Default credentials of an integration type (P4b1).** Five endpoints (64 before, 69 now): `typeCredentials.list` (`GET /integration-types/:id/credentials`, `systems:read`), `typeCredentials.store` (`POST` on the same path, `credentials:write`), `typeCredentials.rotate` (`POST /type-credentials/:credId/rotate`, `credentials:write`), `typeCredentials.revoke` (`POST /type-credentials/:credId/revoke`, `credentials:write`) and `typeCredentials.reveal` (`POST /type-credentials/:credId/reveal`, `credentials:reveal` and the server reveal flag). Each mutation writes an audit row (`type_credential.store`, `.rotate`, `.revoke`, `.reveal`); the permission ids stay at 24. The service (`core/services/type-credentials.ts`) locks the type row and bumps its `credential_version` in the same transaction as every write; store and rotate refuse a soft-deleted type, revoke still works on one.
  - **Screens.** The Credentials tab of a system gains the Flag type, an "Allowed origins" field (one origin per line) on store and rotate, and a read-only "Inherited from type" table (Label, Type, Status) that marks a row "Overridden" when the system has an active credential with the same label. The Endpoints tab gains "Auth credential label" and "API key header" fields, with a warning when no active credential on the system or its type carries the label. The Catalog page gains a "Default credentials" button on each live type (`systems:read`) that opens a dialog with Store, Rotate, Revoke and Reveal, each behind its permission.
  - **Demo.** The demo engine mirrors the endpoints and the rules through the shared schemas and `core/pure/credential-rules.ts`; a `runJob` whose `authLabel` has no active row at either tier dead-letters `CREDENTIAL_MISSING` at attempt 1.
- **Schedule management (P4c1).** Six endpoints (69 before, 75 now): `schedules.get` (`GET /schedules/:id`, `schedules:write`, the only read that returns the payload), `schedules.create` (`POST /schedules`), `schedules.update` (`POST /schedules/:id/update`, cron and payload only), `schedules.delete` and `schedules.restore` (`POST /schedules/:id/delete` and `/restore`, the new Admin-only `schedules:delete`) and `schedules.run` (`POST /schedules/:id/run`, `schedules:write` and `jobs:run`). `schedules.list` takes `view` (`deleted`), and its rows and total share that filter. Each mutation writes an audit row (`schedule.create`, `.update`, `.delete`, `.restore`, `.run`) whose diff carries payload key names, never values. Migration `0010_p4c_schedules.sql` adds `job_schedule.deleted_at`, `last_manual_at`, `updated_at` and the partial index `job_schedule_due_idx`.
  - **Rules.** A schedule belongs to a system by `system_code`, and its handler is the system's integration type handler; the payload is validated by the connector's input schema. Create and update refuse a deleted system, and create refuses a non-outbound one. A new schedule starts disarmed, and editing an armed one recomputes `next_run_at`. Delete is soft: it disarms the row in the same statement (and removes a legacy per-schedule QStash schedule, as disarm does), and restore brings it back disarmed. Every mutation of a deleted schedule, and restoring a live one, is `409`.
  - **Run now.** One transaction locks the schedule row, applies the same system gates as `jobs.run` (`409` for a deleted or non-`ACTIVE` system), wins a compare-and-swap on `last_manual_at` that allows one manual run per schedule per 10 seconds (a second run inside the window is `409`), and inserts the job with idempotency key `schedule:<id>:manual:<last_manual_at>`, which never equals the tick's key, so a tick and a manual run in the same minute make two jobs. A failed insert rolls the stamp back. The audit row and the drain kick follow the commit. It works on a disarmed schedule and leaves `next_run_at` and `last_run_at` alone.
  - **Demo.** The demo engine mirrors the endpoints and rules, including the 10 second window on its own clock.
  - **Screens (P4c2).** The Schedules page has Live and Deleted tabs. A New schedule drawer (`schedules:write` and `systems:read`, since it lists systems) and an Edit drawer share one form: the system (fixed on Edit), the handler it implies, a cron builder and a JSON payload whose server issues land on their field. The cron builder offers presets (every N minutes for steps that divide 60, hourly, daily, weekly, and monthly on days 1 to 28) and a Custom expression; a stored expression opens as a preset only when it is exactly that preset's text, and is never rewritten. It previews the next 5 runs in UTC and in the viewer's local time, or names the problem (`invalid_cron`, `never_fires`) and blocks Save. Each row offers Arm or Disarm, and a menu with Edit, Run now (`schedules:write` and `jobs:run`, with a notice linking to the new job) and Delete (behind a confirmation); the Deleted view offers Restore. Each control checks its own permission, so the operator sees Edit and Run now but never Delete or Restore, and the viewer sees no write control.
- **Jobs filters and export (P4c3).** One endpoint (75 before, 76 now): `jobs.export` (`GET /exports/jobs`, `jobs:read` and `logs:export`; not `/jobs/export`, which `GET /jobs/:id` would match). `jobs.list` gains five filters next to `status` and `scope`: `systemCode`, `integrationTypeId`, `failureCode` (the closed enum) and `from` / `to` on `created_at` (inclusive, ISO with an offset; `from` after `to` is a validation error). P4d adds two more, `queue` (a queue state) and `asOf` (the instant it is read at; `asOf` without `queue` is a validation error), so there are seven filter keys (see "Queue and dashboards (P4d)"). One set of field rules (`core/pure/job-filters.ts`) feeds both endpoints and the Jobs screen's URL parser, and one `buildJobsWhere` (`core/services/jobs.ts`) builds the list, its count and the export, so the three agree. Migration `0011_p4c_job_list_idx.sql` adds `job_created_idx (is_test, created_at DESC, id DESC)` for the newest-first page.
  - **Type filter.** `integrationTypeId` matches through the job's system row (its `system_code`, then that system's type), soft-deleted systems included. A job whose system row no longer exists never matches it, and the screen says so under the filter.
  - **Export.** The newest 5,000 rows of the filter (`JOBS_EXPORT_MAX`), with `total` and `truncated`, and an audit row `jobs.export` that records the filters (`queue` and `asOf` included since P4d), the row count and whether it was cut. The export honours `queue` and `asOf` like the list does. Columns: id, system, handler, direction, status, attempts, failure code, test, created, processed. Never the payload, the entity or the upstream error. The CSV is built in the browser with `toCsv`, so it gets the same neutralisation as the activity export.
  - **Screen.** The Jobs page keeps the filters in the URL. Switching between the Jobs, DLQ and Runs views clears them, and the list footer counts them as filtered. The Type list needs `systems:read`. Export CSV shows only to `logs:export` holders, says when the file was cut ("First 5,000 of N"), and its notice is dropped when the filters change. The demo engine applies the same filters and export.
  - **Bounds and the cut flag (shared with Activity and Audit).** `to` is inclusive to the millisecond on a microsecond column: the bound is `created_at < to + 1 ms` (`upToInclusive`, `core/services/upper-bound.ts`), falling back to `<= to` at 9999-12-31T23:59:59.999Z, the largest value the schemas accept. A `from` or `to` with seconds that arrived in a link is kept on Apply unless that field was edited, so it is never rounded to the minute. Both exports read `max + 1` rows and set `truncated` from that read, not from a separate count, so a row written between the two reads cannot mark a complete export as cut. The Export CSV button states that it uses the applied filters.
- **Auth boundary.** The admin layout (`src/app/admin/layout.tsx`) still performs the database-backed `requireActiveUser()` check and redirects to `/login`; the API is the gate on every call.

### Queue and dashboards (P4d)

The Queue page answers one question: where do queued jobs stand right now, and what is holding them. It adds one endpoint (`dashboard.queue`, 76 before, 77 now), two Jobs filters and one page. There is no new table, permission, environment key or migration.

**Queue states.** A job's place in the queue is one of five states, all read at one instant, `asOf`. The pure rule is `queueStateOf` (`core/pure/queue-state.ts`); the SQL is `queueStateWhere` (`core/services/jobs.ts`), built from the same fragments as the drain's claim and the reclaim cutoff, and the demo engine implements the same rule.

| State | A job is in it when |
|---|---|
| Queued | its status is `PENDING` or `RETRY` |
| Held | it is queued and its system is a live (not soft-deleted) row whose status is not `ACTIVE` |
| Due now | it is queued, not held, and `next_attempt_at` is unset or at or before `asOf`: what the drain would claim |
| Waiting | it is queued, not held, and `next_attempt_at` is after `asOf` |
| Stuck | its status is `PROCESSING` and `coalesce(locked_at, updated_at)` is strictly before `asOf` minus `JOB_STUCK_MINUTES` (default 10): what the next reclaim would take |

Queued is always Due now plus Held plus Waiting. Stuck is separate, because a stuck job is `PROCESSING`, not queued. Stuck is defined by the reclaim sweep (see "Stuck jobs are reclaimed"), not by the `job_stuck` alert rule. A job of a deleted system is not held: it fails without retry when it is claimed.

**One `asOf` for every number.** The endpoint reads the clock once and returns it as `asOf`. Every count, both oldest-job values and every link on the page use that instant. A count's link is the Jobs list with `queue=<state>&asOf=<asOf>` (or `status=<status>`) plus the page's scope, so the list recomputes the same state at the same instant and its total is the number that was clicked. `buildJobsWhere` builds the Jobs list, its count, its export and the dashboard counts (each cell is a `count(*) FILTER` over that one WHERE), so the numbers cannot drift apart. A job can still move between the read and the click; the page says so. Without `asOf`, the Jobs list reads the queue state at the current time.

**The endpoint.** `dashboard.queue` is `GET /dashboard/queue` and needs both `systems:read` and `jobs:read`, because the page shows system rows and job counts side by side. Input (strict): `scope` (`live` by default, as `jobs.list`), `systemCode` and `integrationTypeId`, with the same field rules as the Jobs filters. Output:

- `asOf` and `stuckMinutes`.
- `drain`: `lastWindowAt` (the last after-response drain window, or null), `tickArmed` and `tickSigning`.
- `totals`: the six statuses, the five queue states, `oldestQueued` (`jobId`, `createdAt`) and `oldestDue` (`jobId`, `dueAt`, which is `next_attempt_at` or else `created_at`).
- `byType` and `bySystem`: the same counts per integration type and per system. A job whose system row is gone has no type and shows as one "No type" row. `bySystem` is capped at 500 rows, most queued first, and `bySystemTruncated` says when rows were cut. The totals and `byType` are sums of the grouped query, so they never depend on that cap.

**A passive read.** Unlike `jobs.list`, `jobs.get`, `dlq.list` and `dashboard.summary`, this read never triggers an after-response drain window. Opening the page cannot run a job, claim a job or move the numbers it is showing, and it does not write the `drain_last_run` stamp.

**The posture panel.** The Runner panel reads the `drain` block and states what moves queued jobs: "Timer tick: not armed", "Timer tick: armed", or "Timer tick: armed, but its calls are refused (signing keys not set)", followed by the fallback sentence (queued jobs run when the console is read or a partner call is answered, up to 3 jobs a minute, or right after Run now) and the last drain window ("No drain window yet" when there is none). `tickSigning` is true only when both QStash signing keys are present (or the non-production dev bypass is on). It reports presence, never a key value, and the value is never read out of `core/jobs/qstash-signing.ts`. The same function backs the 503 branch of the QStash verifier, so the panel and the verifier cannot disagree. This is the dormant posture (above) made visible: an armed tick with no signing keys looks on, but every call it makes is refused.

**Jobs filters.** `jobs.list` and `jobs.export` accept `queue` (`queued`, `due`, `held`, `waiting` or `stuck`) and `asOf` (an ISO instant with an offset, years 2000 to 9999). `asOf` without `queue` is a validation error (400), and the Jobs screen's URL parser drops it rather than failing. The filter set is now seven keys, and the `jobs.export` audit row names `queue` and `asOf` with the other filters.

**The page.** Queue (`queue`, needs `systems:read` and `jobs:read`) is the second item of the Monitor group, after Dashboard. Scope comes from the URL: `systemCode` or `integrationTypeId` (a value that fails the Jobs filter rules is dropped, never an error), and a "Show test runs" toggle that sets `scope=all`. The page has the Runner panel, eight cards (Queued, Due now, Held, Waiting, Stuck, Retry backlog, Oldest queued, Oldest due), a Jobs by status row of six cards, and a By integration type and a By system table, each with the five state columns and a Retry column. Every count links to the Jobs list it equals; the two oldest-job cards link to the job. A scope change refetches instead of showing the old numbers under the new header. Entry links to the page appear on the Dashboard, in the Catalog row menu ("Dashboard") and on the system detail page ("Dashboard"), each scoped to what it sits on and each shown only when the role can open the Queue route (capability and every permission it needs), so no link leads to a refused page.

**Demo.** The demo engine implements the same endpoint and the same states, and the seed world has every queue state non-zero, including a Waiting `RETRY` job due a week ahead. In the demo `tickSigning` is true and `lastWindowAt` is the last simulated drain run. Reading the demo console does not drain the seeded queue, so every state stays visible; only a manual run is simulated, and it runs that one job.

Source: `core/pure/queue-state.ts`, `core/services/queue-stats.ts`, `core/services/jobs.ts`, `core/pure/job-filters.ts`, `core/jobs/qstash-signing.ts`, `src/admin-api/endpoints/dashboard.ts`, `src/features/queue/`.

## Where to look in the code

| Concern | File |
|---|---|
| Inbound route | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts` |
| Inbound auth guard | `core/auth/with-partner-auth.ts` |
| Connector contract | `core/connectors/types.ts` |
| Connector resolution | `core/connectors/resolver.ts` |
| Generated manifest | `src/connectors/generated/manifest.ts` |
| Generated catalog (schemas and samples) | `src/connectors/generated/catalog.ts` |
| Manifest codegen (also writes the catalog) | `scripts/generate-connector-manifest.ts` |
| Catalog builder (build time only) | `scripts/connector-catalog.ts` |
| Job lifecycle helpers | `core/jobs/lifecycle.ts` |
| Drain / retry / DLQ, after-response drain | `core/jobs/drain.ts`, `core/jobs/drain-trigger.ts` |
| Retry policy (classification, Retry-After, backoff) | `core/pure/retry-policy.ts` |
| Outbound HTTP client base (guarded call, status mapping, clipped body) | `core/net/outbound-client.ts` |
| Non-retryable connector error | `core/connectors/errors.ts` |
| P4a migration (`job.upstream_error`) | `db/migrations/0008_p4a_engine.sql` |
| Rate limiting | `core/gateway/rate-limit.ts` |
| Activity log | `core/gateway/log-call.ts`, `core/pure/redact.ts`, `core/pure/csv.ts` |
| Activity service, retention prune | `core/services/activity.ts`, `core/jobs/prune.ts` |
| Alert rules, thresholds, messages, reconcile plan | `core/pure/alert-rules.ts` |
| Alert evaluator, throttle, alert and notification retention | `core/jobs/alerts.ts`, `core/jobs/alert-trigger.ts` |
| Alerts and notifications service | `core/services/alerts.ts` |
| Alerts, notifications and health endpoints | `src/admin-api/endpoints/alerts.ts` |
| System health page | `src/features/health/SystemHealthPage.tsx` |
| Alerts migration | `db/migrations/0007_p3c_alerts.sql` |
| Logging audit (Rules A, C, D) | `scripts/audit-log-calls.ts` |
| Service token (verify, mint) | `core/auth/service-token.ts`, `scripts/mint-service-token.ts` |
| Outbound network guard | `core/net/ssrf.ts`, `core/pure/address-policy.ts` |
| SFTP probe | `core/net/sftp-probe.ts` |
| Certificate and SFTP services | `core/services/certificates.ts`, `core/services/sftp.ts` |
| Certificate parsing, status, SFTP shapes | `core/pure/x509-lite.ts`, `core/pure/certificate-status.ts`, `core/pure/sftp-config.ts` |
| Strict connector input | `core/pure/connector-input.ts` |
| Endpoint names | `core/pure/endpoint-name.ts` |
| System export document | `core/pure/system-export.ts` |
| Rate-limit monitor, expiring credentials | `core/services/rate-limits.ts`, `core/services/credentials.ts` |
| Failure codes, job steps | `core/pure/failure-codes.ts`, `core/pure/job-steps.ts` |
| Job detail and payload view | `core/services/job-detail.ts` |
| Process log, process runs | `core/jobs/process-log.ts`, `core/services/process-runs.ts` |
| Audit search and rendered diff | `core/services/audit.ts`, `core/pure/audit-filters.ts`, `core/pure/audit-diff.ts` |
| Queue states (pure rule, SQL) | `core/pure/queue-state.ts`, `queueStateWhere` in `core/services/jobs.ts` |
| Queue dashboard service and endpoint | `core/services/queue-stats.ts`, `src/admin-api/endpoints/dashboard.ts` |
| Queue page and entry links | `src/features/queue/` |
| Signing-keys presence check (never a value) | `core/jobs/qstash-signing.ts` |
| File-drop payload schema (shared by server, schedules and demo) | `core/pure/file-drop-sftp-payload.ts` |
| Run drawer file picker | `src/features/systems/FileDropPicker.tsx`, `src/features/systems/file-drop-file.ts` |
| Tick (reclaim, schedules, drain) | `src/app/api/jobs/tick/route.ts` |
| Schedule dispatch | `src/app/api/jobs/dispatch/route.ts`, `core/jobs/scheduler.ts` |
| Stuck-job reclaim | `core/jobs/reclaim.ts` |
| QStash signature check | `core/jobs/verify-qstash.ts` |
| QStash arming | `core/jobs/qstash.ts` |
| Credential vault | `core/crypto/vault.ts` |
| Config + credential resolution (system row, then type default) | `core/config/resolve-config.ts` |
| Decrypt cache (TTL 300 s, cap 500, version-keyed) | `core/config/credential-cache.ts` |
| Type default credentials service and endpoints | `core/services/type-credentials.ts`, `src/admin-api/endpoints/type-credentials.ts` |
| Credential value, header and origin rules (browser-safe) | `core/pure/credential-rules.ts` |
| Flags | `core/connectors/flags.ts` |
| P4b1 migration (credentials) | `db/migrations/0009_p4b_credentials.sql` |
| OAuth client credentials (secret rules, token client and cache) | `core/pure/oauth-client-secret.ts`, `core/net/oauth-client.ts` |
| P4b2 migration (`oauth_token_cache`) | `db/migrations/0012_p4b2_oauth.sql` |
| Async inbound: `Prefer` parsing | `core/pure/prefer.ts` |
| Async inbound: payload vault (seal, open, purge), purpose-key crypto | `core/jobs/payload-vault.ts`, `core/crypto/vault.ts` (`encryptForPurpose`) |
| Async inbound migration (`accepted_async`, `job_payload_vault`) | `db/migrations/0013_async_inbound.sql` |
| Operational schema (12 tables) | `db/schema/operational.ts` |
| Config/auth schema (19 tables) | `db/schema/config-auth.ts` |
| Opt-in upstream schema | `db/schema/upstream.ts` |
| DB client | `core/db.ts` |
| Admin API contract | `src/admin-api/contract.ts` |
| Admin endpoint registry | `src/admin-api/endpoints/` |
| Admin catch-all route | `src/app/api/admin/[...path]/route.ts` |
| Admin request wrapper | `core/http/with-admin.ts` |
| Edge proxy (request id, CSP, HSTS, CORS) | `src/proxy.ts`, `core/http/security-headers.ts` |
| Credential lifecycle services | `core/services/credentials.ts`, `core/pure/credential-status.ts` |
| System detail page | `src/features/systems/SystemDetailPage.tsx` |
| HTTP and demo implementations | `src/admin-api/http/client.ts`, `src/admin-api/demo/` |
| Console UI and routes | `src/features/`, `src/features/routes.ts` |
| Contract suite | `test/contract/admin-api.contract.ts` |
