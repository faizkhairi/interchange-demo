# Security

Last updated: 2026-10-11

This document describes Interchange's security posture: how credentials are protected at rest, how inbound partner requests are authenticated and confined, how outbound requests are kept away from internal networks, how admin access is controlled, why the dormant-by-default posture is itself a security property, and what an attacker can and cannot do if a partner credential is compromised.

## Credential vault

All secret material goes through a single seam: `core/crypto/vault.ts`. There is no other place in the codebase that encrypts, hashes, or compares a secret. This matters because a single audited seam is much easier to reason about than secret-handling code scattered across connectors and routes.

### Outbound credentials: encrypted at rest

Credentials Interchange holds on behalf of a project, to call *out* to a partner (API keys, OAuth2 client secrets, basic-auth passwords), are encrypted with AES-256-GCM, an authenticated cipher:

- A fresh, random 12-byte IV is generated per value and never reused under the same key.
- The master key is a 64-hex-character (32-byte) value read from the `CREDENTIAL_MASTER_KEY` environment variable. It is never sent to the client and never stored in the database.
- Each stored blob is `{ iv, ciphertext, authTag, keyVersion }`. The `authTag` means any tampering with the ciphertext is detected on decrypt, not silently accepted.
- `keyVersion` supports key rotation: `CREDENTIAL_MASTER_KEY_PREVIOUS` can hold the prior key during a rotation window, so old blobs keep decrypting while new writes use the current key.

These blobs live in `system_credential`, `storage_config`, `sftp_config` (the SFTP password or private key, `enc_secret_*`) and `system_certificate` (an optional certificate private key, `enc_key_*`); see `db/schema/config-auth.ts`. Each has its own `enc_*` column set following the same shape.

Decryption happens in only three places. `core/config/resolve-config.ts` does it at the moment a connector's `execute()` is about to run: the decrypted value is attached to `ctx.config.credentials` for that single invocation, and a connector author never touches the vault directly. Credential reveal (`core/services/credentials.ts`) does it for an outbound credential, and only when reveal is switched on (see "Reveal" below). The SFTP connection test (`core/services/sftp.ts`) decrypts the SFTP secret, and only after the server's host key has matched the pinned one (see "Certificates and SFTP"). The raw encrypted blob never leaves the DB layer. A certificate's private key is never decrypted: it has no read path at all.

### Inbound credentials: hashed, not encrypted

Credentials a *partner* presents to Interchange (their API key, bearer token, or Basic username and password) are never decryptable, by design: they are hashed with SHA-256 plus a random per-value salt (`hashSecret()` in `core/crypto/vault.ts`), and verified with `verifySecret()` using `timingSafeEqual()` for a constant-time comparison. This avoids two separate problems: there is no plaintext to leak from a database dump, and the comparison itself does not leak timing information about how much of the candidate matched.

The plaintext is shown to the admin exactly once, at creation time, and is not retrievable afterward. If it is lost, the only remedy is rotating the credential (which issues a successor with a new secret) or issuing a new one and revoking the old. See "Credential lifecycle" below.

So that the gateway can find a credential row without verifying every salt, each row also carries a lookup index: an unsalted SHA-256 of the full secret for API keys and bearer tokens, and a keyed HMAC of the username for Basic. See "Lookup indexes and the master-key rotation window" below.

Admin login passwords (the `user` table) are hashed with bcrypt (`bcryptjs`, pure JavaScript, cost 10). Accounts created before bcrypt was introduced hold a salted SHA-256 (`user.hash_algo = 'sha256'`) and are upgraded to bcrypt transparently on their next successful login. The seed script refuses to run in production without `SEED_ADMIN_PASSWORD`, so there is no default admin password on a production database.

## Inbound authentication and cross-system confinement

Every call into `/api/inbound/<systemCode>/<endpoint>` first passes the per-IP rate limit, then `authenticatePartner()` in `core/auth/with-partner-auth.ts`, before any connector code runs:

1. **Extract the credential** from `x-api-key`, or an `Authorization: Bearer <token>` or `Authorization: Basic <base64>` header.
2. **Resolve the system named in the URL first.** The `systemCode` path segment is looked up in `external_system` before the credential is checked against anything. Soft-deleted systems are excluded from that lookup, so a deleted system is an unknown system (see "Deleting and suspending systems" below).
3. **Reject if the system is not `ACTIVE`.** A system in `SETUP` or `SUSPENDED` rejects all traffic, even if a valid credential exists for it. This is the dormant posture applied at the auth layer, not just at deploy time (see below).
4. **Find the credential, then verify it in constant time.** An API key or bearer token is located by its `lookup_hash` (`system_api_key` or `system_bearer_token`) across all systems; a Basic credential is located by its keyed `username_lookup` (`system_basic_auth`). The salted hash is then verified with the constant-time comparison described above. A row that belongs to the URL's system is rejected with 401 if it is revoked or expired, and otherwise accepted, which stamps `last_used_at` on that credential. A live row that belongs to a different system is refused with 403 (see "Cross-system confinement" below). For Basic, the username and password comparisons both always run and are then combined, so timing does not reveal which half was wrong. A key or token that predates migration `0002` has no `lookup_hash` yet and is checked by a fallback scoped to the URL's own system.
5. **Check the service token** when the system's integration type requires one (see "Service token" below). It runs after the partner credential, so a stranger learns nothing about the requirement.
6. **Apply the per-system rate limit** (`external_system.rate_limit_rpm`) only after authentication succeeds.
7. **Check the endpoint name and apply the endpoint limit**, if the system declares any (see "Endpoint names and limits" below).
8. **Run the connector's input checks.** Resolve the handler, read `Idempotency-Key`, parse the JSON body, validate it strictly (see "Strict payloads" below) and only then create the job.

### Failure responses

