# Security

Last updated: 2026-07-07

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

Credentials a *partner* presents to Interchange (their API key or bearer token) are never decryptable, by design: they are hashed with SHA-256 plus a random per-value salt (`hashSecret()` in `core/crypto/vault.ts`), and verified with `verifySecret()` using `timingSafeEqual()` for a constant-time comparison. This avoids two separate problems: there is no plaintext to leak from a database dump, and the comparison itself does not leak timing information about how much of the candidate matched.

The plaintext is shown to the admin exactly once, at creation time, and is not retrievable afterward. If it is lost, the only remedy is issuing a new one and revoking the old.

Admin login passwords (the `user` table) use the same `hashSecret()` / `verifySecret()` pair.

## Inbound authentication and cross-system confinement

Every call into `/api/inbound/<systemCode>/<endpoint>` passes through `authenticatePartner()` in `core/auth/with-partner-auth.ts` before any connector code runs:

1. **Extract the credential** from `x-api-key`, or an `Authorization: Bearer <token>` header (Basic auth is scaffolded in the type but intentionally not implemented in this slice).
2. **Resolve the system named in the URL first.** The `systemCode` path segment is looked up in `external_system` before the credential is checked against anything.
3. **Reject if the system is not `ACTIVE`.** A system in `SETUP` or `SUSPENDED` rejects all traffic with a 403, even if a valid credential exists for it. This is the dormant posture applied at the auth layer, not just at deploy time (see below).
4. **Verify the credential against that system's own keys only** (`system_api_key` or `system_bearer_token`, filtered by `external_system_id` and `revoked_at IS NULL`), using the constant-time comparison described above.

The order of steps 2 and 4 is the important part: **a credential is checked only against the system named in the URL.** A valid, active credential belonging to Partner A can never authenticate a request to Partner B's routes, because Partner A's keys are never even queried when the URL names Partner B. This is enforced in code, not by convention: the query in step 4 is scoped by `external_system_id` resolved from the URL, so there is no code path where a different system's keys could satisfy the check.

## RBAC (admin console)

Admin access uses `next-auth` with a credentials provider backed by Interchange's own `user` table (`core/auth/options.ts`). Session strategy is JWT; role names are resolved from `user_role` / `role` at login and carried in the token.

The authoritative gate is `requireRole()` in `core/auth/rbac.ts`, called from every admin server action and `/api/admin/*` handler:

```ts
export async function requireRole(...allowed: string[]): Promise<SessionUser> {
  const u = await getSessionUser();
  if (!u) throw new UnauthorizedError();
  if (u.roles.includes("admin")) return u;
  if (allowed.some((r) => u.roles.includes(r))) return u;
  throw new ForbiddenError(`requires one of: ${allowed.join(", ")}`);
}
```

The seeded roles (`db/seed.ts`) are `admin` (all permissions), `operator` (`system:read`, `system:write`, `job:read`, `job:run`), and `viewer` (`system:read`, `job:read`). `admin` always passes any `requireRole()` check regardless of the specific roles listed, by design.

Client-side route redirects and hidden UI elements are UX conveniences only. `requireRole()` running server-side is the real gate; it is never bypassed by disabling JavaScript or calling a server action directly.

## Audit logging

Every privileged admin mutation writes a row to `admin_audit_log` via `auditLog()` in `core/audit/log.ts`: who (`userId`), what (`action`), and on what (`targetTable`, `targetId`, and a `diff` of the change). Audit writes are fire-and-forget: a failure to write the audit row never blocks or rolls back the mutation itself, so an audit-log outage cannot be used to block legitimate admin operations, though it does mean audit completeness is best-effort rather than transactional. This same fire-and-forget principle applies to `job_log` writes (`makeJobLogger()` in `core/jobs/lifecycle.ts`): a logging failure must never break a job.

## Dormant posture as a security property

The provisioned-dormant default is described in `docs/architecture.md` as a cost-control property; it is equally a security property:

- A freshly deployed Interchange instance, even one seeded with real connector types and even one where an operator has (prematurely, or by mistake) issued API keys for a partner system, **rejects all inbound traffic for that system** until an operator explicitly sets `status = 'ACTIVE'`. There is no state between "provisioned" and "live" where a half-configured system quietly accepts requests.
- No outbound schedule fires and no drain runs on a timer until QStash is explicitly armed (`QSTASH_TOKEN` set and `armSchedule()` called). A deploy with a misconfigured or forgotten schedule does not leak outbound traffic; it does nothing.
- This means the platform's default failure mode, for both a fresh deploy and a partially configured one, is silence rather than partial exposure.

## Threat model: a compromised partner credential

This section describes what an attacker who obtains a valid, active API key or bearer token for one partner system can and cannot do.

**Can do:**

- Call any inbound endpoint (`/api/inbound/<systemCode>/<endpoint>`) that belongs to *that specific system*, as many times as the connector's own logic allows.
- Submit any payload that satisfies the connector's zod `inputSchema`. If the schema is permissive (for example, the `rest-inbound` template accepts an arbitrary `data` record), the attacker can submit arbitrary data within that shape.
- Cause jobs to be created and connector `execute()` logic to run for that system, including any side effects that connector performs (writes, downstream calls, file drops).
- Read whatever the connector's response returns to the caller (the `output` of a successful `execute()`, or error detail on a validation failure).

**Cannot do:**

- Authenticate as, or submit requests on behalf of, any *other* partner system. The systemCode-confinement check means the stolen credential is scoped to exactly one `external_system` row; it is never checked against another system's keys. See "Inbound authentication and cross-system confinement" above.
- Decrypt any outbound credential Interchange holds for that system, or any other system. Inbound API keys and bearer tokens are hashed, not encrypted; there is no reverse operation from a hash back to the original secret, so possessing (or even fully dumping) the `system_api_key` table does not expose usable plaintext credentials, and the outbound `system_credential` vault is a separate, independently protected store.
- Reactivate a `SUSPENDED` or `SETUP` system. Status is controlled only through the admin console (`requireRole()`-gated), never by the inbound path itself.
- Access the admin console. Partner credentials and admin user sessions are entirely separate authentication mechanisms (`with-partner-auth.ts` vs `next-auth` + `rbac.ts`); a partner API key is never accepted as an admin session token.
- Erase evidence of its own activity. Every request that reaches a connector produces a `job` row and associated `job_log` entries; these are platform-owned and not writable by the caller.

**Net exposure of a single compromised credential**: everything one partner system's own connector logic can be made to do, once, repeatedly, until the key is revoked. It does not expose other partners, does not expose the credential vault, and does not grant any administrative capability. Revoking the specific `system_api_key` or `system_bearer_token` row (setting `revoked_at`) immediately closes that path without affecting any other system.
