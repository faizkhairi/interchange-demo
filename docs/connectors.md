# Connectors

Last updated: 2026-11-13

This document explains how to add a new connector type to Interchange, and how to register a partner instance of an existing type. These are the platform's two levers, and they are deliberately different in weight: one is a code change, the other is a UI action.

## The two-lever model

- **Lever A: add a connector type (developer work).** A connector *type* is a piece of integration logic: "call this kind of REST API", "accept this kind of webhook", "drop a file over SFTP". Adding a new type means writing code: a new folder under `src/connectors/`, implementing the contract described below, then running the manifest generator and redeploying. This is a one-time cost per connector *type*, not per partner.
- **Lever B: register a connector instance (operator work).** Once a type exists in the generated manifest and is catalogued as an `integration_type` row, an operator can register unlimited partner *instances* of it through the admin UI: point a new `external_system` at that `integration_type`, configure its endpoint and credentials, and it is live. No code, no redeploy.

In practice this means: if a project needs to onboard its fifth partner that talks plain REST the same way as the first four, that is Lever B, zero engineering time. If a project needs to talk to a partner using a protocol nothing existing supports, that is Lever A, a bounded and well-defined piece of work.

## The folder convention

Every connector lives at `src/connectors/<name>/` and must contain exactly two files the platform reads:

```
src/connectors/<name>/
  index.ts          # default-exports the Connector implementation
  handler.meta.ts    # exports `meta: ConnectorMeta` (handlerKey, direction, version, description, sample, and for an inbound connector optionally asyncInbound)
```

Anything else in the folder (tests, helper modules, named upstream queries) is free-form; the codegen step only looks for these two filenames.

Current connectors, for reference: `rest-inbound`, `rest-outbound`, `webhook-inbound`, `file-drop-sftp`.

## The Connector interface

Defined in `core/connectors/types.ts`:

```ts
export interface ConnectorContext<TInput = unknown> {
  jobId: string;
  systemCode: string;
  direction: "inbound" | "outbound";
  isTest: boolean;           // the job's test flag; outbound rows you cause inherit it
  requestId: string | null;  // id of the request that created the job (null for scheduled or manual jobs)
  payload: TInput;
  config: ConnectorConfig;   // decrypted credentials + flags + endpoint settings; see below
  logger: JobLogger;         // .debug/.info/.warn/.error, persisted to job_log
  upstream?: UpstreamDataSource; // optional, opt-in upstream read capability
}

export interface ConnectorResult<TOutput = unknown> {
  status: "SUCCESS" | "FAILED" | "RETRY";
  output?: TOutput;
  error?: { code: string; message: string; retryable: boolean };
}

interface BaseConnector<TIn, TOut> {
  inputSchema: ZodType<TIn>;
  execute(ctx: ConnectorContext<TIn>): Promise<ConnectorResult<TOut>>;
}

export interface InboundConnector<TIn = unknown, TOut = unknown>
  extends BaseConnector<TIn, TOut> { kind: "inbound"; }

export interface OutboundConnector<TIn = unknown, TOut = unknown>
  extends BaseConnector<TIn, TOut> { kind: "outbound"; }
```

A few things worth knowing before writing one:

- **Validation is not your job.** The platform validates the body against `connector.inputSchema` before `execute()` is ever called. By the time your `execute()` runs, `ctx.payload` is already the validated, typed shape of your zod schema. You never write manual payload-shape checks.
- **`inputSchema` must be a plain `z.object`, and its top level is strict.** The platform parses with `.strict()` at the two entry points (the inbound gateway and a manual run), so an unknown top-level key is a `422` that names the key (never the value) instead of being silently dropped. Do not declare a catch-all and do not wrap the schema in `z.union`, `z.intersection` or an effect: a schema that is not a plain object fails closed with a `500`, and a registry test asserts that every connector's `inputSchema` is an object. Nested objects keep their own behaviour, so add `.strict()` there if you want the same rule. Payloads already stored are parsed leniently when the drain runs them.
- **You never touch the credential vault.** For outbound connectors, `ctx.config.credentials` already contains decrypted, ready-to-use values (see `core/config/resolve-config.ts`); the encrypted blob and the master key never reach connector code. The map is keyed by each credential's label. An outbound credential's label defaults to its credential type when none is given, so a connector reads `ctx.config.credentials[credentialType]` unless the operator chose another label (rows stored before labels were required were backfilled to their credential type, so existing lookups did not change). A label is unique per system among non-revoked rows: storing a second active credential with the same label is refused with a conflict instead of shadowing the first. `resolveSystemConfig` excludes revoked rows and orders by `created_at`, so the result is deterministic. Rotating an outbound credential re-encrypts it in place under the same label; revoking it removes it from the map.
- **Credentials resolve system first, then type.** `ctx.config.credentials` holds the `api_key`, `oauth2` (a static bearer token) and `basic` (`user:password`) values by label. An `oauth2_client` credential is never in this map: its client secret reaches only the platform's token client. A system's own active row wins; otherwise the active default of the system's integration type is used; otherwise the label is absent. The decrypted rows are cached per instance (300 seconds at most) behind two version counters that every credential write bumps, so a revoke or rotation shows on the next job; a job already running finishes with the value it had. See `docs/security.md`.
- **Flags.** `ctx.config.flags` holds the values of `flag` credentials by label. A flag is a secret on/off switch for a partner: it is never auth, never in `ctx.config.credentials`, and has no origin pin. Read it with `flagEnabled(ctx, label)` from `@core/connectors/flags`, which is true for `true`, `on`, `1` or `yes` (any case, trimmed) and false otherwise, including a missing label. An inbound connector receives `ctx.config.flags` only; its config has no credentials.
- **Auth is injected by the platform.** When the endpoint config sets an auth label (`auth_label`, plus `auth_header` for an `api_key`, default `X-API-Key`), `outboundRequest` adds the header itself, replacing a same-named header you set. It does so only when the request origin equals the `baseUrl` origin and is one of the credential's allowed origins (1 to 10 exact origins); in any other case it sends no auth and logs a fixed warning, so a connector that builds a URL on another host never sends the secret there. The resolved header is a hidden property of `ctx.config`, so you cannot read it, and a call that bypasses `outboundRequest` gets no auth. The `Authorization` form of an `oauth2` or `basic` credential is built by the platform, not by you. An `oauth2_client` credential needs nothing from you either: after the origin check passes, `outboundRequest` fetches (or reuses) the access token and sends it as `Authorization: Bearer`, and `ctx.config.auth` stays hidden. If no token can be had, it returns `OAUTH_TOKEN_UNAVAILABLE` (retryable) or `OAUTH_TOKEN_REJECTED` (final) and nothing is sent to the resource; see `docs/security.md`.
- **Return, do not throw, for expected failures.** `ConnectorResult.status` distinguishes `FAILED` (terminal, will not retry) from `RETRY` (transient, will retry with backoff, up to the job's `maxAttempts`). If your connector *does* throw an ordinary error, the job engine catches it and treats it as retryable by default (`HANDLER_THREW`), matching a "transient until proven otherwise" default. Prefer returning `RETRY` explicitly with a `code`/`message` when you know the failure is transient (a 5xx, a 429, a timeout), and `FAILED` with `retryable: false` when you know it is not (a 4xx other than 429, a validation rejection on the partner's side). When the failure is final but you are deep in a call stack, throw `NonRetryableError(code, message)` from `@core/connectors/errors`: the drain dead-letters the job at once with that code (passed through `toFailureCode`) and schedules no retry.
- **A returned error can carry two more fields.** `retryAfterMs` is the partner's `Retry-After` as a delay (the platform clamps it to one hour; the backoff never waits less than it). `upstream` is a clipped copy of the partner's answer. Both stay inside the platform: the drain masks and stores `upstream` in `job.upstream_error` only, and the inbound route never echoes either to a partner. Keep the `message` fixed text; partner text belongs in `upstream`, never in `message`. Most outbound HTTP connectors do not set these by hand, because `outboundRequest` (`core/net/outbound-client.ts`) does the call, the status classification, the `Retry-After` parse and the clipped body for them. It is also where the endpoint's credential is attached (see "Auth is injected by the platform" above); a bound credential that cannot be sent as a header value returns `CREDENTIAL_INVALID`, not retryable, and nothing goes out.
- **Outbound connectors must use `safeFetch`.** Any request to a URL that comes from configuration goes through `safeFetch(url, init, { timeoutMs, log })` from `@core/net/ssrf`, never the global `fetch`. The `log` option is required (`{ systemCode, jobId, isTest, requestId }`): the connector passes `ctx.systemCode`, `ctx.jobId`, `ctx.isTest` and `ctx.requestId` (a fresh `randomUUID()` when `ctx.requestId` is null), and `safeFetch` writes one outbound `activity_log` row per call, success or failure. The type enforces it, and `npm run audit:log-calls` fails the build for a network call that bypasses the wrappers. It refuses private and reserved addresses (also at connect time, against DNS rebinding), requires https in production, never follows a redirect (a `3xx` is returned to you) and enforces the timeout. Pass `ctx.config.timeoutMs`, which comes from the system's `timeout_ms` (default 10000 ms, 100 to 30000 allowed). `safeFetch` rejects only with an `SsrfError` carrying a fixed `code`; never log or return its upstream text. Log the target as origin and path only (no userinfo, no query string). Read or cancel the response body yourself.
- **Map guard codes to results like this:**

  | `SsrfError.code` | Return | Why |
  |---|---|---|
  | `BLOCKED`, `INVALID_URL`, `DNS` | `FAILED`, `retryable: false` | Configuration problems; a retry cannot fix them. |
  | `DNS_TEMPORARY` | `RETRY`, `retryable: true` | A temporary resolver failure; it is retried with backoff. `DNS` above stays non-retryable. |
  | `TIMEOUT`, `CONNECT`, `TLS` | `RETRY`, `retryable: true` | Transient on the partner side. |

  HTTP statuses map the same way (`classifyHttpStatus` in `core/pure/retry-policy.ts`):

  | Status | Code | Return | Why |
  |---|---|---|---|
  | `3xx` | `REDIRECT` | `FAILED`, `retryable: false` | Redirects are never followed. |
  | `429` | `HTTP_429` | `RETRY`, `retryable: true` | The partner asks for a slower pace; its `Retry-After` is honoured. |
  | other `4xx` (408 included) | `HTTP_4XX` | `FAILED`, `retryable: false` | The partner's answer; a retry would repeat it. |
  | `5xx` | `HTTP_5XX` | `RETRY`, `retryable: true` | Transient on the partner side. |

  `rest-outbound` is the reference: it logs the code only and returns `rest-outbound request failed (<CODE>)` for a guard failure, or `rest-outbound request failed with status <N>` for a 4xx or 5xx, with the clipped body in `upstream`. The ranges and the full rationale are in `docs/security.md`.
