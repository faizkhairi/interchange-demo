# Caller authentication

Last updated: 2026-11-16

This document explains how a partner that calls `POST /api/inbound/<systemCode>/<endpoint>` proves who it is. It covers the three partner credential kinds (API key, Bearer token, Basic), the optional per-system service key that signs a short-lived token, the order in which the gateway checks things, and why every failure looks the same to a stranger. Admin console sign-in is a separate mechanism and is covered in [security.md](security.md#rbac-admin-console). Rate limits on these routes are in [rate-limiting.md](rate-limiting.md).

## What the code supports

There are two layers of caller credential, and they are checked in this order:

1. **A partner credential.** Always required. One of an API key, a Bearer token or a Basic username and password. The credentials of one partner system never authenticate a request to another system's route.
2. **A service token.** Required only when the system's integration type has the `partner+service` policy. A short-lived JWT in the `X-Service-Token` header, signed with one of the system's own service keys.

There is **no mTLS**. The gateway reads a credential only from the `x-api-key` header or the `Authorization` header (`core/auth/with-partner-auth.ts:75-85`), and nothing in the repository terminates or checks a client TLS certificate. The Certificates tab in the console stores certificates and alerts before they expire; it does not authenticate callers (see [security.md](security.md#certificates-and-sftp)). If a partner needs mTLS, terminate it in front of the platform.

## The gateway order

The route handler documents its own order in a header comment (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:35-43`). The authentication part is:

| Step | What happens | Failure | Source |
|---|---|---|---|
| 0 | The per-IP rate limit is spent, before any credential work | `429` | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:263-269` |
| 1a | Extract the credential from `x-api-key`, then from `Authorization: Bearer ...` or `Authorization: Basic ...` | `401`, reason "missing credential" | `core/auth/with-partner-auth.ts:75-85`, `core/auth/with-partner-auth.ts:334-335` |
| 1b | Resolve the system named in the URL, excluding soft-deleted systems, and join its integration type | `401`, reason "unknown system" | `core/auth/with-partner-auth.ts:341-352` |
| 1c | Require the system to be `ACTIVE` | `401`, reason "system not active" | `core/auth/with-partner-auth.ts:354-358` |
| 1d | Find the credential row, then verify it with a constant-time comparison | `401` (wrong, revoked, expired) or `403` (a live credential of another system) | `core/auth/with-partner-auth.ts:197-254`, `core/auth/with-partner-auth.ts:256-326`, `core/auth/with-partner-auth.ts:374-376` |
| 2 | If the type's policy is `partner+service`, require and verify `X-Service-Token` | `401` or `403` | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:295-310` |
| 3 | The per-system rate limit, then the endpoint check and per-endpoint limit | `429` or `404` | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:321-363` |

The service token runs after the partner credential, so a caller without a valid partner credential never learns that a service token is required (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:295-296`). The per-system limit also runs only after authentication succeeds, so a stranger cannot burn a partner's quota (`route.ts:321`).

## Partner credentials

All three kinds are issued from the system's Credentials tab in the console, or through the admin API. The server generates the secret and shows it once. Inbound secrets are stored hashed and cannot be revealed afterward (see [security.md](security.md#inbound-credentials-hashed-not-encrypted)).

### API key and Bearer token

- Send an API key as `X-API-Key: <key>`, or a Bearer token as `Authorization: Bearer <token>` (`core/auth/with-partner-auth.ts:76-82`).
- The gateway first finds the row by an indexed `lookup_hash` of the secret across all systems, then verifies the salted hash with the constant-time `verifySecret` (`core/auth/with-partner-auth.ts:206-211`).
- A row that belongs to the URL's system is accepted if it is live, which stamps `last_used_at` (`core/auth/with-partner-auth.ts:218-220`). A revoked or expired row is `401` (`core/auth/with-partner-auth.ts:118-123`, `core/auth/with-partner-auth.ts:218`).
- A row from before the lookup index existed is checked only against the URL's own system and then indexed on first use (`core/auth/with-partner-auth.ts:223-253`). Because such a row (from before migration `0002`) has no `lookup_hash` yet, it cannot be recognised on another system's route: there it is a plain `401`, not `403`, until a successful use on its own system fills in the index (`core/auth/with-partner-auth.ts:226`, `core/auth/with-partner-auth.ts:252`).

### Basic

- Send `Authorization: Basic <base64(username:password)>`. The server chooses the password; the caller chooses the username when the credential is created.
- The row is found by a keyed lookup of the username, then both the username and the password comparisons always run and are combined without short-circuiting, so timing does not reveal which half was wrong (`core/auth/with-partner-auth.ts:264-298`).
- A username match alone never produces a `403`. A `403` needs both halves to match a live row of another system (`core/auth/with-partner-auth.ts:320-324`).

### Cross-system confinement

The URL's system is resolved first and the credential is then looked up across all systems. A real credential presented on the wrong route is recognised and refused:

- `403 {"error":"forbidden"}` only when a live credential's secret verifies but its owner is a different system (`core/auth/with-partner-auth.ts:213-216`). The owning system's code goes to the activity row only (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:278-281`).
- A revoked or expired foreign credential is a plain `401`, the same as a wrong secret (`core/auth/with-partner-auth.ts:214-215`).
- A credential whose owning system is soft-deleted counts as no row, so it gets `401` and never `403` (`core/auth/with-partner-auth.ts:151-152`).

## Service tokens (per-system service keys)

A type with the `partner+service` policy needs a second, short-lived proof on top of the partner credential (`core/auth/with-partner-auth.ts:366`, `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:297`). The proof is a JWT sent in the `X-Service-Token` header (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:298`), verified by `verifyServiceToken` (`core/auth/service-token.ts:133-174`).

### Token rules

| Rule | Value | Source |
|---|---|---|
| Algorithm | `HS256` only. `none`, HS512, RS256 and any other algorithm are refused | `core/auth/service-token.ts:72`, `core/auth/service-token.ts:150` |
| Header `kid` | Must be a uuid: the id of a service key row of this system | `core/auth/service-token.ts:73`, `core/auth/service-key.ts:43` |
| Audience | `interchange-inbound` | `core/auth/service-key.ts:15`, `core/auth/service-token.ts:151` |
| Required claims | `sub`, `iat`, `exp`, `sys` | `core/auth/service-token.ts:153` |
| Lifetime cap | `exp - iat` at most 900 seconds | `core/auth/service-key.ts:16`, `core/auth/service-token.ts:158-160` |
| Clock tolerance | 30 seconds | `core/auth/service-token.ts:48`, `core/auth/service-token.ts:152` |
| Future `iat` | Refused when it is more than the tolerance ahead of now | `core/auth/service-token.ts:161-162` |
| `sys` claim | Must equal the system code in the URL, otherwise `403` | `core/auth/service-token.ts:163-166` |

The header is read before any verification and only to pick the key row (`core/auth/service-token.ts:67-77`). The `sys` check runs after the signature verified, so a token signed with this system's own key that names another system gets the `403` (`core/auth/service-token.ts:148-166`).

### Key lookup

The key row is selected by `kid`, the system named in the URL, `revoked_at` being null and `expires_at` being unset or in the future (`core/auth/service-token.ts:82-95`). An unknown `kid`, another system's `kid`, a revoked key and a key past its grace end all give the same answer (`core/auth/service-token.ts:143-144`). There is no cache: the row is read on every call, so a revoke or the end of a grace window takes effect on the next call on every instance (`core/auth/service-token.ts:42-45`). A key that cannot be decrypted is a `401`, not a `500` (`core/auth/service-token.ts:97-110`, `core/auth/service-token.ts:145-146`).

### Issue, rotate and revoke

- **Issue.** `POST /api/admin/systems/:id/service-keys` (`src/admin-api/endpoints/credentials.ts:33`), or "Issue service key" on the system's Credentials tab. The button is shown only to a user with `credentials:write`, for a live system whose type is inbound and `partner+service` (`src/features/systems/CredentialsTab.tsx:911`, `src/features/systems/use-system-type.ts:10-12`). The body takes an optional `label` and an optional `expiresInDays`, and the answer is `{kid, plaintext}` (`src/admin-api/endpoints/credentials.ts:35-36`). The service generates 32 random bytes, returns their 43 character base64url text once, and stores it encrypted with its id as additional authenticated data (`core/services/credentials.ts:207-216`, `core/auth/service-key.ts:21-23`, `core/auth/service-key.ts:35-41`). The answer includes the `kid` the caller must put in the token header. A refused issue is a `409`: "this system's type does not use service tokens" for a type that is not inbound `partner+service`, so an outbound type never takes a key (`core/services/credentials.ts:229-231`); "system is deleted" for a deleted system (`core/services/credentials.ts:211`, `core/services/systems.ts:301`); and "a credential with this label is already active on the system" when an active key of the system already has the label (`core/services/credentials.ts:100`, `core/services/credentials.ts:234`).
- **No reveal.** `credentials.reveal` reads only the outbound credential table, so a service key id answers `404` "credential not found" there (once reveal is enabled at all). A lost key is rotated, not revealed (`core/services/credentials.ts:808-810`, `core/services/credentials.ts:830`).
- **Rotate with a grace window.** `graceMinutes` is 0 to 1440 and defaults to 0 (`src/admin-api/endpoints/credentials.ts:120`, `core/services/credentials.ts:572`). A rotation creates a successor with a newly generated key (`core/services/credentials.ts:574`, `core/services/credentials.ts:615-617`). With a grace of 0 the old key ends at once; otherwise it keeps working until the grace ends or its own earlier expiry (`core/services/credentials.ts:590-594`). The caller switches to the new `kid` while both work.
- **Revoke.** Sets `revoked_at`. It also ends older keys of the same rotation chain that are still inside their grace window (`core/services/credentials.ts:715-720`). It takes effect on the next call because nothing is cached.
- Service keys appear in the credential list, rotate, revoke and expiring views like the other inbound kinds (`core/services/credentials.ts:459`, `core/services/credentials.ts:497`).
- **Switching a type's auth policy.** `integrationTypes.setAuthPolicy` (`auth-policy:write`) changes only the type row and leaves its systems' service keys untouched (`core/services/integration-types.ts:88-92`). The gateway asks for a token only while the policy is `partner+service` (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:297`), so after a switch to `partner` the keys stay in place but unused and the Credentials tab keeps listing them. Switching back makes every unrevoked, unexpired key valid again (`core/auth/service-token.ts:82-95`); revoke what should end. A system of a `partner+service` type with no usable key answers every call with `401`, so issue its keys before setting it `ACTIVE`.

### A service token is never a partner credential

A Bearer value that matches no partner credential, looks like a JWT and carries a uuid `kid` is checked against the URL system's own service keys only. If its signature verifies (claims ignored), the request is refused with the generic `401` and the activity reason "service token presented as partner credential" (`core/auth/with-partner-auth.ts:233-243`, `core/auth/service-token.ts:176-196`). That reason exists to make a misconfigured client visible in the log. It never authenticates anything. A service key sent as `X-API-Key` is not even that: only a Bearer value gets the special reason, so any other placement is a plain `401` "invalid credential" (`core/auth/with-partner-auth.ts:238`).

### Minting a token

For testing, or for a partner's backend that wants a reference, the repository has a mint script:

```
SERVICE_KEY=<key text> npm run mint:service-token -- --kid <kid> --sub <subject> --sys <system code> --ttl 300
```

| Flag | Required | Meaning | Source |
|---|---|---|---|
| `--kid` | Yes | The key id (a uuid) the issue call returned | `scripts/mint-service-token.ts:43-46` |
| `--sys` | Yes | The system code the token is for | `scripts/mint-service-token.ts:47-50` |
| `--sub` | Yes | The token subject, for example a worker name | `scripts/mint-service-token.ts:51-54` |
| `--ttl` | No | Lifetime in whole seconds, 1 to 900, default 300 | `scripts/mint-service-token.ts:29`, `scripts/mint-service-token.ts:55-59` |

The key text comes from the `SERVICE_KEY` shell variable of the person running the script, never from an argument, so it stays out of shell history and process listings (`scripts/mint-service-token.ts:8-11`, `scripts/mint-service-token.ts:38`). `SERVICE_KEY` is not an application environment key and is never set on a deployment. The script prints only the token on stdout, so it can be captured with `$(...)`. Errors go to stderr and never echo the key (`scripts/mint-service-token.ts:10-12`).

Send the result next to the partner credential:

```
curl -X POST https://interchange-livid.vercel.app/api/inbound/<systemCode>/<endpoint> \
  -H "X-API-Key: <partner key>" \
  -H "X-Service-Token: <token>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

### Accepted risk: replay inside the lifetime

There is no `jti` store, so a captured token can be replayed until it expires: at most 900 seconds plus the 30 second tolerance (`core/auth/service-token.ts:35-41`). An attacker still needs a valid partner credential for the same system. Keep tokens short and send them over https only.

## What a caller sees on failure

Every unauthenticated outcome returns the same body, so a caller cannot tell a missing credential from an unknown system, an inactive system, a wrong secret or a revoked credential. The route builds exactly two bodies from the result: `{"error":"forbidden"}` for a `403` and `{"error":"unauthorized"}` for everything else (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:282-284`, `route.ts:307`). The real reason goes only to the activity log (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:278-281`).

| Situation | Response | Reason in the activity row only |
|---|---|---|
| No credential | `401 {"error":"unauthorized"}` | missing credential |
| Unknown or deleted system | `401` | unknown system |
| System not `ACTIVE` | `401` | system not active |
| Wrong secret, unknown credential | `401` | invalid credential |
| Revoked or expired credential of this system | `401` | credential revoked, credential expired |
| Live credential of another system | `403 {"error":"forbidden"}` | credential belongs to another system |
| `partner+service` and no `X-Service-Token` | `401` | missing service token |
| Bad token, unknown or revoked key, expired, lifetime too long | `401` | invalid service token, unknown service key, service token expired, service token lifetime too long |
| Key row cannot be decrypted | `401` | service key unreadable |
| `sys` names another system | `403` | service token for another system |

Sources: `core/auth/with-partner-auth.ts:205`, `core/auth/with-partner-auth.ts:335`, `core/auth/with-partner-auth.ts:351`, `core/auth/with-partner-auth.ts:357`, `core/auth/with-partner-auth.ts:118-123`, `core/auth/with-partner-auth.ts:71`, `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:300`, `core/auth/service-token.ts:57`, `core/auth/service-token.ts:144-146`, `core/auth/service-token.ts:159`, `core/auth/service-token.ts:165`, `core/auth/service-token.ts:169`.

Two timing details keep the generic answer honest. An unknown or inactive system spends one hash computation, so it takes as long as a wrong secret (`core/auth/with-partner-auth.ts:348-350`, `core/auth/with-partner-auth.ts:355-356`). A miss on Basic spends the same two hashes as a real miss (`core/auth/with-partner-auth.ts:283-287`).

One exception to the "stranger learns nothing" rule is deliberate: a `403` needs a verified live secret, so it only tells a caller who already holds a valid credential that the system code in the URL exists (see [security.md](security.md#cross-system-confinement)).

## Related

- [security.md](security.md): the vault, the threat model for a leaked credential, and everything else about security.
- [rate-limiting.md](rate-limiting.md): the per-IP, per-system and per-endpoint limits on this route.
- The operations runbook (in the source repository): rotating and revoking keys during an incident.
