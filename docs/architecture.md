# Architecture

Last updated: 2026-10-11

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

In order, the inbound chain is: request id, per-IP limit, partner auth, service token (by policy), per-system limit, endpoint allowlist and limit, handler, `Idempotency-Key`, JSON, strict parse, job. A cross-origin browser preflight (`OPTIONS`) takes a separate, shorter path: per-IP limit, then the per-system CORS decision (see `docs/security.md`).

Source: `src/app/api/inbound/[systemCode]/[endpoint]/route.ts`, `core/auth/with-partner-auth.ts`, `core/auth/service-token.ts`, `core/pure/endpoint-name.ts`, `core/pure/connector-input.ts`, `core/gateway/rate-limit.ts`, `core/gateway/log-call.ts`, `core/pure/redact.ts`, `core/connectors/resolver.ts`, `core/jobs/lifecycle.ts`.

Properties worth calling out:

- **systemCode confinement.** The auth guard resolves the partner system named in the URL first, then identifies the presented credential by its lookup index across all systems. A valid credential for system A can never authenticate a call to system B's routes, even if both are ACTIVE: it is refused with a generic `403`, and the owning system is recorded only in the activity row. Revoked and expired credentials get `401`, and so does a credential whose owning system is deleted. A deleted system is an unknown system at the gateway. See `docs/security.md`.
- **Rate limiting is Postgres-backed and fails open.** Each check is one atomic `INSERT ... ON CONFLICT (key, window_start) DO UPDATE SET count = count + 1 RETURNING count` against `rate_limit_bucket` (one-minute fixed windows). Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; a rejected call gets 429 with `Retry-After`. If the database is unreachable the limiter logs a warning and lets the request through, because a limiter outage must not become a gateway outage. The client IP is taken from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, and must parse as an IP address (anything else shares an `unknown` bucket); on a host that does not overwrite `x-forwarded-for` that last fallback is client-controlled, so put a trusted proxy in front. The per-IP limit (`RATE_LIMIT_IP_RPM`, default 250 per minute) is checked before authentication, which throttles credential guessing; the per-system limit (`external_system.rate_limit_rpm`) is checked after, so strangers cannot burn a partner's quota. A system can also declare per-endpoint limits (bucket `sys:<code>:<endpoint>`); the response headers report the tightest bucket consumed. The "Rate limits" page shows the current window's gateway buckets and the last hour of 429s, with client IPs truncated to /24 and /48.
- **Idempotent retries.** A repeated `Idempotency-Key` from the same system returns the original job's result with `Idempotent-Replayed: true` instead of creating a second job or failing. The same key with a different payload is a 422; a duplicate that arrives while the original is still running is a 409. The key is trimmed (an empty key counts as absent), at most 200 printable ASCII characters (else 400), and stored with an `in:` prefix so a partner key can never collide with a scheduler key. Only true replays carry `Idempotent-Replayed: true`; the 422 does not.
- **Every call is logged.** `activity_log` gets one row per call the platform receives or makes, written by `logCall` (`core/gateway/log-call.ts`). `direction`, `isTest`, `request` and `response` are required by the TypeScript type, so a call site cannot forget one (use `{}` or `{ reason }` when there is no body). Inbound logging is total: every exit of the route is a `finish(...)` call and the handler body runs inside a try whose catch also finishes (a 500), so auth failures, rate limits, validation errors and unexpected throws each write a row. A throw in `await params` before the try is a framework fault and sits outside the audit's Rule D scope. Outbound rows come from the three network wrappers: `safeFetch` (`core/net/ssrf.ts`), `probeSftp` (`core/net/sftp-probe.ts`) and `qstashFetch` (`core/jobs/qstash.ts`). Each takes a required `OutboundLog` context (`systemCode`, `jobId`, `isTest`, `requestId`); health checks, SFTP tests and QStash calls are written with `isTest: true`, and a call that fails before any HTTP response logs response code 0 and a fixed error code. `logCall` never throws: a failed write is a warning, so the call being logged is never broken by its own bookkeeping. `scripts/audit-log-calls.ts` (a TypeScript AST audit, `npm run audit:log-calls`, run in CI) enforces this with three rules. Rule A: a network call sits in a function that calls `logCall`, or in a private helper whose every in-module caller does. Rule C: `ssh2`, `undici`, `tls`, `http` and `https`, and the connecting exports of `net`, are imported only in the wrapper files. Rule D: each exported non-`OPTIONS` inbound handler is declarations, then a try whose catch returns `finish(...)`, and every return is `finish(...)`, which calls `logCall`. Redaction runs before storage and clips run after it, so a stored preview never holds a secret. Key rules mask the value of a secret-looking key at any depth; value patterns replace secret-shaped spans inside any string (Bearer and the other Authorization schemes, Basic, JWT, PEM private keys, `keyword=value` pairs, URL userinfo, Luhn-valid card numbers); strings clip at 2,000 characters and a payload at 16,000 (a longer one is stored as `{truncated, originalChars, preview}`); the stored endpoint is masked and capped at 256 characters. Credentials and `Authorization` headers are never stored. The exact rules are in `docs/security.md`. Errors on the inbound catch, `logCall` and prune paths go through `safeErrorLog` (the error name, a SQLSTATE and a short message cut before any bound parameters, never the error object). The inbound row is written after the response is sent (`after()` on the Vercel runtime, inline under test) so logging adds no latency. Retention is controlled by the `app_setting` row `retention` (`{enabled, activityDays}`): `enabled: false` is the kill switch for every automatic delete, and with no row `ACTIVITY_RETENTION_DAYS` (default 30) applies. Two triggers prune in bounded batches: the tick (5,000 rows x 10 batches per run) and a throttled `after()` trigger on inbound traffic (`maybeRunRetention`, 1,000 x 5, at most once per ten minutes across instances through a compare-and-set on the `retention_last_run` row; a stored stamp in the future counts as stale). A row is never deleted while its job has an open (`PENDING_REVIEW`) DLQ item. This table is separate from `admin_audit_log`, which records admin actions. A replay of an inbound call (same `Idempotency-Key`) returns the stored `job.result`, which was redacted and clipped when it was written, so a replay returns the redacted, clipped output and not the original body (Deviation 2, by design).
- **Fail-fast on an unknown handler.** If `integration_type.handler_key` does not exist in the generated manifest, `resolveHandler()` throws `UnresolvedHandlerError` immediately rather than queuing a job that will never succeed.
- **Service token by policy.** Each integration type has an `auth_policy`: `partner` (default) or `partner+service`. Under `partner+service` the request must also carry an `X-Service-Token` JWT (HS256, `aud` `interchange-inbound`, lifetime at most 900 s). It is checked after the partner credential, answers 503 when `SERVICE_JWT_SECRET` is missing (fail closed), and is never accepted as a partner credential. `PartnerIdentity` carries the policy, handler key, direction and CORS origins from one join with `integration_type`, so the route no longer selects the type itself.
- **Endpoint allowlist.** The `[endpoint]` segment is normalised (NFC, lower-case, `^[a-z0-9._-]{1,64}$`). Declaring any endpoint limit turns the list into the system's endpoint allowlist: other names are 404 "unknown endpoint", after authentication.
- **Strict payloads.** The top level of every inbound payload and every manual `jobs.run` payload is parsed strict. Unknown keys give a 422 that names the keys and never echoes a value. The drain parses stored payloads leniently, so older rows are not dead-lettered by the new rule.
- **Soft delete and suspend.** A deleted system is an unknown system. Suspending or deleting a system stops its schedules with no catch-up fire, holds a suspended system's queued jobs, and fails a deleted system's queued jobs without retry (see "Outbound and scheduled job lifecycle"). Deleted system and integration type codes stay reserved, so a code is never reused for a different partner.

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
- **The DLQ (`job_dlq`) is a queryable ledger of terminal failures**, each pointing back at the original `job.id`, with the reason (`retries exhausted` or `non-retryable failure`) and the last error payload. `replayDlq(jobId)` resets a DLQ'd job to `PENDING` with `attempts` cleared, so it re-enters the drain on the next pass. The console calls it through `POST /api/admin/dlq/:jobId/replay`, which requires `jobs:retry` (a job that is no longer replayable returns `CONFLICT`). A manual run is `POST /api/admin/jobs`, which requires `jobs:run`.
- **Schedules (`job_schedule`) fire by cron, exactly once.** Each enabled schedule has a `next_run_at`. A tick claims a due schedule with a compare-and-swap (`UPDATE ... WHERE id = ? AND next_run_at = ? RETURNING`), so two overlapping ticks cannot both fire it, then enqueues one PENDING outbound job and computes the next `next_run_at` from the cron expression (croner, UTC). The claim and the job insert share one transaction, so a failed insert rolls the claim back and the next tick retries the same slot. A schedule registers nothing with QStash: its cadence is its own cron, evaluated by the tick, so it can run no more often than the tick. `/api/jobs/dispatch` does only this enqueue step, and only for due schedules.
- **`/api/jobs/tick` is the single background driver.** One signed QStash call per cadence runs, in order: stuck-job reclaim, due-schedule dispatch, a bounded `drainBatch()`, and a prune of expired rate-limit buckets and old `activity_log` rows (the activity prune reads the `retention` setting: a disabled kill switch makes it delete nothing, and rows still referenced by an open DLQ item are kept; with the tick dormant, the throttled after-response trigger prunes only after inbound traffic). Each stage has its own try/catch, so one failure cannot skip the others; if any stage failed the response is 500 with every stage's result. There is exactly one QStash schedule, cadence `QSTASH_TICK_CRON` (default `*/5 * * * *`), created by the explicit "Arm tick" action on the Schedules page (`armTick()`, id stored in `app_setting`) and removed by "Disarm tick". It is dormant until armed.
- **Stuck jobs are reclaimed.** A job left `PROCESSING` with `locked_at` older than `JOB_STUCK_MINUTES` (default 10) lost its worker. An outbound job returns to `RETRY` with backoff while attempts remain, otherwise goes to the DLQ through the same path as a normal failure; this is at-least-once delivery, so outbound handlers and the partners they call must be idempotent. An inbound job is never re-executed: it goes to `FAILED`, and the partner's retry with the same `Idempotency-Key` receives that recorded failure.
- **Suspended and deleted systems do no background work.** The scheduler joins `external_system` on `system_code` and fires a schedule only for a live, `ACTIVE` system. Any other due schedule has its `next_run_at` advanced to the next cron slot with nothing enqueued (counted in `skipped`), so activating or restoring a system later never produces a catch-up fire. The drain's claim query excludes jobs of live, non-`ACTIVE` systems: a `SUSPENDED` or `SETUP` system's jobs are held (still pending, attempts unchanged, no batch slot taken from other systems) until the system is `ACTIVE`. A deleted system's job fails non-retryable ("system deleted") before the connector runs and lands in the DLQ. Restore leaves a system `SUSPENDED`, so held work waits for an explicit activation. This changed in P2b: before it, a suspended system's schedules fired and its queued jobs ran. A schedule whose `system_code` matches no system row at all is not gated.
- **QStash calls are signature-verified.** `/api/jobs/tick`, `/dispatch` and `/process` verify the `Upstash-Signature` header against the raw body with `QSTASH_CURRENT_SIGNING_KEY` / `QSTASH_NEXT_SIGNING_KEY`. The expected url is built from `APP_URL` (else `NEXTAUTH_URL`) plus the request path and passed to the verifier, so a signature minted for one endpoint cannot be replayed against another. With the keys unset they fail closed (503). The only bypass is an explicit `QSTASH_DEV_BYPASS=1`, honoured outside production only.
- **DLQ review state.** `job_dlq.status` is `PENDING_REVIEW`, `RETRIED` or `DISCARDED`; the dashboard count and the DLQ list share one helper (`core/jobs/dlq.ts`) and filter `PENDING_REVIEW`; the list pages by `offset` in pages of 50 ("Rows A to B of N" with Previous and Next), like the jobs list. `replayDlq()` marks the entry `RETRIED`. The Jobs page has a DLQ view with a replay action; discard and notes screens do not exist yet (planned).