- **Logging is free and persisted, and the steps are not yours.** Anything written through `ctx.logger` lands in `job_log` with step `connector`, shown on the job detail page. The platform, not the connector, writes the named steps around `execute()` (`received`, `claimed`, `validated`, `executed` with its duration, `succeeded`, `retry_scheduled`, `dead_lettered`, `failed`), and the `JobLogger` type is unchanged. A job keeps at most 200 rows (`JOB_LOG_MAX_ROWS`): your lines are stored only while the job holds fewer than 150 rows and at most 20 per attempt, and the rest are dropped and counted in one `truncated` row. Since P4a the terminal step (`succeeded`, `retry_scheduled` or `dead_lettered`) is written after the `truncated` row, so the `truncated` row follows every connector row and the terminal step is last. A chatty connector cannot hide the terminal steps. After redaction a message clips at 500 characters and its meta at 4,000.
- **Failure codes are a closed list.** `error.code` is mapped through `toFailureCode` (`core/pure/failure-codes.ts`) and stored as `job.failure_code` and `job_dlq.last_error.code`. Connector codes: `CONFIG`, `ENDPOINT`, `REDIRECT`, `HTTP_4XX`, `HTTP_5XX`, `HTTP_429`, `NETWORK`. Guard codes (the table above): `BLOCKED`, `DNS`, `DNS_TEMPORARY`, `INVALID_URL`, `TIMEOUT`, `CONNECT`, `TLS`. Codes the platform sets itself: `INVALID_PAYLOAD`, `SYSTEM_DELETED`, `HANDLER_THREW`, `HANDLER_FAILED` (a non-success result with no code), `RECLAIMED`, `UNKNOWN_HANDLER` (no connector is registered for the job's handler key; it is dead-lettered at attempt 1, because a retry cannot fix it), `CREDENTIAL_MISSING` (the endpoint's auth label has no active credential on the system or its type) and `CREDENTIAL_INVALID` (a stored credential does not decrypt, or cannot be sent as a header value) and `PAYLOAD_UNAVAILABLE` (the executable payload of an async inbound job is missing, expired or cannot be decrypted; non-retryable, and `execute()` is not called). Both credential codes are non-retryable and dead-letter at attempt 1; replay the job after the credential is fixed. `OAUTH_TOKEN_UNAVAILABLE` (the OAuth token endpoint could not be reached or answered 5xx or 429, or another caller holds the refresh lease) is retryable and honours `Retry-After`; `OAUTH_TOKEN_REJECTED` (the token endpoint refused the client, or its answer was not a usable token) is final. The old codes `HTTP` and `HTTP4xx` are aliases for `HTTP_5XX` and `HTTP_4XX`; any other code becomes `UNKNOWN`. `rest-outbound` emits `HTTP_5XX` and `HTTP_429` (both retryable), `HTTP_4XX` (not retryable; 429 is the one 4xx that retries) and `NETWORK` for an unknown throw.