Every pre-auth failure (missing credential, unknown or deleted system, inactive system, wrong secret, revoked or expired credential, a credential whose owning system is deleted) returns the same `401 {"error":"unauthorized"}`, so a caller cannot probe which system codes exist, which are active, or whether a credential was once valid. The other authentication outcomes are the confinement `403 {"error":"forbidden"}` described next and the service-token responses described under "Service token". The real reason is recorded only in `activity_log`. Unknown and inactive systems also spend one hash computation so timing matches a wrong secret.

### Cross-system confinement

**A credential for Partner A never authenticates a request to Partner B's routes.** The gateway resolves the URL's system first, then identifies the presented credential by its lookup index across all systems. A real credential presented on the wrong route is therefore recognised and refused, instead of being treated as unknown:

- The refusal is `403` only when a *live* credential proves its identity (its salted hash verifies) but belongs to a different system than the URL names. The response body is the generic `{"error":"forbidden"}`. The owning system's code is written only to the activity row (`ownerSystemCode`, with the reason `credential belongs to another system`) and never appears in a response.
- A revoked or expired credential presented on another system is a plain `401`, indistinguishable from a wrong secret.
- A credential whose owning system is soft-deleted counts as no row at all, for API keys, bearer tokens and Basic alike. Presented on another system's route it gets `401`, never `403`: a `403` would tell the holder of a stale credential that it was once real.
- For Basic, `403` needs both the username and the password to match a live row of another system. A username match alone never yields `403`, so the status is no oracle for which usernames exist.
- A key or token issued before migration `0002` has no `lookup_hash` until it is first used. It is checked only against the URL's own system, so on another system it is `401`, not `403`, until a successful use on its own system backfills the index.
- Because a `403` requires a verified secret, it tells nothing to a caller who does not already hold a valid credential. To a holder of a live key it does confirm that the system code in the URL exists; this is accepted by design, since that caller already holds a credential for this platform.

### Lookup indexes and the master-key rotation window

- **`lookup_hash`** (`system_api_key`, `system_bearer_token`, unique): an unsalted SHA-256 of the full secret. This is safe only because these secrets are 192-bit random values generated by the server: there is nothing to guess, so an unsalted digest can be indexed. It never replaces the salted hash, which is still what is verified.
- **`username_lookup`** (`system_basic_auth`, non-unique): a Basic username has low entropy, so a plain digest would let a leaked index be brute-forced back to usernames. The index is an HMAC-SHA256 of the username under a subkey derived from `CREDENTIAL_MASTER_KEY`, so without the key it cannot be reversed. It is not unique because a rotation grace window can hold two rows for one username.
- **Rotation window.** While `CREDENTIAL_MASTER_KEY_PREVIOUS` is set, the gateway computes the username lookup under both keys (current first) and matches a row indexed under either. When a row of the URL's own system authenticates and its stored lookup is not the current-key value, it is re-indexed to the current key. A row that is not used during the window stays indexed under the previous key, so keep `CREDENTIAL_MASTER_KEY_PREVIOUS` set until every Basic credential in use has authenticated once, or reissue it. Once the previous key is removed, a row still indexed under it is no longer found and gets `401`.
- **Rows that predate migration `0002`.** SQL cannot compute the HMAC, so `username_lookup` is NULL for existing Basic rows, and `lookup_hash` is NULL for existing keys and tokens. The gateway verifies such rows only against the URL's own system, and on a match fills in the index (or moves `username_lookup` to the current key), so legacy rows upgrade on first use.

## Service token

Some partners are called by a backend job rather than a person, and a project may want a second, short-lived proof on top of the partner's long-lived key. Each integration type carries an `auth_policy`:

| Policy | Meaning |
|---|---|
| `partner` (default) | The partner credential alone authenticates the call. |
| `partner+service` | The partner credential must pass first, and the request must also carry a valid service token in the `X-Service-Token` header. |

The policy belongs to the type, so every system of that type follows it. It is set when registering a type, or later with `integrationTypes.setAuthPolicy` (`auth-policy:write`, admin only, audited as `integration_type.set_auth_policy`); the console asks for confirmation before switching. Creating a type with a policy stays under `systems:write`, because a new type has no systems. In production, choosing `partner+service` while `SERVICE_JWT_SECRET` is not configured is refused with `409 CONFLICT`, so the policy cannot be switched on into a gateway that would turn everyone away.

The token is a JWT, verified in `core/auth/service-token.ts`:

- **Algorithm.** HS256 only, under `SERVICE_JWT_SECRET`. `none`, HS512 and every other algorithm are refused. A secret shorter than 32 bytes counts as unset.
- **Claims.** `aud` must be `interchange-inbound`. `sub`, `iat` and `exp` are required. `sys` is optional.
- **Lifetime.** `exp - iat` is at most 900 seconds, and a token issued in the future beyond the tolerance is refused. Clock tolerance is 30 seconds.
- **Scope.** When `sys` is present it must equal the system code in the URL. Without `sys`, a token is valid for any system that uses the policy, so mint tokens with `--sys`.

What the caller sees, always after the partner credential has passed:

| Situation | Response | Activity reason (never in the response) |
|---|---|---|
| Header missing | `401 {"error":"unauthorized"}` | missing service token |
| Bad signature, wrong algorithm, wrong audience, missing claim | `401 {"error":"unauthorized"}` | invalid service token |
| Expired | `401 {"error":"unauthorized"}` | service token expired |
| Lifetime over 900 s | `401 {"error":"unauthorized"}` | service token lifetime too long |
| `sys` names another system | `403 {"error":"forbidden"}` | service token for another system |
| Secret unset or under 32 bytes | `503 {"error":"service unavailable"}` | service auth not configured |

The 503 is fail closed: the policy cannot be met, so nobody gets in, and the cause is logged server-side. Claims are never echoed in a response or a reason. On success the activity row's `authMethod` is the partner method with `+service` appended.

**A service token is never accepted as a partner credential.** It proves nothing on its own. A Bearer value that matches no partner credential is also tested against the service secret (only when it looks like a JWT and the secret is configured). If its signature verifies, with claims ignored so an expired or other-system token counts too, the request is refused with `401` and the activity reason "service token presented as partner credential". It is never `403`, and the test runs only after the credential lookup has missed. It exists to make a misconfigured client visible in the log; it does not create a path that authenticates.

