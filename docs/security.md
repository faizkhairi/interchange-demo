# Security

Last updated: 2026-10-10

This document describes Interchange's security posture: how credentials are protected at rest, how inbound partner requests are authenticated and confined, how admin access is controlled, why the dormant-by-default posture is itself a security property, and what an attacker can and cannot do if a partner credential is compromised.

## Credential vault

All secret material goes through a single seam: `core/crypto/vault.ts`. There is no other place in the codebase that encrypts, hashes, or compares a secret. This matters because a single audited seam is much easier to reason about than secret-handling code scattered across connectors and routes.

### Outbound credentials: encrypted at rest

Credentials Interchange holds on behalf of a project, to call *out* to a partner (API keys, OAuth2 client secrets, basic-auth passwords), are encrypted with AES-256-GCM, an authenticated cipher:

- A fresh, random 12-byte IV is generated per value and never reused under the same key.
- The master key is a 64-hex-character (32-byte) value read from the `CREDENTIAL_MASTER_KEY` environment variable. It is never sent to the client and never stored in the database.
- Each stored blob is `{ iv, ciphertext, authTag, keyVersion }`. The `authTag` means any tampering with the ciphertext is detected on decrypt, not silently accepted.
- `keyVersion` supports key rotation: `CREDENTIAL_MASTER_KEY_PREVIOUS` can hold the prior key during a rotation window, so old blobs keep decrypting while new writes use the current key.

These blobs live in `system_credential`, `storage_config`, and `sftp_config` (see `db/schema/config-auth.ts`), each with its own `enc_*` column set following the same shape.

Decryption happens in exactly one place in the request path: `core/config/resolve-config.ts`, at the moment a connector's `execute()` is about to run. The decrypted value is attached to `ctx.config.credentials` for that single invocation; the raw encrypted blob never leaves the DB layer, and a connector author never touches the vault directly.

### Inbound credentials: hashed, not encrypted

Credentials a *partner* presents to Interchange (their API key, bearer token, or Basic username and password) are never decryptable, by design: they are hashed with SHA-256 plus a random per-value salt (`hashSecret()` in `core/crypto/vault.ts`), and verified with `verifySecret()` using `timingSafeEqual()` for a constant-time comparison. This avoids two separate problems: there is no plaintext to leak from a database dump, and the comparison itself does not leak timing information about how much of the candidate matched.

The plaintext is shown to the admin exactly once, at creation time, and is not retrievable afterward. If it is lost, the only remedy is rotating the credential (which issues a successor with a new secret) or issuing a new one and revoking the old. See "Credential lifecycle" below.

So that the gateway can find a credential row without verifying every salt, each row also carries a lookup index: an unsalted SHA-256 of the full secret for API keys and bearer tokens, and a keyed HMAC of the username for Basic. See "Lookup indexes and the master-key rotation window" below.

Admin login passwords (the `user` table) are hashed with bcrypt (`bcryptjs`, pure JavaScript, cost 10). Accounts created before bcrypt was introduced hold a salted SHA-256 (`user.hash_algo = 'sha256'`) and are upgraded to bcrypt transparently on their next successful login. The seed script refuses to run in production without `SEED_ADMIN_PASSWORD`, so there is no default admin password on a production database.

## Inbound authentication and cross-system confinement

Every call into `/api/inbound/<systemCode>/<endpoint>` first passes the per-IP rate limit, then `authenticatePartner()` in `core/auth/with-partner-auth.ts`, before any connector code runs:

1. **Extract the credential** from `x-api-key`, or an `Authorization: Bearer <token>` or `Authorization: Basic <base64>` header.
2. **Resolve the system named in the URL first.** The `systemCode` path segment is looked up in `external_system` before the credential is checked against anything.
3. **Reject if the system is not `ACTIVE`.** A system in `SETUP` or `SUSPENDED` rejects all traffic, even if a valid credential exists for it. This is the dormant posture applied at the auth layer, not just at deploy time (see below).
4. **Find the credential, then verify it in constant time.** An API key or bearer token is located by its `lookup_hash` (`system_api_key` or `system_bearer_token`) across all systems; a Basic credential is located by its keyed `username_lookup` (`system_basic_auth`). The salted hash is then verified with the constant-time comparison described above. A row that belongs to the URL's system is rejected with 401 if it is revoked or expired, and otherwise accepted, which stamps `last_used_at` on that credential. A live row that belongs to a different system is refused with 403 (see "Cross-system confinement" below). For Basic, the username and password comparisons both always run and are then combined, so timing does not reveal which half was wrong. A key or token that predates migration `0002` has no `lookup_hash` yet and is checked by a fallback scoped to the URL's own system.
5. **Apply the per-system rate limit** (`external_system.rate_limit_rpm`) only after authentication succeeds.

### Failure responses

Every pre-auth failure (missing credential, unknown system, inactive system, wrong secret, revoked or expired credential) returns the same `401 {"error":"unauthorized"}`, so a caller cannot probe which system codes exist, which are active, or whether a credential was once valid. The one other authentication outcome is the confinement `403 {"error":"forbidden"}` described next. The real reason is recorded only in `activity_log`. Unknown and inactive systems also spend one hash computation so timing matches a wrong secret.

### Cross-system confinement

**A credential for Partner A never authenticates a request to Partner B's routes.** The gateway resolves the URL's system first, then identifies the presented credential by its lookup index across all systems. A real credential presented on the wrong route is therefore recognised and refused, instead of being treated as unknown:

- The refusal is `403` only when a *live* credential proves its identity (its salted hash verifies) but belongs to a different system than the URL names. The response body is the generic `{"error":"forbidden"}`. The owning system's code is written only to the activity row (`ownerSystemCode`, with the reason `credential belongs to another system`) and never appears in a response.
- A revoked or expired credential presented on another system is a plain `401`, indistinguishable from a wrong secret.
- For Basic, `403` needs both the username and the password to match a live row of another system. A username match alone never yields `403`, so the status is no oracle for which usernames exist.
- A key or token issued before migration `0002` has no `lookup_hash` until it is first used. It is checked only against the URL's own system, so on another system it is `401`, not `403`, until a successful use on its own system backfills the index.
- Because a `403` requires a verified secret, it tells nothing to a caller who does not already hold a valid credential. To a holder of a live key it does confirm that the system code in the URL exists; this is accepted by design, since that caller already holds a credential for this platform.

### Lookup indexes and the master-key rotation window

- **`lookup_hash`** (`system_api_key`, `system_bearer_token`, unique): an unsalted SHA-256 of the full secret. This is safe only because these secrets are 192-bit random values generated by the server: there is nothing to guess, so an unsalted digest can be indexed. It never replaces the salted hash, which is still what is verified.
- **`username_lookup`** (`system_basic_auth`, non-unique): a Basic username has low entropy, so a plain digest would let a leaked index be brute-forced back to usernames. The index is an HMAC-SHA256 of the username under a subkey derived from `CREDENTIAL_MASTER_KEY`, so without the key it cannot be reversed. It is not unique because a rotation grace window can hold two rows for one username.
- **Rotation window.** While `CREDENTIAL_MASTER_KEY_PREVIOUS` is set, the gateway computes the username lookup under both keys (current first) and matches a row indexed under either. When a row of the URL's own system authenticates and its stored lookup is not the current-key value, it is re-indexed to the current key. A row that is not used during the window stays indexed under the previous key, so keep `CREDENTIAL_MASTER_KEY_PREVIOUS` set until every Basic credential in use has authenticated once, or reissue it. Once the previous key is removed, a row still indexed under it is no longer found and gets `401`.
- **Rows that predate migration `0002`.** SQL cannot compute the HMAC, so `username_lookup` is NULL for existing Basic rows, and `lookup_hash` is NULL for existing keys and tokens. The gateway verifies such rows only against the URL's own system, and on a match fills in the index (or moves `username_lookup` to the current key), so legacy rows upgrade on first use.

## Credential lifecycle

