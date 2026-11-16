# Rate limiting

Last updated: 2026-11-17

Interchange has more than one limiter. They all call the same function, `consumeRateLimit` (`core/gateway/rate-limit.ts:64`), which counts requests per key in one-minute fixed windows in the `rate_limit_bucket` table, unless a caller passes a longer window. This document lists every limiter, its key, its default and where it lives, then explains what a rejected caller sees and the limits of the mechanism. For the security reasoning behind the inbound and admin limits, see [security.md](security.md).

## The limiters

| Limiter | Key | Default | Source |
|---|---|---|---|
| Inbound per IP | `ip:<ip>` | 250 per minute (`DEFAULT_IP_RPM`), override with `RATE_LIMIT_IP_RPM` | `core/gateway/rate-limit.ts:17`, `core/gateway/rate-limit.ts:34-38`, `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:264` (the `OPTIONS` preflight spends the same bucket at `route.ts:103`) |
| Inbound per system | `sys:<code>` | The system's own `rateLimitRpm` | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:322` |
| Inbound per endpoint | `sys:<code>:<endpoint>` | The endpoint's own `limit.rpm` (only for a system that declares endpoint limits) | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:352` |
| Admin API per IP | `admin:ip:<ip>` | 300 per minute (`DEFAULT_ADMIN_IP_RPM`), override with `ADMIN_RATE_LIMIT_IP_RPM` | `core/http/with-admin.ts:128`, `core/http/with-admin.ts:133-136`, `core/http/with-admin.ts:150` |
| Admin API per user | `admin:user:<user id>` | 120 per minute (`DEFAULT_ADMIN_USER_RPM`), override with `ADMIN_RATE_LIMIT_USER_RPM` | `core/http/with-admin.ts:129`, `core/http/with-admin.ts:165` |
| Login per IP | `login:ip:<ip>` | 50 per 15 minutes (`LOGIN_IP_LIMIT`) | `core/auth/login.ts:22`, `core/auth/login.ts:24`, `core/auth/login.ts:63` |
| Login per email | `login:email:<hash of the normalised email>` | 10 per 15 minutes (`LOGIN_EMAIL_LIMIT`) | `core/auth/login.ts:22-23`, `core/auth/login.ts:62`, `core/auth/login.ts:64` |
| Profile update per user | `profile:user:<user id>` | 10 per minute (`PROFILE_UPDATE_LIMIT`) | `core/services/profile.ts:19`, `core/services/profile.ts:29` |
| Change password per user | `pwchange:user:<user id>` | 5 per 15 minutes (`PWCHANGE_LIMIT`, `PWCHANGE_WINDOW_MS`) | `core/services/profile.ts:20-21`, `core/services/profile.ts:49` |
| Credential reveal (system credential) | `admin:reveal:<user id>` | 10 per 10 minutes (`DEFAULT_REVEAL_LIMIT`, `REVEAL_WINDOW_MS`), override the count with `CREDENTIALS_REVEAL_LIMIT` | `core/services/credentials.ts:798-804`, `core/services/credentials.ts:823` |
| Credential reveal (integration type default) | `admin:reveal:<user id>` (the same bucket) | Same limit and window | `core/services/type-credentials.ts:232` |
| Playground validate, per user | `pg:user:<user id>:validate` | 30 per minute | `core/pure/playground.ts:12-13`, `core/pure/playground.ts:20`, `core/services/playground.ts:104` |
| Playground preflight, per user | `pg:user:<user id>:preflight` | 30 per minute | `core/pure/playground.ts:14`, `core/services/playground.ts:173` |
| Playground send, per user | `pg:user:<user id>:send` | 10 per minute | `core/pure/playground.ts:15`, `core/services/playground.ts:213` |
| Playground send, per system | `pg:sys:<system code>` | 20 per minute | `core/pure/playground.ts:16`, `core/pure/playground.ts:21`, `core/services/playground.ts:214` |
| Storage Test, per user | `admin:storage-test:<user id>` | 5 per minute | `core/services/storage.ts:61`, `core/services/storage.ts:63`, `core/services/storage.ts:374` |
| Storage Test, per system | `admin:storage-test:sys:<system id>` | 10 per minute | `core/services/storage.ts:62`, `core/services/storage.ts:378-382` |
| SFTP Test, per user | `admin:sftp-test:<user id>` | 5 per minute | `core/services/sftp.ts:37`, `core/services/sftp.ts:39`, `core/services/sftp.ts:219-223` |
| SFTP Test, per system | `admin:sftp-test:sys:<system id>` | 10 per minute | `core/services/sftp.ts:38`, `core/services/sftp.ts:230-234` |
| Alert delivery Test, per user | `admin:alert-test:<user id>` | 5 per minute (`ALERT_TEST_USER_LIMIT`, `ALERT_TEST_WINDOW_MS`) | `core/services/alert-delivery.ts:31-32`, `core/services/alert-delivery.ts:107` |

