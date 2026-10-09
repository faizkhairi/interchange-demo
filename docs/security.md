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

The plaintext is shown to the admin exactly once, at creation time, and is not retrievable afterward. If it is lost, the only remedy is issuing a new one and revoking the old.

Admin login passwords (the `user` table) are hashed with bcrypt (`bcryptjs`, pure JavaScript, cost 10). Accounts created before bcrypt was introduced hold a salted SHA-256 (`user.hash_algo = 'sha256'`) and are upgraded to bcrypt transparently on their next successful login. The seed script refuses to run in production without `SEED_ADMIN_PASSWORD`, so there is no default admin password on a production database.

## Inbound authentication and cross-system confinement

Every call into `/api/inbound/<systemCode>/<endpoint>` first passes the per-IP rate limit, then `authenticatePartner()` in `core/auth/with-partner-auth.ts`, before any connector code runs:

1. **Extract the credential** from `x-api-key`, or an `Authorization: Bearer <token>` or `Authorization: Basic <base64>` header.
2. **Resolve the system named in the URL first.** The `systemCode` path segment is looked up in `external_system` before the credential is checked against anything.
3. **Reject if the system is not `ACTIVE`.** A system in `SETUP` or `SUSPENDED` rejects all traffic, even if a valid credential exists for it. This is the dormant posture applied at the auth layer, not just at deploy time (see below).
4. **Verify the credential against that system's own credentials only** (`system_api_key` or `system_bearer_token`, filtered by `external_system_id` and `revoked_at IS NULL`; or the system's single `system_basic_auth` row), using the constant-time comparison described above. For Basic, the username and password comparisons both always run and are then combined, so timing does not reveal which half was wrong; an expired or revoked Basic credential is rejected. A successful match stamps `last_used_at` on that credential.
Every pre-auth failure (missing credential, unknown system, inactive system, wrong secret) returns the same `401 {"error":"unauthorized"}`, so a caller cannot probe which system codes exist or which are active. The real reason is recorded only in `activity_log`. Unknown and inactive systems also spend one hash computation so timing matches a wrong secret.
5. **Apply the per-system rate limit** (`external_system.rate_limit_rpm`) only after authentication succeeds.

The order of steps 2 and 4 is the important part: **a credential is checked only against the system named in the URL.** A valid, active credential belonging to Partner A can never authenticate a request to Partner B's routes, because Partner A's keys are never even queried when the URL names Partner B. This is enforced in code, not by convention: the query in step 4 is scoped by `external_system_id` resolved from the URL, so there is no code path where a different system's keys could satisfy the check.

## RBAC (admin console)

Admin access uses `next-auth` with a credentials provider backed by Interchange's own `user` table (`core/auth/options.ts`, `core/auth/login.ts`). Session strategy is JWT with an 8 hour lifetime (session and token `maxAge`); role names, permissions and the user's `token_version` are resolved at login and carried in the token, but nothing authoritative reads them from there.

Authorization is by **permission**, not role name. The catalog is a constant in `core/auth/permissions.ts` (`systems:read`, `systems:write`, `credentials:write`, `credentials:reveal`, `jobs:read`, `jobs:run`, `jobs:retry`, `schedules:read`, `schedules:write`, `audit:read`, `users:read`, `users:write`, `roles:write`, `settings:write`), stored on `role.permissions`, with `*` meaning superadmin. `jobs:run` and `jobs:retry` are in the catalog and granted to `operator`, but the actions that check them (DLQ replay, manual run) are wired in P1b.

The authoritative gate is `requirePermission(perm)` in `core/auth/rbac.ts`, called from every admin server action. Every admin page also calls `guardPage(<domain>:read)` (`src/app/admin/_components/page-guard.tsx`, which uses `requirePermission`): an unauthenticated, disabled or stale-token user is redirected to `/login`, and a signed-in user without the permission sees a 403 state instead of the page. The admin layout does a DB-backed check too (`requireActiveUser()`), and hides navigation entries the user cannot open. It does not trust the JWT: on every call it re-reads the user's current status, `token_version` and role permissions from the database in one join. A role change, a disabled account, or a `token_version` bump (the mechanism for "log out everywhere") therefore takes effect on the next request, and a token carrying a stale `token_version` is rejected.

The seeded roles (`db/seed.ts`) are `admin` (`*`), `operator` (systems read/write, credentials write, jobs read/run/retry, schedules read/write, audit read) and `viewer` (the `:read` permissions for systems, jobs, schedules and audit).

Failed sign-ins write an `auth.login_failed` row to `admin_audit_log` (the attempted email and a reason, never the password). Sign-in is throttled per normalised email (10 attempts per 15 minutes) and per client IP (50 per 15 minutes) through `rate_limit_bucket`; a throttled attempt gets the same generic failure as a wrong password, and the first rejection per bucket writes an `auth.login_throttled` audit row. A bcrypt comparison runs on every path, including unknown users and legacy SHA-256 accounts, so timing does not reveal whether an account exists. There is no account lockout (an attacker who knows an email can hold that email at its limit for the window; the IP cap and the generic response bound the effect).

Client-side route redirects and hidden UI elements are UX conveniences only. `requirePermission()` running server-side is the real gate; it is never bypassed by disabling JavaScript or calling a server action directly. The admin surface today is server actions plus server-rendered pages; a typed `/api/admin/*` HTTP API does not exist yet (planned).

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

- Authenticate as, or submit requests on behalf of, any *other* partner system. The systemCode-confinement check means the stolen credential is scoped to exactly one `external_system` row; it is never checked against another system's keys. See "Inbound authentication and cross-system confinement" above.
- Decrypt any outbound credential Interchange holds for that system, or any other system. Inbound API keys and bearer tokens are hashed, not encrypted; there is no reverse operation from a hash back to the original secret, so possessing (or even fully dumping) the `system_api_key` table does not expose usable plaintext credentials, and the outbound `system_credential` vault is a separate, independently protected store.
- Reactivate a `SUSPENDED` or `SETUP` system. Status is controlled only through the admin console (`requirePermission()`-gated), never by the inbound path itself.
- Access the admin console. Partner credentials and admin user sessions are entirely separate authentication mechanisms (`with-partner-auth.ts` vs `next-auth` + `rbac.ts`); a partner API key is never accepted as an admin session token.
- Erase evidence of its own activity. Every request produces an `activity_log` row, and every request that reaches a connector also produces a `job` row and associated `job_log` entries; these are platform-owned and not writable by the caller.

**Net exposure of a single compromised credential**: everything one partner system's own connector logic can be made to do, once, repeatedly, until the key is revoked. It does not expose other partners, does not expose the credential vault, and does not grant any administrative capability. Revoking the specific `system_api_key`, `system_bearer_token` or `system_basic_auth` row (setting `revoked_at`) immediately closes that path without affecting any other system.