## handler.meta.ts

```ts
import type { ConnectorMeta } from "@core/connectors/types";

export const meta: ConnectorMeta = {
  handlerKey: "rest-inbound",
  direction: "inbound",
  version: 1,
  description:
    "Generic REST inbound connector: accepts a JSON payload from a partner, " +
    "validates it, and records it as a job. A template for real partners.",
  sample: {
    reference: "PO-1001",
    data: { customer: "Example Customer", total: 125.5 },
  },
  // Inbound only: the author's promise that execute() is idempotent.
  asyncInbound: true,
};
```

`handlerKey` is the string that ties everything together: it is what the generated manifest keys the connector under, and it is what an `integration_type` row (see below) points at. It must be unique across every connector folder; the codegen step fails the build if two folders declare the same key.

`sample` is required. It is a fictional example input that passes the connector's `inputSchema`, and the Catalog page shows it next to the schema. Use made-up data only (`PO-1001`, `example.test` hosts, no real names or secrets). The codegen step parses it with the same strict rule as the inbound gateway (`parseConnectorInput`, which refuses unknown keys) and fails the build if it does not pass. The `sample` of an outbound connector can reuse the outbound payload examples; see `src/connectors/rest-outbound/handler.meta.ts`.

`entityPath` is optional. It is a dot path into the job payload (`rest-inbound` declares `reference`, `webhook-inbound` declares `payload.orderId`). When the value found there is a string or a number, the platform stores it as the job's `entity`, redacted and clipped to 128 characters, and the Jobs list shows it. It is display only: there is no filter on it yet. The catalog carries the path (a name, never data), and `npm run generate:connectors` regenerates it.