Notes on the table:

- **Order matters.** On an inbound call the per-IP bucket is spent first, before any credential work (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:263`). The per-system and per-endpoint buckets are spent only after the partner has authenticated, so a stranger cannot burn a partner's quota (`route.ts:321`). On the admin API the per-IP bucket runs before CSRF and session checks, and the per-user bucket only after authorization (`core/http/with-admin.ts:148-151`, `core/http/with-admin.ts:163-166`).
- **Env variables are read per call.** The three env-tunable limiters treat an unset, empty, non-numeric or non-positive value as "use the default" and round a fraction down (`core/gateway/rate-limit.ts:35-38`, `core/http/with-admin.ts:133-136`, `core/services/credentials.ts:801-804`).
- **Reveal is spent before the lookup.** A disabled reveal flag spends nothing, but once reveal is on, the budget is spent before the credential is looked up, so probing ids costs budget (`core/services/credentials.ts:816-823`).
- **Playground order.** The user bucket and then the system bucket are spent before the deleted, status and direction checks, so a refused call still costs budget (`core/services/playground.ts:206-214`).
- **Keys are bounded.** A key is cut to 200 characters before it is stored (`core/gateway/rate-limit.ts:18`, `core/gateway/rate-limit.ts:71`).
- **Alert delivery Test spends first.** The per-user bucket is spent before the stored setting is read, so a call with no URL set still costs budget (`core/services/alert-delivery.ts:107-112`). Delivery of real alerts is not rate limited: it is bounded by the 30 minute throttle, one POST per evaluation and 20 items per POST (see [alerting.md](alerting.md)).
- **Several limiters can apply to one request.** The admin per-IP and per-user limits sit on top of every limit below them in the admin rows (`core/http/with-admin.ts:150`, `core/http/with-admin.ts:165`).

## What a rejected caller sees

The shape depends on the surface.

| Surface | Status and body | Headers on the 429 | Source |
|---|---|---|---|
| Inbound (`/api/inbound/*`) | `429 {"error":"rate limit exceeded"}` | `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, plus `X-Request-ID` and `Vary: Origin` | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:268`, `route.ts:201-207`, `core/gateway/rate-limit.ts:101-110` |
| Inbound `OPTIONS` preflight | `429` with no body | Same four rate-limit headers, plus `X-Request-ID` and `Vary: Origin` | `src/app/api/inbound/[systemCode]/[endpoint]/route.ts:104-109` |
| Admin API, per-IP or per-user limiter | Error envelope with code `RATE_LIMITED` and message "too many requests" | `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` | `core/http/with-admin.ts:138-140`, `core/gateway/rate-limit.ts:101-110` |
| Admin API, limiter inside a service (reveal, profile, change password, playground, Storage Test, SFTP Test, Alert delivery Test) | Error envelope with code `RATE_LIMITED` and a fixed message | `Retry-After` only, no `X-RateLimit-*` | `core/http/with-admin.ts:186-192`, for example `core/services/credentials.ts:824-826` |
| Login (per IP or per email) | No 429: the caller gets the same failure as a wrong password | None | `core/auth/login.ts:65-76` |

`Retry-After` is the whole number of seconds until the current window ends, never less than 1 (`core/gateway/rate-limit.ts:74`). `X-RateLimit-Reset` is the epoch second when the window ends (`core/gateway/rate-limit.ts:73`). `Retry-After` is added only when the request was refused (`core/gateway/rate-limit.ts:108`), while the `X-RateLimit-*` set is also sent on an inbound response that was admitted. On an admitted inbound call the headers report the tightest bucket the request consumed (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:312-319`). The admin API sends the `X-RateLimit-*` headers only on a 429 raised by the per-IP or per-user limiter (`core/http/with-admin.ts:138-140`).

The admin HTTP client surfaces `Retry-After` as `retryAfterSec` on the thrown error, as described in [security.md](security.md#admin-rate-limit).

## Fixed window, not sliding

Each key has one counter per window. The window start is the current time rounded down to the window length (`core/gateway/rate-limit.ts:72`), and each check is one atomic insert-or-increment statement (`core/gateway/rate-limit.ts:76-83`). When the window rolls over, the count starts again at 1.

The consequence is a known limit: a burst that straddles a window boundary can pass up to twice the limit within a few seconds (a full allowance at the end of one window and another at the start of the next). Size per-system and per-endpoint limits with that in mind. The in-browser demo uses a sliding window instead (see the playground and storage notes in [security.md](security.md)), so the demo can differ from the server at a boundary.

## A burst inside the limit can still exhaust the database

A limit caps how many requests are admitted per window, not how many run at once. In a live check on 2026-11-16, 130 concurrent admin reads by one user against the live deployment (free database tier) gave 10 refusals with `429` and `Retry-After` as designed, but 16 of the admitted requests failed with `500` because the database ran out of memory (SQLSTATE 53200). The per-user admin limit of 120 per minute therefore does not protect a small database from one user's concurrent burst.

Such a failure now answers `503` with `Retry-After: 5`, not `500`. A short-lived database failure (SQLSTATE 53000, 53200, 53300, 53400, 57P01, 57P02, 57P03, class 08, or a dropped or timed-out connection) is detected by `isTransientDbError` (`core/services/errors.ts:48-63`) and answered as `UNAVAILABLE` on the admin API (`core/http/with-admin.ts:197-204`, `src/admin-api/contract.ts:99`) and as `{"error":"UNAVAILABLE"}` on the inbound gateway (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:627-635`). The inbound `503` is given only until the job row is written; after that a failure stays `500`, because the write may have committed and a retry of a call without an `Idempotency-Key` could duplicate it (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:166-172`). The console shows "The service is busy" and does not retry a write by itself (`src/features/shared/ui.tsx:33`). Each instance waits at most 10 seconds for a database client, so an exhausted pool fails as a transient error instead of hanging (`core/db.ts:17-21`). This bounds the damage; it does not raise the capacity of a small database.

Once the functions ran next to the database (`vercel.json`, see `docs/deployment.md`), each request held a connection for milliseconds instead of seconds, and the same burst of 130 on 2026-11-16 gave 120 responses of `200` and 10 of `429` with `Retry-After`, with no `500` and no `503`. A larger burst can still exhaust a small database; it then answers `503`.

## Fails open on a store error

If the database cannot be reached, `consumeRateLimit` logs one warning and admits the request (`core/gateway/rate-limit.ts:94-97`). No rate-limit headers are sent in that case (`core/gateway/rate-limit.ts:102`), and the inbound route ignores a failed-open result when it picks the bucket to report (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:316-317`).

This is a documented, deliberate limit: a limiter outage must not become a gateway outage or lock administrators out (`core/gateway/rate-limit.ts:9-13`). The cost is brief over-admission while the store is down. Because the login throttle uses the same function, it fails open too.

The per-IP limits are only as sound as the client address. The address is read from `x-vercel-forwarded-for`, then `x-real-ip`, then the first `x-forwarded-for` entry, and must parse as an IP, otherwise it is `unknown` and shares one bucket (`core/gateway/rate-limit.ts:42-62`). Behind a host that passes client-supplied headers through, a caller can pick its own bucket.

## Not a limiter

`MANUAL_RUN_WINDOW_MS` (`core/services/schedules.ts:39`) is not a rate limiter and does not use `rate_limit_bucket`. It is a 10 second guard on one schedule: a second manual run of the same schedule inside the window is refused, decided by a compare-and-swap on `last_manual_at` in the same transaction as the job insert (`core/services/schedules.ts:278-306`). It protects a schedule from a double click, not the platform from load.

## Monitoring

The Rate limits page in the console shows the current window's gateway buckets and the 429 responses of the last hour. See [security.md](security.md#rate-limit-monitor).

## Related

- [security.md](security.md): why each limit exists and the threat model.
- [deployment.md](deployment.md): the environment variables that tune these limits.
- The testing guide (in the source repository): load testing the gateway limits with k6.
