# Architecture

Last updated: 2026-07-07

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
  |                             | 1. authenticatePartner(headers, code)|
  |                             |    - extract credential from         |
  |                             |      x-api-key or Authorization      |
  |                             |    - look up external_system by      |
  |                             |      systemCode from the URL         |
  |                             |    - require status = ACTIVE         |
  |                             |    - verify against system_api_key   |
  |                             |      or system_bearer_token          |
  |                             |      (constant-time compare)         |
  |                             |------------------------------------->|
  |                             |<-------------------------------------|
  |         401 / 403           |  (unknown system, inactive, bad cred)|
  |<-----------------------------                                      |
  |                             |                                      |
  |                             | 2. resolve the system's              |
  |                             |    integration_type -> handlerKey    |
  |                             | 3. resolveHandler(handlerKey)        |
  |                             |    -> static lookup in the           |
  |                             |       generated connector manifest   |
  |                             |                                      |
  |                             | 4. connector.inputSchema.safeParse() |
  |         422                 |    (payload validation failure)      |
  |<-----------------------------                                      |
  |                             |                                      |
  |                             | 5. createJob(...)  -> INSERT job     |
  |                             |    (status=PROCESSING)               |
  |                             |------------------------------------->|
  |                             | 6. connector.execute(ctx) runs INLINE|
  |                             |    ctx.logger writes to job_log      |
  |                             |------------------------------------->|
  |                             | 7. markSuccess / markFailed          |
  |                             |    -> UPDATE job.status              |
  |                             |------------------------------------->|
  |    200 (or 502 / 500)       |                                      |
  |<-----------------------------                                      |
```

Source: `src/app/api/inbound/[systemCode]/[endpoint]/route.ts`, `core/auth/with-partner-auth.ts`, `core/connectors/resolver.ts`, `core/jobs/lifecycle.ts`.

Two properties worth calling out:

- **systemCode confinement.** The auth guard resolves the partner system named in the URL first, then checks whether the presented credential belongs to that system. A valid credential for system A can never authenticate a call to system B's routes, even if both are ACTIVE.
- **Fail-fast on an unknown handler.** If `integration_type.handler_key` does not exist in the generated manifest, `resolveHandler()` throws `UnresolvedHandlerError` immediately rather than queuing a job that will never succeed.

## Outbound and scheduled job lifecycle

Outbound work (calling out to a partner, or running a recurring task) does not run inline. It is written to the `job` table and picked up by a drain.

```
                       +-------------------+
   createJob(...)  --> |  job              |
   (status=PENDING)    |  status=PENDING   |
                       +-------------------+
                                |
                                | /api/jobs/process  (POST)
                                | called by an armed QStash
                                | schedule, OR manually from
                                | the admin "Run now" action
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

Source: `core/jobs/drain.ts`, `core/jobs/lifecycle.ts`, `db/schema/operational.ts`.

Key mechanics:

- **The queue is not a separate table.** It is `job` rows with `status IN ('PENDING', 'RETRY')` whose `next_attempt_at` is due. This is a deliberate simplification: one source of truth for both "what ran" and "what's next".
- **Claiming is race-safe without Redis.** `FOR UPDATE SKIP LOCKED` inside a transaction lets multiple concurrent invocations of `/api/jobs/process` (which can happen on serverless, since nothing guarantees single-flight) each claim a disjoint batch.
- **Backoff is exponential**, based on the attempt count: `BACKOFF_BASE_MS * 2^(attempts - 1)`.
- **The DLQ (`job_dlq`) is a queryable ledger of terminal failures**, each pointing back at the original `job.id`, with the reason (`retries exhausted` or `non-retryable failure`) and the last error payload. `replayDlq(jobId)` resets a DLQ'd job to `PENDING` with `attempts` cleared, so it re-enters the drain on the next pass.
- **Schedules (`job_schedule`) are dispatched, not drained, by a separate route**: `/api/jobs/dispatch` enqueues one outbound job per `enabled` schedule row and stamps `lastRunAt`. It is armed the same way as the drain (see Dormant posture below).

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

Interchange's own schema (20 core tables plus one opt-in table) is the platform's single source of truth for partner registry, credentials, jobs, and audit history. This is always present.

Separately, a connector may need to read from a partner's or a project's own upstream system: a legacy database, an internal REST API, whatever the integration needs. Interchange models this as `upstream_data_source` (`db/schema/upstream.ts`), explicitly marked as **opt-in and not part of the core 20 tables**:

- The core platform provides only the connection and credential plumbing (`connection_config`, encrypted credential fields, a `kind` enum of `postgres | mysql | rest | odbc_http_bridge`, and a `read_only` flag defaulting to `true`).
- Connectors that need upstream reads bring their own named queries, living in their own connector folder, never in core. Core does not know or care what SQL or API calls a connector issues against its upstream source.
- **The platform runs fully with zero upstream sources configured.** A deployment with no `upstream_data_source` rows is a complete, working Interchange instance; upstream integration is additive, not load-bearing.

This keeps the platform vendor-neutral: nothing in `core/` or the base schema assumes any particular partner's database shape.

## Dormant posture

A fresh deploy is fully built but inert by design:

- Every `external_system` row defaults to `status = 'SETUP'` (see `db/schema/config-auth.ts`). The inbound auth guard rejects all traffic for a system that is not `ACTIVE`, regardless of whether valid credentials exist for it.
- `job_schedule` rows default to `enabled = false` and carry no `qstash_schedule_id` until explicitly armed.
- `core/jobs/qstash.ts`'s `armSchedule()` is a documented no-op when `QSTASH_TOKEN` is unset: it returns `{ armed: false, reason: "QSTASH_TOKEN not set (dormant)" }` rather than throwing, so the platform is fully functional in a dormant deploy with no external scheduler configured.
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
| Schedule dispatch | `src/app/api/jobs/dispatch/route.ts` |
| QStash arming | `core/jobs/qstash.ts` |
| Credential vault | `core/crypto/vault.ts` |
| Config + credential resolution | `core/config/resolve-config.ts` |
| Operational schema (8 tables) | `db/schema/operational.ts` |
| Config/auth schema (12 tables) | `db/schema/config-auth.ts` |
| Opt-in upstream schema | `db/schema/upstream.ts` |
| DB client | `core/db.ts` |
