# Architecture

Last updated: 2026-10-10

Interchange is the single governed junction between external partner systems and a project's core systems. This document describes how a request actually flows through the platform, how the connector plugin model works, how the job engine and dead-letter queue behave, the two-database model, and the dormant-by-default posture.

## Stack

Next.js 16 (App Router) on Vercel, Drizzle ORM, Neon serverless Postgres, TypeScript, zod for schema validation. There is no separate worker process and no Redis: the job queue lives in Postgres itself.

## The core idea

Partners never talk to a project's internal systems directly. They talk to Interchange. Interchange authenticates them, validates their payload, runs (or queues) the work, and logs everything. This gives a project one place to see what happened across every partner integration, instead of that logic being scattered across bespoke handlers.

## Inbound request lifecycle

An inbound request is a partner calling into the platform, for example to submit a record or notify of an event. It is handled synchronously: the caller is HTTP-waiting, so the connector's `execute()` runs inline rather than being queued.

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
  |                             |    - look up external_system by      |
  |                             |      systemCode from the URL         |
  |                             |    - require status = ACTIVE         |
  |                             |    - verify against system_api_key,  |
  |                             |      system_bearer_token or          |
  |                             |      system_basic_auth (timing safe) |
  |                             |    - stamp last_used_at on the match |
  |         401                 |  (all pre-auth failures: one body)   |
  |<-----------------------------                                      |
  |                             | 1b. per-system rate limit (bucket    |
  |         429 + Retry-After   |    sys:<code>, external_system       |
  |<-----------------------------    .rate_limit_rpm)                  |
  |                             | 2. resolve integration_type ->       |
  |                             |    handlerKey, resolveHandler()      |
  |                             |    (static generated manifest)       |
  |                             |                                      |
  |                             | 3. connector.inputSchema.safeParse() |
  |         400 / 422           |    (bad JSON / validation failure)   |
  |<-----------------------------                                      |
  |                             | 4. createJob(...): INSERT ... ON     |
  |                             |    CONFLICT DO NOTHING on            |
  |                             |    (system_code, idempotency_key),   |
  |                             |    status=PROCESSING                 |
  |  replay of a finished job:  |------------------------------------->|
  |  original result +          |<-------------------------------------|
  |  Idempotent-Replayed: true  |                                      |
  |<-----------------------------                                      |
  |                             | 5. connector.execute(ctx) runs INLINE|
  |                             | 6. markSuccess / markFailed          |
  |    200 (or 502 / 500)       |------------------------------------->|
  |<-----------------------------                                      |
  |                             | Every exit above (incl. 401/429/4xx  |
  |                             | also writes an activity_log row      |
```

Source: `src/app/api/inbound/[systemCode]/[endpoint]/route.ts`, `core/auth/with-partner-auth.ts`, `core/gateway/rate-limit.ts`, `core/gateway/activity.ts`, `core/connectors/resolver.ts`, `core/jobs/lifecycle.ts`.

Properties worth calling out:

- **systemCode confinement.** The auth guard resolves the partner system named in the URL first, then checks whether the presented credential belongs to that system. A valid credential for system A can never authenticate a call to system B's routes, even if both are ACTIVE.
- **Rate limiting is Postgres-backed and fails open.** Each check is one atomic `INSERT ... ON CONFLICT (key, window_start) DO UPDATE SET count = count + 1 RETURNING count` against `rate_limit_bucket` (one-minute fixed windows). Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; a rejected call gets 429 with `Retry-After`. If the database is unreachable the limiter logs a warning and lets the request through, because a limiter outage must not become a gateway outage. The client IP is taken from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, and must parse as an IP address (anything else shares an `unknown` bucket); on a host that does not overwrite `x-forwarded-for` that last fallback is client-controlled, so put a trusted proxy in front. The per-IP limit (`RATE_LIMIT_IP_RPM`, default 250 per minute) is checked before authentication, which throttles credential guessing; the per-system limit (`external_system.rate_limit_rpm`) is checked after, so strangers cannot burn a partner's quota. Per-endpoint limits and a rate-limit monitor screen are not built yet (planned).
- **Idempotent retries.** A repeated `Idempotency-Key` from the same system returns the original job's result with `Idempotent-Replayed: true` instead of creating a second job or failing. The same key with a different payload is a 422; a duplicate that arrives while the original is still running is a 409. The key is trimmed (an empty key counts as absent), at most 200 printable ASCII characters (else 400), and stored with an `in:` prefix so a partner key can never collide with a scheduler key. Only true replays carry `Idempotent-Replayed: true`; the 422 does not.
- **Every outcome is logged.** `activity_log` (`core/gateway/activity.ts`) gets one row per inbound call, including auth failures, rate limits and validation errors. Both payloads are required by the TypeScript type (use `{}` or `{ reason }` when there is no body), secret-looking keys (auth, authorization, bearer, jwt, otp, pin, secret, token, password, apikey, card, cvv, ssn, signature, cookie, session and similar, matched case-insensitively at any depth) are redacted, and credentials or `Authorization` headers are never stored. The row is written after the response is sent (`after()` on the Vercel runtime) so logging adds no latency. `activity_log` rows older than `ACTIVITY_RETENTION_DAYS` (default 30) are pruned by the tick. This is separate from `admin_audit_log`, which records admin actions.
- **Fail-fast on an unknown handler.** If `integration_type.handler_key` does not exist in the generated manifest, `resolveHandler()` throws `UnresolvedHandlerError` immediately rather than queuing a job that will never succeed.

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
                (decrypts credentials via the vault,
                 see docs/security.md)
                                |
                connector.execute(ctx)
                                |
              +-----------------+------------------+
              |                 |                   |
         SUCCESS            RETRY/thrown         non-retryable
              |             (transient)           FAILED
              v                 |                   |
      status=SUCCESS            v                   v
      result stored     status=RETRY,        status=DLQ
                         nextAttemptAt =      + job_dlq row
                         now + backoff(n)     (reason, lastError)
                         attempts < maxAttempts   attempts >= maxAttempts
                                |                   OR error.retryable=false
                                |
                        (next drain picks it
                         up again once due)
```

Source: `core/jobs/drain.ts`, `core/jobs/lifecycle.ts`, `core/jobs/scheduler.ts`, `core/jobs/reclaim.ts`, `core/jobs/verify-qstash.ts`, `db/schema/operational.ts`.

Key mechanics:

- **The queue is not a separate table.** It is `job` rows with `status IN ('PENDING', 'RETRY')` whose `next_attempt_at` is due. This is a deliberate simplification: one source of truth for both "what ran" and "what's next".
- **Claiming is race-safe without Redis.** `FOR UPDATE SKIP LOCKED` inside a transaction lets multiple concurrent invocations of the drain (which can happen on serverless, since nothing guarantees single-flight) each claim a disjoint batch.
- **Backoff is exponential**, based on the attempt count: `BACKOFF_BASE_MS * 2^(attempts - 1)`.
- **The DLQ (`job_dlq`) is a queryable ledger of terminal failures**, each pointing back at the original `job.id`, with the reason (`retries exhausted` or `non-retryable failure`) and the last error payload. `replayDlq(jobId)` resets a DLQ'd job to `PENDING` with `attempts` cleared, so it re-enters the drain on the next pass. The function exists and is tested; the admin action and the `jobs:run` / `jobs:retry` permission checks that will call it are wired in P1b.
- **Schedules (`job_schedule`) fire by cron, exactly once.** Each enabled schedule has a `next_run_at`. A tick claims a due schedule with a compare-and-swap (`UPDATE ... WHERE id = ? AND next_run_at = ? RETURNING`), so two overlapping ticks cannot both fire it, then enqueues one PENDING outbound job and computes the next `next_run_at` from the cron expression (croner, UTC). The claim and the job insert share one transaction, so a failed insert rolls the claim back and the next tick retries the same slot. A schedule registers nothing with QStash: its cadence is its own cron, evaluated by the tick, so it can run no more often than the tick. `/api/jobs/dispatch` does only this enqueue step, and only for due schedules.
- **`/api/jobs/tick` is the single background driver.** One signed QStash call per cadence runs, in order: stuck-job reclaim, due-schedule dispatch, a bounded `drainBatch()`, and a prune of expired rate-limit buckets and old `activity_log` rows. Each stage has its own try/catch, so one failure cannot skip the others; if any stage failed the response is 500 with every stage's result. There is exactly one QStash schedule, cadence `QSTASH_TICK_CRON` (default `*/5 * * * *`), created by the explicit "Arm tick" action on the Schedules page (`armTick()`, id stored in `app_setting`) and removed by "Disarm tick". It is dormant until armed.
- **Stuck jobs are reclaimed.** A job left `PROCESSING` with `locked_at` older than `JOB_STUCK_MINUTES` (default 10) lost its worker. An outbound job returns to `RETRY` with backoff while attempts remain, otherwise goes to the DLQ through the same path as a normal failure; this is at-least-once delivery, so outbound handlers and the partners they call must be idempotent. An inbound job is never re-executed: it goes to `FAILED`, and the partner's retry with the same `Idempotency-Key` receives that recorded failure.
- **QStash calls are signature-verified.** `/api/jobs/tick`, `/dispatch` and `/process` verify the `Upstash-Signature` header against the raw body with `QSTASH_CURRENT_SIGNING_KEY` / `QSTASH_NEXT_SIGNING_KEY`. The expected url is built from `APP_URL` (else `NEXTAUTH_URL`) plus the request path and passed to the verifier, so a signature minted for one endpoint cannot be replayed against another. With the keys unset they fail closed (503). The only bypass is an explicit `QSTASH_DEV_BYPASS=1`, honoured outside production only.
- **DLQ review state.** `job_dlq.status` is `PENDING_REVIEW`, `RETRIED` or `DISCARDED`; the dashboard count and the DLQ list share one helper (`core/jobs/dlq.ts`) and filter `PENDING_REVIEW`; the list shows "Showing 50 of N" when capped. `replayDlq()` marks the entry `RETRIED`. There is no admin screen for discard, notes or replay yet (planned).

## The connector plugin model

A connector is a self-contained folder under `src/connectors/<name>/` with exactly two files that matter to the platform:

- `index.ts` - default-exports an object implementing the `Connector` interface (`core/connectors/types.ts`): a `kind` (`"inbound"` or `"outbound"`), a zod `inputSchema`, and an `execute(ctx)` function.
- `handler.meta.ts` - exports a `meta: ConnectorMeta` object with a `handlerKey`, `direction`, `version`, and optional `description`.

Current connectors: `rest-inbound`, `rest-outbound`, `webhook-inbound`, `file-drop-sftp`.

### Why codegen instead of runtime dynamic import

Vercel's serverless bundler (Node File Trace) needs to see every file a route might load at build time. A runtime `import(\`./connectors/${key}\`)` with a dynamic key is invisible to that analysis and would fail in production even though it works locally. `scripts/generate-connector-manifest.ts` runs at build time (wired as `prebuild` and `predev` in `package.json`) and:

1. Globs `src/connectors/*/` for folders containing both `handler.meta.ts` and `index.ts`.
2. Extracts each folder's `handlerKey` with a lightweight regex (no TypeScript evaluation needed at this stage).
3. Fails the build if two folders declare the same `handlerKey`.
4. Emits `src/connectors/generated/manifest.ts`: a file with genuine static `import` statements, one per connector, assembled into a `Record<string, Connector>`.

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

## The two-database model

Interchange's own schema (23 core tables plus one opt-in table) is the platform's single source of truth for partner registry, credentials, jobs, and audit history. This is always present.

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
- Nothing recurring runs, and nothing recurring is billed, until a project explicitly flips a system to `ACTIVE` and/or arms a schedule.

See `docs/deployment.md` for the exact activation steps, and `docs/security.md` for why this is also a security property, not only a cost-control one.

## Where to look in the code

| Concern | File |
|---|---|
| Inbound route | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts` |
| Inbound auth guard | `core/auth/with-partner-auth.ts` |
| Connector contract | `core/connectors/types.ts` |
| Connector resolution | `core/connectors/resolver.ts` |
| Generated manifest | `src/connectors/generated/manifest.ts` |
| Manifest codegen | `scripts/generate-connector-manifest.ts` |
| Job lifecycle helpers | `core/jobs/lifecycle.ts` |
| Drain / retry / DLQ | `core/jobs/drain.ts` |
| Rate limiting | `core/gateway/rate-limit.ts` |
| Activity log | `core/gateway/activity.ts` |
| Tick (reclaim, schedules, drain) | `src/app/api/jobs/tick/route.ts` |
| Schedule dispatch | `src/app/api/jobs/dispatch/route.ts`, `core/jobs/scheduler.ts` |
| Stuck-job reclaim | `core/jobs/reclaim.ts` |
| QStash signature check | `core/jobs/verify-qstash.ts` |
| QStash arming | `core/jobs/qstash.ts` |
| Credential vault | `core/crypto/vault.ts` |
| Config + credential resolution | `core/config/resolve-config.ts` |
| Operational schema (9 tables) | `db/schema/operational.ts` |
| Config/auth schema (14 tables) | `db/schema/config-auth.ts` |
| Opt-in upstream schema | `db/schema/upstream.ts` |
| DB client | `core/db.ts` |