**Minting.** `npm run mint:service-token -- --sub worker-1 --sys ACME-ERP --ttl 300` reads `SERVICE_JWT_SECRET` from the environment and prints only the token. `--sub` is required, `--ttl` is 1 to 900 seconds (default 300). Failures go to stderr and never echo the secret.

**Accepted risk: replay inside the lifetime.** There is no `jti` store. Tokens are stateless, so a captured token can be replayed until it expires: at most 900 seconds plus the 30 second tolerance. An attacker who replays one still needs a valid partner credential for the same system, because the service token is a second factor and not a replacement. Keep tokens short (the script defaults to 300 seconds) and send them over https only.

**One secret, no overlap.** There is a single `SERVICE_JWT_SECRET` and no previous-key slot, so changing it invalidates every outstanding token at once. Tokens live minutes, so the practical effect is that callers mint a fresh one.

**Known limits (deferred to P5).** Two limits are accepted until phase P5. First, the `sys` claim is optional, so a token minted without `--sys` is valid for every `partner+service` system, not one. Second, there is one global `SERVICE_JWT_SECRET`, with no per-system key and no overlap slot for rotation. Until P5, always mint with `--sys`, and rotate the secret on any suspicion (see "A leaked `SERVICE_JWT_SECRET`" under the threat model below).

## Deleting and suspending systems

A system is soft-deleted from the console, or with `POST /systems/:id/delete` (`systems:write`), from any status. Delete sets `deleted_at` and `status = SUSPENDED` in one statement, and the audit row records the previous status. Restore clears `deleted_at` and leaves the system `SUSPENDED`, so a restore never resumes live traffic by itself: an operator must set it `ACTIVE`. Restore is refused with `409` while the system's integration type is deleted. Integration types can be soft-deleted too, but not while a non-deleted system uses them, and a deleted type cannot be chosen for a new, imported or duplicated system.

- **Deleted is unknown at the gateway.** The URL lookup excludes deleted systems, so a deleted system's URL gets the same `401` as one that never existed, with the same dummy hash cost.
- **A deleted owner's credential is `401`, never `403`.** The owner join treats a deleted owner as no row, for API keys, bearer tokens and Basic (see "Cross-system confinement").
- **Mutations are refused.** On a deleted system, edit, status change, credential issue, rotate, store and Basic set, endpoint config, health check, duplicate, arming a schedule and replaying a DLQ entry all answer `409 CONFLICT` "system is deleted". Reads, export and credential revoke still work, because revoking only reduces exposure. Jobs, DLQ, audit and activity history of a deleted system stay listed.
- **Suspend and delete stop background work, with no catch-up.**
  - *Schedules.* The tick fires a schedule only for a live, `ACTIVE` system. Any other due schedule has its `next_run_at` advanced to the next cron slot with nothing enqueued, so activating or restoring a system later never produces a burst of fires for slots that passed while it was off.
  - *Queued jobs of a `SUSPENDED` or `SETUP` system are held.* The drain does not claim them: they stay pending with their attempt count unchanged, and they do not take a batch slot from other systems. They run once the system is set `ACTIVE`.
  - *Queued jobs of a deleted system fail without retry* ("system deleted") and land in the DLQ, where replay is refused while the system stays deleted.
- **Deleted codes stay reserved.** `external_system.system_code` and `integration_type.code` stay unique across soft-deleted rows. Creating, importing or duplicating with a deleted system's code answers `409 CONFLICT` "system code already exists (deleted)", and the same rule covers type codes. There is no code reuse on purpose: old credentials, jobs, audit rows and activity rows all point at the code, and reuse would attach a former partner's history to a new one. To use the code again, restore the row.

## Endpoint names and limits

The last segment of `/api/inbound/<systemCode>/<endpoint>` is validated before anything else uses it:

- **Normalisation.** The segment is Unicode NFC, then lower-cased, then must match `^[a-z0-9._-]{1,64}$`. Anything else is `404 {"error":"unknown endpoint"}`.
- **Limits are stored the same way.** A per-endpoint limit (`endpoint_rate_limit`) is an endpoint name plus a requests-per-minute value from 1 to 100000, up to 50 per system. Names are stored lower-cased under a unique index and a CHECK on the same pattern, so `Orders` and `orders` can never be two limits. Save and import apply the same normalisation.
- **Declaring any endpoint limit turns the list into the system's endpoint allowlist.** The segment is free text that reaches the same handler, so a limit keyed on the raw text would be bypassed by respelling it (`Orders`, `orders2`, `orders.`). Instead, once a system has at least one limit row, any segment outside that set is `404` "unknown endpoint". The check runs after authentication, so strangers learn nothing. The request is logged with that reason, creates no job and spends no counter. A system with no rows accepts any valid segment.
- **Counting.** A listed endpoint consumes the bucket `sys:<systemCode>:<endpoint>`. Over the limit the answer is `429` with `Retry-After` and the activity reason "rate limited (endpoint)". The per-system limit still caps everything, and the `X-RateLimit-*` headers report the tightest bucket the request consumed.

## Strict payloads

The gateway validates the body against the connector's zod schema with the top level strict:

- **Unknown top-level keys are refused**, not dropped: `422 {"error":"payload validation failed","unknownKeys":["extra"],"details":{...}}`. Names are sorted, at most 20 are listed and each is cut at 64 characters; `unknownKeysTruncated: true` marks a longer list. Values are never reported. The activity row stores `{"reason":"unknown keys","unknownKeys":[...]}` only, so a secret sent under a misspelled key is not written to the log.
- **Other validation failures** keep zod's `flatten` shape in `details`, but every message comes from one no-echo mapper: an enum failure lists the allowed values and never the submitted one.
- **Only the top level is strict.** Nested objects follow the connector's own schema.
- **Same rule for a manual run.** `jobs.run` in the admin API refuses an unknown payload key with a validation issue `unrecognized_keys` under `payload`. The drain keeps a lenient parse of payloads already stored, so rows queued before this rule never dead-letter on it.
- **Fail closed on the schema itself.** A connector whose input schema is not a plain object, or uses a catch-all, cannot be checked for unknown keys. The gateway answers `500 {"error":"internal error"}` and a registry test asserts every shipped schema is an object.