## The connector plugin model

A connector is a self-contained folder under `src/connectors/<name>/` with exactly two files that matter to the platform:

- `index.ts` - default-exports an object implementing the `Connector` interface (`core/connectors/types.ts`): a `kind` (`"inbound"` or `"outbound"`), a zod `inputSchema`, and an `execute(ctx)` function.
- `handler.meta.ts` - exports a `meta: ConnectorMeta` object with a `handlerKey`, `direction`, `version`, and optional `description`.

Current connectors: `rest-inbound`, `rest-outbound`, `webhook-inbound`, `file-drop-sftp`.

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

Interchange's own schema (25 core tables plus one opt-in table) is the platform's single source of truth for partner registry, credentials, jobs, and audit history. This is always present.

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

## Admin API and demo

The admin console is one client UI (`src/features/*`, routed by `src/features/shell/ConsoleShell.tsx` from the table in `src/features/routes.ts`) over a single typed interface, `AdminApi` (`src/admin-api/contract.ts`). The UI never knows which implementation it talks to; that is what lets the live console and the public demo share every screen.

- **`HttpAdminApi`** (`src/admin-api/http/client.ts`) calls `/api/admin/*`. On the server, one catch-all route (`src/app/api/admin/[...path]/route.ts`) matches method and path against the endpoint registry (`src/admin-api/endpoints/`). Each endpoint declares its id, method, path, permission, zod input, output and audit action, so there are no per-endpoint route files. An unknown path is 404 and a known path with the wrong method is 405.
- **`DemoAdminApi`** (`src/admin-api/demo/`) runs entirely in the browser over an in-memory engine seeded with fictional data. It makes no `/api` calls, enforces the same permissions per role and returns the same error codes, so the demo behaves like the real console. It lives at `/demo`, which links into `/demo/console?as=admin|operator|viewer`; inside the console a "view as" switcher changes role and a reset button restores the seed.
- **Every `/api/admin` request runs through `withAdmin`** (`core/http/with-admin.ts`): request id, per-IP rate limit (before the CSRF guard and any session lookup), CSRF guard and JSON-only body on non-GET requests, `requirePermission` against the database before input is parsed, per-user rate limit (after authorization), a 64 KB body cap, zod validation, then the error envelope `{error:{code,message,requestId,issues?}}`. Over either limit the answer is 429 `RATE_LIMITED` with `Retry-After`; the limiter fails open. Responses carry `Cache-Control: no-store`. See `docs/security.md` for the full order and reasoning.
- **Param routes and the system detail page.** A route entry in `src/features/routes.ts` may declare a `pattern` such as `systems/:id`. `findRoute` matches it segment by segment (a deeper or shallower path is unknown) and returns the route with its params, which the console router passes to the page. `systems/:id` has no navigation entry and requires `systems:read`; `expiring-credentials` is a second navigation-less drill-down. The navigation has nine items: the original seven plus Rate limits and Activity. Each system code in the Systems list links to the detail page. The page (`?tab=overview|credentials|endpoints|cors|health|certificates|sftp`) shows credential counts (Active, Rotating, Expired, Revoked) that link to the Credentials tab filtered by that status, and the Credentials tab issues, rotates, revokes and (when enabled) reveals credentials. A card count and its list total come from the same `credentials.list` call, so they cannot drift.
- **The registry has 54 endpoints.** P3a added 6 (48 before; see "Activity log and retention (P3a)" below). P2c added 7 (41 before; see "Certificates, SFTP and the catalog" below). P2b added 14 (27 before) and extended three. New: `systems.update` (`POST /systems/:id/settings`, which avoids shadowing `POST /systems/import`), `systems.delete`, `systems.restore`, `systems.duplicate`, `systems.export`, `systems.import`, `systems.checkHealth`, `endpointConfig.get`, `endpointConfig.save`, `integrationTypes.setAuthPolicy`, `integrationTypes.delete`, `integrationTypes.restore`, `rateLimits.monitor` (`GET /rate-limits`) and `credentials.expiring` (`GET /credentials/expiring`). Extended: `systems.list` (`view`, `health` filters), `integrationTypes.list` (`view`) and `integrationTypes.create` (`authPolicy`). A registry test asserts that no two same-method endpoints can match one concrete path.
- **Screens added in P2b.** The Systems list gains a Deleted tab (with Restore), Duplicate, Export and Import. The system detail page gains Endpoints (config and limits), CORS and Health tabs, and a Delete action. The Catalog page (the route slug stays `connectors`) shows each type's auth policy to everyone and can delete and restore types; only an admin (permission `auth-policy:write`) can change a policy, after a confirmation. The Rate limits page (nav) and the Expiring credentials page (drill-down) are new. The dashboard gets a Gateway section with four cards (deleted systems, health failing, rate-limited in the last hour, credentials expiring in 30 days); each count comes from the helper behind its destination list, so a card and the list it opens cannot disagree.
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
- **One contract suite, two implementations.** `test/contract/admin-api.contract.ts` defines the behaviour (counts match filtered lists, validation before side effects, viewers refused on every mutation, duplicate codes conflict) and runs against both `HttpAdminApi` (`test/contract/http.test.ts`) and `DemoAdminApi` (`test/contract/demo.test.ts`). A registry test checks that every endpoint has a permission and every mutation has an audit action.
- **Capability flag.** `AdminApi.capabilities` lets an implementation say what it supports. Notifications exist in the demo only: `DemoAdminApi` reports `notifications: true`, `HttpAdminApi` reports `false` until a later phase adds a notifications table, and the bell and Notifications page render only when the flag is set.
- **Auth boundary.** The admin layout (`src/app/admin/layout.tsx`) still performs the database-backed `requireActiveUser()` check and redirects to `/login`; the API is the gate on every call.

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
| Drain / retry / DLQ | `core/jobs/drain.ts` |
| Rate limiting | `core/gateway/rate-limit.ts` |
| Activity log | `core/gateway/log-call.ts`, `core/pure/redact.ts`, `core/pure/csv.ts` |
| Activity service, retention prune | `core/services/activity.ts`, `core/jobs/prune.ts` |
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
| Tick (reclaim, schedules, drain) | `src/app/api/jobs/tick/route.ts` |
| Schedule dispatch | `src/app/api/jobs/dispatch/route.ts`, `core/jobs/scheduler.ts` |
| Stuck-job reclaim | `core/jobs/reclaim.ts` |
| QStash signature check | `core/jobs/verify-qstash.ts` |
| QStash arming | `core/jobs/qstash.ts` |
| Credential vault | `core/crypto/vault.ts` |
| Config + credential resolution | `core/config/resolve-config.ts` |
| Operational schema (9 tables) | `db/schema/operational.ts` |
| Config/auth schema (16 tables) | `db/schema/config-auth.ts` |
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