`asyncInbound` is optional, inbound only, and can only be `true`. It is your promise that `execute()` is idempotent: a partner may send `Prefer: respond-async` with an `Idempotency-Key`, and for a connector that sets the flag the gateway answers 202 Accepted and runs the job right after the response instead of inline. A reclaimed async job (one stuck in `PROCESSING` past `JOB_STUCK_MINUTES`) runs again from its stored payload, so delivery is at-least-once and a repeat run must be harmless: write with a stable key (the job id is available as `ctx.jobId`) or tolerate a duplicate. Both inbound templates set it because they only echo. Leave it out when a repeat run is not safe: the call then always runs inline, and a stuck inline job is failed, never re-run. The codegen step fails the build if an outbound connector sets it, or if it is set to anything but `true`. The catalog carries it, and the Catalog page shows an "Accepts async (202)" chip. An async job's payload is held encrypted for at most 24 hours (`docs/security.md`); `ctx.payload` is the full original payload, as for an inline call.

## The file-drop-sftp payload (P4d)

`file-drop-sftp` is an outbound connector whose job payload carries the file itself: `{filename, contentBase64}`. There is no upload endpoint. The Run drawer of a `file-drop-sftp` system offers "File to send" in place of the JSON box, and the file goes through the existing `jobs.run` (a schedule takes the same payload). No new permission, table, environment key, dependency or migration was needed.

The rules live in one browser-safe schema, `core/pure/file-drop-sftp-payload.ts`, which the connector, the real job and schedule services and the demo engine all import, so every backend accepts and refuses the same files:

- **Size:** the decoded file is 1 to 46,080 bytes (45 KiB). The cap keeps the request under the 64 KB admin body cap once base64 and the JSON wrapper are added. An empty file is refused.
- **Content:** strict base64 (standard alphabet, correct `=` padding, a multiple of 4 characters, no whitespace).
- **Filename:** 1 to 255 characters, no `/`, no `\`, no control characters, and not `.` or `..`.

The guards sit on the two fields, not on the object, because the gateway needs a plain object schema to check for unknown keys. The Run drawer reads the chosen file, shows its name, size and SHA-256, and refuses a bad file with the schema's own message before any request; the server check is still the gate. The connector is a no-network template stub: it logs `would upload <filename> (<n> bytes) via SFTP` and reports success. Larger files are a later phase. Object storage (P5f) is managed from a system's Storage tab and is not used by this connector.

## The catalog

Every connector appears on the console's Catalog page (route slug `connectors`, endpoint `GET /connectors/catalog`, permission `systems:read`). Each entry shows the `handlerKey`, direction, version, description, the connector's input as a JSON Schema (strict: unknown keys are refused) and the `sample`, each with a copy button. The data comes from `src/connectors/generated/catalog.ts`, which the codegen step writes from your `index.ts` (the zod `inputSchema`) and `handler.meta.ts` (the `sample`), so the documented schema cannot drift from the one the gateway enforces. The file is committed, and a test fails if it is stale. The JSON Schema is produced at build time by zod's own `z.toJSONSchema`.

To add a sample for a new connector, add `sample` to its `handler.meta.ts`, run `npm run generate:connectors`, and commit the regenerated `catalog.ts`. If the sample is wrong the command fails and names the connector and the offending field.

## Step by step: adding a new connector type

1. **Create the folder.** `src/connectors/my-partner-format/`
2. **Write `index.ts`** implementing `Connector` (see the minimal example below).
3. **Write `handler.meta.ts`** with a unique `handlerKey` and a `sample`.
4. **Generate the manifest:**

   ```
   npm run generate:connectors
   ```

   This runs `scripts/generate-connector-manifest.ts`, which globs every folder under `src/connectors/` containing both required files, extracts each `handlerKey`, and writes `src/connectors/generated/manifest.ts` with real static `import` statements (not a runtime dynamic import, which would not survive Vercel's bundler). It then writes `src/connectors/generated/catalog.ts` (see "The catalog") and fails if your `sample` does not pass your `inputSchema`. The script also runs automatically as `predev` and `prebuild`, so a normal `npm run dev` or deploy regenerates it; running it manually is mainly useful to confirm the new connector was picked up before committing.

5. **Commit the generated manifest and catalog along with your connector.** They are checked into the repo (not gitignored), so a code review can see exactly which handlers a deploy will resolve.
6. **Register an `integration_type` row** with `handler_key` matching your connector's key, and the correct `direction`. This is the catalog entry that lets an `external_system` reference your new connector without knowing anything about the code.
7. **Register the system in the admin UI** (Lever B, from here on): create an `external_system` pointing at that `integration_type`, set its endpoint config and credentials if it is outbound, issue it an API key, bearer token or Basic credential from its detail page if it is inbound, and flip its status to `ACTIVE` when it is ready for live traffic (see `docs/deployment.md`).

## Minimal connector example

Based on the real `rest-inbound` connector (`src/connectors/rest-inbound/index.ts`), trimmed to show the shape:

```ts
// src/connectors/rest-inbound/index.ts
import { z } from "zod";
import type {
  InboundConnector,
  ConnectorContext,
  ConnectorResult,
} from "@core/connectors/types";