## Credential lifecycle

Inbound credentials (API key, bearer token, Basic) and outbound credentials (encrypted, held to call a partner) are managed from the system's detail page in the console, or through the admin API under `/api/admin`. Every mutation writes an `admin_audit_log` row (`api_key.issue`, `bearer_token.issue`, `basic_auth.set`, `credential.store`, `credential.rotate`, `credential.revoke`, `credential.reveal`). No response and no audit row carries a stored secret, hash, salt or Basic username.

- **Kinds and status.** The kinds are `api_key`, `bearer`, `basic` and `outbound`. Status is derived from timestamps by one shared function, so the server, the list counts and the demo agree: `REVOKED` (`revoked_at` is set; wins over everything), `EXPIRED` (`expires_at` has passed, which includes the end of a rotation grace), `ROTATING` (a successor exists and the row is still inside its grace), otherwise `ACTIVE`. Only `ACTIVE` and `ROTATING` credentials authenticate.
- **Issue.** `POST /systems/:id/api-keys`, `/bearer-tokens` and `/basic-auth` need `credentials:write`. The server generates the secret (`ic_` plus 24 random bytes, base64url) and returns it once. For Basic the caller picks the username and the server generates the password. `label` (1 to 100 characters) and `expiresInDays` (1 to 3650) are optional: an omitted label becomes a generated `key-`, `token-` or `basic-` name, and an omitted expiry means the credential does not expire. An outbound credential is stored with `POST /systems/:id/credentials`; its label defaults to its credential type, which is the key connectors read it under.
- **Label uniqueness.** A label is unique per system among credentials that are neither revoked nor rotated away (outbound: not revoked), enforced by a partial unique index per table. Issuing a label that is already active or rotating answers `409 CONFLICT`, including when two requests race. This also means two active outbound credentials can never share a label, so one cannot shadow another when connector config is resolved. An index predicate cannot use the current time, so an expired but unrevoked row still holds its label; issuing that label first stamps the expired row revoked, in the same transaction.
- **Expiry.** The gateway rejects an expired credential with `401`, and the list reports it as `EXPIRED`. The API cannot create an already-expired credential (the minimum is one day).
- **Rotate.** `POST /credentials/:kind/:credId/rotate`. For inbound kinds it creates a successor with the same label and a new server-generated secret, returned once; Basic keeps its username and gets a new password. `graceMinutes` (0 to 1440, default 0) decides what happens to the old row. With 0 it is revoked at once. Otherwise it keeps working until the grace ends (or until its own earlier expiry), shows as `ROTATING`, then becomes `EXPIRED`. The successor inherits the original lifetime, so a 30 day key rotates into a 30 day key and a key without expiry stays without one. The old row is updated before the successor is inserted, in one transaction, because the unique index would otherwise reject the new row. A revoked, expired or already rotated credential answers `409`. An outbound rotation needs a new `secret`, re-encrypts it in place under the same label and stamps `rotated_at`; it has no grace window. An inbound rotation refuses a supplied `secret`.
- **Revoke.** `POST /credentials/:kind/:credId/revoke` sets `revoked_at`. It takes effect on the next request: the gateway answers `401`, and a revoked outbound credential drops out of the resolved connector config. Revoking a credential also revokes any predecessor still inside its rotation grace, so revoking the successor closes the whole chain. It is deliberately not idempotent: a second revoke answers `409`, so a double click or a stale screen is visible rather than silently accepted.
- **Reveal (off by default).** Only outbound credentials can be revealed; inbound secrets are hashed and cannot be recovered. Two independent switches must both be on: the user holds `credentials:reveal`, and the server has `CREDENTIALS_REVEAL_ENABLED=1`. The second exists because the `admin` role holds `*`, so a permission alone could never make reveal off by default. With the variable unset, even an admin gets `403 FORBIDDEN` ("Reveal is disabled"), checked before any lookup. A revoked credential answers `409`. Reveal is fail-closed: the plaintext is returned only after the `credential.reveal` audit row (the label, never the value) has committed, so a reveal that cannot be audited does not happen. Reveal also has its own per-user throttle, `CREDENTIALS_REVEAL_LIMIT` reveals per 10 minutes (default 10), on top of the admin rate limit. `GET /api/admin/me` reports `features.credentialReveal`, so the console hides the Reveal action when the server has it off.
- **Listing.** `GET /systems/:id/credentials` (`systems:read`) takes `status`, `kind` and `offset`, and returns at most 50 rows with a `total` for the same filters. Rows are mapped through the status function first and filtered second, so a status count and the list behind it always agree. A response never includes a secret, hash or salt. `GET /credentials/expiring` (`systems:read`) lists the `ACTIVE` inbound credentials of live systems that expire within the next 30 days, soonest first, and the dashboard's "expiring" count is that list's total.
- **Console.** The system detail page (`systems/<id>`, reached from the Systems list) has an Overview tab with Active, Rotating, Expired and Revoked counts that link to the Credentials tab filtered by that status. The Credentials tab lists the credentials and offers Issue, Rotate (with a grace field), Revoke and, when enabled, Reveal. A new secret is shown once. A viewer sees no mutation controls.

## Certificates and SFTP

Two per-system records hold sensitive material that is not a gateway credential: certificates (`system_certificate`) and one SFTP connection config (`sftp_config`). Both are managed from the system detail page (Certificates and SFTP tabs) or the admin API. Reads need `systems:read`; every mutation is permission-gated and writes an `admin_audit_log` row.

**Certificates** (`certificates:write`; audit `certificate.upload`, `certificate.revoke`).