Inbound credentials (API key, bearer token, Basic) and outbound credentials (encrypted, held to call a partner) are managed from the system's detail page in the console, or through the admin API under `/api/admin`. Every mutation writes an `admin_audit_log` row (`api_key.issue`, `bearer_token.issue`, `basic_auth.set`, `credential.store`, `credential.rotate`, `credential.revoke`, `credential.reveal`). No response and no audit row carries a stored secret, hash, salt or Basic username.

- **Kinds and status.** The kinds are `api_key`, `bearer`, `basic` and `outbound`. Status is derived from timestamps by one shared function, so the server, the list counts and the demo agree: `REVOKED` (`revoked_at` is set; wins over everything), `EXPIRED` (`expires_at` has passed, which includes the end of a rotation grace), `ROTATING` (a successor exists and the row is still inside its grace), otherwise `ACTIVE`. Only `ACTIVE` and `ROTATING` credentials authenticate.
- **Issue.** `POST /systems/:id/api-keys`, `/bearer-tokens` and `/basic-auth` need `credentials:write`. The server generates the secret (`ic_` plus 24 random bytes, base64url) and returns it once. For Basic the caller picks the username and the server generates the password. `label` (1 to 100 characters) and `expiresInDays` (1 to 3650) are optional: an omitted label becomes a generated `key-`, `token-` or `basic-` name, and an omitted expiry means the credential does not expire. An outbound credential is stored with `POST /systems/:id/credentials`; its label defaults to its credential type, which is the key connectors read it under.
- **Label uniqueness.** A label is unique per system among credentials that are neither revoked nor rotated away (outbound: not revoked), enforced by a partial unique index per table. Issuing a label that is already active or rotating answers `409 CONFLICT`, including when two requests race. This also means two active outbound credentials can never share a label, so one cannot shadow another when connector config is resolved. An index predicate cannot use the current time, so an expired but unrevoked row still holds its label; issuing that label first stamps the expired row revoked, in the same transaction.
- **Expiry.** The gateway rejects an expired credential with `401`, and the list reports it as `EXPIRED`. The API cannot create an already-expired credential (the minimum is one day).
- **Rotate.** `POST /credentials/:kind/:credId/rotate`. For inbound kinds it creates a successor with the same label and a new server-generated secret, returned once; Basic keeps its username and gets a new password. `graceMinutes` (0 to 1440, default 0) decides what happens to the old row. With 0 it is revoked at once. Otherwise it keeps working until the grace ends (or until its own earlier expiry), shows as `ROTATING`, then becomes `EXPIRED`. The successor inherits the original lifetime, so a 30 day key rotates into a 30 day key and a key without expiry stays without one. The old row is updated before the successor is inserted, in one transaction, because the unique index would otherwise reject the new row. A revoked, expired or already rotated credential answers `409`. An outbound rotation needs a new `secret`, re-encrypts it in place under the same label and stamps `rotated_at`; it has no grace window. An inbound rotation refuses a supplied `secret`.
- **Revoke.** `POST /credentials/:kind/:credId/revoke` sets `revoked_at`. It takes effect on the next request: the gateway answers `401`, and a revoked outbound credential drops out of the resolved connector config. Revoking a credential also revokes any predecessor still inside its rotation grace, so revoking the successor closes the whole chain. It is deliberately not idempotent: a second revoke answers `409`, so a double click or a stale screen is visible rather than silently accepted.
- **Reveal (off by default).** Only outbound credentials can be revealed; inbound secrets are hashed and cannot be recovered. Two independent switches must both be on: the user holds `credentials:reveal`, and the server has `CREDENTIALS_REVEAL_ENABLED=1`. The second exists because the `admin` role holds `*`, so a permission alone could never make reveal off by default. With the variable unset, even an admin gets `403 FORBIDDEN` ("Reveal is disabled"), checked before any lookup. A revoked credential answers `409`. Reveal is fail-closed: the plaintext is returned only after the `credential.reveal` audit row (the label, never the value) has committed, so a reveal that cannot be audited does not happen. Reveal also has its own per-user throttle, `CREDENTIALS_REVEAL_LIMIT` reveals per 10 minutes (default 10), on top of the admin rate limit. `GET /api/admin/me` reports `features.credentialReveal`, so the console hides the Reveal action when the server has it off.
- **Listing.** `GET /systems/:id/credentials` (`systems:read`) takes `status`, `kind` and `offset`, and returns at most 50 rows with a `total` for the same filters. Rows are mapped through the status function first and filtered second, so a status count and the list behind it always agree. A response never includes a secret, hash or salt.
- **Console.** The system detail page (`systems/<id>`, reached from the Systems list) has an Overview tab with Active, Rotating, Expired and Revoked counts that link to the Credentials tab filtered by that status. The Credentials tab lists the credentials and offers Issue, Rotate (with a grace field), Revoke and, when enabled, Reveal. A new secret is shown once. A viewer sees no mutation controls.