const inputSchema = z.object({
  reference: z.string().min(1),
  data: z.record(z.string(), z.unknown()),
});

type Input = z.infer<typeof inputSchema>;

const connector: InboundConnector<Input, { received: string }> = {
  kind: "inbound",
  inputSchema,
  async execute(
    ctx: ConnectorContext<Input>,
  ): Promise<ConnectorResult<{ received: string }>> {
    ctx.logger.info("rest-inbound received payload", {
      reference: ctx.payload.reference,
      keys: Object.keys(ctx.payload.data),
    });
    return {
      status: "SUCCESS",
      output: { received: ctx.payload.reference },
    };
  },
};

export default connector;
```

An outbound connector adds the required `log` context to its `safeFetch` call (trimmed from `src/connectors/rest-outbound/index.ts`):

```ts
import { randomUUID } from "node:crypto";
import { safeFetch } from "@core/net/ssrf";

const res = await safeFetch(
  url,
  { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(ctx.payload.body) },
  {
    timeoutMs,
    log: {
      systemCode: ctx.systemCode,
      jobId: ctx.jobId,
      isTest: ctx.isTest,
      requestId: ctx.requestId ?? randomUUID(),
    },
  },
);
```

```ts
// src/connectors/rest-inbound/handler.meta.ts
import type { ConnectorMeta } from "@core/connectors/types";

export const meta: ConnectorMeta = {
  handlerKey: "rest-inbound",
  direction: "inbound",
  version: 1,
  description:
    "Generic REST inbound connector: accepts a JSON payload from a partner, " +
    "validates it, and records it as a job. A template for real partners.",
  sample: {
    reference: "PO-1001",
    data: { customer: "Example Customer", total: 125.5 },
  },
  asyncInbound: true,
};
```

This is intentionally a template: it accepts almost anything (`reference` plus a free-form `data` record) and only logs and echoes back. A real partner-specific connector narrows `inputSchema` to that partner's actual contract and does real work in `execute()`: writing to a project's own system, transforming the payload, calling out to another service, and so on.

For an outbound example that shows retry classification (mapping HTTP status codes and guard codes to `RETRY` vs `FAILED`, through `safeFetch` with the system's `timeout_ms`), see `core/net/outbound-client.ts` (the shared base `rest-outbound` calls: `outboundRequest` makes the guarded call and maps status codes, guard codes and `Retry-After` to a result) and `src/connectors/rest-outbound/index.ts`. For a connector that reads its own settings and credentials out of `ctx.config`, see the same file's use of `ctx.config.baseUrl`.

## Optional: reading from an upstream data source

If a connector needs to read from a project's own upstream system (a legacy database, an internal API), it can accept `ctx.upstream` and issue its own named queries against it. This is opt-in per `docs/architecture.md`'s two-database model: core never defines what queries exist, only the connection and credential plumbing (`upstream_data_source` in `db/schema/upstream.ts`). A connector that needs this ships its own named queries in its own folder; nothing in core assumes any particular upstream shape.