- **Parsed defensively.** An upload takes a label, one PEM certificate and, optionally, its private key. The certificate reader (`core/pure/x509-lite.ts`) never throws: exactly one `CERTIFICATE` block, strict base64, at most 16384 characters, DER capped at 12288 bytes, depth and length checks at every level. Garbage, a bundle, a private key in the certificate field or an oversized body is `400 VALIDATION` (or `413` past the 64 KB body cap), never `500`.
- **The key is write-only.** It must be one unencrypted PEM block (a passphrase-protected key is refused) and must match the certificate, checked server side with `X509Certificate.checkPrivateKey`. Every `node:crypto` call on the pair sits in one `try`, and any throw becomes a fixed validation message. The key is stored only through `encryptSecret` (`enc_key_*`). No endpoint, export or demo payload returns it: views carry `hasPrivateKey` and never the key, the certificate PEM or an `enc_` field. Nothing consumes the key yet, so there is no decrypt path for it. The audit row records the label, the fingerprint and whether a key was stored.
- **Status is derived, never stored.** One function (`core/pure/certificate-status.ts`) decides, first match wins: `REVOKED`, `EXPIRED`, `NOT_YET_VALID`, `EXPIRING` (ends within 30 days), `VALID`. The five count cards on the Certificates tab and the lists they open use that one function, so a count equals its list. A list with no status filter hides revoked rows; they appear only under `REVOKED`.
- **Limits.** A system holds at most 50 active certificates. An active label and an active fingerprint are each unique per system (partial unique indexes, `409 CONFLICT`, including under a race). Revoke sets `revoked_at` and is not idempotent (a second revoke is `409`). Upload needs a live system; list and revoke only need it to exist, so access can always be withdrawn.
- **Demo.** The in-browser demo keeps only the `hasPrivateKey` boolean and skips the key-match check.

**SFTP** (`sftp:write` to save and pin, `sftp:test` to test; audit `sftp.save`, `sftp.test`).

- **Config.** Host, port, username, auth method (`password` or `privateKey`), a write-only secret and optional inbox, outbox and archive paths (absolute, no `..` segment). The secret is encrypted into `enc_secret_*` and views carry `hasSecret` only. A private key must parse with `ssh2` and must not need a passphrase. The `sftp.save` audit row records the host, port, method and whether the secret or the pin changed, never the secret.
- **A pinned host key, or no authentication.** `host_key_sha256` holds the server's key in OpenSSH form (`SHA256:` and 43 base64 characters). With no pin, or a different key, the test stops at stage `hostkey` ("host key not pinned", "host key does not match the pinned key") before any authentication message is sent and before the secret is decrypted. An unpinned test returns the fingerprint the server presented, so an administrator can check it out of band and pin it. Trust on first use is deliberately not offered: it would pin whoever answers first, then send them the password. Changing the host or port clears the pin unless the same save supplies a new one. Changing the host, port or username also clears the stored secret unless the same save supplies a new one, so a stored password or key can never be sent to a new target. An omitted port keeps the stored port (22 applies only to a first save).
- **Single-resolution SSRF with a socket handoff.** The test calls `resolvePublicHost` (`core/net/ssrf.ts`) once: the static host rules, then one DNS resolution in which every answer must be public. The probe dials one of those checked addresses (IPv4 first, the next answer on a connect error) and gives `ssh2` the open socket. `ssh2` never receives the host name, so it can never resolve it a second time and DNS rebinding has nothing to exploit. The classifier is the one `safeFetch` uses.
- **Fixed messages, one budget.** Stages are `preflight`, `connect`, `handshake`, `hostkey`, `auth` and `done`. Every message comes from a fixed table; `ssh2` and socket errors are mapped by level and code, and no error text, secret or key is copied out. One 10 second budget covers all stages. There is a single authentication attempt, no SFTP subsystem is opened and nothing is transferred. If `ssh2` cannot load, only this test fails ("ssh client unavailable").
- **Rate limit.** 5 tests per user and 10 per system per minute (constants, not environment variables), checked before any DNS lookup or decrypt: `429 RATE_LIMITED` with `Retry-After`. The `sftp.test` audit row (`ok` and `stage` only) is written only for a test that ran. Save and test need a live system.
- **Demo.** The demo's test is a simulation: it opens no socket and has no limiter.

## Outbound request guard (SSRF)

Interchange makes requests to URLs an administrator typed: a system's `baseUrl` (outbound connectors) and its `healthUrl`. A mistyped or malicious URL that points at loopback, a private network or a cloud metadata address would turn the platform into a proxy into places it can reach and the partner cannot. Every such HTTP request goes through `safeFetch` in `core/net/ssrf.ts`, and a connector must not call `fetch` itself (see `docs/connectors.md`).

The SFTP host an administrator typed is the second guarded outbound path. It is not HTTP, so it does not use `safeFetch`: `resolvePublicHost` in the same file resolves the name once with the same classifier, refuses unless every answer is public, and the probe dials the checked address and hands the open socket to the SSH client (see "Certificates and SFTP").

**What is blocked.** The classifier is `core/pure/address-policy.ts`, pure and shared with the in-browser demo:

- IPv4: `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10` (carrier-grade NAT), `127.0.0.0/8`, `169.254.0.0/16` (link-local, which includes the cloud metadata address), `172.16.0.0/12`, `192.0.0.0/24`, `192.0.2.0/24`, `192.88.99.0/24` (6to4 relay), `192.168.0.0/16`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`, `224.0.0.0/4` (multicast) and `240.0.0.0/4` (reserved, includes broadcast).
- IPv6: `::/96` (unspecified, loopback and IPv4-compatible), `fc00::/7`, `fe80::/10`, `fec0::/10`, `ff00::/8`, `2001::/32` (Teredo), `2001:db8::/32`, `2002::/16` (6to4) and `64:ff9b:1::/48`. IPv4-mapped (`::ffff:a.b.c.d`, also written `::ffff:7f00:1`) and NAT64 (`64:ff9b::/96`) addresses are judged by the IPv4 address inside them.
- Names: `localhost`, `*.localhost`, `*.local` and `*.internal`.
- Anything that does not parse as a canonical address is `blocked:invalid`, never public. Dotted quads with leading zeros are refused because some stacks read them as octal. Numeric spellings such as `2130706433` and `0x7f.1` are normalised to `127.0.0.1` by the URL parser before the check, so they are caught.

**When it is checked.**

1. *At save.* `baseUrl` and `healthUrl` get syntax, scheme, literal-address and reserved-name checks, with no DNS, so an obvious mistake fails in the form with the issue code `ssrf_blocked`. The demo applies the same rules. On the server, http is refused at save when `NODE_ENV` is `production`; the demo always refuses it. A hostname that merely resolves to a private address is not caught at save, only at use.
2. *Preflight, at use.* In production only https is allowed. The host is resolved to every address, and the request is refused if any one of them is blocked.
3. *At connect time.* The HTTP client dials through a lookup that resolves again and refuses any blocked answer, so the address actually dialled is the one that was judged. A DNS answer that is public at preflight and private at connect (rebinding) fails with `BLOCKED`. An IP-literal host skips that lookup in the socket layer, which is why the preflight judges literals itself.
4. *Redirects are never followed.* A `3xx` is returned to the caller, and the `rest-outbound` connector treats it as a non-retryable failure.
5. *Every call is bounded by a timeout:* the system's `timeout_ms` for connectors, 5 seconds for the health check.

**Fixed error codes.** Failures surface as one of seven codes with a fixed message. Nothing the upstream said (resolver text, socket errors, TLS details, response bodies) is copied into an error, a job log or a health reason, so callers can log and store them safely.

| Code | Meaning | `rest-outbound` result |
|---|---|---|
| `BLOCKED` | A blocked address or reserved name | `FAILED`, not retried |
| `INVALID_URL` | Not an absolute URL, scheme not allowed, http in production, no host | `FAILED`, not retried |
| `DNS` | The host does not exist or the resolver refused permanently | `FAILED`, not retried |
| `DNS_TEMPORARY` | A temporary resolver failure, such as `EAI_AGAIN` | `RETRY` with backoff; `DLQ` as "retries exhausted" after the attempt limit |
| `TIMEOUT` | The timeout elapsed | `RETRY` |
| `CONNECT` | The connection failed | `RETRY` |
| `TLS` | Handshake or certificate failure | `RETRY` |

The first three are configuration problems that a retry cannot fix. The last four are treated as transient. The health check does not retry: on a temporary resolver failure it records `DOWN` with the reason `DNS_TEMPORARY`, and the operator presses Check now again.

**Consumers.** The `rest-outbound` connector and the system health check (`systems.checkHealth`, `systems:write`): a `GET` with a 5 second timeout whose body is cancelled unread. A `2xx` stores `UP`; anything else stores `DOWN` with a `health_reason` that is a guard code or `HTTP_<status>`, never an upstream body or message. Any future outbound connector must use `safeFetch` too. A test-only policy override exists so tests can run positive controls against a local server; it is ignored when `NODE_ENV` is `production` and no route can reach it.

## Importing and exporting systems

`GET /systems/:id/export` returns a JSON document describing one system's configuration, `POST /systems/import` creates a system from one, and `POST /systems/:id/duplicate` copies one inside the platform.

- **No secrets, by construction.** The export query reads no credential, certificate or SFTP table and the document schema has no field for one. A duplicate never reads those tables either, so the copy starts with no credentials, no certificates and no SFTP config. Imported and duplicated systems land in `SETUP`, so nothing is reachable until credentials are issued and an operator activates the system.
- **Strict at every level.** The document schema refuses an unknown key anywhere: a smuggled `apiKey` or an extra `endpointConfig` field is a validation error with an `unrecognized_keys` issue at that path, not a silently ignored field.
- **`endpointConfig.settings` is free-form.** It is exported as stored, so keep secrets out of it. Store them with "Store credential", which encrypts them in the vault.
- **Audited.** Import, duplicate and delete write `system.import`, `system.duplicate` and `system.delete` rows.

## RBAC (admin console)

Admin access uses `next-auth` with a credentials provider backed by Interchange's own `user` table (`core/auth/options.ts`, `core/auth/login.ts`). Session strategy is JWT with an 8 hour lifetime (session and token `maxAge`); role names, permissions and the user's `token_version` are resolved at login and carried in the token, but nothing authoritative reads them from there.

Authorization is by **permission**, not role name. The catalog is a constant in `core/auth/permissions.ts` (`systems:read`, `systems:write`, `credentials:write`, `credentials:reveal`, `certificates:write`, `sftp:write`, `sftp:test`, `jobs:read`, `jobs:run`, `jobs:retry`, `schedules:read`, `schedules:write`, `audit:read`, `users:read`, `users:write`, `roles:write`, `settings:write`, `auth-policy:write`), stored on `role.permissions`, with `*` meaning superadmin. No default role lists `auth-policy:write`, so only `admin` (the `*` wildcard) holds it. `jobs:run` and `jobs:retry` are in the catalog and granted to `operator`, and the endpoints that check them are `jobs.run` (manual run, `jobs:run`) and `dlq.replay` (DLQ replay, `jobs:retry`).

The authoritative gate is `requirePermission(perm)` in `core/auth/rbac.ts`, run by `withAdmin` (`core/http/with-admin.ts`) on every `/api/admin/*` request, before any input is parsed: an unauthenticated, disabled or stale-token caller gets 401, and a signed-in caller without the endpoint's permission gets 403. The admin layout does a DB-backed check too (`requireActiveUser()`, redirecting to `/login`), and the client hides navigation entries the user cannot open. Neither trusts the JWT: on every call each re-reads the user's current status, `token_version` and role permissions from the database in one join. A role change, a disabled account, or a `token_version` bump (the mechanism for "log out everywhere") therefore takes effect on the next request, and a token carrying a stale `token_version` is rejected.

The seeded roles (`db/seed.ts`) are `admin` (`*`), `operator` (systems read/write, credentials write, certificates write, SFTP write and test, jobs read/run/retry, schedules read/write, audit read) and `viewer` (the `:read` permissions for systems, jobs, schedules and audit). `admin` holds the three new permissions through `*` and `viewer` holds none. `operator` gets them because it already holds `credentials:write`, and the SFTP test is rate limited, SSRF guarded and sends nothing to an unpinned host. In the console, Upload, Revoke, Save, Pin and Test connection render only when the user holds the matching permission; the server check is the real gate.

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
4. applies the env-list CORS rule below to `/api/*`, except `/api/inbound/*`, which it leaves alone: the inbound route answers CORS itself, per system.

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

CORS is **denied by default**, in every environment, and two rules apply depending on the path. Neither ever enables credentialed requests, and `/api/admin/*` is outside both.

**1. The env list: every `/api/*` path except `/api/inbound/*` and `/api/admin/*`.**

- It applies only to a cross-origin request (an `Origin` header that differs from the request's own origin). Same-origin requests and requests with no `Origin` are untouched.
- `CORS_ALLOWED_ORIGINS` is a comma-separated list of exact origins (scheme, host and port). Unset or empty allows nothing. Entries that are not an `http` or `https` origin, including `*`, are dropped, so a wildcard cannot be configured by accident.
- A request from an allowed origin gets `Access-Control-Allow-Origin` set to that exact origin, plus `Vary: Origin`. Any other cross-origin request gets no `Access-Control-Allow-Origin`, so the browser blocks the response.
- On these paths a cross-origin `OPTIONS` preflight is answered by the proxy itself with `204`. For an allowed origin it also carries `Access-Control-Allow-Methods: POST, OPTIONS`, `Access-Control-Allow-Headers: Authorization, Content-Type, X-API-Key, X-Request-ID, Idempotency-Key, X-Service-Token` (one shared constant for both rules) and `Access-Control-Max-Age: 600`; for any other origin it carries no `Access-Control-Allow-*` header.
- `/api/admin/*` never gets any CORS header, whatever the allowlist says. The admin API does its own same-origin CSRF check and is not meant for other origins. The path is normalised (decoded, lower-cased, repeated slashes collapsed) before this check, and an undecodable path is treated as admin.
- Credentialed CORS (`Access-Control-Allow-Credentials`) is never enabled, on any path.

**2. The system's own list: `/api/inbound/*`.** The proxy neither answers nor decorates these paths. The inbound route decides per system, because a partner's browser origin belongs to that partner and not to the whole platform.

- Each system has a `cors_origins` list, edited on the CORS tab of its detail page (`systems.update`, `systems:write`). Entries follow the same rules as the env list: exact `http` or `https` origins, `*` impossible, at most 20. An invalid entry is rejected at save with a validation error at its index, not silently dropped. Every list starts empty, so the default is still deny.
- **Preflight.** A cross-origin `OPTIONS` first spends the per-IP rate limit, then looks up the system by code. Only a live (not deleted), `ACTIVE` system whose list contains the origin gets `204` with `Access-Control-Allow-Origin` set to that exact origin, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers` (the headers the gateway reads, now including `X-Service-Token`), `Access-Control-Max-Age` and `Vary: Origin`.
- **The env list is a ceiling.** When `CORS_ALLOWED_ORIGINS` is set, an origin must be in both the system's list and the env list. When it is unset, the system's list governs alone, so a partner origin needs one edit, not two.
- **No oracle.** An unknown system, a deleted system, a system that is not `ACTIVE`, an origin that is not listed, and an origin the env ceiling excludes all get the same answer: `204` with `Vary: Origin` and no `Access-Control-Allow-Origin`. A script cannot use preflight to learn which system codes exist or which origins are registered.
- **Actual responses.** Every `POST` response carries `Vary: Origin`. It carries `Access-Control-Allow-Origin` (and `Access-Control-Expose-Headers` for `Retry-After`, the `X-RateLimit-*` set, `X-Request-ID` and `Idempotent-Replayed`) only once the partner credential has authenticated and the origin is allowed. A `401` from the partner credential check never carries it, so a browser caller with a bad key sees an opaque CORS error rather than a hint that the system exists.
- **CORS is not authentication.** An allowed origin only lets browser script read the response; every request still needs the partner credential. List a browser origin only when partner credentials are meant to be used from browser code.

## Job endpoint authentication

`/api/jobs/tick`, `/api/jobs/dispatch` and `/api/jobs/process` accept only requests signed by Upstash QStash. The `Upstash-Signature` JWT is verified against the raw body with `QSTASH_CURRENT_SIGNING_KEY` or `QSTASH_NEXT_SIGNING_KEY` (`core/jobs/verify-qstash.ts`). With the keys unset the endpoints return 503 (fail closed); The verifier is given the expected url (`APP_URL` plus the request path), so a signature for one endpoint is not accepted on another. `QSTASH_DEV_BYPASS=1` is honoured only when `NODE_ENV` is not `production`. The SDK's own `QSTASH_DEV` mode, which would substitute public development keys, is disabled.

## Rate limiting and activity logging

Inbound traffic is limited per client IP (taken from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, validated as an IP address, else `unknown`) (`RATE_LIMIT_IP_RPM`, default 250 per minute) per system (`external_system.rate_limit_rpm`) and, for a system that declares endpoint limits, per endpoint (see "Endpoint names and limits"), using one atomic upsert per bucket per request against `rate_limit_bucket`. A rejected call returns 429 with `Retry-After`. The limiter fails open if the database is unreachable (a logged warning), trading brief over-admission for availability. Every inbound outcome, including 401, 429 and validation failures, writes an `activity_log` row with the request id, status, timing and both payloads; secret-looking body keys (case-insensitive, recursive) are redacted, rows older than `ACTIVITY_RETENTION_DAYS` (default 30) are pruned by the tick, and credentials or `Authorization` headers are never stored. A 422 caused by unknown payload keys stores the key names only, never the values.

### Rate-limit monitor

`GET /rate-limits` (`systems:read`, the "Rate limits" page) shows the current one-minute window's gateway buckets and the `429` responses of the last hour.

- **Gateway buckets only.** Only `ip:` and `sys:` buckets (system and endpoint) are shown. `admin:*` buckets are never returned, because their keys carry user ids.
- **IPs are truncated on the server.** Client addresses are personal data and the page is open to the read-only viewer role, so every address in the monitor (bucket scope and rejected events) is cut to its /24 (IPv4) or /48 (IPv6) before it leaves the endpoint, in the live console and the demo alike. `ip:` buckets are grouped by that prefix: one row per prefix with the summed count, the number of addresses, and the highest single-address count measured against the per-IP limit, so a prefix never shows as one bucket with a misleading ratio. An IPv4-mapped IPv6 address is unwrapped before truncation, and the `unknown` sentinel is shown as is.
- **Rejected events** come from `activity_log` rows with `direction = 'inbound'`, not a test call, and response code 429 in the last hour, newest first, 50 per page. One helper counts them for both the list total and the dashboard card, so an outbound partner 429 or a test row can never inflate either.

## Audit logging

Every privileged admin mutation writes a row to `admin_audit_log` via `auditLog()` in `core/audit/log.ts`: who (`userId`), what (`action`), and on what (`targetTable`, `targetId`, and a `diff` of the change). Failed sign-ins are recorded too (`auth.login_failed`). Audit writes are fire-and-forget: a failure to write the audit row never blocks or rolls back the mutation itself, so an audit-log outage cannot be used to block legitimate admin operations, though it does mean audit completeness is best-effort rather than transactional. This same fire-and-forget principle applies to `job_log` writes (`makeJobLogger()` in `core/jobs/lifecycle.ts`): a logging failure must never break a job.

## Dormant posture as a security property

The provisioned-dormant default is described in `docs/architecture.md` as a cost-control property; it is equally a security property:

- A freshly deployed Interchange instance, even one seeded with real connector types and even one where an operator has (prematurely, or by mistake) issued API keys for a partner system, **rejects all inbound traffic for that system** until an operator explicitly sets `status = 'ACTIVE'`. There is no state between "provisioned" and "live" where a half-configured system quietly accepts requests.
- No outbound schedule fires and no drain runs on a timer until QStash is explicitly armed (`QSTASH_TOKEN` set and the platform tick armed from the Schedules page; arming a single schedule registers nothing with QStash; the job endpoints additionally require signed requests). A deploy with a misconfigured or forgotten schedule does not leak outbound traffic; it does nothing.
- Leaving the live state also stops a system's background work. A `SUSPENDED` or `SETUP` system fires no schedules and has its queued jobs held, and a deleted system's queued jobs fail; nothing catches up when it is activated or restored (see "Deleting and suspending systems").
- This means the platform's default failure mode, for both a fresh deploy and a partially configured one, is silence rather than partial exposure.

## Threat model: a compromised partner credential

This section describes what an attacker who obtains a valid, active API key or bearer token for one partner system can and cannot do.

**Can do:**

- Call any inbound endpoint (`/api/inbound/<systemCode>/<endpoint>`) that belongs to *that specific system*, as many times as the per-system rate limit and the connector's own logic allow.
- Submit any payload that satisfies the connector's zod `inputSchema` with no unknown top-level keys (unknown keys are refused, see "Strict payloads"). If the schema is permissive (for example, the `rest-inbound` template accepts an arbitrary `data` record), the attacker can submit arbitrary data within that shape.
- Cause jobs to be created and connector `execute()` logic to run for that system, including any side effects that connector performs (writes, downstream calls, file drops).
- Read whatever the connector's response returns to the caller (the `output` of a successful `execute()`, or error detail on a validation failure).

**Cannot do:**

- Authenticate as, or submit requests on behalf of, any *other* partner system. A stolen credential is scoped to exactly one `external_system`: presented on another system's route it is recognised and refused with 403 (logged with the owning system in the activity row), never accepted. See "Inbound authentication and cross-system confinement" above.
- Decrypt any outbound credential Interchange holds for that system, or any other system. Inbound API keys and bearer tokens are hashed, not encrypted; there is no reverse operation from a hash back to the original secret, so possessing (or even fully dumping) the `system_api_key` table does not expose usable plaintext credentials (the unsalted `lookup_hash` beside the salted hash is a SHA-256 of a 192-bit random secret, so it does not change that), and the outbound `system_credential` vault is a separate, independently protected store.
- Call a system whose integration type uses the `partner+service` policy with the partner credential alone: the service token is required as well, and it cannot be presented as the partner credential.
- Reach a deleted system. Its URL is an unknown system and its credentials are `401`.
- Reactivate a `SUSPENDED` or `SETUP` system. Status is controlled only through the admin console (`requirePermission()`-gated), never by the inbound path itself.
- Access the admin console. Partner credentials and admin user sessions are entirely separate authentication mechanisms (`with-partner-auth.ts` vs `next-auth` + `rbac.ts`); a partner API key is never accepted as an admin session token.
- Erase evidence of its own activity. Every request produces an `activity_log` row, and every request that reaches a connector also produces a `job` row and associated `job_log` entries; these are platform-owned and not writable by the caller.

**Net exposure of a single compromised credential**: everything one partner system's own connector logic can be made to do, once, repeatedly, until the key is revoked. It does not expose other partners, does not expose the credential vault, and does not grant any administrative capability. Revoking the specific credential, from the system's Credentials tab in the console or with `POST /api/admin/credentials/:kind/:credId/revoke`, sets `revoked_at` and immediately closes that path without affecting any other system. Rotating with a grace of 0 does the same while issuing a replacement, and a credential with an expiry stops working on its own at `expires_at`.

**A leaked `SERVICE_JWT_SECRET`** lets its holder mint service tokens for the systems that use the `partner+service` policy, and for any of them when `sys` is omitted. It does not authenticate anything by itself: the partner credential is still required. Rotate the secret (all outstanding tokens stop working) and review the activity log for `+service` calls.
