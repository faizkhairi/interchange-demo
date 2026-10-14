# Interchange

**Where every system meets and routes.**

Interchange is a vendor-neutral integration platform and API gateway. It is the single governed junction for all your external systems: partners never touch your core, and every request passes through one controlled point where it is authenticated, validated, rate-limited, and logged.

## Live demo

Try the fully interactive demo. Log in with one click and use the whole console: register connectors, activate systems, run jobs and watch them flow through the lifecycle, receive notifications, browse the audit trail.

**[Open the live demo](https://interchange-livid.vercel.app/demo)**

Everything you do in the demo lives only in your browser. Refresh the page and it resets to a clean state. The demo makes zero database calls, so it stays pristine for every visitor.

## What it does

- **Pluggable connectors.** A connector is a self-contained module with a small `execute` contract. Adding a connector type is a code change plus a rebuild; registering unlimited partner instances of an existing type is a pure UI action, with no redeploy.
- **Config-driven, not hardcoded.** Systems, integration types, credentials, endpoints, and schedules live in the database. Outbound credentials are encrypted at rest with AES-256-GCM; inbound API keys, bearer tokens and Basic credentials are hashed. Everything is managed through the admin console.
- **One governed inbound gateway.** External partners authenticate with an API key, bearer token or Basic credentials and are confined to their own system's routes. A partner cannot reach another partner's endpoints. Every request is rate-limited and logged.
- **Provisioned-dormant by default.** A fresh deploy ships fully built but idle. Systems start in a setup state, no schedules are armed, notifications are off. Nothing runs, and nothing consumes recurring hosting quota, until a project explicitly activates it.
- **Serverless-native job engine.** Inbound work runs inline. Outbound and scheduled work drains from a Postgres-backed queue with retry, backoff, and a dead-letter queue that supports replay. No Redis, no long-running worker.

## How a request flows

```
partner request
   -> inbound gateway rate-limits by IP, then authenticates the caller (API key / bearer / basic)
      and confirms the URL system code matches the caller (no cross-system access)
   -> if the system's integration type requires it, a short-lived service token must be presented as well
   -> the per-system rate limit (and any per-endpoint limit) is applied; the system must be ACTIVE (a dormant system rejects traffic)
   -> the bound connector validates the payload against its schema
   -> the connector runs; a job row records the outcome
   -> every outcome (including rejections) is logged; outbound/scheduled work drains from a queue with retry + DLQ
```

## What exists today, and what is planned

Built and working in the real application:

- Inbound gateway with API key, bearer and Basic authentication, per-system confinement (a credential presented on another system's route is refused with 403 and logged), per-IP and per-system rate limiting (`X-RateLimit-*`, `Retry-After`, fail-open), request ids, idempotent retries, and an activity log row for every outcome. Also: an optional short-lived service token as a second credential per integration type, per-endpoint limits that double as an endpoint allowlist, strict payloads (unknown keys are refused by name), and per-system CORS origin lists.
- Credential lifecycle: issue API keys, bearer tokens and Basic credentials (optional label and expiry), rotate with an optional grace window, revoke, and list with Active, Rotating, Expired and Revoked status; store, rotate and revoke outbound credentials; labels are unique per system; revealing a stored outbound secret is off by default and needs both a permission and a server flag. Outbound credentials come in four types (API key, static bearer token, Basic and a secret on/off flag), can be stored once per integration type as defaults that a system overrides by label, are bound to an endpoint by an auth label and are sent only to an origin the credential is pinned to. Decrypted values are cached per instance behind version counters, so a revoke applies from the next job, and a missing or unusable credential dead-letters the job at attempt 1 as `CREDENTIAL_MISSING` or `CREDENTIAL_INVALID`.
- System detail page in the console: overview with credential status counts, a credentials tab for issuing, rotating, revoking and revealing, and tabs for endpoint config and limits, CORS origins, health, certificates and SFTP.
- System lifecycle: soft delete and restore (a deleted system is unknown at the gateway, and its code stays reserved), duplicate, and export and import of a system's configuration (never any credential, certificate or SFTP secret). Suspended systems fire no schedules and hold their queued jobs, with no catch-up when they resume.
- Outbound SSRF guard: URLs the platform calls (connector base URLs, health URLs) are checked against private and reserved address ranges before connecting and again at connect time, redirects are not followed, and failures surface as fixed codes. The SFTP host gets the same check, with one DNS resolution whose checked address is the one dialled.
- Certificates and SFTP per system: upload a certificate (optionally with its private key, which is encrypted, checked against the certificate and never returned), see Valid, Expiring, Not yet valid, Expired and Revoked counts that open the matching list, and revoke. Save an SFTP connection (password or private key, write-only), pin the server's host key, and run a rate-limited connection test that refuses to authenticate before a key is pinned. New `certificates:write`, `sftp:write` and `sftp:test` permissions, held by the operator role.
- Connector catalog: every connector's input as a JSON Schema plus a validated sample, generated at build time.
- Operations screens: a rate-limit monitor (current buckets and the last hour of 429s, client IPs truncated), an expiring-credentials list, a health check per system, and a Gateway section on the dashboard.
- Security headers: a per-request Content-Security-Policy with a nonce, HSTS over https, and CORS off unless an origin is allowed (an env allowlist for `/api/*`, each system's own origin list for inbound); a per-IP and per-user rate limit on the admin API (429 with `Retry-After`, fail-open).
- Encrypted credential vault, connector manifest and catalog, Postgres-backed job queue with retry (capped, jittered backoff that honours a partner's `Retry-After`; 429 and 5xx retry, other 4xx fail at once), a dead-letter queue that keeps the partner's masked error text behind `logs:payload`, a throttled after-response drain, stuck-job reclaim, and cron schedules fired exactly once by one signed QStash tick (dormant until an operator arms it).
- Admin console with working sign-in, permission-based access control (checked against the database on every request, 8 hour sessions, throttled sign-in), and pages for systems, catalog, jobs, schedules and audit.
- A typed admin API: one `AdminApi` contract, implemented over HTTP (`/api/admin/*`, one catch-all route driven by an endpoint registry, every request through `withAdmin`) and in the browser for the demo, behind a single client UI. One contract test suite runs against both implementations.
- Manual job run, DLQ replay and DLQ discard from the console, gated by the `jobs:run`, `jobs:retry` and `jobs:discard` permissions. A replay or discard can carry a note (required for a discard), and only an item still awaiting review can be acted on.
- Job detail: each job has a page with its metadata, an ordered step timeline with durations, a failure code from a fixed list and its dead-letter history. The payload loads on demand, redacted, and only for roles that hold `logs:payload`; each view is audited.
- Process log: every scheduled tick, dispatch, drain and retention run is recorded, a duplicate QStash delivery runs nothing, and the Jobs page has a Runs view. Retention (`jobDays`) ages out jobs, step logs and runs under the same kill switch.
- Audit search: the audit log filters by action, actor, target and date range (kept in the URL), pages by 50, and shows a server-rendered diff built from an allowlist that hides any field it does not recognise.
- Activity log screen: every inbound and outbound call is recorded with secrets redacted before storage; filters kept in the URL, payloads loaded on demand for roles that hold `logs:payload`, CSV export, cleanup, and an editable retention setting. Test runs of jobs stay out of the live counts unless "Show test runs" is on.
- CI on every push: typecheck, lint, tests (against an in-memory Postgres) and build.

Alerting is built (P3c): seven rules (dead letters, job failure rate, inbound error rate, certificate expiry, system down, stale scheduler tick, stuck jobs) raise deduplicated alerts and per-user notifications, with a System health page and an admin "Evaluate now" action. In this demo the rules run in your browser over the fictional data. Alert messages carry only codes, counts, labels and dates, never payload text.

Planned, not built yet: user and role management screens, an API playground and generated OpenAPI docs.

## Architecture and security

- [Architecture](docs/architecture.md): system design, the connector model, the job engine, the dormant posture.
- [Security](docs/security.md): the credential vault, inbound authentication and cross-system confinement, rate limiting, permission-based access control, and the threat model.

## Stack

Next.js (App Router) + Drizzle ORM + Neon serverless Postgres, deployed on Vercel. TypeScript throughout, zod for validation.

## Note

This is a public showcase. The source lives in a private repository. All data shown in the demo is generic and fictional.