## RBAC (admin console)

Admin access uses `next-auth` with a credentials provider backed by Interchange's own `user` table (`core/auth/options.ts`, `core/auth/login.ts`). Session strategy is JWT with an 8 hour lifetime (session and token `maxAge`); role names, permissions and the user's `token_version` are resolved at login and carried in the token, but nothing authoritative reads them from there.

Authorization is by **permission**, not role name. The catalog is a constant in `core/auth/permissions.ts` (`systems:read`, `systems:write`, `credentials:write`, `credentials:reveal`, `jobs:read`, `jobs:run`, `jobs:retry`, `schedules:read`, `schedules:write`, `audit:read`, `users:read`, `users:write`, `roles:write`, `settings:write`), stored on `role.permissions`, with `*` meaning superadmin. `jobs:run` and `jobs:retry` are in the catalog and granted to `operator`, and the endpoints that check them are `jobs.run` (manual run, `jobs:run`) and `dlq.replay` (DLQ replay, `jobs:retry`).

The authoritative gate is `requirePermission(perm)` in `core/auth/rbac.ts`, run by `withAdmin` (`core/http/with-admin.ts`) on every `/api/admin/*` request, before any input is parsed: an unauthenticated, disabled or stale-token caller gets 401, and a signed-in caller without the endpoint's permission gets 403. The admin layout does a DB-backed check too (`requireActiveUser()`, redirecting to `/login`), and the client hides navigation entries the user cannot open. Neither trusts the JWT: on every call each re-reads the user's current status, `token_version` and role permissions from the database in one join. A role change, a disabled account, or a `token_version` bump (the mechanism for "log out everywhere") therefore takes effect on the next request, and a token carrying a stale `token_version` is rejected.

The seeded roles (`db/seed.ts`) are `admin` (`*`), `operator` (systems read/write, credentials write, jobs read/run/retry, schedules read/write, audit read) and `viewer` (the `:read` permissions for systems, jobs, schedules and audit).

Failed sign-ins write an `auth.login_failed` row to `admin_audit_log` (the attempted email and a reason, never the password). Sign-in is throttled per normalised email (10 attempts per 15 minutes) and per client IP (50 per 15 minutes) through `rate_limit_bucket`; a throttled attempt gets the same generic failure as a wrong password, and the first rejection per bucket writes an `auth.login_throttled` audit row. A bcrypt comparison runs on every path, including unknown users and legacy SHA-256 accounts, so timing does not reveal whether an account exists. There is no account lockout (an attacker who knows an email can hold that email at its limit for the window; the IP cap and the generic response bound the effect).

Client-side route redirects and hidden UI elements are UX conveniences only. `requirePermission()` running server-side is the real gate; it is never bypassed by disabling JavaScript or calling `/api/admin/*` directly. The admin surface is the typed `/api/admin/*` HTTP API: one catch-all route (`src/app/api/admin/[...path]/route.ts`) driven by an endpoint registry, with every request wrapped in `withAdmin`. In order, `withAdmin`:

1. assigns a request id (an inbound `X-Request-ID` is kept only when it is short and made of safe characters; the edge proxy resolves it first and `withAdmin` reuses it; the id is echoed on every response);
2. applies the per-IP rate limit (`admin:ip:<client ip>`), before the CSRF guard and before any session lookup, so anonymous and cross-origin floods are throttled too (429 `RATE_LIMITED`);
3. on non-GET requests, refuses cross-origin callers (CSRF guard: `Origin` must equal the request's own origin, scheme, host and port, or, when `Origin` is absent, `Sec-Fetch-Site` must be `same-origin`) and any body that is not `application/json` (415);
4. checks the endpoint's permission against the database (`requirePermission`), before any input is parsed;
5. applies the per-user rate limit (`admin:user:<user id>`), only after authorization, so refused and anonymous requests never spend a real user's budget (429 `RATE_LIMITED`);
6. reads the body with a 64 KB cap (413) and validates input with zod (400);
7. runs the service and returns JSON in one error envelope, `{"error":{"code","message","requestId","issues?"}}`. Validation `issues` carry paths and messages, never submitted values, and an unexpected failure returns a fixed `INTERNAL` message while the detail is logged server-side with the request id.

Every `/api/admin/*` response carries `Cache-Control: no-store`, because one of them returns a plaintext API key exactly once.

## Admin rate limit

`/api/admin/*` is limited per minute, in one-minute fixed windows, by the same Postgres-backed limiter the gateway uses (`rate_limit_bucket`):

| Limit | Bucket | Default | Override | Checked |
|---|---|---|---|---|
| Per client IP | `admin:ip:<client ip>` | 300 per minute | `ADMIN_RATE_LIMIT_IP_RPM` | first, before the CSRF guard and any session lookup |
| Per user | `admin:user:<user id>` | 120 per minute | `ADMIN_RATE_LIMIT_USER_RPM` | after authorization, before the body is read |

- The per-IP limit runs before authentication on purpose, so an anonymous or cross-origin flood is throttled without touching the session store. The per-user limit runs after authorization on purpose, so refused and anonymous requests never spend a signed-in user's budget.
- Over a limit the response is `429` with the error envelope code `RATE_LIMITED` ("too many requests"), a `Retry-After` header (whole seconds until the window ends) and `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`. The HTTP client surfaces `Retry-After` as `retryAfterSec` on the thrown error.
- Both variables are read on every request. An unset, empty, non-numeric or non-positive value means the default; a fractional value is rounded down.
- The limiter **fails open**: if the database cannot be reached, the request is admitted, a warning is logged and no rate-limit headers are sent. A limiter outage must not lock administrators out.
- The client IP is read from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, and must parse as an IP address (see "Rate limiting and activity logging"). The per-IP limit is therefore only sound behind Vercel or a trusted proxy that overwrites those headers: on a host that passes client-supplied values through, a caller can choose its own bucket and evade the per-IP limit. An unrecognised address shares the `unknown` bucket. The per-user limit does not depend on the IP.

## Edge headers: CSP, HSTS and CORS

Four static headers come from `next.config.ts` and apply to every response, static assets included: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin` and a `Permissions-Policy` that disables camera, microphone and geolocation. Everything that varies per request is set by the Next proxy (`src/proxy.ts`, Node runtime), using the pure builders in `core/http/security-headers.ts`. The proxy skips `_next/static`, `_next/image`, the favicon files and prefetch requests; that is why the four static headers stay in `next.config.ts`.

For each request the proxy:

1. resolves the request id (a valid `X-Request-ID` is kept, otherwise a fresh UUID is generated) and puts it on the forwarded request and the response, where `withAdmin` and the inbound route reuse it;
2. generates a nonce and forwards it as `x-nonce` and inside the request's `Content-Security-Policy`, which is how Next attaches the nonce to its own scripts; the root layout puts the same nonce on the one inline theme script;
3. sets the `Content-Security-Policy` on the response, plus HSTS over https;
4. applies the CORS rule below to `/api/*`.

### Content-Security-Policy

```
default-src 'self'
script-src 'self' 'nonce-<per-request>' 'strict-dynamic'
style-src 'self' 'unsafe-inline'
img-src 'self' data: blob:
font-src 'self'
connect-src 'self'
object-src 'none'
base-uri 'self'
form-action 'self'
frame-ancestors 'none'
upgrade-insecure-requests        (https requests only)
```

- Scripts get no `'unsafe-inline'`: only the per-request nonce plus `'strict-dynamic'` (which also covers scripts those load). The nonce is a base64 encoding of a random UUID and differs on every request.
- `style-src` keeps `'unsafe-inline'` because pages render many server-side `style=` attributes, and adding a nonce to `style-src` would make browsers ignore `'unsafe-inline'`.
- Only when `NODE_ENV` is `development`, `script-src` adds `'unsafe-eval'` (React debugging) and `connect-src` adds `ws:` (hot reload).
- A nonce needs a request, so every page renders dynamically and loses static caching. The demo at `/demo` still makes no `/api` calls.

### HSTS

`Strict-Transport-Security: max-age=63072000; includeSubDomains`, sent only when the request arrived over https (the first hop of `x-forwarded-proto`, else the request URL scheme). There is no `preload`, which is a domain-wide commitment. `upgrade-insecure-requests` follows the same rule; sent on a plain http origin such as a local `127.0.0.1` run, it would rewrite subresource requests and break the page.

### CORS

CORS is **deny-all unless configured**, in every environment.

- It applies to `/api/*` and only to a cross-origin request (an `Origin` header that differs from the request's own origin). Same-origin requests and requests with no `Origin` are untouched.
- `CORS_ALLOWED_ORIGINS` is a comma-separated list of exact origins (scheme, host and port). Unset or empty allows nothing. Entries that are not an `http` or `https` origin, including `*`, are dropped, so a wildcard cannot be configured by accident.
- A request from an allowed origin gets `Access-Control-Allow-Origin` set to that exact origin, plus `Vary: Origin`. Any other cross-origin request gets no `Access-Control-Allow-Origin`, so the browser blocks the response.
- A cross-origin `OPTIONS` preflight is answered by the proxy itself with `204`. For an allowed origin it also carries `Access-Control-Allow-Methods: POST, OPTIONS`, `Access-Control-Allow-Headers: Authorization, Content-Type, X-API-Key, X-Request-ID, Idempotency-Key` and `Access-Control-Max-Age: 600`; for any other origin it carries no `Access-Control-Allow-*` header.
- `/api/admin/*` never gets any CORS header, whatever the allowlist says. The admin API does its own same-origin CSRF check and is not meant for other origins. The path is normalised (decoded, lower-cased, repeated slashes collapsed) before this check, and an undecodable path is treated as admin.
- Credentialed CORS (`Access-Control-Allow-Credentials`) is never enabled. Per-system origin lists are planned, not built.

## Job endpoint authentication

`/api/jobs/tick`, `/api/jobs/dispatch` and `/api/jobs/process` accept only requests signed by Upstash QStash. The `Upstash-Signature` JWT is verified against the raw body with `QSTASH_CURRENT_SIGNING_KEY` or `QSTASH_NEXT_SIGNING_KEY` (`core/jobs/verify-qstash.ts`). With the keys unset the endpoints return 503 (fail closed); The verifier is given the expected url (`APP_URL` plus the request path), so a signature for one endpoint is not accepted on another. `QSTASH_DEV_BYPASS=1` is honoured only when `NODE_ENV` is not `production`. The SDK's own `QSTASH_DEV` mode, which would substitute public development keys, is disabled.

## Rate limiting and activity logging

Inbound traffic is limited per client IP (taken from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, validated as an IP address, else `unknown`) (`RATE_LIMIT_IP_RPM`, default 250 per minute) and per system (`external_system.rate_limit_rpm`), using one atomic upsert per request against `rate_limit_bucket`. A rejected call returns 429 with `Retry-After`. The limiter fails open if the database is unreachable (a logged warning), trading brief over-admission for availability. Every inbound outcome, including 401, 429 and validation failures, writes an `activity_log` row with the request id, status, timing and both payloads; secret-looking body keys (case-insensitive, recursive) are redacted, rows older than `ACTIVITY_RETENTION_DAYS` (default 30) are pruned by the tick, and credentials or `Authorization` headers are never stored.

## Audit logging

Every privileged admin mutation writes a row to `admin_audit_log` via `auditLog()` in `core/audit/log.ts`: who (`userId`), what (`action`), and on what (`targetTable`, `targetId`, and a `diff` of the change). Failed sign-ins are recorded too (`auth.login_failed`). Audit writes are fire-and-forget: a failure to write the audit row never blocks or rolls back the mutation itself, so an audit-log outage cannot be used to block legitimate admin operations, though it does mean audit completeness is best-effort rather than transactional. This same fire-and-forget principle applies to `job_log` writes (`makeJobLogger()` in `core/jobs/lifecycle.ts`): a logging failure must never break a job.

## Dormant posture as a security property

The provisioned-dormant default is described in `docs/architecture.md` as a cost-control property; it is equally a security property:

- A freshly deployed Interchange instance, even one seeded with real connector types and even one where an operator has (prematurely, or by mistake) issued API keys for a partner system, **rejects all inbound traffic for that system** until an operator explicitly sets `status = 'ACTIVE'`. There is no state between "provisioned" and "live" where a half-configured system quietly accepts requests.
- No outbound schedule fires and no drain runs on a timer until QStash is explicitly armed (`QSTASH_TOKEN` set and the platform tick armed from the Schedules page; arming a single schedule registers nothing with QStash; the job endpoints additionally require signed requests). A deploy with a misconfigured or forgotten schedule does not leak outbound traffic; it does nothing.
- This means the platform's default failure mode, for both a fresh deploy and a partially configured one, is silence rather than partial exposure.

## Threat model: a compromised partner credential

This section describes what an attacker who obtains a valid, active API key or bearer token for one partner system can and cannot do.

**Can do:**

- Call any inbound endpoint (`/api/inbound/<systemCode>/<endpoint>`) that belongs to *that specific system*, as many times as the per-system rate limit and the connector's own logic allow.
- Submit any payload that satisfies the connector's zod `inputSchema`. If the schema is permissive (for example, the `rest-inbound` template accepts an arbitrary `data` record), the attacker can submit arbitrary data within that shape.
- Cause jobs to be created and connector `execute()` logic to run for that system, including any side effects that connector performs (writes, downstream calls, file drops).
- Read whatever the connector's response returns to the caller (the `output` of a successful `execute()`, or error detail on a validation failure).

**Cannot do:**

- Authenticate as, or submit requests on behalf of, any *other* partner system. A stolen credential is scoped to exactly one `external_system`: presented on another system's route it is recognised and refused with 403 (logged with the owning system in the activity row), never accepted. See "Inbound authentication and cross-system confinement" above.
- Decrypt any outbound credential Interchange holds for that system, or any other system. Inbound API keys and bearer tokens are hashed, not encrypted; there is no reverse operation from a hash back to the original secret, so possessing (or even fully dumping) the `system_api_key` table does not expose usable plaintext credentials (the unsalted `lookup_hash` beside the salted hash is a SHA-256 of a 192-bit random secret, so it does not change that), and the outbound `system_credential` vault is a separate, independently protected store.
- Reactivate a `SUSPENDED` or `SETUP` system. Status is controlled only through the admin console (`requirePermission()`-gated), never by the inbound path itself.
- Access the admin console. Partner credentials and admin user sessions are entirely separate authentication mechanisms (`with-partner-auth.ts` vs `next-auth` + `rbac.ts`); a partner API key is never accepted as an admin session token.
- Erase evidence of its own activity. Every request produces an `activity_log` row, and every request that reaches a connector also produces a `job` row and associated `job_log` entries; these are platform-owned and not writable by the caller.

**Net exposure of a single compromised credential**: everything one partner system's own connector logic can be made to do, once, repeatedly, until the key is revoked. It does not expose other partners, does not expose the credential vault, and does not grant any administrative capability. Revoking the specific credential, from the system's Credentials tab in the console or with `POST /api/admin/credentials/:kind/:credId/revoke`, sets `revoked_at` and immediately closes that path without affecting any other system. Rotating with a grace of 0 does the same while issuing a replacement, and a credential with an expiry stops working on its own at `expires_at`.
